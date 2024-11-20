# RaspberryPi-FDE-pre-boot-authentication

This tutorial is based on https://forums.raspberrypi.com/viewtopic.php?f=29&t=248057
Since it is outdated and I encrypt a Raspberry PI5 with Bookwork I decided to update it here.

You need an external USB storage media in the setup process to temporarly store the systemfiles, this is NOT the backup as written above. Freespace should be about 1.5 times bigger then your current rootfs usage.

On this Tutorial we will implement the RAM content deletion like described here https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=247870

 I've decided not to make any scripts, because you should understand what you're doing here. 

`sudo apt install initramfs-tools secure-delete dropbear-initramfs`

 create file in /etc/initramfs-tools/hooks/myHook 

```
#!/bin/sh

set -e

PREREQ=""

prereqs () {
        echo "${PREREQ}"
}

case "${1}" in
        prereqs)
                prereqs
                exit 0
                ;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/bin/sdmem /usr/bin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/dumpe2fs /sbin
copy_exec /sbin/resize2fs /sbin
copy_exec /bin/lsblk /sbin
copy_exec /sbin/e2fsck /sbin

exit 0
```

**Important `chmod +x myHook`**

Place in /etc/initramfs-tools/scripts/init-top/sdmem

```
#!/bin/sh
PREREQ=""
prereqs()
{
   echo "$PREREQ"
}

case $1 in
prereqs)
   prereqs
   exit 0
   ;;
esac

/bin/sdmem -llv
```

**Also here ensure that it is executable `chmod +x sdmem`**




 
