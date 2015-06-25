The files in this directory are all you need to boot up ZedBoard with Linux,
Xenomai real-time support, and NASA's Core Flight System. It is assumed that
you are on a Linux machine.

NOTE: The cfs files aren't where I say they are yet. They'll be there soon.

1. Copy all files in this directory to your SD card (well, technically you
   don't need to copy this readme, but it's kind of nice to have, isn't it?).
2. Unmount your SD card, take it out, and put it in the ZedBoard.
3. See the five jumpers just above the Digilent logo on the board? Make sure
   that MIO4 and MIO5 are shorted to 3V3 and that the other three are shorted
   to ground.
4. Connect a USB-to-MicroUSB cord from your PC to ZedBoard.
5. Install Minicom on your PC. (eg "sudo apt-get install minicom")
6. Type "dmesg | grep tty" in a terminal - whichever TTY it says was recently
   connected is the one you'll need (eg ttyUSB0).
7. Type "sudo minicom -s", go to "Serial port setup", and make sure the
   serial device is correct based on the previous step (eg /dev/ttyUSB0). 
8. Make sure the lockfile is /var/run, Bps/Par/Bits is 115200 8N1, and both
   flow control settings are No.
9. Press enter, then "Save setup as dfl", then "Exit from Minicom".
10. Flip the switch to turn on the board (it's by the power cable).
11. Type "sudo minicom" to connect to the board as it boots.
12. When the board finishes booting, the username and password are both 
    "root".
13. Mount the SD card: "mount /dev/mmcblk0p1 /mnt"
14. Copy the CFS files: "cp -r /mnt/exe/ /opt"
15. cd /opt/exe
16. ./core-linux.bin

If that was painful, don't worry, you can skip steps 3-9 from now on.

These are instructions for re-creating the files on this card from scratch. A 
few of the instructions were done from memory, so contact me (Bryan Anderson) 
if anything does not work. The instructions for getting and compiling CFS are
not here yet - I'll add them soon.

1. Get pre-built files for boot
http://www.wiki.xilinx.com/Zynq+14.5+-+2013.1+Release
http://www.wiki.xilinx.com/file/view/14.5-release.tar.xz/431555724/14.5-release.tar.xz
Place the following files on a FAT32-formatted SD card:
zed/boot.bin
uImage
uramdisk.image.gz
zed/devicetree.dtb

2. Build kernel with Xenomai
cd ~
mkdir zedboard
(Download http://download.gna.org/xenomai/stable/xenomai-2.6.3.tar.bz2)
tar -xf xenomai-2.6.3.tar.bz2
git clone https://github.com/Xilinx/u-boot-xlnx.git
cd u-boot-xlnx
git checkout 20a6cdd301941b97961c9c5425b5fbb771321aac
make zynq_zed_defconfig
make
cd ..
git clone https://github.com/Xilinx/linux-xlnx.git
cd linux-xlnx
git checkout 6a0bedad60e2bca8d9b50bf81b9895e29e31a6d7
git remote add stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
git fetch stable
git merge v3.8.13 
(Accept the merge commit.)
patch -p1 < ../xenomai-2.6.3/ksrc/arch/arm/patches/zynq/ipipe-core-3.8-zynq-pre.patch
patch -p1 < ../xenomai-2.6.3/ksrc/arch/arm/patches/ipipe-core-3.8.13-arm-3.patch 
patch -p1 < ../xenomai-2.6.3/ksrc/arch/arm/patches/zynq/ipipe-core-3.8-zynq-post.patch
../xenomai-2.6.3/scripts/prepare-kernel.sh --arch=arm --linux=../../linux-xlnx
PATH=/opt/Xilinx/SDK/2015.1/gnu/arm/lin/bin:/home/{YOUR_USERNAME_HERE}/zedboard/u-boot-xlnx/tools:$PATH
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- xilinx_zynq_defconfig
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- menuconfig
(Disable "Remoteproc drivers" under Device Drivers and then "Support for hot-pluggable CPUs" under Kernel Features, exit and save.)
(While you're at it, you'll want to enable NFS Server Support (for NFSv3) under File Systems -> Network File Systems)
make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- UIMAGE_LOADADDR=0x8000 uImage
(The kernel image is in arch/arm/boot, called uImage. Replace your existing uImage (on your SD card) with this one.)

3. Make Xenomai userspace support
cd ~/zedboard
mkdir xentemp
mkdir xenbuild
cd xenbuild
../xenomai-2.6.3/configure CFLAGS="-march=armv7-a" LDFLAGS="-march=armv7-a" --build=i686-pc-linux-gnu --host=arm-xilinx-linux-gnueabi-
(If that fails, try without the dash between 7 and a.)
make DESTDIR=../xentemp install
(Now you should have two directories: dev containing some device files and usr containing a directory called xenomai.)
(Copy your uramdisk.image.gz from step 1 to this directory)
(All the sudos in the upcoming section are there for a reason - it may not work without them)
sudo mkdir tmprootfs
dd if=uramdisk.image.gz of=ramdisk.image.gz bs=64 skip=1
mv uramdisk.image.gz old_uramdisk.image.gz
gunzip -c ramdisk.image.gz | sudo sh -c 'cd /tmp/rootfs/ && cpio -i'
sudo mv dev/* tmprootfs/dev/
sudo mv usr/* tmprootfs/usr/
sh -c 'cd tmprootfs/ && sudo find . | sudo cpio -H newc -o' | gzip -9 > ramdisk.image.gz
mkimage -A arm -T ramdisk -C gzip -d ramdisk.image.gz uramdisk.image.gz
rm ramdisk.image.gz
(Copy your new uramdisk.image.gz to the SD card)

4. Boot Linux and test Xenomai
(Insert SD card, turn on ZedBoard. If boot fails (it probably will) and returns to u-boot command line, do the next few steps.)
setenv sdboot 'echo Copying Linux from SD to RAM... && mmcinfo && fatload mmc 0 0x6600000 ${kernel_image} && fatload mmc 0 0x6000000 ${devicetree_image} && fatload mmc 0 0x2000000 ${ramdisk_image} && bootm 0x6600000 0x2000000 0x6000000'
setenv ramdisk_size 0x4000000
saveenv
run sdboot
(Now it should boot. If it booted to begin with, well, lucky you. Pick back up here. Username root, password root.)
cd /usr/xenomai/bin
./xeno latency
(If a latency test runs, congrats, it worked. If not, go to your Xenomai source directory and read TROUBLESHOOTING.)
