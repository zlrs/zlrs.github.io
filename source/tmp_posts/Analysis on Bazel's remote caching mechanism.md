# Analysis on Bazel's remote caching mechanism
## What is bazel's remote-caching mechanism? Why do you need it?
The bazel's build cache can save compile time by reusing outputs produced by previous builds. The build cache is stored locally in the traditional way. However, Bazel allows you to set up a server to be the remote cache for build outputs. This means you can reuse build outputs from machine to machine, which will definitely optimize the compile time of very large codebases. 
## What is a cache key? What is it based on?
A cache key uniquely associates to an output artifact of an action.
Bazel breaks a build into discrete steps, which are called actions. Each action has inputs, output names, a command line, and environment variables. Required inputs and expected outputs are declared explicitly for each action.
Bazel computes a cache key that uniquely defines the action's output artifacts. Some factors contribute to the generation of the cache key, according to the comment of a Bazel commiter.
- Command line options
- Input files
- Output files
- Environment variables for each action
The relative paths, not absolute paths, are used for the cache key.
The computation of cache keys
## Where are cache keys generated?
Bazel's cache keys, in the view of C-S architecture, are originally generated from the client side(i.e. from the bazel command). The cache's backend can be just plain object storage(such as Google Cloud Storage or Amazon S3). 
This design makes it possible that 
(1) the key computation logic resides in the bazel repository. 
(2) The simplicity of the storage backend. As a result, users can easily utilize handy cloud object storage services or build one of their own.
## Computation method
Relevent project paths:
- https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/actions/cache
- https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/actions/ActionKeyCacher.java
- https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/rules/cpp/CppCompileAction.java
- https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/util/Fingerprint.java
`ActionCache` can be viewed as an object pointer on the client-side, which contains the cache key of the object. Through the cache key, `ActionCache` points to the object stored in remote-caching server (if the action is cached).
The abstract class, `ActionKeyCacher`, declares the `computeKey()` interface to compute cache keys. Each concrete action, if it wants to be cached, implements the method based on its own requirement. The implementation must ensure that the cache key is unique between different runs of the action that produce different output. Here is a simple class hierarchy, just to help understand the relationship among classes. 
![](media/16379133229260/16379133458344.jpg)

`Fingerprint` is a cryptographic hash of a message that encodes the representation of an general object. It is used to collect factors contributing to the hash method and generate the cache key.
Here is the source code that computes the cache key of a `CppCompileAction`. It appears the fingerprint object takes (at least) the following data as the hash's input:
- The actionClassId
  - Identifier for the actual execution time behavior of the action.
  - hard-coded in Bazel source code
- Action environment
- Command line arguments necessary for cache correctness
- Declared header files in srcs or hdrs
- Declared include directories
- ...

```cpp
// https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/rules/cpp/CppCompileAction.java
/** For actions that discover inputs, the key must include input names. */
@Override
public void computeKey(
    ActionKeyContext actionKeyContext,
    @Nullable Artifact.ArtifactExpander artifactExpander,
    Fingerprint fp)
    throws CommandLineExpansionException, InterruptedException {
  computeKey(
      actionKeyContext,
      fp,
      actionClassId,
      env,
      compileCommandLine.getEnvironment(),
      executionInfo,
      getCommandLineKey(),
      ccCompilationContext.getDeclaredIncludeSrcs(),
      getMandatoryInputs(),
      additionalPrunableHeaders,
      ccCompilationContext.getLooseHdrsDirs(),
      builtInIncludeDirectories,
      inputsForInvalidation,
      cppConfiguration.validateTopLevelHeaderInclusions());
}

// Separated into a helper method so that it can be called from CppCompileActionTemplate.
static void computeKey(
    ActionKeyContext actionKeyContext,
    Fingerprint fp,
    UUID actionClassId,
    ActionEnvironment env,
    Map<String, String> environmentVariables,
    Map<String, String> executionInfo,
    byte[] commandLineKey,
    NestedSet<Artifact> declaredIncludeSrcs,
    NestedSet<Artifact> mandatoryInputs,
    NestedSet<Artifact> prunableHeaders,
    NestedSet<PathFragment> declaredIncludeDirs,
    List<PathFragment> builtInIncludeDirectories,
    NestedSet<Artifact> inputsForInvalidation,
    boolean validateTopLevelHeaderInclusions)
    throws CommandLineExpansionException, InterruptedException {
  fp.addUUID(actionClassId);
  env.addTo(fp);
  fp.addStringMap(environmentVariables);
  fp.addStringMap(executionInfo);
  fp.addBytes(commandLineKey);
  fp.addBoolean(validateTopLevelHeaderInclusions);

  actionKeyContext.addNestedSetToFingerprint(fp, declaredIncludeSrcs);
  fp.addInt(0); // mark the boundary between input types
  actionKeyContext.addNestedSetToFingerprint(fp, mandatoryInputs);
  fp.addInt(0);
  actionKeyContext.addNestedSetToFingerprint(fp, prunableHeaders);

  /*
   * getArguments() above captures all changes which affect the compilation command and hence the
   * contents of the object file. But we need to also make sure that we re-execute the action if
   * any of the fields that affect whether {@link #validateInclusions} will report an error or
   * warning have changed, otherwise we might miss some errors.
   */
  actionKeyContext.addNestedSetToFingerprint(fp, declaredIncludeDirs);
  fp.addPaths(builtInIncludeDirectories);

  // This is needed for CppLinkstampCompile.
  fp.addInt(0);
  actionKeyContext.addNestedSetToFingerprint(fp, inputsForInvalidation);
}
```

## bazel-remote codebase structure
bazel-remote is an open source remote build cache that can be used as the backend of Bazel's remote caching system. It is typically deployed on a remote server. It stores cached data in the disk of the server and submits it to an optional proxy if it's configured; and provides cached data from the disk or from the same proxy. 
Bazel-remote provides HTTP and GRPC(optional) service. Let's focus on the HTTP way since it is more familiar to most people. The principle is the same, though.
The HTTP service can handle PUT/GET/HEAD requests from the client(bazel command). 
For PUT request, the data is stored in the local disk directory configured at program's initialization with the cache key required by the client, and proxyed to the optional proxy server.
For GET request, the local disk directory is searched by the path derived by the cache key required by the client. If a local cache miss occurs, the request is proxyed to the optional proxy server.
For HEAD request, which is used to query if server contains a particular cache, the local disk directory is searched first, or the request is handed over to the proxy server if there is a local miss.

![](media/16379133229260/16379133905949.jpg)


## Userful links
- Official doc covering remote-caching
https://docs.bazel.build/versions/main/remote-caching.html
- Bazel committers talk about the factors affecting the generation of hash key.
https://github.com/bazelbuild/bazel/issues/2998#issuecomment-301160663
- Discuss threads on the need to use environment variables for an action, and how it can affect caching, and the introduction of --action-env
https://bazel.build/designs/2016/06/21/environment.html
https://github.com/bazelbuild/bazel/issues/3320
https://github.com/bazelbuild/bazel/issues/4558
https://github.com/bazelbuild/bazel/commit/6e940e573d20a3220ac433901c5650ee74226a17?diff=unified
- An open source remote-caching server for bazel
https://github.com/buchgr/bazel-remote
