---
title: "Adventures in booting Linux on Raspberry Pi 4"
date: 2020-06-26T00:00:00-07:00
draft: false
tags: ['linux', 'raspberry pi']
---

Almost two months ago, I started building a four node Raspberry Pi 4 cluster for a project I’m working on. Figuring out the best way to get Linux on each node lead me down a rabbit hole and I spent the next four weeks [yak shaving](http://www.catb.org/~esr/jargon/html/Y/yak-shaving.html).

The most common way to boot a Pi is from an SD card. But they are slow and unreliable - I don't like them. Certain models of Pi 2 and Pi 3 support [booting from USB storage](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/msd.md), but Pi 4 cannot do that yet. So, I’m stuck with using an SD card for each Pi. Or so I thought.

### Enter PXE boot
Turns out, Raspberry Pi 2 and 3 also [support network booting](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net.md). A beta firmware was released late last year that enables [PXE](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) for Pi 4. I found [an article on LinuxHit](https://linuxhit.com/raspberry-pi-pxe-boot-netbooting-a-pi-4-without-an-sd-card) detailing the process of installing the firmware and setting up network boot.

Just to test this out, I updated the firmware and configured PXE server on one of the Pis. I tried booting another Pi over the network and...it worked! 
Here is the gist of PXE server setup process:
1. Copy the contents of `/boot` and the rest of `/` into separate folders on your local filesystem. I copied mine to `/tftp/raspbian-boot` and `/nfs/raspbian-root` respectively.
2. Install and configure `dnsmasq` to enable TFTP and use `/tftp` as `tftp-root`
3. Install and configure NFS server to share `/tftp/raspbian-boot` and `/nfs/raspbian-root`
4. Edit the contents of `/nfs/raspbian-root/fstab` to mount the boot partition.
5. Edit `/nfs/raspbian-boot/cmdline.txt` to
``` bash
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=<IP addr>:/nfs/raspbian-root,vers=4.1,proto=tcp rw ip=dhcp rootwait elevator=deadline
```

### PXE booting 3 nodes using OverlayFS
I cannot just boot all three nodes from the same boot and root filesystems since they are not read-only. And I don’t want to maintain a copy of these for each node. But this is a solved problem - Docker already lets you spin up multiple containers from a single base image. So I'll just do [what Docker does](https://docs.docker.com/storage/storagedriver/overlayfs-driver/#how-the-overlay2-driver-works) - create [OverlayFS](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html) for each node using raspbian-root and raspbian-boot as my lower directories. I’ll share them over NFS like before.

Configuration for one of the nodes is below. Other two nodes follow the same pattern. Note that I moved raspbian-root and raspbian-boot to a USB hard drive (mounted at `/mnt`) to get better r/w performance. dc-a6-32-XX-XX-XX is the MAC address of the node. I'm using `tftp-unique-root=mac` option in `dnsmasq` to maintain a separate boot environment for each node based on its MAC address.
``` bash
$ mount -t overlay nog-boot -o lowerdir=/mnt/raspbian-boot,upperdir=/mnt/upper/nog-boot,workdir=/mnt/work/nog-boot -o nfs_export=on -o index=on -o redirect_dir=nofollow /tftpboot/dc-a6-32-XX-XX-XX

$ mount -t overlay nog-root -o lowerdir=/mnt/raspbian-root,upperdir=/mnt/work/nog-root,workdir=/mnt/work/nog-root -o nfs_export=on -o index=on -o redirect_dir=nofollow /nfs/nog

$ cat /tftpboot/dc-a6-32-XX-XX-XX/cmdline.txt
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=<IP addr>:/nfs/nog,vers=4.1,proto=tcp rw ip=dhcp rootwait elevator=deadline
```
(In case you are wondering what `nog` is: I name my servers after planets from the Star Wars universe. Following this tradition, I’ve named the Pis Mandalore (PXE server), Nog, Ordo and Werda)

You are probably thinking that this worked. It did not. 

NFS and OverlayFS did not play nice with each other. After a weekend trying to work around some weird issues, I gave up. 

But OverlayFS is not the only copy-on-write filesystem in existence, is it? ZFS and BTRFS offer [subvolumes and snapshots](https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs-subvolume) that I can use instead. But not before I ~~procrastinate~~ spend 2 weeks deciding which one to use.

### Moving to BTRFS
The plan is simple: I’ll create a subvolume each for raspbian-root and raspbian-boot. I’ll then create three snapshots of each subvolume - one for every node. I created a [btrfs file system](https://en.wikipedia.org/wiki/RAS_syndrome) on my external drive and ran the following commands

``` bash
$ btrfs subvolume create raspbian-root
$ btrfs subvolume create raspbian-boot

# Repeat the following for each node changing folder names as necessary
$ btrfs subvolume snapshot raspbian-boot nog-boot
$ btrfs subvolume snapshot raspbian-root nog-root
$ mount --bind /mnt/nog-boot /tftpboot/dc-a6-32-XX-XX-XX
$ mount --bind /mnt/nog-root /nfs/nog

# Don’t forget to edit cmdline.txt for each node
$ cat /tftpboot/dc-a6-32-XX-XX-XX/cmdline.txt
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=<IP addr>:/nfs/nog,vers=4.1,proto=tcp rw ip=dhcp rootwait elevator=deadline
```
Well, this worked! I have a Pi running without an SD card!! Now all I have to do is create systemd unit files to start all required services and mount snapshots. I can finally start working on my project.

Wait a minute. Take a closer look at the output of `cat /tftpboot/dc-a6-32-XX-XX-XX/cmdline.txt`. 
If you are like me, you will realize for the first time that `root=` in this file defines where the root filesystem will be mounted from. Right now, we are telling it to mount from `/dev/nfs`. What if I attach an external drive and set `root=` to point to the external drive?

### Network booting into a USB hard drive
RPi 4 cannot directly boot from a USB drive. As in - it cannot find `/boot` on a USB drive when powering on. But what if we provide `/boot` with network boot and mount `/` from a USB drive using `cmdline.txt`? Time to test.

I copied raspbian-root to a USB drive, edited `fstab` to mount `/` using the partition’s PARTUUID and connected it to Nog. I then updated the contents of `/tftpboot/dc-a6-32-XX-XX-XX/cmdline.txt`. 
``` bash
$ cat /tftpboot/dc-a6-32-XX-XX-XX/cmdline.txt
 console=serial0,115200 console=tty1 root=PARTUUID=c1f95d14-01 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait
```

After a couple of reboots, it worked! I now have a RPi 4 running off a USB hard drive!

### What did I gain?
Let's start with what I was looking for. All I wanted was to not deal with SD cards because they are slow and unreliable. Reliability is relative - everything fails at some point. But a decent quality USB hard drive will likely outlive an SD card. So let’s just look at some read/write performance numbers and see how they compare.
``` bash
$ sync; dd if=/dev/zero of=twogeefile bs=1M count=2048; sync  # Write performance
$ sudo sh -c "sync && echo 3 > /proc/sys/vm/drop_caches"
$ dd if=twogeefile of=/dev/null bs=1M count=2048              # Read performance
```

| Storage  | Read (MB/s)  | Write (MB/s)  |
|----------|-------------:|--------------:|
| Sandisk Ultra MicroSDXC Class 10  | 45.7 | 20.2 |
| Network Boot with BTRFS | 63.3 | 18.8  |
| / mounted from USB hard drive  | 113.0  | 92.7 |

With network boot, I was able to get read and write speeds similar to a class 10 SD card. Note that the boot images are hosted on a USB hard drive connected to Mandalore (PXE server). NFS seems to be the bottleneck here - raw r/w speeds of the hard drive are much better than this. All Pis are connected to Netgear's 8 port gigabit ethernet switch.

To no one's surprise, things improve considerably when Pis are running off a USB hard disk. Note that the external drives I used are 5400RPM hard disks repurposed from very old MacBooks. I bought them on eBay for $10 a pop. YMMV.

### Current setup
Right now, I have Mandalore booting from an SD card and `/` mounted from a USB hard disk. Nog, Ordo and Werda fetch `/boot` from Mandalore over the network and `/` mounted from a USB hard disk.

![from top to bottom - Mandalore, Nog, Ordo, Werda](/img/pi-cluster.jpg)

I can finally start working on my proj… wait, what?

[A new beta firmware is out for RPi 4 that lets you boot from USB drives directly.](https://www.tomshardware.com/how-to/boot-raspberry-pi-4-usb)

Oh well…

¯\\\_(ツ)\_/¯

***Update:** This post was discussed on [Hacker News](https://news.ycombinator.com/item?id=23666564). As jordybg [points out](https://news.ycombinator.com/item?id=23667247), USB boot is no longer "beta" and is available in the latest (2020-06-15) "stable" firmware release.*

