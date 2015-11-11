# pogoplug-v4-bodhi-rootfs-debian
bodhi's Debian rootfs, prepared specifically for Pogoplug V4 devices.

## Rationale

User 'bodhi' at http://forum.doozan.com has prepared a Debian rootfs tarball for use with many small ARM machines (see this thread: http://forum.doozan.com/read.php?2,12096).

However, his release still requires a few additional steps in order to prepare it for the Pogoplug.

This github repository aims to provide versions of his rootfs which are fully ready to be used with Pogoplug V4 devices (Pogoplug Mobile and Pogoplug Series 4).

## Producedure I Used to Modify bodhi's rootfs

I performed the following steps to prepare the rootfs and disk images.

All of these commands should be run as **root**.

These instructions are based on the following env vars.  Adjust as needed for your situation:

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
cd ${mnt}/boot
cp -a zImage-3.18.5-kirkwood-tld-1  zImage.fdt
cat dts/kirkwood-pogoplug_v4.dtb  >> zImage.fdt
mv uImage uImage.orig
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-3.18.5-kirkwood-tld-1 -d zImage.fdt uImage
```

Unmount the SD card:

```
umount ${mnt}
sync
```

### Preparing the rootfs for distribution

Insert the SD card into a Pogoplug and **boot it**.

Comment out the `deb-src` entry in /etc/sources.list (speeds up apt a bit):

```
sed -i'' 's/^deb-src/#deb-src/' sources.list
```

Install the latest security updates:

```
apt-get update
apt-get dist-upgrade
```

Remove cached packages:

```
apt-get clean
```

Restore root's `~/.profile`:

```
cp -a /usr/share/base-files/profile /root/.profile
```

Clear out `/etc/udev/rules.d/70-persistent-net.rules`:

```
echo -n > /etc/udev/rules.d/70-persistent-net.rules
```

Set root's password to be the empty string.  Manually edit `/etc/shadow` so that root's password hash is `s8OebdUUcHNvg`.  It should look something like this:

```
root:s8OebdUUcHNvg:15910:0:99999:7:::
```

Edit `/etc/ssh/sshd_config` to allow root login with no password:

```
PermitRootLogin yes
PermitEmptyPasswords yes
```

Remove the ssh host keys:

```
rm -f /etc/ssh/ssh_host*
```

Add the following to `/etc/rc.local` to have the ssh host keys regenerated upon next boot:

(thanks to https://forums.opensuse.org/showthread.php/477727-Regenerate-SSH-config-after-virtual-machine-cloning)

```
if [ ! -e /etc/ssh/ssh_host_rsa_key ]
then
    dpkg-reconfigure openssh-server
    /etc/init.d/ssh restart || true
fi
```

Don't forget to make sure `/etc/rc.local` is set executable:

```
chmod +x /etc/rc.local
```

