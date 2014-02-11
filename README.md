 abootimg - manipulate Android Boot Images.
===========================================

 (c) 2010 Gilles Grandou <gilles@grandou.net>


NOTE: Gilles's code makes assumptions about the structure of the boot image header that are not valid for all devices (mainly due to Android fragmentation since 2010). You should unpack and repack the boot image you're working with and do a binary diff of the repacked image with the original to verify abootimg roundtrips. -dlg


Android Boot Images
-------------------

It a special partition format defined by the Android Open Source Porject.
See bootimg.h in the source tree for more information about the structure.

It's used by Android Bootloaders to boot the OS. 

Boot images generally include:
- a kernel image
- a ramdisk image 
- optionaly, a 2nd stage bootloader
- the command line passed to the kernel when booting.

The official tool, [mkbootimg](https://android.googlesource.com/platform/system/core/+/master/mkbootimg/) used to create boot images is part of the [Android Project](https://android.googlesource.com).

abootimg can work directly on block devices, or, more safely, on a file image.
File images can be read/written with dd:

	$ dd if=/dev/mmcblk0p2 of=boot.img
	$ dd if=boot.img of=/dev/mmcblk0p2

You obviously need to have the rights to use the block device.

Android boot images contain a 32-byte ID. The specification does not actually 
mandates any specific implementation of this ID (it can be a timestamp, a CRC 
checksum, a SHA hash, ...). The bootloader appears to do nothing of this ID, it's 
solely here for tracking purpose. Currently abootimg does nothing with it, it's 
never touched/modified.



Building abootimg
-------------------

On a linux system, it's simply as:

	$ make

blkid library is needed to perform some sanity checks when writing boot image
directly on a block device (to avoid writing a valid existing filesystem).


On Mac OS X, just do:

	$ make -f Makefile.osx

 Jeff Verkoeyen's [fmemopen for BSD systems](https://github.com/jverkoey/fmemopen)
 has been included as a squashed subtree.


Looking at an Android Boot Image
----------------------------------

Basic Information can be extracted from a boot image, using:


	$ abootimg -i <bootimg>

Here is an example:


	$ ./abootimg -i boot.img

	Android Boot Image Info:

	* file name = boot.img

	* image size = 8388608 bytes (8.00 MB)
	  page size  = 2048 bytes

	* Boot Name = ""

	* kernel size       = 3002744 bytes (2.86 MB)
	  ramdisk size      = 1639626 bytes (1.56 MB)

	* load addresses:
	  kernel:       0x10008000
	  ramdisk:      0x11000000
	  tags:         0x10000100

	* cmdline = mem=448M@0M nvmem=64M@448M vmalloc=320M video=tegrafb console=tty0 usbcore.old_scheme_first=1 quiet splash elevator=noop tegraboot=sdmmc cmdpart=1:7168:10240,2:17408:16384,3:35840:614400,4:4004864:27096064

	* id = 0x07571070 0x13950a6a 0x185c996f 0x9ab7b64d 0xcccd09bd 0x00000000 0x00000000 0x00000000 



Extracting elements from an Android Boot Image
------------------------------------------------

All parts of the boot image can be extracted with:

	$ abootimg -x <bootimg> [<bootimg.cfg> [<kernel> [<ramdisk> [<secondstage>]]]]

Parts name are optional. Defaults are used if none are given:
* bootimg.cfg for the configuration file
* zImage for the kernel image
* initrd.img for the ramdisk
* stage2.img for the second stage image

Here is an example:

	$ abootimg -x boot.img
	writing boot image config in bootimg.cfg
	extracting kernel in zImage
	extracting ramdisk in initrd.img



Boot Configuration file
-------------------------

It's an editable ascii file which is basically a dump of the header content, 
to be able to rebuild a compatible image later.

Each entry takes one line and is in the form of:

	<entry> = <value>

You can put any number of spaces/tab before/after the `=` sign.  `<value>` is evaluated starting from the fisrt non space character following the `=` until the end of line.

Numerical values can be given in decimal (12345) or hexadecimal (0x1234abcd).

Known configuration entries are:

* `bootsize` The size of the boot image to produce.

* `pagesize` All sizes have to be a multiple of the page size.  The standard page size is 2048 bytes. I don't know if other page sizes are supported by Android bootloader.

* `kerneladdr, ramdiskaddr, secondaddr, tagsaddr` Address in RAM used to load the kernel, ramdisk, 2nd stage bootloader, and tags table.

* `name` Name given to the boot image. Not used by the bootloader, but it can be useful to keep track of what the image actually contains.

* `cmdline` The command line passed to the kernel when booting
	 
	
Updating an existing Android Boot Image
-----------------------------------------


An existing valid Boot Image can be updated with:


	$ abootimg -u <bootimg> [-c "param=value"] [-f <bootimg.cfg>] [-k <kernel>] [-r <ramdisk>] [-s <secondstage>]

Any part of the image can be individully updated. As an example:

	$ abootimg -u boot.img -k zImage.new

		will update the kernel

	$ abootimg -u boot.img -r initrd.new.img

		will update the image

	$ abootimg -u boot.img -k zImage.new -r initrd.new.img

		will update both


Image configuration can be updated either by giving a configuration file 
(-f option) or by giving individual config entries on the command line (-c).
Severial config entries (-c) can be given on comamd line.

As an example:

	$ abootimg -u boot.img -f bootimg.new.cfg

		will update the configuration without touching the kernel 
		nor the ramdisk

	$ abootimg -u boot.img -c "cmdline = mem=448M@0M nvmem=64M@448M vmalloc=320M \
	video=tegrafb console=tty0 usbcore.old_scheme_first=1 quiet splash elevator=noop \
	tegraboot=sdmmc tegrapart=recovery:700:a00:800,boot:1100:1000:800,mbr:2100:200:800,\
	system:2300:25800:800,cache:27b00:32000:800,misc:59b00:400:800,userdata:5a000:9a600:800"

		update the boot conmand line

	$ abootimg -u boot.img -c "bootsize=0x500000"

		update the boot image size to 5MB.
		It's usefull if you want to shrink an existing image to make it 
		fit in another smaller boot partition.
		(On Toshiba AC100, it allows you to take a part06.img and 
		transform it to fit inside part05)


The original boot image has to be valid, or abootimg will refuse to 
update it.




Creating a new Android Boot Image from scratch
------------------------------------------------


A new boot image can be created with:

	$ abootimg --create <bootimg> [-c "param=value"] [-f <bootimg.cfg>] -k <kernel> -r <ramdisk> [-s <secondstage>]

Parameters are the same as for updating an image; the only difference is that specifying the
kernel and ramdisk is mandatory.



Working directly of Block Devices
-----------------------------------


Instead of manipulating boot images as regular files, you can work directly on
block device.

Some examples:

	$ sudo abootimg -i /dev/mmcblk0p2

		read the current boot partition

	$ sudo abootimg -u /dev/mmcblk0p2 -k arch/arm/boot/zImage

		update the boot partition with the kernel you have just built

	$ sudo abootimg -u /dev/mmcblk0p2 -c "cmdline=..."

		update the boot partition with a new boot cmdline

	$ sudo abootimg --create /dev/mmcblk0p2 -f boot.cfg -k zImage -r initrd.img

		overwrite the boot partition (which can be damaged) with a 
		brand new image

If abootimg has to write to a block device (-u and --create), some sanity 
check are performed:

	* you need read/write access to the block device,

	* the actual partition must not be identified as containing a valid 
	  filesystem as recognised by blkid library (specifically, it cannot 
	  contains a ext2/3 filesystem -- this avoids overwriting the
	  root filesystem accidentally),

	* the updated/created boot image has to be same size as the block 
	  device you try to write on,

	* if abootimg is in update mode , the current boot partition has to contain a valid 
	  Android Boot Image.
	  
Failing any of these tests will abort the operation on the block device.

It's by definition more risky to manipulate block device, as a bad 
manipulation can as bad manipulation can make your system unbootable if you
don't fix it before the next reboot.

On the other hand, manipulating the block device allows abootimg to prevent 
most of the stupid mistakes which can be made when writing boot image with dd
(overwriting another filesystem, writing a boot image bigger than the 
partition, writing the wrong file or an invalid partition, ...)


