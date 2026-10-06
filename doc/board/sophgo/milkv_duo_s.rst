.. SPDX-License-Identifier: GPL-2.0+

Milk-V Duo S
============

SG2000 RISC-V SoC
-----------------
The SG2000 is a high-performance, low-power 64-bit RISC-V/ARM SoC from Sophgo.

Mainline support
----------------
Support for the following drivers is already enabled:

1. ns16550 UART Driver.
2. Synopsys Designware MSHC Driver.
3. Synopsys Designware Ethernet Controller.

Building
~~~~~~~~
1. Add the RISC-V toolchain to your PATH.
2. Setup ARCH & cross compilation environment variable:

.. code-block:: console

   export CROSS_COMPILE=<riscv64 toolchain prefix>
   cd <U-Boot-dir>
   make milkv_duo_s_defconfig
   make

This will generate u-boot.bin

Boot flow
~~~~~~~~~
The board boots from a FIP image, fip.bin, which contains the following
binaries:

- FSBL (First Stage Boot Loader): the vendor bootloader [1]. It serves the
  same purpose as U-Boot SPL, but U-Boot does not support SPL for this SoC
  yet, so the vendor FSBL is required.
- Coprocessor firmware: the vendor firmware (cvirtos.bin) or custom firmware
  for the C906L coprocessor. It is not needed to boot U-Boot, but it is
  required by fiptool.
- OpenSBI (fw_dynamic.bin), built with the board devicetree.
- U-Boot (u-boot.bin).

The boot flow is:

1. The boot ROM loads fip.bin from the FAT partition of the SD card and runs
   the FSBL.
2. The FSBL initializes the clocks and DRAM, starts the coprocessor firmware
   on the C906L, and loads OpenSBI and U-Boot into DRAM.
3. The FSBL jumps to OpenSBI, which runs in M-mode.
4. OpenSBI starts U-Boot in S-mode.

The fip.bin image is created with the Sophgo fiptool [2], which provides
prebuilt vendor binaries: data/fsbl/cv181x.bin, the FSBL for the CV181x/SG200x
SoCs, and data/cvirtos.bin, the coprocessor firmware. The fiptool also requires
the data/ddr_param.bin file.

Booting
~~~~~~~
Currently, we rely on the vendor FSBL to initialize the clock and DRAM, and
load OpenSBI and the u-boot image, then bootup from it.

To run u-boot.bin on top of FSBL, follow these steps:

1. Generate a compatible u-boot.bin using U-Boot with the Milk-V Duo S
   default configuration.

2. Use mainline OpenSBI v1.5+ (v1.5 is the first release supporting the SG2000
   [3]). Use FW_FDT_PATH to point to sg2000-milkv-duo-s.dtb device tree file
   while compiling fw_dynamic. Example:

.. code-block:: console

   cd <OpenSBI-dir>
   make PLATFORM=generic FW_FDT_PATH=<U-Boot-dir>/dts/upstream/src/riscv/sophgo/sg2000-milkv-duo-s.dtb

This will generate build/platform/generic/firmware/fw_dynamic.bin.

3. Use the vendor-provided tool [2] to create a unified fip.bin file containing
   FSBL, coprocessor firmware, OpenSBI, and U-Boot. Note that you will have to
   use the file cv181x.bin as the FSBL, and that the U-Boot load address must
   match CONFIG_TEXT_BASE (0x80200020). The tool also requires the
   data/ddr_param.bin file. Example:

.. code-block:: console

   cd <fiptool-dir>
   ./fiptool \
       --fsbl data/fsbl/cv181x.bin \
       --ddr_param data/ddr_param.bin \
       --rtos data/cvirtos.bin \
       --opensbi <OpenSBI-dir>/build/platform/generic/firmware/fw_dynamic.bin \
       --uboot <U-Boot-dir>/u-boot.bin \
       --text-base $((0x80200020))

4. Place the generated fip.bin file into the FAT partition of the SD card.

5. Set the board's boot core selection switch to RISC-V.

6. Insert the SD card into the board and power it on.

The board will automatically execute the FSBL from the fip.bin file.
Subsequently, it will transition to OpenSBI, and finally, OpenSBI will invoke
U-Boot.

[1]: https://github.com/sophgo/fsbl

[2]: https://github.com/sophgo/fiptool

[3]: https://github.com/riscv-software-src/opensbi/releases/tag/v1.5

Sample boot log from Milk-V Duo S board
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: none

  U-Boot 2026.10-g3c39950264fd (Oct 23 2026 - 21:28:58 -0300) milkv_duo_s

  DRAM:  512 MiB
  Core:  28 devices, 13 uclasses, devicetree: separate
  MMC:   mmc@4310000: 0, mmc@4320000: 2, mmc@4300000: 1
  Loading Environment from nowhere... OK
  In:    serial@4140000
  Out:   serial@4140000
  Err:   serial@4140000
  Net:
  Warning: ethernet@4070000 (eth0) using random MAC address - 4e:27:14:9f:7c:d6
  eth0: ethernet@4070000
  Hit any key to stop autoboot: 0
  milkv_duo_s#
