# OpenWRT SignalK

17 January 2025  
OpenWRT v 23.05.5  
Raspberry Pi 3B

The idea is to run signalk from on a router running OpenWRT. The router will be "always on" and provide monitoring. SignalK can be run in Docker and this project is to test my ability to run a signalk server in docker and have a way to easily backup my settings. I currently do not have a OpenWRT compatible router, so I am using a raspberry pi running the latest openwrt as a bench testing environment.

## Raspberry Pi

### Install the Image

I followed this guide and downloaded the latest image available   
https://openwrt.org/toh/raspberry\_pi\_foundation/raspberry\_pi

### First Boot

On the first boot, I initially thought the boot process failed or was hung. Pressing enter on a keyboard connected to the Pi brings up the openwrt busybox terminal.

## Initial Setup

It may be necessary to manually set an IP address. In my case, I could not access the luci interface on my local network until I set a static ip address, gateway and dns. The following commands done using a keyboard connected directly to the raspberry pi.

*uci set network.lan.ipaddr="192.168.0.211"*  
*uci set network.lan.gateway="192.168.0.1"*  
*uci set network.lan.dns="192.168.0.1"*  
*uci commit*  
*/etc/init.d/network restart*

Set your password on the luci interface   
Enable ssh access in luci \> System \> SSH Access  
Add your public key in  luci \> System \> SSH-Keys

The rest of the commands are issued over ssh using a terminal from my laptop where I can copy and paste commands. 

## Install Packages

I list all of the packages needed for the project here in one long install command. I tried to include the individual install commands under each section.  
*opkg update*  
*opkg install dockerd docker docker-compose luci-app-dockerman nano kmod-fs-btrfs btrfs-progs lsblk git block-mount kmod-fs-ext4 e2fsprogs parted kmod-usb-storage cfdisk git-http*

## Configure Overlay 

This will require about 4 GIB of storage space. On the Raspberry Pi image, the remainder of the SD card is left free. Below are the instructions to add a partition to the free space and mount it to /overlay.  
This should also work with an external USB drive, but view instructions for usb specifics. 

Docker was having issues pulling images with a ext4 filesystem, so I will use btrfs for overlay.

Parts of this guide were used [https://openwrt.org/docs/guide-user/additional-software/extroot\_configuration](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)  
I will be using the empty space on the sd-card to create the overlay partition. The section "Configuring extroot" was helpful. I followed my own partition plan and since /overlay did not already exist on my system, the sections "Configuring rootfs\_data / ubifs" and "Transferring data" were unnecessary.

### Required Software

*opkg install kmod-fs-btrfs btrfs-progs lsblk block-mount parted kmod-usb-storage cfdisk*

### Identify Your Device

List partitions and take note of your device(s). In my case the base device is */dev/mmcblk0*  
	*df \-h*  
*lsblk*

### Add Partition & Format

Add partition, change /dev/mmcblk0 to your device found above. Take note of the new   
	*cfdisk /dev/mmcblk0*  
Take note of the new partition you created. In my case the new partition is */dev/mmcblk0p3*  
Format the new partition, change /dev/mmcblk0p3 to the partition you created above  
	*mkfs.btrfs \-L extroot /dev/mmcblk0p3*

### Mount New Partition

Instructions in this guide ([https://openwrt.org/docs/guide-user/additional-software/extroot\_configuration](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)) were unclear and no preexisting /overlay existed on my install. I needed to manually define $MOUNT as /overlay.  
The first two lines below define variables used in the rest of the commands. Modify them to match the device identified above.

*DEVICE="/dev/mmcblk0p3"*  
*MOUNT="/overlay"*  
*eval $(block info ${DEVICE} | grep \-o \-e 'UUID="\\S\*"')*  
*eval $(block info | grep \-o \-e 'MOUNT="\\S\*/overlay"')*  
*uci \-q delete fstab.extroot*  
*uci set fstab.extroot="mount"*  
*uci set fstab.extroot.uuid="${UUID}"*  
*uci set fstab.extroot.target="/overlay"*  
*uci commit fstab*

*reboot*

### Verify

The instructions here were helpful in verifying the above was successful  [https://openwrt.org/docs/guide-user/additional-software/extroot\_configuration\#testing](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#testing)

Running *lsblk* over ssh confirmed the new partition is mounted. Your output will look different depending on your device names. For me, */dev/mmcblk0p3* is mounted as */overlay* with a disk size of *14.7G*  
*lsblk*  
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS  
mmcblk0     179:0    0 14.8G  0 disk   
├─mmcblk0p1 179:1    0   64M  0 part /boot  
├─mmcblk0p2 179:2    0  104M  0 part /rom  
└─mmcblk0p3 179:3    0 14.7G  0 part /overlay

## Github

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

### Required Software

*opkg install git git-http openssh-keygen openssh-client-utils*

### Configure Keys

*ssh-keygen -t ed25519 -C "youremail@here.com"*
*cat ~/.ssh/id_ed25519.pub*

### Add Keys on Github

[https://github.com/settings/keys](https://github.com/settings/keys)

### Notes

If you cloned a repository using https you can change the remote using the following command.   
	*git remote set-url origin git@github.com:mefenlon/docker\_signalk.git*

## Signalk in Docker

### Required Software

*opkg install dockerd docker docker-compose luci-app-dockerman* 

### Clone the Project

*cd \~*  
*git clone git@github.com:mefenlon/docker\_signalk.git*

### Set Permission for Settings Directory

*chmod 777 ./docker\_signalk/signalk\_conf*

### Launch SignalK Server

*cd docker\_signalk*  
*docker compose up \-d*

### Notes

To make the image restart on reboot, check that *dockerd* is listed and set to enable in luci \> System \> Startup. In thedocker-compose.yml,setting *restart: always* will make sure the image is started after a reboot or power cycle.  
I followed this guide [https://openwrt.org/docs/guide-user/virtualization/docker\_host?s\[\]=docker](https://openwrt.org/docs/guide-user/virtualization/docker_host?s[]=docker)  
I was initially getting an error (*openwrt failed to register layer: lsetxattr security.capability /usr/bin/node*) when attempting to pull the signalk image. On the raspberry pi I am using for testing, I believe this was due to the /overlay partition being formatted ext4. Creating a btrfs partition and mounting it as /Overlay (instructions above) fixed the error.