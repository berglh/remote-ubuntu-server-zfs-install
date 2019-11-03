# Ubuntu 19.10 Root on ZFS

## 0. Important Information

This guide covers how to install Ubuntu Server 19.10 with a ZFS root. It is the second part of the Remote SSH Ubuntu intsallation series. The same process will work for physical or virtual machines in a general setup with root ZFS.

Full disk encryption isn't the focus of this installation, but it's possible to accomplish this with LUKS or using ZFS encryption. Full encryption of root with native ZFS encryption hasn't been tested.

This guide doesn't cover UEFI installation, the general process should be similar, refer to previous guides for UEFI installs. This guide does contain some correct ordering of operations, namely around `grub`.

### Caution
* This guide uses a whole physical disk.
* Do not use these instructions for dual-booting.
* Backup your data. Any existing data will be lost.

Installing on a drive which presents 4KiB logical sectors (a “4Kn” drive) only works with UEFI booting. This not unique to ZFS. [GRUB does not and will not work on 4Kn with legacy (BIOS) booting.](http://savannah.gnu.org/bugs/?46700)

Computers that have less than 2 GiB of memory run ZFS slowly.  4 GiB of memory is recommended for normal performance in basic workloads.  If you wish to use deduplication, you will need [massive amounts of RAM](http://wiki.freebsd.org/ZFSTuningGuide#Deduplication). Enabling deduplication is a permanent change that cannot be easily reverted.


### 0.1 Remote SSH Installation

To follow on the remote SSH installation of Ubuntu 19.10, the server to be converted to ZFS will now be booting off a small swap partition from a minimal Ubuntu 19.10, preferably at the end of the disk. Login via SSH to a bash shell.

### 0.2 Testing from Ubuntu ISO

For just testing a ZFS install of Ubuntu Server 19.10, the [Ubuntu Desktop ISO](http://releases.ubuntu.com/19.10/) is needed. Boot the ISO (USB/DVD/etc) into the the Try Ubuntu without installing menu option. This will boot a live desktop environment with full bash terminal.

### 0.3 Installing via (K)VM

For just tyring this out using KVM first, best results are found using BIOS firmware instead of UEFI. It's possible to adopt this method using the UEFI approaches used elsewhere in older guides. [FIX: add references here]

Virtual Machine Manager is a graphical approach to testing this with KVM.

Install command:
```bash
apt-get install qemu virt-manager
```

*Note*: Working results were found using VirtIO drivers for disk, video and network. Other generic options may not work as expected.

**Important**: The virtual disk used for installation needs to have a serial number defined so that the disk device appears in `/dev/disk/by-id/` folder. This ensures ZFS is able to locate the devices correctly during boot. Configure it in the following way.

![alt text](./img/vm-serial-number.png "Logo Title Text 1")

Boot the VM using the [Ubuntu Desktop ISO](http://releases.ubuntu.com/19.10/) into the Try Ubuntu without installing mode and launch the terminal.

## 1: Prepare the Install Environment


1.1  Boot the Ubuntu Live CD. Select Try Ubuntu.  Connect the system to the Internet as appropriate and open a terminal by pressing `Ctrl-Alt-T`

1.2  Setup and update the repositories (Commands):
```bash
sudo apt-add-repository universe
sudo apt update
```
1.3  Optional: If testing on a VM, it's useful to use a remote SSH connection to continue with this guide. If following the remote SSH installation senario, SSH will already be in use. Set a password for the `ubuntu` user

```bash
passwd
There is no current password; hit enter at that prompt.
sudo apt install --yes openssh-server
```
Edit `/etc/ssh/sshd_config` to enable `PasswordAuthentication` by uncommenting and the restarting the SSH unit in systemd.
```bash
vi /etc/ssh/sshd_config
systemctl restart sshd
```

**Hint:** You can find your IP address with `ip addr show scope global | grep inet`.  Then, from your main machine, connect with `ssh ubuntu@IP`.

1.4 Elevate to root.
```bash
sudo -i
```

1.5  Install ZFS in the Ubuntu Live environment:
```bash
apt install --yes debootstrap gdisk zfs-initramfs zfsutils-linux
```

## 2: Disk Partitioning

**Hint:** `ls -la /dev/disk/by-id` will list all the available aliases, ensure you locate the correct disk.

Always use the long `/dev/disk/by-id/*` aliases with ZFS.  Using the `/dev/sd*` device nodes directly can cause sporadic import failures, especially on systems that have more than one storage pool.

2.1  If you are re-using a disk, clear it as necessary.

Clear the partition table:
```bash
sgdisk --zap-all /dev/disk/by-id/virtio-serial_number
```

2.2  Partition your disk(s):

2.2.1 BIOS partition: A BIOS boot partition is required, follow previous guides for UEFI. A UEFI partition will follow the BIOS partition and be number 2 (`n2 and t2`).
```bash
sgdisk -a1 -n1:24K:+1000K -t1:EF02 /dev/disk/by-id/virtio-serial_number
```

2.2.2 ZFS Boot partition: Create a ZFS boot partition.
```bash
sgdisk -n2:0:+1G -t2:BF01 /dev/disk/by-id/virtio-serial_number
```

2.2.3 ZFS Root partition: Create a ZFS root partition, this guide is using a small root, this should be at least 20GB typically.
```bash
sgdisk -n3:0:+10GB -t3:BF01 /dev/disk/by-id/virtio-serial_number
```

## 3: ZFS Pools

3.1 Create the boot ZFS pool

This guide calls it `boot`, other ZFS guides refer to this as `bpool`. This names are arbitrary, but they will need to be consistent for the guide to complete successfully.

GRUB does not support all of the zpool features. See `spa_feature_names` in [grub-core/fs/zfs/zfs.c](http://git.savannah.gnu.org/cgit/grub.git/tree/grub-core/fs/zfs/zfs.c#n276). This step creates a separate boot pool for `/boot` with the features limited to only those that GRUB supports, allowing the root pool to use any/all features. Note that GRUB opens the pool read-only, so all read-only compatible features are "supported" by GRUB.

You should not need to customize any of the options for the boot pool.

**Important**: You need to match the `part#` suffix of the device path to partitioned made in section 2 of this guide.

```bash
zpool create -o ashift=12 -d \
    -o feature@async_destroy=enabled \
    -o feature@bookmarks=enabled \
    -o feature@embedded_data=enabled \
    -o feature@empty_bpobj=enabled \
    -o feature@enabled_txg=enabled \
    -o feature@extensible_dataset=enabled \
    -o feature@filesystem_limits=enabled \
    -o feature@hole_birth=enabled \
    -o feature@large_blocks=enabled \
    -o feature@lz4_compress=enabled \
    -o feature@spacemap_histogram=enabled \
    -o feature@userobj_accounting=enabled \
    -O acltype=posixacl -O canmount=off -O compression=lz4 -O devices=off \
    -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=/ -R /mnt \
    boot /dev/disk/by-id/virtio-serial_number-part2
```

3.2 Create the root ZFS pool

This is guide is for an unencrypted root partition as it's assuming the access to the machine is via SSH only. In this senario, you won't be able to enter the passphrase to decrypt the partition on boot. This pool will be called `ubuntu` in this guide, other guides refer to this as `rpool`. On systems that can automatically install to ZFS, the root pool is named `rpool` by default, so do as you wish.

```bash
zpool create -o ashift=12 \
    -O acltype=posixacl -O canmount=off -O compression=lz4 \
    -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=/ -R /mnt \
    ubuntu /dev/disk/by-id/virtio-serial_number-part3
```
## 4: ZFS Datasets


It's typical to create container datasets for easy snapshotting with ZFS, this guide ommits the container datasets and in turn ommits easy snapshotting. Refer to previous guides on how to nest your boot and root pools inside of containers.

With ZFS, it is not normally necessary to use a mount command (either `mount` or `zfs mount`). This situation is an exception because of `canmount=noauto`.

4.1 Create `ubuntu/root` and `boot/bios` ZFS datasets.
```bash
zfs create -o canmount=noauto -o mountpoint=/ ubuntu/root
zfs mount ubuntu/root
zfs create -o canmount=noauto -o mountpoint=/boot boot/bios
zfs mount boot/bios
```
4.2 Create datasets within `ubuntu/root` for OS paths.

4.2.1 Create home container & datasets.
```bash
zfs create ubuntu/home
zfs create -o mountpoint=/root ubuntu/home/root
```

4.2.2 Create var container & datasets
```bash
# var container dataset
zfs create -o mountpoint=/var -o canmount=off ubuntu/var
# datasets beneath var
zfs create ubuntu/var/lib
zfs create ubuntu/var/log # system logs folder
zfs create ubuntu/var/spool # print spool folder
zfs create -o com.sun:auto-snapshot=false ubuntu/var/cache
zfs create ubuntu/var/mail # system mail folder
zfs create ubuntu/var/snap # ubuntu snaps
zfs create -o com.sun:auto-snapshot=false -o mountpoint=/tmp ubuntu/tmp # temp folder
# mark temp as world writeable
chmod 1777 /mnt/tmp
```
If using docker, opt or other datasets you want to control snapshotting individually, add them too.
```bash
zfs create -o com.sun:auto-snapshot=false ubuntu/var/lib/docker
zfs create ubuntu/var/www
```

4.2.3 Create swap dataset (Optional depending on requirements)

**Caution**: On systems with extremely high memory pressure, using a zvol for swap can result in lockup, regardless of how much swap is still available.  This issue is currently being investigated in: https://github.com/zfsonlinux/zfs/issues/7734

The compression algorithm is set to `zle` because it is the cheapest available algorithm.  As this guide recommends `ashift=12` (4 kiB blocks on disk), the common case of a 4 kiB page size means that no compression algorithm can reduce I/O.  The exception is all-zero pages, which are dropped by ZFS; but some form of compression has to be enabled to get this behavior.

**Important**: Always use long `/dev/zvol` aliases in configuration files. Never use a short `/dev/zdX` device name.

You can adjust the size (the `2G` part) to your needs.

```bash
zfs create -V 2G -b 4096 -o compression=zle -o logbias=throughput -o sync=always -o primarycache=metadata -o secondarycache=none -o com.sun:auto-snapshot=false ubuntu/swap
# mark it as swap
mkswap -f /dev/zvol/ubuntu/swap
echo RESUME=none > /etc/initramfs-tools/conf.d/resume
```

4.3 Check mountings and remount with dev enabled.

**Important** If at this stage `boot/bios` or any of the `ubuntu` datasets are mounted with the `nodev` flag, remount it with dev enabled.
```bash
# check the mount flags
mount -l | grep -e ubuntu -e boot
# remount with dev option
mount -i -o remount,exec,dev boot/bios
```

4.4 Check that all the datasets are mounted correctly

Container datasets should appear as not mounted.
```bash
zfs get mounted
NAME                 PROPERTY  VALUE    SOURCE
boot                 mounted   no       -
boot/bios            mounted   yes      -
ubuntu               mounted   no       -
ubuntu/home          mounted   yes      -
ubuntu/home/root     mounted   yes      -
ubuntu/root          mounted   yes      -
ubuntu/tmp           mounted   yes      -
ubuntu/var           mounted   no       -
ubuntu/var/cache     mounted   yes      -
ubuntu/var/lib       mounted   yes      -
ubuntu/var/log       mounted   yes      -
ubuntu/var/mail      mounted   yes      -
ubuntu/var/snap      mounted   yes      -
ubuntu/var/spool     mounted   yes      -
```
If any of the non-container datasets appear as not mounted, mount them.
```bash
# a specific set
zfs mount ubuntu/var/lib
# mount all, not ubuntu/root needs to be mounted before boot/bios
zfs mount -a
```

## 5: Install Ubuntu 19.10 to the ZFS root

5.1 Ensure `debootstrap` is installed and then run it targeting `eoan` and `/mnt`.
```bash
apt install --yes+ debootstrap
debootstrap eoan /mnt
zfs set devices=off ubuntu
```

## 6: Configure the Install

6.1 Configure hostname.
```bash
echo ubuntu > /mnt/etc/hostname
```

6.2 Add loopback (`127.0.0.1   hostname`) for hostname beneath localhost.
```bash
vi /mnt/etc/hosts
```

6.3 Networking. Get the interface name, configure netplan yaml.
```bash
ip addr show
cd /mnt/etc/netplan/
vi 01-network.yaml
```
Content may need to be configured with a static address, check the local netplan config for an existing server and copy the netplan config across from `/etc/netplan/01-netplan.yaml` to `/mnt/etc/netplan`. For a simple DHCP config, it should read be as follows.
```yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: true
```

6.4 Configure `apt` sources. Remove any src/universe/multiverse/backports as desired.
```bash
vi /mnt/etc/apt/sources.list
```
Contents:
```conf
deb http://au.archive.ubuntu.com/ubuntu/ eoan main restricted
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan main restricted

deb http://au.archive.ubuntu.com/ubuntu/ eoan-updates main restricted
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan-updates main restricted

deb http://au.archive.ubuntu.com/ubuntu/ eoan universe
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan universe
deb http://au.archive.ubuntu.com/ubuntu/ eoan-updates universe
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan-updates universe

deb http://au.archive.ubuntu.com/ubuntu/ eoan multiverse
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan multiverse
deb http://au.archive.ubuntu.com/ubuntu/ eoan-updates multiverse
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan-updates multiverse

deb http://au.archive.ubuntu.com/ubuntu/ eoan-backports main restricted universe multiverse
deb-src http://au.archive.ubuntu.com/ubuntu/ eoan-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu eoan-security main restricted
deb-src http://security.ubuntu.com/ubuntu eoan-security main restricted
deb http://security.ubuntu.com/ubuntu eoan-security universe
deb-src http://security.ubuntu.com/ubuntu eoan-security universe
deb http://security.ubuntu.com/ubuntu eoan-security multiverse
deb-src http://security.ubuntu.com/ubuntu eoan-security multiverse
```

6.5 **Optional**: Copy terminfo if you're using a different terminal emulator such as kitty at this point:
```
cp -r ~/.terminfo/ /mnt/root/
```

## 7. Configure basic system

7.1 Enter chroot.
```bash
mount --rbind --make-rslave /dev  /mnt/dev
mount --rbind --make-rslave /proc /mnt/proc
mount --rbind --make-rslave /sys  /mnt/sys
chroot /mnt /bin/bash --login
```
7.2 Populate the `apt` cache.
```bash
apt update
```

7.3 Configure locales and timezone.
```bash
locale-gen --purge "en_AU.UTF-8"
update-locale LANG=en_AU.UTF-8 LANGUAGE=en_AU
dpkg-reconfigure --frontend noninteractive locales
dpkg-reconfigure tzdata
```

7.4 Install zfs-initramfs & generic linux kernel.
```bash
apt install --yes zfs-initramfs zfsutils-linux
apt install --yes --no-install-recommends linux-image-generic
```

7.5 Create the ZFS import service for `boot` dataset

7.5.1 Add a zfs import systemd unit file for import on boot.

```bash
vi /etc/systemd/system/zfs-import-boot.service
```
Add the `zfs-import-boot.service` file contents.
```ini
[Unit]
DefaultDependencies=no
Before=zfs-import-scan.service
Before=zfs-import-cache.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/zpool import -N -o cachefile=none boot

[Install]
WantedBy=zfs-import.target
```

7.5.2 Enable the systemd unit
```
systemctl enable zfs-import-boot.service
```

7.6 Add system groups
```
addgroup --system lpadmin
addgroup --system sambashare
```


7.6 Configure fstab & grub

7.6.1 Convert datasets to legacy for boot compatability, add mounts to fstab.
```bash
# for boot dataset
zfs set mountpoint=legacy boot/bios
echo boot/bios /boot zfs nodev,relatime,x-systemd.requires=zfs-import-boot.service 0 0 >> /etc/fstab
mount boot/bios
# for ubuntu container datasets needed during kernal startup
zfs set mountpoint=legacy ubuntu/var/log
echo ubuntu/var/log /var/log zfs nodev,relatime 0 0 >> /etc/fstab
zfs set mountpoint=legacy ubuntu/var/spool
echo ubuntu/var/spool /var/spool zfs nodev,relatime 0 0 >> /etc/fstab
zfs set mountpoint=legacy ubuntu/tmp
echo ubuntu/tmp /tmp zfs nodev,relatime 0 0 >> /etc/fstab
echo /dev/zvol/ubuntu/swap none swap discard 0 0 >> /etc/fstab
```
7.6.2 Install grub, but *don't install to disk* yet when prompted.

```bash
apt install --yes grub-pc
```
Edit the grub config
```bash
vi /etc/default/grub
```

Edit `grub` file contents, add/uncomment/comment lines as required. These are set for diagnostic values.
```ini
GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/ubuntu"
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=5
GRUB_RECORDFAIL_TIMEOUT=5
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_TERMINAL=console
```

7.6.3 Check grub sees `/boot` as `zfs`.
```bash
grub-probe /boot
```

7.6.4 Update initramfs to add link for kernel in `/boot`.
```bash
update-initramfs -u -k all
```

7.6.5 Install grub to disk.
```bash
grub-install /dev/disk/by-id/${DISK_ID}
```

7.6.6 Update grub, it should find the kernel linked by initramfs. If it does not list the kernel here, something is wrong. Go back & figure out where it went wrong. *Ignore* the probe error here.
```bash
update-grub
```
7.6.7 Check zfs is present in /boot/grub, do not continue if files are not found.
```bash
ls /boot/grub/*/zfs.mod
```

7.7 Install SSH on the new server

It's important that this is working correctly in the remote SSH install senario
It's best practice to create a user first and then configure authorized_keys. If you're feeling brave and didn't add a user first, then allow root login with password temporarily in `sshd_config`.

```bash
apt install ssh
vi /etc/ssh/sshd_config
```
Uncomment and edit the following in `sshd_config` if you will login via password with the root user. If you are logging in with a created user, do not uncomment `PermitRootLogin`.

**Important** It's of the utmost importance that you disable password authentication as soon as possible in favour of authorized_keys once the system is booting. Public facing SSH servers with passwords have the possibility of being brute forced and the server pwned.
```bash
PermitRootLogin yes
PasswordAuthentication yes
```
Double check that SSH is enabled on start-up.
```bash
systemctl enable sshd
```

7.8 Configure root password.
```bash
passwd
```

## 8: Preparing to Reboot

8.1 Take install snaphots just in case something goes wrong, remove these later if desired.
```bash
zfs snapshot boot/bios@install
zfs snapshot ubuntu/root@install
```
8.2 Exit the `chroot` environment returning to the Ubuntu Live environment.
```bash
exit
```
8.3 Unmount all filesystems created.
**Important**: If any zpools are not exported, they will fail to import on boot. In the remote SSH installation senario, this will leave you in a non-recoverable state. It's important that both pools are unmounted correctly.
```bash
mount | grep -v zfs | tac | awk '/\/mnt/ {print $3}' | xargs -i{} umount -lf {}
zpool export -a
```
8.4 Cross phalanges, it's time to reboot.
```bash
reboot
```

## 9: Additional Configuration

Wait for the newly installed system to boot normally. Login as root.

9.1  Create a user account:
```bash
zfs create rpool/home/${USERNAME}
adduser ${USERNAME}
cp -a /etc/skel/. /home/${USERNAME}
chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}
```

9.2 **Optional**: Add your user account to the default set of groups for an administrator.
```bash
usermod -a -G adm,cdrom,dip,lpadmin,plugdev,sambashare,sudo ${USERNAME}
```

9.3 **Important**: Configure authorized_key and disable password and root login in sshd `/etc/ssh/sshd_config`

9.4 If setup earlier, ensure `swap` is on.
```bash
swapon -av
```

9.5 **Optional**: Disable log compression.

As `/var/log` is already compressed by ZFS, logrotate’s compression is going to burn CPU and disk I/O for (in most cases) very little gain.  Also, if you are making snapshots of `/var/log`, logrotate’s compression will actually waste space, as the uncompressed data will live on in the snapshot.  You can edit the files in `/etc/logrotate.d` by hand to comment out `compress`, or use this loop (copy-and-paste highly recommended):
```bash
for file in /etc/logrotate.d/* ; do
    if grep -Eq "(^|[^#y])compress" "$file" ; then
        sed -i -r "s/(^|[^#y])(compress)/\1#\2/" "$file"
    fi
done
```

9.6 Upgrade the minimal system.
```bash
apt dist-upgrade --yes
```

9.7 **Optional**: Install a regular set of software.
```bash
apt install --yes ubuntu-standard
```