
#System Configuration and Initialization

This guide describes how Mynewt manages system configuration and initialization. It shows you how to 
tell Mynewt to use default or customized values to initialize packages that you develop or use to build a target. This guide:

* Assumes you have read the [Concepts](/os/get_started/vocabulary.md) section that describes the Mynewt 
package hierarchy and its use of the `pkg.yml` and `syscfg.yml` files.   
* Assumes you have read the [Newt Tool Theory of Operation](/newt/newt_operation.md) and are familiar with how newt determines 
package dependencies for your target build.
* Covers only the system initialization for hardware independent packages. It does not cover the Board Support Package (BSP) and other hardware dependent system initialization.  

Mynewt defines several configuration parameters in the `pkg.yml` and `syscfg.yml` files. The newt tool uses this information to: 

* Generate a system initialization function that calls all the package-specific system initialization functions. 
* Generate a system configuration header file that contains all the package configuration settings and values.
* Display the system configuration settings and values in the `newt target config` command.

The benefits with this approach include:

* Allows Mynewt developers to reuse other packages and easily change their configuration settings without updating source or header files when implementing new packages.
* Allows application developers to easily view the system configuration settings and values and determine the values to override for a target build.

<br>

###System Configuration Setting Definitions and Values 

A package can optionally:

* Define and expose the system configuration settings to allow other packages to override 
the default setting values. 
* Override the system configuration setting values defined by the packages that it depends on. 

You use the `defs` parameter in a `syscfg.yml` file to define the system configuration settings 
for a package. `defs` is a mapping (or associative array) of system configuration setting definitions. It 
has the following syntax:  
 
```no-highlight

syscfg.defs:
    PKGA_SYSCFG_NAME1:
       description:
       value:
       type:
       restrictions:
    PKGA_SYSCFG_NAME2:
       description:
       value:
       type:
       restrictions:

```

<br>

Each setting definition consists of the following key-value mapping:  

* A setting name for the key, such as `PKGA_SYSCFG_NAME1` in the syntax example above.
Note: A system configuration setting name must be unique.  The newt tool aborts the build 
when multiple packages define the same setting. 
* A mapping of fields for the value.  Each field itself is a key-value pair of attributes.  The field keys are `description`, `value`, `type`, and `restrictions`. They are described in 
following table:

<table style="width:90%", align="center">
<tr>
<th>Field</th>
<th>Description</th>
</tr>
<tr>
<td><code>description</code></td>
<td>Describes the usage for the setting. <b>This field is optional.</b></td>
<tr>
<td><code>value<code></td>
<td>Specifies the default value for the setting. <b>This field is required.</b> The value depends on the <code>type</code> that you specify and can be an empty string. 
<tr>
<td><code>type</code></td>
<td>Specifies the data type for the <code>value</code> field. <b>This field is optional.</b> You can specify one of three types:
<ul>
<li><code>raw</code> - The <code>value</code> data is uninterpreted. This is the default <code>type</code>.</li>
<li><code>task_priority</code> - Specifies a Mynewt task priority number.  The task priority number assigned to each setting must be unique and between 0 and 239.  <code>value</code> can be one of the following: 
<ul>
<li>A number between 0 and 239 - The task priority number to use for the setting.</li>
<li><code>any</code> - Specify <code>any</code> to have newt automatically assign a priority for the setting.  
newt alphabetically orders all system configuration settings of this type and assigns the next highest available 
task priority number to each setting. </li>
</ul>
</li>
<li><code>flash_owner</code> - Specifies a flash area. The <code>value</code> should be the name of a flash area 
defined in the BSP flash map for your target board. 
</li>
</ul>
</td>
</tr>
<tr>
<td><code>restrictions</code></td>
<td>Specifies a list of restrictions on the setting value. <b>This field is optional.</b> You can specify two formats:
<ul>
<li><code>$notnull</code> - Specifies that the setting cannot have the empty string for a value. It essentially means that an empty string is not a sensible value and a package must override it with an appropriate value. 
<br>
</li>
<li><code>expression</code> - Specifies a boolean expression of the form <code>[!]&ltrequired-setting>[if &ltbase-value>]</code>
<br>Examples:
<ul>
<li><code>restrictions: !LOG_FCB</code> - Can only enable this setting when <code>LOG_FCB</code> is false.
<li><code>restrictions: LOG_FCB if 0 </code> - Can only disable this setting when <code>LOG_FCB</code> is true.
</ul>
</li>
</ul>
</td>
</tr>
</table>

