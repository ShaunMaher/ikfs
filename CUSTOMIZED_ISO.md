# Customising the installer CD by adding your new kernel
I'm honestly not sure this is worth the effort.  The installer (Desktop or
Server) doesn't offer to install to ZFS datasets anyway.

One guide I read had the user install into a EXT4 formatted zvol and then copy
the installed contents into a ZFS dataset.

https://help.ubuntu.com/community/LiveCDCustomization

# Probably easier way
I took two USB sticks.  On one I put the Ubuntu Desktop (to be more honest, the
KDE Neon) installation ISO (simple DD operation).  I then booted the target
machine with the installation stick and installed an OS onto the other USB
stick.  Reboot into the OS installed on the second USB stick (by removing the
installation stick when prompted during restart).

Add the repository that ikfs creates and install the customised kernel and zfs
packages.  This is pretty slow since we're running off a usb stick.  Reboot to
get the new kernel.

Create the desired zpool and zfs datasets, using encryption as desired.

Use debootstrap to install a base into the target zfs dataset.

Chroot into the target dataset and install the ubuntu-server metapackage.
Install our repo.  Update and install our custom kernel.

Stuff about with grub.

Reboot removing the second USB stick.  Happiness.  If anything goes wrong, boot
off the second USB stick (since it has the updated kernel) and fix things.

## Frankenstein together the existing initrd with the new one
```
mkdir orig
mkdir new
cd orig/
zcat ../../extract-cd/install/initrd.gz.dist | cpio -i
cd ../new
zcat ../initrd.gz | cpio -i+
cd ../orig
cd lib/modules
cp -r ../../../new/lib/modules/* .
cd ../etc
cp -r ../../new/etc/zfs .
cd ../sbin
cp ../../new/sbin/* .
cd ../lib
cp ../../new/lib/* .
cp -r ../../new/lib/x86_64-linux-gnu/* x86_64-linux-gnu/
sudo bash -c "find . | cpio -o -H newc | gzip -9 > ../initrd.gz"
```

At the end of this process I could create an ISO image that booted the desired
kernel and launched the Ubuntu Server installer.  The installer couldn't find
the kernel modules for the running kernel (because the CD will have the modules
for the original kernel baked into the filesystem.squashfs which I didn't
modify), so I aborted.  I was doing this in a VM.

I also couldn't get the generated .ISO to boot when DDed to a USB stick.  This
might be fixable but I struggle to see the point in doing so at this point.
