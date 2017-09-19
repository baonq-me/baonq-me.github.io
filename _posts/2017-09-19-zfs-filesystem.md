---
layout: post
title: ZFS Filesystem - Part 1
tags: [note, linux, storage]
categories:
- blog
---

ZFS is a combined file system and logical volume manager designed by Sun Microsystems. The features of ZFS include protection against data corruption, support for high storage capacities, efficient data compression, integration of the concepts of filesystem and volume management, snapshots and copy-on-write clones, continuous integrity checking and automatic repair, RAID-Z and native NFSv4 ACLs.

The ZFS name is registered as a trademark of Oracle Corporation; although it was briefly given the retrofitted expanded name "Zettabyte File System", it is no longer considered an initialism. Originally, ZFS was proprietary, closed-source software developed internally by Sun as part of Solaris, with a team led by the CTO of Sun's storage business unit and Sun Fellow, Jeff Bonwick. In 2005, the bulk of Solaris, including ZFS, was licensed as open-source software under the Common Development and Distribution License (CDDL), as the OpenSolaris project. ZFS became a standard feature of Solaris 10 in June 2006.

---

# Create pool

## RAID0 (Block-level striping without parity or mirroring)

```
RAID0:
  - Disk 1 (A1,A3,A5,A7,A9)
  - Disk 2 (A2,A4,A6,A8,A10)
```

- Minimum drives:     2
- Space efficiency:   1
- Fault tolerance:    None
- Read performance:   Nx
- Write performance:  Nx

Command:

`zpool create pool1 /dev/sda /dev/sdb`

## RAID1 (Mirroring without parity or striping)

```
RAID1:
  - Disk 1 (A1,A2,A3,A4,A5)
  - Disk 2 (A1,A2,A3,A4,A5)
```

- Minimum drives:     2
- Space efficiency:   1/N
- Fault tolerance:    N-1
- Read performance:   Nx
- Write performance:  1

Command:

`zpool create pool2 mirror /dev/sda /dev/sdb`

## RAID5 (Block-level striping with distributed parity)

```
RAID5:
  - Disk 1 (A1,B1,C)
  - Disk 2 (A2,B,C1)
  - Disk 3 (A,B2,C2)
```

- Minimum drives:     3
- Space efficiency:   1 - 1/N
- Fault tolerance:    1
- Read performance:   Nx
- Write performance:  (N-1)x

Command:

`zpool create pool3 raidz /dev/sda /dev/sdb /dev/sdc`


## RAID6 (Block-level striping with double distributed parity)

```
RAID6:
  - Disk 1 (A1,B1,C,D)
  - Disk 2 (A2,B,C,D1)
  - Disk 3 (A,B,C1,D2)
  - Disk 4 (A,B2,C2,D)
```

- Minimum drives:     4
- Space efficiency:   1 - 2/N
- Fault tolerance:    2
- Read performance:   Nx
- Write performance:  (N-2)x

Command:

`zpool create pool4 raidz2 /dev/sdb /dev/sdc /dev/sdd`

# Status of the ZPool Storage

## List pools

```bash
root@ubuntu:~# zpool list
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
rpool  31.8G  2.85G  28.9G         -     7%     8%  1.00x  ONLINE  -
```

## View basic dataset information

```bash
root@ubuntu:~# zfs list
NAME                           USED  AVAIL  REFER  MOUNTPOINT
rpool                         3.26G  27.5G   192K  /rpool
rpool/ROOT                    1.93G  27.5G   192K  /rpool/ROOT
rpool/ROOT/pve-1              1.93G  27.5G  1.93G  /
rpool/data                     402M  27.5G   192K  /rpool/data
rpool/data/subvol-101-disk-1   401M  4.61G   401M  /rpool/data/subvol-101-disk-1
rpool/data/vm-100-disk-1       128K  27.5G   128K  -
rpool/swap                     953M  27.9G   532M  -
```

## View health and status of pools

