---
title: "The trials and tribulations of a Headless Linux Install on the Raspberry Pi"
date: 2021-03-18
---
_The problems and pitfalls of installing and using various Linux distros headlessly on Raspberry Pi and the oddyssee on which I encountered them and worked or worked not around them_

## Preamble

I recently wasted a weekend and then some, trying to find a distro for the Raspberry Pi 3B+ to accomplish very few but very clearly defined goals.  
I wanted to:

- Have an aarch64 Kernel and up to date 64bit software/libraries
- Install the distro in question headlessly
- run a networked pulseaudio server
- run a shairport-sync server in conjunction with the aforementioned pulseaudio server to get AirPlay support
- run a, (always on) Syncthing server
- run a recent version of docker for a [ Portainer ] (https://www.portainer.io/products/community-edition) and [ Nextcloud ] (https://hub.docker.com/_/nextcloud) server
- experiment with the currently developed [ ownCloud-oCIS ] (https://owncloud.github.io/ocis/) server.  
  Preferably in docker.
- be able to update the host OS and reboot the machine. Yes, this needs to be said unfortunately.

Here's the list of Distro's used and where to download them.

| Distro | Homepage | Download Page |
| ------- | ------- | ------- |
|Raspberry Pi OS Minimal (32bit armhf version)|[Link](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit)|[Link](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit)|
|Raspberry Pi OS Minimal (64bit arm64 version)|[Link](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit)|[Link](https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2020-08-24/)|
|Ubuntu Server 20.04.2 LTS 64bit Raspberry Pi version|[Link](https://ubuntu.com/download/raspberry-pi)|[Link](https://ubuntu.com/download/raspberry-pi)|
|openSUSE Leap 15.2 aarch64 Raspberry Pi 3 version|[Link](https://en.opensuse.org/HCL:Raspberry_Pi3)|[Link](https://download.opensuse.org/ports/aarch64/distribution/leap/15.2/appliances/)|
|Fedora 33 Minimal aarch64 Raspberry Pi version|[Link](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi)|[Link](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi#Downloading_the_Fedora_ARM_image)|
|Alpine Linux aarch64 v3.13 Raspberry Pi Version|[Link](https://www.alpinelinux.org)|[Link](https://www.alpinelinux.org/downloads/)|
|BalenaOS aarch64 Raspberry Pi 3 version|[Link](https://www.balena.io/os/)|[Link](https://www.balena.io/os/) -- [Link (via Cloud-Dashboard for autoconnect)](https://dashboard.balena-cloud.com/)|

{:toc}

## How the Journey started

The starting condition was the following things being set up already:
- Docker
- Nextcloud
- Portainer
- Pulseaudio
- Shairport-sync
- Syncthing
- Base OS: [ openSUSE Leap 15.2 aarch64, Raspberry Pi 3 version, JeOS variant ](https://download.opensuse.org/ports/aarch64/distribution/leap/15.2/appliances/)

**The Good Things First**  
openSUSE offers ALL of their various distros ported to a wide varietey of architectures.  
This includes the two big ones, the rolling release distro tumbleweed and the point release distro Leap just as well as the transactional distro microOS and the container host distro Kubic.

I did decide to go with Leap here as using a slow updating point-release distro seemed prudent for a server.  
Furthermore I choose the JeOS variant, which is openSUSE's equivalent to other distro's minimal variants.  
Meaning no UI.

The installation process was actually praiseworthy as the device setup networking and SSH automatically and was reachable via SSH with a [standard passwort](https://en.opensuse.org/HCL:Raspberry_Pi3) (at the time of writing root:linux) no more than 30 seconds after power on.

From here on out I was dropped immediately into the yast installer and guided through the setup.

So far everything worked as it should. Setup of my software stack was trivial. No bad surprises there.

## Where everything breaks  
A normal admin task is to update software and to reboot the system from time to time when required.  
What happened when I did so on the openSUSE install it failed to boot.

On further investigation it turns out that it now failed to find the root-fs on boot despite no UUID having changed.
In all fairness: It might or might not have had something to do with my disabling of the `snd_bcm2835` by adding it to `/etc/modprobe.d/50-blacklist.conf` and recreating the initrd via `sudo dracut -f`.  
This, however should NOT be the cause, seeing as the loading of this module can be disabled in Raspberry Pi OS, Ubuntu and Alpine Linux by means of turning it off in `/boot/config.txt`

The original reasoning behind disabling this module was to not have the internal soundcard of the Raspi show up when making the USB DAC available over the network.

After being stuck this way I backed up the important files by plugging the sdcard into my workstation and decided to give Fedora Minimal another try. A distro, it should be noted, which I already wasted half a day on, trying to get it to be setup-able over network.

## Fedora — An exercise in frustration — Sisyphus I feel your pain
First things first.  
Fedora does offer images —mainly for containerhosts and IoT— that can be pre-setup with an SSH-key to allow for immediate login upon boot. See [Fedora Docs for Fedora IoT](https://docs.fedoraproject.org/en-US/iot/ignition/) 

So far the theory. Now onto the practical.  
As described in the [docs](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi#Scripted) a setup via ssh should be possible.  
At least when using the image writer, which is only available for Fedora. Without it though, this should be possible too, seeing as ssh is being started automatically by the image, even if it is unusable seeing as the only user on the system is root and that one is locked.  
Anyway, to do what the image writer does manually, one should not need to do more than unlock the root account by modifying `/etc/passwd` and adding a key in `/root/.ssh/authorized_keys`.

Said and done. Did it work? Of course not.  
So why not? First thought: Maybe it's because of Anaconda, the installer Fedora uses.  
Now, Anaconda can be scripted using so called kickstart files. Docs [here](https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/appendixes/Kickstart_Syntax_Reference/)  
So what did I do. I did modify the kickstart file in `/root/anaconda-ks.cfg` and set the [kernel-boot](https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/advanced/Boot_Options/) flags `inst.sshd`, `inst.vnc`, `inst.vncpassword`.

Did it work?  
Nope. So obviously anaconda was not being run but some initial config script was definitely being started.

Long story short I gave up and connected a keyboard and a monitor to the pi to see if I could get it at least installed.  
That did work, right? Right?

Nope. Not a chance in hell.  
The CLI installer had me choose whatever I wanted to setup via number keys. I was only interested in unlocking the root account and setting a password as well as setting up a normal user account here.

Said and done..ehem, tried I meant.  
The root-acc dialog prompted me for a new password. No matter how often I typed it in it would just prompt for a password again. Useless.  
Maybe setting up a normal user account would work? Then I could ssh into that and `su` to root.  
Come on, at least this should work!

Once again. Nope. The hacked together dumpsterfire of a botchjob that was that lobotomized setup tool did not even allow me type in a username. It threw me into another dialog where I had to choose with num-keys what info to supply. None of the keys did anything.

O-FUCKING-KAY! Fuck this.  
I leave the installer-slash-setup tool and do it by hand. I know how to work basic UNIX commands.

Ah, yes. Wasn't there some tiny little unassuming detail I forgot about Fedora? Yes, yes there was.  
Fedora in their brilliance \<s/\> have no user account in their base images and the root account locked. 

"fOr sEcURiTtY rEaSoNs"

Yeah, right. Nothing's more secure than not being able to do anything, right? Eh. Well fuck you too, Fedora!

So after leaving the installer and being proudly presented with an utterly bricked OS I decided to leave for —hopefully— greener pastures.

## From the dark valley of Fedora to high on the mountain tops — Alpine Linux
Alpine Linux is great.  
It has up to date software is incredibly lightweight and very fast to install.

Unfortunately for my use-case I would require a persistent install rather than the standard RAM-disk one.
There are of course instructions on how to achieve this in the [docs](https://wiki.alpinelinux.org/wiki/Raspberry_Pi), despite no OOTB support for this in the Raspi image.

First things first though. Headless install, means SSH on boot. Alpine traditionally has an empty password for root so the total opposite of the monumental fuck up that is Fedora. Too bad, that there is no SSH on boot by default. Eh, easily rectified.  
According to the [wiki](https://wiki.alpinelinux.org/wiki/Raspberry_Pi_-_Headless_Installation) one has to create an `apkovl.tar.gz` file at the root of the boot disk. This file basically contains the required config of the etc directory, including the openrc service file to load and start ssh. 

Said and done. Worked like a charm with the provided file.  
Now onto the hard part. Making Alpine usable as container host.  
What is required for this? Certainly a measure of persistence, as the images/containers/volumes can't possibly fit into the RAM-Disk that is still being used. The wiki, as linked before, suggests more or less two ways of doing this. 

One is by creating a file on a second rw-mounted partition to use as overlayfs target for the directories we want to write into.  
The other is by manually making a classical sysroot install using `setup-disk`.

It is suggested to do this onto an ext4 partition using `setup disk -o /media/mmcblk0p1 /targetPartition` Implied here is `-m sys` according to the [Handbook](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/manual.html#_setup_disk) 

Apparently this is outdated though. It complains about only supporting FAT32 even after installing `e2fsprogs`.  
Furthermore it wants to unmount the ro-mounted boot disk. Done but no change. It does not work.

Not a problem, smart Ozy thought, I just put the aarch64 rootfs Alpine Linux so graciously provides as download on the persistant partition and copy over the relevant files from /etc.  
Said done. `/etc/fstab` and `cmdline.conf` on the boot-partition adapted and rebooted. 

Bricked Pi, you've become a far too frequent visitor for my liking.

To be fair though. I did not check if it did boot correctly. It didn't start SSH so there was that. Might have needed to remove the loopback or squashfs arguments from the cmline.  
According to the wiki though, all that should have been needed was adding `root=/dev/mmcblk0p2`

Anyway. That was failure number 3.  
Would be nice if it had worked. 

## Back to the roots and I hit the ground running — Raspberry Pi OS, 32bit Edition
Install was easy.  
The [docs](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md) provide a very easy tweak to get it to be reachable over ssh on boot.  
Which consists of simply creating an empty file called `ssh` on the Boot partiion.

Said and done. Now there's two problems. One potentially being more of one than the other.  
One, the 32bit Kernel  
Two, the Docker Version 18.09 instead of 20 something like on openSUSE or Alpine Linux.

The first is easily fixed. As described in the [docs](https://www.raspberrypi.org/documentation/configuration/config-txt/boot.md) adding `arm_64bit=1` to `/boot/config.txt` makes the Pi boot the kernel8.img which is a aarch64 kernel.

What this enables me to do is —in theory— run aarch64 based Docker images on the armhf docker client.  
It also enables me to run 64bit ARM distros like Alpine using systemd-nspawns, which can be installed with the `systemd-container` package.

Problem with this second option is that running docker inside of these kinds of containers is anything but trivial.  
While there are some guides on how to do it (see [Archwiki](https://wiki.archlinux.org/index.php/Systemd-nspawn#Run_docker_in_systemd-nspawn)) these don't really work as they are. Docker requires A LOT of lowlevel access to all kinds of system interfaces and kernel capabilities.

The first one can be solved with blanket flags like `--capabilities all` and adding exception to the syscall-filter using the `--syscall-call-filter` flag. All in all it will fall apart though. Unless a proper bridge interface is setup as to give the container full access to it's own `/proc/net` tree. Even after that however, the docker requires full permissions to at least it's own cgroup. This means it might be required to bind-mount `/sys/fs/cgroups`in there too which might or might not fail.

All in all this a whole other can of worms. Suffice to say after openSUSE, Fedora and Alpine I had no patience left for it other than a couple of quick tries. Apparently with LXD there are proven working ways too, according to user "vaab" on stackoverflow ([link](https://stackoverflow.com/a/25885682))

As LXD is not in the Raspberry Pi OS repos I opted for installing the snapd daemon and using the LXD snap as I've done in the past.  
Which was promptly slapped in my face with an http error indicating missing resources. Seems Canonical stopped supplying 32bit snaps to other distros, seeing as their own products stopped using them.

I could have stopped it here but I didn't want to accept a downgrade from openSUSE Leap at this point. Especially not seeing as there was the distinct possibility of ownCloud-oCIS' Docker-container not working, in which case I'd have to try the standalone binary, which —in all likeliness— might have gotten problems if run on a 64bit kernel but 32bit userland.

Sure I could have just used an Alpine nspawn just for this but damnit! This is a matter of principle!

## Why does it always rain on me? — BalenaOS, Cloud I guess.
I was promised a "write to sdcard, power on, connect, go"-workflow for this one. Did it deliver?
BalenaOS is based on 

Short answer: yes.  
Long answer: no. Yes it did. Especially if the image is downloaded via the cloud console. That way it automatically connected to my balena cloud account. Would have been nice if it was actually meant to be used for my use-caes. It wasn't though. Balena Cloud is meant to build docker images in the cloud or on end-devices and deploy them on various IoTs or SBCs. Problem is, that there is no way to properly configure or deploy the kind of images I want to run. Things like a Nextcloud+nginx stack is just not deployable with the tools given. Especially as even for communication with the node directly, Balena recommends the use of their nodejs based cli tool. There simply is no way to easily deploy compose files.

So in short: This one was a failure as well.

## Climbing the stairway to heaven — Raspberry Pi OS, 64bit Edition
This one was tricky to get as it is not available on the website. The website only shows the 64bit desktop variant.  
Cutting the offical download link down to [https://downloads.raspberrypi.org/](https://downloads.raspberrypi.org/) Leads to the full directory though.  
Here I downloaded raspios-buster-arm64.

Same setup as on the 32bit version and this time no messing around with containers. The 18.09 version of docker works just fine and ownCloud-oCIS binary worked just fine too.

## Closing words
After a day of pain I arrived at a reliable system that survives updates, supplies firmware updates, and is natively 64bit.

More details on the specific configurations of the different services listed at the beginning of this post in a later post.

Cheers!
