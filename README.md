# Ubuntu Installation over SSH with ZFS Root

This repository contains a set of three guides to remotely reinstall a server where the only access to the server is via SSH. This can be the case with leased dedicated servers in data centres or remote customer servers without any remote console access (iDRAC/hILO/etc.).

It also includes instructions on installing Ubuntu Server with a ZFS (0.80) root filesystem and a guide on creating an encrypted data pool for securely storing data on the remote server.

1. [Ubuntu Server Remote SSH Reinstall](ubuntu-server-remote-ssh-installation.md)
2. [Ubuntu Server 19.10 on ZFS Root File System](ubuntu-server-19.10-zfs-root.md)
3. [Ubuntu Server 19.10 Encrypted ZFS Data Pool](ubuntu-server-encrypted-zfs-data-pool.md)

## Disclaimers

### 1: Regarding Data Loss
These guides are intended as a reference only and it's entirely possible that it will result in data loss if necessary precautions are not taken beforehand, such as a full system back-up. I am not responsible for any errors in this guide that may result in the loss of your data.

It's recommended to experiment with these procedures in virtual machines beforehand to emulate your exact scenarios in preparation to performing them on production systems.

### 2: Regarding ZFS Root File System
There are performance penalties with using ZFS as a root file system, particularly with respects to single disk systems. A lot of the benefits of ZFS integrity are lost in this configuration. 

That being said, the use of ZFS snapshots and native encryption may be of benefit. If maximum performance is the desire, stick to ext4/xfs root partitions and LUKS encryption. Also consider btrfs which is becoming a more suitable option with a similar feature set to ZFS.

### 3: Regarding UEFI installations
These guides to not cover any instructions for UEFI systems. Please find instructions for these in the References below. The general process of this Ubuntu Server 19.10 on ZFS Root guide is still an improvement on previous ZFS Root guides in terms of order of operation, so keep it in mind when following some of the older guides below.

### 4: Regarding Support
These guides are provided as is without any official support channels. If you require help, try the IRC channels `#zfsonlinux`, `#ubuntu-server`, `#ubuntu` on [freenode](https://freenode.net/).

## Thanks & References

Thank-you to the authors of the following guides that sourced the methods used in this series.

### 1: Remote Installation
- [Ubuntu OverSSH Reinstallation](https://gitlab.com/aasaam/ubuntu-overssh-reinstallation/tree/master) by aasam
- [HOWTO - Install Debian Onto a Remote Linux System](http://www.underhanded.org/papers/debian-conversion/remotedeb.html) by Erik Jacobson

### 2: Ubuntu on ZFS Root
- [Installing Ubuntu on a ZFS root, with encryption and mirroring](https://saveriomiroddi.github.io/Installing-Ubuntu-on-a-ZFS-root-with-encryption-and-mirroring/) by Saverio Miroddi
- [Ubuntu 18.04 Root on ZFS](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-18.04-Root-on-ZFS) by Richard Laager
- [Installing UEFI ZFS Root on Ubuntu 19.04](https://www.medo64.com/2019/04/installing-uefi-zfs-root-on-ubuntu-19-04/) by Josip Medved

