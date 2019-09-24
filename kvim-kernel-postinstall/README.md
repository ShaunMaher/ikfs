The script in this folder takes a kernel image, an initramfs and a device tree
and compiles from them a FIT image suitable for booting from U-boot.

Ultimately it should be included in the kernel package and dropped into the
/etc/kernel/postinst.d directory.  It still needs a bit of work first though.
