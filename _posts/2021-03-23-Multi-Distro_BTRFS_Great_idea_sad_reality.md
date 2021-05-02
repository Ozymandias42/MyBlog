---
title: Multi-Distro BTRFS a pipedream
date: 2021-03-23
published: false
---

_BTRFS to use for Multi-Distro Boot purposes and why it fails._

So I was trying to reconfigure the subvolume layout of my openSUSE BTRFS installation to adapt it to a Layout more suited to multi-distro system.

The idea was to implement a hierarchy like this:
```
.
..
boot/grub2/i386-modules
boot/grub2/x86_64-modules
home
distros/
  distros/openSUSE
    distros/openSUSE/.snapshots
        distros/openSUSE/.snapshots/1/snapshot  <-- contains rootfs
        distros/openSUSE/.snapshots/2/snapshot
    distros/openSUSE/boot
    distros/openSUSE/root
    distros/openSUSE/var
flatpaks
gnu
nix
opt
usr/local
srv
swap/swapfile
```

from what it looks like by default on openSUSE
```
.snapshots
boot
home
opt
root
srv
tmp
usr/local
var
```

Now, this seems easy in theory. So what would I have to do?
All I'd have to do is to

- move/reflink-copy the .snapshots folder to the distros/openSUSE subfolder
- move/reflink-copy /var as well

So this would be enough?  
Well not quite. To understand why not we first need to look at how Grub2 works on UEFI.
The actual grub2-bootloader executable loaded by the BIOS/UEFI resides on the EFI System Partition, ESP for short.
It's usually located somewhere inside the /boot/efi directory. In this case /boot/efi/opensuse/grubx64.efi  
By default grub usually looks for a grub.cfg in the same directory as the executable.  
As such there is also grub.cfg in there.
The contents of this file look sth like this:
```
set btrfs_relative_path="yes"
search --fs-uuid --set=root 65f64868-6d72-497b-902b-9ef7b9bf11e2
set prefix=(${root})/boot/grub2
source "${prefix}/grub.cfg"
```

What we can see from this is, that it points to the btrfs partition to then sources -speak load- the actual grub.cfg from the actual /boot directory on there.

What isn't immediately apparent here is, that loads the btrfs partition in general and not a specific subvolume.

Now on a clean btrfs partition, as well as with other btrfs distros, this would mount the root-subvolume with the id 5.

Not so on openSUSE.  
openSUSE makes use of the ability to change the default volume to any subvolume.  
It does so to the active snapshot, speak the one used as fsroot, in \<subvolid 5\>/.snapshots/\<N\>/snapshot.

This means that the grub.cfg with the menu and entries is actually being loaded from one of the snapshots.

For openSUSE alone this is not a problem as the grub.cfg for the one distro I want to boot to is always available. For a multiboot-system this would mean however, that entries for other distros or OS' would need to be added via openSUSE via `/etc/grub.d/40-custom.cfg` 

These entries would need to be config load entries for the grub.cfg files of other distros. Especially those with bootable snapshots.

If openSUSE did not change the btrfs default-root on every snapshot rollback, one could simply set it back to 5 or a special boot subvolume with a manually made grub.cfg whose only job is chainloading distro specific other grub.cfg's from the respective distro's /boot directories.

To get around this issue, it would be required to change the snapper scripts changing the default-root to stop doing so and instead change the grub.cfg of the openSUSE /boot directory.

Only then could a multi-boot enviroment for multiple btrfs snapshots using distros be realised.
