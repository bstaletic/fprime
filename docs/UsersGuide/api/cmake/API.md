**Note:** auto-generated from comments in: ./API.cmake

## API.cmake:

API of the fprime CMake system. These functions represent the external interface to all of the fprime CMake system.
Users and developers should understand these functions in order to perform basic CMake setup while building as part
of an fprime project.

The standard patterns include:
- Add a directory to the fprime system. Use this in place of add_subdirectory to get cleanly organized builds.
- Register an fprime module/executable/ut to receive the benefits of autocoding.
- Register an fprime build target/build stage to allow custom build steps. (Experimental)



## Function `add_fprime_subdirectory`:

Adds a subdirectory to the build system. This allows the system to find new available modules,
executables, and unit tests. Every module, used or not, by the deployment/root CMAKE file should
be added as a subdirectory somewhere in the tree. CMake's dependency system will prevent superfluous building, and
`add_fprime_subdirectory` calls construct the super-graph from which the build graph is realized. Thus
it is inconsequential to add a subdirectory that will not be used, but all code should be found within this
super-graph to be available to the build.

Every subdirectory added should declare a `CMakeLists.txt`. These in-turn may add their own sub-
directories. This creates a directed acyclic graph of modules, one subgraph of which will be built
for each executable/module/library defined in the system.  The subgraph should also be a DAG.

This directory is computed based off the closest path in `FPRIME_BUILD_LOCATIONS`. It must be set to
be used. Otherwise, an error will occur.

A user can specify an optional argument to set the build-space, creating a sub-directory under
the `CMAKE_BINARY_DIR` to place the outputs of the builds of this directory. This is typically
**not needed**. `EXCLUDE_FROM_ALL` can also be supplied.
See: https://cmake.org/cmake/help/latest/command/add_fprime_subdirectory.html

**Note:** Replaces CMake `add_subdirectory` call in order to automate the [binary_dir] argument.
          fprime subdirectories have specific binary roots to avoid collisions, and provide for
          the standard fprime #include paths rooted at the root of the repo.

**Arguments:**
 - **FP_SOURCE_DIR:** directory to add (same as add_directory)
 - **EXCLUDE_FROM_ALL:** (optional) exclude any targets from 'all'. See:
                         https://cmake.org/cmake/help/latest/command/add_fprime_subdirectory.html


## Function `register_fprime_module`:

Registers a module using the fprime build system. This comes with dependency management and fprime
autocoding capabilities. The caller should first set two variables before calling this function to define the
autocoding and source inputs, and (optionally) any non-standard link dependencies.

Required variables (defined in calling scope):

- **SOURCE_FILES:** cmake list of input source files. Place any "*Ai.xml", "*.c", "*.cpp"
  etc files here. This list will be split into autocoder inputs, and hand-coded sources based on the name/type.

**i.e.:**
```
set(SOURCE_FILES
    MyComponentAi.xml
    SomeFile.cpp
    MyComponentImpl.cpp)
```
- **MOD_DEPS:** (optional) cmake list of extra link dependencies. This is optional, and only
  needed if non-standard link dependencies are used, or if a dependency cannot be inferred from the include graph of
  the autocoder inputs to the module. If not set or supplied, only fprime
  inferable dependencies will be available. Link flags like "-lpthread" can be here.

**i.e.:**
```
set(LINK_DEPS
    Os
    Module1
    Module2
    -lpthread)
```

**Note:** if desired, these fields may be supplied in-order as arguments to the function. Passing
          these as positional arguments overrides any specified in the parent scope.  This is typically not done.


### Standard `add_fprime_module` Example ###

Standard modules don't require extra modules, and define both autocoder inputs and standard source
files. Thus, only the SOURCE_FILE variable needs to be set and then the register call can be made.
This is the only required lines in a module CMakeLists.txt.

```
set(SOURCE_FILE
    MyComponentAi.xml
    SomeFile.cpp
    MyComponentImpl.cpp)

register_fprime_module()
```

### Non-Autocoded and Autocode-Only Modules Example ###

