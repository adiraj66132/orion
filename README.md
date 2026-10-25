# Orion

Orion is an educational *monolithic* kernel designed to be Linux-compatible at
the binary level. It runs on i686 and RISC-V 64, and can also be built for
x86_64. Its small scale and simple design make it a practical playground for
kernel development while keeping the ability to run the *same* userspace bits on
Linux. Orion does not need a custom set of applications: it runs mainstream
Linux programs such as **BusyBox**, **Vim**, **TinyCC**, **MicroPython**, **Lua**,
and framebuffer applications such as **fbDOOM**.

Contents
----------------------------------------
* [Overview](#overview)
  - [What is Orion?](#what-is-orion)
    * [Future plans](#future-plans)
  - [What Orion is not](#what-orion-is-not)
    * [Orion vs Linux](#orion-vs-linux)
* [Features](#features)
  - [i686 support](#i686-support)
  - [riscv64 support](#riscv64-support)
  - [File systems](#file-systems)
  - [Processes and signals](#processes-and-signals)
  - [I/O](#io)
  - [Console](#console)
  - [Userspace applications](#userspace-applications)
* [Booting Orion](#booting-orion)
  - [Orion's bootloader](#orions-bootloader)
  - [Third-party bootloaders](#third-party-bootloaders)
    * [GRUB support](#grub-support)
* [Documentation and HOWTOs](#documentation-and-howtos)
  - [Building Orion](#building-orion)
  - [Testing Orion](#testing-orion)
  - [Debugging Orion](#debugging-orion)
    * [Orion's debug panel](#orions-debug-panel)
* [A comment about user experience](#a-comment-about-user-experience)
* [FAQ](#faq)
* [License and lineage](#license-and-lineage)

Overview
----------------------------------------

### What is Orion?

Orion is an educational *monolithic* kernel written from scratch. It aims to be
Linux-compatible at the binary level, so programs built for Linux can run
unmodified as long as they use the supported syscall and device interfaces. The
kernel currently runs on i686 and RISC-V 64, with an x86_64 build target used for
development and testing.

The project deliberately stays small. Most architecture-specific code is
isolated, which keeps the bulk of the kernel portable and allows it to be
compiled for the unit tests on the host. Orion has no custom libc and no custom
userspace: a `gcc`/`musl` cross-toolchain is enough to build programs for it.
The result is a kernel that is small enough to understand end to end, but real
enough to run a shell, edit files, compile code, and exercise POSIX interfaces.

#### Future plans

In the long term, Orion may be useful for **embedded systems** that require
deterministic behavior and low latency. It could fill part of the gap between
*Embedded Linux* and real-time operating systems such as *FreeRTOS* or *Zephyr*.
The kernel already runs on RISC-V 64, and an ARM port is a possible future step.
It may also be adapted for MMU-less CPUs. A small memory footprint has always
been a design point: the kernel can boot and run on a QEMU machine with just a
few megabytes of RAM today.

Networking and storage are also part of the long-term plan. Networking may
start with UDP and IP on a limited set of network cards. Storage support will
likely start with a small number of block devices and file systems, with FUSE
support considered later. A major milestone would be a complete networking and
storage stack on a specific SoC, but that depends on the ARM port and is not
currently scheduled.

### What Orion is not

* Orion is not an attempt to rewrite or replace the Linux kernel. It is a
  different kernel with *partial* Linux compatibility, used to take advantage of
  Linux programs and toolchains. Compatibility is also a correctness tool: if a
  program works on Linux, it should work the same way on Orion unless it uses an
  unimplemented feature. Orion will eventually grow its own features and may
  diverge where that makes sense for its goals.

* Orion is not a desktop operating system. It does not aim to run an X server or
  a full desktop environment. That would require a large amount of additional
  functionality and is outside the project's scope.

#### Orion vs Linux

Orion is fundamentally different from Linux because it does not target
multi-user servers or desktop machines. Linux is not large and complex because
of a poor implementation; it is large because it solves an enormous range of
problems. Orion chooses a smaller problem set and trades features for:

- simpler code
- a smaller binary
- deterministic behavior
- low latency
- easier development and testing
- additional robustness

Orion is still an educational project, but it is written with those goals in
mind and has a test infrastructure that aims to be rigorous: unit tests,
self-tests, system tests, and automated interactive tests all live in the same
repository and are integrated with the build system.

Features
----------------------------------------

Orion is a preemptable monolithic *NIX kernel implementing about a hundred Linux
syscalls. The kernel is not x86-centric even though x86 was its first target.
Almost everything architecture-specific is isolated, so most of the kernel can
also be compiled for the host and exercised by the unit tests.

#### i686 support

The kernel uses a fair amount of **legacy hardware**: the 8259 PICs for IRQs,
the 8254 PIT for the system timer, the 16550 UART for serial communication, the
8042 keyboard controller, the 8237 ISA DMA controller, and the Sound Blaster 16
sound card (QEMU only). It also supports newer hardware features such as SSE,
AVX, AVX2, PAT, the i686 `sysenter` instruction, PCI Express enumeration through
ECAM, and **ACPI** through ACPICA. ACPI is used for power-button events, reboot
and power-off, and battery information when the platform exposes it through ACPI
control methods.

##### Running on physical x86 hardware

Orion is intended to run on real hardware, not only virtual machines. It has
been tested on i686 machines, older x86_64 machines with BIOS-only firmware,
machines with UEFI and CSM, and pure-UEFI systems. The project's rule is simple:
if a feature cannot be tested on real hardware, it should not become a core
feature. Some drivers are developed and tested mainly under QEMU for their
educational value, but that is the exception rather than the rule. Reliability
on real machines is a core goal.

#### riscv64 support

Orion's RISC-V 64 support focuses on embedded applications. The kernel runs in
RV64 supervisor mode, uses SV39 three-level page tables, and supports floating
point in userspace.

##### riscv64 hardware peripheral support

RISC-V uses a device-tree-based driver model. Orion obtains hardware information
from the device tree passed by the bootloader, so the RISC-V kernel code does not
contain board-level descriptions. The system timer invokes the SBI monitor
(OpenSBI) through the RISC-V SBI interface instead of implementing a timer
driver in the kernel. The RTC is provided by a simple goldfish virtual RTC
driver, which works on the QEMU virtual platform.

Orion on RISC-V does not implement typical PC peripherals such as a keyboard:
input and output use serial ports. The common `ns16550` UART driver is
implemented. To manage the nested interrupt controllers found on embedded
chips, Orion implements a simplified framework similar to Linux IRQ domains,
with support for the common INTC and PLIC controllers.

##### Board support

RISC-V boards are selected with the `BOARD` build variable. Orion currently
supports the qemu-virt virtual board (the default) and the
[Sipeed Lichee RV Nano][licheerv]. QEMU-virt has framebuffer support enabled by
default. Framebuffer support on the Lichee RV Nano requires patching the
bootloader provided by Sipeed.

[licheerv]: https://wiki.sipeed.com/hardware/en/lichee/RV_Nano/1_intro.html

#### File systems

Orion has a simple but full-featured **ramfs** implementation, including soft
and hard links, file holes, and memory mapping. It also provides a minimalistic
**devfs**, read-only **FAT16/FAT32** support for the initrd (with memory-mapped
files), and a **sysfs** implementation that exposes ACPI's namespace, the list of
PCI(e) devices, and the kernel's compile-time configuration. A small **VFS**
layer ties the file systems together. There is no block-device support yet, so
all file systems are in memory.

#### Processes and signals

Orion uses the concept of a thread internally, but multithreading is not exposed
to userspace. Kernel threads exist. `fork()` and `vfork()` are implemented, and
forked processes use copy-on-write. `waitpid()` is fully implemented, including
process groups. POSIX signal support is partial: custom handlers work through
`rt_sigaction()`, but most `SA_*` flags are unsupported and handlers cannot
interrupt each other yet. `rt_sigprocmask()`, `rt_sigpending()`,
`rt_sigsuspend()`, and special signals such as `SIGSTOP`, `SIGCONT`, and
`SIGCHLD` work.

Orion also supports TLS through `set_thread_area()`, because `musl` requires it
even for single-threaded processes.

#### I/O

In addition to `read()` and `write()`, Orion supports vectored I/O through
`readv()` and `writev()`, non-blocking I/O, `select()`, and `poll()`. No program
has required `epoll` so far.

#### Console

Orion's console supports more than 90% of the Linux console's features. It uses
layers of abstraction so the same console implementation works in text mode and
framebuffer mode. The console was developed with the goal of making **Vim** work
smoothly, including syntax highlighting. Running a program as complex as Vim on
a small kernel is a useful demonstration that Orion can handle real userspace
software.

#### Userspace applications

Orion runs a fair amount of console software, including the **BusyBox** suite,
**Vim**, **TinyCC**, **MicroPython**, **Lua**, and framebuffer applications such
as **fbDOOM**. That is not the primary goal of the project, but it is a useful
compatibility test. See the [syscalls] document for the implemented interfaces.

[syscalls]: docs/syscalls.md

Booting Orion
----------------------------------------

### Orion's bootloader

Orion includes an interactive bootloader that works on legacy BIOS and UEFI
systems. It allows the user to select the video mode and the kernel file, and to
edit the kernel command line.

### Third-party bootloaders

Orion can be loaded by any bootloader that supports Multiboot 1.0. QEMU's
built-in Multiboot loader works well:

    qemu-system-i386 -kernel ./build/orion -initrd ./build/fatpart

That is also how the system tests boot the kernel. A shortcut is:

    ./build/run_multiboot_qemu

#### GRUB support

Orion can be booted with GRUB. Add an entry to `/etc/grub.d/40_custom` (or
another GRUB configuration file):

```
menuentry "Orion" {
    multiboot <PATH-TO-ORION-BUILD-DIR>/orion
    module --nounzip <PATH-TO-ORION-BUILD-DIR>/fatpart
    boot
}
```

Then run `update-grub` as root and reboot.

Documentation and HOWTOs
----------------------------------------

The main documentation lives in the `docs/` directory. Below is a quick starter
guide for the most common scenarios.

### Building Orion

The default build needs a Linux x86_64 host or Microsoft's WSL. The steps are:

* Enter the project's root directory.
* Build the toolchain once with: `./scripts/build_toolchain`
* Build the kernel and the bootable image with: `make`

The resulting image is `build/orion.img`. The easiest way to try it is:

    ./build/run_qemu

#### Running it on physical hardware

`build/orion.img` is bootable on physical machines, both with UEFI and with
legacy BIOS. Write the image to a USB stick with `dd` and boot from it.

#### Other configurations

See the [building] guide for the full set of build options, supported host
distributions, cross-compilation, and configuration targets.

[building]: docs/building.md

### Testing Orion

Orion has **unit tests**, **kernel self-tests**, **system tests** driven through
the syscall interface, and **automated interactive tests** that simulate real
user input through QEMU's monitor. They are all integrated with the build
system. Code-coverage support and HTML report generation are also available; see
the [coverage] guide. The repository's GitHub Actions workflows build and test
the supported configurations.

#### Running Orion's tests

The test runners need Python 3. For the self-tests and the classic system tests,
run:

    <BUILD_DIR>/st/run_all_tests -c

To run the unit tests:

* Install GoogleTest once with:
  `./scripts/build_toolchain -s host_gtest`
* Build the tests with: `make gtests`
* Run them with: `<BUILD_DIR>/gtests`

The `run_all_tests` runner can also select a test type, list tests, and filter by
name. Interactive tests use QEMU's QMP interface to send keyboard input and are
run with:

    <BUILD_DIR>/st/run_all_tests -T interactive -t any

The interactive tests require a specific build configuration. See the [testing]
document for the full set of options and constraints.

[testing]: docs/testing.md
[coverage]: docs/coverage.md

### Debugging Orion

QEMU's integrated GDB server makes it possible to debug Orion with GDB much as
you would debug a regular process. Context switches are the main complication,
because GDB follows one CPU context at a time. The basic steps are:

* (Optional) Prepare a debug build for a better debugging experience.
* Run the VM with: `./build/run_qemu`, and stop at the bootloader.
* In another terminal, run: `gdb ./build/orion_unstripped`.
* In GDB, connect to QEMU with: `target remote :1234`.
* Set a breakpoint, for example: `break kmain`.
* Type `c` to continue, then press ENTER in the bootloader to boot the kernel.

Orion ships with **GDB scripts** in `other/gdb_scripts`. They can list tasks,
show the handles opened by a process, and display other kernel state. See the
[debugging] document for details.

[debugging]: docs/debugging.md

#### Orion's debug panel

GDB is convenient while Orion runs in a VM, but not always available, for
example on real hardware. Some kernel information is also easier to expose
directly from the kernel than through GDB. `sysfs` is one option, but it is not
always the most convenient interface when *interaction* with the kernel is
needed. Orion therefore includes a **debug panel**, a TUI application inside the
kernel with debug information and tracing support for user processes. It started
as a Linux-Magic-SysRq-like facility and has grown into a practical debugging
tool. See the [debugging] document for more information.

A comment about user experience
----------------------------------

Orion distinguishes itself from many open-source projects by caring about the
**user experience** of the developer. It is not a low-level project that takes
hours to configure or requires a long list of host packages. The goal is that a
developer with basic Linux knowledge can build and test it without frustration.

The project includes a script that builds its own cross-toolchain on the major
Linux distributions and a CMake-based build system. The build produces an image
ready for QEMU or a USB stick. Scripts are included for running QEMU in several
configurations: BIOS boot, UEFI boot, and direct Multiboot with QEMU's `-kernel`
option. To some degree this resembles what Buildroot does for Linux, but much
simpler and focused on one kernel.

#### Motivation

A good developer experience is part of the project, not an afterthought. Even
experienced engineers should be able to build and run Orion without reading a
long setup guide. The same applies to students and junior developers who are
curious about kernel development and want to write a small program or add a
`printk()` to their fork. If the barrier to entry is low, more people can
experiment with the kernel and eventually contribute.

Some parts of the kernel are necessarily complex, but building it and running
its tests should be straightforward.

FAQ
----------------------------------------

**Why does Orion not have the feature or abstraction XYZ that other kernels
have?**

Orion implements what it needs to run real programs and to remain
understandable. Features are added when they solve a concrete problem, not to
match the architecture of larger kernels. The project favors a small, direct
implementation over generality that has no user yet.

**Can I build other projects on top of Orion?**

Yes, within the limits of the implemented Linux interfaces. Orion runs binaries
produced by a Linux `gcc`/`musl` toolchain, so existing programs can be compiled
for it without a custom SDK. The available syscalls and device interfaces are
listed in the [syscalls] document.

**Why does Orion support FAT32?**

The initrd is a FAT partition. Read-only FAT16/FAT32 support lets the kernel
mount it without a block-device driver or a custom image format. It also makes
the image easy to inspect and modify from the host.

**Why is the initrd kept mounted?**

The initrd contains the userspace programs and data used by the tests and the
interactive environment. Keeping it mounted avoids copying everything into
ramfs at boot and makes the running system match the image on disk.

**Why does Orion use three spaces for indentation?**

It is the project's coding style. It is consistent across the kernel, and the
repository includes `.clang-format` and style-checking tools.

**Can a lightweight X server run on Orion today?**

No. Orion does not target desktop use, and the graphics and input support needed
for an X server are outside its current scope. Framebuffer console applications
are supported.

**Does Orion aim to replace Linux?**

No. Orion is an educational kernel with partial Linux compatibility. It targets a
smaller problem space where a compact, deterministic kernel is useful.

License and lineage
----------------------------------------

Orion is distributed under the BSD 2-Clause License. See [LICENSE](LICENSE) for
the full text and [NOTICE](NOTICE) for third-party attributions.

Orion started from the Tilck kernel. The original BSD-2-Clause copyright notices
and the third-party notices are preserved in this repository.
