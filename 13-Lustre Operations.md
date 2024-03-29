# Lustre Operations

Once you have the Lustre file system up and running, you can use the procedures in this section to perform these basic Lustre administration tasks.

- [Lustre Operations](#lustre-operations)
  * [Mounting by Label](#mounting-by-label)
  * [Starting Lustre](#starting-lustre)
  * [Mounting a Server](#mounting-a-server)
  * [Stopping the Filesystem](#stopping-the-filesystem)
  * [Unmounting a Specific Target on a Server](#unmounting-a-specific-target-on-a-server)
  * [Specifying Failout/Failover Mode for OSTs](#specifying-failoutfailover-mode-for-osts)
  * [Handling Degraded OST RAID Arrays](#handling-degraded-ost-raid-arrays)
  * [Running Multiple Lustre File Systems](#running-multiple-lustre-file-systems)
  * [Creating a sub-directory on a specific MDT](#creating-a-sub-directory-on-a-specific-mdt)L2.4
  * [Creating a directory striped across multiple MDTs](#creating-a-directory-striped-across-multiple-mdts)L2.8
  * [Setting and Retrieving Lustre Parameters](#setting-and-retrieving-lustre-parameters)
    + [Setting Tunable Parameters with `mkfs.lustre`](#setting-tunable-parameters-with-mkfslustre)
    + [Setting Parameters with `tunefs.lustre`](#setting-parameters-with-tunefslustre)
    + [Setting Parameters with `lctl`](#setting-parameters-with-lctl)
      - [Setting Temporary Parameters](#setting-temporary-parameters)
      - [Setting Permanent Parameters](#setting-permanent-parameters)
      - [Setting Permanent Parameters with lctl set_param -P](#setting-permanent-parameters-with-lctl-set_param--p)L2.5
      - [Listing Parameters](#listing-parameters)
      - [Reporting Current Parameter Values](#reporting-current-parameter-values)
  * [Specifying NIDs and Failover](#specifying-nids-and-failover)
  * [Erasing a File System](#erasing-a-file-system)
  * [Reclaiming Reserved Disk Space](#reclaiming-reserved-disk-space)
  * [Replacing an Existing OST or MDT](#replacing-an-existing-ost-or-mdt)
  * [Identifying To Which Lustre File an OST Object Belongs](#identifying-to-which-lustre-file-an-ost-object-belongs)

## Mounting by Label

The file system name is limited to 8 characters. We have encoded the file system and target information in the disk label, so you can mount by label. This allows system administrators to move disks around without worrying about issues such as SCSI disk reordering or getting the `/dev/device` wrong for a shared target. Soon, file system naming will be made as fail-safe as possible. Currently, Linux disk labels are limited to 16 characters. To identify the target within the file system, 8 characters are reserved, leaving 8 characters for the file system name:

```
fsname-MDT0000 or 
fsname-OST0a19
```

To mount by label, use this command:

```
mount -t lustre -L 
file_system_label 
/mount_point
```

This is an example of mount-by-label:

```
mds# mount -t lustre -L testfs-MDT0000 /mnt/mdt
```

**Caution**

Mount-by-label should NOT be used in a multi-path environment or when snapshots are being created of the device, since multiple block devices will have the same label.

Although the file system name is internally limited to 8 characters, you can mount the clients at any mount point, so file system users are not subjected to short names. Here is an example:

```
client# mount -t lustre mds0@tcp0:/short 
/dev/long_mountpoint_name
```

## Starting Lustre

On the first start of a Lustre file system, the components must be started in the following order:

1. Mount the MGT.

   ### Note

   If a combined MGT/MDT is present, Lustre will correctly mount the MGT and MDT automatically.

2. Mount the MDT.

   ### Note

   Introduced in Lustre 2.4Mount all MDTs if multiple MDTs are present.

3. Mount the OST(s).

4. Mount the client(s).

## Mounting a Server

Starting a Lustre server is straightforward and only involves the mount command. Lustre servers can be added to `/etc/fstab`:

```
mount -t lustre
```

The mount command generates output similar to this:

```
/dev/sda1 on /mnt/test/mdt type lustre (rw)
/dev/sda2 on /mnt/test/ost0 type lustre (rw)
192.168.0.21@tcp:/testfs on /mnt/testfs type lustre (rw)
```

In this example, the MDT, an OST (ost0) and file system (testfs) are mounted.

```
LABEL=testfs-MDT0000 /mnt/test/mdt lustre defaults,_netdev,noauto 0 0
LABEL=testfs-OST0000 /mnt/test/ost0 lustre defaults,_netdev,noauto 0 0
```

In general, it is wise to specify noauto and let your high-availability (HA) package manage when to mount the device. If you are not using failover, make sure that networking has been started before mounting a Lustre server. If you are running Red Hat Enterprise Linux, SUSE Linux Enterprise Server, Debian operating system (and perhaps others), use the `_netdev` flag to ensure that these disks are mounted after the network is up.

We are mounting by disk label here. The label of a device can be read with `e2label`. The label of a newly-formatted Lustre server may end in `FFFF` if the `--index` option is not specified to `mkfs.lustre`, meaning that it has yet to be assigned. The assignment takes place when the server is first started, and the disk label is updated. It is recommended that the `--index` option always be used, which will also ensure that the label is set at format time.

**Caution**

Do not do this when the client and OSS are on the same node, as memory pressure between the client and OSS can lead to deadlocks.

**Caution**

Mount-by-label should NOT be used in a multi-path environment.



## Stopping the Filesystem

A complete Lustre filesystem shutdown occurs by unmounting all clients and servers in the order shown below. Please note that unmounting a block device causes the Lustre software to be shut down on that node.

**Note**

Please note that the `-a -t lustre` in the commands below is not the name of a filesystem, but rather is specifying to unmount all entries in /etc/mtab that are of type `lustre`

1. Unmount the clients

   On each client node, unmount the filesystem on that client using the `umount` command:

   `umount -a -t lustre`

   The example below shows the unmount of the `testfs` filesystem on a client node:

   ```
   [root@client1 ~]# mount |grep testfs
   XXX.XXX.0.11@tcp:/testfs on /mnt/testfs type lustre (rw,lazystatfs)
   
   [root@client1 ~]# umount -a -t lustre
   [154523.177714] Lustre: Unmounted testfs-client
   ```

2. Unmount the MDT and MGT

   On the MGS and MDS node(s), run the `umount` command:

   `umount -a -t lustre`

   The example below shows the unmount of the MDT and MGT for the `testfs` filesystem on a combined MGS/MDS:

   ```
   [root@mds1 ~]# mount |grep lustre
   /dev/sda on /mnt/mgt type lustre (ro)
   /dev/sdb on /mnt/mdt type lustre (ro)
   
   [root@mds1 ~]# umount -a -t lustre
   [155263.566230] Lustre: Failing over testfs-MDT0000
   [155263.775355] Lustre: server umount testfs-MDT0000 complete
   [155269.843862] Lustre: server umount MGS complete
   ```

   For a seperate MGS and MDS, the same command is used, first on the MDS and then followed by the MGS.

3. Unmount all the OSTs

   On each OSS node, use the `umount` command:

   `umount -a -t lustre`

   The example below shows the unmount of all OSTs for the `testfs` filesystem on server `OSS1`:

   ```
   [root@oss1 ~]# mount |grep lustre
   /dev/sda on /mnt/ost0 type lustre (ro)
   /dev/sdb on /mnt/ost1 type lustre (ro)
   /dev/sdc on /mnt/ost2 type lustre (ro)
   
   [root@oss1 ~]# umount -a -t lustre
   [155336.491445] Lustre: Failing over testfs-OST0002
   [155336.556752] Lustre: server umount testfs-OST0002 complete
   ```

For unmount command syntax for a single OST, MDT, or MGT target please refer to [*the section called “ Unmounting a Specific Target on a Server”*](#unmounting-a-specific-target-on-a-server)

## Unmounting a Specific Target on a Server

To stop a Lustre OST, MDT, or MGT , use the `umount */mount_point*` command.

The example below stops an OST, `ost0`, on mount point `/mnt/ost0` for the `testfs` filesystem:

```
[root@oss1 ~]# umount /mnt/ost0
[  385.142264] Lustre: Failing over testfs-OST0000
[  385.210810] Lustre: server umount testfs-OST0000 complete
```

Gracefully stopping a server with the `umount` command preserves the state of the connected clients. The next time the server is started, it waits for clients to reconnect, and then goes through the recovery procedure.

If the force ( `-f`) flag is used, then the server evicts all clients and stops WITHOUT recovery. Upon restart, the server does not wait for recovery. Any currently connected clients receive I/O errors until they reconnect.

**Note**

If you are using loopback devices, use the `-d` flag. This flag cleans up loop devices and can always be safely specified.

## Specifying Failout/Failover Mode for OSTs

In a Lustre file system, an OST that has become unreachable because it fails, is taken off the network, or is unmounted can be handled in one of two ways:

- In `failout` mode, Lustre clients immediately receive errors (EIOs) after a timeout, instead of waiting for the OST to recover.
- In `failover` mode, Lustre clients wait for the OST to recover.

By default, the Lustre file system uses `failover` mode for OSTs. To specify `failout` mode instead, use the `--param="failover.mode=failout"` option as shown below (entered on one line):

```
oss# mkfs.lustre --fsname=
fsname --mgsnode=
mgs_NID --param=failover.mode=failout 
      --ost --index=
ost_index 
/dev/ost_block_device
```

In the example below, `failout` mode is specified for the OSTs on the MGS `mds0` in the file system `testfs`(entered on one line).

```
oss# mkfs.lustre --fsname=testfs --mgsnode=mds0 --param=failover.mode=failout 
      --ost --index=3 /dev/sdb 
```

**Caution**

Before running this command, unmount all OSTs that will be affected by a change in `failover`/ `failout` mode.

**Note**

After initial file system configuration, use the `tunefs.lustre` utility to change the mode. For example, to set the `failout` mode, run:

```
$ tunefs.lustre --param failover.mode=failout 
/dev/ost_device
```

## Handling Degraded OST RAID Arrays

Lustre includes functionality that notifies Lustre if an external RAID array has degraded performance (resulting in reduced overall file system performance), either because a disk has failed and not been replaced, or because a disk was replaced and is undergoing a rebuild. To avoid a global performance slowdown due to a degraded OST, the MDS can avoid the OST for new object allocation if it is notified of the degraded state.

A parameter for each OST, called `degraded`, specifies whether the OST is running in degraded mode or not.

To mark the OST as degraded, use:

```
lctl set_param obdfilter.{OST_name}.degraded=1
```

To mark that the OST is back in normal operation, use:

```
lctl set_param obdfilter.{OST_name}.degraded=0
```

To determine if OSTs are currently in degraded mode, use:

```
lctl get_param obdfilter.*.degraded
```

If the OST is remounted due to a reboot or other condition, the flag resets to `0`.

It is recommended that this be implemented by an automated script that monitors the status of individual RAID devices, such as MD-RAID's `mdadm(8)` command with the `--monitor` option to mark an affected device degraded or restored.

## Running Multiple Lustre File Systems

Lustre supports multiple file systems provided the combination of `NID:fsname` is unique. Each file system must be allocated a unique name during creation with the `--fsname` parameter. Unique names for file systems are enforced if a single MGS is present. If multiple MGSs are present (for example if you have an MGS on every MDS) the administrator is responsible for ensuring file system names are unique. A single MGS and unique file system names provides a single point of administration and allows commands to be issued against the file system even if it is not mounted.

Lustre supports multiple file systems on a single MGS. With a single MGS fsnames are guaranteed to be unique. Lustre also allows multiple MGSs to co-exist. For example, multiple MGSs will be necessary if multiple file systems on different Lustre software versions are to be concurrently available. With multiple MGSs additional care must be taken to ensure file system names are unique. Each file system should have a unique fsname among all systems that may interoperate in the future.

By default, the `mkfs.lustre` command creates a file system named `lustre`. To specify a different file system name (limited to 8 characters) at format time, use the `--fsname` option:

```
mkfs.lustre --fsname=
file_system_name
```

**Note**

The MDT, OSTs and clients in the new file system must use the same file system name (prepended to the device name). For example, for a new file system named `foo`, the MDT and two OSTs would be named `foo-MDT0000`, `foo-OST0000`, and `foo-OST0001`.

To mount a client on the file system, run:

```
client# mount -t lustre 
mgsnode:
/new_fsname 
/mount_point
```

For example, to mount a client on file system foo at mount point /mnt/foo, run:

```
client# mount -t lustre mgsnode:/foo /mnt/foo
```

**Note**

If a client(s) will be mounted on several file systems, add the following line to `/etc/xattr.conf` file to avoid problems when files are moved between the file systems: `lustre.* skip`

**Note**

To ensure that a new MDT is added to an existing MGS create the MDT by specifying: `--mdt --mgsnode= *mgs_NID*`.

A Lustre installation with two file systems ( `foo` and `bar`) could look like this, where the MGS node is `mgsnode@tcp0` and the mount points are `/mnt/foo` and `/mnt/bar`.

```
mgsnode# mkfs.lustre --mgs /dev/sda
mdtfoonode# mkfs.lustre --fsname=foo --mgsnode=mgsnode@tcp0 --mdt --index=0
/dev/sdb
ossfoonode# mkfs.lustre --fsname=foo --mgsnode=mgsnode@tcp0 --ost --index=0
/dev/sda
ossfoonode# mkfs.lustre --fsname=foo --mgsnode=mgsnode@tcp0 --ost --index=1
/dev/sdb
mdtbarnode# mkfs.lustre --fsname=bar --mgsnode=mgsnode@tcp0 --mdt --index=0
/dev/sda
ossbarnode# mkfs.lustre --fsname=bar --mgsnode=mgsnode@tcp0 --ost --index=0
/dev/sdc
ossbarnode# mkfs.lustre --fsname=bar --mgsnode=mgsnode@tcp0 --ost --index=1
/dev/sdd
```

To mount a client on file system foo at mount point `/mnt/foo`, run:

```
client# mount -t lustre mgsnode@tcp0:/foo /mnt/foo
```

To mount a client on file system bar at mount point `/mnt/bar`, run:

```
client# mount -t lustre mgsnode@tcp0:/bar /mnt/bar
```

 

Introduced in Lustre 2.4

## Creating a sub-directory on a specific MDT

It is possible to create individual directories, along with its files and sub-directories, to be stored on specific MDTs. To create a sub-directory on a given MDT use the command:

```
client# lfs mkdir –i
mdt_index
/mount_point/remote_dir
```

This command will allocate the sub-directory `remote_dir` onto the MDT of index `mdt_index`. For more information on adding additional MDTs and `mdt_index` see [2](02.07-Configuring%20a%20Lustre%20File%20System.md##configuring-a-simple-lustre-file-system)

**Warning**

An administrator can allocate remote sub-directories to separate MDTs. Creating remote sub-directories in parent directories not hosted on MDT0000 is not recommended. This is because the failure of the parent MDT will leave the namespace below it inaccessible. For this reason, by default it is only possible to create remote sub-directories off MDT0000. To relax this restriction and enable remote sub-directories off any MDT, an administrator must issue the following command on the MGS:

```
mgs# lctl conf_param fsname.mdt.enable_remote_dir=1
```

For Lustre filesystem 'scratch', the command executed is:

```
mgs# lctl conf_param scratch.mdt.enable_remote_dir=1
```

To verify the configuration setting execute the following command on any MDS:

```
mds# lctl get_param mdt.*.enable_remote_dir
```

Introduced in Lustre 2.8

With Lustre software version 2.8, a new tunable is available to allow users with a specific group ID to create and delete remote and striped directories. This tunable is `enable_remote_dir_gid`. For example, setting this parameter to the 'wheel' or 'admin' group ID allows users with that GID to create and delete remote and striped directories. Setting this parameter to `-1` on MDT0000 to permanently allow any non-root users create and delete remote and striped directories. On the MGS execute the following command:

`mgs# lctl conf_param *fsname*.mdt.enable_remote_dir_gid=-1`

For the Lustre filesystem 'scratch', the commands expands to:

`mgs# lctl conf_param scratch.mdt.enable_remote_dir_gid=-1`. 

The change can be verified by executing the following command on every MDS:

`mds# lctl get_param mdt.***.enable_remote_dir_gid`

 Introduced in Lustre 2.8

## Creating a directory striped across multiple MDTs



The Lustre 2.8 DNE feature enables individual files in a given directory to store their metadata on separate MDTs (a *striped directory*) once additional MDTs have been added to the filesystem, see *the section called “Adding a New MDT to a Lustre File System”*. The result of this is that metadata requests for files in a striped directory are serviced by multiple MDTs and metadata service load is distributed over all the MDTs that service a given directory. By distributing metadata service load over multiple MDTs, performance can be improved beyond the limit of single MDT performance. Prior to the development of this feature all files in a directory must record their metadata on a single MDT.

This command to stripe a directory over *mdt_count* MDTs is:

```
client# lfs mkdir -c
mdt_count
/mount_point/new_directory
```

The striped directory feature is most useful for distributing single large directories (50k entries or more) across multiple MDTs, since it incurs more overhead than non-striped directories.

###  Directory creation by space/inode usage

If the starting MDT is not specified when creating a new directory, this directory and its stripes will be distributed on MDTs by space usage. For example the following will create a directory and its stripes on MDTs with balanced space usage:

```
lfs mkdir -c 2 <dir1>
```

Alternatively, if a default directory stripe is set on a directory, the subsequent syscall `mkdir` under will `<dir1>` have the same effect:

```
lfs setdirstripe -D -c 2 <dir1>
```

The policy is: 

• If free inodes/blocks on all MDT are almost the same, i.e. `max_inodes_avail * 84% < min_inodes_avail `and `max_blocks_avail * 84% < min_blocks_avail`, then choose MDT roundrobin. 

• Otherwise, create more subdirectories on MDTs with more free inodes/blocks.

## Setting and Retrieving Lustre Parameters

Several options are available for setting parameters in Lustre:

- When creating a file system, use mkfs.lustre. See *the section called “Setting Tunable Parameters with `mkfs.lustre`”*below.
- When a server is stopped, use tunefs.lustre. See *the section called “Setting Parameters with `tunefs.lustre`”* below.
- When the file system is running, use lctl to set or retrieve Lustre parameters. See *the section called “Setting Parameters with `lctl`”* and *the section called “Reporting Current Parameter Values”* below.

### Setting Tunable Parameters with `mkfs.lustre`

When the file system is first formatted, parameters can simply be added as a `--param` option to the `mkfs.lustre` command. For example:

```
mds# mkfs.lustre --mdt --param="sys.timeout=50" /dev/sda
```

For more details about creating a file system,see *Configuring a Lustre File System*. For more details about `mkfs.lustre`, see *System Configuration Utilities*.

### Setting Parameters with `tunefs.lustre`

If a server (OSS or MDS) is stopped, parameters can be added to an existing file system using the `--param` option to the `tunefs.lustre` command. For example:

```
oss# tunefs.lustre --param=failover.node=192.168.0.13@tcp0 /dev/sda
```

With `tunefs.lustre`, parameters are *additive*-- new parameters are specified in addition to old parameters, they do not replace them. To erase all old `tunefs.lustre` parameters and just use newly-specified parameters, run:

```
mds# tunefs.lustre --erase-params --param=
new_parameters 
```

The tunefs.lustre command can be used to set any parameter settable via `lctl conf_param` and that has its own OBD device, so it can be specified as `*obdname|fsname*. *obdtype*. *proc_file_name*= *value*`. For example:

```
mds# tunefs.lustre --param mdt.identity_upcall=NONE /dev/sda1
```

For more details about `tunefs.lustre`, see *System Configuration Utilities*.

### Setting Parameters with `lctl`

When the file system is running, the `lctl` command can be used to set parameters (temporary or permanent) and report current parameter values. Temporary parameters are active as long as the server or client is not shut down. Permanent parameters live through server and client reboots.

**Note**

The `lctl list_param` command enables users to list all parameters that can be set. See [the section called “Listing Parameters”](#listing-parameters).

For more details about the `lctl` command, see the examples in the sections below and [*System Configuration Utilities*](06.07-System%20Configuration%20Utilities.md).

#### Setting Temporary Parameters

Use `lctl set_param` to set temporary parameters on the node where it is run. These parameters map to items in `/proc/{fs,sys}/{lnet,lustre}`. The `lctl set_param` command uses this syntax:

```
lctl set_param [-n] [-P]
obdtype.
obdname.
proc_file_name=
value
```

For example:

```
# lctl set_param osc.*.max_dirty_mb=1024
osc.myth-OST0000-osc.max_dirty_mb=32
osc.myth-OST0001-osc.max_dirty_mb=32
osc.myth-OST0002-osc.max_dirty_mb=32
osc.myth-OST0003-osc.max_dirty_mb=32
osc.myth-OST0004-osc.max_dirty_mb=32
```

#### Setting Permanent Parameters

Use `lctl set_param -P` or `lctl conf_param` command to set permanent parameters. In general, the `lctl conf_param` command can be used to specify any parameter settable in a `/proc/fs/lustre` file, with its own OBD device. The `lctl conf_param`command uses this syntax (same as the `mkfs.lustre` and `tunefs.lustre` commands):

```
obdname|fsname.
obdtype.
proc_file_name=
value) 
```

Here are a few examples of `lctl conf_param` commands:

```
mgs# lctl conf_param testfs-MDT0000.sys.timeout=40
$ lctl conf_param testfs-MDT0000.mdt.identity_upcall=NONE
$ lctl conf_param testfs.llite.max_read_ahead_mb=16
$ lctl conf_param testfs-MDT0000.lov.stripesize=2M
$ lctl conf_param testfs-OST0000.osc.max_dirty_mb=29.15
$ lctl conf_param testfs-OST0000.ost.client_cache_seconds=15
$ lctl conf_param testfs.sys.timeout=40 
```

**Caution**

Parameters specified with the `lctl conf_param` command are set permanently in the file system's configuration file on the MGS.

Introduced in Lustre 2.5

#### Setting Permanent Parameters with lctl set_param -P

The `lctl set_param -P` command can also set parameters permanently. This command must be issued on the MGS. The given parameter is set on every host using `lctl` upcall. Parameters map to items in `/proc/{fs,sys}/{lnet,lustre}`. The `lctl set_param` command uses this syntax:

```
lctl set_param -P 
obdtype.
obdname.
proc_file_name=
value
```

For example:

```
# lctl set_param -P osc.*.max_dirty_mb=1024
osc.myth-OST0000-osc.max_dirty_mb=32
osc.myth-OST0001-osc.max_dirty_mb=32
osc.myth-OST0002-osc.max_dirty_mb=32
osc.myth-OST0003-osc.max_dirty_mb=32
osc.myth-OST0004-osc.max_dirty_mb=32 
```

Use `-d`(only with -P) option to delete permanent parameter. Syntax:

```
lctl set_param -P -d
obdtype.
obdname.
proc_file_name
```

For example:

```
# lctl set_param -P -d osc.*.max_dirty_mb 
```

#### Listing Parameters

To list Lustre or LNet parameters that are available to set, use the `lctl list_param` command. For example:

```
lctl list_param [-FR] 
obdtype.
obdname
```

The following arguments are available for the `lctl list_param` command.

`-F` Add ' `/`', ' `@`' or ' `=`' for directories, symlinks and writeable files, respectively

`-R` Recursively lists all parameters under the specified path

For example:

```
oss# lctl list_param obdfilter.lustre-OST0000 
```

#### Reporting Current Parameter Values

To report current Lustre parameter values, use the `lctl get_param` command with this syntax:

```
lctl get_param [-n] 
obdtype.
obdname.
proc_file_name
```

This example reports data on RPC service times.

```
oss# lctl get_param -n ost.*.ost_io.timeouts
service : cur 1 worst 30 (at 1257150393, 85d23h58m54s ago) 1 1 1 1 
```

This example reports the amount of space this client has reserved for writeback cache with each OST:

```
client# lctl get_param osc.*.cur_grant_bytes
osc.myth-OST0000-osc-ffff8800376bdc00.cur_grant_bytes=2097152
osc.myth-OST0001-osc-ffff8800376bdc00.cur_grant_bytes=33890304
osc.myth-OST0002-osc-ffff8800376bdc00.cur_grant_bytes=35418112
osc.myth-OST0003-osc-ffff8800376bdc00.cur_grant_bytes=2097152
osc.myth-OST0004-osc-ffff8800376bdc00.cur_grant_bytes=33808384
```

## Specifying NIDs and Failover

If a node has multiple network interfaces, it may have multiple NIDs, which must all be identified so other nodes can choose the NID that is appropriate for their network interfaces. Typically, NIDs are specified in a list delimited by commas ( `,`). However, when failover nodes are specified, the NIDs are delimited by a colon ( `:`) or by repeating a keyword such as `--mgsnode=` or `--servicenode=`).

To display the NIDs of all servers in networks configured to work with the Lustre file system, run (while LNet is running):

```
lctl list_nids
```

In the example below, `mds0` and `mds1` are configured as a combined MGS/MDT failover pair and `oss0` and `oss1` are configured as an OST failover pair. The Ethernet address for `mds0` is 192.168.10.1, and for `mds1` is 192.168.10.2. The Ethernet addresses for`oss0` and `oss1` are 192.168.10.20 and 192.168.10.21 respectively.

```
mds0# mkfs.lustre --fsname=testfs --mdt --mgs \
        --servicenode=192.168.10.2@tcp0 \
        -–servicenode=192.168.10.1@tcp0 /dev/sda1
mds0# mount -t lustre /dev/sda1 /mnt/test/mdt
oss0# mkfs.lustre --fsname=testfs --servicenode=192.168.10.20@tcp0 \
        --servicenode=192.168.10.21 --ost --index=0 \
        --mgsnode=192.168.10.1@tcp0 --mgsnode=192.168.10.2@tcp0 \
        /dev/sdb
oss0# mount -t lustre /dev/sdb /mnt/test/ost0
client# mount -t lustre 192.168.10.1@tcp0:192.168.10.2@tcp0:/testfs \
        /mnt/testfs
mds0# umount /mnt/mdt
mds1# mount -t lustre /dev/sda1 /mnt/test/mdt
mds1# lctl get_param mdt.testfs-MDT0000.recovery_status
```

Where multiple NIDs are specified separated by commas (for example, `10.67.73.200@tcp,192.168.10.1@tcp`), the two NIDs refer to the same host, and the Lustre software chooses the *best* one for communication. When a pair of NIDs is separated by a colon (for example, `10.67.73.200@tcp:10.67.73.201@tcp`), the two NIDs refer to two different hosts and are treated as a failover pair (the Lustre software tries the first one, and if that fails, it tries the second one.)

Two options to `mkfs.lustre` can be used to specify failover nodes. The `--servicenode` option is used to specify all service NIDs, including those for primary nodes and failover nodes. When the `--servicenode`option is used, the first service node to load the target device becomes the primary service node, while nodes corresponding to the other specified NIDs become failover locations for the target device. An older option, `--failnode`, specifies just the NIDS of failover nodes. For more information about the `--servicenode` and `--failnode` options, see *Configuring Failover in a Lustre File System*.

## Erasing a File System

If you want to erase a file system and permanently delete all the data in the file system, run this command on your targets:

```
$ "mkfs.lustre --reformat"
```

If you are using a separate MGS and want to keep other file systems defined on that MGS, then set the `writeconf` flag on the MDT for that file system. The `writeconf` flag causes the configuration logs to be erased; they are regenerated the next time the servers start.

To set the `writeconf` flag on the MDT:

1. Unmount all clients/servers using this file system, run:

   ```
   $ umount /mnt/lustre
   ```

2. Permanently erase the file system and, presumably, replace it with another file system, run:

   ```
   $ mkfs.lustre --reformat --fsname spfs --mgs --mdt --index=0 /dev/
   {mdsdev}
   ```

3. If you have a separate MGS (that you do not want to reformat), then add the `--writeconf` flag to `mkfs.lustre` on the MDT, run:

   ```
   $ mkfs.lustre --reformat --writeconf --fsname spfs --mgsnode=
   mgs_nid --mdt --index=0 
   /dev/mds_device
   ```

**Note**

If you have a combined MGS/MDT, reformatting the MDT reformats the MGS as well, causing all configuration information to be lost; you can start building your new file system. Nothing needs to be done with old disks that will not be part of the new file system, just do not mount them.

## Reclaiming Reserved Disk Space

All current Lustre installations run the ldiskfs file system internally on service nodes. By default, ldiskfs reserves 5% of the disk space to avoid file system fragmentation. In order to reclaim this space, run the following command on your OSS for each OST in the file system:

```
tune2fs [-m reserved_blocks_percent] /dev/
{ostdev}
```

You do not need to shut down Lustre before running this command or restart it afterwards.

**Warning**

Reducing the space reservation can cause severe performance degradation as the OST file system becomes more than 95% full, due to difficulty in locating large areas of contiguous free space. This performance degradation may persist even if the space usage drops below 95% again. It is recommended NOT to reduce the reserved disk space below 5%.

## Replacing an Existing OST or MDT

To copy the contents of an existing OST to a new OST (or an old MDT to a new MDT), follow the process for either OST/MDT backups in *the section called “ Backing Up and Restoring an MDT or OST (ldiskfs Device Level)”*or *the section called “ Backing Up an OST or MDT (Backend File System Level)”*. For more information on removing a MDT, see *the section called “Removing an MDT from the File System”*.

## Identifying To Which Lustre File an OST Object Belongs

Use this procedure to identify the file containing a given object on a given OST.

1. On the OST (as root), run `debugfs` to display the file identifier ( `FID`) of the file associated with the object.

   For example, if the object is `34976` on `/dev/lustre/ost_test2`, the debug command is:

   ```
   # debugfs -c -R "stat /O/0/d$((34976 % 32))/34976" /dev/lustre/ost_test2 
   ```

   The command output is:

   ```
   debugfs 1.45.6.wc1 (20-Mar-2020)
   /dev/lustre/ost_test2: catastrophic mode - not reading inode or group bitmaps
   Inode: 352365   Type: regular    Mode:  0666   Flags: 0x80000
   Generation: 2393149953    Version: 0x0000002a:00005f81
   User:  1000   Group:  1000   Size: 260096
   File ACL: 0    Directory ACL: 0
   Links: 1   Blockcount: 512
   Fragment:  Address: 0    Number: 0    Size: 0
   ctime: 0x4a216b48:00000000 -- Sat May 30 13:22:16 2009
   atime: 0x4a216b48:00000000 -- Sat May 30 13:22:16 2009
   mtime: 0x4a216b48:00000000 -- Sat May 30 13:22:16 2009
   crtime: 0x4a216b3c:975870dc -- Sat May 30 13:22:04 2009
   Size of extra inode fields: 24
   Extended attributes stored in inode body:
     fid = "b9 da 24 00 00 00 00 00 6a fa 0d 3f 01 00 00 00 eb 5b 0b 00 00 00 0000
   00 00 00 00 00 00 00 00 " (32)
     fid: objid=34976 seq=0 parent=[0x200000400:0x122:0x0] stripe=1
   EXTENTS:
   (0-64):4620544-4620607
   ```

2. The parent FID will be of the form [0x200000400:0x122:0x0] and can be resolved directly using the `lfs fid2path [0x200000404:0x122:0x0] /mnt/lustre` command on any Lustre client, and the process is complete.

3. In cases of an upgraded 1.x inode (if the first part of the FID is below 0x200000400), the MDT inode number is `0x24dab9` and generation `0x3f0dfa6a` and the pathname can also be resolved using `debugfs`.

4. On the MDS (as root), use `debugfs` to find the file associated with the inode:

   ```
   # debugfs -c -R "ncheck 0x24dab9" /dev/lustre/mdt_test 
   ```

   Here is the command output:

   ```
   debugfs 1.42.3.wc3 (15-Aug-2012)
   /dev/lustre/mdt_test: catastrophic mode - not reading inode or group bitmap\
   s
   Inode      Pathname
   2415289    /ROOT/brian-laptop-guest/clients/client11/~dmtmp/PWRPNT/ZD16.BMP
   ```

The command lists the inode and pathname associated with the object.

**Note**

`Debugfs`' ''ncheck'' is a brute-force search that may take a long time to complete.

**Note**

To find the Lustre file from a disk LBA, follow the steps listed in the document at this URL: <http://smartmontools.sourceforge.net/badblockhowto.html>. Then, follow the steps above to resolve the Lustre filename.
