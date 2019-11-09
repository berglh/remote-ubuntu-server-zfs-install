# Ubuntu Server 19.10 ZFS Root

## 0. Important Information

This guide covers how to install Ubuntu Server 19.10 with a ZFS root. It is the second part of the Remote SSH Ubuntu installation series. The same process will work for physical or virtual machines in a general setup with root ZFS.

Full disk encryption isn't the focus of this installation, but it's possible to accomplish this with LUKS or using ZFS encryption. Full encryption of root with native ZFS encryption hasn't been tested by this author.

This guide doesn't cover UEFI installation, the general process should be similar, refer to previous guides for UEFI installs linked in the [README](README.md). This guide does contain some correct ordering of operations, namely around `grub`, so it's worth following along and making the changes specific to UEFI along the way.

### Caution
* This guide uses a whole physical disk.
* Do not use these instructions for dual-booting.
* Backup the system data. Any existing data will be lost.

Installing on a drive which presents 4KiB logical sectors (a “4Kn” drive) only works with UEFI booting. This not unique to ZFS. [GRUB does not and will not work on 4Kn with legacy (BIOS) booting.](http://savannah.gnu.org/bugs/?46700)

Computers that have less than 2 GiB of memory run ZFS slowly.  4 GiB of memory is recommended for normal performance in basic workloads. To use the deduplication features of ZFS, the system will need [massive amounts of RAM](http://wiki.freebsd.org/ZFSTuningGuide#Deduplication). Enabling deduplication is a permanent change that cannot be easily reverted.


### 0.1 Remote SSH Installation

To follow on the remote SSH installation of Ubuntu 19.10, the server to be converted to ZFS will now be booting off a small swap partition from a minimal Ubuntu 19.10, preferably at the end of the disk. Login via SSH to a bash shell.

### 0.2 Testing from Ubuntu ISO

For just testing a ZFS install of Ubuntu Server 19.10, the [Ubuntu Desktop ISO](http://releases.ubuntu.com/19.10/) is needed. Boot the ISO (USB/DVD/etc) into the the Try Ubuntu without installing menu option. This will boot a live desktop environment with full bash terminal.

### 0.3 Installing via (K)VM

For just trying out the process using a VM first, best results are found using BIOS firmware instead of UEFI. It's possible to adopt this method using the UEFI approaches used elsewhere in older guides. See the [README](README.md) for links to these methods.

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
sudo apt-add-repository universe && apt update
```
1.3  Optional: If testing on a VM, it's useful to use a remote SSH connection to continue with this guide. If following the remote SSH installation scenario, SSH will already be in use. Set a password for the `ubuntu` user

```bash
passwd
sudo apt install --yes openssh-server
```
Edit `/etc/ssh/sshd_config` to enable `PasswordAuthentication` by uncommenting and the restarting the SSH unit in systemd.
```bash
sudo vi /etc/ssh/sshd_config
systemctl restart sshd
```

**Hint:** The IP address can be located with 
```
ip addr show scope global | grep inet
```
Connect from a terminal. 
```
ssh ubuntu@IP
```
If there a public key errors, force password authentication.
```
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no ubuntu@IP
```

1.4 **Optional**: If a non-standard terminal emulator is used, copy terminfo from the ubuntu user folder.
```
sudo cp -r .terminfo/ /root
```

1.5 Elevate to root.
```bash
sudo -i
```

1.6  Install ZFS in the Ubuntu Live environment:
```bash
apt install --yes debootstrap gdisk zfs-initramfs zfsutils-linux
```

## 2: Disk Partitioning

**Hint:** `ls -la /dev/disk/by-id` will list all the available aliases, ensure to locate the correct disk.

Always use the long `/dev/disk/by-id/*` aliases with ZFS.  Using the `/dev/sd*` device nodes directly can cause sporadic import failures, especially on systems that have more than one storage pool.

2.1  If the disk is being re-used, clear the partition information.

Clear the partition table:
```bash
sgdisk --zap-all /dev/disk/by-id/virtio-serial_number
```

2.2  Partition the disk(s):

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

2.3 **Important**: If following the remote SSH install process, the kernel will complain about the `/dev/vda5` preventing the new partition table from being accessible.

Install partprobe to tell the kernel the partitions have changed, even though `/dev/vda5` is still in use.
```
apt install parted
partprobe
```

## 3: ZFS Pools

3.1 Create the boot ZFS pool

This guide calls it `boot`, other ZFS guides refer to this as `bpool`. This names are arbitrary, but they will need to be consistent for the guide to complete successfully.

GRUB does not support all of the zpool features. See `spa_feature_names` in [grub-core/fs/zfs/zfs.c](http://git.savannah.gnu.org/cgit/grub.git/tree/grub-core/fs/zfs/zfs.c#n276). This step creates a separate boot pool for `/boot` with the features limited to only those that GRUB supports, allowing the root pool to use any/all features. Note that GRUB opens the pool read-only, so all read-only compatible features are "supported" by GRUB.

There should not be any need to customise any of the options for the boot pool.

**Important**: Carefully match the `part#` suffix of the device path to partitioned made in section 2 of this guide.

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
If an error occurs, use the `-f` flag to force the override.

3.2 Create the root ZFS pool

This is guide is for an unencrypted root partition as it's assuming the access to the machine is via SSH only. In this scenario, it won't be possible to enter the passphrase to decrypt the partition on boot. This pool will be called `ubuntu` in this guide, other guides refer to this as `rpool`. On systems that can automatically install to ZFS, the root pool is named `rpool` by default. The choice of name is arbitrary, choose one that works for the server environment.

```bash
zpool create -o ashift=12 \
    -O acltype=posixacl -O canmount=off -O compression=lz4 \
    -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa \
    -O mountpoint=/ -R /mnt \
    ubuntu /dev/disk/by-id/virtio-serial_number-part3
```
## 4: ZFS Datasets


It's typical to create container datasets for easy snapshotting with ZFS, this guide omits the container datasets and in turn omits easy snapshotting. Refer to previous guides on how to nest the boot and root pools inside of containers available in the references section of the [README](README.md).

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
If using docker, opt or other datasets, optionally add them too. This can also be done later on an encrypted zpool as demonstrated in the [Ubuntu Server Encrypted ZFS Data Pool](ubuntu-server-encrypted-zfs-data-pool.md) guide.
```bash
zfs create -o com.sun:auto-snapshot=false ubuntu/var/lib/docker
zfs create ubuntu/var/www
```

4.2.3 Create swap dataset (Optional depending on requirements)

**Caution**: On systems with extremely high memory pressure, using a zvol for swap can result in lock-up, regardless of how much swap is still available.  This issue is currently being investigated in: https://github.com/zfsonlinux/zfs/issues/7734

The compression algorithm is set to `zle` because it is the cheapest available algorithm.  As this guide recommends `ashift=12` (4 kiB blocks on disk), the common case of a 4 kiB page size means that no compression algorithm can reduce I/O.  The exception is all-zero pages, which are dropped by ZFS; but some form of compression has to be enabled to get this behaviour.

**Important**: Always use long `/dev/zvol` aliases in configuration files. Never use a short `/dev/zdX` device name.

The size can be adjusted (the `2G` part) to the server requirements.

```bash
zfs create -V 2G -b 4096 -o compression=zle -o logbias=throughput -o sync=always -o primarycache=metadata -o secondarycache=none -o com.sun:auto-snapshot=false ubuntu/swap
# mark it as swap
mkswap -f /dev/zvol/ubuntu/swap
swapoff -a
```

4.3 Check mountings and remount with dev enabled.

**Important** If at this stage `boot/bios` or any of the `ubuntu` datasets are mounted with the `nodev` flag, remount it with dev enabled.
```bash
# check the mount flags
mount -l | grep -e ubuntu -e boot
# remount with dev option
mount -i -o remount,dev boot/bios
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
# mount all, ubuntu/root needs to be mounted before boot/bios and other ubuntu datasets
zfs mount -a
```

## 5: Install Ubuntu 19.10 to the ZFS root

5.1 Ensure `debootstrap` is installed and then run it targeting `eoan` and `/mnt`.
```bash
apt install --yes debootstrap
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
vi /mnt/etc/netplan/01-network.yaml
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

6.5 **Optional**: Copy terminfo if using a different terminal emulator such as kitty at this point:
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

7.4 Install zfs tools
```bash
apt install --yes zfs-initramfs zfsutils-linux
```

7.5 Install generic kernel
```bash
apt install --yes --no-install-recommends linux-image-generic
```

7.6 Create the ZFS import service for `boot` dataset

7.6.1 Add a zfs import systemd unit file for import on boot.

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

7.6.2 Enable the systemd unit
```
systemctl enable zfs-import-boot.service
```

7.7 Add system groups
```
addgroup --system lpadmin
addgroup --system sambashare
```

7.8 Configure fstab & grub

7.8.1 Convert datasets to legacy for boot compatability, add mounts to fstab.
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
# add resume option for swap
echo RESUME=none > /etc/initramfs-tools/conf.d/resume
```
7.8.2 Install grub, but *don't install to disk* yet when prompted. Don't make any selections and just enter on OK, then yes.

```bash
apt install --yes grub-pc
```
Edit the grub config
```bash
vi /etc/default/grub
```

Edit `grub` file contents, add/uncomment/comment lines as required. These are set for diagnostic values.
```ini
GRUB_CMDLINE_LINUX="root=ZFS=ubuntu/root"
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=5
GRUB_RECORDFAIL_TIMEOUT=5
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_TERMINAL=console
```

7.8.3 Check grub sees `/boot` as `zfs`.
```bash
grub-probe /boot
```

7.8.4 Update initramfs to add link for kernel in `/boot`.
```bash
update-initramfs -u -k all
```

7.8.5 Install grub to disk.
```bash
grub-install /dev/disk/by-id/${DISK_ID}
```

7.8.6 Update grub, it should find the kernel linked by initramfs. If it does not list the kernel here, something is wrong. Go back & figure out where it went wrong. *Ignore* the probe error here.
```bash
update-grub
```
7.8.7 Check zfs is present in /boot/grub, do not continue if files are not found.
```bash
ls /boot/grub/*/zfs.mod
```

7.9 Install SSH on the new server

It's important that this is working correctly in the remote SSH install senario.

It's best practice to create a user first and then configure `authorized_keys`. As this guide creates the user after the first reboot, it will use password authentication. There will be less risk losing access to the server using password authentication temporarily, however this needs to be removed for security once the first user account has been created.

Install SSH and edit the config.

```bash
apt install --yes ssh
vi /etc/ssh/sshd_config
```
Uncomment and edit the following in `sshd_config` to login via password authentication with the root user.

```bash
PermitRootLogin yes
PasswordAuthentication yes
```
**Important Reminder** It's of the utmost importance that password authentication is disabled as soon as possible in favour of `authorized_keys` once the system is booting. Public facing SSH servers with passwords have the possibility of being brute forced and the server pwned.

7.10 Configure root password, make it reasonably strong as the login requests trying to hack into the server will start straight away.
```bash
passwd
```

## 8: Preparing to Reboot

8.1 Take install snaphots just in case something goes wrong, remove these later if desired.
```bash
zfs snapshot boot/bios@install
zfs snapshot ubuntu/root@install
```

*** Need to umount legacy fses, figure out if any file locks exist inside chroot.


8.2 Exit the `chroot` environment returning to the Ubuntu Live environment.
```bash
#umount legacy first
sudo umount /boot
#sudo umount /var/log
#sudo umount /var/spool
#sudo umount /tmp # this shouldn't be mountd
exit
```
8.3 Unmount all filesystems created.
**Important**: If any zpools are not exported, they will fail to import on boot. In the remote SSH installation senario, this will leave the server in a non-recoverable state. It's important that both pools are unmounted correctly.
```bash
for mnt in dev sys proc; do
  umount --recursive --force --lazy /mnt/$mnt
done
sudo zfs unmount -a
sudo umount ubuntu/root
zpool export boot
zpool export ubuntu
```
If a device busy error on umount of `ubuntu/root` or `ubuntu/var/lib` occurs, kill processes to release the pool.
```bash
# kill the process with the path /var/lib/os-prober/mount
pkill mount
# try umount again
zfs umount -a
# if the error continues, try this method which kills all non-critical processes
for p in $(sudo ps -ef | grep -v -e PID -e '\[' -e init -e sshd -e bash | awk '{print $2}'); do sudo kill $p; done
# it will unmount now
zfs umount -a
zpool export boot
zpool export ubuntu
```

8.4 Cross phalanges, it's time to reboot.
```bash
reboot
```

## 9: Additional Configuration

Wait for the newly installed system to boot normally. Login as via SSH as root.
```bash
ssh root@server-address
```

9.1  Create a user account:
```bash
zfs create ubuntu/home/${USERNAME}
adduser ${USERNAME}
cp -a /etc/skel/. /home/${USERNAME}
# optionally copy terminfo if using kitty etc
cp -r /root/.terminfo /home/${USERNAME}
# ensure all files are owned by the user
chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}
```

9.2 Add the user account to the default set of groups for an administrator.
```bash
usermod -a -G adm,cdrom,dip,lpadmin,plugdev,sambashare,sudo ${USERNAME}
```

9.3 **Important**: Configure authorized_key and disable password and root login in sshd `/etc/ssh/sshd_config`. Be careful with this configuration.

9.3.1 Add authorised key to the created user.
```bash
mkdir /home/username/.ssh
# paste a public SSH key into authorized_keys
vi /home/username/.ssh/authorized_keys
# set correct permissions for .ssh
# if these are not set, SSH key login will fail
chmod 700 /home/username/.ssh
chmod 600 /home/username/authorized_keys
chown -R username:username /home/username/.ssh
```
**Important**: Diconnect and try login with `authorized_key` before disabling the insecure login methods. If it is configured incorrectly and password login is disabled, login to the system via SSH will fail.

9.3.2 Disable insecure SSH login methods
```
vi /etc/ssh/sshd_config
```
Comment out PermitRootLogin and set PasswordAuthentication to no.
```bash
#PermitRootLogin yes
PasswordAuthentication no
```
9.3.3 Restart SSH
```
systemctl restart ssh
```
9.3.4 Disconnect and test login with password to ensure it's denying passowrd authentication. The permission denied error should appear.

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no username@server-address
username@server-address: Permission denied (publickey).
```

9.4 If setup earlier, ensure `swap` is on.
```bash
swapon -av
swapon -s
```

9.5 **Optional**: Disable log compression.

As `/var/log` is already compressed by ZFS, logrotate’s compression is going to burn CPU and disk I/O for (in most cases) very little gain.  Also, if snapshots are created of `/var/log`, logrotate’s compression will actually waste space, as the uncompressed data will live on in the snapshot.  Edit the files in `/etc/logrotate.d` by hand to comment out `compress`, or use this loop (copy-and-paste highly recommended):
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
9.8 **Optional**: Follow the [Ubuntu Server Encrypted ZFS Data Pool](ubuntu-server-encrypted-zfs-data-pool.md) guide to create secure storage solution for storing server data, databases and Docker volumes.