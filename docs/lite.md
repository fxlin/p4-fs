# p4 lite: Filesystem inspection

In this experiment, we will inspect filesystem images. We will reason about how filedata and metadata are stored and maintained. 

<!--This is a light version of "Lab4: filesystem image forensics".--> 

## Objectives
* (primary) Reinforcing understanding of ext2
* (secondary) Familiarizing with Linux file tools

## Roadmap

* Mount an ext2 filesystem image with a few dir/files. 
* Use Linux tools to examine key data structures: inodes/bitmaps…
* Show how such a filesystem image is created

## Prerequisites 
```
git clone https://github.com/fxlin/p4-fs
```

**Required environment** 

Fiddling with a whole filesystem (and disk images) requires the root privilege. So do the following experiment either on your Linux box which you have root; or do it on the server (e.g. granger1), inside a QEMU emulator. The problem with QEMU: you will need a system image with all the utilities, e.g. dumpe2fs, etc. Can be built with buildroot which requires some exploration.

* Linux users: do this lab either on your local box which you have root. Or do it on the server, inside a QEMU emulator. The problem with QEMU: you will need a system image with all the utilities, e.g. dumpe2fs, etc. Can be built with buildroot which requires some exploration.
* Windows users: do this on WSL. These [instructions](./wsl.md) may help (credits: Andrew Jackman). 
* Mac Users: try https://www.digitalocean.com/ for which you can have a free account

**The tools we will use** 

* ls: show directory content
* dd: read/write raw content of a file (or a disk partition)
* mkfs.ext2: create a new ext2 filesystem
*   debugfs: ext filesystem debugger 
*   stat: get file status
*   dumpe2fs: dump ext filesystem info
*   xxd, hexdump: display binary data

## Step 1. Grab the disk image, mount, & basic inspection

The image file is a byte-to-byte dump of a small disk. The disk size is 2MB. 

```
# grab the file from this git repo (path: lite/disk.img.gz), and unzip it 
gzip -d disk.img.gz
```

How large is the disk image, compressed and decompressed? 

```
$ ls -lh disk.img.gz
-rwxr--r-- 1 xzl xzl 2.6K Nov 25 10:39 disk.img.gz

$ ls -lh disk.img
-rw-r--r-- 1 xzl xzl 2.0M Apr  6 13:33 disk.img
```
> Because our filesystem only uses a small fraction of the disk space (we will see soon), most of these bytes are zero. That's why we compress the disk image (as .gz) for distribution. 

Next, we mount the disk image as a loop device. (Recall: a loop device treats a file as a physical disk; see our lecture slides). **AGAIN, YOU WONT' BE ABLE TO DO THIS ON GRANGER1/2** (cf: the landing page)

```
# create our mount point
$ mkdir /tmp/myfs

# mount the disk image. Note the sudo command
$ sudo mount -o loop disk.img /tmp/myfs
```

Now we are ready to tinker with the filesystem. 

### Check the disk usage

```
$ df -hP /tmp/myfs/
Filesystem   Size Used Avail Use% Mounted on
/dev/loop2   2.0M  21K 1.9M  2% /tmp/myfs
```

This shows that our disk size is 2.0M, of which only 2% is used. It shows the mount point (/tmp/myfs) of our  filesystem. It also shows that the kernel allocates a loop device (`/dev/loop2`) as the backing store of the filesystem. The actual device name may vary, e.g. you may see `/dev/loop10`. This is the device name of our disk (everything is a file, right?)

### Check the filesystem type & mount options
So, what's the filesystem type? We can use `mount`, which lists all filesystems currently mounted. From its output, we look for our mount point. 

```
$ mount|grep myfs
/tmp/disk.img on /tmp/myfs type ext2 (rw,relatime)
```

