---
title: "A Primer on X-Remoting with VNC, RDP, XDMCP and SSH"
---

**Tools used:**
- vncserver
- xrdp
- xdm(cp)
- xpra
- ssh

**Testplatforms - Server**
A headless openSUSE VM and a headless FreeBSD VM without xserver installed.

**Testplatforms - Client**
Any System with an Xserver and Xephyr installed. In this case a MacBook with macOS 10.12.6

There are different ways of remoting X-Applications from a server to a client. All with their own set of advantages and disadvantages. This article is comparing some of them and will explain how they are set up.

For most users who remotely access their Linux or BSD machines SSH will be enough. There is an easy way however to start applications on the host and have their GUI appear on the local machine.

**X-Forwarding over SSH** 

`ssh -X` and `ssh -Y`respectively allow for this. The difference between them is a reliance on `xauth`which might not be present on a pure server install without any X-Tools.

if `xauth` is present on the server the `-X` flag prompts `ssh` to create a temporarily valid session token in the `xauth` database which allows access to of the remote Xserver.
For X-Forwarding to work the server needs to have the `ForwardX11` flag enabled in it's sshd_config file under /etc/ssh.
Unless `ForwardX11Trusted` is also enabled the access to the local Xserver granted to the remote server is handled as untrusted and restricted to the main `DISPLAY`. The session token in `xauth` expires 20min after the session is closed. This can be modified by setting a value for the ForwardX11Timeout flag in sshd_config (see `man ssh_config`).

If `xauth` is not present and therefore the creation of a 'MAGIC COOKIE' for the current session is not possible X-Forwarding may be enabled via the `-Y` flag in SSH which does not make use of the X11 SECURITY extension (see `man ssh`). Since xauth's authentication mechanism is inherently flawed and weakly encrypted using `-Y` offers better security.

Advantages of X-forwarding over SSH are among others, that no window manager and no XServer are required as the window manager and Xserver of the host are used instead. There is no loss in image quality or resolution either as the settings of the local Xserver are used for this and, given the bandwidth, the application feels nearly local when used. Disadvantages are relatively high bandwidth requirements as the SSH protocol does not support UDP and the video stream is not compressed. It is for this very reason that some of the protocols in this article cannot be tunneled over ssh as they require UDP.
SSH requires TCP port 22 to be open.

**xpra**
… is probably the most sophisticated X-Remoting tool on the list. Xpra acts as window manager and Xserver on the server and allows forwarding of not only single applications and full desktop sessions but allows for this in a plethora of   different ways. xpra can be accessed via tcp-sockets, ssh, ssl, tcp-aes and even over webrtc via browser or via memory mapped files on shared drives.
xpra also allows to forward GL-accelerated applications sound and devices and can make use of hardware accelerated decoding and encoding of the video stream in different formats. In it's simplest form xpra does not require any configuration and can be started from the client via `xpra start ssh/user(at)server-ip --start-child=name-of-program`.

An advantage is that xpra can use common video codecs like h264 to reduce bandwidth required.  
One disadvantage of xpra is it's long dependency-list. It is more suited for remoting of Desktops than servers.

**VNC**
The **V**irtual **N**etwork **C**omputing protocol is the last in the list that can be run over SSH. VNC on Linux requires either a running Xserver to attach to using tools like x11vnc or the special Xvnc Xserver that creates VNC accessible virtual sessions.
**Note:** on FreeBSD Xvnc is neither in `pkg` nor in `ports` it is available, however as part of the `tightvnc` package.
To use Xvnc one usually uses the bundled vncserver tool. On first run it asks for a password to be set and from then on runs in the background listening for incomming VNC connections on ports 5900 to 5900+highest display number.
Depending on the configuration VNC creates a new session for each connection or allows connection to one session by multiple participants. To kill a session run `vncserver -kill :<sessionnumber>`
Xvnc, by default, looks for `$HOME/.vnc/xstartup` instead of  `$HOME/.xinitrc`. To use your users xinitrc replace xstartup with a symbolic link accordingly.

**xrdp**  
The **R**emote **D**esktop **P**rotocol was developed by Microsoft and ported to Linux and FreeBSD in the form of `xrdp` 
xrdp comes in two parts. the xrdp service and the `xrdp-sesman` the session manager.
RDP differs from VNC insofar as it's not intended to attach to an existing session like `x11vnc` allows for but it intended to start a new session per connection. xrdp listens on tcp port `3389`
to automatically start xrdp at boot do `systemctl enable xrdp` and `systemctl enable xrdp-sesman` as root on linux.
On FreeBSD add `xrdp_enable="YES"` and `xrdp_sesman_enable="YES"` to /etc/rc.conf
xrdp, by default, looks for `$HOME/.Xsession` by default. Change this to your .xinitrc in `/etc/xrdp/sesman.ini` for linux and `/usr/local/etc/xrdp/sesman.ini` in FreeBSD

**XDMCP**
… is a remoting protocol for the X display manager (xdm).  
xdm's config can be found in /etc/X11/xdm on linux and /usr/etc/X11/xdm on FreeBSD.  
Important files are Xaccess, Xservers, Xsession and xdm-config.

xdm-config defines what files to use and is used to enable XDMCP.  
Xaccess functions as access control list (ACL) for clients.  
Xservers defines what X-servers to start at which tty. If the server is only used to remote into Xorg is not needed. Xvnc will do. An example line in Xservers might be `:0 local /usr/bin/Xvnc -nolisten tcp -br`
Finally, make sure to have a line utilizing your .xinitrc in Xsession if you use .xinitrc.

Xstartup, Xsession, Xreset, Xresources and Xwilling aren't usually needed to be modified. These files handle Xresources and processes preceeding and superceeding the start of the user session such as the start of the chooser.

To query the X display manager from a remote machine, be it directly or indirectly, or have it broadcast its presence on the network the line `DisplayManager-requestPort: 0` has to be commented out: `!DisplayManager-requestPort: 0` in xdm-config. xdm listens on UDP port 177 for xdmcp requests.

Xaccess allows to set filter lists of who can acess xdm over xdmcp in what way. On trusted networks you usually want these two lines active.

```ini
* #any host can get a login window
* CHOOSER BROADCAST #any indirect host can get a chooser
```


