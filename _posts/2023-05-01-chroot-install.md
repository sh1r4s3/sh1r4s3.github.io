---
layout: post
title:  "Replace existing system with a different distribution"
date:   2023-05-01
---

# Prehistoric events
Once I stumbled upon a situation when my new VPS provider doesn't had my favorite Linux distribution in the list of possible candidates to install to VPS. I was a bit disappointed but the price was too tempting for me. The problem was that I wasn't be able to attach and boot from any additional drive. After hacking chroot with my new system, bricking main rootfs a few times, I found a simple solution for the problem.

# Do I have a plan? I have a concept of the plan!
I use the following algorithm:

1. Create a directory for new rootfs `/newroot/`. Download a rootfs archive of the favorite Linux distribution and unpack it inside `/newroot/`.

2. Mount `sysfs, procfs, devtmpfs, devpts, tmpfs`, etc. Also mount your rootfs partition into `/newroot/root/`.

3. Chroot into `/newroot/` and install the kernel, bootloader, base system packages, setup localtime, hostname, network, etc.

4. After the new system is prepared inside `/newroot/`, bootloader configured and installed it is time to move this `/newroot/` to `/newroot/root/`. And that's it, if everything is clean after reboot the new system should boot.

It's time to smash some keyboard and chew bubblegum. As a base system I will use Debian and as a distro to install I will use Arch.

Create the <code>/newroot/</code> directory and extract rootfs into it:

{% highlight sh linenos %}
root@debian:~# curl -O "https://mirrors.atviras.lt/archlinux/iso/2023.04.01/archlinux-bootstrap-x86_64.tar.gz"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  159M  100  159M    0     0  15.3M      0  0:00:10  0:00:10 --:--:-- 17.0M
root@debian:~# mkdir /newroot
root@debian:~# cd /newroot/
root@debian:/newroot# tar -x -f /root/archlinux-bootstrap-x86_64.tar.gz -pz --same-owner --numeric-owner --strip-components=1
{% endhighlight %}

Mount the required filesystems including <code>/</code>:

{% highlight sh linenos %}
root@debian:/# mkdir /newroot/oldroot
root@debian:/# mount --bind / /newroot/oldroot/
root@debian:/# mount proc -t proc /newroot/proc
root@debian:/# mount -t sysfs sys /newroot/sys/
root@debian:/# mount -t devtmpfs udev /newroot/dev
root@debian:/# mount -t devpts devpts /newroot/dev/pt
root@debian:/# mount -t devpts devpts /newroot/dev/pts -o mode=0620,gid=5
root@debian:/# mount -t tmpfs shm /newroot/dev/shm -o mode=1777
root@debian:/# mount -t tmpfs run /newroot/run
root@debian:/# mount -t tmpfs tmp /newroot/tmp/ -o mode=1777
root@debian:/# mount --bind /etc/resolv.conf /newroot/etc/resolv.conf
{% endhighlight %}

Now let's rock, I mean chroot to the newly created <code>/newroot/</code> and install the kernel, bootloader, packages, and other fun stuff.

{% highlight sh linenos %}
root@debian:/# PROMPT_COMMAND='PS1="\u@archbox:\w\$ "; unset PROMPT_COMMAND' chroot /newroot/ bash
root@archbox:/$ # First we need to uncomment the desired mirror
root@archbox:/$ # In my example I'd use the first mirror from Lithuania
root@archbox:/$ sed -i '/^## Lithuania/{n;s/^#\(.*\)/\1/g}' /etc/pacman.d/mirrorlist
root@archbox:/$ # Let's check the results
root@archbox:/$ grep -A2 '^## Lithuania' /etc/pacman.d/mirrorlist
## Lithuania
Server = http://mirrors.atviras.lt/archlinux/$repo/os/$arch
#Server = https://mirrors.atviras.lt/archlinux/$repo/os/$arch
root@archbox:/$
root@archbox:/$ # Update indexes
root@archbox:/$ pacman -Sy
:: Synchronizing package databases...
 core          153.3 KiB   608 KiB/s 00:00 [#######################################################################################] 100%
 extra        1772.4 KiB  6.87 MiB/s 00:00 [#######################################################################################] 100%
 community       7.3 MiB  11.5 MiB/s 00:01 [#######################################################################################] 100%
root@archbox:/$ # Now, if we'd try to update the system it'd give us an error
root@archbox:/$ pacman -Su
:: Starting full system upgrade...
resolving dependencies...
looking for conflicting packages...

[snap]

Total Download Size:    31.00 MiB
Total Installed Size:  123.83 MiB
Net Upgrade Size:       44.03 MiB

:: Proceed with installation? [Y/n] y
error: could not determine cachedir mount point /var/cache/pacman/pkg
error: failed to commit transaction (not enough free disk space)
Errors occurred, no packages were upgraded.
root@archbox:/$
root@archbox:/$ # To overcome this I use the following change in pacman.conf
root@archbox:/$ sed -i 's/CheckSpace/#&/g' /etc/pacman.conf
root@archbox:/$ # Also we need to init archlinux GPG keyring and populate it with the keys
root@archbox:/$ pacman-key --init
[snap]
root@archbox:/$ pacman-key --populate
[snap]
root@archbox:/$ # Finally we can update the system
root@archbox:/$ pacman -Su
:: Starting full system upgrade...
resolving dependencies...
looking for conflicting packages...
[snap]
root@archbox:/$ # And install the required packages
root@archbox:/$ pacman -S glibc linux{,-api-headers,-firmware,-firmware-whence} util-linux vim grub mkinitcpio inetutils
root@archbox:/$ # Setup localtime
root@archbox:/$ ln -s /usr/share/zoneinfo/Europe/Vilnius /etc/localtime
root@archbox:/$ # Let's setup network
root@archbox:/$ # loopback interface
root@archbox:/$ cat > /etc/systemd/network/1-loopback.network << EOF
[Match]
Name=lo

[Network]
Address=127.0.0.1/8
Address=::1/128
EOF
root@archbox:/$ # ethernet interface
root@archbox:/$ cat > /etc/systemd/network/2-ethernet.network << EOF
[Match]
Name=e*

[Network]
DHCP=yes
EOF
root@archbox:/$ # Install the GRUB bootloader
root@archbox:/$ grub-install /dev/sda
Installing for i386-pc platform.
Installation finished. No error reported.
root@archbox:/$ # And generate the GRUB configuration file
root@archbox:/$ grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  initramfs-linux-fallback.img
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
root@archbox:/$ # Generate password
root@archbox:/$ passwd
New password:
Retype new password:
passwd: password updated successfully
root@archbox:/$ # Set hostname
root@archbox:/$ echo shub_niggurath > /etc/hostname
{% endhighlight %}

And finally fire the new system:

{% highlight sh linenos %}
Arch Linux 6.2.13-arch1-1 (ttyS0)

shubniggurath login: root
Password:
Last login: Tue May  2 00:30:19 on tty1
[root@shubniggurath ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 10.0.2.15/24 metric 1024 brd 10.0.2.255 scope global dynamic ens3
       valid_lft 86385sec preferred_lft 86385sec
    inet6 fec0::5054:ff:fe12:3456/64 scope site dynamic mngtmpaddr noprefixroute
       valid_lft 86387sec preferred_lft 14387sec
    inet6 fe80::5054:ff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever
[root@shubniggurath ~]$ # Let's test the Internet on some BBS
[root@shubniggurath ~]$ telnet 116.202.123.100
{% endhighlight %}

![](/img/articles/chroot-install/bbs.png){: .align-center style="width: 50em; padding: 1em;"}

Nice work! Now we have our system of choice on a VPS.
