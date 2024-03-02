---
title: "Cross-compiling for Raspberry Pi on an Apple silicon Mac"
date: 2024-03-02
---

While developing the [device software][454-sw] for [454 Bio][454]'s DNA sequencer, I needed to be able to build [rpicam-apps][rpicam-apps] and our [HAL][454-hal]. While I could compile directly on the Pi 4 it was running on, it was quite slow:

```text
aarmea@raspberrypi:~/Code/tirf-hal/builddir $ meson compile --clean && time meson compile
Cleaning... 0 files.
[70/70] Linking target apps/capture-service
    
real    4m12.382s
user    14m38.012s
sys     1m8.128s
```

Instead, I set up my much faster M1 Mac to cross-compile this and our custom code, which is almost 10x faster and allowed me to iterate much more quickly:[^clean-build]

[^clean-build]: I know a clean build is a little contrived here, but it really adds up. This is the difference between an incremental build taking a few seconds versus a minute or more.

```text
debian@debian:~/Code/454-hal/builddir$ meson compile --clean && time meson compile
Cleaning... 0 files.
[70/70] Linking target apps/capture-service

real    0m32.847s
user    1m46.109s
sys     0m14.082s
```

Because the environment is virtualized, it was much easier to set up and integrate Pi-specific libraries than with other cross-compilation toolchains.

[454]: https://454.bio
[454-sw]: https://454.bio/docs/build/instrument-sw/
[454-hal]: https://github.com/454bio/tirf-hal
[rpicam-apps]: https://github.com/raspberrypi/rpicam-apps

<!--more-->

## tl;dr

In a nutshell, we'll be doing the following:

