Introduced in Lustre 2.12 

## Lazy Size on MDT (LSoM)

- [Lazy Size on MDT (LSoM)](#lazy-size-on-mdt-lsom)
- [Introduction to Lazy Size on MDT (LSoM)](#introduction-to-lazy-size-on-mdt-lsom)
- [Enable LSoM](#enable-lsom)
- [User Commands](#user-commands)
  * [lfs getsom for LSoM data](#lfs-getsom-for-lsom-data)
    + [lfs getsom Command](#lfs-getsom-command)
  * [Syncing LSoM data](#syncing-lsom-data)
    + [llsom_sync Command](#llsom_sync-command)

This chapter describes Lazy Size on MDT (LSoM).

## Introduction to Lazy Size on MDT (LSoM)

In the Lustre file system, MDSs store the ctime, mtime, owner, and other file attributes. The OSSs store the size and number of blocks used for each file. To obtain the correct file size, the client must contact each OST that the file is stored across, which means multiple RPCs to get the size and blocks for a file when a file is striped over multiple OSTs. The Lazy Size on MDT (LSoM) feature stores the file size on the MDS and avoids the need to fetch the file size from the OST(s) in cases where the application understands that the size may not be accurate. Lazy means there is no guarantee of the accuracy of the attributes stored on the MDS.

Since many Lustre installations use SSD for MDT storage, the motivation for the LSoM work is to speed up the time it takes to get the size of a file from the Lustre file system by storing that data on the MDTs. We expect this feature to be initially used by Lustre policy engines that scan the backend MDT storage, make decisions based on broad size categories, and do not depend on a totally accurate file size. Examples include Lester, Robinhood, Zester, and various vendor offerings. Future improvements will allow the LSoM data to be accessed by tools such as `lfs find`.

## Enable LSoM

LSoM is always enabled and nothing needs to be done to enable the feature for fetching the LSoM data when scanning the MDT inodes with a policy engine. It is also possible to access the LSoM data on the client via the `lfs getsom` command. Because the LSoM data is currently accessed on the client via the xattr interface, the`xattr_cache` will cache the file size and block count on the client as long as the inode is cached. In most cases this is desirable, since it improves access to the LSoM data. However, it also means that the LSoM data may be stale if the file size is changed after the xattr is first accessed or if the xattr is accessed shortly after the file is first created.

If it is necessary to access up-to-date LSoM data that has gone stale, it is possible to flush the xattr cache from the client by cancelling the MDC locks via `lctl set_param ldlm.namespaces.*mdc*.lru_size=clear`. Otherwise, the file attributes will be dropped from the client cache if the file has not been accessed before the LDLM lock timeout. The timeout is stored via `lctl get_param ldlm.namespaces.*mdc*.lru_max_age`.

If repeated access to LSoM attributes for files that are recently created or frequently modified from a specific client, such as an HSM agent node, it is possible to disable xattr caching on a client via: `lctl set_param llite.*.xattr_cache=0`. This may cause extra overhead when accessing files, and is not recommended for normal usage.

## User Commands

Lustre provides the `lfs getsom` command to list file attributes that are stored on the MDT.

The `llsom_sync` command allows the user to sync the file attributes on the MDT with the valid/up-to-date data on the OSTs. `llsom_sync` is called on the client with the Lustre file system mount point. `llsom_sync` uses Lustre MDS changelogs and, thus, a changelog user must be registered to use this utility.

### lfs getsom for LSoM data
The `lfs getsom` command lists file attributes that are stored on the MDT. `lfs getsom` is called with the full path and file name for a file on the Lustre file system. If no flags are used, then all file attributes stored on the MDS will be shown.

#### lfs getsom Command

```
lfs getsom [-s] [-b] [-f] <filename>
```

The various `lfs getsom` options are listed and described below.

| **Option** | **Description**                                              |
| ---------- | ------------------------------------------------------------ |
| `-s`       | Only show the size value of the LSoM data for a given file. This is an optional flag |
| `-b`       | Only show the blocks value of the LSoM data for a given file. This is an optional flag |
| `-f`       | Only show the flag value of the LSoM data for a given file. This is an optional flag. Valid flags are:SOM_FL_UNKNOWN = 0x0000 - Unknown or no SoM data, must get size from OSTs.SOM_FL_STRICT = 0x0001 - Known strictly correct, FLR file (SoM guaranteed)SOM_FL_STALE = 0x0002 - Known stale -was right at some point in the past, but it is known (or likely) to be incorrect now (e.g. opened for write)SOM_FL_LAZY= 0x0004 - Approximate, may never have been strictly correct, need to sync SOM data to achieve eventual consistency. |

### Syncing LSoM data

The `llsom_sync` command allows the user to sync the file attributes on the MDT with the valid/up-to-date data on the OSTs. `llsom_sync` is called on the client with the client mount point for the Lustre file system. `llsom_sync` uses Lustre MDS changelogs and, thus, a changelog user must be registered to use this utility.

#### llsom_sync Command

```
llsom_sync --mdt|-m <mdt> --user|-u <user_id>
              [--daemonize|-d] [--verbose|-v] [--interval|-i] [--min-age|-a]
              [--max-cache|-c] [--sync|-s] <lustre_mount_point>
```

The various `llsom_sync` options are listed and described below.

| **Option**              | **Description**                                              |
| ----------------------- | ------------------------------------------------------------ |
| `--mdt | -m <mdt>`      | The metadata device which need to be synced the LSoM xattr of files. A changelog user must be registered for this device.Required flag. |
| `--user | -u <user_id>` | The changelog user id for the MDT device. Required flag.     |
| `--daemonize | -d`      | Optional flag to “daemonize” the program. In daemon mode, the utility will scan, process the changelog records and sync the LSoM xattr for files periodically. |
| `--verbose | -v`        | Optional flag to produce verbose output.                     |
| `--interval | -i`       | Optional flag for the time interval to scan the Lustre changelog and process the log record in daemon mode. |
| `--min-age | -a`        | Optional flag for the time that `llsom_sync` tool will not try to sync the LSoM data for any files closed less than this many seconds old. The default min-age value is 600s(10 minutes). |
| `--max-cache | -c`      | Optional flag for the total memory used for the FID cache which can be with a suffix [KkGgMm].The default max-cache value is 256MB. For the parameter value < 100, it is taken as the percentage of total memory size used for the FID cache instead of the cache size. |
| `--sync | -s`           | Optional flag to sync file data to make the dirty data out of cache to ensure the blocks count is correct when update the file LSoM xattr. This option could hurt server performance significantly if thousands of fsync requests are sent. |