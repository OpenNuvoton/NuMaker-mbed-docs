# Using Eclipse CDT Standalone Debugger as GDB Frontend

This document gets you debugging GCC-compiled programs running on Nuvoton's Cortex-M series SoC in an easy-to-construct and use way.
Thanks to [Eclipse CDT Debugger supporting Standalone](https://github.com/eclipse-cdt/cdt/blob/main/StandaloneDebugger.md),
the approach based on it has the following advantages:
-   More friendly GUI instead of CLI GDB console.
    Users debug their programs on Eclipse CDT Standalone Debugger GUI and GDB runs as backend.
-   No need to generate or export IDE project extra.
    Most open source projects use CMake or else as build system and are not bound to IDE.
    In appropriate conditions, the built artifact ELF file is enough for enabling debugging,
    without the extra effort of converting original CMake projects to IDE projects.

## Hardware requirements

-   Target board based on Nuvoton's Cortex-M series SoC
-   [Nu-Link or Nu-Link2 adapter](https://github.com/OpenNuvoton/Nuvoton_Tools#nu-link2-pro-debugging-and-programming-adapter)

## Software requirements

Below list necessary tools and their versions against which Eclipse CDT Standalone Debugger can work successfully with this document:

-   Host operating system: Windows 10 64-bit

    > **_NOTE:_** Most users of Nuvoton's Cortex-M series SoC develop on Windows, so this document favors this environment.

-   [NuEclipse](https://github.com/OpenNuvoton/Nuvoton_Tools#numicro-software-development-tools) 1.02.02

-   OpenOCD: Attached with NuEclipse installation package

-   [Git](https://gitforwindows.org/) 2.37.3

    > **_NOTE:_** To enable Eclipse CDT Standalone Debugger, POSIX shell environment is needed.
    On Windows, this document shows using git-bash shell.

-   Cross compiler: [GNU Arm Embedded Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm) 10.3-2021.07

## Setting up Standalone Debugger environment

Before debugging programs, we need to set up the Standalone Debugger environment.
In the following, we will operate under get-bash shell environment.
First, open one git-bash shell.

### Installing Standalone Debugger launch script

Assume NuEclipse has installed in the path below. Hold it using the shell variable `NUECLIPSE_HOME`:
```
$ NUECLIPSE_HOME="/C/Program Files (x86)/Nuvoton Tools/NuEclipse/V1.02.021/NuEclipse"
```

Find the path to `install.sh` and hold it using the shell variable `ECLIPSE_CDT_SCRIPT_DIR`:
```
$ find "$NUECLIPSE_HOME" -name install.sh
/C/Program Files (x86)/Nuvoton Tools/NuEclipse/V1.02.021/NuEclipse/eclipse/plugins/org.eclipse.cdt.debug.application_1.2.200.202012191801/scripts/install.sh

$ ECLIPSE_CDT_SCRIPT_DIR="/C/Program Files (x86)/Nuvoton Tools/NuEclipse/V1.02.021/NuEclipse/eclipse/plugins/org.eclipse.cdt.debug.application_1.2.200.202012191801/scripts"
```

Install Standalone Debugger launch script which will get tailored for the environment:
```
$ cd "$ECLIPSE_CDT_SCRIPT_DIR"
$ ./install.sh
Installation complete
```

It will install under `$HOME/cdtdebugger`:
```
$ ls "$HOME/cdtdebugger"
cdtdebug.sh*  config.ini  dev.properties
```

### Patching Standalone Debugger launch script

The installed Standalone Debugger launch script needs patch due to path discrepancy between POSIX and Windows.
Use any text editor to open `cdtdebug.sh` and make the following modifications:

```sh
# Contributors:
#    Red Hat Inc. - initial API and implementation
#    Marc Khouzam (Ericsson) - Update for remote debugging support (bug 450080)
###############################################################################

# PATCH: Windows native program doesn't recognize $HOME path in POSIX form.
#        Convert to WIndows form, but with forward slash '/' instead of
#        backslash '\'.
HOME=`cygpath -m "$HOME"`

......

# PATCH: /bin/sh cannot treat '(' or ')' characters in ECLIPSE_HOME assignment.
#        Quote the assignment and also convert the path to Windows form.
ECLIPSE_HOME="/C/Program Files (x86)/Nuvoton Tools/NuEclipse/V1.02.021/NuEclipse/eclipse"
ECLIPSE_HOME=$(cygpath -m "$ECLIPSE_HOME")

```

### Configuring development environment in Standalone Debugger

In the following, we will enter Standalone Debugger and do configurations to match the development environment.

1.  Run `cdtdebug.sh` without giving arguments to enter Standalone Debugger anyway:
    ```
    $ cd "$HOME/cdtdebugger"
    $ ./cdtdebug.sh
    ```
    On **Debug New Executable** tab, click **Cancel** to not yet start debug session.

    > **_NOTE:_** The first launch of Standalone Debugger will create one default workspace in the path `$HOME/workspace-cdtdebug`.

1.  Go to GDB configuration tab through the menu **Window** > **Preferences** > **C/C++** > **Debug** > **GDB**.
    1.  Set **GDB debugger** to e.g.:
        ```
        C:/Program Files (x86)/GNU Arm Embedded Toolchain/10 2021.07/bin/arm-none-eabi-gdb.exe
        ```
    1.  Create one empty text file named `.gdbinit`. It can be used later.
        Set **GDB command file** to the path.

    Then click **Apply**.

## Debugging program

In the following, we will enable GDB remote debugging to debug program by:
1.  Launch OpenOCD as GDB server
1.  Launch Standalone Debugger as GDB client

### Launching OpenOCD as GDB server

Follow the steps below to launch OpenOCD as GDB server

1.  Open another git-bash shell

1.  Assume OpenOCD has installed in the path below. Hold it using the shell variable `OPENOCD_HOME`:
    ```
    $ OPENOCD_HOME="/C/Program Files (x86)/Nuvoton Tools/OpenOCD"
    ```

1.  Add OpenOCD executable into `PATH`:
    ```
    $ PATH="$OPENOCD_HOME/bin":$PATH
    ```

1.  Check OpenOCD has been ready:
    ```
    $ which openocd; openocd --version
    /C/Program Files (x86)/Nuvoton Tools/OpenOCD/bin/openocd
    Open On-Chip Debugger 0.10.0-dev-00476-gd1de37d6-dirty (2021-12-08-10:03)
    ```

1.  Check target board's debug interface has set up and USB has connected with host computer.

1.  Launch OpenOCD as GDB server
    ```
    $ openocd \
    -s "$OPENOCD_HOME" \
    -f 'scripts/interface/nulink.cfg' \
    -f 'scripts/target/numicroM4.cfg' \
    -c 'tcl_port 6333' \
    -c 'telnet_port 4444' \
    -c 'gdb_port 3333' \
    -c 'init' \
    -c 'targets' \
    -c 'reset halt'
    ```

    > **_NOTE:_** Adjust target configuration file to meet your target board.
    For example, *numicroM0.cfg*/*numicroM4.cfg*/*numicroM23.cfg*/*numicroM23_NS.cfg* for
    Cortex-M0/M4/M23/M23 NS respectively.

### Launching Standalone Debugger as GDB client

Follow the steps below to launch Standalone Debugger and
start debug session which will bring up GDB client.

> **_NOTE:_** Build flow is separate from this document.
Just assume debug information (`-g`) is available in the built artifact ELF file.

> **_NOTE:_** Though OpenOCD is involved with flashing target board, it is not covered here for simple.
Just assume the target board has flashed in platform-specific manner beforehand.

1.  Turn back to Standalone Debugger window

1.  Start debug session through the menu **File** > **Debug Remote Executable**.
    In **Debug Remote Executable** tab, fill the fields below and leave others blank:
    1.  Set **Binary** to e.g. *C:/foo.elf* to debug
    1.  Set **Host name or IP address** to *localhost*
    1.  Set **Port number** to *3333*

    > **_NOTE:_** On launching Standalone Debugger, you can straight start debug session by:
    ```
    $ ./cdtdebug.sh \
    -r localhost:3333 \
    -e "C:/foo.elf"
    ```

## Advanced tips

In the chapter, we address advanced tips with Standalone Debugger.

-   On starting debug session, Standalone Debugger defaults to *Run to main*.
    If you want to start debugging from reset handler, follow the steps after you have started debug session:
    1.  If target is running, stop it first through the menu **Run** > **Suspend**.
    1.  In Debugger Console tab, run:
        ```
        monitor reset halt
        ```
    1.  Through the menu **Run** > **Step Into** or **Step Over**,
    you can find program executes from reset handler point.

        > **_NOTE:_** Locate Debugger Console tab through the menu
        **Window** > **Show View** > **Debugger Console**.

-   Restart debug session without re-entering Standalone Debugger. This can be achieved following:
    1.  On Standalone Debugger, close current debug session through the menu **Run** > **Terminate**.
    1.  In the git-bash shell which launched OpenOCD as GDB server, press *Ctrl-C* to exit if not yet (may need multiple times).
    1.  Kept in the same shell, relaunch OpenOCD as GDB server.
    1.  Back on Standalone Debugger, restart debug session through the menu **Run** > **Debug**.
    
-   Debug multiple programs, for example, bootloader application.
    This can be achieved following:
    1.  In the `.gdbinit` file created above, add extra ELF files which are to debug like:
        ```
        add-symbol-file "C:/bar.elf"
        ```
    1.  Restart debug session

## Trouble-shooting

In the chapter, we address known issues with Standalone Debugger.

-   When you find GDB seems to misbehave, try to recover it following:
    1.  On Standalone Debugger, close current debug session through the menu **Run** > **Terminate**.
    1.  In the git-bash shell which launched OpenOCD as GDB server, press *Ctrl-C* to exit if not yet (may need multiple times).
    1.  Re-flash target board in platform-specific manner.
    1.  Back in the git-bash shell, relaunch OpenOCD as GDB server.
    1.  Back on Standalone Debugger, restart debug session through the menu **Run** > **Debug**.