1. [Create an equivalent `aarch64` Debian VM](#initial-setup)
2. [Install application-specific dependencies in it](#installing-application-specific-packages), [including packages exclusive to Raspbian](#installing-pi-only-packages-in-the-vm)
3. [Compile as usual (but faster)](#compiling)

## Initial setup

This is possible for a few reasons:

* Raspbian is derived from Debian
* Raspberry Pi 3 and newer and Raspberry Pi Zero 2 and newer use [64-bit ARM processors][pi-aarch64]
* Raspbian is available as a 64-bit build[^32-vs-64]
* Apple silicon processors are also 64-bit ARM

[pi-aarch64]: https://www.raspberrypi.com/news/raspberry-pi-os-64-bit/

To get started, first install a virtualization platform on your Mac. I recommend the free and open-source [UTM][utm], which also offers [prebuilt Debian images][utm-gallery].

[utm]: https://mac.getutm.app/
[utm-gallery]: https://mac.getutm.app/gallery/

After installing UTM, install the version of Debian that corresponds to the version installed on your Pi. The rest of this guide assumes 64-bit Bullseye, but Bookworm should work as well.

[^32-vs-64]: If your Pi is running a 32-bit operating system, I highly recommend upgrading to 64-bit instead. Support for 32-bit builds is outside of the scope of this article as it will require either using an emulated VM (much slower) or installing a proper cross-compilation environment inside a 64-bit VM (more complicated).

Adjust the VM's settings before booting up. 4 cores with 6 GB of RAM was a good balance on my MacBook Pro with 16 GB of RAM.[^force-multicore]

![VM settings showing 4 cores with 6 GB of RAM](/post/x-compile-pi-arm-mac/vm-settings.png)

[^force-multicore]: "Force Multicore" doesn't make a difference in this scenario because UTM is using Apple's hypervisor for the processor rather than emulating via QEMU.

Then, boot up the VM and open up the terminal to do some basic housekeeping:

```bash
# Initial setup
sudo apt update
sudo apt upgrade

# Convenience packages
sudo apt install htop tmux mosh git build-essential rsync
```

### Better integration via "remote" access

If you're comfortable working inside the VM's GUI, feel free to skip this section. If not, you can access the VM over SSH.

First, retrieve the VM's IP address on UTM's shared interface:

```text
debian@debian:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 3c:13:d6:23:10:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.4/24 brd 192.168.64.255 scope global enp0s1
       valid_lft forever preferred_lft forever
    ...
```

In my case, the relevant interface is `enp0s1` and the private IP address is `192.168.64.4`. This is static and can be entered in to your Mac's SSH config at `~/.ssh/config`:

```text
aarmea@Alberts-MacBook-Pro x-compile-pi-arm-mac % cat ~/.ssh/config
Host *
	AddKeysToAgent yes
	UseKeychain yes

Host pi-vm
	HostName 192.168.64.4
	User debian
```

Then, use `ssh-copy-id pi-vm` to use your SSH keys to log into the VM and bypass the password prompt every time.

With that complete, you can now directly `ssh pi-vm` and use related tools, like `scp` and `rclone`, and even connect to it using Visual Studio Code.

{{< rawhtml >}}
  <video autoplay playsinline loop muted class="playpause-with-visibility">
    <source src="/post/x-compile-pi-arm-mac/vscode-remote.mov">
  </video>
{{< /rawhtml >}}

## Installing application-specific packages

I particularly needed to be able to build `rpicam-apps`, so I then installed the subset of [its dependencies][official-rpicam-apps-building] that were common to both Debian and Raspbian:

[official-rpicam-apps-building]: https://www.raspberrypi.com/documentation/computers/camera_software.html#building-libcamera-and-rpicam-apps

```bash
sudo apt install -y python3-pip
sudo apt install -y libepoxy-dev libjpeg-dev libtiff5-dev
sudo apt install -y qtbase5-dev libqt5core5a libqt5gui5 libqt5widgets5
sudo apt install -y libavcodec-dev libavdevice-dev libavformat-dev libswresample-dev
sudo apt install -y cmake libboost-program-options-dev libdrm-dev libexif-dev
sudo apt install -y libboost-dev
sudo apt install -y libgnutls28-dev openssl libtiff5-dev
sudo apt install -y qtbase5-dev libqt5core5a libqt5gui5 libqt5widgets5
sudo apt install -y meson
sudo apt install -y libglib2.0-dev libgstreamer-plugins-base1.0-dev
sudo apt install -y libpng-dev
sudo pip3 install jinja2
sudo pip3 install pyyaml ply
sudo pip3 install --upgrade meson
```

### Installing Pi-only packages in the VM

In my case, 454's HAL requires `libcamera-dev` and `libcamera0` to use the camera and `libpigpio1` and `libpigpio-dev` for GPIO.

The Pi-specific packages are a little trickier as they are not directly available from the Debian repositories. If you try to install one directly, you'll see an error like this:

```text
debian@debian:~$ sudo apt install libpigpio1
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
E: Unable to locate package libpigpio1
```

Instead, these will have to be downloaded manually from the relevant Pi repository and installed manually[^x-distro-considered-harmful] using `dpkg`. Manually copying packages forces you to think about the dependency tree, which is very important across distributions. If you're not careful, you could end up breaking your VM by installing an incompatible `glibc` or similar.

[^x-distro-considered-harmful]: Normally, [installing packages compiled for a different Linux distribution is a Bad Idea™][x-distro-so]. In this case, Raspbian and Debian are both very closely related and stable (i.e. slow to update but also slow to break things) distributions.

[x-distro-so]: https://superuser.com/questions/337845/is-it-safe-to-install-ubuntu-packages-on-debian


If you know what you're doing, you might be able to get away with using `add-apt-repository` on the VM with the Pi repository at <http://archive.raspberrypi.org/debian>.

#### Obtain the URL

On the Pi, run `apt-cache show $PACKAGE_NAME` and look for the line that begins with `Filename`:

```text
aarmea@raspberrypi:~ $ apt-cache show libcamera-dev
Package: libcamera-dev
Source: libcamera
Version: 0~git20230720+bde9b04f-1
Architecture: arm64
Maintainer: Debian Multimedia Maintainers <debian-multimedia@lists.debian.org>
Installed-Size: 138
Depends: libcamera0 (= 0~git20230720+bde9b04f-1)
Multi-Arch: same
Homepage: https://libcamera.org/
Priority: optional
Section: libdevel
Filename: pool/main/libc/libcamera/libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb
Size: 24388
SHA256: 50f4043be060f0dea24e4aa70ea8e6a0ce3e65cd4bd074ed296c33d0304db983
SHA1: 7939d7515b72741448251f0d6dcdb18c51300dfe
MD5sum: 87016da23324afd025c12a0d7b89a9eb
Description: complex camera support library (development files)
 libcamera is a complex camera support library which handles low-level
 control of the camera devices, providing a unified higher-level
 programming interface to the applications.
 .
 This package provides the necessary files needed for development
Description-md5: a817fddde055009800fbb7c2566de922
```

Append this line to the Pi repository root and you'll get a URL like the following:

<http://archive.raspberrypi.org/debian/pool/main/libc/libcamera/libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb>

#### Download and install the package

Back on the VM, download the package using `wget` with this URL, and install it using `dpkg`:

```text
debian@debian:~/Downloads $ wget http://archive.raspberrypi.org/debian/pool/main/libc/libcamera/libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb
--2024-03-02 09:17:08--  http://archive.raspberrypi.org/debian/pool/main/libc/libcamera/libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb
...
2024-03-02 09:17:08 (213 KB/s) - ‘libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb’ saved [24388/24388]
debian@debian:~/Downloads$ sudo dpkg -i libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb 
Selecting previously unselected package libcamera-dev:arm64.
(Reading database ... 105377 files and directories currently installed.)
Preparing to unpack libcamera-dev_0~git20230720+bde9b04f-1_arm64.deb ...
Unpacking libcamera-dev:arm64 (0~git20230720+bde9b04f-1) ...
dpkg: dependency problems prevent configuration of libcamera-dev:arm64:
 libcamera-dev:arm64 depends on libcamera0 (= 0~git20230720+bde9b04f-1); however:
  Package libcamera0 is not installed.

dpkg: error processing package libcamera-dev:arm64 (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 libcamera-dev:arm64
```

As you run, make note of `dpkg` errors of the form "Package ... is not installed". If that package is not in your list of packages to install and you don't suspect it will cause issues, add it to your list and keep going.

#### Validate

Once you believe you have installed all of the Pi-specific packages for your application, ensure  that APT is in a consistent state by running `sudo apt install -f`. If it only attempts to install packages from the Debian repository, allow it and you are good to go.

Be careful: if it instead tries to *remove* packages...

```text
debian@debian:~/Downloads$ sudo apt install -f
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Correcting dependencies... Done
The following packages will be REMOVED:
  libcamera-dev
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
1 not fully installed or removed.
After this operation, 141 kB disk space will be freed.
Do you want to continue? [Y/n] n
Abort.
```

... you are likely missing a package from the Pi repository, and proceeding will undo some of your work. Double-check that all of the packages and their dependencies are installed before re-running. Tools like `aptitude`, run on both the Pi and VM, may make it easier to identify the missing packages.

For what it's worth, the full list of packages that `tirf-hal` requires from the Pi repository is:
* `libcamera-dev`
* `libcamera0`
* `libpigpio1`
* `libpigpio-dev`
* `raspberrypi-kernel` -- note that simply installing the kernel does not configure GRUB to boot from it by default. I haven't tested booting the Pi kernel in the VM and also don't recommend it.

## Compiling

I patterned `tirf-hal` around `rpicam-apps` so I could directly reference its headers in my code. As a result, it also uses `meson` as its build system and all of the normal commands work after cloning the repository:

```text
debian@debian:~/Code $ git clone git@github.com:454bio/tirf-hal.git
Cloning into 'tirf-hal'...
...
debian@debian:~/Code $ cd tirf-hal/
debian@debian:~/Code/tirf-hal $ git submodule update --init --recursive
...
Submodule path 'subprojects/rpicam-apps': checked out '7e4d3d71867f60f5398687180972798baad85f1b'

debian@debian:~/Code/tirf-hal $ meson setup builddir
The Meson build system
Version: 1.3.2
Source dir: /home/debian/Code/tirf-hal
Build dir: /home/debian/Code/tirf-hal/builddir
...
Build targets in project: 14
                    
libcamera-apps 1.2.1 
                    
  libcamera                    
    location             : /usr/lib/aarch64-linux-gnu
    version              : 0.0.5
                    
  Build configuration              
    libav encoder        : YES
    drm preview          : YES
    egl preview          : YES
    qt preview           : YES
    OpenCV postprocessing: NO
    TFLite postprocessing: NO
                    
454-hal 7            

  Subprojects
    rpicam-apps: YES

Found ninja-1.10.1 at /usr/bin/ninja
debian@debian:~/Code/tirf-hal $ cd builddir
debian@debian:~/Code/tirf-hal/builddir $ meson compile
[70/70] Linking target apps/capture-service
```

## Deploying to the Pi

The final step is to deploy the compiled binaries to the Pi. I like to use `rsync` for this so I can easily and efficiently copy everything to the Pi over the network:

```text
debian@debian:~/Code/tirf-hal/builddir$ rsync -avzdP ./ root@$YOUR_PI_HOSTNAME:/454/hal/
...
```

Then you can run your code:

```text
aarmea@raspberrypi:/454/hal/ $ ./apps/capture-service
Usage:
./apps/capture-service ...
```
