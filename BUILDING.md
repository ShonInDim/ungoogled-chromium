# Building ungoogled-chromium

## IMPORTANT - Please read this section first

**Statuses of platform support**: Because platform support varies across stable versions, [this Wiki page tracks platform support for the current stable](//github.com/Eloston/ungoogled-chromium/wiki/statuses). *Please check the status before attempting a build or posting an issue*.

**Choosing branches**: The `master` branch contains stable code, and `develop` is for unstable code. Please do not use `develop` unless you know what you are doing.

## Contents

There are two major sections of this document:

* [Standard building instructions](#standard-building-instructions) contains standard building instructions for supported platforms.
* [Advanced building information](#advanced-building-information) - For users who are building on unsupported systems or want a rough overview of the building procedure.

## Standard building instructions

This section contains standard building instructions for supported platforms.

Contents:

* [Debian and its derivatives](#debian-and-its-derivatives)
* [Windows](#windows)
* [macOS](#macos)
* [Arch Linux](#arch-linux)
* [Other Linux distributions](#other-linux-distributions)

### Debian and its derivatives


These instructions will create `.deb` packages. It uses ungoogled-chromium's variation of Debian's `debian` directory.

The build should work on the CPU architectures `amd64`, `i386`, `arm64`, and `armhf`.

The final size of the sandbox with build artifacts is over 5 GB. On systems with enough RAM, it can be built entirely within `tmpfs` without swap memory.

#### Setting up the build environment

Install base requirements: `# apt install packaging-dev python3 ninja-build`

On Debian 9 (stretch), `stretch-backports` APT source is used to obtain LLVM 5.0.

#### Setting up the buildspace tree and packaging files

Procedure for Debian 9 (stretch):

```
mkdir -p buildspace/downloads # Alternatively, buildspace/ can be a symbolic link
./buildkit-launcher.py genbun debian_stretch
./buildkit-launcher.py getsrc
./buildkit-launcher.py subdom
./buildkit-launcher.py genpkg debian --flavor standard
```
TODO: Investigate using dpkg-source to build a source package

The buildspace tree can be relocated to another system for building if necessary.

#### Invoking build

```
cd buildspace/tree
# Use dpkg-checkbuilddeps (from dpkg-dev) or mk-build-deps (from devscripts) to check for additional packages.
dpkg-buildpackage -b -uc
```

Packages will appear under `buildspace/`.

#### Notes for Debian derivatives

Ubuntu 17.10 (artful): Same as Debian 9 except the `ubuntu_artful` base bundle is used.

Ubuntu 16.04 (xenial), Debian 8.0 (jessie), and other older versions: See [Other Linux distributions](#other-linux-distributions)

### Windows

**NOTE**: There is no official maintainer for this platform. If there is a problem, please submit a pull request or issue (after checking the status page in the Wiki first).

Google only supports [Windows 7 x64 or newer](https://chromium.googlesource.com/chromium/src/+/64.0.3282.168/docs/windows_build_instructions.md#system-requirements). These instructions are tested on Windows 10 Home x64.

NOTE: The default configuration will build 64-bit binaries for maximum security (TODO: Link some explanation). This can be changed to 32-bit by changing `target_cpu` to `"x32"` (*with* quotes) in the user config bundle GN flags config file (default path is `buildspace/user_bundle/gn_flags.map`

#### Setting up the build environment

##### Setting up Visual Studio

[Follow the official Windows build instructions](https://chromium.googlesource.com/chromium/src/+/64.0.3282.168/docs/windows_build_instructions.md#visual-studio).

**IMPORTANT**: According to [a Chromium developer in Google Groups](https://groups.google.com/a/chromium.org/d/msg/chromium-dev/PsqFiJ-j5B4/9wO3wflWCQAJ), due to bugs in the 10.0.16299.15 SDK (that comes with Visual Studio 2017 as of Feburary 2018) *will not work* to build Chromium. The 10.0.15063 SDK must be downloaded and installed. This can be downloaded from the [Windows SDK Archive](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive).

When installing the SDK, the "Debugging Tools for Windows" feature must be enabled. Visual Studio 2017 does not enable this by default, so it has to be added in by selecting "Modify" on the SDK entry in "Add or remove programs".

##### Other build requirements

TODO: Fix the mess and move some information elsewhere.

1. Setup the following:

    * [LLVM](https://llvm.org/)
        * TODO: Document matching Google's clang version by downloading a snapshot from `CLANG_REVISION`. As of 64.0.3282.168, this is r318667, with download URL http://prereleases.llvm.org/win-snapshots/LLVM-6.0.0-r318667-win64.exe (from [LLVM Snapshot Builds](http://llvm.org/builds/))
        * The LLVM installer can be opened up with 7-Zip and unpacked into third_party/llvm-build/Release+Asserts in the buildspace tree.
        * NOTE: When LLVM 6.0.0 gets released, this should be used instead of the snapshot build.
        * NOTE: Using LLVM 5.0 with `is_clang=true` will result in errors like `cannot mangle this built-in __float128` from [Chromium Issue 780311](https://bugs.chromium.org/p/chromium/issues/detail?id=780311).
        * NOTE: Using Visual Studio exclusively with `is_clang=false` will cause multiple build failures due to hard-coded dependencies to `clang-cl.exe`
        * NOTE: Using `is_clang=false` while LLVM is located in the default location will cause errors due to includes between Visual C++ and LLVM being different (e.g. `(function signature): overriding virtual function has less restrictive exception specification than base class virtual member function (function signature)` with `third_party/openvr/src/src/json/json.h`)
    * Python 2 for scripts in Chromium
    * Python 3 for buildkit
    * [Ninja](https://ninja-build.org/)
    * [MSYS2](https://www.msys2.org/) *(recommended, not required)* - MSYS2 provides both git and [quilt](https://savannah.nongnu.org/projects/quilt/); the latter provides a convenient system to apply and manage patches.
        * After installing and updating all packages to the latest version, install git and quilt: `pacman -S git quilt`
    * [gperf from GNUWin32](http://gnuwin32.sourceforge.net/packages/gperf.htm)
    * [bison from GNUWin32](http://gnuwin32.sourceforge.net/packages/bison.htm)
        * Get the Binaries, Developer files, Sources, and Dependencies
        * **NOTE**: Make sure to place gperf and bison in a path without spaces, otherwise the build will fail.

2. Make sure the following are accessible in `PATH` (the PATH overrides feature can be used on the directories containing the actual executable):

    * Python 2 as `python`
    * Ninja as `ninja`

#### Setting up the buildspace tree and packaging files

Setting up via CMD:

```
mkdir buildspace\downloads
py buildkit-launcher.py genbun windows
py buildkit-launcher.py getsrc
py buildkit-launcher.py subdom
py buildkit-launcher.py genpkg windows
```

The buildspace tree can be relocated to another system for building if necessary.

#### Invoking build

1. Read and follow the instructions in the comments of `ungoogled_packaging\build.bat` (which resides in the buildspace tree)
2. Apply patches via MSYS2 shell in the buildspace tree:

    ```
    ./ungoogled_packaging/apply_patches.sh
    ```

3. In a new CMD instance, ensure the following are in `%PATH%`:

    * Python 2 as `python`
    * Ninja as `ninja`

    Then run `ungoogled_packaging\build.bat` in the buildspace tree.

TODO: Add packaging script

### macOS

**NOTE**: There is no official maintainer for this platform. If there is a problem, please submit a pull request or issue (after checking the status page in the Wiki first).

Tested on macOS 10.11-10.13

#### Additional Requirements

* Xcode 7-9
* Homebrew
* Perl (for creating a `.dmg` package)

#### Setting up the build environment

1. Install Quilt via Homebrew: `brew install quilt`
2. Install Ninja via Homebrew: `brew install ninja`
3. Install GNU coreutils (for `greadlink` in packaging script): `brew install coreutils`

#### Setting up the buildspace tree and packaging files

```
mkdir -p buildspace/downloads # Alternatively, buildspace/ can be a symbolic link
./buildkit-launcher.py genbun macos
./buildkit-launcher.py getsrc
./buildkit-launcher.py subdom
./buildkit-launcher.py genpkg macos
```

The buildspace tree can be relocated to another system for building if necessary.

#### Invoking build

```
cd buildspace/tree
chmod +x ungoogled_packaging/build.sh
./ungoogled_packaging/build.sh
```

A `.dmg` should appear in `buildspace/`

### Arch Linux

**NOTE**: There is no official maintainer for this platform. If there is a problem, please submit a pull request or issue (after checking the status page in the Wiki first).

There are two methods to build for Arch Linux outlined in the following sections.

#### Use PKGBUILD

These steps are for using a PKGBUILD to create a package. The PKGBUILD handles downloading, unpacking, building, and packaging (which uses a copy of buildkit internally).

Requirements: Python 3 is needed to generate the PKGBUILD. The PKGBUILD contains build dependency information.

Generate the PKGBUILD:

```
mkdir -p buildspace/tree
python3 buildkit-launcher.py genpkg -b archlinux archlinux
```

A PKGBUILD will be generated in `buildspace/tree/ungoogled_packaging`. It is a standalone file that can be relocated as necessary.

#### Create a compressed tar archive

These steps create an archive of the build outputs.

Requirements: Same as the build dependencies in the PKGBUILD (which can be seen in `resources/packaging/archlinux/PKGBUILD.in`).

The instructions are the same as [Other Linux distributions](#other-linux-distributions), except that the `archlinux` base bundle is used in the `genbun` command.

### Other Linux distributions

These are for building on Linux distributions that do not have support already. It builds without distribution-optimized flags and patches for maximum compatibility.

**NOTE**: There is no official maintainer for this platform. If there is a problem, please submit a pull request or issue (after checking the status page in the Wiki first).

#### Requirements

Debian-based: `# apt install packaging-dev python3 ninja-build`

* If not building a `.deb` package, replace `packaging-dev` with `quilt python clang llvm-dev`

Other:

* Python 3 (tested on 3.5) for buildkit
* Python 2 (tested on 2.7) for building GN and running other build-time scripts
* [Ninja](//ninja-build.org/) for running the build command
* [Quilt](//savannah.nongnu.org/projects/quilt/) for applying patches

#### Setting up the buildspace tree

First, setup the source tree:

```
mkdir -p buildspace/downloads
./buildkit-launcher.py genbun linux_simple
./buildkit-launcher.py subdom
```

#### Generating packaging files and invoking build

**Debian package**

Builds a `deb` package for any Debian-based system

```
./buildkit-launcher.py genpkg debian --flavor minimal
# The buildspace tree can be relocated to another system for building
cd buildspace/tree
# Use dpkg-checkbuilddeps (from dpkg-dev) or mk-build-deps (from devscripts) to check for additional packages.
# If necessary, change the dependencies in debian/control to accomodate your environment.
# If necessary, modify CLANG_BASE_PATH in debian/rules to change the LLVM and Clang installation path
# (which contains files like bin/clang++, include/llvm, etc.).
dpkg-buildpackage -b -uc
```
Packages will appear in `buildspace/`

**Archive**

Builds a compressed tar archive

```
./buildkit-launcher.py genpkg linux_simple
# The buildspace tree can be relocated to another system for building
cd buildspace/tree
# Use "export CLANG_BASE_PATH=/path/to/llvm_root" to set the LLVM and Clang installation path
# (which contains files like bin/clang++, include/llvm, etc.).
# If left unset, it defaults to /usr.
./ungoogled_packaging/build.sh
./ungoogled_packaging/package.sh
```
A compressed tar archive will appear in `buildspace/`

## Advanced building information

This section holds some information about building for unsupported systems and a rough building outline.

It is recommended to have an understanding of [DESIGN.md](DESIGN.md).

**Note for unsupported systems**: There is no set procedure for building ungoogled-chromium on unsupported systems. One should already be able to build Chromium for their system before attempting to include ungoogled-chromium changes. More information about the Chromium build procedure is on [the Chromium project website](https://www.chromium.org/Home). One should also understand [DESIGN.md](DESIGN.md) before including ungoogled-chromium changes.

### Essential building requirements

Here are the essential building requirements: 

* Python 3 (tested on 3.5) for running buildkit
* Python 2 (tested on 2.7) for building GN and running other scripts
* [Ninja](//ninja-build.org/) for running the build command
* [Quilt](//savannah.nongnu.org/projects/quilt/) is recommended for patch management.
    * [python-quilt](//github.com/bjoernricks/python-quilt) can be used as well.

Alternatively, [depot_tools](//www.chromium.org/developers/how-tos/install-depot-tools) can provide Python 2 and Ninja.

### Outline building procedure

This section has a rough outline of the entire building procedure.

In the following steps, `buildkit` represents the command to invoke buildkit's CLI.

Note that each buildkit command has a help page associated with it. Pass in `-h` or `--help` for more information.

1. Create `buildspace/` and `buildspace/downloads`. Other directories are created already.
2. Generate a user config bundle from a base config bundle: `buildkit genbun base_bundle`
3. Modify the user config bundle (default location is `buildspace/user_bundle`)
4. Create the buildspace tree: `buildkit getsrc`
5. Apply domain substitution: `buildkit subdom`
6. Generate packaging files into the buildspace tree: `buildkit genpkg package_type [options]`
7. Relocate the buildspace tree (with packaging files) to the proper machine for building.
8. Invoke the packaging scripts to build and package ungoogled-chromium.
