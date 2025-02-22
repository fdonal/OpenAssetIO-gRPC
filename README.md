# OpenAssetIO gRPC

## Goal

Provide a low-friction way for hosts integrate OpenAssetIO such that
side-effects on it's runtime are minimised.

> **Note:**
> This is a proof-or-concept project that aims to explore and understand
> the potential benefits, and costs of using gRPC to achieve this goal.
> The code is by no means well robust or well tested.

## Background

The use of Python in common deployment scenarios can cause assorted
problems. In applications where the UI is implemented using some Python
based framework, the dreaded GIL means that any other work done in
Python can ultimately stall the UI event loop. For example, a large
number of calls to a python Manager implementation from a background
work thread.

## Approach

Employ gRPC to move OpenAssetIO functionality out-of-process. This will
add some per-call overhead, but in theory provides the following
benefits:

- Decouples the Host runtime and its dependencies from OpenAssetIO,
  this could include compiler or python versions.
- Manager caching could be leveraged across multiple processes.
- Could be used as a bridge into other languages.
- Multiple out-of-process servers could be pooled.

There are some downsides to this idea:

- There is most likely to be a non-trivial performance overhead
  compared to an in-process implementation.
- The manager plugin no longer has direct access to the host
  environment.

### Architecture

The project consists of an alternate `ManagerImplementationFactory`
(`GRPCManagerImplementationFactrory`). When used by a host, this returns
a custom `ManagerInterface` shim, that proxies all API calls to the
configured gRPC endpoint - assuming it implements the service defined in
[openassetio.proto](./src/openassetio.proto).

In theory, this shim could be manually instantiated by other means, but
this is currently unsupported.

There are three components to this project:

- The `openassetio-grpc` library, that provides an SDK for OpenAssetIO
  Hosts.
- The `openassetio-grpc-server` binary that instantiates managers to
  service the bridged API requests.
- The `openassetio-grpc-testhost` that provides a high-level
  integration example.

### Building

The code currently requires gRPC and OpenAssetIO to be installed in
such a way as to be visible to CMake. The included `conanfile.txt` can
be used along with `cmake --toolchain conan_paths.cmake` if so desired.

Assuming your working directory is in the root of this repository.

```bash
cmake -S . build
cmake --build build && cmake --install build
```

This will populate `build/dist` with the project components.

> **Warning:**
> The current build infrastructure is somewhat work-in-progress.
> Components are built as shared libraries to avoid having to juggle
> the gPRC global state at this point.

### Swapping out the factory

All you need to do to use the bridge in a host is to use the gRPC
manager impl factory in place of your existing one (`auto` used for
brevity). NB. Port configuration is currently hard coded.

```cpp
auto logger = ...
auto implFactory = GRPCManagerImplementationFactory::make("0.0.0.0:50051", logger);
auto hostInterface = ...
auto mgrFactory = ManagerFactory::make(hostInterface, implFactory, logger);
```

## Testing steps

> **Note:**
> In its present state, the testing process is somewhat manual. A
> production version of this code would probably take care of managing a
> server process for you.

The following assumes you are at the root of this repository and have
successfully built the various components.

### Server

The `openassetio-grpc-server` binary receives requests over gRPC and
fulfills them using managers it instantiates using the usual means (i.e
the `PythonPluginSystemManagerImplementationFactory`). As such, it links
to Python. This means we needs to set up a few env vars before it can be
run:

```shell
# Required for the embedded interpreter
export PYTHONHOME=/path/to/python/root/used/for/build
# Ensure OpenAssetIO + bridge libs are available
export LD_LIBRARY_PATH=./build/dist/lib:/path/to/OpenAssetIO/libs
export PYTHONPATH=/path/to/OpenAssetIO/site-packages:$PYTHONPATH
# Keep an eye on things
export OPENASSETIO_LOGGING_SEVERITY=0
```

N.B If `importlib_metadata` is missing from the Python distribution you
used to build, you'll need to manually set `$OPENASSETIO_PLUGIN_PATH` if
you are relying on entry point discovery.

```bash
export OPENASSETIO_PLUGIN_PATH=/path/to/OpenAssetio-Manager-BAL/plugin
```

You should now be able to run the server:

```bash
$ ./build/dist/openassetio-grpc-server
       info: Server listening on 0.0.0.0:50051
```

### Test host

The `openassetio-grpc-testhost` is a simple C++ CLI tool that uses
the gRPC bridge to resolve a string through the default configured
manager (see [here]("https://openassetio.github.io/OpenAssetIO/classopenassetio_1_1v1_1_1host_api_1_1_manager_factory.html#a8b6c44543faebcb1b441bbf63c064c76)
for more info).

Similar to the server, it currently needs a few vars setting. In a new
shell:

```bash
# Ensure OpenAssetIO and bridge libs are available
export LD_LIBRARY_PATH=./build/dist/lib:/path/to/OpenAssetIO/libs
# Ensure we can see the program output
export OPENASSETIO_LOGGING_SEVERITY=0
```

As the test host uses the default manager config mechanism, we can use
the included BAL setup for a quick test:

```bash
export OPENASSETIO_DEFAULT_CONFIG=./openassetio_conf.toml
```

Running the test host with a valid entity ref from the test library
should return some data:

```bash
$ ./build/dist/openassetio-grpc-testhost bal:///anAsset
       info: Available managers:
   debugApi: gRPC: Instantiated 'org.openassetio.examples.manager.bal' [0x7fb2c8003c40]
       info: Basic Asset Library 📖 [org.openassetio.examples.manager.bal]
       info: Done
       info: Default manager:
      debug: Loading default manager config from './openassetio_conf.toml' [OPENASSETIO_DEFAULT_CONFIG]
   debugApi: gRPC: Instantiated 'org.openassetio.examples.manager.bal' [0x7fb2c8005d70]
       info: Basic Asset Library 📖
       info: Management Policy for openassetio-mediacreation:content.LocatableContent [read]:
       info: openassetio-mediacreation:managementPolicy.Managed
       info: Management Policy for openassetio-mediacreation:content.LocatableContent [write]:
       info: Resolving bal:///anAsset
       info: openassetio-mediacreation:content.LocatableContent: file:///dev/null

```

## Notes and limitations

- There is currently very little in the way of error handling, the
  unhappy path will most likely cause either the test host of server to
  fall over.

- No conscious effort has been made to optimize the implementation or
  proto.

## Todo

- Complete `ManagerInterface` methods.
- Return logging to the host.
- Manager servers in the impl factory.
- Proper error handling.
- Proper test coverage.
- Support manager state objects.
- Allow the shim interface to be user-configured without the custom
  factory.
