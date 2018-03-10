---
layout: post
title:  "Antergos/Arch Linux on MacBook Pro 11,4"
date:   2017-12-09 10:00:00
categories: antergos arch linux ubuntu macbook pro 11,4
comments: True
permalink: posts/antergos-mbp
shortUrl: https://goo.gl/8L2is7
is_post: true
---

I've been wanting to run a Linux distro on my MacBook Pro for about 6 years
now. About a year ago I decided to give it another go and had a lot of success.
After doing some research on the topic I came to the conclusion that
[Arch Linux][arch] is the most promising distro, at least in terms documentation
for MacBook Pros. To make the installation
easier I decided to go with [Antergos][antergos] for the installation process.
This post will cover replacing macOS on your MacBook Pro 11,4. After following
this post you will only have Antergos installed on your system.

### Pre-installation

The first thing
you should do is make a bootable clone of your internal SSD to an external
hard drive. For this you can boot into recovery mode (hold Command+R on startup)
and use Disk Utility's "Restore" feature. Make sure you have the correct source
and destination selected, that is, ensure you don't erase your internal drive
before making
a backup. Making the backup may take a while-- for me it took approximately two
hours for 190GB to an old hard drive over Firewire 800. I would recommend
verifying that you are able to boot from the external drive, just to be on the
safe side. Another prerequisite is either having a thunderbolt to
ethernet adapter or downloading and copying the wifi driver to a USB stick.
I have not done the latter and cannot offer guidance on it.

With at least a bootable backup in place, let's get to installing Arch Linux.
First you should boot into macOS and turn the volume all the way down. This will
disable the startup sound, which will no longer be possible once Antergos is
installed. Now download an Antergos ISO and copy it to a USB drive. I would
recommend getting the full live image, which still fits on a 2GB flash drive.
You can find instructions on how to create a bootable USB flash drive from
the [Antergos website][antergos_usb]. Once the USB drive is ready, reboot
from macOS (again, make sure your volume is off) and hold down the left
alt/option key to see a list of EFI devices you can boot from. Select the
USB drive and wait for the installer to load.

### First post-installation steps

The main installation process for Antergos is as straightforward as it is
for Ubuntu and macOS. Once you have completed the installation you should visit
the [Arch Linux wiki for MBP 11,4][arch_mbp11_4], which is what I used to
get things up and running initially. First install `yaourt` by
running `sudo pacman -S yaourt`. `yaourt` allows you to install community
packages, which greatly increases the number of available packages but
comes with the risk of installing malicious packages (from my experience, this
risk is extremely low). The first thing I installed was the [wifi][wifi] driver.
However, I believe it has now been integrated into the kernel. In case I'm
incorrect you can install the driver by running `yaourt -S broadcom-wl-dkms`
and rebooting. Next I installed the [webcam driver][cam]
by running `yaourt -S bcwc-pcie-dkms` and again rebooted. Then I installed
the [modified Linux kernel for MacBooks][linux_macbook], which you can install
in one of two ways. Pre-built binaries are downloadable from pacman by
adding the following to `/etc/pacman.conf`:

{% highlight text linenos %}
[linux-macbook]
SigLevel = Optional TrustAll
Server = http://libpcap.net/repo/linux-macbook
{% endhighlight %}

and running `sudo pacman -Sy linux-macbook`. Alternatively, you can build
the kernel yourself (warning, this can take over an hour) by running
`yaourt -S linux-macbook`.

Once the `linux-macbook` kernel is compiled you need to modify GRUB (the
menu that lets you select the OS and kernel upon boot) to boot
using the patched kernel by default. I found that the easiest way to get
that to work was to append:

{% highlight bash linenos %}
# To be able to boot the linux-macbook kernel by default we disable submenu
# We also changed GRUB_DEFAULT=2 because this is the entry corresponding
# to linux-macbook kernel (counting starts at 0).
GRUB_DISABLE_SUBMENU=y
{% endhighlight %}

