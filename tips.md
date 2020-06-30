# cmake tips 

Tips courtesy of [@jrmadsen,](https://github.com/jrmadsen) sorted and annotated by [@sleak-lbl](https://github.com/sleak-lbl)

I (@sleak-lbl) also used <https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/> and <https://cliutils.gitlab.io/modern-cmake/chapters/intro/dodonot.html> to improve my understanding of cmake, while sorting these.

## Think in terms of targets, not variables or directories 

- Use target-based semantics, e.g. `target_include_directories(...)` instead of `include_directories(...)` (which are scoped to the directory)
- Understand `PUBLIC` vs. `PRIVATE` vs. `INTERFACE` w.r.t targets
	- `PRIVATE` for things used internally to build a target
	- `INTERFACE` for external things that will use the target
	- avoid `PUBLIC` where possible
- Make use of `INTERFACE` libraries — these are not real libraries even though you might think they are because you do `add_library(mylibrary INTERFACE)`. Use these to hold build settings, e.g. compile-definitions, compile flags, libraries to link to, etc.

## Variables

- Avoid modifying most global variables that start with `CMAKE_` with a few exceptions. In particular, **do not modify** `CMAKE_C_FLAGS`, `CMAKE_CXX_FLAGS`, `CMAKE_Fortran_FLAGS`, etc.
- Understand the difference between `CACHE` variables and local variables. Local variables will override cache variables and when you run `cmake -DSOME_VARIABLE=...` you are creating a cache variable
- Understand that `set(<SOME_VARIABLE> <VALUE> CACHE <TYPE> "<DOC_STRING>")` **will only set the cache variable if it is not already in the cache** — if you want to modify any existing cache variable, you need to force it (generally not recommended) or use a local variable of the same name to override it
- Understand `CMAKE_PROJECT_NAME` vs. `PROJECT_NAME`, `CMAKE_SOURCE_DIR` vs. `PROJECT_SOURCE_DIR`  etc. — you can have projects within projects
	- **`CMAKE_` variables reference the topmost project**.
- Use generator expressions for things that need logic at compile-time instead of configure-time (i.e. cmake-time), e.g. `target_include_directories(myexe PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include>)`

## Project structure tips

- Enforce out-of-source builds via check that `CMAKE_SOURCE_DIR` does not equal `CMAKE_BINARY_DIR`
- Declare the project with the version and the languages, e.g. `project(MyProject LANGUAGES C CXX VERSION 0.0.1)`
- Export a `<PACKAGE_NAME>Config.cmake` or `<PACKAGE_NAME>-config.cmake` so that other packages can use `find_package(<PACKAGE_NAME>)` if it will be linked against

## Cmake installation, settings and locations

- Be sure to look in `ls $(dirname $(which cmake))/../share/cmake-*/Modules/` for any pre-existing `Find<PACKAGE_NAME>.cmake` and look for what interface libraries it exports, e.g. `find_package(MPI)` will produce a `MPI::MPI_C` target and `MPI::MPI_CXX` target which will include all the include directories, link-flags, etc. for MPI when you link to it — along that note, **don’t set the compilers to compiler-wrappers** like mpicc etc.
- There are many helper scripts in the `Modules/` folder of the cmake install
- Put custom cmake scripts in `cmake/Modules` and prefix that to `CMAKE_MODULE_PATH` at the beginning of the project, e.g. `set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules ${CMAKE_MODULE_PATH})` to ensure your scripts are found before other scripts of the same name in the scope of your project


## Cmake usage

- Set the minimum CMake version high and fail if not met, e.g. `cmake_minimum_required(VERSION 3.11 FATAL_ERROR)`
- Understand build-modes, e.g. `CMAKE_BUILD_TYPE=Debug` vs. `CMAKE_BUILD_TYPE=Release` — these will automatically add `-g` and `-O3 -DNDEBUG` etc. for the compiler. And realize **the default build mode is not set** so put this early on to set the default build mode to `Release`:
  ```
  if("${CMAKE_BUILD_TYPE}" STREQUAL "")
      set(CMAKE_BUILD_TYPE Release CACHE STRING "CMake build mode" FORCE)
  endif()
  ```
 - also, it is useful when debugging to `make VERBOSE=1` to see the verbose build commands. Also, be aware that generators allow CMake to generate other build systems instead of Makefiles. For example, I (@jrmadsen) almost always use `-G Ninja` instead of `-G "Unix Makefiles"` because `ninja` is less verbose and is essentially, `make -j` (i.e. parallel build)
- `ccmake` is a terminal-based GUI for CMake. You will see all cache variables there. `cmake-gui` is the Qt-based GUI. Also, CMake integrates with many IDEs.
- use environment variable `CMAKE_PREFIX_PATH` like `PATH/LD_LIBRARY_PATH/etc`. to point to the root directory of the package(s) that you want to find.


## Troubleshooting

- When the wrong package is found, e.g. the wrong installation of MPI, you have two choices: either delete `CMakeCache.txt` completely and start from scratch to undefine all the cache variables for MPI. I (@jrmadsen) prefer the latter and recommend this function to be added to `.bashrc`:
  ```
  cmake-undefine () 
  { 
      for i in $@;
      do
          _tmp=$(grep "^${i}" CMakeCache.txt | grep -v 'ADVANCED' | sed 's/:/ /g' | awk '{print $1}');
          for j in ${_tmp};
          do
              echo "-U${j}";
          done;
      done
  }
  ```
  
and then run something like:
```
cmake $(cmake-undefine MPI) /path/to/source/dir
```
and it will generate something like: 

```
cmake -UMPIEXEC_EXECUTABLE -UMPIEXEC_MAX_NUMPROCS -UMPIEXEC_NUMPROC_FLAG -UMPIEXEC_POSTFLAGS -UMPIEXEC_PREFLAGS -UMPI_CXX_ADDITIONAL_INCLUDE_DIRS -UMPI_CXX_COMPILER -UMPI_CXX_COMPILER_INCLUDE_DIRS -UMPI_CXX_COMPILE_DEFINITIONS -UMPI_CXX_COMPILE_OPTIONS -UMPI_CXX_HEADER_DIR -UMPI_CXX_LIB_NAMES -UMPI_CXX_LINK_FLAGS -UMPI_CXX_SKIP_MPICXX -UMPI_C_ADDITIONAL_INCLUDE_DIRS -UMPI_C_COMPILER -UMPI_C_COMPILER_INCLUDE_DIRS -UMPI_C_COMPILE_DEFINITIONS -UMPI_C_COMPILE_OPTIONS -UMPI_C_HEADER_DIR -UMPI_C_LIB_NAMES -UMPI_C_LINK_FLAGS -UMPI_HEADER -UMPI_mpi_LIBRARY -UMPI_mpicxx_LIBRARY -UMPI_pmpi_LIBRARY -UMPI_RESULT_CXX_test_mpi_MPICXX -UMPI_RESULT_CXX_test_mpi_normal -UMPI_RESULT_C_test_mpi_normal /path/to/source
```