<br>

####Examples of configuration settings

**Example 1:** The following example is an excerpt from the `sys/log` package `syscfg.yml` file. It defines the 
`LOG_LEVEL` configuration setting to specify the log level and the `LOG_NEWTMGR` configuration setting to specify whether
to enable or disable the newtmgr logging feature.

```no-highlight

syscfg.defs:
    LOG_LEVEL:
        description: 'Log Level'
        value: 0
        type: raw

       ...       

    LOG_NEWTMGR: 
        description: 'Enables or disables newtmgr command tool logging'
        value: 0

```

<br>

**Example 2:** The following example is an excerpt from the `net/nimble/controller` package `syscfg.yml` file. It defines the `BLE_LL_PRIO` 
configuration setting with a `task_priority` type and assigns task priority 0 to the BLE link layer task.

```no-highlight

syscfg.defs:
    BLE_LL_PRIO:
        description: 'BLE link layer task priority'
        type: 'task_priority'
        value: 0

```

<br>

**Example 3:** The following example is an excerpt from the `fs/nffs` package `syscfg.yml` file. 

```no-highlight

syscfg.defs:
    NFFS_FLASH_AREA:
        description: 'The flash area to use for the Newtron Flash File System'
        type: flash_owner
        value:
        restrictions:
            - $notnull
```

It defines the `NFFS_FLASH_AREA` configuration setting with a `flash_owner` type indicating that a flash area needs to be specified for the Newtron Flash File System. The flash areas are typically defined by the BSP in its `bsp.yml` file. For example, the `bsp.yml` for nrf52dk board (`hw/bsp/nrf52dk/bsp.yml`)  defines an area named `FLASH_AREA_NFFS`:

```no-highlight
    FLASH_AREA_NFFS:
        user_id: 1
        device: 0
        offset: 0x0007d000
        size: 12kB
```
The `syscfg.yml` file for the same board (`hw/bsp/nrf52dk/syscfg.yml`) specifies that the above area be used for `NFFS_FLASH_AREA`.

```no-highlight
syscfg.vals:
    CONFIG_FCB_FLASH_AREA: FLASH_AREA_NFFS
    REBOOT_LOG_FLASH_AREA: FLASH_AREA_REBOOT_LOG
    NFFS_FLASH_AREA: FLASH_AREA_NFFS
    COREDUMP_FLASH_AREA: FLASH_AREA_IMAGE_1
```
   
Note that the `fs/nffs/syscfg.yml` file indicates that the `NFFS_FLASH_AREA` setting cannot be a null string; so a higher priority package must set a non-null value to it. That is exactly what the BSP package does. For more on priority of packages in setting values, see the next section.

<br>

###Overriding System Configuration Setting Values

A package may use the `vals` parameter in its `syscfg.yml` file to override the configuration values defined
by other packages.  This mechanism allows:

* Mynewt developers to implement a package and easily override the system configuration setting values 
   that are defined by the packages it depends on. 
* Application developers to easily and cleanly override default configuration settings in a single place and build a customized target. You can use the `newt target config <target-name>` command to check all the system configuration setting definitions and
   values in your target to determine the setting values to override. See [newt target](/newt/command_list/newt_target.md). 

`vals` specifies the mappings of system configuration setting name-value pairs as follows: 

