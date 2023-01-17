# Debug Mbed project overview

Arm Mbed has had [official debugging document](https://os.mbed.com/docs/mbed-os/v5.15/tutorials/debugging.html) for debugging Mbed program.
Read it first for common base knowledge.
This document is complement to it and focuses on Nuvoton targets.

## Common with debugger

This section lists topics on enabling debugger on Nuvoton targets.
All Nuvoton targets are shipped with Nu-Link/Nu-Link2 USB adapters, which enable programming and debugging.

-   Third-party IDE selection for development/debugging

    The table below lists popular IDEs for developing/debuging Mbed projects.
    Keil uVision is most supported and preferred.

    | IDE                   | SUPPORTED | PREFERRED | Notes
    ------------------------|-----------|-----------|-------------------------------------------
    | Arm Mbed Studio       | NO        | NO        |
    | Keil uVision          | YES       | YES       | Need separate [Keil Nu-Link debugger driver](https://github.com/OpenNuvoton/Nuvoton_Tools)
    | IAR EWARM             | YES       | NO        | Need separate [IAR Nu-Link debugger driver](https://github.com/OpenNuvoton/Nuvoton_Tools)
    | Eclipse               | NO        | NO        |
    | Visual Studio Code    | NO        | NO        |

-   Develop/debug on Mbed Online Compiler exported Keil uVision project

    Arm Mbed doesn't directly support debugging with Mbed Online Compiler.
    Export Keil uVision project first and then develop/debug on Keil uVision.
    Check [Export Keil uVision project](https://os.mbed.com/docs/mbed-os/v5.15/tools/debugging.html#exporting-your-project).

    **Note1**: Keil uVision must upgrade to 5.29 or above per verification.

    **Note2**: Check [Enable debugging on Nuvoton's targets with Keil uVision](Keil/DEBUG_KEIL.md) for needed setup.

-   Develop/debug on Mbed CLI exported Keil uVision project

    Arm Mbed doesn't directly support debugging with Mbed CLI.
    Export Keil uVision project first and then develop/debug on Keil uVision.
    Check [Export Keil uVision project](https://os.mbed.com/docs/mbed-os/v5.15/tools/debugging.html#exporting-your-project).

    **Note1**: Keil uVision must upgrade to 5.29 or above per verification.

    **Note2**: Check [Enable debugging on Nuvoton's targets with Keil uVision](Keil/DEBUG_KEIL.md) for needed setup.

-   [Build with Mbed CLI ARM/ARMC6/GCC_ARM and debug with Keil uVision](Keil/BUILD_MBEDCLI_DEBUG_KEIL.md)

    This approach builds your project with Mbed CLI with `ARM`/`ARMC6`/`GCC_ARM` toolchain, and debugs on compatible ELF file with Keil uVision.

    **Note1**: This approach is not officially supported by Arm Mbed and has drawbacks. Check it for details.

    **Note2**: Check [Enable debugging on Nuvoton's targets with Keil uVision](Keil/DEBUG_KEIL.md) for needed setup.

-   [Build with GCC and debug with Eclipse CDT Standalone Debugger](Eclipse/BUILD_GCC_DEBUG_CDTDEBUGGER.md)

    This approach builds your project with GCC toolchain, and debugs with Eclipse CDT Standalone Debugger.

    **Note**: This approach only requires GCC toolchain and is build system-agnostic.

-   [Enable post-mortem debugging with Keil uVision](Keil/POSTMORTEM_DEBUG_KEIL.md)

    This document instructs how to enable post-mortem debugging with Keil uVision on a crashed device.

-   [Enable debugging on Nuvoton's targets with Keil uVision](Keil/DEBUG_KEIL.md)

    This document instructs how to enable debugging with Keil uVision on Nuvoton's Mbed Enabled targets.

-   [Enable debugging on Nuvoton's targets with IAR EWARM](IAR/DEBUG_IAR.md)

    This document instructs how to to enable debugging with IAR EWARM on Nuvoton's Mbed Enabled targets.

## Common development/debugging issues

This section lists common development/debugging issues.

-   No output in terminal program

    Most Nuvoton targets default to 9600 baud rate, so configure your terminal program to **9600/8-N-1**.
    If your project has the **mbed_app.json** file and it has the following line, configure to **115200/8-N-1**.

    ```json
    "target_overrides": {
        "*": {
            "platform.stdio-baud-rate"              : 115200,
    ```

    For more information, check [Debugging using printf() statements](https://os.mbed.com/docs/mbed-os/v5.15/tutorials/debugging-using-printf-statements.html).

-   Crash on `printf(...)` call in interrupt context

    `printf(...)` has mutex guard code in it and mutex guard code cannot run in interrupt context.
    Check [Printf() from an interrupt context](https://os.mbed.com/docs/mbed-os/v5.15/tutorials/debugging-using-printf-statements.html).

-   Continuing above, need to call `printf(...)` like function in interrupt context for debugging purpose

    If `printf(...)` call in interrupt context is still necessary for debugging purpose, there's an approach to it:
    1.  `Use mbed_error_printf(...)` instead of `printf(...)`.
    1.  On `GCC_ARM`/`IAR`, call `printf(...)` or `mbed_error_printf(...)` at least once actively in thread context (e.g. in `main()`) before `mbed_error_printf(...)` call in interrupt context.
        This is to trigger stderr open sequence which involves mutex code. On `ARMC6`, this is unnecessary.
    1.  Make sure stdio serial is unbuffered, that is, don’t set `platform.stdio-buffered-serial` to `true`. Default is unbuffered.

    Code snippet:

    ```C++
    #include "mbed.h"

    InterruptIn button(SW2);

    void flip()
    {
        /* Call mbed_error_printf(...) instead of printf(...) in interrupt context */
        mbed_error_printf("Called in ISR\r\n");
    }

    void main()
    {
        button.rise(&flip);

        /* Call printf(...) or mbed_error_printf(...) in thread context at least once before the mbed_error_printf(...) call in interrupt context */
        mbed_error_printf("Called in main\r\n");

        while (true);
    }
    ```

    **Note1**: Use the trick for debugging purpose only, not for production.

    **Note2**: The trick is verified on Mbed OS 5.14.0, 5.15.3, and 6.0.0 beta-1.

    **Note3**: The trick is not officially anounced by Arm Mbed, so it isn't guaranteed to work across Mbed OS versions.

-   Faulted PC doesn't reflect fault right

    This may be caused by imprecise bus fault. Check [Debugging imprecise bus faults](https://os.mbed.com/docs/mbed-os/v5.15/tutorials/analyzing-mbed-os-crash-dump.html) for its resolution.
