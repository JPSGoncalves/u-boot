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

Booting
~~~~~~~
Currently, we rely on vendor FSBL (First Stage Boot Loader) to initialize the
clock and load OpenSBI and the u-boot image, then bootup from it.

To run u-boot.bin on top of FSBL, follow these steps:

1. Generate a compatible u-boot.bin using U-Boot with the Milk-V Duo S
   default configuration.

2. Use mainline OpenSBI with a newer version than 1.5 to generate fw_dynamic.
   Use FW_FDT_PATH to point to sg2000-milkv-duo-s.dtb device tree file while
   compiling fw_dynamic. Example:

.. code-block:: console

   cd open-sbi
   PLATFORM=generic FW_FDT_PATH=../u-boot/dts/upstream/src/riscv/sophgo/sg2000-milkv-duo-s.dtb make

This will generate build/platform/generic/firmware/fw_dynamic.bin.

3. Use the vendor-provided tool [1] to create a unified fip.bin file containing
   FSBL, OpenSBI, and U-Boot.
   Note that you will have to use the file cv181x.bin as the FSBL.

4. Place the generated fip.bin file into the FAT partition of the SD card.

5. Set the board's boot core selection switch to RISC-V.

6. Insert the SD card into the board and power it on.

The board will automatically execute the FSBL from the fip.bin file.
Subsequently, it will transition to OpenSBI, and finally, OpenSBI will invoke
U-Boot.

[1]: https://github.com/sophgo/fiptool

Sample boot log from Milk-V Duo S board
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code-block:: none

  U-Boot 2026.10-g3c39950264fd (Oct 16 2026 - 19:45:23 +0000) milkv_duo_s

  DRAM:  512 MiB
  Core:  28 devices, 13 uclasses, devicetree: separate
  MMC:   mmc@4310000: 0, mmc@4320000: 2, mmc@4300000: 1
  Loading Environment from nowhere... OK
  In:    serial@4140000
  Out:   serial@4140000
  Err:   serial@4140000
  Net:
  Warning: ethernet@4070000 (eth0) using random MAC address - de:19:4d:5d:74:41
  eth0: ethernet@4070000
  Hit any key to stop autoboot: 0
  milkv_duo_s#
