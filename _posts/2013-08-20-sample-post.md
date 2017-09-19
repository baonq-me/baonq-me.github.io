---
layout: post
title: Sample post
tags: [code, jekyll]
categories:
- blog
---

Tattooed roof party *vinyl* freegan single-origin coffee wayfarers tousled, umami yr 
meggings hella selvage. Butcher bespoke seitan, cornhole umami gentrify put a bird 
on it occupy trust fund. Umami whatever kitsch, locavore fingerstache Tumblr pork belly
[keffiyeh](#). Chia Echo Park Pitchfork, Blue Bottle [hashtag](#) stumptown skateboard selvage 
mixtape. Echo Park retro butcher banjo cardigan, seitan flannel Brooklyn paleo fixie 
Truffaut. Forage mustache Thundercats next level disrupt. Bicycle rights forage tattooed
chia, **wayfarers** swag raw denim hashtag biodiesel occupy gastropub!

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

```
root@ubuntu:~# zpool list
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
rpool  31.8G  2.85G  28.9G         -     7%     8%  1.00x  ONLINE  -
```

## View basic dataset information

```
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

```
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

```
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
