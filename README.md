# Armbian Swaproot

I like Armbian very much and use it often as small test box for tinkering with
Linux.
As I use my single board computers headless, I do not want to swap sdcards and
reflash images.

Therefore I wrote small scripts that swap root filesystems that reside on an
extra usb flash drive.

## License

These scripts are released under the MIT License.

## Maturity and Caveats

While these scripts have been actively tested on Orange Pi Zero with latest Armbian,
it is quickly hacked to just work in my context to have a quick means of switching
between fresh armbian installs on my single board computer.

There is currently not much error handling.

Also the armbian community stresses the importance of high-quality storage media.
Especially sd cards. Storing and restoring a ext dump introduces a lot of extra
write IOPS on the storage device. This may wear out the thumb drive more quickly.

Therefore I recommend this setup only for testing purposes and hope you are doing
frequent backups to an offsite location for your important data.

While this method has been used quite often in an embedded environment, other
techniques like ostree seem to be better suited and I hope it will be available
on armbian in the future.

If you are looking for a robust method, I would recommend to go for ostree.

https://ostree.readthedocs.io/en/latest/

I for myself use two root filesystems for now and store docker overlays on the
data partition.

## Storage Layout

~~~
                  ║                                                    
                  ║                                                    
┌────────────────┐║┌───────────┐┌───────────┐┌───────────┐┌───────────┐
│ /dev/mmcblk0p1 │║│ /dev/sda1 ││ /dev/sda2 ││ /dev/sda3 ││ /dev/sda4 │
│      BOOT      │║│  STAGING  ││   ROOTA   ││   ROOTB   ││   DATA    │
└────────────────┘║└───────────┘└───────────┘└───────────┘└───────────┘
                  ║                                                    
                  ║                                                    
~~~ 

## Prepare USB flash drive

The following script prepares an attached USB flash drive for the swaproot mecha
nism.

PLEASE MAKE SURE THAT THIS IS THE ONLY USB DISK ATTACHED.
THE DISK WILL BE FORMATTED!

~~~bash
target=/dev/sda
dd if=/dev/zero of=$target bs=1M count=10
parted --align optimal --script $target mklabel msdos
parted --align optimal --script $target mkpart primary 1MiB 1024MiB
parted --align optimal --script $target mkpart primary 1024MiB 5120MiB
parted --align optimal --script $target mkpart primary 5120MiB 9216MiB
parted --align optimal --script $target mkpart primary 9216MiB 100%
parted $target print
mkfs.ext4 -F -L STAGING -m 0 /dev/sda1
mkfs.ext4 -F -L ROOTA -m 0 /dev/sda2
mkfs.ext4 -F -L ROOTB -m 0 /dev/sda3
mkfs.ext4 -F -L DATA -m 0 /dev/sda4
test -d /media/staging || mkdir -p /media/staging
mount --label STAGING /media/staging/
cd /media/staging
which dump || apt-get install -y dump
dump -j9 -0af /media/staging/root.dump /
test -d /media/roota || mkdir -p /media/roota
mount --label ROOTA /media/roota/
cd /media/roota
restore -rf /media/staging/root.dump
test -d /media/rootb || mkdir -p /media/rootb
mount --label ROOTB /media/rootb/
cd /media/rootb
restore -rf /media/staging/root.dump
sed -i 's/^rootdev=.*/rootdev=LABEL=ROOTA/g' /boot/armbianEnv.txt
sync
reboot
~~~

## Overview of the scripts

The following scripts are part of the armbian swaproot mechanism:

### showroot

Displays the device that is used for the active root filesystem.

### storeroot

Saves the currently booted root filesystem as dump.

### swaproot

Alternates the active root device between ROOTA and ROOTB labeled filesystems.

### restoreroot

Restores the current backup of the root filesystem on the alternate root
partition.
