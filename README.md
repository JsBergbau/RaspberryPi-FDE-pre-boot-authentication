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

Additionally to the original tutorial you need to modify `/etc/initramfs-tools/modules` and add
```
dm_crypt
aes
sha256
```
Without the AES libraries won't be available in the initramfs. Cryptsetup then can't create the /dev/mapper device and will print an error `device-mapper: reload ioctl on   failed: No such file or directory`. dmesg will show following.
```
[   89.086304] device-mapper: table: 254:0: crypt: Error allocating crypto tfm (-ENOENT)
[   89.094189] device-mapper: ioctl: error adding target to table
```
Don't know why you need sha256. ChatGPT suggested these 3 modules, thats why I copied it.


Also edit aditionally `/boot/firmware/config.txt` comment line `dtoverlay=vc4-kms-v3d` and add `dtoverlay=vc4-kms-v3d`.
This setting prevents the screen hanging at a message similiar to `vc4-drm axi:gpu: bcm2712_iommu_of_xlate MMU 1000005200.iommu` or being black in initramfs.
```
#dtoverlay=vc4-kms-v3d
dtoverlay=vc4-fkms-v3d
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

`sed -i '$s/$/ cryptdevice=\/dev\/mmcblk0p2:sda2_crypt/' /boot/firmware/cmdline.txt`

Change mmcblk0p2 to sda2 if you are booting from an usb storage drive.

```
ROOT_CMD="$(sed -n 's|^.*root=\(\S\+\)\s.*|\1|p' /boot/firmware/cmdline.txt)"
sed -i -e "s|$ROOT_CMD|/dev/mapper/sda2_crypt|g" /boot/firmware/cmdline.txt
```
 We change the root point. Rebooting after that causes kernel to stop in initramfs. Since the root filesystem can't be found anymore. **Don't reboot now**

Edit /etc/fstab
Search for line that contains `/` as the mountpoint. In the example `PARTUUID=64865f4b-02  /               ext4    defaults,noatime  0       1`
now change `PARTUUID=64865f4b-02` to `/dev/mapper/sda2_crypt`. Complete line is then `/dev/mapper/sda2_crypt  /               ext4    defaults,noatime  0       1`

We add the dmcrypt mountpoint to fstab. Hint: The root= Parameter in cmdline.txt is for the Kernel at boottime. At a later stage (I think right before systemd starts) root "/" gets remounted to the value in fstab

We append it to crypttab. This is necessary that "cryptroot-unlock" finds that partition, asks for password and opens the encrypted device. Otherwise we would have to rember and enter the commands by ourselves. 

`echo 'sda2_crypt /dev/sda2 none luks' | sudo tee --append /etc/crypttab > /dev/null` (for external USB-drive)
or 
`echo 'sda2_crypt /dev/mmcblk0p2 none luks' | sudo tee --append /etc/crypttab > /dev/null` (for sdcard)

 We now we create the initramfs 
`sudo update-initramfs -u -k all`

There should be an output like

```
update-initramfs: Generating /boot/initrd.img-6.6.51+rpt-rpi-v8
W: Couldn't identify type of root file system for fsck hook
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
'/boot/initrd.img-6.6.51+rpt-rpi-v8' -> '/boot/firmware/initramfs8'
update-initramfs: Generating /boot/initrd.img-6.6.51+rpt-rpi-2712
W: Couldn't identify type of root file system for fsck hook
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
WARNING: Unknown X keysym "dead_belowmacron"
'/boot/initrd.img-6.6.51+rpt-rpi-2712' -> '/boot/firmware/initramfs_2712'
```

Please ignore the warnings and check the last line which confirms modification of /boot/config.txt
Line "WARNING: Unknown X keysym "dead_belowmacron" only occurs with German Keyboard Layout, this is a bug in debian, see [https://bugs.debian.org/cgi-bin/bugrepo ... bug=903393](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=903393) and [https://forums.puri.sm/t/what-does-this ... ron/6393/3](https://forums.puri.sm/t/what-does-this-mean-warning-unknown-x-keysym-dead-belowmacron/6393/3) with possible solution if you want to get rid of this warning.

Now reboot. It stops in the initramfs. You can connect via SSH user "root" and the private key. Maybe your IP has changed. Simplest way is to connect a monitor and look for the IP address or watch in your router/DHCP-Server.
First execute 
`PATH=/sbin:/usr/sbin:/bin:/usr/bin`

Don't know why in dropbear ssh shell the standard path enviroment doesn't include the sbin directories. In initramfs shell it is included in the Path. If somebody knows how to fix this permanently please write.

 Now we reduce the filesystem 

```
e2fsck -f /dev/sda2
resize2fs -fM /dev/sda2
```

resize2fs will tell you the blockcount it reduced. Please note that down. This will be the number for count in the next step.
If you have an ext4 filesystem on your USB storage media you can mount it (you could also include ntfs-3g or other filesystems in initramfs, but that is not part of this tutorial), otherwise we write directly to the media.

Each ext4 block is 4KB, so multiplying the number of blocks gives you the filesystem size in KB. To speed things up we block copy the whole filesytem. 

```
dd if=/dev/sda2 of=<mountdir/filename> bs=16M count=<number see below>
or
of=/dev/sdb for direct copy, attention all existing data on the USB device is lost then[/b]> 
```

Get the count by dividing your blockcounts from resize2fs by 4096 (short for * 4 / 16 / 1024) round that number up. So e.g. if you get 922044 as number of blocks, devide that by 4096. This gives 225,10839... and so your count number is 226 . dd in initramfs doesn't support status=progress so you won't see any progress. Make sure there are no errors during copy!

Hint: There is also a luks inplace enryption (cryptsetup-reencrypt). You could add that to your initramfs as well, but for data security reasons I feel more comfortable with this method.

Now we encrypt 
`cryptsetup -v --type luks2 --cipher aes-xts-plain64 --pbkdf argon2id --key-size 512 --hash sha256 --iter-time 4000 --verify-passphrase --use-random --pbkdf-memory 1048576 --pbkdf-parallel 2 luksFormat /dev/sda2`


We use 256 Bit AES (because of XTS we need 512 bit key, one half is for XTS). For password protetion against bruteforce we use the latest hash function argon2id which secures against bruteforce by dedicated hardware and side channel attacks (these one are not so important here). These is the safest setup I currenctly know. Please use a strong password / passphrase!
Type YES in uppercase letters to confirm overrding your existing files.
Note: With adding `--pbkdf-memory 1048576` you should use 1 GB of memory. `--pbkdf-parallel 2` uses 2 CPU threads. 

Now we open the encrypted volume:

```
cryptsetup luksOpen /dev/sda2 sda2_crypt
#or to enable TRIM support, my recommended way, this sets a FLAG that discards are allowed, see with /usr/sbin/cryptsetup luksDump /dev/sda2 Flags: allow-discards
cryptsetup --allow-discards --persistent open /dev/sda2 sda2_crypt
```

Now we copy the data back
`dd if=<path of "of=" from above> of=/dev/mapper/sda2_crypt bs=1M`
There is no need for entering a count now, since dd will stop when it has copied our file. 

 Now we have to resize the filesystem to maximum filesize again. Since nothing has been written we can omit the file system check. Just execute now 

 `resize2fs -f /dev/mapper/sda2_crypt`
