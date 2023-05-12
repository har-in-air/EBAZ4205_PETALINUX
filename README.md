<h1> Petalinux 2022.2 for EBAZ4205 board with Zynq 7010 </h1>

**Environment** 
* Ubuntu 22.04LTS on amdx64 machine
* [Xilinx Unified Installer 2022.2 SFD](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools.html)
   * Select Vitis - this will install Vitis and Vivado Design suite
   * Run again and select Petalinux (install in the same directory) 
   * [Install cable drivers](https://support.xilinx.com/s/article/66440?language=en_US)


**Credits**
   
* [Petalinux 101 - Getting Started Quickly - Youtube](https://www.youtube.com/watch?v=k03r2Ud42jY)
* [Embedded Linux + FPGA/SoC (Zynq Part 5) - Phil's Lab - Youtube](https://www.youtube.com/watch?v=OfozFBfvWeY)
* [Local sstate-cache and download mirrors](https://www.xilinx.com/content/dam/xilinx/support/download/plnx/sstate_rel_2022.2_README.txt)
* https://support.xilinx.com/s/question/0D52E00006hpRM7SAM/cant-boot-petalinux-from-sd?language=en_US
* https://support.xilinx.com/s/question/0D52E00006hpPmhSAE/petalinux-20192-rootfs-in-sd?language=en_US
* 

<h1> EBAZ4205 hardware modifications </h1>


1. Installed microSD socket (came with the board, but not soldered)
1. Installed 25MHz crystal Y3 and 22pF caps for Ethernet PHY. R1485 not populated.
2. Installed 25MHz crystal oscillator X5 and associated passive R,C,L components for PL clock, connected to the Zynq 7010 pin N18.
3. Installed SPDT switch for selecting boot via SD card or Nand Flash.
For SD boot, Zynq pin U12-IO0_0 is connected to GND via R2584. For SD boot, pin U12-IO0_0U12-IO0_0 is connected to VCC via R2577. When SD boot is selected and the SD card is not detected on power-on, falls back to JTAG.
4. Installed tactile push button switch S3 plus associated R, C components. 
5. Installed diode SS810 (D24) to supply power from the ATX connector. The board documentation suggests 12V, but that was originally required for the cooling fans in the original application (bitmining). I use a standard 5V 2A adapter to supply power.


<h1> Create project using Vivado generated hardware description file (.xsa)</h1>

Set environment variables for Petalinux paths. 

```
$ cd <petaLinux_tool_install_dir>
$ source ./settings.sh
```

```
$ cd <petalinux_workspace_dir>
$ petalinux-create --type project --template zynq --name petalinux_sd

$ cd petalinux_sd
```

Specify the exported hardware description file from the Vivado project

```
$ petalinux-config  --get-hw-description=/home/nair/fpga/ebit/ebaz4205/vivado/ebit_z7010_top_wrapper.xsa
```

In the pop-up menuconfig dialog :
 		
* Check hardware components (DDR, Enet, UART, SDIO) are present

* Image Packaging Configuration
 	* For SD boot : change Root filesystem type from `RAM initrd` to `ext4 (sd/eMMC/SATA/USB)` 
 	* Disable Copy final images to tftpboot
 
* Yocto settings to use pre-downloaded archives during the build process instead of downloading from network
   * change sstate archive from network site `http://xxx` to local directory `file:///home/nair/installs/fpga/pl_sstate/arm/`
 		
   * change pre-mirror url from network site `http://xxx` to local directory `file:///home/nair/installs/fpga/pl_downloads/downloads/`
		

	* For SD boot : Use EXT4 rootfs during boot

	*Method A: PetaLinux config*

	Disable `DTG settings -> Kernel Bootargs -> generate boot args automatically`
   
   Update User Set Kernel Bootargs to 
   ```
   earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=8M 
   ```

	*Method B: device tree*

	Update in system-user.dtsi
	Add chosen node in root in addition to the previous changes to this file.
   ```
   / {
      chosen {
         bootargs = "earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=512M";
      };
   };
   ```


<h1> Configure the kernel </h1>

```
$ petalinux-config -c kernel
```

Nothing to change in menuconfig dialog, exit and save

<h1> Configure u-boot </h1>

```
$ petalinux-config -c u-boot
```

**Boot Media**

For SD boot, enable booting from SD Card, disable other options. 

For initrd RAM boot, leave all unchecked.


<h1> Configure Root file system </h1>

```
$ petalinux-config -c rootfs
```

User packages : add gpio-demo and peekpoke
	 	
<h1> Build </h1>

```
$ petalinux-build
```

<h1> JTAG download and boot using initrd RAM </h1>

**First verify JTAG connection to target**

I am using a  Xilinx Platform Cable USB II  JTAG adapter (DLC10).

Open Vivado project, open Hardware Manager, open target, connect and verify
that you can see the Zynq target. Then disconnect.


```
$ petalinux-boot --jtag --kernel
```

This will download to and boot from DDR memory

```
system.bit
zynq_fsbl.elf
system.dtb at 0x00100000
u-boot.elf
uImage at 0x00200000
rootfs.cpio.gz.u-boot at 0x04000000
boot.scr at 0x03000000
```

after booting, 

```
login : petalinux
set the password
```

<h1>Boot from SD card </h1>

<h2> Create the BOOT.BIN file </h2>


```
$ cd images/linux
$ petalinux-package --boot --fsbl zynq_fsbl.elf --fpga system.bit --u-boot u-boot.elf -o BOOT.BIN --force
```

This is the output 

```
[INFO] Sourcing buildtools
INFO: Getting system flash information...
INFO: File in BOOT BIN: "/home/nair/fpga/ebit/ebaz4205/petalinux_ram/images/linux/zynq_fsbl.elf"
INFO: File in BOOT BIN: "/home/nair/fpga/ebit/ebaz4205/petalinux_ram/images/linux/system.bit"
INFO: File in BOOT BIN: "/home/nair/fpga/ebit/ebaz4205/petalinux_ram/images/linux/u-boot.elf"
INFO: File in BOOT BIN: "/home/nair/fpga/ebit/ebaz4205/petalinux_ram/images/linux/system.dtb"
INFO: Generating zynq binary package BOOT.BIN...

****** Xilinx Bootgen v2022.2
  **** Build date : Sep 26 2022-06:24:42
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

[WARNING]: Partition zynq_fsbl.elf.0 range is overlapped with partition system.bit.0 memory range
[WARNING]: Partition system.bit.0 range is overlapped with partition system.dtb.0 memory range

[INFO]   : Bootimage generated successfully

INFO: Binary is ready.
```

The warning about overlap can be ignored, both files are being copied to the same memory partition, but are not actually overlapping in destination memory address ranges.

**Check layout of BOOT.BIN**

```
$ bootgen -arch zynq -read BOOT.BIN
```


```
****** Xilinx Bootgen v2022.2
  **** Build date : Feb  7 2023-10:06:59
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

--------------------------------------------------------------------------------
   BOOT HEADER
--------------------------------------------------------------------------------
        boot_vectors (0x00) : 0xeafffffeeafffffeeafffffeeafffffeeafffffeeafffffeeafffffeeafffffe
     width_detection (0x20) : 0xaa995566
            image_id (0x24) : 0x584c4e58
 encryption_keystore (0x28) : 0x00000000
      header_version (0x2c) : 0x01010000
   fsbl_sourceoffset (0x30) : 0x00001700
         fsbl_length (0x34) : 0x00014008
   fsbl_load_address (0x38) : 0x00000000
   fsbl_exec_address (0x3C) : 0x00000000
   fsbl_total_length (0x40) : 0x00014008
    qspi_config-word (0x44) : 0x00000001
            checksum (0x48) : 0xfc16c530
          iht_offset (0x98) : 0x000008c0
          pht_offset (0x9c) : 0x00000c80
--------------------------------------------------------------------------------
   IMAGE HEADER TABLE
--------------------------------------------------------------------------------
             version (0x00) : 0x01020000        total_images (0x04) : 0x00000003
          pht_offset (0x08) : 0x00000c80           ih_offset (0x0c) : 0x00000900
       hdr_ac_offset (0x10) : 0x00000000
--------------------------------------------------------------------------------
   IMAGE HEADER (zynq_fsbl.elf)
--------------------------------------------------------------------------------
          next_ih(W) (0x00) : 0x00000250
         next_pht(W) (0x04) : 0x00000320
    total_partitions (0x0c) : 0x00000001
                name (0x10) : zynq_fsbl.elf
--------------------------------------------------------------------------------
   IMAGE HEADER (u-boot.elf)
--------------------------------------------------------------------------------
          next_ih(W) (0x00) : 0x00000260
         next_pht(W) (0x04) : 0x00000330
    total_partitions (0x0c) : 0x00000001
                name (0x10) : u-boot.elf
--------------------------------------------------------------------------------
   IMAGE HEADER (system.dtb)
--------------------------------------------------------------------------------
          next_ih(W) (0x00) : 0x00000000
         next_pht(W) (0x04) : 0x00000340
    total_partitions (0x0c) : 0x00000001
                name (0x10) : system.dtb
--------------------------------------------------------------------------------
   PARTITION HEADER TABLE (zynq_fsbl.elf.0)
--------------------------------------------------------------------------------
    encrypted_length (0x00) : 0x00005002  unencrypted_length (0x04) : 0x00005002
        total_length (0x08) : 0x00005002           load_addr (0x0c) : 0x00000000
           exec_addr (0x10) : 0x00000000    partition_offset (0x14) : 0x000005c0
          attributes (0x18) : 0x00000010       section_count (0x1C) : 0x00000001
     checksum_offset (0x20) : 0x00000000          iht_offset (0x24) : 0x00000240
           ac_offset (0x28) : 0x00000000            checksum (0x3c) : 0xffff07e8
 attribute list -
               trustzone [non-secure]            el [el-0]         
              exec_state [aarch-32]     dest_device [none]         
              encryption [no]                  core [none]         
--------------------------------------------------------------------------------
   PARTITION HEADER TABLE (u-boot.elf.0)
--------------------------------------------------------------------------------
    encrypted_length (0x00) : 0x0003b002  unencrypted_length (0x04) : 0x0003b002
        total_length (0x08) : 0x0003b002           load_addr (0x0c) : 0x04000000
           exec_addr (0x10) : 0x04000000    partition_offset (0x14) : 0x000055d0
          attributes (0x18) : 0x00000010       section_count (0x1C) : 0x00000001
     checksum_offset (0x20) : 0x00000000          iht_offset (0x24) : 0x00000250
           ac_offset (0x28) : 0x00000000            checksum (0x3c) : 0xf7f497c8
 attribute list -
               trustzone [non-secure]            el [el-0]         
              exec_state [aarch-32]     dest_device [none]         
              encryption [no]                  core [none]         
--------------------------------------------------------------------------------
   PARTITION HEADER TABLE (system.dtb.0)
--------------------------------------------------------------------------------
    encrypted_length (0x00) : 0x000011ed  unencrypted_length (0x04) : 0x000011ed
        total_length (0x08) : 0x000011ed           load_addr (0x0c) : 0x00100000
           exec_addr (0x10) : 0x00000000    partition_offset (0x14) : 0x000405e0
          attributes (0x18) : 0x00000013       section_count (0x1C) : 0x00000001
     checksum_offset (0x20) : 0x00000000          iht_offset (0x24) : 0x00000260
           ac_offset (0x28) : 0x00000000            checksum (0x3c) : 0xffebc1e4
 attribute list -
               trustzone                         el [el-0]         
              exec_state [aarch-32]     dest_device [none]         
              encryption [no]                  core [none]         
--------------------------------------------------------------------------------
```


<h2> Prepare SD card </h2>


<h3> Create boot and root file system partitions </h3>

On my system, the SD card shows up as `/dev/sdb`. I used a 32GB card, with 1GB allocated for the boot partition and the remainder for the root file system.

```
$ sudo fdisk /dev/sdb
```
*Create two partitions*

	partition 1, primary, default offset, size +1024MB, FAT32 (0x0b), bootable
	partition 2, primary, default offset, default size, Linux (0x83)

*Format the partitions*

```
$ sudo mkfs.msdos -n BOOT /dev/sdb1
$ sudo mkfs.ext4 -L rootfs /dev/sdb2
```

*Mount the fat32 partition and copy the files*

```
$ cp BOOT.BIN boot.scr image.ub /media/nair/BOOT
$ sync
```

**Write root file system**

*Method 1 : using rootfs.ext4 build output*

If the ext4 partition is mounted, unmount it before dd command

```
$ sudo umount /dev/sdb2
$ sudo dd if=rootfs.ext4 of=/dev/sdb2
$ sudo resize2fs /dev/sdb2
$ sudo e2label /dev/sdb2 "rootfs"
```

Mount the partition again

*Method 2 : using rootfs.cpio build output*

```
$ cp images/linux/rootfs.cpio /media/nair/rootfs
$ cd /media/nair/rootfs
$ sudo pax -rvf rootfs.cpio
```

*Method 3 : using rootfs.tar.gz build output*

Ensure the root partition is mounted

```
$ sudo tar xvzf rootfs.tar.gz -C /media/nair/rootfs
```

Set ownership and permissions

```
$ sudo chown root:root /media/nair/rootfs/
$ sudo chmod 755 /media/nair/rootfs/
```

Flush caches to SD

```
$ sync
```

Safely eject the card


<h2> Clean project before rebuilding </h2>

```
rm -rf components/plnx_workspace
petalinux-build -x distclean
petalinux-build -x mrproper --force
```

<h1> Archive petalinux project </h1>

*Method 1 : generate bsp*

```
$ petalinux-package --bsp -p ./petalinux_sd --hwsource ./vivado --output petalinux_sd_may12.bsp
```

*Method 2 : Clean and tar*

```
$ petalinux-build -x mrproper
$ cd ..
$ tar -czvf petalinux_sd_may12.tar.gz ./petalinux_sd
```

<h1> Re-create petalinux project from bsp file </h1>

```
$ petalinux-create -t project -s petalinux_sd_may12.bsp
```
