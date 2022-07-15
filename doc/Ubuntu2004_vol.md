# How to Deploy ZFS Volume for Mirror Disk with Ubuntu 20.04
- This article shows how to crate zfs volume for a mirror disk.

## Sample Configuration
- Ubuntu 20.04.4 LTS
  - 5.4.0-121-generic
- EXPRESSCLUSTER X 4.3 (4.3.0-1)
```
+---------------------------------+     +---------------------------------+
| sda                             |     | sda                             |
| server1                         |     | server2                         |
| - Ubuntu 20.04                  +-----+ - Ubuntu 20.04                  |
| - EXPRESSCLUSTER X              |     | - EXPRESSCLUSTER X              |
+------------+--------------------+     +------------+--------------------+
             |                                       |
+------------+--------------------+     +------------+--------------------+
| sdb                             |     | sdb                             |
| +-----------------------------+ |     | +-----------------------------+ |
| | zpool (tank1)               | |     | | zpool (tank1)               | |
| | +-------------------------+ | |     | | +-------------------------+ | |
| | | ZFS volume (tank1/vol1) | | |     | | | ZFS volume (tank1/vol1) | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | | | vol1-part1          | | | |     | | | | vol1-part1          | | | |
| | | | - Cluster Partition | | | |     | | | | - Cluster Partition | | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | | | vol1-part2          | | | |     | | | | vol1-part2          | | | |
| | | | - Data Partition    | | | |     | | | | - Data Partition    | | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | +-------------------------+ | |     | | +-------------------------+ | |
| |                             | |     | |                             | |
| | +-------------------------+ | |     | | +-------------------------+ | |
| | | ZFS volume (tank1/vol2) | | |     | | | ZFS volume (tank1/vol2) | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | | | vol2-part1          | | | |     | | | | vol2-part1          | | | |
| | | | - Cluster Partition | | | |     | | | | - Cluster Partition | | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | | | vol2-part2          | | | |     | | | | vol2-part2          | | | |
| | | | - Data Partition    | | | |     | | | | - Data Partition    | | | |
| | | +---------------------+ | | |     | | | +---------------------+ | | |
| | +-------------------------+ | |     | | +-------------------------+ | |
| +-----------------------------+ |     | +-----------------------------+ |
+---------------------------------+     +---------------------------------+
```

## Prerequisite
- Create a cluster without mirror disk resources.

## Install ZFS Utils
1. Install zfsutils-linux on all servers.
   ```sh
   sudo apt install zfsutils-linux
   ```

## Create ZFS Volume
1. Create a zpool on both servers.
   ```sh
   sudo zpool create tank1 /dev/sdb 
   ```
1. Create ZFS volume on both servers.
   ```sh
   zfs create -V 3g tank1/vol1
   ```
1. Create two partitions on both servers.
   ```sh
   fdisk fdisk /dev/zvol/tank1/vol1
   ```
   - /dev/zvol/tank1/vol1-part1
     - 1 GiB for cluster partition
   - /dev/zvol/tank1/vol1-part2
     - Any size for data partition
1. Create file system (e.g. ext4) on server1.
   ```sh
   mkfs -t ext4 /dev/zvol/tank1/vol1-part2
   ```

## Add Mirror Disk Resource
1. Start Cluster WebUI and add a mirror disk resource as below.
   - Mirror Disk Resource
     - Mirror Partition Device Name: /dev/NMP1
     - Mount Point: /mnt/md1
     - Data Partition Device Name: /dev/zvol/tank1/vol1-part2
     - Cluster Partition Device Name: /dev/zvol/tank1/vol1-part1
     - File System: ext4

