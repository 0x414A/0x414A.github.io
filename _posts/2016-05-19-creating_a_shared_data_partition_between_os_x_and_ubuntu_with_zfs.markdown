---
layout: post
title: "Creating a shared data partition between OS X and Ubuntu with ZFS"
---

A common problem with dual-booting is sharing data between different operating systems as the filesystems supported on one system are often not (fully) supported on another. 

An example of this is HFS+, which on OS X is fully supported due to it being a propriety, Apple format. 

On Linux, HFS+ read/write is supported once you install `hfsprogs`, but only if you turn off journaling for the disk/partition that you want to write to.

Enter [ZFS](https://en.wikipedia.org/wiki/ZFS), provided by [OpenZFS on OS X](http://open-zfs.org/wiki/OpenZFSOnOSX) and [ZFS for Linux](http://zfsonlinux.org/).

Since both the OS X (10.11.x) and Linux (Kernel 4.4.0+) implementations of ZFS appear to have feature parity, ZFS seemed well-suited to replacing my old HFS+ shared partition.

Thanks to [this](http://www.kaapstorm.com/post/using-zfs-share-data-linux-mac-os-x/) post by Norman Hooper, I've been able to get ZFS working on both OS X 10.11 and Ubuntu 16.04. 

**Disclaimer** I'm pretty new to ZFS myself, so take these steps with a grain of salt. If you have any suggestions for changes or alternatives, please comment below.

## Contents

* TOC
{:toc}

# Overview

Say you have an existing HFS+ data partition that sits between your apple-scented OS X partition (HFS+), and your humanist partition (ext4). You're not sure you like the fact that you have to disable journaling on this data partition, so you look into this ZFS thing.

What you might end up doing could look something like what I've done below.

Basically, my OS X system partition exists at `/dev/sda2` / `/dev/disk0s2` and my data partition calls `/dev/sda4` / `/dev/disk0s4` its humble abode. I wanted to make the latter my pool's base.

## Background

I was a little confused with the terminology, so here's an attempt at explaining it to others.

When you manage a ZFS storage devices, you deal with something called a _zpool_ and a _vdev_.

A _zpool_ is a group of _vdev_, ie. virtual devices like your `/dev/sda4`.

# Requirements 

You'll need to have ZFS utilities installed for your Operating System.

On Ubuntu 16.04, these are now included in Ubuntu's repos:

``` shell 
sudo apt install zfsutils-linux 
```

[OpenZFS on OS X](http://open-zfs.org/wiki/OpenZFSOnOSX).


# Before you begin

First, backup _all_ the files that live on that data partition that you deem crucial to a sufficiently large external drive.

This is important as **ZFS utils will overwrite existing data on setup of pools and all that jazz**.

# Setup ZFS

## Create a new zpool:

``` shell
sudo zpool create -o ashift=12 <pool-name> <device>
```

where `<pool-name>` could be something like `data` and `<device>` could be something like `/dev/sda4` (which is a 120GB partition I allocated for shared data between OS X and Linux).

In _Linux_, this will create a mountpoint at `/<pool-name>` and be mounted automatically. ex. `/data`.

In _OS X_, the mountpoint will be at `/Volumes/<pool-name>`.

You may want to turn off deduplication if you don't have at least 8GB of RAM.

``` shell
sudo zfs set atime=off data
sudo zfs set dedup=off data
```

## Create a new vdev:

Now, you can create a _vdev_ that will be mounted under the `<pool-name>`:

For example:

``` shell
sudo zfs create data/mysharedcrap
```

If you navigate to `/data`, you'll now see `mysharedcrap`.


## Set up correct permissions

Then, ZFS write/read permissions need to be changed to enable write access for your user:

``` shell
sudo zfs allow aj create,destroy,mount data
```

Like with HFS, NFS partitions shared between Linux and OS X, the UIDs and GIDs on both systems _must_ match with each other in order for read/write permissions to be granted between the systems. Otherwise, you'll need to `chown -R $USER`  every time you boot into the other operating system.

For this, you'll have to match the UID and GID of the user on Linux you primarily access the ZFS file systems. 

On OS X, get the `uid`  and `gid` of your user with `id $USER`. The output will look something like this:

``` text
uid=501(aj) gid=20(aj) groups=20(aj),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),106(input),113(lpadmin),128(sambashare),130(libvirtd),999(docker)
```

Then in Linux, temporarily enable the `root` account if it is disabled by setting a password:

```
sudo passwd root
```

While in Linux, change your UIDs and GIDs with the following incantations, replace `501` with whatever your `uid` is, and `YOURUSER` with your username:

``` shell 
usermod -u 501 YOURUSER 
vigr # change existing group with id 20 to something else
find / -user 1000 -exec chown -h 501 {} \;
groupmod -g 20 YOURUSER 
find / -group 1000 -exec chgrp -h 20 {} \;
usermod -g 20 YOURUSER 
```

If you use a display manager, also change the `UID_MIN` in `/etc/login.defs` to `500`.

Once that's done, the root account can be locke again.

``` shell
passwd -l root
```

## Frustratingly manual part: import/export

Supposedly, if you add these lines to `/etc/default/zfs`, you will not need to do the subsequent steps:

``` shell
ZFS_MOUNT='yes'
ZFS_UNMOUNT='yes'
```

However, this did not seem to work?

When you want to boot into your other OS, you'll need to unmount the ZFS devices and `export` them or you will not be able to write to the _vdev_. This is apparently a security measure to prevent race conditions in data transmission. Sort of like mutex locks. 

``` shell
sudo zfs umount -a
sudo zfs export data
```

Then, on the OS you want to switch to:

``` shell
sudo zfs import data
```

Note that for me, `import` didn't seem necessary.

## Close

Feel free to leave a comment below if you have any questions, corrections, suggestions, etc.