```bash
root@ubuntu:~# zpool status
  pool: rpool
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	rpool       ONLINE       0     0     0
	  sda2      ONLINE       0     0     0
	  sdb       ONLINE       0     0     0
	  sdc       ONLINE       0     0     0
	  sdd       ONLINE       0     0     0

errors: No known data errors
```

## View I/O statistics of pools

```bash
root@pve:~# zpool iostat
               capacity     operations    bandwidth
pool        alloc   free   read  write   read  write
----------  -----  -----  -----  -----  -----  -----
rpool       2.85G  28.9G      1     36  73.6K   146K
```

```
root@pve:~# zpool iostat -v
               capacity     operations    bandwidth
pool        alloc   free   read  write   read  write
----------  -----  -----  -----  -----  -----  -----
rpool       2.85G  28.9G      1     36  72.9K   146K
  sda2       971M  6.99G      0      9  24.8K  31.8K
  sdb        971M  6.99G      0      9  24.3K  35.1K
  sdc        971M  6.99G      0      8  23.6K  47.2K
  sdd       8.33M  7.93G      0     10    216  34.4K
----------  -----  -----  -----  -----  -----  -----
```

# Adding Aditional Storage

## Adding a spare drive

`root@pve:~# zpool add -f rpool spare /dev/sde`

Recheck

```bash
root@pve:~# zpool status -v
  pool: rpool
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	rpool       ONLINE       0     0     0
	  sda2      ONLINE       0     0     0
	  sdb       ONLINE       0     0     0
	  sdc       ONLINE       0     0     0
	  sdd       ONLINE       0     0     0
	spares
	  sde       AVAIL   

errors: No known data errors
```

This will add a spare drive to be used as a hot spare in event that one of the drives in the zpool fails. It mean that when an active drive in your pool fails, spare drive will automatically be activated to replace that failed drive.

Spare drive will not work when pool is ran under RAID0 because of its none tolerance, just lose a single drive then you lose the whole pool. 

## Adding additional drive to existing pools

Adding additional storage to a RAID1, RAID5 or RAID6 pool must be done in like RAID groups. For example, if you are running a RAID5 pool that contain 3 drives, you have to add exact 3 more drives (same number of existing drives).

In this example, my pool is running RAID0 with 4 drives then I can add more drives as many as I want.

```bash
root@pve:~# zpool add -f rpool /dev/sde
root@pve:~# zpool status -v
  pool: rpool
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	rpool       ONLINE       0     0     0
	  sda2      ONLINE       0     0     0
	  sdb       ONLINE       0     0     0
	  sdc       ONLINE       0     0     0
	  sdd       ONLINE       0     0     0
	  sde       ONLINE       0     0     0

errors: No known data errors
```

In case I use RAID6 with 4 drives for example, the command could be like:

`zpool add rpool raidz2 /dev/sde /dev/sdf /dev/sdg /dev/sdh`

# Deleting a pool

By deleting the pool `mypool`, you will also destroy all data that is stored inside.

`zpool destroy mypool`


# Removing a pool

Removing a pool is very similar to unmounting a drive from a mountpoint.

To remove a pool

`zpool export mypool`

And to add it again

`zpool import mypool`

This is used when the pool is located on an removale drive like USB drives, memory cards.

# Creating a Zpool on a disk image

Though not recommended for normal use, it is possible to create a zpool on top of a file.

`dd if=/dev/zero of=filename.img bs=1M count=1000`

`zpool create nameofzpool /absolute/path/to/filename.img`

will create an image of 1GB. It is also possible to create a sparse image, to create an image that can hold 100GB:

`dd if=/dev/zero of=filename.img bs=1k count=1 seek=100M`


# Reference
- [ZFS - From Wikipedia, the free encyclopedia](https://en.wikipedia.org/wiki/ZFS)
- [ZFS/ZPool - Ubuntu Wiki](https://wiki.ubuntu.com/ZFS/ZPool)