```no-highlight

syscfg.vals:
    PKGA_SYSCFG_NAME1: VALUE1
    PKGA_SYSCFG_NAME2: VALUE2
              ...
    PKGN_SYSCFG_NAME1: VALUEN

```
Note: The newt tool ignores overrides of undefined system configuration settings.  

<br>

####Resolving Override Conflicts

The newt tool uses package priorities to determine whether a package can override a value and resolve conflicts when multiple packages override the same system configuration setting. The following rules apply:

* A package can only override the default values of system configuration settings that 
  are defined by lower priority packages.
* When packages with different priorities override the same system configuration setting value, newt uses 
   the value from the highest priority package.
* Packages of equal priority cannot override the same system configuration setting with different values. 
   newt aborts the build unless a higher priority package also overrides the value.

The following package types are listed from highest to lowest priority:

* Target
* App
* unittest - A target can include either an app or unit test package, but not both.
* BSP
* Lib - Includes all other system level packages such as os, lib, sdk, and compiler.

It is recommended that you override defaults at the target level instead of updating individual 
package `syscfg.yml` files.

<br>

####Examples of Overrides

**Example 4:** The following example is an excerpt from the `apps/slinky` package `syscfg.yml` file.  The application package overrides, 
in addition to other packages, the `sys/log` package system configuration settings defined in *Example 1*. It changes the LOG_NEWTMGR system configuration setting value from `0` to `1`.

```no-highlight

syscfg.vals:
    # Enable the shell task.
    SHELL_TASK: 1

       ...

    # Enable newtmgr commands.
    STATS_NEWTMGR: 1
    LOG_NEWTMGR: 1

```

**Example 5:** The following example are excerpts from the `hw/bsp/native` package `bsp.yml` and `syscfg.yml` files. 
The package defines the flash areas for the BSP flash map in the `bsp.yml` file, and sets the `NFFS_FLASH_AREA` 
configuration setting value to use the flash area named `FLASH_AREA_NFFS` in the `syscfg.yml` file.

```no-highlight

bsp.flash_map:
    areas:
        # System areas.
        FLASH_AREA_BOOTLOADER:
            device: 0
            offset: 0x00000000
            size: 16kB

             ...

        # User areas.
        FLASH_AREA_REBOOT_LOG:
            user_id: 0
            device: 0
            offset: 0x00004000
            size: 16kB
        FLASH_AREA_NFFS:
            user_id: 1
            device: 0
            offset: 0x00008000
            size: 32kB


syscfg.vals:
    NFFS_FLASH_AREA: FLASH_AREA_NFFS

```

<br>

###Generated syscfg.h

The newt tool processes all the package `syscfg.yml` files and generates the
`<target-path>/generated/include/syscfg/syscfg.h` include file with `#define` statements for each system configuration 
setting defined.  newt creates a `#define` for a setting name as follows: 

* Adds the prefix `MYNEWT_VAL_`.
* Replaces all occurrences of "/", "-", and " " in the setting name with "_".
* Converts all characters to upper case.

For example, the #define for `my-config-name` setting name  is `MYNEWT_VAL_MY_CONFIG_NAME`.

Newt groups the settings in `syscfg.h` by the packages that defined them. It also indicates the 
package that changed a system configuration setting value.  

**Note:** You only need to include `syscfg/syscfg.h` in your source files to access the `syscfg.h` file.  The newt tool sets the correct include path to build your target. 

Here is an excerpt from a sample `syscfg.h` file generated for an app/slinky target.  It lists 
the `sys/log` package definitions and also indicates that `app/slinky` changed the value 
for the `LOG_NEWTMGR` settings.  

