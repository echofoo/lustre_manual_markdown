# Managing the File System and I/O

- [Managing the File System and I/O](#managing-the-file-system-and-io)
  * [Handling Full OSTs](#handling-full-osts)
    + [Checking OST Space Usage](#checking-ost-space-usage)
    + [Disabling creates on a Full OST](#disabling-creates-on-a-full-ost)
    + [Migrating Data within a File System](#migrating-data-within-a-file-system)
    + [Returning an Inactive OST Back Online](#returning-an-inactive-ost-back-online)
    + [Migrating Metadata within a Filesystem](#migrating-metadata-within-a-filesystem)
      - [Whole Directory Migration](#whole-directory-migration)
      - [Striped Directory Migration](#striped-directory-migration)
  * [Creating and Managing OST Pools](#creating-and-managing-ost-pools)
    + [Working with OST Pools](#working-with-ost-pools)
      - [Using the lfs Command with OST Pools](#using-the-lfs-command-with-ost-pools)
    + [Tips for Using OST Pools](#tips-for-using-ost-pools)
  * [Adding an OST to a Lustre File System](#adding-an-ost-to-a-lustre-file-system)
  * [Performing Direct I/O](#performing-direct-io)
    + [Making File System Objects Immutable](#making-file-system-objects-immutable)
  * [Other I/O Options](#other-io-options)
    + [Lustre Checksums](#lustre-checksums)
      - [Changing Checksum Algorithms](#changing-checksum-algorithms)
    + [Ptlrpc Client Thread Pool](#ptlrpc-client-thread-pool)
      - [ptlrpcd parameters](#ptlrpcd-parameters)

## Handling Full OSTs

Sometimes a Lustre file system becomes unbalanced, often due to incorrectly-specified stripe settings, or when very large files are created that are not striped over all of the OSTs. Lustre will automatically avoid allocating new files on OSTs that are full. If an OST is completely full and more data is written to files already located on that OST, an error occurs. The procedures below describe how to handle a full OST.

The MDS will normally handle space balancing automatically at file creation time, and this procedure is normally not needed, but manual data migration may be desirable in some cases (e.g. creating very large files that would consume more than the total free space of the full OSTs).

 Checking OST Space Usage

The example below shows an unbalanced file system:

```
client# lfs df -h
UUID                       bytes           Used            Available       \
Use%            Mounted on
testfs-MDT0000_UUID        4.4G            214.5M          3.9G            \
4%              /mnt/testfs[MDT:0]
testfs-OST0000_UUID        2.0G            751.3M          1.1G            \
37%             /mnt/testfs[OST:0]
testfs-OST0001_UUID        2.0G            755.3M          1.1G            \
37%             /mnt/testfs[OST:1]
testfs-OST0002_UUID        2.0G            1.7G            155.1M          \
86%             /mnt/testfs[OST:2] ****
testfs-OST0003_UUID        2.0G            751.3M          1.1G            \
37%             /mnt/testfs[OST:3]
testfs-OST0004_UUID        2.0G            747.3M          1.1G            \
37%             /mnt/testfs[OST:4]
testfs-OST0005_UUID        2.0G            743.3M          1.1G            \
36%             /mnt/testfs[OST:5]
 
filesystem summary:        11.8G           5.4G            5.8G            \
45%             /mnt/testfs
```

In this case, OST0002 is almost full and when an attempt is made to write additional information to the file system (even with uniform striping over all the OSTs), the write command fails as follows:

```
client# lfs setstripe /mnt/testfs 4M 0 -1
client# dd if=/dev/zero of=/mnt/testfs/test_3 bs=10M count=100
dd: writing '/mnt/testfs/test_3': No space left on device
98+0 records in
97+0 records out
1017192448 bytes (1.0 GB) copied, 23.2411 seconds, 43.8 MB/s
```
### Checking OST Space Usage

The example below shows an unbalanced file system:

```
client# lfs df -h
UUID                       bytes           Used            Available       \
Use%            Mounted on
testfs-MDT0000_UUID        4.4G            214.5M          3.9G            \
4%              /mnt/testfs[MDT:0]
testfs-OST0000_UUID        2.0G            751.3M          1.1G            \
37%             /mnt/testfs[OST:0]
testfs-OST0001_UUID        2.0G            755.3M          1.1G            \
37%             /mnt/testfs[OST:1]
testfs-OST0002_UUID        2.0G            1.7G            155.1M          \
86%             /mnt/testfs[OST:2] ****
testfs-OST0003_UUID        2.0G            751.3M          1.1G            \
37%             /mnt/testfs[OST:3]
testfs-OST0004_UUID        2.0G            747.3M          1.1G            \
37%             /mnt/testfs[OST:4]
testfs-OST0005_UUID        2.0G            743.3M          1.1G            \
36%             /mnt/testfs[OST:5]
 
filesystem summary:        11.8G           5.4G            5.8G            \
45%             /mnt/testfs
```

In this case, OST0002 is almost full and when an attempt is made to write additional information to the file system (even with uniform striping over all the OSTs), the write command fails as follows:

```
client# lfs setstripe /mnt/testfs 4M 0 -1
client# dd if=/dev/zero of=/mnt/testfs/test_3 bs=10M count=100
dd: writing '/mnt/testfs/test_3': No space left on device
98+0 records in
97+0 records out
1017192448 bytes (1.0 GB) copied, 23.2411 seconds, 43.8 MB/s
```

### Disabling creates on a Full OST

To avoid running out of space in the file system, if the OST usage is imbalanced and one or more OSTs are close to being full while there are others that have a lot of space, the MDS will typically avoid file creation on the full OST(s) automatically. The full OSTs may optionally be deactivated manually on the MDS to ensure the MDS will not allocate new objects there.

1. Log into the MDS server and use the `lctl` command to stop new object creation on the full OST(s):

   ```
   mds# lctl set_param osp.fsname-OSTnnnn*.max_create_count=0
   ```

When new files are created in the file system, they will only use the remaining OSTs. Either manual space rebalancing can be done by migrating data to other OSTs, as shown in the next section, or normal file deletion and creation can passively rebalance the space usage.

### Migrating Data within a File System

If there is a need to move the file data from the current OST(s) to new OST(s), the data must be migrated (copied) to the new location. The simplest way to do this is to use the `lfs_migrate` command, as described in [the section called “ Adding a New OST to a Lustre File System”](03.03-Lustre%20Maintenance.md#adding-a-new-ost-to-a-lustre-file-system).

### Returning an Inactive OST Back Online

Once the full OST(s) no longer are severely imbalanced, due to either active or passive data redistribution, they should be reactivated so they will again have new files allocated on them.

```
[mds]# lctl set_param osp.testfs-OST0002.max_create_count=20000
```

### Migrating Metadata within a Filesystem

Introduced in Lustre 2.8

#### Whole Directory Migration

Lustre software version 2.8 includes a feature to migrate metadata (directories and inodes therein) between MDTs. This migration can only be performed on whole directories. Striped directories are not supported until Lustre 2.12. For example, to migrate the contents of the `/testfs/remotedir` directory from the MDT where it currently is located to MDT0000 to allow that MDT to be removed, the sequence of commands is as follows:

```
$ cd /testfs 
$ lfs getdirstripe -m ./remotedir *which MDT is dir on?* 
1 
$ touch ./remotedir/file.{1,2,3}.txt*create test files* 
$ lfs getstripe -m ./remotedir/file.*.txt*check files are on MDT0001* 
1 
1 
1 
$ lfs migrate -m 0 ./remotedir *migrate testremote to MDT0000* 
$ lfs getdirstripe -m ./remotedir *which MDT is dir on now?* 
0 
$ lfs getstripe -m ./remotedir/file.*.txt*check files are on MDT0000* 
0 
0 
0
```

For more information, see `man lfs-migrate`.

##### Warning

During migration each file receives a new identifier (FID). As a consequence, the file will report a new inode number to userspace applications. Some system tools (for example, backup and archiving tools, NFS, Samba) that identify files by inode number may consider the migrated files to be new, even though the contents are unchanged. If a Lustre system is re-exporting to NFS, the migrated files may become inaccessible during and after migration if the client or server are caching a stale file handle with the old FID. Restarting the NFS service will flush the local file handle cache, but clients may also need to be restarted as they may cache stale file handles as well.



Introduced in Lustre 2.12

#### Striped Directory Migration

Lustre 2.8 included a feature to migrate metadata (directories and inodes therein) between MDTs, however it did not support migration of striped directories, or changing the stripe count of an existing directory. Lustre 2.12 adds support for migrating and restriping directories. The `lfs migrate -m` command can only only be performed on whole directories, though it will migrate both the specified directory and its sub-entries recursively. For example, to migrate the contents of a large directory `/testfs/largedir` from its current location on MDT0000 to MDT0001 and MDT0003, run the following command:

`$ lfs migrate -m 1,3 /testfs/largedir`

Metadata migration will migrate file dirent and inode to other MDTs, but it won't touch file data. During migration, directory and its sub-files can be accessed like normal ones, though the same warning above applies to tools that depend on the file inode number. Migration may fail for various reasons such as MDS restart, or disk full. In those cases, some of the sub-files may have been migrated to the new MDTs, while others are still on the original MDT. The files can be accessed normally. The same `lfs migrate -m` command should be executed again when these issues are fixed to finish this migration. However, you cannot abort a failed migration, or migrate to different MDTs from previous migration command.

#### Directory Restriping

Lustre 2.14 includs a feature to change the stripe count of an existing directory. The `lfs setdirstripe -c`command can be performed on an existing directory to change its stripe count. For example, a directory `/testfs/testdir` is becoming large, run the following command to increase its stripe count to 2:

```
$ lfs setdirstripe -c 2 /testfs/testdir
```

By default directory restriping will migrate sub-file dirents only, but it won't move inodes. To enable moving both dirents and inodes, run the following command on all MDS's:

```
mds$ lctl set_param mdt.*.dir_restripe_nsonly=0
```

It's not allowed to specify MDTs in directory restriping, instead server will pick MDTs for the added stripes by space and inode usages. During restriping, directory and its sub-files can be accessed like normal ones, which is the same as directory migration. Similarly you cannot abort a failed restriping, and server will resume the failed restriping automatically when it notices an unfinished restriping.

#### 23.1.5.4. Directory Auto-Split

Lustre 2.14 includs a feature to automatically increase the stripe count of a directory when it becomes large. This can be enabled by the following command:

```
mds$ lctl set_param mdt.*.enable_dir_auto_split=1
```

The sub file count that triggers directory auto-split is 50k, and it can be changed by the following command:

```
mds$ lctl set_param mdt.*.dir_split_count=value
```

The directory stripe count will be increased from 0 to 4 if it's a plain directory, and from 4 to 8 upon the second split, and so on. However the final stripe count won't exceed total MDT count, and it will stop splitting when it's distributed among all MDTs. This delta value can be changed by the following command:

```
mds$ lctl set_param mdt.*.dir_split_delta=value
```

## Creating and Managing OST Pools

The OST pools feature enables users to group OSTs together to make object placement more flexible. A 'pool' is the name associated with an arbitrary subset of OSTs in a Lustre cluster.

OST pools follow these rules:

- An OST can be a member of multiple pools.
- No ordering of OSTs in a pool is defined or implied.
- Stripe allocation within a pool follows the same rules as the normal stripe allocator.
- OST membership in a pool is flexible, and can change over time.

When an OST pool is defined, it can be used to allocate files. When file or directory striping is set to a pool, only OSTs in the pool are candidates for striping. If a stripe_index is specified which refers to an OST that is not a member of the pool, an error is returned.

OST pools are used only at file creation. If the definition of a pool changes (an OST is added or removed or the pool is destroyed), already-created files are not affected.

**Note**

An error ( `EINVAL`) results if you create a file using an empty pool.

**Note**

If a directory has pool striping set and the pool is subsequently removed, the new files created in this directory have the (non-pool) default striping pattern for that directory applied and no error is returned.

### Working with OST Pools

OST pools are defined in the configuration log on the MGS. Use the lctl command to:

- Create/destroy a pool
- Add/remove OSTs in a pool
- List pools and OSTs in a specific pool

The lctl command MUST be run on the MGS. Another requirement for managing OST pools is to either have the MDT and MGS on the same node or have a Lustre client mounted on the MGS node, if it is separate from the MDS. This is needed to validate the pool commands being run are correct.

**Caution**

Running the `writeconf` command on the MDS erases all pools information (as well as any other parameters set using `lctl conf_param`). We recommend that the pools definitions (and `conf_param`settings) be executed using a script, so they can be reproduced easily after a `writeconf` is performed.

To create a new pool, run:

```
mgs# lctl pool_new 
fsname.
poolname
```

**Note**

The pool name is an ASCII string up to 15 characters.

To add the named OST to a pool, run:

```
mgs# lctl pool_add 
fsname.
poolname 
ost_list
```

Where:

- `*ost_list*is *fsname*-OST *index_range*`
- `*index_range*is *ost_index_start*- *ost_index_end[,index_range]*` or `*ost_index_start*-*ost_index_end/step*`

If the leading `*fsname* `and/or ending `_UUID` are missing, they are automatically added.

For example, to add even-numbered OSTs to `pool1` on file system `testfs`, run a single command ( `pool_add`) to add many OSTs to the pool at one time:

```
lctl pool_add testfs.pool1 OST[0-10/2]
```

**Note**

Each time an OST is added to a pool, a new `llog` configuration record is created. For convenience, you can run a single command.

To remove a named OST from a pool, run:

```
mgs# lctl pool_remove 
fsname.
poolname 
ost_list
```

To destroy a pool, run:

```
mgs# lctl pool_destroy 
fsname.
poolname
```

**Note**

All OSTs must be removed from a pool before it can be destroyed.

To list pools in the named file system, run:

```
mgs# lctl pool_list 
fsname|pathname
```

To list OSTs in a named pool, run:

```
lctl pool_list 
fsname.
poolname
```

#### Using the lfs Command with OST Pools

Several lfs commands can be run with OST pools. Use the `lfs setstripe` command to associate a directory with an OST pool. This causes all new regular files and directories in the directory to be created in the pool. The lfs command can be used to list pools in a file system and OSTs in a named pool.

To associate a directory with a pool, so all new files and directories will be created in the pool, run:

```
client# lfs setstripe --pool|-p pool_name 
filename|dirname 
```

To set striping patterns, run:

```
client# lfs setstripe [--size|-s stripe_size] [--offset|-o start_ost]
           [--stripe-count|-c stripe_count] [--overstripe-count|-C stripe_count]
 		   [--pool|-p pool_name]
           
dir|filename
```

**Note **

If you specify striping with an invalid pool name, because the pool does not exist or the pool name was mistyped, `lfs setstripe` returns an error. Run `lfs pool_list` to make sure the pool exists and the pool name is entered correctly.

**Note**

The `--pool` option for lfs setstripe is compatible with other modifiers. For example, you can set striping on a directory to use an explicit starting index.

### Tips for Using OST Pools

Here are several suggestions for using OST pools.

- A directory or file can be given an extended attribute (EA), that restricts striping to a pool.
- Pools can be used to group OSTs with the same technology or performance (slower or faster), or that are preferred for certain jobs. Examples are SATA OSTs versus SAS OSTs or remote OSTs versus local OSTs.
- A file created in an OST pool tracks the pool by keeping the pool name in the file LOV EA.

## Adding an OST to a Lustre File System

To add an OST to existing Lustre file system:

1. Add a new OST by passing on the following commands, run:

   ```
   oss# mkfs.lustre --fsname=testfs --mgsnode=mds16@tcp0 --ost --index=12 /dev/sda
   oss# mkdir -p /mnt/testfs/ost12
   oss# mount -t lustre /dev/sda /mnt/testfs/ost12
   ```

2. Migrate the data (possibly).

   The file system is quite unbalanced when new empty OSTs are added. New file creations are automatically balanced. If this is a scratch file system or files are pruned at a regular interval, then no further work may be needed. Files existing prior to the expansion can be rebalanced with an in-place copy, which can be done with a simple script.

   The basic method is to copy existing files to a temporary file, then move the temp file over the old one. This should not be attempted with files which are currently being written to by users or applications. This operation redistributes the stripes over the entire set of OSTs.

   A very clever migration script would do the following:

   - Examine the current distribution of data.
   - Calculate how much data should move from each full OST to the empty ones.
   - Search for files on a given full OST (using `lfs getstripe`).
   - Force the new destination OST (using `lfs setstripe`).
   - Copy only enough files to address the imbalance.

If a Lustre file system administrator wants to explore this approach further, per-OST disk-usage statistics can be found under `/proc/fs/lustre/osc/*/rpc_stats`

## Performing Direct I/O

The Lustre software supports the `O_DIRECT` flag to open.

Applications using the `read()` and `write()` calls must supply buffers aligned on a page boundary (usually 4 K). If the alignment is not correct, the call returns `-EINVAL`. Direct I/O may help performance in cases where the client is doing a large amount of I/O and is CPU-bound (CPU utilization 100%).

### Making File System Objects Immutable

An immutable file or directory is one that cannot be modified, renamed or removed. To do this:

```
chattr +i 
file
```

To remove this flag, use `chattr -i`

## Other I/O Options

This section describes other I/O options, including checksums, and the ptlrpcd thread pool.

### Lustre Checksums

To guard against network data corruption, a Lustre client can perform two types of data checksums: in-memory (for data in client memory) and wire (for data sent over the network). For each checksum type, a 32-bit checksum of the data read or written on both the client and server is computed, to ensure that the data has not been corrupted in transit over the network. The `ldiskfs` backing file system does NOT do any persistent checksumming, so it does not detect corruption of data in the OST file system.

The checksumming feature is enabled, by default, on individual client nodes. If the client or OST detects a checksum mismatch, then an error is logged in the syslog of the form:

```
LustreError: BAD WRITE CHECKSUM: changed in transit before arrival at OST: \
from 192.168.1.1@tcp inum 8991479/2386814769 object 1127239/0 extent [10240\
0-106495]
```

If this happens, the client will re-read or re-write the affected data up to five times to get a good copy of the data over the network. If it is still not possible, then an I/O error is returned to the application.

To enable both types of checksums (in-memory and wire), run:

```
lctl set_param llite.*.checksum_pages=1
```

To disable both types of checksums (in-memory and wire), run:

```
lctl set_param llite.*.checksum_pages=0
```

To check the status of a wire checksum, run:

```
lctl get_param osc.*.checksums
```

#### Changing Checksum Algorithms

By default, the Lustre software uses the adler32 checksum algorithm, because it is robust and has a lower impact on performance than crc32. The Lustre file system administrator can change the checksum algorithm via `lctl get_param`, depending on what is supported in the kernel.

To check which checksum algorithm is being used by the Lustre software, run:

```
$ lctl get_param osc.*.checksum_type
```

To change the wire checksum algorithm, run:

```
$ lctl set_param osc.*.checksum_type=
algorithm
```

**Note **

The in-memory checksum always uses the adler32 algorithm, if available, and only falls back to crc32 if adler32 cannot be used.

In the following example, the `lctl get_param` command is used to determine that the Lustre software is using the adler32 checksum algorithm. Then the `lctl set_param` command is used to change the checksum algorithm to crc32. A second `lctl get_param` command confirms that the crc32 checksum algorithm is now in use.

```
$ lctl get_param osc.*.checksum_type
osc.testfs-OST0000-osc-ffff81012b2c48e0.checksum_type=crc32 [adler]
$ lctl set_param osc.*.checksum_type=crc32
osc.testfs-OST0000-osc-ffff81012b2c48e0.checksum_type=crc32
$ lctl get_param osc.*.checksum_type
osc.testfs-OST0000-osc-ffff81012b2c48e0.checksum_type=[crc32] adler
```
### Ptlrpc Client Thread Pool

The use of large SMP nodes for Lustre clients requires significant parallelism within the kernel to avoid cases where a single CPU would be 100% utilized and other CPUs would be relativity idle. This is especially noticeable when a single thread traverses a large directory.

The Lustre client implements a PtlRPC daemon thread pool, so that multiple threads can be created to serve asynchronous RPC requests, even if only a single userspace thread is running. The number of ptlrpcd threads spawned is controlled at module load time using module options. By default two service threads are spawned per CPU socket.

One of the issues with thread operations is the cost of moving a thread context from one CPU to another with the resulting loss of CPU cache warmth. To reduce this cost, PtlRPC threads can be bound to a CPU. However, if the CPUs are busy, a bound thread may not be able to respond quickly, as the bound CPU may be busy with other tasks and the thread must wait to schedule.

Because of these considerations, the pool of ptlrpcd threads can be a mixture of bound and unbound threads. The system operator can balance the thread mixture based on system size and workload.

#### ptlrpcd parameters

These parameters should be set in `/etc/modprobe.conf` or in the `etc/modprobe.d` directory, as options for the ptlrpc module.

```
options ptlrpcd ptlrpcd_per_cpt_max=XXX
```

Sets the number of ptlrpcd threads created per socket. The default if not specified is two threads per CPU socket, including hyper-threaded CPUs. The lower bound is 2 threads per socket.

```
options ptlrpcd ptlrpcd_bind_policy=[1-4]
```

Controls the binding of threads to CPUs. There are four policy options.

- `PDB_POLICY_NONE`(ptlrpcd_bind_policy=1) All threads are unbound.
- `PDB_POLICY_FULL`(ptlrpcd_bind_policy=2) All threads attempt to bind to a CPU.
- `PDB_POLICY_PAIR`(ptlrpcd_bind_policy=3) This is the default policy. Threads are allocated as a bound/unbound pair. Each thread (bound or free) has a partner thread. The partnering is used by the ptlrpcd load policy, which determines how threads are allocated to CPUs.
- `PDB_POLICY_NEIGHBOR`(ptlrpcd_bind_policy=4) Threads are allocated as a bound/unbound pair. Each thread (bound or free) has two partner threads.
