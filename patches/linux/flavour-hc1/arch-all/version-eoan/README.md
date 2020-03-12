# Patchset to build Linux Kernel 5.3 for the Odroid HC1
These patches are a re-working and attempt to backport of the work on the 5.4
Linux kernel by mihailescu2m.  They take the source from the 5.4.y branch here:
https://github.com/mihailescu2m/linux/tree/odroidxu4-5.4.y, extract the changes
by comparing mihailescu2m's source with the same version of the Torvalds
source.  Then these changes are applied to the 5.3 kernel source.

GPU and Media drivers are included for completeness, even though the are unused
on the HC1.

Some changes related to "CHIPID" are omitted as the changed files don't exist
in the 5.3 source.