```no-highlight

#ifndef H_MYNEWT_SYSCFG_
#define H_MYNEWT_SYSCFG_

     ...

/*** kernel/os */
#ifndef MYNEWT_VAL_MSYS_1_BLOCK_COUNT
#define MYNEWT_VAL_MSYS_1_BLOCK_COUNT (12)
#endif

#ifndef MYNEWT_VAL_MSYS_1_BLOCK_SIZE
#define MYNEWT_VAL_MSYS_1_BLOCK_SIZE (292)
#endif

     ...

/*** sys/log */

#ifndef MYNEWT_VAL_LOG_LEVEL
#define MYNEWT_VAL_LOG_LEVEL (0)
#endif

     ...

/* Overridden by apps/slinky (defined by sys/log) */
#ifndef MYNEWT_VAL_LOG_NEWTMGR
#define MYNEWT_VAL_LOG_NEWTMGR (1)
#endif

#endif
```

<br>

### System Initialization

An application's `main()` function must first call the Mynewt `sysinit()` function to 
initialize the software before it performs any other processing.
`sysinit()` calls the `sysinit_app()` function to perform system 
initialization for the packages in the target.  You can, optionally, specify an 
initialization function that `sysinit_app()` calls to initialize a package. 

A package init function must have the following prototype:

```no-highlight

void init_func_name(void)

```
Package init functions are called in stages to ensure that lower priority packages 
are initialized before higher priority packages.

You specify an init function in the `pkg.yml` file for a package as follows:

* Use the `init_function` parameter to specify an init function name. 

           pkg.init_function: pkg_init_func_name

      where `pkg_init_func_name` is the C function name of package init function. 

* Use the `init_stage` parameter to specify when to call the package init function.

           pkg.init_stage: stage_number

       where `stage_number` is a number that indicates when this init function is called relative to the other 
       package init functions.  Mynewt calls the package init functions in increasing stage number order
       and in alphabetic order of init function names for functions in the same stage.
       *Note:* The init function will be called at stage 0 if `pkg.init_stage` is not specified.
 
*Note:* You must include the `sysinit/sysinit.h` header file to access the `sysinit()` function.

<br>

#### Generated sysinit_app() Function

The newt tool processes the `init_function` and `init_stage` parameters in all the pkg.yml files for a target,
generates the `sysinit_app()` function in the `<target-path>/generated/src/<target-name>-sysinit_app.c` file, and 
includes the file in the build. Here is an example `sysinit_app()` function:

```no-highlight

/**
 * This file was generated by Apache Newt (incubating) version: 1.0.0-dev
 */

#if !SPLIT_LOADER

void os_init(void);
void split_app_init(void);
void os_pkg_init(void);
void imgmgr_module_init(void);
void nmgr_pkg_init(void);

      ...

void console_pkg_init(void);
void log_init(void);

      ...

void
sysinit_app(void)
{
    os_init();

    /*** Stage 0 */
    /* 0.0: kernel/os */
    os_pkg_init();
    /* 0.1: sys/console/full */
    console_pkg_init();

        ...

    /*** Stage 1 */
    /* 1.0: sys/log */
    log_init();

        ...

    /*** Stage 5 */
    /* 5.0: boot/split */
    split_app_init();
    /* 5.1: mgmt/imgmgr */
    imgmgr_module_init();
    /* 5.2: mgmt/newtmgr */
    nmgr_pkg_init();
        ...
}

#endif

```

<br>

###Conditional Configurations
You can use the system configuration setting values to conditionally specify parameter values
in `pkg.yml` and `syscfg.yml` files. The syntax is:

```no-highlight

parameter_name.PKGA_SYSCFG_NAME:
     parameter_value

```
This specifies that `parameter_value` is only set for `parameter_name` if the `PKGA_SYSCFG_NAME` configuration setting value 
is non-zero. Here is an example from the `libs/os` package `pkg.yml` file:

```
pkg.deps:
    - sys/sysinit
    - util/mem

pkg.deps.OS_CLI
    - sys/shell

```
This example specifies that the `os` package depends on the `sysinit` and `mem` packages, and also depends on the 
`shell` package when `OS_CLI` is enabled. 

The newt tool aborts the build when it detects circular conditional dependencies. 
