# Getting started with TF-M for Nuvoton platform

This document gets you started with [Trusted Firmware M](https://www.trustedfirmware.org/projects/tf-m/) (TF-M for short) for Nuvoton's platform.
It walks you through:

-   Constructing one reference TF-M build environment on Windows
-   Building first TF-M for Nuvoton's platform
-   Customizing TF-M code and updating into [Mbed OS](https://github.com/ARMmbed/mbed-os/) for Nuvoton's platform

## Hardware requirements

-   [NuMaker-M2354 board](https://os.mbed.com/platforms/NUMAKER-M2354/)

Currently, Nuvoton supports two TF-M platforms: M2351 and M2354.
Only M2354 enables TF-M/Mbed integration due to memory limitation.
In the following, M2354 is chosen for demo.

## Software requirements

To build TF-M, a POSIX-like build environment is needed.
In the following, Windows/MSYS2 is chosen as reference for its easy setup.
Below list necessary tools and their versions against which TF-M can build successfully with this document:

> **_NOTE:_** For detailed TF-M software requirements, navigate [TF-M](https://www.trustedfirmware.org/projects/tf-m/).
Then go through **DOCS** → **Getting Started Guides** → **Software requirements**.

> **_NOTE:_** **Native Windows** emphasizes this tool is installed through normal Windows installer. **DO NOT** use MSYS2 Pacman version.

> **_NOTE:_** **MSYS2 Pacman** means this tool is installed through MSYS2 Pacman package manager. Their native Windows versions may not be easy to find.

-   Host operating system: Windows 10 64-bit

-   [MSYS2](https://www.msys2.org/) 20210604
    > **_NOTE:_** Running MSYS2 installer from **Windows File Browser**, if MSYS2 install window fails to pop up, try running this installer from **Windows Command Prompt** instead.

-   [Git](https://gitforwindows.org/) 2.30.1 (Native Windows)
    > **_NOTE:_** If you have `.gitconfig` in `C:/Users/<WINDOWS-USER>`, also copy it to `C:/msys64/home/<WINDOWS-USER>` for consistent behavior.
    Thie is because Git, when running in MSYS shell, sees `C:/msys64/home/<WINDOWS-USER>` instead of `C:/Users/<WINDOWS-USER>` as home directory.

-   [CMake](https://cmake.org/) 3.19.3 (Native Windows)

-   [Python3](https://www.python.org/) 3.8.7 (Native Windows)
    -   Python3 packages: Listed in `trusted-firmware-m/tools/requirements.txt`

-   Cross compiler: [GNU Arm Embedded Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm) 9-2020-q2-update 9.3.1 (Native Windows) or
    Arm Compiler 6.12  (Native Windows)
    > **_NOTE:_** **GNU Arm Embedded Toolchain 10-2020-q4-major** built code **FAILS** to run. Avoid this toolchain version. Check [its bug report](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99157).

-   GNU make (MSYS2 Pacman)

-   SRecord (MSYS2 Pacman)

-   Host compiler: GCC (MSYS2 Pacman)
    > **_NOTE:_** This is needed for building [PSA compliance test](https://github.com/ARM-software/psa-arch-tests).

### Configuring proxy for the tools

If your network is behind the firewall, configure proxy for the tools which need to communicate with outside.
If not, just skip this step.

Edit the lines below to meet your environment, and add them at the end of `C:/msys64/home/<WINDOWS-USER>/.bashrc`:
```bash
# Configure proxy for MSYS2 Pacman
export http_proxy=http://<USER>:<PASSWORD>@<PROXY-SERVER>:<PORT>/
export https_proxy=$http_proxy
export ftp_proxy=$http_proxy
export rsync_proxy=$http_proxy

# Configuring proxy for Python3 pip
#
# Luckily, Python3 pip also honors the environmental variables http_proxy/https_proxy, which have been set appropriately above.
#

# Configuring proxy for Git
#
# Luckily, Git also honors the environmental variables http_proxy/https_proxy, which have been set appropriately above.
#
```

### Configuring paths for native Windows tools

MSYS2 strips the environmental variable `PATH` for clean POSIX-like environment.
We need to add paths of the tools into MSYS2 shell manually.

Edit the lines below to meet your environment, and add them at the end of `C:/msys64/home/<WINDOWS-USER>/.bashrc`:
```bash
# Arm Compiler 6
PATH="/C/Keil_v5/ARM/ARMCLANG/bin":$PATH

# GNU Arm Embedded Toolchain
PATH="/C/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/bin":$PATH

# CMake
PATH="/C/Program Files/CMake/bin":$PATH

# Python3
PATH="/C/Python38":"/C/Python38/Scripts":$PATH

# Git
PATH="/C/Program Files/Git/bin":$PATH
```

### Installing MSYS2 Pacman package tools

(Re-)run **MSYS2 MinGW 64-bit** shell from **Windows Start** menu.
Install GNU make, SRecord, and x86_64 GCC through MSYS2 Pacman package manager:
```
$ pacman -S --needed make mingw-w64-x86_64-srecord mingw-w64-x86_64-gcc
```

### Confirming all tools are ready

This step is to confirm all tools are ready.
If you are confident, just skip this step.
In MSYS2 shell, check if all tools are present and of right versions:
```
$ which git; git version
/C/Program Files/Git/mingw64/bin
git version 2.30.1.windows.1

$ which cmake; cmake --version
/C/Program Files/CMake/bin/cmake
cmake version 3.19.3

$ which python; python --version; which pip; pip --version
/C/Python38/python
Python 3.8.7
/C/Python38/Scripts/pip
pip 21.1.2 from c:\python38\lib\site-packages\pip (python 3.8)

$ which arm-none-eabi-gcc; arm-none-eabi-gcc --version
/C/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/bin/arm-none-eabi-gcc
arm-none-eabi-gcc.exe (GNU Arm Embedded Toolchain 9-2020-q2-update) 9.3.1 20200408 (release)
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ which armclang; armclang --version
/C/Keil_v5/ARM/ARMCLANG/bin/armclang
Product: MDK Plus 5.28
Component: ARM Compiler 6.12
Tool: armclang [5d624a00]

$ which make; make --version
/usr/bin/make
GNU Make 4.3

$ which srec_cat; srec_cat --version
/mingw64/bin/srec_cat
srec_cat version 1.64.D001

$ which gcc; gcc --version
/mingw64/bin/gcc
gcc.exe (Rev2, Built by MSYS2 project) 10.3.0
```

> **_NOTE:_** `git-apply` patch could fail with line ending. Recommend enabling automatic line ending conversion if not.

```
$ git config --global core.autocrlf true
```

## Building first TF-M

In MSYS2 shell, follow the instructions below to start building TF-M.

First, clone Nuvoton-forked TF-M source code and switch to the branch enabling Mbed integration:
```
$ git clone https://github.com/OpenNuvoton/trusted-firmware-m
$ cd trusted-firmware-m
$ git checkout -f nuvoton_mbed_m2354_tfm-1.3
```

Install necessary Python packages. This finalizes software requirements:
```
$ pip install -r tools/requirements.txt
```

### Example: Building TF-M + regression tests for M2354 using GCC

In the example, we build TF-M enabling regression tests for M2354, using GCC.

> **_NOTE:_** Run `rm -rdf cmake_build` first for fresh on rebuild.

> **_NOTE:_** For detailed TF-M build instructions, navigate [TF-M](https://www.trustedfirmware.org/projects/tf-m/).
Then go through **DOCS** → **Getting Started Guides** → **Build instructions**.

Build TF-M:
```
$ cmake -S . \
-B cmake_build \
-DTFM_PLATFORM=nuvoton/m2354 \
-DTFM_TOOLCHAIN_FILE=toolchain_GNUARM.cmake \
-DTFM_PSA_API=ON \
-DTFM_ISOLATION_LEVEL=2 \
-DTEST_S=ON \
-DTEST_NS=ON \
-G"Unix Makefiles"
```
```
$ cmake --build cmake_build -- install
```

We will get the following images in the directory `cmake_build/bin`:
-   bl2.bin: [MCUboot](https://github.com/mcu-tools/mcuboot) bootloader binary
-   tfm_s.bin: TF-M secure binary
-   tfm_ns.bin: TF-M non-secure binary
-   tfm_s_ns_signed.bin: `tfm_s.bin` and `tfm_ns.bin` signed together

Combine `cmake_build/bin/bl2.bin` and `cmake_build/bin/tfm_s_ns_signed.bin` into one, using SRecord:
```
$ srec_cat cmake_build/bin/bl2.bin -Binary -offset 0x0 \
cmake_build/bin/tfm_s_ns_signed.bin -Binary -offset 0x20000 \
-o cmake_build/bin/bl2_tfm_s_ns_signed.hex -Intel
```

Drag-n-drop `cmake_build/bin/bl2_tfm_s_ns_signed.hex` onto M2354 board to flash the image.

Configure terminal program with **115200/8-N-1**, and you would see console log with:
```
[INF] Starting bootloader
[WRN] Cannot upgrade: slots have non-compatible sectors
[INF] Bootloader chainload address offset: 0x20000
[INF] Jumping to the first image slot
[Sec Thread] Secure image initializing!
Booting TFM v1.3.0
Non-Secure system starting...

```

## Integrating with Mbed OS

In the section, we guide how to customize TF-M code and then update into Mbed for M2354.

First, clone Mbed OS source code and switch to the branch `master`:
```
$ git clone https://github.com/ARMmbed/mbed-os
$ cd mbed-os
$ git checkout -f master
```

Since [Mbed OS 6.1](https://os.mbed.com/blog/entry/Arm-Mbed-OS-61-released/), it changes its TF-M integration mechanism and pre-imports TF-M stuff of specific version, possibly latest release.
The Mbed pre-imported TF-M version can be found at: `mbed-os/platform/FEATURE_EXPERIMENTAL_API/FEATURE_PSA/TARGET_TFM/TARGET_TFM_LATEST/VERSION.txt`.
For customizing TF-M code for M2354, the patched TF-M branch and the Mbed pre-imported TF-M version must match.
For example, the patched TF-M branch `nuvoton_mbed_m2354_tfm-1.3` matches the Mbed pre-imported TF-M version `TF-Mv1.3.0`.
If you find no such branch in Nuvoton forked `trusted-firmware-m` repository, it means Nuvoton hasn't yet supported the TF-M version enabling integration with Mbed.

### Example: Customizing TF-M code for M2354

Currently, we support the following customization.

#### Defining Flash for TF-M/Mbed

To define memory spec of Flash for TF-M/Mbed, search/change the line in
`trusted-firmware-m/platform/ext/target/nuvoton/m2354/partition/flash_layout.h`:
```C
/* Max Flash size for TF-M + bootloader information */
#define FLASH_S_PARTITION_SIZE          (0x50000)
/* Max Flash size for Mbed + bootloader information */
#define FLASH_NS_PARTITION_SIZE         (0x90000)
```

M2354 has 1024KiB Flash in total, among which 128KiB have been allocated for bootloader code and ITS/PS storage.
896KiB are left for `FLASH_S_PARTITION_SIZE` and `FLASH_NS_PARTITION_SIZE`.
`FLASH_S_PARTITION_SIZE` for TF-M (secure) and `FLASH_NS_PARTITION_SIZE` for Mbed (non-secure) respectively.

> **_NOTE:_** `FLASH_S_PARTITION_SIZE` and `FLASH_NS_PARTITION_SIZE` must be sector size (2KiB)-aligned.

#### Defining SRAM for TF-M/Mbed

To define memory spec of SRAM for TF-M/Mbed, search/change the line in
`trusted-firmware-m/platform/ext/target/nuvoton/m2354/partition/region_defs.h`:
```C
/* Max SRAM size for TF-M */
#define S_DATA_SIZE     (80 * 1024)
/* Max SRAM size for Mbed = Total - Max SRAM size for TF-M */
#define NS_DATA_SIZE    (TOTAL_RAM_SIZE - S_DATA_SIZE)
```

`S_DATA_SIZE` for TF-M (secure) and `NS_DATA_SIZE` for Mbed (non-secure) respectively.

> **_NOTE:_** `S_DATA_SIZE` and `NS_DATA_SIZE` must be 16KiB-aligned required by M2354 Security Configuration Unit (SCU).

After finished changing TF-M code for M2354, we can rebuild TF-M for integration with Mbed OS:
```
$ cmake -S . \
-B cmake_build \
-DTFM_PLATFORM=nuvoton/m2354 \
-DTFM_TOOLCHAIN_FILE=toolchain_GNUARM.cmake \
-DTFM_PSA_API=ON \
-DTFM_ISOLATION_LEVEL=2 \
-G"Unix Makefiles"
```
```
$ cmake --build cmake_build -- install
```

> **_NOTE:_** For integration with Mbed, we don't enable TF-M regression tests (`TEST_S`/`TEST_NS`).

### Example: Updating TF-M exported stuff into Mbed for M2354

In the path `mbed-os/targets/TARGET_NUVOTON/TARGET_M2354/TARGET_TFM/TARGET_NU_M2354/COMPONENT_TFM_S_FW` (`MBED_M2354_TFM_IMPORT` for short),
the stuffs listed below are exported by TF-M and are to update into Mbed for M2354:
-   bl2.bin: MCUboot bootloader binary

-   tfm_s.bin: TF-M secure binary

-   s_veneers.o: TF-M secure gateway library

-   partition/: Flash layout for image signing and concatenating in post-build process

-   signing_key/: TF-M secure boot signing key

Below summarize the copy paths from TF-M into Mbed:

-   `trusted-firmware-m/cmake_build/install/outputs/NUVOTON/M2354/bl2.bin` → `MBED_M2354_TFM_IMPORT/bl2.bin`
-   `trusted-firmware-m/cmake_build/install/outputs/NUVOTON/M2354/tfm_s.bin` → `MBED_M2354_TFM_IMPORT/tfm_s.bin`
-   `trusted-firmware-m/cmake_build/install/export/tfm/lib/s_veneers.o` → `MBED_M2354_TFM_IMPORT/s_veneers.o`
-   `trusted-firmware-m/platform/ext/target/nuvoton/m2354/partition/flash_layout.h` → `MBED_M2354_TFM_IMPORT/partition/flash_layout.h`
-   `trusted-firmware-m/platform/ext/target/nuvoton/m2354/partition/region_defs.h` → `MBED_M2354_TFM_IMPORT/partition/region_defs.h`
-   `trusted-firmware-m/cmake_build/install/image_signing/layout_files/signing_layout_s_ns.o` → `MBED_M2354_TFM_IMPORT/partition/signing_layout_preprocessed.h`
-   `trusted-firmware-m/cmake_build/install/image_signing/keys/root-RSA-3072.pem` → `MBED_M2354_TFM_IMPORT/signing_key/nuvoton_m2354-root-rsa-3072.pem`
    → `mbed-os/tools/targets/nuvoton_m2354-root-rsa-3072.pem`

> **_NOTE:_** Some files are renamed.

> **_NOTE:_** The `mbed-os/tools/targets` path is for legacy Mbed CLI build tool.

Now, we can compile Mbed programs for M2354 normally, specifying target name `NU_M2354`.

## Further reading

In the section, advanced topics are addressed here.

### Example: Changing secure boot signing key for M2354

TF-M supports secure boot using [MCUboot](https://github.com/mcu-tools/mcuboot).
Default RSA key pairs are placed in `trusted-firmware-m/bl2/ext/mcuboot/root-RSA-*.pem`.
They are exclusively for development and cannot for production.
In the following, we guide how to change the signing key, assuming default MCUboot configurations for M2354 below:
-   MCUBOOT_SIGNATURE_TYPE: RSA
-   MCUBOOT_SIGNATURE_KEY_LEN: 3072
-   MCUBOOT_IMAGE_NUMBER: 1
-   MCUBOOT_HW_KEY: True


> **_NOTE:_** For detailed TF-M secure boot, navigate [TF-M](https://www.trustedfirmware.org/projects/tf-m/).
Then go through **DOCS** → **References** → **Secure boot**.

First generate a new RSA key pair.
This overrides `trusted-firmware-m/bl2/ext/mcuboot/root-RSA-3072.pem`:
```
$ cd trusted-firmware-m
$ imgtool keygen -t rsa-3072 -k bl2/ext/mcuboot/root-RSA-3072.pem
```

Dump public key hash from the generated key pair (for `MCUBOOT_HW_KEY`(`True`)):
```
$ openssl rsa -in bl2/ext/mcuboot/root-RSA-3072.pem -RSAPublicKey_out -outform DER |\
openssl dgst -sha256 -binary |\
hexdump -e '8/1 "0x%02x, " "\n"'
```
And then override specific code in `trusted-firmware-m/platform/ext/common/template/tfm_rotpk.c`:
```C
#elif (MCUBOOT_SIGN_RSA_LEN == 3072)
/* Hash of public key: bl2/ext/mcuboot/root-rsa-3072.pem */
uint8_t rotpk_hash_0[ROTPK_HASH_LEN] = {
    /* OVERRIDE ME */
};
```

> **_NOTE:_** The public key embedded into MCUboot is in PKCS1 format, so we specify `-RSAPublicKey_out`.

Dump public key from the generated key pair (for `MCUBOOT_HW_KEY`(`False`)):
```
$ imgtool getpub -k bl2/ext/mcuboot/root-RSA-3072.pem
```
And then override specific code in `trusted-firmware-m/bl2/ext/mcuboot/keys.c`:
```C
#elif MCUBOOT_SIGN_RSA_LEN == 3072
#define HAVE_KEYS
const unsigned char rsa_pub_key[] = {
    /* OVERRIDE ME */
};
const unsigned int rsa_pub_key_len = /* OVERRIDE ME */;
```

> **_NOTE:_** For `MCUBOOT_SIGNATURE_KEY_LEN`(`2048`), override the `trusted-firmware-m/bl2/ext/mcuboot/root-RSA-2048*.pem` file and the `MCUBOOT_SIGN_RSA_LEN == 2048` code instead.

> **_NOTE:_** For `MCUBOOT_IMAGE_NUMBER`(`2`), generating second RSA key pair, override the `trusted-firmware-m/bl2/ext/mcuboot/root-RSA-*_1.pem` file and the `MCUBOOT_IMAGE_NUMBER == 2` code extra.

> **_NOTE:_** Depending on `MCUBOOT_HW_KEY`, go dump public key hash or dump public key.

> **_NOTE:_** For the change to take effect, we need to re-build TF-M and re-integrate with Mbed. Check above examples.
