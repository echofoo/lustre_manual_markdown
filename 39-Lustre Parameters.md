# Lustre Parameters

- [Lustre Parameters](#lustre-parameters)
  * [Introduction to Lustre Parameters](#introduction-to-lustre-parameters)
    + [Identifying Lustre File Systems and Servers](#identifying-lustre-file-systems-and-servers)
  * [Tuning Multi-Block Allocation (mballoc)](#tuning-multi-block-allocation-mballoc)
  * [Monitoring Lustre File System I/O](#monitoring-lustre-file-system-io)
    + [Monitoring the Client RPC Stream](#monitoring-the-client-rpc-stream)
    + [Monitoring Client Activity](#monitoring-client-activity)
    + [Monitoring Client Read-Write Offset Statistics](#monitoring-client-read-write-offset-statistics)
    + [Monitoring Client Read-Write Extent Statistics](#monitoring-client-read-write-extent-statistics)
      - [Client-Based I/O Extent Size Survey](#client-based-io-extent-size-survey)
      - [Per-Process Client I/O Statistics](#per-process-client-io-statistics)
    + [Monitoring the OST Block I/O Stream](#monitoring-the-ost-block-io-stream)
  * [Tuning Lustre File System I/O](#tuning-lustre-file-system-io)
    + [Tuning the Client I/O RPC Stream](#tuning-the-client-io-rpc-stream)
    + [Tuning File Readahead and Directory Statahead](#tuning-file-readahead-and-directory-statahead)
      - [Tuning File Readahead](#tuning-file-readahead)
      - [Tuning Directory Statahead and AGL](#tuning-directory-statahead-and-agl)
    + [Tuning OSS Read Cache](#tuning-oss-read-cache)
      - [Using OSS Read Cache](#using-oss-read-cache)
    + [Enabling OSS Asynchronous Journal Commit](#enabling-oss-asynchronous-journal-commit)
    + [Tuning the Client Metadata RPC Stream](#tuning-the-client-metadata-rpc-stream)
      - [Configuring the Client Metadata RPC Stream](#configuring-the-client-metadata-rpc-stream)
      - [Monitoring the Client Metadata RPC Stream](#monitoring-the-client-metadata-rpc-stream)
  * [Configuring Timeouts in a Lustre File System](#configuring-timeouts-in-a-lustre-file-system)
    + [Configuring Adaptive Timeouts](#configuring-adaptive-timeouts)
      - [Interpreting Adaptive Timeout Information](#interpreting-adaptive-timeout-information)
    + [Setting Static Timeouts](#setting-static-timeouts)
  * [Monitoring LNet](#monitoring-lnet)
  * [Allocating Free Space on OSTs](#allocating-free-space-on-osts)
  * [Configuring Locking](#configuring-locking)
  * [Setting MDS and OSS Thread Counts](#setting-mds-and-oss-thread-counts)
  * [Enabling and Interpreting Debugging Logs](#enabling-and-interpreting-debugging-logs)
    + [Interpreting OST Statistics](#interpreting-ost-statistics)
    + [Interpreting MDT Statistics](#interpreting-mdt-statistics)


The `/proc` and `/sys` file systems acts as an interface to internal data structures in the kernel. This chapter describes parameters and tunables that are useful for optimizing and monitoring aspects of a Lustre file system. It includes these sections:

- [the section called “Enabling and Interpreting Debugging Logs”](#enabling-and-interpreting-debugging-logs)

## Introduction to Lustre Parameters

Lustre parameters and statistics files provide an interface to internal data structures in the kernel that enables monitoring and tuning of many aspects of Lustre file system and application performance. These data structures include settings and metrics for components such as memory, networking, file systems, and kernel housekeeping routines, which are available throughout the hierarchical file layout.

Typically, metrics are accessed via `lctl get_param` files and settings are changed by via `lctl set_param`. While it is possible to access parameters in `/proc` and `/sys` directly, the location of these parameters may change between releases, so it is recommended to always use `lctl` to access the parameters from userspace scripts. Some data is server-only, some data is client-only, and some data is exported from the client to the server and is thus duplicated in both locations.

**Note**

In the examples in this chapter, `#` indicates a command is entered as root. Lustre servers are named according to the convention `*fsname*-*MDT|OSTnumber*`. The standard UNIX wildcard designation (*) is used.

Some examples are shown below:

- To obtain data from a Lustre client:

  ```
  # lctl list_param osc.*
  osc.testfs-OST0000-osc-ffff881071d5cc00
  osc.testfs-OST0001-osc-ffff881071d5cc00
  osc.testfs-OST0002-osc-ffff881071d5cc00
  osc.testfs-OST0003-osc-ffff881071d5cc00
  osc.testfs-OST0004-osc-ffff881071d5cc00
  osc.testfs-OST0005-osc-ffff881071d5cc00
  osc.testfs-OST0006-osc-ffff881071d5cc00
  osc.testfs-OST0007-osc-ffff881071d5cc00
  osc.testfs-OST0008-osc-ffff881071d5cc00
  ```

  In this example, information about OST connections available on a client is displayed (indicated by "osc").

- To see multiple levels of parameters, use multiple wildcards:

  ```
  # lctl list_param osc.*.*
  osc.testfs-OST0000-osc-ffff881071d5cc00.active
  osc.testfs-OST0000-osc-ffff881071d5cc00.blocksize
  osc.testfs-OST0000-osc-ffff881071d5cc00.checksum_type
  osc.testfs-OST0000-osc-ffff881071d5cc00.checksums
  osc.testfs-OST0000-osc-ffff881071d5cc00.connect_flags
  osc.testfs-OST0000-osc-ffff881071d5cc00.contention_seconds
  osc.testfs-OST0000-osc-ffff881071d5cc00.cur_dirty_bytes
  ...
  osc.testfs-OST0000-osc-ffff881071d5cc00.rpc_stats
  ```

- To view a specific file, use `lctl get_param`:

  ```
  # lctl get_param osc.lustre-OST0000*.rpc_stats
  ```

For more information about using `lctl`, see [*the section called “Setting Parameters with `lctl`”*](03.02-Lustre%20Operations.md#setting-parameters-with-lctl).

Data can also be viewed using the `cat` command with the full path to the file. The form of the `cat` command is similar to that of the `lctl get_param` command with some differences. Unfortunately, as the Linux kernel has changed over the years, the location of statistics and parameter files has also changed, which means that the Lustre parameter files may be located in either the `/proc` directory, in the `/sys` directory, and/or in the`/sys/kernel/debug` directory, depending on the kernel version and the Lustre version being used. The `lctl`command insulates scripts from these changes and is preferred over direct file access, unless as part of a high-performance monitoring system. In the `cat` command:

- Replace the dots in the path with slashes.

- Prepend the path with the appropriate directory component:

  ```
  /{proc,sys}/{fs,sys}/{lustre,lnet}
  ```

For example, an `lctl get_param` command may look like this:

```
# lctl get_param osc.*.uuid
osc.testfs-OST0000-osc-ffff881071d5cc00.uuid=594db456-0685-bd16-f59b-e72ee90e9819
osc.testfs-OST0001-osc-ffff881071d5cc00.uuid=594db456-0685-bd16-f59b-e72ee90e9819
...
```

The equivalent `cat` command may look like this:

```
# cat /proc/fs/lustre/osc/*/uuid
594db456-0685-bd16-f59b-e72ee90e9819
594db456-0685-bd16-f59b-e72ee90e9819
...
```

or like this:

```
# cat /sys/fs/lustre/osc/*/uuid
594db456-0685-bd16-f59b-e72ee90e9819
594db456-0685-bd16-f59b-e72ee90e9819
...
```

The `llstat` utility can be used to monitor some Lustre file system I/O activity over a specified time period. For more details, see [*the section called “ llstat”*](06.07-System%20Configuration%20Utilities.md#llstat).

Some data is imported from attached clients and is available in a directory called `exports` located in the corresponding per-service directory on a Lustre server. For example:

```
oss:/root# lctl list_param obdfilter.testfs-OST0000.exports.*
# hash ldlm_stats stats uuid
```

### Identifying Lustre File Systems and Servers

Several parameter files on the MGS list existing Lustre file systems and file system servers. The examples below are for a Lustre file system called `testfs` with one MDT and three OSTs.

- To view all known Lustre file systems, enter:

  ```
  mgs# lctl get_param mgs.*.filesystems
  testfs
  ```

- To view the names of the servers in a file system in which least one server is running, enter:

  ```
  lctl get_param mgs.*.live.<filesystem name>
  ```

  For example:

  ```
  mgs# lctl get_param mgs.*.live.testfs
  fsname: testfs
  flags: 0x20     gen: 45
  testfs-MDT0000
  testfs-OST0000
  testfs-OST0001
  testfs-OST0002 
  
  Secure RPC Config Rules: 
  
  imperative_recovery_state:
      state: startup
      nonir_clients: 0
      nidtbl_version: 6
      notify_duration_total: 0.001000
      notify_duation_max:  0.001000
      notify_count: 4
  ```

- To list all configured devices on the local node, enter:

  ```
  # lctl device_list
  0 UP mgs MGS MGS 11
  1 UP mgc MGC192.168.10.34@tcp 1f45bb57-d9be-2ddb-c0b0-5431a49226705
  2 UP mdt MDS MDS_uuid 3
  3 UP lov testfs-mdtlov testfs-mdtlov_UUID 4
  4 UP mds testfs-MDT0000 testfs-MDT0000_UUID 7
  5 UP osc testfs-OST0000-osc testfs-mdtlov_UUID 5
  6 UP osc testfs-OST0001-osc testfs-mdtlov_UUID 5
  7 UP lov testfs-clilov-ce63ca00 08ac6584-6c4a-3536-2c6d-b36cf9cbdaa04
  8 UP mdc testfs-MDT0000-mdc-ce63ca00 08ac6584-6c4a-3536-2c6d-b36cf9cbdaa05
  9 UP osc testfs-OST0000-osc-ce63ca00 08ac6584-6c4a-3536-2c6d-b36cf9cbdaa05
  10 UP osc testfs-OST0001-osc-ce63ca00 08ac6584-6c4a-3536-2c6d-b36cf9cbdaa05
  ```

  The information provided on each line includes:

  \- Device number

  \- Device status (UP, INactive, or STopping)

  \- Device name

  \- Device UUID

  \- Reference count (how many users this device has)

- To display the name of any server, view the device label:

  ```
  mds# e2label /dev/sda
  testfs-MDT0000 
  ```

## Tuning Multi-Block Allocation (mballoc)

Capabilities supported by `mballoc` include:

- Pre-allocation for single files to help to reduce fragmentation.
- Pre-allocation for a group of files to enable packing of small files into large, contiguous chunks.
- Stream allocation to help decrease the seek rate.

The following `mballoc` tunables are available:

| **Field**           | **Description**                                              |
| ------------------- | ------------------------------------------------------------ |
| `mb_max_to_scan`    | Maximum number of free chunks that `mballoc` finds before a final decision to avoid a livelock situation. |
| `mb_min_to_scan`    | Minimum number of free chunks that `mballoc` searches before picking the best chunk for allocation. This is useful for small requests to reduce fragmentation of big free chunks. |
| `mb_order2_req`     | For requests equal to 2^N, where N >= `mb_order2_req`, a fast search is done using a base 2 buddy allocation service. |
| `mb_small_req`      | `mb_small_req` - Defines (in MB) the upper bound of "small requests".`mb_large_req` - Defines (in MB) the lower bound of "large requests".Requests are handled differently based on size:< `mb_small_req` - Requests are packed together to form large, aggregated requests.> `mb_small_req` and < `mb_large_req` - Requests are primarily allocated linearly.> `mb_large_req` - Requests are allocated since hard disk seek time is less of a concern in this case.In general, small requests are combined to create larger requests, which are then placed close to one another to minimize the number of seeks required to access the data. |
| `mb_large_req`      |                                                              |
| `prealloc_table`    | A table of values used to preallocate space when a new request is received. By default, the table looks like this:`prealloc_table 4 8 16 32 64 128 256 512 1024 2048 `When a new request is received, space is preallocated at the next higher increment specified in the table. For example, for requests of less than 4 file system blocks, 4 blocks of space are preallocated; for requests between 4 and 8, 8 blocks are preallocated; and so forthAlthough customized values can be entered in the table, the performance of general usage file systems will not typically be improved by modifying the table (in fact, in ext4 systems, the table values are fixed). However, for some specialized workloads, tuning the `prealloc_table` values may result in smarter preallocation decisions. |
| `mb_group_prealloc` | The amount of space (in kilobytes) preallocated for groups of small requests. |

Buddy group cache information found in `/sys/fs/ldiskfs/*disk_device*/mb_groups` may be useful for assessing on-disk fragmentation. For example:

```
cat /proc/fs/ldiskfs/loop0/mb_groups 
#group: free free frags first pa [ 2^0 2^1 2^2 2^3 2^4 2^5 2^6 2^7 2^8 2^9 
     2^10 2^11 2^12 2^13] 
#0    : 2936 2936 1     42    0  [ 0   0   0   1   1   1   1   2   0   1 
     2    0    0    0   ]
```

In this example, the columns show:

- \#group number
- Available blocks in the group
- Blocks free on a disk
- Number of free fragments
- First free block in the group
- Number of preallocated chunks (not blocks)
- A series of available chunks of different sizes

## Monitoring Lustre File System I/O

A number of system utilities are provided to enable collection of data related to I/O activity in a Lustre file system. In general, the data collected describes:

- Data transfer rates and throughput of inputs and outputs external to the Lustre file system, such as network requests or disk I/O operations performed
- Data about the throughput or transfer rates of internal Lustre file system data, such as locks or allocations.

**Note**

It is highly recommended that you complete baseline testing for your Lustre file system to determine normal I/O activity for your hardware, network, and system workloads. Baseline data will allow you to easily determine when performance becomes degraded in your system. Two particularly useful baseline statistics are:

- `brw_stats` – Histogram data characterizing I/O requests to the OSTs. For more details, see [*the section called “Monitoring the OST Block I/O Stream”*](#monitoring-the-ost-block-io-stream).
- `rpc_stats` – Histogram data showing information about RPCs made by clients. For more details, see [*the section called “Monitoring the Client RPC Stream”*](#monitoring-the-client-rpc-stream).

### Monitoring the Client RPC Stream

The `rpc_stats` file contains histogram data showing information about remote procedure calls (RPCs) that have been made since this file was last cleared. The histogram data can be cleared by writing any value into the `rpc_stats` file.

**Example:**

```
# lctl get_param osc.testfs-OST0000-osc-ffff810058d2f800.rpc_stats
snapshot_time:            1372786692.389858 (secs.usecs)
read RPCs in flight:      0
write RPCs in flight:     1
dio read RPCs in flight:  0
dio write RPCs in flight: 0
pending write pages:      256
pending read pages:       0

                     read                   write
pages per rpc   rpcs   % cum % |       rpcs   % cum %
1:                 0   0   0   |          0   0   0
2:                 0   0   0   |          1   0   0
4:                 0   0   0   |          0   0   0
8:                 0   0   0   |          0   0   0
16:                0   0   0   |          0   0   0
32:                0   0   0   |          2   0   0
64:                0   0   0   |          2   0   0
128:               0   0   0   |          5   0   0
256:             850 100 100   |      18346  99 100

                     read                   write
rpcs in flight  rpcs   % cum % |       rpcs   % cum %
0:               691  81  81   |       1740   9   9
1:                48   5  86   |        938   5  14
2:                29   3  90   |       1059   5  20
3:                17   2  92   |       1052   5  26
4:                13   1  93   |        920   5  31
5:                12   1  95   |        425   2  33
6:                10   1  96   |        389   2  35
7:                30   3 100   |      11373  61  97
8:                 0   0 100   |        460   2 100

                     read                   write
offset          rpcs   % cum % |       rpcs   % cum %
0:               850 100 100   |      18347  99  99
1:                 0   0 100   |          0   0  99
2:                 0   0 100   |          0   0  99
4:                 0   0 100   |          0   0  99
8:                 0   0 100   |          0   0  99
16:                0   0 100   |          1   0  99
32:                0   0 100   |          1   0  99
64:                0   0 100   |          3   0  99
128:               0   0 100   |          4   0 100
```

The header information includes:

- `snapshot_time` - UNIX epoch instant the file was read.
- `read RPCs in flight` - Number of read RPCs issued by the OSC, but not complete at the time of the snapshot. This value should always be less than or equal to `max_rpcs_in_flight`.
- `write RPCs in flight` - Number of write RPCs issued by the OSC, but not complete at the time of the snapshot. This value should always be less than or equal to `max_rpcs_in_flight`.
- `dio read RPCs in flight` - Direct I/O (as opposed to block I/O) read RPCs issued but not completed at the time of the snapshot.
- `dio write RPCs in flight` - Direct I/O (as opposed to block I/O) write RPCs issued but not completed at the time of the snapshot.
- `pending write pages` - Number of pending write pages that have been queued for I/O in the OSC.
- `pending read pages` - Number of pending read pages that have been queued for I/O in the OSC.

The tabular data is described in the table below. Each row in the table shows the number of reads or writes (ios) occurring for the statistic, the relative percentage (%) of total reads or writes, and the cumulative percentage (cum %) to that point in the table for the statistic.

| **Field**      | **Description**                                              |
| -------------- | ------------------------------------------------------------ |
| pages per RPC  | Shows cumulative RPC reads and writes organized according to the number of pages in the RPC. A single page RPC increments the 0: row. |
| RPCs in flight | Shows the number of RPCs that are pending when an RPC is sent. When the first RPC is sent, the 0: row is incremented. If the first RPC is sent while another RPC is pending, the 1: row is incremented and so on |
| offset         | The page index of the first page read from or written to the object by the RPC. |

**Analysis:**

This table provides a way to visualize the concurrency of the RPC stream. Ideally, you will see a large clump around the `max_rpcs_in_flight value`, which shows that the network is being kept busy.

For information about optimizing the client I/O RPC stream, see [*the section called “Tuning the Client I/O RPC Stream”*](#tuning-the-client-io-rpc-stream).

### Monitoring Client Activity

The `stats` file maintains statistics accumulate during typical operation of a client across the VFS interface of the Lustre file system. Only non-zero parameters are displayed in the file.

Client statistics are enabled by default.

**Note**

Statistics for all mounted file systems can be discovered by entering:

```
lctl get_param llite.*.stats
```

**Example:**

```
client# lctl get_param llite.*.stats
snapshot_time          1308343279.169704 secs.usecs
dirty_pages_hits       14819716 samples [regs]
dirty_pages_misses     81473472 samples [regs]
read_bytes             36502963 samples [bytes] 1 26843582 55488794
write_bytes            22985001 samples [bytes] 0 125912 3379002
brw_read               2279 samples [pages] 1 1 2270
ioctl                  186749 samples [regs]
open                   3304805 samples [regs]
close                  3331323 samples [regs]
seek                   48222475 samples [regs]
fsync                  963 samples [regs]
truncate               9073 samples [regs]
setxattr               19059 samples [regs]
getxattr               61169 samples [regs]
```

The statistics can be cleared by echoing an empty string into the `stats` file or by using the command:

```
lctl set_param llite.*.stats=0
```

The statistics displayed are described in the table below.

| **Entry**           | **Description**                                              |
| ------------------- | ------------------------------------------------------------ |
| `snapshot_time`     | UNIX epoch instant the stats file was read.                  |
| `dirty_page_hits`   | The number of write operations that have been satisfied by the dirty page cache. See [*the section called “Tuning the Client I/O RPC Stream”*](#tuning-the-client-io-rpc-stream) for more information about dirty cache behavior in a Lustre file system. |
| `dirty_page_misses` | The number of write operations that were not satisfied by the dirty page cache. |
| `read_bytes`        | The number of read operations that have occurred. Three additional parameters are displayed:minThe minimum number of bytes read in a single request since the counter was reset.maxThe maximum number of bytes read in a single request since the counter was reset.sumThe accumulated sum of bytes of all read requests since the counter was reset. |
| `write_bytes`       | The number of write operations that have occurred. Three additional parameters are displayed:minThe minimum number of bytes written in a single request since the counter was reset.maxThe maximum number of bytes written in a single request since the counter was reset.sumThe accumulated sum of bytes of all write requests since the counter was reset. |
| `brw_read`          | The number of pages that have been read. Three additional parameters are displayed:minThe minimum number of bytes read in a single block read/write (`brw`) read request since the counter was reset.maxThe maximum number of bytes read in a single `brw` read requests since the counter was reset.sumThe accumulated sum of bytes of all `brw` read requests since the counter was reset. |
| `ioctl`             | The number of combined file and directory `ioctl` operations. |
| `open`              | The number of open operations that have succeeded.           |
| `close`             | The number of close operations that have succeeded.          |
| `seek`              | The number of times `seek` has been called.                  |
| `fsync`             | The number of times `fsync` has been called.                 |
| `truncate`          | The total number of calls to both locked and lockless `truncate`. |
| `setxattr`          | The number of times extended attributes have been set.       |
| `getxattr`          | The number of times value(s) of extended attributes have been fetched. |

**Analysis:**

Information is provided about the amount and type of I/O activity is taking place on the client.

### Monitoring Client Read-Write Offset Statistics

When the `offset_stats` parameter is set, statistics are maintained for occurrences of a series of read or write calls from a process that did not access the next sequential location. The `OFFSET` field is reset to 0 (zero) whenever a different file is read or written.

### Note

By default, statistics are not collected in the `offset_stats`, `extents_stats`, and `extents_stats_per_process` files to reduce monitoring overhead when this information is not needed. The collection of statistics in all three of these files is activated by writing anything, except for 0 (zero) and "disable", into any one of the files.

**Example:**

```
# lctl get_param llite.testfs-f57dee0.offset_stats
snapshot_time: 1155748884.591028 (secs.usecs)
             RANGE   RANGE    SMALLEST   LARGEST
R/W   PID    START   END      EXTENT     EXTENT    OFFSET
R     8385   0       128      128        128       0
R     8385   0       224      224        224       -128
W     8385   0       250      50         100       0
W     8385   100     1110     10         500       -150
W     8384   0       5233     5233       5233      0
R     8385   500     600      100        100       -610
```

In this example, `snapshot_time` is the UNIX epoch instant the file was read. The tabular data is described in the table below.

The `offset_stats` file can be cleared by entering:

```
lctl set_param llite.*.offset_stats=0
```

| **Field**             | **Description**                                              |
| --------------------- | ------------------------------------------------------------ |
| R/W                   | Indicates if the non-sequential call was a read or write     |
| PID                   | Process ID of the process that made the read/write call.     |
| RANGE START/RANGE END | Range in which the read/write calls were sequential.         |
| SMALLEST EXTENT       | Smallest single read/write in the corresponding range (in bytes). |
| LARGEST EXTENT        | Largest single read/write in the corresponding range (in bytes). |
| OFFSET                | Difference between the previous range end and the current range start. |

**Analysis:**

This data provides an indication of how contiguous or fragmented the data is. For example, the fourth entry in the example above shows the writes for this RPC were sequential in the range 100 to 1110 with the minimum write 10 bytes and the maximum write 500 bytes. The range started with an offset of -150 from the `RANGE END` of the previous entry in the example.

### Monitoring Client Read-Write Extent Statistics

For in-depth troubleshooting, client read-write extent statistics can be accessed to obtain more detail about read/write I/O extents for the file system or for a particular process.

**Note**

By default, statistics are not collected in the `offset_stats`, `extents_stats`, and `extents_stats_per_process` files to reduce monitoring overhead when this information is not needed. The collection of statistics in all three of these files is activated by writing anything, except for 0 (zero) and "disable", into any one of the files.

#### Client-Based I/O Extent Size Survey

The `extents_stats` histogram in the `llite` directory shows the statistics for the sizes of the read/write I/O extents. This file does not maintain the per process statistics.

**Example:**

```
# lctl get_param llite.testfs-*.extents_stats
snapshot_time:                     1213828728.348516 (secs.usecs)
                       read           |            write
extents          calls  %      cum%   |     calls  %     cum%

0K - 4K :        0      0      0      |     2      2     2
4K - 8K :        0      0      0      |     0      0     2
8K - 16K :       0      0      0      |     0      0     2
16K - 32K :      0      0      0      |     20     23    26
32K - 64K :      0      0      0      |     0      0     26
64K - 128K :     0      0      0      |     51     60    86
128K - 256K :    0      0      0      |     0      0     86
256K - 512K :    0      0      0      |     0      0     86
512K - 1024K :   0      0      0      |     0      0     86
1M - 2M :        0      0      0      |     11     13    100
```

In this example, `snapshot_time` is the UNIX epoch instant the file was read. The table shows cumulative extents organized according to size with statistics provided separately for reads and writes. Each row in the table shows the number of RPCs for reads and writes respectively (`calls`), the relative percentage of total calls (`%`), and the cumulative percentage to that point in the table of calls (`cum %`).

The file can be cleared by issuing the following command:

```
# lctl set_param llite.testfs-*.extents_stats=1
```

#### Per-Process Client I/O Statistics

The `extents_stats_per_process` file maintains the I/O extent size statistics on a per-process basis.

**Example:**

```
# lctl get_param llite.testfs-*.extents_stats_per_process
snapshot_time:                     1213828762.204440 (secs.usecs)
                          read            |             write
extents            calls   %      cum%    |      calls   %       cum%
 
PID: 11488
   0K - 4K :       0       0       0      |      0       0       0
   4K - 8K :       0       0       0      |      0       0       0
   8K - 16K :      0       0       0      |      0       0       0
   16K - 32K :     0       0       0      |      0       0       0
   32K - 64K :     0       0       0      |      0       0       0
   64K - 128K :    0       0       0      |      0       0       0
   128K - 256K :   0       0       0      |      0       0       0
   256K - 512K :   0       0       0      |      0       0       0
   512K - 1024K :  0       0       0      |      0       0       0
   1M - 2M :       0       0       0      |      10      100     100
 
PID: 11491
   0K - 4K :       0       0       0      |      0       0       0
   4K - 8K :       0       0       0      |      0       0       0
   8K - 16K :      0       0       0      |      0       0       0
   16K - 32K :     0       0       0      |      20      100     100
   
PID: 11424
   0K - 4K :       0       0       0      |      0       0       0
   4K - 8K :       0       0       0      |      0       0       0
   8K - 16K :      0       0       0      |      0       0       0
   16K - 32K :     0       0       0      |      0       0       0
   32K - 64K :     0       0       0      |      0       0       0
   64K - 128K :    0       0       0      |      16      100     100
 
PID: 11426
   0K - 4K :       0       0       0      |      1       100     100
 
PID: 11429
   0K - 4K :       0       0       0      |      1       100     100
 
```

This table shows cumulative extents organized according to size for each process ID (PID) with statistics provided separately for reads and writes. Each row in the table shows the number of RPCs for reads and writes respectively (`calls`), the relative percentage of total calls (`%`), and the cumulative percentage to that point in the table of calls (`cum %`).

### Monitoring the OST Block I/O Stream

The `brw_stats` parameter file below the `osd-ldiskfs` or `osd-zfs` directory contains histogram data showing statistics for number of I/O requests sent to the disk, their size, and whether they are contiguous on the disk or not.

**Example:**

Enter on the OSS or MDS:

```
oss# lctl get_param osd-*.*.brw_stats
snapshot_time: 1372775039.769045 (secs.usecs)
 read | write
pages per bulk r/w rpcs % cum % | rpcs % cum %
1: 108 100 100 | 39 0 0
2: 0 0 100 | 6 0 0
4: 0 0 100 | 1 0 0
8: 0 0 100 | 0 0 0
16: 0 0 100 | 4 0 0
32: 0 0 100 | 17 0 0
64: 0 0 100 | 12 0 0
128: 0 0 100 | 24 0 0
256: 0 0 100 | 23142 99 100
 read | write
discontiguous pages rpcs % cum % | rpcs % cum %
0: 108 100 100 | 23245 100 100
 read | write
discontiguous blocks rpcs % cum % | rpcs % cum %
0: 108 100 100 | 23243 99 99
1: 0 0 100 | 2 0 100
 read | write
disk fragmented I/Os ios % cum % | ios % cum %
0: 94 87 87 | 0 0 0
1: 14 12 100 | 23243 99 99
2: 0 0 100 | 2 0 100
 read | write
disk I/Os in flight ios % cum % | ios % cum %
1: 14 100 100 | 20896 89 89
2: 0 0 100 | 1071 4 94
3: 0 0 100 | 573 2 96
4: 0 0 100 | 300 1 98
5: 0 0 100 | 166 0 98
6: 0 0 100 | 108 0 99
7: 0 0 100 | 81 0 99
8: 0 0 100 | 47 0 99
9: 0 0 100 | 5 0 100
 read | write
I/O time (1/1000s) ios % cum % | ios % cum %
1: 94 87 87 | 0 0 0
2: 0 0 87 | 7 0 0
4: 14 12 100 | 27 0 0
8: 0 0 100 | 14 0 0
16: 0 0 100 | 31 0 0
32: 0 0 100 | 38 0 0
64: 0 0 100 | 18979 81 82
128: 0 0 100 | 943 4 86
256: 0 0 100 | 1233 5 91
512: 0 0 100 | 1825 7 99
1K: 0 0 100 | 99 0 99
2K: 0 0 100 | 0 0 99
4K: 0 0 100 | 0 0 99
8K: 0 0 100 | 49 0 100
 read | write
disk I/O size ios % cum % | ios % cum %
4K: 14 100 100 | 41 0 0
8K: 0 0 100 | 6 0 0
16K: 0 0 100 | 1 0 0
32K: 0 0 100 | 0 0 0
64K: 0 0 100 | 4 0 0
128K: 0 0 100 | 17 0 0
256K: 0 0 100 | 12 0 0
512K: 0 0 100 | 24 0 0
1M: 0 0 100 | 23142 99 100
```

The tabular data is described in the table below. Each row in the table shows the number of reads and writes occurring for the statistic (`ios`), the relative percentage of total reads or writes (`%`), and the cumulative percentage to that point in the table for the statistic (`cum %`).

| **Field**              | **Description**                                              |
| ---------------------- | ------------------------------------------------------------ |
| `pages per bulk r/w`   | Number of pages per RPC request, which should match aggregate client `rpc_stats` (see [*the section called “Monitoring the Client RPC Stream”*](#monitoring-the-client-rpc-stream)). |
| `discontiguous pages`  | Number of discontinuities in the logical file offset of each page in a single RPC. |
| `discontiguous blocks` | Number of discontinuities in the physical block allocation in the file system for a single RPC. |
| `disk fragmented I/Os` | Number of I/Os that were not written entirely sequentially.  |
| `disk I/Os in flight`  | Number of disk I/Os currently pending.                       |
| `I/O time (1/1000s)`   | Amount of time for each I/O operation to complete.           |
| `disk I/O size`        | Size of each I/O operation.                                  |

**Analysis:**

This data provides an indication of extent size and distribution in the file system.

## Tuning Lustre File System I/O

Each OSC has its own tree of tunables. For example:

```
$ lctl lctl list_param osc.*.*
osc.myth-OST0000-osc-ffff8804296c2800.active
osc.myth-OST0000-osc-ffff8804296c2800.blocksize
osc.myth-OST0000-osc-ffff8804296c2800.checksum_dump
osc.myth-OST0000-osc-ffff8804296c2800.checksum_type
osc.myth-OST0000-osc-ffff8804296c2800.checksums
osc.myth-OST0000-osc-ffff8804296c2800.connect_flags
:
:
osc.myth-OST0000-osc-ffff8804296c2800.state
osc.myth-OST0000-osc-ffff8804296c2800.stats
osc.myth-OST0000-osc-ffff8804296c2800.timeouts
osc.myth-OST0000-osc-ffff8804296c2800.unstable_stats
osc.myth-OST0000-osc-ffff8804296c2800.uuid
osc.myth-OST0001-osc-ffff8804296c2800.active
osc.myth-OST0001-osc-ffff8804296c2800.blocksize
osc.myth-OST0001-osc-ffff8804296c2800.checksum_dump
osc.myth-OST0001-osc-ffff8804296c2800.checksum_type
:
:
```

The following sections describe some of the parameters that can be tuned in a Lustre file system.

### Tuning the Client I/O RPC Stream

Ideally, an optimal amount of data is packed into each I/O RPC and a consistent number of issued RPCs are in progress at any time. To help optimize the client I/O RPC stream, several tuning variables are provided to adjust behavior according to network conditions and cluster size. For information about monitoring the client I/O RPC stream, see [*the section called “Monitoring the Client RPC Stream”*](#monitoring-the-client-rpc-stream).

RPC stream tunables include:

- `osc.*osc_instance*.checksums` - Controls whether the client will calculate data integrity checksums for the bulk data transferred to the OST. Data integrity checksums are enabled by default. The algorithm used can be set using the `checksum_type` parameter.

- `osc.*osc_instance*.checksum_type` - Controls the data integrity checksum algorithm used by the client. The available algorithms are determined by the set of algorihtms. The checksum algorithm used by default is determined by first selecting the fastest algorithms available on the OST, and then selecting the fastest of those algorithms on the client, which depends on available optimizations in the CPU hardware and kernel. The default algorithm can be overridden by writing the algorithm name into the `checksum_type` parameter. Available checksum types can be seen on the client by reading the `checksum_type` parameter. Currently supported checksum types are: `adler`, `crc32`, `crc32c`

  Introduced in Lustre 2.12In Lustre release 2.12 additional checksum types were added to allow end-to-end checksum integration with T10-PI capable hardware. The client will compute the appropriate checksum type, based on the checksum type used by the storage, for the RPC checksum, which will be verified by the server and passed on to the storage. The T10-PI checksum types are: `t10ip512`, `t10ip4K`, `t10crc512`, `t10crc4K`

- `osc.*osc_instance*.max_dirty_mb` - Controls how many MiB of dirty data can be written into the client pagecache for writes by *each* OSC. When this limit is reached, additional writes block until previously-cached data is written to the server. This may be changed by the `lctl set_param` command. Only values larger than 0 and smaller than the lesser of 2048 MiB or 1/4 of client RAM are valid. Performance can suffers if the client cannot aggregate enough data per OSC to form a full RPC (as set by the `max_pages_per_rpc`) parameter, unless the application is doing very large writes itself.

  To maximize performance, the value for `max_dirty_mb` is recommended to be at least 4 * `max_pages_per_rpc` * `max_rpcs_in_flight`.

- `osc.*osc_instance*.cur_dirty_bytes` - A read-only value that returns the current number of bytes written and cached by this OSC.

- `osc.*osc_instance*.max_pages_per_rpc` - The maximum number of pages that will be sent in a single RPC request to the OST. The minimum value is one page and the maximum value is 16 MiB (4096 on systems with `PAGE_SIZE` of 4 KiB), with the default value of 4 MiB in one RPC. The upper limit may also be constrained by `ofd.*.brw_size` setting on the OSS, and applies to all clients connected to that OST. It is also possible to specify a units suffix (e.g. `max_pages_per_rpc=4M`), so the RPC size can be set independently of the client `PAGE_SIZE`.

- `osc.*osc_instance*.max_rpcs_in_flight` - The maximum number of concurrent RPCs in flight from an OSC to its OST. If the OSC tries to initiate an RPC but finds that it already has the same number of RPCs outstanding, it will wait to issue further RPCs until some complete. The minimum setting is 1 and maximum setting is 256. The default value is 8 RPCs.

  To improve small file I/O performance, increase the `max_rpcs_in_flight` value.

- `llite.*fsname_instance*.max_cache_mb` - Maximum amount of read+write data cached by the client. The default value is 1/2 of the client RAM.

**Note**

The value for `*osc_instance*` and `*fsname_instance*` are unique to each mount point to allow associating osc, mdc, lov, lmv, and llite parameters with the same mount point. However, it is common for scripts to use a wildcard `*` or a filesystem-specific wildcard `*fsname-\**` to specify the parameter settings uniformly on all clients. For example:

```
client$ lctl get_param osc.testfs-OST0000*.rpc_stats
osc.testfs-OST0000-osc-ffff88107412f400.rpc_stats=
snapshot_time:         1375743284.337839 (secs.usecs)
read RPCs in flight:  0
write RPCs in flight: 0
```

### Tuning File Readahead and Directory Statahead

File readahead and directory statahead enable reading of data into memory before a process requests the data. File readahead prefetches file content data into memory for `read()` related calls, while directory statahead fetches file metadata into memory for `readdir()` and `stat()` related calls. When readahead and statahead work well, a process that accesses data finds that the information it needs is available immediately in memory on the client when requested without the delay of network I/O.

#### Tuning File Readahead

File readahead is triggered when two or more sequential reads by an application fail to be satisfied by data in the Linux buffer cache. The size of the initial readahead is determined by the RPC size and the file stripe size, but will typically be at least 1 MiB. Additional readaheads grow linearly and increment until the per-file or per-system readahead cache limit on the client is reached.

Readahead tunables include:

- `llite.*fsname_instance*.max_read_ahead_mb` - Controls the maximum amount of data readahead on a file. Files are read ahead in RPC-sized chunks (4 MiB, or the size of the `read()` call, if larger) after the second sequential read on a file descriptor. Random reads are done at the size of the `read()` call only (no readahead). Reads to non-contiguous regions of the file reset the readahead algorithm, and readahead is not triggered until sequential reads take place again.

  This is the global limit for all files and cannot be larger than 1/2 of the client RAM. To disable readahead, set`max_read_ahead_mb=0`.

- `llite.*fsname_instance*.max_read_ahead_per_file_mb` - Controls the maximum number of megabytes (MiB) of data that should be prefetched by the client when sequential reads are detected on a file. This is the per-file readahead limit and cannot be larger than `max_read_ahead_mb`.

- `llite.*fsname_instance*.max_read_ahead_whole_mb` - Controls the maximum size of a file in MiB that is read in its entirety upon access, regardless of the size of the `read()` call. This avoids multiple small read RPCs on relatively small files, when it is not possible to efficiently detect a sequential read pattern before the whole file has been read.

  The default value is the greater of 2 MiB or the size of one RPC, as given by `max_pages_per_rpc`.

#### Tuning Directory Statahead and AGL

Many system commands, such as `ls –l`, `du`, and `find`, traverse a directory sequentially. To make these commands run efficiently, the directory statahead can be enabled to improve the performance of directory traversal.

The statahead tunables are:

- `statahead_max` - Controls the maximum number of file attributes that will be prefetched by the statahead thread. By default, statahead is enabled and `statahead_max` is 32 files.

  To disable statahead, set `statahead_max` to zero via the following command on the client:

  ```
  lctl set_param llite.*.statahead_max=0
  ```

  To change the maximum statahead window size on a client:

  ```
  lctl set_param llite.*.statahead_max=n
  ```

  The maximum `statahead_max` is 8192 files.

  The directory statahead thread will also prefetch the file size/block attributes from the OSTs, so that all file attributes are available on the client when requested by an application. This is controlled by the asynchronous glimpse lock (AGL) setting. The AGL behaviour can be disabled by setting:

  ```
  lctl set_param llite.*.statahead_agl=0
  ```

- `statahead_stats` - A read-only interface that provides current statahead and AGL statistics, such as how many times statahead/AGL has been triggered since the last mount, how many statahead/AGL failures have occurred due to an incorrect prediction or other causes.

  ### Note

  AGL behaviour is affected by statahead since the inodes processed by AGL are built by the statahead thread. If statahead is disabled, then AGL is also disabled.

### Tuning Server Read Cache

The server read cache feature provides read-only caching of file data on an OSS or MDS (for Data-onMDT). This functionality uses the Linux page cache to store the data and uses as much physical memory as is allocated.

The server read cache can improves Lustre file system performance in these situations:

- Many clients are accessing the same data set (as in HPC applications or when diskless clients boot from the Lustre file system).
- One client is writing data while another client is reading it (i.e., clients are exchanging data via the filesystem).
- A client has very limited caching of its own.

The server read cache offers these benefits:

- Allows servers to cache read data more frequently.
- Improves repeated reads to match network speeds instead of storage speeds.
- Provides the building blocks for server write cache (small-write aggregation).

#### Using OSS Read Cache

The server read cache is implemented on the OSS and MDS, and does not require any special support on the client side. Since the server read cache uses the memory available in the Linux page cache, the appropriate amount of memory for the cache should be determined based on I/O patterns. If the data is mostly reads, then more cache is beneficial on the server than would be needed for mostly writes.

The server read cache is managed using the following tunables. Many tunables are available for both `osd-ldiskfs` and `osd-zfs`, but in some cases the implementation of `osd-zfs` prevents their use.

- `read_cache_enable` \- High-level control of whether data read from storage during a read request is kept in memory and available for later read requests for the same data, without having to re-read it from storage. By default, read cache is enabled (`read_cache_enable=1`) for HDD OSDs and automatically disabled for flash OSDs (`nonrotational=1`). The read cache cannot be disabled for `osd-zfs`, and as a result this parameter is unavailable for that backend.

  When the server receives a read request from a client, it reads data from storage into its memory and sends the data to the client. If read cache is enabled for the target, and the RPC and object size also meet the other criterion below, this data may stay in memory after the client request has completed. If later read requests for the same data are received, if the data is still in cache the server skips reading it from storage. The cache is managed by the Linux kernel globally across all targets on that server so that the infrequently used cache pages are dropped from memory when the free memory is running low.

  If read cache is disabled (`read_cache_enable=0`), or the read or object is large enough that it will not benefit from caching, the server discards the data after the read request from the client is completed. For subsequent read requests the server again reads the data from storage.

  To disable read cache on all targets of a server, run:

  ```
   oss1# lctl set_param osd-*.*.read_cache_enable=0
  ```

  To re-enable read cache on one target, run:

  ```
   oss1# lctl set_param osd-*.{target_name}.read_cache_enable=1
  ```

  To check if read cache is enabled on targets on a server, run:

  ```
   oss1# lctl get_param osd-*.*.read_cache_enable
  ```

- `writethrough_cache_enable` - High-level control of whether data sent to the server as a write request is kept in the read cache and available for later reads, or if it is discarded when the write completes. By default, writethrough cache is enabled (`writethrough_cache_enable=1`) for HDD OSDs and automatically disabled for flash OSDs (`nonrotational=1`). The write cache cannot be disabled for `osd-zfs`, and as a result this parameter is unavailable for that backend.

  When the server receives write requests from a client, it fetches data from the client into its memory and writes the data to storage. If the writethrough cache is enabled for the target, and the RPC and object size meet the other criterion below, this data may stay in memory after the write request has completed. If later read or partial-block write requests for this same data are received, if the data is still in cache the server skips reading it from storage.

  If the writethrough cache is disabled (`writethrough_cache_enabled=0`), or the write or object is large enough that it will not benefit from caching, the server discards the data after the write request from the client is completed. For subsequent read requests, or partial-page write requests, the server must re-read the data from storage.

  Enabling writethrough cache is advisable if clients are doing small or unaligned writes that would cause partial-page updates, or if the files written by one node are immediately being read by other nodes. Some examples where enabling writethrough cache might be useful include producer-consumer I/O models or shared-file writes that are not aligned on 4096-byte boundaries.

  Disabling the writethrough cache is advisable when files are mostly written to the file system but are not re-read within a short time period, or files are only written and re-read by the same node, regardless of whether the I/O is aligned or not.

  To disable writethrough cache on all targets on a server, run:

  ```
   oss1# lctl set_param osd-*.*.writethrough_cache_enable=0
  ```

  To re-enable the writethrough cache on one OST, run:

  ```
   oss1# lctl set_param osd-*.{OST_name}.writethrough_cache_enable=1
  ```

  To check if the writethrough cache is enabled, run:

  ```
   oss1# lctl get_param osd-*.*.writethrough_cache_enable
  ```

- `readcache_max_filesize` - Controls the maximum size of an object that both the read cache and writethrough cache will try to keep in memory. Objects larger than `readcache_max_filesize` will not be kept in cache for either reads or writes regardless of the read_cache_enable or `writethrough_cache_enable` settings.

  Setting this tunable can be useful for workloads where relatively small objects are repeatedly accessed by many clients, such as job startup objects, executables, log objects, etc., but large objects are read or written only once. By not putting the larger objects into the cache, it is much more likely that more of the smaller objects will remain in cache for a longer time.

  When setting readcache_max_filesize, the input value can be specified in bytes, or can have a suffix to indicate other binary units such as K (kibibytes), M (mebibytes), G (gibibytes), T (tebibytes), or P (pebibytes).

  To limit the maximum cached object size to 64 MiB on all OSTs of a server, run:

  ```
   oss1# lctl set_param osd-*.*.readcache_max_filesize=64M
  ```

  To disable the maximum cached object size on all targets, run:

  ```
   oss1# lctl set_param osd-*.*.readcache_max_filesize=-1
  ```

  To check the current maximum cached object size on all targets of a server, run:

  ```
   oss1# lctl get_param osd-*.*.readcache_max_filesize
  ```

* `readcache_max_io_mb` - Controls the maximum size of a single read IO that will be cached in memory. Reads larger than `readcache_max_io_mb` will be read directly from storage and bypass the page cache completely. This avoids significant CPU overhead at high IO rates. The read cache cannot be disabled for osd-zfs, and as a result this parameter is unavailable for that backend.

  When setting `readcache_max_io_mb`, the input value can be specified in mebibytes, or can have a suffix to indicate other binary units such as K (kibibytes), M (mebibytes), G (gibibytes), T (tebibytes), or P (pebibytes).

* `writethrough_max_io_mb` - Controls the maximum size of a single writes IO that will be cached in memory. Writes larger than `writethrough_max_io_mb` will be written directly to storage and bypass the page cache entirely. This avoids significant CPU overhead at high IO rates. The write cache cannot be disabled for `osd-zfs`, and as a result this parameter is unavailable for that backend.

  When setting `writethrough_max_io_mb`, the input value can be specified in mebibytes, or can have a suffix to indicate other binary units such as K (kibibytes), M (mebibytes), G (gibibytes), T (tebibytes), or P (pebibytes).

### Enabling OSS Asynchronous Journal Commit

The OSS asynchronous journal commit feature asynchronously writes data to disk without forcing a journal flush. This reduces the number of seeks and significantly improves performance on some hardware.

**Note**

Asynchronous journal commit cannot work with direct I/O-originated writes (`O_DIRECT` flag set). In this case, a journal flush is forced.

When the asynchronous journal commit feature is enabled, client nodes keep data in the page cache (a page reference). Lustre clients monitor the last committed transaction number (`transno`) in messages sent from the OSS to the clients. When a client sees that the last committed `transno` reported by the OSS is at least equal to the bulk write `transno`, it releases the reference on the corresponding pages. To avoid page references being held for too long on clients after a bulk write, a 7 second ping request is scheduled (the default OSS file system commit time interval is 5 seconds) after the bulk write reply is received, so the OSS has an opportunity to report the last committed `transno`.

If the OSS crashes before the journal commit occurs, then intermediate data is lost. However, OSS recovery functionality incorporated into the asynchronous journal commit feature causes clients to replay their write requests and compensate for the missing disk updates by restoring the state of the file system.

By default, `sync_journal` is enabled (`sync_journal=1`), so that journal entries are committed synchronously. To enable asynchronous journal commit, set the `sync_journal` parameter to `0` by entering:

```
$ lctl set_param obdfilter.*.sync_journal=0 
obdfilter.lol-OST0001.sync_journal=0
```

An associated `sync-on-lock-cancel` feature (enabled by default) addresses a data consistency issue that can result if an OSS crashes after multiple clients have written data into intersecting regions of an object, and then one of the clients also crashes. A condition is created in which the POSIX requirement for continuous writes is violated along with a potential for corrupted data. With `sync-on-lock-cancel` enabled, if a cancelled lock has any volatile writes attached to it, the OSS synchronously writes the journal to disk on lock cancellation. Disabling the `sync-on-lock-cancel` feature may enhance performance for concurrent write workloads, but it is recommended that you not disable this feature.

The `sync_on_lock_cancel` parameter can be set to the following values:

- `always` - Always force a journal flush on lock cancellation (default when `async_journal` is enabled).
- `blocking` - Force a journal flush only when the local cancellation is due to a blocking callback.
- `never` - Do not force any journal flush (default when `async_journal` is disabled).

For example, to set `sync_on_lock_cancel` to not to force a journal flush, use a command similar to:

```
$ lctl get_param obdfilter.*.sync_on_lock_cancel
obdfilter.lol-OST0001.sync_on_lock_cancel=never
```

Introduced in Lustre 2.8

### Tuning the Client Metadata RPC Stream

The client metadata RPC stream represents the metadata RPCs issued in parallel by a client to a MDT target. The metadata RPCs can be split in two categories: the requests that do not modify the file system (like getattr operation), and the requests that do modify the file system (like create, unlink, setattr operations). To help optimize the client metadata RPC stream, several tuning variables are provided to adjust behavior according to network conditions and cluster size.

Note that increasing the number of metadata RPCs issued in parallel might improve the performance metadata intensive parallel applications, but as a consequence it will consume more memory on the client and on the MDS.

#### Configuring the Client Metadata RPC Stream

The MDC `max_rpcs_in_flight` parameter defines the maximum number of metadata RPCs, both modifying and non-modifying RPCs, that can be sent in parallel by a client to a MDT target. This includes every file system metadata operations, such as file or directory stat, creation, unlink. The default setting is 8, minimum setting is 1 and maximum setting is 256.

To set the `max_rpcs_in_flight` parameter, run the following command on the Lustre client:

```
client$ lctl set_param mdc.*.max_rpcs_in_flight=16
```

The MDC `max_mod_rpcs_in_flight` parameter defines the maximum number of file system modifying RPCs that can be sent in parallel by a client to a MDT target. For example, the Lustre client sends modify RPCs when it performs file or directory creation, unlink, access permission modification or ownership modification. The default setting is 7, minimum setting is 1 and maximum setting is 256.

To set the `max_mod_rpcs_in_flight` parameter, run the following command on the Lustre client:

```
client$ lctl set_param mdc.*.max_mod_rpcs_in_flight=12
```

The `max_mod_rpcs_in_flight` value must be strictly less than the `max_rpcs_in_flight` value. It must also be less or equal to the MDT `max_mod_rpcs_per_client` value. If one of theses conditions is not enforced, the setting fails and an explicit message is written in the Lustre log.

The MDT `max_mod_rpcs_per_client` parameter is a tunable of the kernel module `mdt` that defines the maximum number of file system modifying RPCs in flight allowed per client. The parameter can be updated at runtime, but the change is effective to new client connections only. The default setting is 8.

To set the `max_mod_rpcs_per_client` parameter, run the following command on the MDS:

```
mds$ echo 12 > /sys/module/mdt/parameters/max_mod_rpcs_per_client
```

#### Monitoring the Client Metadata RPC Stream

The `rpc_stats` file contains histogram data showing information about modify metadata RPCs. It can be helpful to identify the level of parallelism achieved by an application doing modify metadata operations.

**Example:**

```
client$ lctl get_param mdc.*.rpc_stats
snapshot_time:         1441876896.567070 (secs.usecs)
modify_RPCs_in_flight:  0

                        modify
rpcs in flight        rpcs   % cum %
0:                       0   0   0
1:                      56   0   0
2:                      40   0   0
3:                      70   0   0
4                       41   0   0
5:                      51   0   1
6:                      88   0   1
7:                     366   1   2
8:                    1321   5   8
9:                    3624  15  23
10:                   6482  27  50
11:                   7321  30  81
12:                   4540  18 100
```

The file information includes:

- `snapshot_time` - UNIX epoch instant the file was read.
- `modify_RPCs_in_flight` - Number of modify RPCs issued by the MDC, but not completed at the time of the snapshot. This value should always be less than or equal to `max_mod_rpcs_in_flight`.
- `rpcs in flight` - Number of modify RPCs that are pending when a RPC is sent, the relative percentage (`%`) of total modify RPCs, and the cumulative percentage (`cum %`) to that point.

If a large proportion of modify metadata RPCs are issued with a number of pending metadata RPCs close to the`max_mod_rpcs_in_flight` value, it means the `max_mod_rpcs_in_flight` value could be increased to improve the modify metadata performance.

## Configuring Timeouts in a Lustre File System

In a Lustre file system, RPC timeouts are set using an adaptive timeouts mechanism, which is enabled by default. Servers track RPC completion times and then report back to clients estimates for completion times for future RPCs. Clients use these estimates to set RPC timeout values. If the processing of server requests slows down for any reason, the server estimates for RPC completion increase, and clients then revise RPC timeout values to allow more time for RPC completion.

If the RPCs queued on the server approach the RPC timeout specified by the client, to avoid RPC timeouts and disconnect/reconnect cycles, the server sends an "early reply" to the client, telling the client to allow more time. Conversely, as server processing speeds up, RPC timeout values decrease, resulting in faster detection if the server becomes non-responsive and quicker connection to the failover partner of the server.

### Configuring Adaptive Timeouts

The adaptive timeout parameters in the table below can be set persistently system-wide using `lctl conf_param` on the MGS. For example, the following command sets the `at_max` value for all servers and clients associated with the file system `testfs`:

```
lctl conf_param testfs.sys.at_max=1500
```

**Note**

Clients that access multiple Lustre file systems must use the same parameter values for all file systems.

| **Parameter**      | **Description**                                              |
| ------------------ | ------------------------------------------------------------ |
| `at_min`           | Minimum adaptive timeout (in seconds). The default value is 0. The `at_min` parameter is the minimum processing time that a server will report. Ideally, `at_min` should be set to its default value. Clients base their timeouts on this value, but they do not use this value directly.If, for unknown reasons (usually due to temporary network outages), the adaptive timeout value is too short and clients time out their RPCs, you can increase the `at_min` value to compensate for this. |
| `at_max`           | Maximum adaptive timeout (in seconds). The `at_max` parameter is an upper-limit on the service time estimate. If `at_max` is reached, an RPC request times out.Setting `at_max` to 0 causes adaptive timeouts to be disabled and a fixed timeout method to be used instead (see [*the section called “Setting Static Timeouts”*](#setting-static-timeouts)                                                                                                   **Note**                                                                                                                                                                                                                                             If slow hardware causes the service estimate to increase beyond the default value of `at_max`, increase `at_max` to the maximum time you are willing to wait for an RPC completion. |
| `at_history`       | Time period (in seconds) within which adaptive timeouts remember the slowest event that occurred. The default is 600. |
| `at_early_margin`  | Amount of time before the Lustre server sends an early reply (in seconds). Default is 5. |
| `at_extra`         | Incremental amount of time that a server requests with each early reply (in seconds). The server does not know how much time the RPC will take, so it asks for a fixed value. The default is 30, which provides a balance between sending too many early replies for the same RPC and overestimating the actual completion time.When a server finds a queued request about to time out and needs to send an early reply out, the server adds the `at_extra` value. If the time expires, the Lustre server drops the request, and the client enters recovery status and reconnects to restore the connection to normal status.If you see multiple early replies for the same RPC asking for 30-second increases, change the `at_extra` value to a larger number to cut down on early replies sent and, therefore, network load. |
| `ldlm_enqueue_min` | Minimum lock enqueue time (in seconds). The default is 100. The time it takes to enqueue a lock, `ldlm_enqueue`, is the maximum of the measured enqueue estimate (influenced by `at_min` and `at_max` parameters), multiplied by a weighting factor and the value of `ldlm_enqueue_min`.Lustre Distributed Lock Manager (LDLM) lock enqueues have a dedicated minimum value for `ldlm_enqueue_min`. Lock enqueue timeouts increase as the measured enqueue times increase (similar to adaptive timeouts). |

#### Interpreting Adaptive Timeout Information

Adaptive timeout information can be obtained via `lctl get_param {osc,mdc}.*.timeouts` files on each client and `lctl get_param {ost,mds}.*.*.timeouts` on each server. To read information from a `timeouts` file, enter a command similar to:

```
# lctl get_param -n ost.*.ost_io.timeouts
service : cur 33  worst 34 (at 1193427052, 1600s ago) 1 1 33 2
```

In this example, the `ost_io` service on this node is currently reporting an estimated RPC service time of 33 seconds. The worst RPC service time was 34 seconds, which occurred 26 minutes ago.

The output also provides a history of service times. Four "bins" of adaptive timeout history are shown, with the maximum RPC time in each bin reported. In both the 0-150s bin and the 150-300s bin, the maximum RPC time was 1. The 300-450s bin shows the worst (maximum) RPC time at 33 seconds, and the 450-600s bin shows a maximum of RPC time of 2 seconds. The estimated service time is the maximum value in the four bins (33 seconds in this example).

Service times (as reported by the servers) are also tracked in the client OBDs, as shown in this example:

```
# lctl get_param osc.*.timeouts
last reply : 1193428639, 0d0h00m00s ago
network    : cur  1 worst  2 (at 1193427053, 0d0h26m26s ago)  1  1  1  1
portal 6   : cur 33 worst 34 (at 1193427052, 0d0h26m27s ago) 33 33 33  2
portal 28  : cur  1 worst  1 (at 1193426141, 0d0h41m38s ago)  1  1  1  1
portal 7   : cur  1 worst  1 (at 1193426141, 0d0h41m38s ago)  1  0  1  1
portal 17  : cur  1 worst  1 (at 1193426177, 0d0h41m02s ago)  1  0  0  1
```

In this example, portal 6, the `ost_io` service portal, shows the history of service estimates reported by the portal.

Server statistic files also show the range of estimates including min, max, sum, and sum-squared. For example:

```
# lctl get_param mdt.*.mdt.stats
...
req_timeout               6 samples [sec] 1 10 15 105
...
```

### Setting Static Timeouts

The Lustre software provides two sets of static (fixed) timeouts, LND timeouts and Lustre timeouts, which are used when adaptive timeouts are not enabled.

- **LND timeouts** - LND timeouts ensure that point-to-point communications across a network complete in a finite time in the presence of failures, such as packages lost or broken connections. LND timeout parameters are set for each individual LND.

  LND timeouts are logged with the `S_LND` flag set. They are not printed as console messages, so check the Lustre log for `D_NETERROR` messages or enable printing of `D_NETERROR` messages to the console using:

  ```
  lctl set_param printk=+neterror
  ```

  Congested routers can be a source of spurious LND timeouts. To avoid this situation, increase the number of LNet router buffers to reduce back-pressure and/or increase LND timeouts on all nodes on all connected networks. Also consider increasing the total number of LNet router nodes in the system so that the aggregate router bandwidth matches the aggregate server bandwidth.

- **Lustre timeouts** - Lustre timeouts ensure that Lustre RPCs complete in a finite time in the presence of failures when adaptive timeouts are not enabled. Adaptive timeouts are enabled by default. To disable adaptive timeouts at run time, set `at_max` to 0 by running on the MGS:

  ```
  # lctl conf_param fsname.sys.at_max=0
  ```

  ### Note

  Changing the status of adaptive timeouts at runtime may cause a transient client timeout, recovery, and reconnection.

  Lustre timeouts are always printed as console messages.

  If Lustre timeouts are not accompanied by LND timeouts, increase the Lustre timeout on both servers and clients. Lustre timeouts are set using a command such as the following:

  ```
  # lctl set_param timeout=30
  ```

  Lustre timeout parameters are described in the table below.

| Parameter          | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `timeout`          | The time that a client waits for a server to complete an RPC (default 100s). Servers wait half this time for a normal client RPC to complete and a quarter of this time for a single bulk request (read or write of up to 4 MB) to complete. The client pings recoverable targets (MDS and OSTs) at one quarter of the timeout, and the server waits one and a half times the timeout before evicting a client for being "stale."Lustre client sends periodic 'ping' messages to servers with which it has had no communication for the specified period of time. Any network activity between a client and a server in the file system also serves as a ping. |
| `ldlm_timeout`     | The time that a server waits for a client to reply to an initial AST (lock cancellation request). The default is 20s for an OST and 6s for an MDS. If the client replies to the AST, the server will give it a normal timeout (half the client timeout) to flush any dirty data and release the lock. |
| `fail_loc`         | An internal debugging failure hook. The default value of `0` means that no failure will be triggered or injected. |
| `dump_on_timeout`  | Triggers a dump of the Lustre debug log when a timeout occurs. The default value of `0` (zero) means a dump of the Lustre debug log will not be triggered. |
| `dump_on_eviction` | Triggers a dump of the Lustre debug log when an eviction occurs. The default value of `0`(zero) means a dump of the Lustre debug log will not be triggered. |

## Monitoring LNet

LNet information is located via `lctl get_param` in these parameters:

- `peers` - Shows all NIDs known to this node and provides information on the queue state.

  Example:

  ```
  # lctl get_param peers
  nid                refs   state  max  rtr  min   tx    min   queue
  0@lo               1      ~rtr   0    0    0     0     0     0
  192.168.10.35@tcp  1      ~rtr   8    8    8     8     6     0
  192.168.10.36@tcp  1      ~rtr   8    8    8     8     6     0
  192.168.10.37@tcp  1      ~rtr   8    8    8     8     6     0
  ```

  The fields are explained in the table below:

  | **Field** | **Description**                                              |
  | --------- | ------------------------------------------------------------ |
  | `refs`    | A reference count.                                           |
  | `state`   | If the node is a router, indicates the state of the router. Possible values are:`NA` - Indicates the node is not a router.`up/down`- Indicates if the node (router) is up or down. |
  | `max`     | Maximum number of concurrent sends from this peer.           |
  | `rtr`     | Number of available routing buffer credits.                  |
  | `min`     | Minimum number of routing buffer credits seen.               |
  | `tx`      | Number of available send credits.                            |
  | `min`     | Minimum number of send credits seen.                         |
  | `queue`   | Total bytes in active/queued sends.                          |

  Credits are initialized to allow a certain number of operations (in the example above the table, eight as shown in the max column. LNet keeps track of the minimum number of credits ever seen over time showing the peak congestion that has occurred during the time monitored. Fewer available credits indicates a more congested resource.

  The number of credits currently available is shown in the tx column. The maximum number of send credits is shown in the max column and never changes. The number of currently active transmits can be derived by (max - tx), as long as tx is greater than or equal to 0. Once tx is less than 0, it indicates the number of transmits on that peer which have been queued for lack of credits.

  The number of router buffer credits available for consumption by a peer is shown in rtr column. The number of routing credits can be configured separately at the LND level or at the LNet level by using the `peer_buffer_credits` module parameter for the appropriate module. If the routing credits is not set explicitly, it'll default to the maximum transmit credits defined by peer_credits module parameter. Whenever a gateway routes a message from a peer, it decrements the number of available routing credits for that peer. If that value goes to zero, then messages will be queued. Negative values show the number of queued message waiting to be routed. The number of messages which are currently being routed from a peer can be derived by `(max_rtr_credits - rtr)`.

  LNet also limits concurrent sends and number of router buffers allocated to a single peer so that no peer can occupy all these resources.

- `nis` - Shows current queue health on the node.

  Example:

  ```
  # lctl get_param nis
  nid                    refs   peer    max   tx    min
  0@lo                   3      0       0     0     0
  192.168.10.34@tcp      4      8       256   256   252
  ```

  The fields are explained in the table below.

  | **Field** | **Description**                                              |
  | --------- | ------------------------------------------------------------ |
  | `nid`     | Network interface.                                           |
  | `refs`    | Internal reference counter.                                  |
  | `peer`    | Number of peer-to-peer send credits on this NID. Credits are used to size buffer pools. |
  | `max`     | Total number of send credits on this NID.                    |
  | `tx`      | Current number of send credits available on this NID.        |
  | `min`     | Lowest number of send credits available on this NID.         |
  | `queue`   | Total bytes in active/queued sends.                          |

  **Analysis:**

  Subtracting `max` from `tx` (`max` - `tx`) yields the number of sends currently active. A large or increasing number of active sends may indicate a problem.

## Allocating Free Space on OSTs

  Free space is allocated using either a round-robin or a weighted algorithm. The allocation method is determined by the maximum amount of free-space imbalance between the OSTs. When free space is relatively balanced across OSTs, the faster round-robin allocator is used, which maximizes network balancing. The weighted allocator is used when any two OSTs are out of balance by more than a specified threshold.

  Free space distribution can be tuned using these two tunable parameters:

  - `lod.*.qos_threshold_rr` - The threshold at which the allocation method switches from round-robin to weighted is set in this file. The default is to switch to the weighted algorithm when any two OSTs are out of balance by more than 17 percent.
  - `lod.*.qos_prio_free` - The weighting priority used by the weighted allocator can be adjusted in this file. Increasing the value of `qos_prio_free` puts more weighting on the amount of free space available on each OST and less on how stripes are distributed across OSTs. The default value is 91 percent weighting for free space rebalancing and 9 percent for OST balancing. When the free space priority is set to 100, weighting is based entirely on free space and location is no longer used by the striping algorithm.
  - Introduced in Lustre 2.9`osp.*.reserved_mb_low` - The low watermark used to stop object allocation if available space is less than this. The default is 0.1% of total OST size.
  - Introduced in Lustre 2.9`osp.*.reserved_mb_high` - The high watermark used to start object allocation if available space is more than this. The default is 0.2% of total OST size.

  For more information about monitoring and managing free space, see [*the section called “Managing Free Space”*](03.08-Managing%20File%20Layout%20(Striping)%20and%20Free%20Space.md#managing-free-space).

## Configuring Locking

  The `lru_size` parameter is used to control the number of client-side locks in the LRU cached locks queue. LRU size is normally dynamic, based on load to optimize the number of locks cached on nodes that have different workloads (e.g., login/build nodes vs. compute nodes vs. backup nodes).

  The total number of locks available is a function of the server RAM. The default limit is 50 locks/1 MB of RAM. If memory pressure is too high, the LRU size is shrunk. The number of locks on the server is limited to*num_osts_per_oss \* num_clients * lru_size* as follows:

  - To enable automatic LRU sizing, set the `lru_size` parameter to 0. In this case, the `lru_size` parameter shows the current number of locks being used on the client. Dynamic LRU resizing is enabled by default.
  - To specify a maximum number of locks, set the `lru_size` parameter to a value other than zero. A good default value for compute nodes is around `100 * *num_cpus*`. It is recommended that you only set `lru_size` to be signifivantly larger on a few login nodes where multiple users access the file system interactively.

  To clear the LRU on a single client, and, as a result, flush client cache without changing the `lru_size` value, run:

  ```
  # lctl set_param ldlm.namespaces.osc_name|mdc_name.lru_size=clear
  ```

  If the LRU size is set lower than the number of existing locks, *unused* locks are canceled immediately. Use `clear` to cancel all locks without changing the value.

  **Note**

  The `lru_size` parameter can only be set temporarily using `lctl set_param`, it cannot be set permanently.

  To disable dynamic LRU resizing on the clients, run for example:

  ```
  # lctl set_param ldlm.namespaces.*osc*.lru_size=5000
  ```

  To determine the number of locks being granted with dynamic LRU resizing, run:

  ```
  $ lctl get_param ldlm.namespaces.*.pool.limit
  ```

  The `lru_max_age` parameter is used to control the age of client-side locks in the LRU cached locks queue. This limits how long unused locks are cached on the client, and avoids idle clients from holding locks for an excessive time, which reduces memory usage on both the client and server, as well as reducing work during server recovery.

  The `lru_max_age` is set and printed in milliseconds, and by default is 3900000 ms (65 minutes).

  Introduced in Lustre 2.11Since Lustre 2.11, in addition to setting the maximum lock age in milliseconds, it can also be set using a suffix of `s`or `ms` to indicate seconds or milliseconds, respectively. For example to set the client's maximum lock age to 15 minutes (900s) run:

  ```
  # lctl set_param ldlm.namespaces.*MDT*.lru_max_age=900s
  # lctl get_param ldlm.namespaces.*MDT*.lru_max_age
  ldlm.namespaces.myth-MDT0000-mdc-ffff8804296c2800.lru_max_age=900000
  ```

## Setting MDS and OSS Thread Counts

  MDS and OSS thread counts tunable can be used to set the minimum and maximum thread counts or get the current number of running threads for the services listed in the table below.

| **Service**                  | **Description**                             |
| ---------------------------- | ------------------------------------------- |
| `mds.MDS.mdt`                | Main metadata operations service            |
| `mds.MDS.mdt_readpage`       | Metadata `readdir` service                  |
| `mds.MDS.mdt_setattr`        | Metadata `setattr/close` operations service |
| `ost.OSS.ost`                | Main data operations service                |
| `ost.OSS.ost_io`             | Bulk data I/O services                      |
| `ost.OSS.ost_create`         | OST object pre-creation service             |
| `ldlm.services.ldlm_canceld` | DLM lock cancel service                     |
| `ldlm.services.ldlm_cbd`     | DLM lock grant service                      |

  For each service, tunable parameters as shown below are available.

  - To temporarily set these tunables, run:

    ```
    # lctl set_param service.threads_min|max|started=num 
    ```

  - To permanently set this tunable, run:

    ```
    # lctl conf_param obdname|fsname.obdtype.threads_min|max|started 
    ```

    Introduced in Lustre 2.5For version 2.5 or later, run:`# lctl set_param -P *service*.threads_*min|max|started*`

  The following examples show how to set thread counts and get the number of running threads for the service `ost_io` using the tunable `*service*.threads_*min|max|started*`.

  - To get the number of running threads, run:

    ```
    # lctl get_param ost.OSS.ost_io.threads_started
    ost.OSS.ost_io.threads_started=128
    ```

  - To set the number of threads to the maximum value (512), run:

    ```
    # lctl get_param ost.OSS.ost_io.threads_max
    ost.OSS.ost_io.threads_max=512
    ```

  - To set the maximum thread count to 256 instead of 512 (to avoid overloading the storage or for an array with requests), run:

    ```
    # lctl set_param ost.OSS.ost_io.threads_max=256
    ost.OSS.ost_io.threads_max=256
    ```

  - To set the maximum thread count to 256 instead of 512 permanently, run:

    ```
    # lctl conf_param testfs.ost.ost_io.threads_max=256
    ```

    Introduced in Lustre 2.5For version 2.5 or later, run:`# lctl set_param -P ost.OSS.ost_io.threads_max=256 ost.OSS.ost_io.threads_max=256 `

  - To check if the `threads_max` setting is active, run:

    ```
    # lctl get_param ost.OSS.ost_io.threads_max
    ost.OSS.ost_io.threads_max=256
    ```

  **Note**

  If the number of service threads is changed while the file system is running, the change may not take effect until the file system is stopped and rest. If the number of service threads in use exceeds the new `threads_max` value setting, service threads that are already running will not be stopped.

  See also [*Tuning a Lustre File System*](04.03-Tuning%20a%20Lustre%20File%20System.md)

## Enabling and Interpreting Debugging Logs

By default, a detailed log of all operations is generated to aid in debugging. Flags that control debugging are found via `lctl get_param debug`.

The overhead of debugging can affect the performance of Lustre file system. Therefore, to minimize the impact on performance, the debug level can be lowered, which affects the amount of debugging information kept in the internal log buffer but does not alter the amount of information to goes into syslog. You can raise the debug level when you need to collect logs to debug problems.

The debugging mask can be set using "symbolic names". The symbolic format is shown in the examples below.

- To verify the debug level used, examine the parameter that controls debugging by running:

  ```
  # lctl get_param debug 
  debug=
  ioctl neterror warning error emerg ha config console
  ```

- To turn off debugging except for network error debugging, run the following command on all nodes concerned:

  ```
  # sysctl -w lnet.debug="neterror" 
  debug=neterror
  ```

- To turn off debugging completely (except for the minimum error reporting to the console), run the following command on all nodes concerned:

  ```
  # lctl set_param debug=0 
  debug=0
  ```

- To set an appropriate debug level for a production environment, run:

  ```
  # lctl set_param debug="warning dlmtrace error emerg ha rpctrace vfstrace" 
  debug=warning dlmtrace error emerg ha rpctrace vfstrace
  ```

  The flags shown in this example collect enough high-level information to aid debugging, but they do not cause any serious performance impact.

- To add new flags to flags that have already been set, precede each one with a "`+`":

  ```
  # lctl set_param debug="+neterror +ha" 
  debug=+neterror +ha
  # lctl get_param debug 
  debug=neterror warning error emerg ha console
  ```

- To remove individual flags, precede them with a "`-`":

  ```
  # lctl set_param debug="-ha" 
  debug=-ha
  # lctl get_param debug 
  debug=neterror warning error emerg console
  ```

Debugging parameters include:

- `subsystem_debug` - Controls the debug logs for subsystems.
- `debug_path` - Indicates the location where the debug log is dumped when triggered automatically or manually. The default path is `/tmp/lustre-log`.

These parameters can also be set using:

```
sysctl -w lnet.debug={value}
```

Additional useful parameters:

- `panic_on_lbug` - Causes ''panic'' to be called when the Lustre software detects an internal problem (an `LBUG`log entry); panic crashes the node. This is particularly useful when a kernel crash dump utility is configured. The crash dump is triggered when the internal inconsistency is detected by the Lustre software.

- `upcall` - Allows you to specify the path to the binary which will be invoked when an `LBUG` log entry is encountered. This binary is called with four parameters:

  \- The string ''`LBUG`''.

  \- The file where the `LBUG` occurred.

  \- The function name.

  \- The line number in the file

### Interpreting OST Statistics

**Note**

See also [*the section called “ llobdstat”*](06.07-System%20Configuration%20Utilities.md#llobdstat)(`llobdstat`) and [*the section called “ `CollectL` ”*](03.01-Monitoring%20a%20Lustre%20File%20System.md#collectl) (`collectl`).

OST `stats` files can be used to provide statistics showing activity for each OST. For example:

```
# lctl get_param osc.testfs-OST0000-osc.stats 
snapshot_time                      1189732762.835363
ost_create                 1
ost_get_info               1
ost_connect                1
ost_set_info               1
obd_ping                   212
```

Use the `llstat` utility to monitor statistics over time.

To clear the statistics, use the `-c` option to `llstat`. To specify how frequently the statistics should be reported (in seconds), use the `-i` option. In the example below, the `-c` option clears the statistics and `-i10` option reports statistics every 10 seconds:

```
$ llstat -c -i10 ost_io
 
/usr/bin/llstat: STATS on 06/06/07 
        /proc/fs/lustre/ost/OSS/ost_io/ stats on 192.168.16.35@tcp
snapshot_time                              1181074093.276072
 
/proc/fs/lustre/ost/OSS/ost_io/stats @ 1181074103.284895
Name        Cur.  Cur. #
            Count Rate Events Unit  last   min    avg       max    stddev
req_waittime 8    0    8    [usec]  2078   34     259.75    868    317.49
req_qdepth   8    0    8    [reqs]  1      0      0.12      1      0.35
req_active   8    0    8    [reqs]  11     1      1.38      2      0.52
reqbuf_avail 8    0    8    [bufs]  511    63     63.88     64     0.35
ost_write    8    0    8    [bytes] 169767 72914  212209.62 387579 91874.29
 
/proc/fs/lustre/ost/OSS/ost_io/stats @ 1181074113.290180
Name        Cur.  Cur. #
            Count Rate Events Unit  last    min   avg       max    stddev
req_waittime 31   3    39   [usec]  30011   34    822.79    12245  2047.71
req_qdepth   31   3    39   [reqs]  0       0     0.03      1      0.16
req_active   31   3    39   [reqs]  58      1     1.77      3      0.74
reqbuf_avail 31   3    39   [bufs]  1977    63    63.79     64     0.41
ost_write    30   3    38   [bytes] 1028467 15019 315325.16 910694 197776.51
 
/proc/fs/lustre/ost/OSS/ost_io/stats @ 1181074123.325560
Name        Cur.  Cur. #
            Count Rate Events Unit  last    min    avg       max    stddev
req_waittime 21   2    60   [usec]  14970   34     784.32    12245  1878.66
req_qdepth   21   2    60   [reqs]  0       0      0.02      1      0.13
req_active   21   2    60   [reqs]  33      1      1.70      3      0.70
reqbuf_avail 21   2    60   [bufs]  1341    63     63.82     64     0.39
ost_write    21   2    59   [bytes] 7648424 15019  332725.08 910694 180397.87
```

The columns in this example are described in the table below.

| **Parameter** | **Description**                                              |
| ------------- | ------------------------------------------------------------ |
| `Name`        | Name of the service event. See the tables below for descriptions of service events that are tracked. |
| `Cur. Count`  | Number of events of each type sent in the last interval.     |
| `Cur. Rate`   | Number of events per second in the last interval.            |
| `# Events`    | Total number of such events since the events have been cleared. |
| `Unit`        | Unit of measurement for that statistic (microseconds, requests, buffers). |
| `last`        | Average rate of these events (in units/event) for the last interval during which they arrived. For instance, in the above mentioned case of `ost_destroy` it took an average of 736 microseconds per destroy for the 400 object destroys in the previous 10 seconds. |
| `min`         | Minimum rate (in units/events) since the service started.    |
| `avg`         | Average rate.                                                |
| `max`         | Maximum rate.                                                |
| `stddev`      | Standard deviation (not measured in some cases)              |

Events common to all services are shown in the table below.

| **Parameter**  | **Description**                                              |
| -------------- | ------------------------------------------------------------ |
| `req_waittime` | Amount of time a request waited in the queue before being handled by an available server thread. |
| `req_qdepth`   | Number of requests waiting to be handled in the queue for this service. |
| `req_active`   | Number of requests currently being handled.                  |
| `reqbuf_avail` | Number of unsolicited lnet request buffers for this service. |

Some service-specific events of interest are described in the table below.

| **Parameter**  | **Description**                                              |
| -------------- | ------------------------------------------------------------ |
| `ldlm_enqueue` | Time it takes to enqueue a lock (this includes file open on the MDS) |
| `mds_reint`    | Time it takes to process an MDS modification record (includes `create`, `mkdir`, `unlink`, `rename`and `setattr`) |

### Interpreting MDT Statistics

**Note**

See also [*the section called “ llobdstat”*](06.07-System%20Configuration%20Utilities.md#llobdstat)(`llobdstat`) and [*the section called “ `CollectL` ”*](03.01-Monitoring%20a%20Lustre%20File%20System.md#collectl) (`collectl`).

MDT `stats` files can be used to track MDT statistics for the MDS. The example below shows sample output from an MDT `stats` file.

```
# lctl get_param mds.*-MDT0000.stats
snapshot_time                   1244832003.676892 secs.usecs 
open                            2 samples [reqs]
close                           1 samples [reqs]
getxattr                        3 samples [reqs]
process_config                  1 samples [reqs]
connect                         2 samples [reqs]
disconnect                      2 samples [reqs]
statfs                          3 samples [reqs]
setattr                         1 samples [reqs]
getattr                         3 samples [reqs]
llog_init                       6 samples [reqs] 
notify                          16 samples [reqs]
```

 
