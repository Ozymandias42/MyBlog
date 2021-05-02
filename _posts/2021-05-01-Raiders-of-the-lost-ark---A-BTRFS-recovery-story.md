---
title: "Raiders of the lost ark - A BTRFS recovery story"
date: 2021-05-01
---

Recently I had to deal with a worst-case scenario.  
My archive-drive, a 2TB Toshiba USB hardrive with BTRFS became unmountable.   
This is how I repaired it.

This happened after I foolishly unplugged my Raspi 4 that it was connected to at the time, when shutdown took to long for my tastes.

[TOC]

# The discovery: Studying logs in *dmesg*

A quick look in `dmesg` showed the following:

```
[23079.925185] BTRFS info (device sdb2): bdev /dev/sdb2 errs: wr 0, rd 0, flush 0, corrupt 51, gen 0
[23080.156357] BTRFS critical (device sdb2): corrupt leaf: root=1 block=449881636864 slot=18 ino=2276, invalid inode generation: has 13167 expect (0, 13165]
[23080.156377] BTRFS error (device sdb2): block=449881636864 read time tree block corruption detected
[23080.169840] BTRFS critical (device sdb2): corrupt leaf: root=1 block=449881636864 slot=18 ino=2276, invalid inode generation: has 13167 expect (0, 13165]
[23080.169859] BTRFS error (device sdb2): block=449881636864 read time tree block corruption detected
[23080.169891] BTRFS error (device sdb2): failed to read block groups: -5
[23080.173682] BTRFS error (device sdb2): open_ctree failed
```

Seeing this, the last error is in opening one of the tree-structures of the BTRFS filesystem. 

To make the next steps easier to understand the next part will be a short overview of the trees in a BTRFS filesystem and what they are for.

---

## BTRFS trees — An overview

BTRFS consists of a number of different trees. 

- **root-tree**
  Contains the addresses of all other trees.
  
- **fs-tree**  
  The fs-tree or filesystem-tree contains the individual subvolumes file-structures.  
  On BTRFS each subvolume represents it's own filesystem-tree.  
  This is means, that BTRFS is a filesystem whose main tree can have arbitrary many child-trees, which can be mounted and treated as their own filesystem, all while sharing the storage space with the main volume.

  This differs from LVM insofar as each volume there requires both a full filesystem itself and a set pre-allocation of storage that is exclusive to that volume.  
  Granted, there are thin-volumes that allow over-provisioning but in that case it is still left to the user to make sure more space than actually available isn't attempted to be used.

  With BTRFS subvolumes, each volume sees the full space available to the whole pool, unless restricted via quota-groups or qgroups for short.

- **extent-tree**  
  The extent tree contains two kinds of extent. Block group extents and data extent.  
  Block Group extents are _"contiguous chunks of storage allocate for use in some way"_   
  Data extents are allocated from _"within block group extents"_ and represent the the sequence of bytes that contain data or metadata.

- **chunk-tree**  
  The chunk tree does the logical to physical block address mapping. Meaning it maps the addresses of chunks to actual blocks on the harddrive.

- **csum-tree**  
  The csum-tree contains checksums for all files on the disk, which allows the filesystem to detect corruption of actual data.

- **log-tree**  
  The log-tree  saves all file-transactions and does what is known as journaling.  
  This allows filesystems to recover from interrupted write-operations on sudden power-failure.  
  Without this, half-written data might end up overwriting a file and corrupting it without that being noticed.

- **device-tree**  
  The device tree is only really relevant in multi-drive-setups and contains the reverse of the mapping of the chunk tree.  
  It maps which parts of the physical device have been allocated into chunks.

---

# First recovery attempt — _btrfs check_

Seeing this I tried to run `btrfs check` to see if I could find the problem and possibly repair it.  
I did remember `btrfs check` having the `--init-csum-tree` repair-option and seeing as it apparently was the ctree that wouldn't open regenerating that one might help.

```
$ sudo btrfs check /dev/sdb2
Opening filesystem to check...
parent transid verify failed on 449881636864 wanted 12615 found 13172
parent transid verify failed on 449881636864 wanted 12615 found 13172
parent transid verify failed on 449881636864 wanted 12615 found 13172
Ignoring transid failure
leaf parent key incorrect 449881636864
ERROR: failed to read block groups: Operation not permitted
ERROR: cannot open file system
```

