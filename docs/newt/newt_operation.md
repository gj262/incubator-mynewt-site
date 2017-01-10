## Newt Tool - Theory of Operations

Newt has a fairly smart package manager that can read a directory tree, build a dependency tree, and emit the right build artifacts.

### Building dependencies

Newt can read a directory tree, build a dependency tree, and emit the right build artifacts.  An example newt source tree is in incubator-mynewt-blinky/develop:

```hl_lines="7 12"
$ tree -L 3 
.
├── DISCLAIMER
├── LICENSE
├── NOTICE
├── README.md
├── apps
│   └── blinky
│       ├── pkg.yml
│       └── src
├── project.yml
└── targets
     ├── my_blinky_sim
     │   ├── pkg.yml
     │   └── target.yml
     └── unittest
         ├── pkg.yml
         └── target.yml

6 directories, 10 files
```

<br>

When Newt sees a directory tree that contains a "project.yml" file it knows that it is in the base directory of a project, and automatically builds a package tree. You can see that there are two essential package directories, "apps" and "targets." 

<br>

#### "apps" Package Directory

`apps` is where applications are stored, and applications are where the main() function is contained.  The base project directory comes with one simple app called `blinky` in the `apps` directory. The core repository `@apache-mynewt-core` comes with many additional sample apps in its `apps` directory. At the time of this writing, there are several example BLE apps, the boot app, slinky app for using newt manager protocol, and more in that directory.

```
@~/dev/myproj$ ls repos/apache-mynewt-core/apps/
blecent		bleprph_oic	bleuart		ffs2native	slinky_oic	test
blehci		bletest		boot		ocf_sample	spitest		timtest
bleprph		bletiny		fat2native	slinky		splitty
```

Along with the `targets` directory, `apps` represents the top-level of the build tree for the particular project, and define the dependencies and features for the rest of the system. Mynewt users and developers can add their own apps to the project's `apps` directory.   

The app definition is contained in a `pkg.yml` file. For example, blinky's `pkg.yml` file is:

```
$ more apps/blinky/pkg.yml
<snip>
pkg.name: apps/blinky
pkg.type: app
pkg.description: Basic example application which blinks an LED.
pkg.author: "Apache Mynewt <dev@mynewt.incubator.apache.org>"
pkg.homepage: "http://mynewt.apache.org/"
pkg.keywords:

pkg.deps:
    - "@apache-mynewt-core/kernel/os"
    - "@apache-mynewt-core/hw/hal"
    - "@apache-mynewt-core/sys/console/full"
```

<br>

This file says that the name of the package is apps/blinky, and it 
depends on kernel/os, hw/hal and sys/console/full packages.

**NOTE:** @apache-mynewt-core is a repository descriptor, and this will be 
covered in the "repository" section. 

<br>

#### "targets" Package Directory

`targets` is where targets are stored, and each target is a collection of parameters that must be passed to Newt in order to generate a reproducible build. Along with the `apps` directory, `targets` represents the top of the build tree. Any packages or parameters specified at the target level cascades down to all dependencies.

Most targets consist of:

* app: The application to build
* bsp: The board support package to combine with that application
* build_profile: Either debug or optimized.

The `my_blinky_sim` target that is included by default has the following settings:

```
$ newt target show
targets/my_blinky_sim
    app=apps/blinky
    bsp=@apache-mynewt-core/hw/bsp/native
    build_profile=debug
$ ls targets/my_blinky_sim/
pkg.yml		target.yml
```
There are helper functions to aid the developer specify parameters for a target. 

* **vals**: Displays all valid values for the specified parameter type (e.g. bsp for a target)
* **target show**: Displays the build artifacts for specified or all targets

In general, the three basic parameters of a target (`app`, `bsp`, and `build_profile`) are stored in the `target.yml` file in that target's build directory under `targets`. You will also see a `pkg.yml` file in the same directory. Since targets are packages, a `pkg.yml` is expected. It contains typical package descriptors, dependencies, and additional parameters such as the following:

* Cflags: Any additional compiler flags you might want to specify to the build
* Aflags: Any additional assembler flags you might want to specify to the build
* Lflags: Any additional linker flags you might want to specify to the build

<br>


### Resolving dependencies

When newt is told to build a project, it will:

* find the top-level project.yml file
* recurse the packages in the package tree, and build a list of all 
source packages

Newt then looks at the target that the user set, for example, blinky_sim:

```
$ more targets/my_blinky_sim/
pkg.yml     target.yml
$ more targets/my_blinky_sim/target.yml
### Target: targets/my_blinky_sim
target.app: "apps/blinky"
target.bsp: "@apache-mynewt-core/hw/bsp/native"
target.build_profile: "debug"
```

