# Ubuntu Server 19.10 Encrypted ZFS Data Pool

## 0. Important Information

This guide covers how to setup an encrypted zpool on Ubuntu Server 19.10. It is the third part of the Ubuntu Installation over SSH with ZFS Root installation series. 

This series does not use an encrypted root ZFS pool, although it is possible to do so as documented in the reference links in the [README](README.md). Physical access to the server is required on reboots in order to enter the decryption passphrase.

### 0.1 Goals

1. *Docker*: The remote server in this guide will be running it's applications using `docker` containers. A zfs dataset on an encrypted volume will help protect the container data at rest. As `docker` defaults to `zfs` storage in Ubuntu Server, this seems like a good fit.
2. *Database*: The backend databases for several applications will be `postgresql`, so a zfs dataset with optimal settings for RDBMS on an encrypted zpool is desirable.
3. *File Storage*: Additional datasets will be used on the encrypted zpool for virtual web server artifacts, image collections and other larger file storage.

### 0.2 Requirements

- The server will need some unpartitioned space for the encrypted zpool
- ZFS is already installed on Ubuntu Server

### 0.3 Caveats

- As with all encrypted partitions, a passphrase will need to be entered manually to mount the partition after OS boot.
- In the case of `docker`, it will need to be disabled on start-up, otherwise it may establish new ZFS volumes on-top of the root dataset.
- Any other `services` that rely on the partition will need to be started manually or to be configured in systemd to depend on the zfs dataset to be mounted.

## 1: Disk Partitioning

1.1 SSH into the server
```bash
ssh username@server-address
```

1.2 Elevate to root
```bash
sudo -i
```

1.3 Check that there is available space on the target disk. Change the device name to the disk you are working on.
```bash
sfdisk --list-free /dev/disk/by-id/virtio-serial_number
```

```bash
# Command Output
Unpartitioned space /dev/disk/by-id/virtio-6459748598: 1.102 GiB, 2146442752 bytes, 4192271 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes

   Start       End              Sectors Size
    2048        47 18446744073709549616  16E
23070720 125829086            102758367  49G
```

1.4 Check the current partition numbers, to ensure that the new partition is using an unused number.
```bash
sgdisk -p /dev/disk/by-id/virtio-serial_number
```
```bash
# Command Output
Number  Start (sector)    End (sector)  Size       Code  Name
   1              48            2047   1000.0 KiB  EF02  
   2            2048         2099199   1024.0 MiB  BF01  
   3         2099200        23070719   10.0 GiB    BF01  
```

1.5 Create the ZFS data partition and print the new partition table. 

The new partition in this case will be number `4` and the server will have `2G` reserved at the end of the disk using the `-2G` directive, just in case the need to perform a remote installation from a temporary partition is required again.
```bash
sgdisk -n4:0:-2G -t4:BF01 /dev/disk/by-id/virtio-serial_number
sgdisk -p /dev/disk/by-id/virtio-serial_number
```
The command `sgdisk` shows the new partition, along with the 2GB that was reserved at the end of the disk for recovery work.
```bash
# Command Output
Disk /dev/vda: 125829120 sectors, 60.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 8BD162A0-482B-422D-90F6-30328DC28036
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 125829086
Partitions will be aligned on 16-sector boundaries
Total free space is 4194318 sectors (2.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1              48            2047   1000.0 KiB  EF02  
   2            2048         2099199   1024.0 MiB  BF01  
   3         2099200        23070719   10.0 GiB    BF01  
   4        23070720       121634782   47.0 GiB    BF01  
```
## 2: Encrypted Zpool

2.1 **Optional**: A strong passphrase is recommended for encrypting the `zpool`. Store the passphrase in a safe place, such as a password manager like KeePassXC, this will be needed on every reboot of the server. 

Use `pwqgen` to generate a strong passphrase. Run it several times to find a suitable passphrase.
```
apt install --yes passwdqc
pwqgen random=72
```

2.1 Create the encrypted zpool. 

This guide will call the encrypted pool `vault`, change this to the desired pool name. Ensure the correct `-part#` is suffixed on the partition path as created in step `1.5`.

Enter the passphrase twice when prompted.

```bash
zpool create -o ashift=12 \
    -O encryption=on -O keylocation=prompt -O keyformat=passphrase \
    -O acltype=posixacl -O compression=lz4 -O dnodesize=auto \
    -O relatime=on -O xattr=sa -O normalization=formD -O devices=off \
    -O canmount=off -O mountpoint=none \
    vault /dev/disk/by-id/virtio-serial_number-part4
```
```bash
# Command Output
Enter passphrase: 
Re-enter passphrase:
```
2.2 List the zpools.
```bash
zpool list
```
```bash
# Command Output
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
boot     960M   113M   847M        -         -     0%    11%  1.00x    ONLINE  -
ubuntu  9.50G   826M  8.69G        -         -     1%     8%  1.00x    ONLINE  -
vault   46.5G  1.46M  46.5G        -         -     0%     0%  1.00x    ONLINE  -
```

## 3: ZFS Datasets

3.1 *Database*: Create a tweaked zfs dataset for databases.

- `recordsize`: matches the typical RDBMSs page size (8 KiB)
- `primarycache`: disable ZFS data caching, as RDBMSs have their own.
- `logbias`: disables log-based writes, relying on the RDBMSs' integrity measures.
- `quota`: sets a storage limit for the dataset on the zpool.
- `reservation`: reserves a chunk of blocks for the zpool.

This dataset is created first with the `reservation` targeted at the beginning of the pool. Theoretically, this will target the faster seek portion of the disk being towards the beginning of the zpool. However, the zpool may not actually function in this way, perhaps someone who knows how the dataset operates at the block level can tell me if this is misguided.

