
What is the purpose of this directory
--------------------------------------

This directory has been created in order to facilitate the integration of
3rd-party projects with Orion's build system. If a 3rd-party project follows
the rules below, Orion's build system will automatically compile and copy
project's binaries in Orion's image. An excellent example project is the
`tfblib` project: https://github.com/vvaltchev/tfblib. It can be built
independently as any CMake project for Linux on any architecture but, when
a symlink (or a copy) of its root directory is put here in `extra`, it
integrates *smoothly* with Orion's build system and the binaries it
produces become visible from `Orion` in the `/usr/bin` directory.

Integration rules
-------------------

   1. The project must use `CMake`
   2. The project must be written in C/C++
   3. The project must produce exclusively statically linked binaries
   4. The project must work on same architecture `Orion` is built for
   5. The project must build correctly using a `libmusl` GCC toolchain
   6. The project must be careful with the syscalls used. Many `Linux` syscalls
      are not supported yet on `Orion`. See the supported syscalls in
      ../../docs/syscalls.md
            
   7. The project must *not* use the `project()` CMake command when it is being
      compiled in Orion. In order to detect that, use something like:

         ```CMake
         if (NOT "${CMAKE_PROJECT_NAME}" STREQUAL "orion")

            project(Cool3rdPartyProject C)

         else()

            message(STATUS "Building Cool3rdPartyProject as part of Orion")

         endif()
         ```

   8. At the end (preferably) of the root `CMakeLists.txt`, the 3rd project have
      to list the absolute path of all the binaries that have to be copied in
      Orion's image, using the `tilck_add_extra_app()` CMake macro. For example:

         ```CMake
         if (${CMAKE_PROJECT_NAME} STREQUAL "orion")
            tilck_add_extra_app(${CMAKE_CURRENT_BINARY_DIR}/SomeOutputDir/app1)
         endif()
         ```

      The binaries enlisted this way will be visible from `Orion` in `/usr/bin`.
