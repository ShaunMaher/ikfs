# Install Kernel From Source
## The Problem
I've been a very happy Ubuntu user for several years both on the desktop and
server.  I'm also a big fan of ZFS, at least until bcachefs gains a few more
features.  A couple of recent developments have made me want to push ahead of
what Canonical might consider stable.
* ZFS Native Encryption - No more need for a separate LUKS layer
* The Linux Kernel developers decision to deny access to some FPU functionality
  for non-GPL kernel modules.  This causes a massive ZFS performance drop.

This project's goal is to produce a kernel (and some additional support
packages) that is a combination of Canonical's latest Kernel version and kernel
configuration, a patch that restores the FPU functions to non-GPL modules and
the latest experimental ZFS module source from upstream Debian.

As an additional bonus, I'm also interested in all the generated components
being cryptographically signed so that, in combination with Secure Boot, the
entire startup environment is secure.

***Update: 1*** The developers at ZFS-on-Linux have completed modifications to
allow ZFS to operate at optimal performance without needing the blocked FPU
functionality.

***Update: 2*** Ubuntu recently announced that Ubuntu 19.10 may have ZFS root
filesystem support in the installer.  The Eoan (19.10) kernel source tree is
also using ZFS 0.8.1 which includes encryption support.  I have adjusted the
build process so that using the Debian ZFS source package is now optional.

By the time I get this project finished, 20.04 will be released, with ZFS on
root and native encryption, the project will be redundant.

## Initial Setup
### GPG for signing your repository files
TODO

### Kernel and Module Signing for Secure Boot
If no kernel signing certificate is provided, a new certificate is generated
during the build and this new certificate is used to sign the kernel and
modules.  If you have no interest in Secure Boot, you need do nothing more.

If, however, you would like a kernel that can be used for Secure Boot, you need
to provide a suitable public and private key pair.

We are going to use the GPG keys generated earlier to encrypt the signing keys.
If you skipped the GPG setup phase from earlier, please circle back and do it
now.

Copy the file `x509.genkey` from the `templates` directory into a temporary
location and then open it in a text editor.  You might want to personalise the
key a little by customising (and uncommenting as necessary) the values in the
`req_distinguished_name` section.

```
openssl req -new -nodes -utf8 -sha512 -days 36500 \
		-batch -x509 -config x509.genkey \
		-outform PEM -out signing_key.pem \
		-keyout signing_key.pem
```

Now encrypt the signing key so that it can only be used in the presence of your
unlocked personal GPG private key.

```
gpg -a --yes --output signing_key.pem.gpg --local-user shaun@ghanima.net -r shaun@ghanima.net --sign --encrypt signing_key.pem
```

## Using the repository
### Hosting the repository with as a simple HTTP service
Download Minio: TODO
```
mkdir repository/.minio
minio --config-dir ./repository/.minio server ./repository
mc policy download localhost/ubuntu
```

### Add your repository to a machine
```
echo "deb [arch=amd64] http://172.30.0.201:9000/ubuntu bionic main contrib non-free" | sudo tee /etc/apt/sources.list.d/signed-kernels.list
wget -O- http://172.30.0.201:9000/ubuntu/signing_key.gpg | sudo apt-key add -
```

## Known issues and TODOs
### Package Info
The generated packages include the default Canonical meta-data so they look like
they are official packages (at least as far as maintainer email address, etc.).
Ideally, we should update the changelog file to include our own contact details
and to make it clear that these are not real Canonical packages.

### Package Names
The generated Kernel image package is called "-unsigned", even though it is
signed.

In the interests of, again, clearly marking these packages as not being
real Canonical provided packages, it would be great if we could change
"-generic" to something like "-personal" (now partially implemented but
untested)

I would also like to have a series of patches that are only applied to certain
kernel "flavours".  For example, I'd like a flavour called "kvim" and only this
flavour gets the Khadas Vim patches.

### DKMS Module Signing issue (worked around)
```
II: dkms-build installing zfs into /usr/src/linux/debian/linux-modules-5.0.0-20-generic/lib/modules/5.0.0-20-generic/kernel/zfs
signing zfs.ko
At main.c:160:
- SSL error:02001002:system library:fopen:No such file or directory: ../crypto/bio/bss_file.c:72
- SSL error:2006D080:BIO routines:BIO_new_file:no such file: ../crypto/bio/bss_file.c:79
sign-file: /usr/src/linux/debian/build/build-generic/certs/signing_key.pem: No such file or directory
debian/rules.d/2-binary-arch.mk:112: recipe for target 'install-generic' failed
make: *** [install-generic] Error 1
```
This is because the signing_key.pem file doesn't exist in /usr/src/linux/debian/build/build-generic/certs/.
signing_key.x509 exists in this location.  What copies signing_key.x509 but does
not copy signing_key.pem?

The above is triggered by line 150 of: https://kernel.ubuntu.com/git/ubuntu/ubuntu-disco.git/tree/debian/scripts/dkms-build#n150

$sign refers to the third argument passed to that script by https://kernel.ubuntu.com/git/ubuntu/ubuntu-disco.git/tree/debian/rules.d/2-binary-arch.mk
on line 63. which in turn points back to the build_dkms_sign function in https://kernel.ubuntu.com/git/ubuntu/ubuntu-disco.git/tree/debian/rules.d/2-binary-arch.mk
on line 54

#### Workaround
The file `patches/linux/arch-all/version-all/0003-signing_key.pem_missing_in_dkms-build.patch`
patches one of the build scripts so that the missing file is copied before it is
needed.  This is a bit of a hack.

## Secure boot notes
Most sources say that if you want secure boot, UEFI without Grub and signed
initramfs you need to have your initramfs ready at kernel compile time.  This
would be super inconvieient if you have multiple systems wanting to use the same
kernel but be able to have their own initramfs.  You also need to have the
kernel command line baked into the kernel which won't work if you have multiple
systems with different rootfs paths.

The following article suggests that you can combine kernel, initramfs and
commandline into a single signed file after compile time:
https://bentley.link/secureboot/

## History - So I don't try to fix things the same incorrect way twice
### Alternatives to using gcc-8 for builds of the eoan kernel
The error message about your compiler not supporting retpoline is mis-leading.
The gcc-7 (7.4.0) compiler does support the initial retpoline fixes but not the
"fcf-protection=none" fix.  As far as I can tell, none of the gcc-7 compilers
do.

I did attempt to use a PPA that provided more recent versions of gcc-7 but this
did not resolve the issue.