`mount` shows our filesystem as ext2. It also shows the mount options (rw=readable & writeable; relatime~=skipping updating a file's access time for most accesses, see [details](https://blog.confirm.ch/mount-options-atime-vs-relatime/)). 

### List the directory structure

```
$ ls -lh /tmp/myfs/
total 14K
-rw-r--r-- 1 root root   10 Apr  6 11:44 aa
lrwxrwxrwx 1 root root    2 Apr  6 12:30 aa-symlink -> aa
drwx------ 2 root root  12K Apr  2 23:53 lost+found
drwxr-xr-x 2 root root 1.0K Apr  6 11:35 testdir
```

## Step 2. inspect metadata

Recall that an ext2 filesystem has a global "superblock", which is the metadata for the entire filesystem. To dump its contents, we of course can write our own program to parse the disk image. But there's already a handy tool -- `dumpe2fs`.  It will accept the disk device name (dev/loop2 in my case; this may be different in your case, see discussion above). 

```
$ sudo dumpe2fs  /dev/loop2
dumpe2fs 1.42.13 (17-May-2015)
Filesystem volume name:   <none>
Last mounted on:          /tmp/myfs
Filesystem UUID:          6430eccd-11d0-4ea9-bd26-ce6e946dc02b
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      ext_attr resize_inode dir_index filetype sparse_super large_file
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              256
Block count:              2048
Reserved block count:     102
Free blocks:              1990
Free inodes:              244
First block:              1
Block size:               1024
Fragment size:            1024
Reserved GDT blocks:      7
Blocks per group:         8192
Fragments per group:      8192
Inodes per group:         256
Inode blocks per group:   32
Filesystem created:       Thu Apr  2 23:53:31 2020
Last mount time:          Thu Apr  2 23:56:50 2020
Last write time:          Thu Apr  2 23:57:02 2020
Mount count:              2
Maximum mount count:      -1
Last checked:             Thu Apr  2 23:53:31 2020
Check interval:           0 (<none>)
Lifetime writes:          8 kB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               128
Default directory hash:   half_md4
Directory Hash Seed:      15323132-fa4b-4822-ab37-f49b15b57487

Group 0: (Blocks 1-2047)
  Primary superblock at 1, Group descriptors at 2-2
  Reserved GDT blocks at 3-9
  Block bitmap at 10 (+9), Inode bitmap at 11 (+10)
  Inode table at 12-43 (+11)
  1988 free blocks, 242 free inodes, 3 directories
  Free blocks: 59-512, 514-2047
  Free inodes: 15-256 
```

### Metadata overview

It's a lot of information! From here, we can see familiar things, echoing what we have learnt about ext2 in class:

* There are 256 inodes in total ("Inode count..."). These inodes are pre-allocated when the filesystem is initialized. 
* There are 2048 blocks ("Block count...". Since this disk is 2MB in total, each block is 2MB/2048=1KB)
* The first inode is inode 11 ("First inode...)". Which file is this? We will see soon. 
* Each inode has 128 bytes ("Inode size..."). Recall this is for per file metadata plus a bunch of pointers to disk blocks of this file. 

### Block group 

You also see "groups" in the above output. Recall that ext2 partitions all blocks of a disk into groups, so that access within each group has better spatial locality. The output shows that each group can have up to 8192 blocks ("Blocks per group..."). As our disk is as small as 2048 blocks, ext2 creates only one group (Group 0), which spans all blocks of the disk. 

### Bitmaps  
Recall that ext2 use bitmaps to track free/used blocks as well as free/used inodes. In such a bitmap, 1 means used, 0 means free. 

Where are the bitmaps stored on disk? Check the last few lines of the output. Block bitmap starts from block 10; inode bitmap starts from block 11. Block 12-43 are for the actual inodes. 
<!---Since we have 2048 blocks, we need 2048 bits which are 2 blocks? why there's only block 10?> --->

Let's dump the bitmap contents from the disk image. 
```
# dump inode bitmap. 256 inodes (display in 32 octets). Bitmap at block 11. 
$ sudo dd if=/dev/loop2 bs=1024 skip=11 count=1 status=none |xxd -b -l 32
00000000: 11111111 00111111 00000000 00000000 00000000 00000000  .?....
00000006: 00000000 00000000 00000000 00000000 00000000 00000000  ......
0000000c: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000012: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000018: 00000000 00000000 00000000 00000000 00000000 00000000  ......
0000001e: 00000000 00000000                                      ..
 
# dump block bitmap. 2048 blocks (256 display octets). Bitmap at block 10
$ sudo dd if=/dev/loop2 bs=1024 skip=10 count=1 status=none |xxd -b -l 256
00000000: 11111111 11111111 11111111 11111111 11111111 11111111  ......
00000006: 11111111 00000011 00000000 00000000 00000000 00000000  ......
0000000c: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000012: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000018: 00000000 00000000 00000000 00000000 00000000 00000000  ......
0000001e: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000024: 00000000 00000000 00000000 00000000 00000000 00000000  ......
```

In the commands above, `dd` will read the contents of the disk device (note: your /dev/loop device name may vary). "bs=1024 skip=11 count=1" means that `dd` will skip the first 11 blocks and read in 1 block, using 1024 bytes as the block size. 

>  note "blocks" here are just data blocks for dd to read in; it's orthogonal to disk blocks)

The content of the read block is piped to `xxd`, whose job is to display the content nicely in binary format. "-b" means xxd will display **every bit**. "-l" means the number of "octets" to display. Each octets is a "display group" of eight characters (e.g. "11111111"). Since we are display bits, each chaterters is a bit and one octet is a byte. 

## Step 3. Inspect per-file metadata

### Check the directories again 

We can use `ls` to get more information. 

```
$ ls -ila
total 54
      2 drwxr-xr-x  4 root root  1024 Apr  6 11:35 .
5636097 drwxrwxrwt 38 root root 36864 Apr  6 11:35 ..
     12 -rw-r--r--  1 root root     0 Apr  2 23:56 aa
     11 drwx------  2 root root 12288 Apr  2 23:53 lost+found
     13 drwxr-xr-x  2 root root  1024 Apr  6 11:35 testdir
     
$ ls -ila testdir/
total 2
13 drwxr-xr-x 2 root root 1024 Apr  6 11:35 .
 2 drwxr-xr-x 4 root root 1024 Apr  6 11:35 ..
```

Here "-i" means to print the inode number; "-l" asks for the long listing format; "-a" asks for listing files/dirs that start with "." which are normally hidden. 

In the output, "total" shows the number of blocks used by the listed directories. The first column shows the inode number for each file/dir. For instance, inode 12 is for file "aa". 

From dump2efs we know the first inode is inode 11. Do you see it here? 

### Dump inode contents

We will use `debugfs`, which is a powerful debugger for the ext family of filesystems (ext2/3/4). 

```
# inode 12. testfile “aa”
$ sudo debugfs -R "stat <12>" /dev/loop2
Inode: 12   Type: regular    Mode:  0644   Flags: 0x0
Generation: 4138714773    Version: 0x00000001
User:     0   Group:     0   Size: 10
File ACL: 0    Directory ACL: 0
Links: 1   Blockcount: 2
Fragment:  Address: 0    Number: 0    Size: 0
ctime: 0x5e8b4e5f -- Mon Apr  6 11:44:31 2020
atime: 0x5e8b4e4d -- Mon Apr  6 11:44:13 2020
mtime: 0x5e8b4e5f -- Mon Apr  6 11:44:31 2020
BLOCKS:
(0):513
TOTAL: 1
 
# inode 2. root dir
$ sudo debugfs -R "stat <2>" /dev/loop2
Inode: 2   Type: directory    Mode:  0755   Flags: 0x0
Generation: 0    Version: 0x00000002
User:     0   Group:     0   Size: 1024
File ACL: 0    Directory ACL: 0
Links: 4   Blockcount: 2
Fragment:  Address: 0    Number: 0    Size: 0
ctime: 0x5e8b4c40 -- Mon Apr  6 11:35:28 2020
atime: 0x5e8b4c41 -- Mon Apr  6 11:35:29 2020
mtime: 0x5e8b4c40 -- Mon Apr  6 11:35:28 2020
BLOCKS:
(0):44
TOTAL: 1
```

To  invoke debugfs, we supply the disk device name (e.g. /dev/loop2) and a command (-R ...) that we want debugfs to execute. In this case, our command is "stat", which is to dump contents of a given inode, e.g. inode 12 (file "aa") and inode 2 (directory "/"). More debugfs details [here](https://linux.die.net/man/8/debugfs). 

So much information to see from inodes! File types ("regular" and "directory"), access times, blocks, etc. These are exactly what we mentioned in the lectures. 

There are alternative debugfs commands that show same information but in a slight different format: 

```
# Showing inodes and type (2=dir, 1=file)
$ sudo debugfs -R "ls -l <2>" /dev/loop2
      2   40755 (2)      0      0    1024  6-Apr-2020 11:35 .
      2   40755 (2)      0      0    1024  6-Apr-2020 11:35 ..
     11   40700 (2)      0      0   12288  2-Apr-2020 23:53 lost+found
     12  100644 (1)      0      0       0  2-Apr-2020 23:56 aa
     13   40755 (2)      0      0    1024  6-Apr-2020 11:35 testdir

# dump the block IDs used by the root dir
$ sudo debugfs -R "blocks /." /dev/loop2
debugfs 1.42.13 (17-May-2015)
44
```
Keep in mind: all such information comes from inodes. 

## Step 4. Inspect filedata

### A directory

Recall a directory is nothing but a special file. Let's dump the content of the root dir "/". We've learnt that the root dir ("/") has inode 2. Fire up debugfs!

```
$ sudo debugfs -R "cat <2>" /dev/loop2 | hexdump -C
debugfs 1.42.13 (17-May-2015)
00000000  02 00 00 00 0c 00 01 02  2e 00 00 00 02 00 00 00  |................|
00000010  0c 00 02 02 2e 2e 00 00  0b 00 00 00 14 00 0a 02  |................|
00000020  6c 6f 73 74 2b 66 6f 75  6e 64 00 00 0c 00 00 00  |lost+found......|
00000030  0c 00 02 01 61 61 00 00  0d 00 00 00 c8 03 07 02  |....aa..........|
00000040  74 65 73 74 64 69 72 00  00 00 00 00 00 00 00 00  |testdir.........|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400
```
Here, "cat <2>" is for debugfs to dump the raw content of file with inode 2. The content is piped to `hexdump`, which nicely formats the data in hex format. "-C" tells hexdump to put hex display  (left) and the ASCII display (right) side by side. ASCII is for human to quickly eyeball interesting textual strings. 

> the last row, 0400, mark the end offset of the file. 

What we see here? "lost+found", "aa", "testdir" ... They are the filenames belonging to "/". So, "/" indeed stores the names of enclosed files. 

### A regular file

inode 12 is for file "aa", which has one line "mycontent". We could ask debugfs to dump that info. 

```
sudo debugfs -R "cat <12>" /dev/loop10 | hexdump -C
debugfs 1.44.1 (24-Mar-2018)
00000000  6d 79 63 6f 6e 74 65 6e  74 0a                    |mycontent.|
0000000a
```
Alternatively, we can locate the block of the file (we know it only spans 1 block) and dump the content of that block. 

```
$ sudo debugfs -R "blocks <12>" /dev/loop2
debugfs 1.42.13 (17-May-2015)
513

# direct inspect block 513 on the disk image
$ sudo dd if=/dev/loop2 bs=1024 skip=513 count=1 status=none |hexdump -C
00000000  6d 79 63 6f 6e 74 65 6e  74 0a 00 00 00 00 00 00  |mycontent.......|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400
```

The first command shows the block is 513. The second command directly dump block 513. We see the content "mycontent"!

### A symbolic link

File "aa-symlink" is a symbolic link pointing to file "aa". 

```
$ ls -ila
total 55
      2 drwxr-xr-x  4 root root  1024 Apr  6 12:30 .
5636097 drwxrwxrwt 38 root root 36864 Apr  6 12:31 ..
     12 -rw-r--r--  1 root root    10 Apr  6 11:44 aa
     14 lrwxrwxrwx  1 root root     2 Apr  6 12:30 aa-symlink -> aa
     11 drwx------  2 root root 12288 Apr  2 23:53 lost+found
     13 drwxr-xr-x  2 root root  1024 Apr  6 11:35 testdir
```
Recall that a symbolic link is a special file containing the path of another file ("aa" in our case). aa-symlink and aa should be two files, i.e. having separate inodes. This is confirmed: 
inode 12 is for "aa" and inode 14 is for "aa-symlink". 

Can we inspect the content of "aa-symlink"? We expect to see the string "aa" there. However, this seems not as straightforward: 

```
xzl@precision[myfs]$ stat aa-symlink
  File: 'aa-symlink' -> 'aa'
  Size: 2               Blocks: 0          IO Block: 1024   symbolic link
Device: 702h/1794d      Inode: 14          Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-04-06 12:31:13.000000000 -0400
Modify: 2020-04-06 12:30:57.000000000 -0400
Change: 2020-04-06 12:30:57.000000000 -0400
 Birth: -
```
So there is not block for "aa-symlink" (Blocks: 0)? The reason is that the string is inlined in the inode. So no block is used. 

 *Symbolic links are also filesystem objects with inodes. For all symlink shorter than 60 bytes long, the data is stored within the inode itself; it uses the fields which would normally be used to store the pointers to data blocks. This is a worthwhile optimisation as it we avoid allocating a full block for the symlink, and most symlinks are less than 60 characters long.* https://www.nongnu.org/ext2-doc/ext2.html

## Step 0. Initialize the filesystem

Why we go back to step 0? Because this is how we created the disk image and initialized the filesystem that we gave to you in the first place. Let's close the loop. 

### Create a disk image
```
xzl@precision[tmp]$ dd if=/dev/zero of=/tmp/disk.img bs=2M count=1
1+0 records in
1+0 records out
2097152 bytes (2.1 MB, 2.0 MiB) copied, 0.00403917 s, 519 MB/s
```
We see `dd` above. It's a simple tool write/read contents to/from a file. We use it to create a  file of 2MB (note bs, the block size). "if" means input file, i.e. where we get content from. "/dev/zero" is a special (or pseudo) file implemented by the kernel. Whenever you read from it, the kernel will just return 0. As a result, disk.img contains all zeros. 

Now, "disk.img" is just a normal file. Nothing special. 

### Set up a loop device atop the disk image

```
xzl@precision[tmp]$ sudo losetup -fP disk.img
xzl@precision[tmp]$ losetup -a
/dev/loop1: []: (/tmp/disk.img)
```

To be able to create a filesystem on the disk image, we first ask the kernel to set up a loop device atop the disk image (disk.img). `losetup` is for this purpose. "-f" find the first unused loop device; "-P" force scanning the partition table. [Details](https://man7.org/linux/man-pages/man8/losetup.8.html). 

### Initialize an ext2 filesystem
```
$ sudo mkfs.ext2 /dev/loop1
mke2fs 1.42.13 (17-May-2015)
Discarding device blocks: done
Creating filesystem with 2048 1k blocks and 256 inodes
 
Allocating group tables: done
Writing inode tables: done
Writing superblocks and filesystem accounting information: done
```
The superblock, all inodes, bitmaps... are created at this moment. We end up having a blank ext2 filesystem. 

Check our new filesystem is mounted, create the files/dirs for testing (the ones you have tinkered with), and unmount which writes the contents back to disk.img: 
```
$ mount |grep myfs
/dev/loop1 on /tmp/myfs type ext2 (rw,relatime)

$ cd /tmp/myfs
$ sudo echo 'test' > aa
$ sudo mkdir testdir
$ sudo ln -s aa aa-symlink

$ sudo umount /tmp/myfs
```

### Inspect the disk image

```
$ ls -la disk.img
-rw-r--r-- 1 xzl xzl 2097152 Apr  6 13:33 disk.img

# the “file” utility – smart enough to detect a filesystem within
$ file disk.img
disk.img: Linux rev 1.0 ext2 filesystem data, UUID=6430eccd-11d0-4ea9-bd26-ce6e946dc02b (large files)

# there will be tons of ‘0’s. So zip it for distributing
$ gzip disk.img
$ ls -la disk.img.gz
-rw-r--r-- 1 xzl xzl 2657 Apr  6 13:33 disk.img.gz
```