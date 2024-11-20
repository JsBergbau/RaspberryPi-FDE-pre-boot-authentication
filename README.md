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

Initramfs creation ensures that needed libraries are automatically copied.

The system scripts are in /usr/share/initramfs-tools/hooks/ and are executed first upon initramfs generation.

Since Debian bookworm uses an initramfs by default, we don't need the custom scripts for rpi-initramfs generation as in the original Tutorial linked at the beginning.

For updating initramfs it is sufficient to execute `sudo update-initramfs -u -k all`. At this time we will get an error.

Now we come to main enryption part. Create
/etc/initramfs-tools/conf.d/my containing 

```
BUSYBOX=y
DROPBEAR=y
```

Dropbear ssh Server enables remote unlock. It seems that it supports only public key auth in initramfs mode. So if you don't have a keypair for remote login in, just create one or copy your key 

```
ssh-keygen -f key -b 4096
#copy it to dropbear
sudo cp key.pub /etc/dropbear/initramfs/authorized_keys
```

You can enter a password for the Key. Please protect this key. Additional password protection makes it harder for an adversary which gets the key. Because with the key in plaintext he could probably compromise initramfs and place some code that could record your entered password. However just evesdropping the line won't be a security risk since according to [https://security.stackexchange.com/ques ... 519-vs-rsa](https://security.stackexchange.com/questions/90077/ssh-key-ed25519-vs-rsa) because ssh uses forward secrecy. However if the attaker has the private key in cleartext he can setup a man in the middle attack which you can't notice since he can copy the server key since he can authenticate with your private key. 

Note: Supporting only Public Key Pair auth instead of password auth, means protection against man in the middle attacks (as long as you keep this key away from the adversary). Without public key authentication you always have to compare server key hash to be sure you're connected to the right machine. Since you're entering your main encryption password, I now don't recommend enabling password auth like asimiklit described below, anymore. References h[ttps://security.stackexchange.com/ques ... is-it-true](https://security.stackexchange.com/questions/180230/ssh-key-based-login-is-not-vulenerable-to-mitm-attack-is-it-true) and [https://security.stackexchange.com/ques ... is-it-true](https://security.stackexchange.com/questions/180230/ssh-key-based-login-is-not-vulenerable-to-mitm-attack-is-it-true)

Copy the key to your PC so that you can log into!

Exceute the following in a root shell like sudo -i




 