Modules that do not require autocoding need not specify *.xml files as source. Thus, code-only modules just define
*.cpp. **Note:** no dependency inference is done without autocoder inputs.

```
set(SOURCE_FILE
    SomeFile1.cpp
    Another2.cpp)

register_fprime_module()
```
Modules requiring only autocoding can just specify *.xml files.

```
set(SOURCE_FILE
    MyComponentAi.xml)

register_fprime_module()
```

### Specific Dependencies and Linking in Modules Example ###

Some modules need to pick a specific set of dependencies and link flags. This can be done
with the `MOD_DEPS` variable. This feature can be used to pick specific implementations
for some fprime modules that implement to a generic interface like the console logger implementation.

```
set(SOURCE_FILE
    MyComponentAi.xml
    SomeFile.cpp
    MyComponentImpl.cpp)

set(MOD_DEPS
    Module1
    -lpthread)

register_fprime_module()
```



## Function `register_fprime_executable`:

Registers an executable using the fprime build system. This comes with dependency management and
fprime autocoding capabilities. This requires three variables to define the executable name,
autocoding and source inputs, and (optionally) any non-standard link dependencies.

Required variables (defined in calling scope):


- **EXECUTABLE_NAME:** (optional) executable name supplied. If not set, nor passed in, then
                    PROJECT_NAME from the CMake definitions is used.

- **SOURCE_FILES:** cmake list of input source files. Place any "*Ai.xml", "*.c", "*.cpp"
                 etc. files here. This list will be split into autocoder inputs and sources.
**i.e.:**
```
set(SOURCE_FILES
    MyComponentAi.xml
    SomeFile.cpp
    MyComponentImpl.cpp)
```

- **MOD_DEPS:** (optional) cmake list of extra link dependencies. This is optional, and only
  needed if non-standard link dependencies are used, or if a dependency cannot be inferred from the include graph of
  the autocoder inputs to the module. If not set or supplied, only fprime
  inferable dependencies will be available. Link flags like "-lpthread" can be here.

**i.e.:**
```
set(LINK_DEPS
    Module1
    Module2
    -lpthread)
```

**Note:** if desired, these fields may be supplied in-order as arguments to the function. Passing
          these as positional arguments overrides any specified in the parent scope.

**Note:** this operates almost identically to `register_fprime_module` with respect to the variable definitions. The
          difference is this call will yield an optionally named linked binary file.

### Standard fprime Deployment Example ###

To create a standard fprime deployment, an executable needs to be created. This executable
uses the CMake PROJECT_NAME as the executable name. Thus, it can be created with the following
source lists. In most fprime deployments, some modules must be specified as they don't tie
directly to an Ai.xml.

```
set(SOURCE_FILES
  "${CMAKE_CURRENT_LIST_DIR}/RefTopologyAppAi.xml"
  "${CMAKE_CURRENT_LIST_DIR}/Topology.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/Main.cpp"
)
# Note: supply non-explicit dependencies here. These are implementations to an XML that is
# defined in a different module.
set(MOD_DEPS
  Svc/PassiveConsoleTextLogger
  Svc/SocketGndIf
  Svc/LinuxTime
)
register_fprime_executable()
```
### fprime Executable With Autocoding/Dependencies ###

Developers can make executables or other utilities that take advantage of fprime autocoding
and fprime dependencies. These can be registered using the same executable registrar function
but should specify a specific executable name.

```
set(EXECUTABLE_NAME "MyUtility")

set(SOURCE_FILES
  "${CMAKE_CURRENT_LIST_DIR)/ModuleAi.xml"
  "${CMAKE_CURRENT_LIST_DIR}/Main.cpp"
)
set(MOD_DEPS
  Svc/LinuxTime
  -lm
  -lpthread
)
register_fprime_executable()
```



## Function `register_fprime_ut`:

Registers an executable unit-test using the fprime build system. This comes with dependency
management and fprime autocoding capabilities. This requires three variables defining the
unit test name, autocoding and source inputs for the unit test, and (optionally) any
non-standard link dependencies.

**Note:** This is ONLY run when the build type is TESTING. Unit testing is restricted to this build type as fprime
          sets additional flags when building for unit tests.