to `/etc/default/grub`. This removes the normal submenu hierarchy in GRUB.
I then set the default menu item by specifying `GRUB_DEFAULT=2` in
`/etc/default/grub`. The reason for the number `2` is that the `linux-macbook`
kernel was the third entry from the top when I booted into GRUB without the
submenu. Finally, while we are editing GRUB we should add `acpi-osi=` to line 4,
which starts with `GRUB_CMDLINE_LINUX_DEFAULT=`. Make sure to add `acpi-osi=`
inside the quotation marks. To update GRUB run
`sudo grub-mkconfig -o /boot/grub/grub.cfg`.
Now reboot your machine and make sure you've booted with the `linux-macbook`
kernel by running `uname -r` in a terminal. As of this writing I get the output
`4.12.14-1-macbook`.

### Staying asleep

The next thing to get working properly is sleep. In addition to requiring the
patched Linux kernel, I also had to disable wakeup from USB 3.0, which would
cause repeated waking of my MacBook after closing the lid. I will discuss a
summarized version of [this blog post.](https://joshtronic.com/2017/03/13/getting-suspend-in-linux-working-on-a-macbook-pro/)

First create a new systemd unit by creating the file
`/etc/systemd/system/mbp_fix_sleep.service` and add the following to it:

{% highlight text linenos %}
[Unit]
Description=Fix sleep on MBP

[Service]
Type=oneshot
ExecStart=/bin/bash -c "echo XHC1 > /proc/acpi/wakeup"
RemainAfterExit=true
StandardOutput=journal

[Install]
WantedBy=multi-user.target
{% endhighlight %}

then run `systemctl daemon-reload && systemctl enable mbp_fix_sleep` to
enable at boot and `systemctl start mbp_fix_sleep` to have it take
effect immediately. You should now be able to close the lid and have your
MacBook remain asleep. If you have trouble, try rebooting, and making sure
that when you run `less /proc/acpi/wakeup` that `XHC1` is disabled. If you
continue to have trouble you can try disabling lid sleep/wake by replacing
the `ExecStart` line above with

{% highlight text linenos %}
ExecStart=/bin/bash -c "echo XHC1 > /proc/acpi/wakeup && echo LID0 > /proc/acpi/wakeup"
{% endhighlight %}

Rerun the `systemctl` commands above, and see if things now work. If sleep
still does not work unfortunately I cannot offer more advice since the
above worked for me.

### HiDPI with second display

I like to use a dual-monitor setup on occasion, which is currently not
supported by X11, so you need to change to running [Wayland][wayland].
A word of
caution, Wayland is still under heavy development, but has proven to
stable and very usable for me. I haven't encountered any issues. As far as I'm
aware even LibreOffice should now be working under Wayland. The biggest
obstacle I've faced is that if you want to do custom key remapping you'll
need to use xkb directly instead of something else like xmodmap. However,
I'll have a list at the end of this section of issues and solutions I've
encountered.

To get Wayland working you must switch from using lightdm to gdm. To install
gdm run `sudo pacman -Sy gdm`. Next we need to disable lightdm and enable gdm.
To do this run `sudo systemctl disable lightdm && sudo systemctl enable gdm`
and reboot. To verify that you are running Wayland run
`echo $XDG_SESSION_TYPE`. If Wayland is running you'll see `wayland`
and if X11 is running you'll see `x11`.

#### Issues and Solutions

- *GParted GUI not working:* from [here](gparted_gui_fix) run
  `xhost +SI:localuser:root && sudo gpartedbin`
- *Screen sharing in Chrome does not work with non-Chrome windows.* This is
  because of a security vulnerability that is used for screen sharing under
  X11 being absent in Wayland. PipeWire is designed to solve this problem but
  I have not yet tried it.
- The custom keyboard shortcut method is described below.

### Disable red audio light

Disable the red light by running:

{% highlight bash linenos %}
sudo echo 1 > /sys/module/snd_hda_intel/parameters/power_save
{% endhighlight %}

as root. To have this be run at startup automatically create the file
`/etc/systemd/system/mbp_spdif.service` with the contents:

{% highlight text linenos %}
[Unit]
Description=Disable S/PDIF red light in audio jack

[Service]
Type=oneshot
ExecStart=/bin/bash -c "echo 1 > /sys/module/snd_hda_intel/parameters/power_save"

[Install]
WantedBy=multi-user.target
{% endhighlight %}

then run `systemctl daemon-reload && systemctl enable mbp_spdif` to enable at
boot and `systemctl start mbp_spdif` to have it take effect immediately.
This is necessary because the sound card is not in power save mode
by default (see [here][fedora_mbp_chromabits]). Note that the light does come
back on when playing audio, even if it's through the built-in speakers and not
the audio jack.

### Bluetooth Headset

If you are having trouble with bluetooth headsets or headphones, give
[this page][bluetooth] a read.

### Parallel yaourt builds

You can speed up the build process of packages by building in parallel. To
enable building in parallel edit `/etc/makepkg.conf` by searching for
`MAKEFLAGS` and if the line is commented out then uncomment it. The make flag
`-jNumberOfCores` is how you control on how many cores to build. I have
8 hyperthreads and find that `-j6` allows me to continue working uninterrupted
while building packages.

### Emacs shortcuts everywhere

To get Emacs shortcuts everywhere follow the instructions from the Arch Linux
wiki [here](https://wiki.archlinux.org/index.php/GTK%2B#Emacs_keybindings).
First add the following:

{% highlight text linenos %}
[Settings]
gtk-key-theme-name = Emacs
{% endhighlight %}

to the file `~/.config/gtk-3.0/settings.ini`. Then run

{% highlight bash linenos %}
gsettings set org.gnome.desktop.interface gtk-key-theme "Emacs"
{% endhighlight %}

### Reduce title bar size

To reduce the size of the title bar edit `~/.config/gtk-3.0/gtk.css`
adding

{% highlight css linenos %}
window.ssd headerbar.titlebar {
    padding-top: 3px;
    padding-bottom: 3px;
    min-height: 0;
    font-size: 0.7em;
}

window.ssd headerbar.titlebar button.titlebutton {
    padding: 0px;
    min-height: 0;
    min-width: 0;
}
{% endhighlight %}

### Custom keyboard shortcuts

Custom keyboard shortcuts can be added using `AutoKey`. To install run
`yaourt -S autokey-py3`. Once installed open up AutoKey, which will live in the
system tray. You can open the main window from the system tray icon. Let's map
`C-m` to `enter`, like any sane person would want to. To do this open
`My Phrases` and add a new phrase. In the white text box you should have only
`<enter>`. Next, set the `Hotkey:` below to `<ctrl>+m`. You will also want to
add `AutoKey` as a startup application by adding it to
`Tweak Tool`->`Startup Applications`. While `AutoKey` isn't supported on
Wayland, I have observed that `C-m` works as return if I switch caps lock
to something other than "Disabled" and then back to "Disabled" in
`Tweak Tool`->`Keyboard & Mouse`->`Additional Layout Options`.

### Gnome extensions

I use the following Gnome extensions:
- AlternateTab
- Caffeine (had troubles under Wayland for a while, haven't retried)
- Dask to Dock
- Removable Drive Menu
- Remove Drop Arrows
- TopIcons Plus
- User themes

### Gnome appearance

I use the following appearance configuration:
- Global Dark Theme
- GTK+: Adwaita-dark
- Icons: Numix
- Shell theme: Lexis

# Scientific Computing on Arch Linux

Since many scientific libraries either have some fortran code or call fortran
code that's in a library we need to install `gfortran`. This is done by running
`sudo pacman -S gfortran`. If you also would like to use the clang compiler for
C and C++ you can install it and extras like [Clang-Tidy][clang-tidy] and
[ClangFormat][clang-format] by running `sudo pacman -S clang` and
`sudo pacman -S clang-tools-extra`. You can install [libc++][libcxx] from AUR using
`yaourt -S libc++ && yaourt -S libc++abi`. You may have to add the missing
gpg key to your trusted list using `gpg --recv-keys <the-key>`. You should then
be able to link libc++ instead of stdlibc++.

Now that we have the basics set up that we need for compiling code we can worry
about how to actually manage different installations of libraries for different
compilers. I personally really like what [Spack][spack] has done, which allows
you to easily install use modules. In order to use modules we need to first
install [lmod][lmod] using `yaourt -S lmod`. You'll then need to add

{% highlight bash linenos %}
# Load LMod for modules support
. /etc/profile.d/modules.sh
{% endhighlight %}

to your `~/.bashrc` file. You are now ready to follow the installation
instructions for [Spack][spack], which as of this writing required cloning
the GitHub repository. To "install" it for use everywhere add the following
to your `~/.bashrc` file

{% highlight bash linenos %}
# Add Spack bin path
export PATH=$PATH:/path/to/spack/bin
# Source Spack env
. /path/to/spack/share/spack/setup-env.sh
{% endhighlight %}

replacing `/path/to` with the path to where you cloned Spack.
You can now use Spack to manage (nearly) all of your scientific package needs.
For example, installing OpenBLAS and GSL was really simple. I suggest reading
the [Spack documentation][spack_dox] to learn more.

### Installing packages

I found it very easy to install a variety of applications that I use. For
example, I use [Slack][slack] for work and installing it was as easy as
running `yaourt -S slack-desktop` (though I've switched to using
[Franz][franz]). Another application I use frequently is
[Spotify][], which I was able to install using `yaourt -S spotify`.
I've also had no trouble using [Zsh][zsh] and [oh-my-zsh][oh-my-zsh]
instead of bash. You can
search the Arch Linux user repository (AUR) [online][aur] to see if the
packages you need are available.

### Summary

In this post I went over how to setup Arch Linux/Antergos on a MacBook Pro
11,4, though I expect models close to that will be similar. I've been using
this setup for about one year now and am really happy with it. I have
absolutely no desire to switch back to macOS. I find that for the type of
work I do that the Linux environment is simply a much better fit.
Thanks for reading and I hope you found the post useful!

[arch]: https://www.archlinux.org/
[antergos]: https://antergos.com/
[antergos_usb]: https://antergos.com/wiki/install/create-a-working-live-usb/
[arch_mbp11_4]: https://wiki.archlinux.org/index.php/MacBookPro11,x
[wifi]: https://wiki.archlinux.org/index.php/MacBookPro11,x#Wi-Fi
[cam]: https://wiki.archlinux.org/index.php/MacBookPro11,x#Web_cam
[linux_macbook]: https://wiki.archlinux.org/index.php/MacBookPro11,x#Backlight_keys_.2F_Suspend_support
[slack]: https://slack.com/
[spotify]: https://www.spotify.com/us/
[aur]: https://aur.archlinux.org/
[clang-tidy]: http://clang.llvm.org/extra/clang-tidy/
[clang-format]: https://clang.llvm.org/docs/ClangFormat.html
[libcxx]: https://libcxx.llvm.org/
[lmod]: https://aur.archlinux.org/packages/lmod/
[spack_dox]: http://spack.readthedocs.io/en/latest/
[fedora_mbp_chromabits]: https://chromabits.com/posts/2015/12/28/fedora-23-on-mbp/
[gparted_gui_fix]: http://gparted-forum.surf4.info/viewtopic.php?id=17446
[franz]: https://meetfranz.com/
[spack]: https://github.com/spack/spack
[bluetooth]: https://wiki.archlinux.org/index.php/Bluetooth_headset
[wayland]: https://wayland.freedesktop.org/
[zsh]: http://www.zsh.org/
[oh-my-zsh]: https://github.com/robbyrussell/oh-my-zsh