This left me with a conundrum. I could apparently not open the file-system with the check tool, so I was not able to use repair modes.

Seeing this I decided to try mounting it again but with some rescue flags.
The first thing I tried was `rescue=no_log_replay`seeing as this looked like a typical write-time corruption at first glance. (Spoiler: It wasn't. dmesg explicitly said it was read-time)

What `no_log_replay` does it to NOT try and use the transaction log to restore the last valid state. If the mount operation fails because this internal repair-function fails using `no_log_replay` can mount the drive again at the cost of an inconsistent filesystem state.

As this didn't work I also tried to do it with the `btrfs restore` userspace tool to the same effect.  
`sudo btrfs rescue zero-log /dev/sdb2` according to the `--help` page this does:

```
btrfs rescue zero-log <device>
        Clear the tree log. Usable if it's corrupted and prevents mount.
```

Of course this didn't work either.  
I tried with a few other mount-flags like `rescue=usebackuproot`
Which didn't work either.

# A deeper look — _btrfs inspect-internal_

The reason for why that was became apparent when I decided to look deeper into the filesystem.

```
$ sudo btrfs inspect-internal dump-super -f /dev/sdb2
superblock: bytenr=65536, device=/dev/sdb2
---------------------------------------------------------
csum_type               0 (crc32c)
csum_size               4
csum                    0x236347b4 [match]
bytenr                  65536
flags                   0x1
                         ( WRITTEN )
magic                   _BHRfS_M [match]
fsid                    2566e7c1-1e0f-40eb-ab68-914e5c346884
metadata_uuid           2566e7c1-1e0f-40eb-ab68-914e5c346884
label                   BTRFS-Archive
generation              13164
root                    2271131025408
sys_array_size          129
chunk_root_generation   13160
root_level              1
chunk_root              22036480
chunk_root_level        1
log_root                0
log_root_transid        0
log_root_level          0
total_bytes             2000188080128
bytes_used              1759527608320
sectorsize              4096
nodesize                16384
leafsize (deprecated)   16384
stripesize              4096
root_dir                6
num_devices             1
compat_flags            0x0
compat_ro_flags         0x0
incompat_flags          0x179
                         ( MIXED_BACKREF |
                           COMPRESS_LZO |
                           COMPRESS_ZSTD |
                           BIG_METADATA |
                           EXTENDED_IREF |
                           SKINNY_METADATA )
cache_generation        13164
uuid_tree_generation    13164
dev_item.uuid           5dda7f3a-edeb-45ed-8a94-d1c11bc74400
dev_item.fsid           2566e7c1-1e0f-40eb-ab68-914e5c346884 [match]
dev_item.type           0
dev_item.total_bytes    2000188080128
dev_item.bytes_used     1765256724480
dev_item.io_align       4096
dev_item.io_width       4096
dev_item.sector_size    4096
dev_item.devid          1
dev_item.dev_group      0
dev_item.seek_speed     0
dev_item.bandwidth      0
dev_item.generation     0
sys_chunk_array[2048]:
         item 0 key (FIRST_CHUNK_TREE CHUNK_ITEM 22020096)
                 length 8388608 owner 2 stripe_len 65536 type SYSTEM|DUP
                 io_align 6 5536 io_width 65536 sector_size 4096
                 num_stripes 2 sub_stripes 0
                         stripe 0 devid 1 offset 22020096
                         dev_uuid 5dda7f3a-edeb-45ed-8a94-d1c11bc74400
                         stripe 1 devid 1 offset 30408704
                         dev_uuid 5dda7f3a-edeb-45ed-8a94-d1c11bc74400
backup_roots[4]:
         backup 0:
                 backup_tree_root:       2271131025408   gen: 13164      
level: 1
                 backup_chunk_root:      22036480        gen: 13160      
level: 1
                 backup_extent_root:     2271131041792   gen: 13164      
level: 2
                 backup_fs_root:         2271132139520   gen: 13163      
level: 2
                 backup_dev_root:        2271131910144   gen: 13164      
level: 1
                 backup_csum_root:       2271144263680   gen: 13164      
level: 2
                 backup_total_bytes:     2000188080128
                 backup_bytes_used:      1759527608320
                 backup_num_devices:     1

         backup 1:
                 backup_tree_root:       2271522717696   gen: 13161      
level: 1
                 backup_chunk_root:      22036480        gen: 13160      
level: 1
                 backup_extent_root:     2271427756032   gen: 13161      
level: 2
                 backup_fs_root:         2271659212800   gen: 13162      
level: 2
                 backup_dev_root:        2271845613568   gen: 13160      
level: 1
                 backup_csum_root:       2271115018240   gen: 13161      
level: 2
                 backup_total_bytes:     2000188080128
                 backup_bytes_used:      1759125020672
                 backup_num_devices:     1

         backup 2:
                 backup_tree_root:       2271131172864   gen: 13162      
level: 1
                 backup_chunk_root:      22036480        gen: 13160      
level: 1
                 backup_extent_root:     2271126110208   gen: 13162      
level: 2
                 backup_fs_root:         2271659212800   gen: 13162      
level: 2
                 backup_dev_root:        2271845613568   gen: 13160      
level: 1
                 backup_csum_root:       2272123355136   gen: 13162      
level: 2
                 backup_total_bytes:     2000188080128
                 backup_bytes_used:      1759290830848
                 backup_num_devices:     1

         backup 3:
                 backup_tree_root:       2271143755776   gen: 13163      
level: 1
                 backup_chunk_root:      22036480        gen: 13160      
level: 1
                 backup_extent_root:     2271141412864   gen: 13163      
level: 2
                 backup_fs_root:         2271132139520   gen: 13163      
level: 2
                 backup_dev_root:        2271845613568   gen: 13160      
level: 1
                 backup_csum_root:       2271132188672   gen: 13163      
level: 2
                 backup_total_bytes:     2000188080128
                 backup_bytes_used:      1759527608320
                 backup_num_devices:     1
```

To see why the four backup superblocks didn't work we need to look at the `gen: ` entries.
The `dmesg` at the beginning clearly stated it had generation `13167` while actually expecting `13165`

The newest  generation in the backup superblocks is `13164` which is STILL too young.  
The cache's generation is also `13164`.

Now, what does this mean? According to the btrfs wiki the generation field corresponds to the transaction-id in the log-tree and is changed on CoW operations by being incremented by one.
As this Archive drive only really had one main subvolume, the root subvolume 5, it was now clear what went wrong.

I had obviously taken away power from the drive when a `sync()` operation was ongoing and trying to update the superblock to contain the newest generation.

# Backup before disaster

Since the next steps could potentially harm the already corrupted system more than it already is I decided on making a backup.
To save space I did this backup onto a squashfs image.

```bash
sudo mksquashfs Internal-Archive/empty-dir Internal-Archive/squash.img -info -b 1M -mem 7G -p 'BTRFS-Archive_backup.img f 444 root root pv -Itab -s 2000G /dev/sdb2'
```

This took about 7 hours to complete and managed to save about 400GB

# First steps to repair it — _--clear-space-cache_

With this in mind it became clear, that I would have to recreate one of the btrfs trees.  
Seeing as the `dmesg` complained about inode generation mismatches it was obvious that the `--clear-space-cache` was most likely the one needed. That or the `--clear-ino-cache`

Unfortunately this did not work as `btrfs check` refused to work on the trees without being able to open them all.

Luckily I have had the foresight to use the DUP profile on the harddrive to have the most important tree structures in duplicate. Usually this would be used in RAID setups but I set it to be done on that single disk too for precisely this thing.

Remember this part from the `inspect-internal` output?

```
sys_chunk_array[2048]:
         item 0 key (FIRST_CHUNK_TREE CHUNK_ITEM 22020096)
                 length 8388608 owner 2 stripe_len 65536 type SYSTEM|DUP
                 io_align 6 5536 io_width 65536 sector_size 4096
                 num_stripes 2 sub_stripes 0
                         stripe 0 devid 1 offset 22020096
                         dev_uuid 5dda7f3a-edeb-45ed-8a94-d1c11bc74400
         here ==>        stripe 1 devid 1 offset 30408704
                         dev_uuid 5dda7f3a-edeb-45ed-8a94-d1c11bc74400
```

Using the offset of stripe 1 devid 1 in `btrfs check` with the `–tree-root <bytenum>` flag fortunately did work.

So 6 hours later after the clearing the inode-cache  I was finally able to mount the disk again and recover the files.

 

