# pogoplug-v4-bodhi-rootfs-debian
bodhi's Debian rootfs, prepared specifically for Pogoplug V4 devices.

## Rationale

User 'bodhi' at http://forum.doozan.com has prepared a Debian rootfs tarball for use with many small ARM machines (see this thread: http://forum.doozan.com/read.php?2,12096).

However, his release still requires a few additional steps in order to prepare it for the Pogoplug.

This github repository aims to provide versions of his rootfs which are fully ready to be used with Pogoplug V4 devices (Pogoplug Mobile and Pogoplug Series 4).

## Producedure

I performed the following steps to prepare the rootfs and disk images.

These instructions are based on the following env vars.  Adjust as needed for your situation.

```
# SD card device:
dev=sdf

# temporary mount point:
mnt=/mnt/tmp
```

### Install the rootfs onto an SD card

Partition and format the SD card:

```
sfdisk /dev/${dev} << 'EOF'
,,,*
EOF
```

```
mke2fs -j -L rootfs /dev/${dev}1
mount /dev/${dev}1 ${mnt}
```

Unpack the tarball:

```
if [ ! -e ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 ]
then
    wget -O ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 https://bitly.com/1ELvRaw
    md5sum -c - << EOF
b5057448e7e08c747793f205e7027395  ${HOME}/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2
EOF
fi

cd ${mnt}
cat ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 | bunzip2 | tar x
```

Edit `/etc/fstab` to use `ext3` as the rootfs filesystem.  The `fstab` entry should look like so:

```
/dev/root / ext3 noatime,errors=remount-ro 0 1
```

Setup the kernel image for the bootloader:

```
cd boot
cp -a zImage-3.18.5-kirkwood-tld-1  zImage.fdt
cat dts/kirkwood-pogoplug_v4.dtb  >> zImage.fdt
mv uImage uImage.orig
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-3.18.5-kirkwood-tld-1 -d zImage.fdt uImage
sync
```

At this point the rootfs should be bootable by a Pogoplug.


### Preparing the rootfs for distribution

Install the latest secyrity updates.  Reboot to be sure everything still works.

```
apt-get update
apt-get dist-upgrade
reboot
```


