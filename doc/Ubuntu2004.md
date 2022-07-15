# How to Deploy ZFS on Mirror Disk with Ubuntu 20.04
- This article shows how to crate zpool on a mirror disk.

## Sample Configuration
- Ubuntu 20.04.4 LTS
  - 5.4.0-121-generic
- EXPRESSCLUSTER X 4.3 (4.3.0-1)
```
+-------------------------+     +-------------------------+
| sda                     |     | sda                     |
| server1                 |     | server2                 |
| - Ubuntu 20.04          +-----+ - Ubuntu 20.04          |
| - EXPRESSCLUSTER X      |     | - EXPRESSCLUSTER X      |
+------------+------------+     +------------+------------+
             |                               |
+------------+------------+     +------------+------------+
| sdb                     |     | sdb                     |
| +---------------------+ |     | +---------------------+ |
| | sdb1                | |     | | sdb1                | |
| | - Cluster Partition | |     | | - Cluster Partition | |
| +---------------------+ |     | +---------------------+ |
| | sdb2                | |     | | sdb2                | |
| | - Data Partition    | |     | | - Data Partition    | |
| | +-----------------+ | |     | | +-----------------+ | |
| | | zpool: tank1    | | |     | | | zpool: tank1    | | |
| | +-----------------+ | |     | | +-----------------+ | |
| +---------------------+ |     | +---------------------+ |
+-------------------------+     +-------------------------+
```

<!--
```
+--------------------+     +--------------------+
| vda                |     | vda                |
| server1            |     | server2            |
| - Ubuntu 22.04     +--+--+ - Ubuntu 22.04     |
| - EXPRESSCLUSTER X |     | - EXPRESSCLUSTER X |
+---------+----------+     +---------+----------+
          |                          |
+---------+----------+     +---------+----------+
| vdb                |     | vdb                |
| +----------------+ |     | +----------------+ |
| | zpool: tank1   | |     | | zpool: tank1   | |
| | +------------+ | |     | | +------------+ | |
| | | disk1      | | |     | | | disk1      | | |
| | +------------+ | |     | | +------------+ | |
| +----------------+ |     | +----------------+ |
+--------------------+     +--------------------+
```
-->
## Create Base Cluster
1. Create a cluster has the following resources at least.
   - Mirror Disk Resource
     - Mirror Partition Device Name: /dev/NMP1
     - Mount Point: (NULL)
     - Data Partition Device Name: /dev/sdb2 
     - Cluster Partition Device Name: /dev/sdb1
     - File System: none

## Install ZFS Utils
1. Install zfsutils-linux on all servers.
   ```sh
   sudo apt install zfsutils-linux
   ```

## Create zpool on Mirror Disk
1. Start the failover group on server1 and do the following steps on server1.
1. Create a zpool.
   ```sh
   sudo zpool create tank1 /dev/NMP1
   ```
1. Check if the zpool is created.
   ```sh
   zpool list
   ```
   - If the zpool is created successfully, you can get the following result.
     ```
     NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
     tank1  8.50G   716K  8.50G        -         -     0%     0%  1.00x    ONLINE  -
     ```
1. Check the mount point.
   ```sh
   zfs get mountpoint tank1
   ```
   - You can get the following resutl.
     ```
     NAME   PROPERTY    VALUE       SOURCE
     tank1  mountpoint  /tank1      default
     ```
1. Disable auto mount.
   ```sh
   sudo zfs set mountpoint=legacy tank1
   ```
1. Check the mount point, again.
   ```sh
   zfs get mountpoint tank1
   ```
   - You can get the following resutl.
     ```
     NAME   PROPERTY    VALUE       SOURCE
     tank1  mountpoint  legacy      local
     ```
1. Create a mount point.
   ```sh
   sudo mkdir /mnt/tank1
   ```
1. Mount the [Mirror Partition Device] (e.g. /dev/NMP1).
   ```sh
   sudo mount.zfs /dev/NMP1 /mnt/tank1
   ```
1. Create a file and write data.
   ```sh
   echo "Hello, ZFS!" |sudo tee /mnt/tank1/test.txt
   ```
1. Unmount the device.
   ```sh
   sodo umount /mnt/tank1
   ```
1. Export the zpool.
   ```sh
   sudo zpool export tank1
   ```
1. Move the failover group to server2. Do the following steps on server2.
   ```sh
   sudo clpgrp -m failover
   ```
1. Create a mount point.
   ```sh
   sudo mkdir /mnt/tank1
   ```
1. Import the zpool.
   ```sh
   sudo zpool import tank1
   ```
1. Mount /dev/NMP1.
   ```sh
   sudo mount.zfs /dev/NMP1 /mnt/tank1
   ```
1. Check the file.
   ```sh
   cat /mnt/tank1/test.txt
   ```
   - Yeah!!!
     ```
     Hello, ZFS!
     ```
1. Unmount the device.
   ```sh
   sodo umount /mnt/tank1
   ```
1. Export the zpool.
   ```sh
   sudo zpool export tank1
   ```

## Add Exec Resource to Control ZFS
1. Start Cluster WebUI.
1. Add exec resource and edit start.sh and stop.sh as below.
   - start.sh
     ```sh
     #! /bin/sh
     #***************************************
     #*              start.sh               *
     #***************************************
     
     #ulimit -s unlimited
     
     zpool list
     zfs list
     mount |grep zfs
     
     zpool import -f tank1
     if [ $? -ne 0 ]; then
         clplogcmd -m "zpool import failed (tank1)"
         exit 1
     fi
     
     mount.zfs /dev/NMP1 /mnt/tank1
     if [ $? -ne 0 ]; then
         clplogcmd -m "mount failed (tank1)"
         exit 1
     fi
     
     zpool list
     zfs list
     mount |grep zfs
     
     exit 0
     ```
   - stop.sh
     ```sh
     #! /bin/sh
     #***************************************
     #*               stop.sh               *
     #***************************************
     
     #ulimit -s unlimited
     
     zpool list
     zfs list
     mount |grep zfs
     
     umount /mnt/tank1
     if [ $? -ne 0 ]; then
         clplogcmd -m "umount failed (tank1)"
         exit 1
     fi
     
     zpool export tank1
     if [ $? -ne 0 ]; then
         clplogcmd -m "zpool export failed (tank1)"
         exit 1
     fi
     
     zpool list
     zfs list
     mount |grep zfs
     
     exit 0
     ```
1. Click [Tuning] button and go to [Maintenance] tab. And set the following parameters.
   - Log Output Path: /opt/nec/clusterpro/log/exec-zfs
   - Rotate Log: Check
   - Rotation Size: 1000000 [byte]
1. Apply the configuration file.