**Hint**: Compression is left enabled on this file system, as there are tangible benefits in write performance as long as the server has a quad-core CPU or better. Consder disabling compression on less powerful systems.

Change the mountpoint as desired, it could be the default postgres location `/var/lib/postgresql/`. In the case of this server, `/data/db` will be where the databases will be stored.

```
zfs create -o recordsize=8K -o primarycache=metadata -o logbias=throughput -o quota=15G -o reservation=15G -o mountpoint=/data/db vault/db
```

3.2 *Docker*: Create encrypted dataset for docker storage.

As docker manages it's own datasets & snapshots, disable the snapshot feature on this volume. This server will mount the dataset in the default location for docker storage in Ubuntu. If this is changed to another mountpoint, ensure docker is configured to reflect this change.

**Optional**: Set a quota for the dataset.

```
zfs create -o com.sun:auto-snapshot=false -o quota=15G -o mountpoint=/var/lib/docker vault/docker
```

3.3 *File Storage*: Create encrypted dataset for general secure storage.

As this volume will be storing media that is often compressed, such as `JPG` and `MP4` from mobile camera devices, leaving compression enabled will be inefficient. Diasble compression with the `compression=off` option. 

**Optional**: Set a quota for the dataset.

**Note**: If following this guide and nesting `vault/db` inside of `vault/data`, ensure `vault/db` is unounted first and the place holder folder `/data/db` removed prior to creating this dataset:

```bash
# optional
zfs umount vault/db
rm -rf /data/db
# create the dataset
zfs create -o compression=off -o quota=15G -o mountpoint=/data vault/data
zfs mount vault/db
```
3.4 List the zfs datasets.
```bash
zfs list
```
```bash
# Command Output
NAME               USED  AVAIL     REFER  MOUNTPOINT
boot               113M   719M       96K  /
boot/bios          112M   719M     65.0M  legacy
ubuntu            2.93G  6.27G       96K  /
ubuntu/home        424K  6.27G       96K  /home
ubuntu/home/berg   172K  6.27G      172K  /home/berg
ubuntu/home/root   156K  6.27G      156K  /root
ubuntu/root        637M  6.27G      631M  /
ubuntu/swap       2.13G  8.40G       56K  -
ubuntu/tmp         128K  6.27G      128K  legacy
ubuntu/var         187M  6.27G       96K  /var
ubuntu/var/cache   105M  6.27G      105M  /var/cache
ubuntu/var/lib    79.9M  6.27G     79.9M  /var/lib
ubuntu/var/log    1.85M  6.27G     1.85M  legacy
ubuntu/var/mail     96K  6.27G       96K  /var/mail
ubuntu/var/snap     96K  6.27G       96K  /var/snap
ubuntu/var/spool   104K  6.27G      104K  legacy
vault             15.0G  30.0G      192K  none
vault/data         200K  15.0G      200K  /data
vault/db           192K  15.0G      192K  /data/db
vault/docker       192K  15.0G      192K  /var/lib/docker
```

3.5 List the status of compression on the datasets.
```bash
zfs get compression
```
```bash
# Command output
NAME                 PROPERTY     VALUE     SOURCE
boot                 compression  lz4       local
boot/bios            compression  lz4       inherited from boot
boot/bios@install    compression  -         -
ubuntu               compression  lz4       local
ubuntu/home          compression  lz4       inherited from ubuntu
ubuntu/home/berg     compression  lz4       inherited from ubuntu
ubuntu/home/root     compression  lz4       inherited from ubuntu
ubuntu/root          compression  lz4       inherited from ubuntu
ubuntu/root@install  compression  -         -
ubuntu/swap          compression  zle       local
ubuntu/tmp           compression  lz4       inherited from ubuntu
ubuntu/var           compression  lz4       inherited from ubuntu
ubuntu/var/cache     compression  lz4       inherited from ubuntu
ubuntu/var/lib       compression  lz4       inherited from ubuntu
ubuntu/var/log       compression  lz4       inherited from ubuntu
ubuntu/var/mail      compression  lz4       inherited from ubuntu
ubuntu/var/snap      compression  lz4       inherited from ubuntu
ubuntu/var/spool     compression  lz4       inherited from ubuntu
vault                compression  lz4       local
vault/data           compression  off       local
vault/db             compression  lz4       inherited from vault
vault/docker         compression  lz4       inherited from vault
```
## 4: Mounting Procedure

4.1 Unmount the encrypted datasets and export the zpool
```bash
zfs umount vault/db
zfs umount vault/docker
zfs umount vault/data
zpool export vault
```
4.2 Check the zpool is not present.
```
zpool list
```
4.3 Import zpool and mount datasets.

**Important**: This is the process that will need to be followed upon reboot of the server.

4.3.1 Create a mount script.

The `zfs load-key` command unlocks the zpool using the passphrase used during setup. It is a good idea to turn this into a shell script for easy mounting during restarts.

Alternatively, run the commands manually in the bash script to test the process.

```
vi ~/mount-vault.sh
```
File contents:
```bash
#!/bin/bash
zpool import vault
zfs load-key vault
zfs mount vault/data
zfs mount vault/db
zfs mount vault/docker
```
4.3.2 Make the script as executable, run the script, and enter the encrypted zpool passphrase.
```bash
chmod +x ~/mount-vault.sh
./mount-vault.sh 
Enter passphrase for 'vault': 
```
4.4 Check the zpool and zfs datasets status.
```
zpool list
zfs list
```

## 5: Systemd Unit Files

TBC: This section will detail how to configure Docker and other daemons to wait until the zfs datasets are mounted before starting.