<br>

The target specifies two major things:

* Application (target.app): The application to build
* Board Support Package (target.bsp): The board support package to build 
along with that application.

Newt goes and builds the dependency tree specified by all the packages. While building this tree, it does a few other things:

- Any package that depends on another package, automatically gets the include directories from the package it includes.  Include directories in the
newt structure must always be prefixed by the package name. For example, libs/os has the following include tree and its include directory files contains the package name "os" before any header files.  This is so in order to avoid any header file conflicts.


```
$ tree
.
├── README.md
├── include
│   └── os
│       ├── arch
│       │   ├── cortex_m0
│       │   │   └── os
│       │   │       └── os_arch.h
│       │   ├── cortex_m4
│       │   │   └── os
│       │   │       └── os_arch.h
│       │   └── sim
│       │       └── os
│       │           └── os_arch.h
│       ├── endian.h
│       ├── os.h
│       ├── os_callout.h
│       ├── os_cfg.h
│       ├── os_eventq.h
│       ├── os_heap.h
│       ├── os_malloc.h
│       ├── os_mbuf.h
│       ├── os_mempool.h
│       ├── os_mutex.h
│       ├── os_sanity.h
│       ├── os_sched.h
│       ├── os_sem.h
│       ├── os_task.h
│       ├── os_test.h
│       ├── os_time.h
│       └── queue.h
├── pkg.yml
└── src
    ├── arch
<snip>

```

<br>

- API requirements are validated.  Packages can export APIs they 
implement, (i.e. pkg.api: hw-hal-impl), and other packages can require 
those APIs (i.e. pkg.req_api: hw-hal-impl).


In order to properly resolve all dependencies in the build system, Newt recursively processes the package dependencies until there are no new dependencies or features (because features can add dependencies.)  And it builds a big list of all the packages that need to be build.

Newt then goes through this package list, and builds every package into 
an archive file.

**NOTE:** The Newt tool generates compiler dependencies for all of these packages, and only rebuilds the packages whose dependencies have changed. Changes in package & project dependencies are also taken into account. It is smart, after all!

### Producing artifacts

Once Newt has built all the archive files, it then links the archive files together.  The linkerscript to use is specified by the board support package (BSP.)

NOTE: One common use of the "features" option above is to overwrite 
which linkerscript is used, based upon whether or not the BSP is being 
build for a raw image, bootable image or bootloader itself.

The newt tool places all of it's artifacts into the bin/ directory at 
the top-level of the project, prefixed by the target name being built, 
for example:

```
$ tree -L 4 bin/
bin/
└── my_blinky_sim
     ├── apps
     │   └── blinky
     │       ├── blinky.a
     │       ├── blinky.a.cmd
     │       ├── blinky.elf
     │       ├── blinky.elf.cmd
     │       ├── blinky.elf.dSYM
     │       ├── blinky.elf.lst
     │       ├── main.d
     │       ├── main.o
     │       └── main.o.cmd
     ├── hw
     │   ├── bsp
     │   │   └── native
     │   ├── hal
     │   │   ├── flash_map.d
     │   │   ├── flash_map.o
<snip>
```

<br>

As you can see, a number of files are generated:

- Archive File
- *.cmd: The command use to generate the object or archive file
- *.lst: The list file where symbols are located
- *.o The object files that get put into the archive file

### Download/Debug Support

Once a target has been build, there are a number of helper functions 
that work on the target.  These are:

* **load**     Download built target to board
* **debug**        Open debugger session to target
* **size**         Size of target components
* **create-image**  Add image header to target binary
* **run**  The equivalent of build, create-image, load, and debug on specified target

`load` and `debug` handles driving GDB and the system debugger.  These 
commands call out to scripts that are defined by the BSP.

```
$ more repos/apache-mynewt-core/hw/bsp/nrf52dk/nrf52dk_debug.sh
<snip>
. $CORE_PATH/hw/scripts/jlink.sh

FILE_NAME=$BIN_BASENAME.elf

if [ $# -gt 2 ]; then
    SPLIT_ELF_NAME=$3.elf
    # TODO -- this magic number 0x42000 is the location of the second image
    # slot. we should either get this from a flash map file or somehow learn
    # this from the image itself
    EXTRA_GDB_CMDS="add-symbol-file $SPLIT_ELF_NAME 0x8000 -readnow"
fi

JLINK_DEV="nRF52"

jlink_debug

```

The idea is that every BSP will add support for the debugger environment 
for that board.  That way common tools can be used across various development boards and kits.