Required variables (defined in calling scope):


- **UT_NAME:** (optional) executable name supplied. If not supplied, or passed in, then
  the <MODULE_NAME>_ut_exe will be used.

- **UT_SOURCE_FILES:** cmake list of UT source files. Place any "*Ai.xml", "*.c", "*.cpp"
  etc. files here. This list will be split into autocoder inputs or sources. These sources only apply to the unit
  test.

 **i.e.:**
```
set(UT_SOURCE_FILES
    MyComponentAi.xml
    SomeFile.cpp
    MyComponentImpl.cpp)
```

- **UT_MOD_DEPS:** (optional) cmake list of extra link dependencies. This is optional, and only
  needed if non-standard link dependencies are used. If not set or supplied, only
  fprime detectable dependencies will be available. Link flags like "-lpthread"
  can be supplied here.

**i.e.:**
```
set(UT_MOD_DEPS
    Module1
    Module2
    -lpthread)
```

  **Note:** if desired, these fields may be supplied in-order as arguments to the function. Passing
            these as positional arguments overrides any specified in the parent scope.

  **Note:** UTs automatically depend on the module. In order to prevent this, explicitly pass in args
            to this module, excluding the module.

        e.g. register_fprime_ut("MY_SPECIAL_UT" "${SOME_SOURCE_FILE_LIST}" "") #No dependencies.

 **Note:** this is typically called after any other register calls in the module.

### Unit-Test Example ###

A standard unit test defines only UT_SOURCES. These sources have the test cpp files and the module
Ai.xml of the module being tested. This is used to generate the GTest and TesterBase files from this
Ai.xml. The other UT source files define the implementation of the test.

```
set(UT_SOURCE_FILES
  "${FPRIME_FRAMEWORK_PATH}/Svc/CmdDispatcher/CommandDispatcherComponentAi.xml"
  "${CMAKE_CURRENT_LIST_DIR}/test/ut/CommandDispatcherTester.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/test/ut/CommandDispatcherImplTester.cpp"
)
register_fprime_ut()
```

### Unit-Test Without GTest/TesterBase Example ###

Some unit tests run without the need for the autocoding the GTest and TesterBase files. This can be
done without specifying the Ai.xml file. Most of the time, this style requires specifying some module
dependencies.

```
set(UT_SOURCE_FILES
  "${FPRIME_FRAMEWORK_PATH}/Svc/CmdDispatcher/CommandDispatcherComponentAi.xml"
  "${CMAKE_CURRENT_LIST_DIR}/test/ut/CommandDispatcherTester.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/test/ut/CommandDispatcherImplTester.cpp"
)
set(UT_MOD_DEPS
  Os
)
register_fprime_ut()
```



## Function `register_fprime_target`:

Some custom targets require a multi-phase build process that is run for each module, and for the
deployment/executable that is being built. These must therefore register module-specific and
deployment specific instructions.

**Examples:**
- dict: build sub dictionaries for each module, and roll-up into a global deployment dictionary
- sloc: lines of code are counted per-module
- docs: documentation is also per-module

This function allows the user to register a file containing two functions `add_module_target`
and `add_global_target`. `add_global_target` adds a top-level target like `make dict` which will
then depend on every one of the targets created in `add_module_target`.

**TARGET_FILE_PATH:** path to file defining above functions

function(register_fprime_target TARGET_FILE_PATH)
    # Check for some problems moving forward
    if (NOT EXISTS ${TARGET_FILE_PATH})
        message(FATAL_ERROR "${TARGET_FILE_PATH} does not exist.")
        return()
    endif()
    # Update the global list of target files
    set(TMP "${FPRIME_TARGET_LIST}")
    list(APPEND TMP "${TARGET_FILE_PATH}")
    list(REMOVE_DUPLICATES TMP)
    SET(FPRIME_TARGET_LIST "${TMP}" CACHE INTERNAL "FPRIME_TARGET_LIST: custom fprime targets" FORCE)
    #Setup global target. Note: module targets found during module processing
    setup_global_target("${TARGET_FILE_PATH}")
endfunction(register_fprime_target)



##
