
Contents of this directory
---------------------------

This directory contains all the user applications built with using Orion's
build system. Each application has to be explicitly added into the
CMakeLists.txt file. Applications can be made by more than once C file and the
pattern used by the build system is to treat all C files matching the wildcard
`${APPNAME}*.c` as part of the application named `${APPNAME}`.


How to run other apps on Orion
--------------------------------

If you want to test your own application on Orion, you have two choices:

   1. Make it compatible with Orion's CMake build system and put a symlink
      to its root directory in `extra` [see `extra/README.md` for more].

   2. Compile it and link it *statically* using Orion's GCC libmusl toolchain
      [`toolchain3/x86_gcc_toolchain`]. After that, could can just drop the
      executable in sysroot/usr/bin. In order to just rebuild Orion's image,
      updating the sysroot, use the `make rebuild_img` command.
