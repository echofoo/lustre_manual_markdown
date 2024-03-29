# Configuring and Managing Quotas

- [Configuring and Managing Quotas](#configuring-and-managing-quotas)
  * [Working with Quotas](#working-with-quotas)
  * [Enabling Disk Quotas](#enabling-disk-quotas)
    + [Enabling Disk Quotas (Lustre Software Release 2.4 and later)](#enabling-disk-quotas-lustre-software-release-24-and-later)
      - [Quota Verification](#quota-verification)
  * [Quota Administration](#quota-administration)
  * [Default Quota](#Default Quota)
  * [Quota Allocation](#quota-allocation)
  * [Quotas and Version Interoperability](#quotas-and-version-interoperability)
  * [Granted Cache and Quota Limits](#granted-cache-and-quota-limits)
  * [Lustre Quota Statistics](#lustre-quota-statistics)
    + [Interpreting Quota Statistics](#interpreting-quota-statistics)
  * [Pool Quotas] (#Pool Quotas)

## Working with Quotas

Quotas allow a system administrator to limit the amount of disk space a user, group, or project can use. Quotas are set by root, and can be specified for individual users, groups, and/or projects. Before a file is written to a partition where quotas are set, the quota of the creator's group is checked. If a quota exists, then the file size counts towards the group's quota. If no quota exists, then the owner's user quota is checked before the file is written. Similarly, inode usage for specific functions can be controlled if a user over-uses the allocated space.

Lustre quota enforcement differs from standard Linux quota enforcement in several ways:

- Quotas are administered via the `lfs` and `lctl` commands (post-mount).

- The quota feature in Lustre software is distributed throughout the system (as the Lustre file system is a distributed file system). Because of this, quota setup and behavior on Lustre is somewhat different from local disk quotas in the following ways:
  - No single point of administration: some commands must be executed on the MGS, other commands on the MDSs and OSSs, and still other commands on the client.
  - Granularity: a local quota is typically specified for kilobyte resolution, Lustre uses one megabyte as the smallest quota resolution.
  - Accuracy: quota information is distributed throughout the file system and can only be accurately calculated with a quiescent file system in order to minimize performance overhead during normal use.
  
- Quotas are allocated and consumed in a quantized fashion.

- Client does not set the `usrquota` or `grpquota` options to mount. Space accounting is enabled by default and quota enforcement can be enabled/disabled on a per-filesystem basis with `lctl conf_param`.

   It is worth noting that both `lfs quotaon`,`lfs quotaoff`, `lfs quotacheck` and `quota_type` sub-commands are deprecated as of Lustre 2.4.0, and removed completely in Lustre 2.8.0.

**Caution**

Although a quota feature is available in the Lustre software, root quotas are NOT enforced.

`lfs setquota -u root` (limits are not enforced)

`lfs quota -u root` (usage includes internal Lustre data that is dynamic in size and does not accurately reflect mount point visible block and inode usage).

## Enabling Disk Quotas

The design of quotas on Lustre has management and enforcement separated from resource usage and accounting. Lustre software is responsible for management and enforcement. The back-end file system is responsible for resource usage and accounting. Because of this, it is necessary to begin enabling quotas by enabling quotas on the back-end disk system. 

**Caution**

Quota setup is orchestrated by the MGS and *all setup commands in this section must be run directly on the MGS*. Support for project quotas specifically requires Lustre Release 2.10 or later. A *patched server* may be required, depending on the kernel version and backend filesystem type:

| **Configuration**                             | **Patched Server Required?** |
| --------------------------------------------- | ---------------------------- |
| *ldiskfs with kernel version < 4.5*           | Yes                          |
| *ldiskfs with kernel version >= 4.5*          | No                           |
| *zfs version >=0.8 with kernel version < 4.5* | Yes                          |
| *zfs version >=0.8 with kernel version > 4.5* | No                           |

**Note**: Project quotas are not supported on zfs versions earlier than 0.8.

Once setup, verification of the quota state must be performed on the MDT. Although quota enforcement is managed by the Lustre software, each OSD implementation relies on the back-end file system to maintain per-user/group/project block and inode usage. Hence, differences exist when setting up quotas with ldiskfs or ZFS back-ends:

- For ldiskfs backends, `mkfs.lustre` now creates empty quota files and enables the QUOTA feature flag in the superblock which turns quota accounting on at mount time automatically. e2fsck was also modified to fix the quota files when the QUOTA feature flag is present. The project quota feature is disabled by default, and`tune2fs` needs to be run to enable every target manually. If user, group, and project quota usage is inconsistent, run `e2fsck -f` on all unmounted MDTs and OSTs.
- For ZFS backend, *the project quota feature is not supported on zfs versions less than 0.8.0.* Accounting ZAPs are created and maintained by the ZFS file system itself. While ZFS tracks per-user and group block usage, it does not handle inode accounting for ZFS versions prior to zfs-0.7.0. The ZFS OSD previously implemented its own support for inode tracking. Two options are available:
  1. The ZFS OSD can estimate the number of inodes in-use based on the number of blocks used by a given user or group. This mode can be enabled by running the following command on the server running the target: `lctl set_param osd-zfs.${FSNAME}-${TARGETNAME}.quota_iused_estimate=1`.
  2. Similarly to block accounting, dedicated ZAPs are also created the ZFS OSD to maintain per-user and group inode usage. This is the default mode which corresponds to `quota_iused_estimate` set to 0.

**Note**

To (re-)enable space usage quota on ldiskfs filesystems, run `tune2fs -O quota` against all targets. This command sets the QUOTA feature flag in the superblock and runs e2fsck internally. As a result, the target must be offline to build the per-UID/GID disk usage database.

Introduced in Lustre 2.10

Lustre filesystems formatted with a Lustre release prior to 2.10 can be still safely upgraded to release 2.10, but will not have project quota usage reporting functional until 2.15.0 or `tune2fs -O project` is run against all ldiskfs backend targets. This command sets the PROJECT feature flag in the superblock and runs e2fsck (as a result, the target must be offline). See *the section called “ Quotas and Version Interoperability”* for further important considerations.

**Caution**

Lustre requires a version of e2fsprogs that supports quota to be installed on the server nodes when using ldiskfs backend (e2fsprogs is not needed with ZFS backend). In general, we recommend to use the latest e2fsprogs version available on [http://downloads.whamcloud.com/public/e2fsprogs/](http://downloads.whamcloud.com/e2fsprogs/).

The ldiskfs OSD relies on the standard Linux quota to maintain accounting information on disk. As a consequence, the Linux kernel running on the Lustre servers using ldiskfs backend must have`CONFIG_QUOTA`, `CONFIG_QUOTACTL` and `CONFIG_QFMT_V2` enabled.

Quota enforcement is turned on/off independently of space accounting which is always enabled. There is a single per-file system quota parameter controlling inode/block quota enforcement. Like all permanent parameters, this quota parameter can be set via `lctl conf_param` on the MGS via the following syntax:

```
lctl conf_param fsname.quota.ost|mdt=u|g|p|ugp|none
```

- `ost` -- to configure block quota managed by OSTs
- `mdt` -- to configure inode quota managed by MDTs
- `u` -- to enable quota enforcement for users only
- `g` -- to enable quota enforcement for groups only
- `p` -- to enable quota enforcement for projects only
- `ugp` -- to enable quota enforcement for all users, groups and projects
- `none` -- to disable quota enforcement for all users, groups and projects

Examples:

To turn on user, group, and project quotas for block only on file system `testfs1`, *on the MGS* run:

```
mgs# lctl conf_param testfs1.quota.ost=ugp 
```

To turn on group quotas for inodes on file system `testfs2`, on the MGS run:

```
mgs# lctl conf_param testfs2.quota.mdt=g 
```

To turn off user, group, and project quotas for both inode and block on file system `testfs3`, on the MGS run:

```
mgs# lctl conf_param testfs3.quota.ost=none
mgs# lctl conf_param testfs3.quota.mdt=none
```
#### Quota Verification

Once the quota parameters have been configured, all targets which are part of the file system will be automatically notified of the new quota settings and enable/disable quota enforcement as needed. The per-target enforcement status can still be verified by running the following *command on the MDS(s)*:

```
$ lctl get_param osd-*.*.quota_slave.info
osd-zfs.testfs-MDT0000.quota_slave.info=
target name:    testfs-MDT0000
pool ID:        0
type:           md
quota enabled:  ug
conn to master: setup
user uptodate:  glb[1],slv[1],reint[0]
group uptodate: glb[1],slv[1],reint[0]
```

## Quota Administration

Once the file system is up and running, quota limits on blocks and inodes can be set for user, group, and project. This is *controlled entirely from a client* via three quota parameters:

**Grace period**-- The period of time (in seconds) within which users are allowed to exceed their soft limit. There are six types of grace periods:

- user block soft limit
- user inode soft limit
- group block soft limit
- group inode soft limit
- project block soft limit
- project inode soft limit

The grace period applies to all users. The user block soft limit is for all users who are using a blocks quota.

**Soft limit** -- The grace timer is started once the soft limit is exceeded. At this point, the user/group/project can still allocate block/inode. When the grace time expires and if the user is still above the soft limit, the soft limit becomes a hard limit and the user/group/project can't allocate any new block/inode any more. The user/group/project should then delete files to be under the soft limit. The soft limit MUST be smaller than the hard limit. If the soft limit is not needed, it should be set to zero (0).

**Hard limit** -- Block or inode allocation will fail with `EDQUOT`(i.e. quota exceeded) when the hard limit is reached. The hard limit is the absolute limit. When a grace period is set, one can exceed the soft limit within the grace period if under the hard limit.

Due to the distributed nature of a Lustre file system and the need to maintain performance under load, those quota parameters may not be 100% accurate. The quota settings can be manipulated via the `lfs` command, executed on a client, and includes several options to work with quotas:

- `quota` -- displays general quota information (disk usage and limits)
- `setquota` -- specifies quota limits and tunes the grace period. By default, the grace period is one week.

Usage:

```
lfs quota [-q] [-v] [-h] [-o obd_uuid] [-u|-g|-p uname|uid|gname|gid|projid] /mount_point
lfs quota -t {-u|-g|-p} /mount_point
lfs setquota {-u|--user|-g|--group|-p|--project} username|groupname [-b block-softlimit] \
             [-B block_hardlimit] [-i inode_softlimit] \
             [-I inode_hardlimit] /mount_point
```

To display general quota information (disk usage and limits) for the user running the command and his primary group, run:

```
$ lfs quota /mnt/testfs
```

To display general quota information for a specific user (" `bob`" in this example), run:

```
$ lfs quota -u bob /mnt/testfs
```

To display general quota information for a specific user (" `bob`" in this example) and detailed quota statistics for each MDT and OST, run:

```
$ lfs quota -u bob -v /mnt/testfs
```

To display general quota information for a specific project (" `1`" in this example), run:

```
$ lfs quota -p 1 /mnt/testfs
```

To display general quota information for a specific group (" `eng`" in this example), run:

```
$ lfs quota -g eng /mnt/testfs
```

To limit quota usage for a specific project ID on a specific directory ("`/mnt/testfs/dir`" in this example), run:

```
$ lfs project -s -p 1 -r /mnt/testfs/dir
$ lfs setquota -p 1 -b 307200 -B 309200 -i 10000 -I 11000 /mnt/testfs
```

Recursively list all descendants'(of the directory) project attribute on directory ("/mnt/testfs/dir" in this example), run:

```
$ lfs project -r /mnt/testfs/dir
```

Please note that if it is desired to have lfs quota -p show the space/inode usage under the directory properly (much faster than du), then the user/admin needs to use different project IDs for different directories.

To display block and inode grace times for user quotas, run:

```
$ lfs quota -t -u /mnt/testfs
```

To set user or group quotas for a specific ID ("bob" in this example), run:

```
$ lfs setquota -u bob -b 307200 -B 309200 -i 10000 -I 11000 /mnt/testfs
```

In this example, the quota for user "bob" is set to 300 MB (309200*1024) and the hard limit is 11,000 files. Therefore, the inode hard limit should be 11000.

The quota command displays the quota allocated and consumed by each Lustre target. Using the previous `setquota`example, running this `lfs` quota command:

```
$ lfs quota -u bob -v /mnt/testfs
```

displays this command output:

```
Disk quotas for user bob (uid 6000):
Filesystem          kbytes quota limit grace files quota limit grace
/mnt/testfs         0      30720 30920 -     0     10000 11000 -
testfs-MDT0000_UUID 0      -      8192 -     0     -     2560  -
testfs-OST0000_UUID 0      -      8192 -     0     -     0     -
testfs-OST0001_UUID 0      -      8192 -     0     -     0     -
Total allocated inode limit: 2560, total allocated block limit: 24576
```

Global quota limits are stored in dedicated index files (there is one such index per quota type) on the quota master target (aka QMT). The QMT runs on MDT0000 and exports the global indices via *lctl get_param*. The global indices can thus be dumped via the following command:

```
# lctl get_param qmt.testfs-QMT0000.*.glb-*
```

The format of global indexes depends on the OSD type. The ldiskfs OSD uses an IAM files while the ZFS OSD creates dedicated ZAPs.

Each slave also stores a copy of this global index locally. When the global index is modified on the master, a glimpse callback is issued on the global quota lock to notify all slaves that the global index has been modified. This glimpse callback includes information about the identifier subject to the change. If the global index on the QMT is modified while a slave is disconnected, the index version is used to determine whether the slave copy of the global index isn't up to date any more. If so, the slave fetches the whole index again and updates the local copy. The slave copy of the global index can also be accessed via the following command:

```
lctl get_param osd-*.*.quota_slave.limit*
```

## Default Quota

The default quota is used to enforce the quota limits for any user, group, or project that do not have quotas set by administrator. The default quota can be disabled by setting limits to 0.

### Usage

```
lfs quota [-U|--default-usr|-G|--default-grp|-P|--default-prj] /mount_point
lfs setquota {-U|--default-usr|-G|--default-grp|-P|--default-prj} [-b block-softlimit] \
 [-B block_hardlimit] [-i inode_softlimit] [-I inode_hardlimit] /mount_point
lfs setquota {-u|-g|-p} username|groupname -d /mount_point
```

To set the default user quota: 

```
# lfs setquota -U -b 10G -B 11G -i 100K -I 105K /mnt/testfs 
```

To set the default group quota: 
```
# lfs setquota -G -b 10G -B 11G -i 100K -I 105K /mnt/testfs 
```
To set the default project quota: 
```
# lfs setquota -P -b 10G -B 11G -i 100K -I 105K /mnt/testfs
```
To disable the default user quota: 
```
# lfs setquota -U -b 0 -B 0 -i 0 -I 0 /mnt/testfs
```
To disable the default group quota: 
```
# lfs setquota -G -b 0 -B 0 -i 0 -I 0 /mnt/testfs
```
To disable the default project quota: 
```
# lfs setquota -P -b 0 -B 0 -i 0 -I 0 /mnt/testfs
```
**Note**
If quota limits are set for some user, group or project, it will use those specific quota limits instead of the default quota. Quota limits for any user, group or project will use the default quota by setting its quota limits to 0.

## Quota Allocation

In a Lustre file system, quota must be properly allocated or users may experience unnecessary failures. The file system block quota is divided up among the OSTs within the file system. Each OST requests an allocation which is increased up to the quota limit. The quota allocation is then quantized to reduce the number of quota-related request traffic.

The Lustre quota system distributes quotas from the Quota Master Target (aka QMT). Only one QMT instance is supported for now and only runs on the same node as MDT0000. All OSTs and MDTs set up a Quota Slave Device (aka QSD) which connects to the QMT to allocate/release quota space. The QSD is setup directly from the OSD layer.

To reduce quota requests, quota space is initially allocated to QSDs in very large chunks. How much unused quota space can be held by a target is controlled by the qunit size. When quota space for a given ID is close to exhaustion on the QMT, the qunit size is reduced and QSDs are notified of the new qunit size value via a glimpse callback. Slaves are then responsible for releasing quota space above the new qunit value. The qunit size isn't shrunk indefinitely and there is a minimal value of 1MB for blocks and 1,024 for inodes. This means that the quota space rebalancing process will stop when this minimum value is reached. As a result, quota exceeded can be returned while many slaves still have 1MB or 1,024 inodes of spare quota space.

If we look at the `setquota` example again, running this `lfs quota` command:

```
# lfs quota -u bob -v /mnt/testfs
```

displays this command output:

```
Disk quotas for user bob (uid 500):
Filesystem          kbytes quota limit grace       files  quota limit grace
/mnt/testfs         30720* 30720 30920 6d23h56m44s 10101* 10000 11000
6d23h59m50s
testfs-MDT0000_UUID 0      -     0     -           10101  -     10240
testfs-OST0000_UUID 0      -     1024  -           -      -     -
testfs-OST0001_UUID 30720* -     29896 -           -      -     -
Total allocated inode limit: 10240, total allocated block limit: 30920
```

The total quota limit of 30,920 is allocated to user bob, which is further distributed to two OSTs.

Values appended with ' `*`' show that the quota limit has been exceeded, causing the following error when trying to write or create a file:

```
$ cp: writing `/mnt/testfs/foo`: Disk quota exceeded.
```

**Note**

It is very important to note that the block quota is consumed per OST and the inode quota per MDS. Therefore, when the quota is consumed on one OST (resp. MDT), the client may not be able to create files regardless of the quota available on other OSTs (resp. MDTs).

Setting the quota limit below the minimal qunit size may prevent the user/group from all file creation. It is thus recommended to use soft/hard limits which are a multiple of the number of OSTs * the minimal qunit size.

To determine the total number of inodes, use `lfs df -i`(and also `lctl get_param *.*.filestotal`). For more information on using the `lfs df -i` command and the command output, see *the section called “Checking File System Free Space”*.

Unfortunately, the `statfs` interface does not report the free inode count directly, but instead reports the total inode and used inode counts. The free inode count is calculated for `df` from (total inodes - used inodes). It is not critical to know the total inode count for a file system. Instead, you should know (accurately), the free inode count and the used inode count for a file system. The Lustre software manipulates the total inode count in order to accurately report the other two values.

## Quotas and Version Interoperability

Introduced in Lustre 2.10

To use the project quota functionality introduced in Lustre 2.10, **all Lustre servers and clients must be upgraded to Lustre release 2.10 or later for project quota to work correctly**. Otherwise, project quota will be inaccessible on clients and not be accounted for on OSTs. Furthermore, the **servers may be required to use a patched kernel,** for more information see *the section called “Enabling Disk Quotas (Lustre Software Release 2.4 and later)”*

Introduced in Lustre 2.14

`df` and `lfs df` will return the amount of space available to that project rather than the total filesystem space, if the project quota limit is smaller. **Only client need be upgraded to Lustre release 2.14 or later to apply this new behavior.**

## Granted Cache and Quota Limits

In a Lustre file system, granted cache does not respect quota limits. In this situation, OSTs grant cache to a Lustre client to accelerate I/O. Granting cache causes writes to be successful in OSTs, even if they exceed the quota limits, and will overwrite them.

The sequence is:

1. A user writes files to the Lustre file system.
2. If the Lustre client has enough granted cache, then it returns 'success' to users and arranges the writes to the OSTs.
3. Because Lustre clients have delivered success to users, the OSTs cannot fail these writes.

Because of granted cache, writes always overwrite quota limitations. For example, if you set a 400 GB quota on user A and use IOR to write for user A from a bundle of clients, you will write much more data than 400 GB, and cause an out-of-quota error ( `EDQUOT`).

**Note**

The effect of granted cache on quota limits can be mitigated, but not eradicated. Reduce the maximum amount of dirty data on the clients (minimal value is 1MB):

- `lctl set_param osc.*.max_dirty_mb=8`

## Lustre Quota Statistics

The Lustre software includes statistics that monitor quota activity, such as the kinds of quota RPCs sent during a specific period, the average time to complete the RPCs, etc. These statistics are useful to measure performance of a Lustre file system.

Each quota statistic consists of a quota event and `min_time`, `max_time` and `sum_time` values for the event.

| **Quota Event**                                              | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **sync_acq_req**                                             | Quota slaves send a acquiring_quota request and wait for its return. |
| **sync_rel_req**                                             | Quota slaves send a releasing_quota request and wait for its return. |
| **async_acq_req**                                            | Quota slaves send an acquiring_quota request and do not wait for its return. |
| **async_rel_req**                                            | Quota slaves send a releasing_quota request and do not wait for its return. |
| **wait_for_blk_quota (lquota_chkquota)**                     | Before data is written to OSTs, the OSTs check if the remaining block quota is sufficient. This is done in the lquota_chkquota function. |
| **wait_for_ino_quota (lquota_chkquota)**                     | Before files are created on the MDS, the MDS checks if the remaining inode quota is sufficient. This is done in the lquota_chkquota function. |
| **wait_for_blk_quota (lquota_pending_commit)**               | After blocks are written to OSTs, relative quota information is updated. This is done in the lquota_pending_commit function. |
| **wait_for_ino_quota (lquota_pending_commit)**               | After files are created, relative quota information is updated. This is done in the lquota_pending_commit function. |
| **wait_for_pending_blk_quota_req (qctxt_wait_pending_dqacq)** | On the MDS or OSTs, there is one thread sending a quota request for a specific UID/GID for block quota at any time. At that time, if other threads need to do this too, they should wait. This is done in the qctxt_wait_pending_dqacq function. |
| **wait_for_pending_ino_quota_req (qctxt_wait_pending_dqacq)** | On the MDS, there is one thread sending a quota request for a specific UID/GID for inode quota at any time. If other threads need to do this too, they should wait. This is done in the qctxt_wait_pending_dqacq function. |
| **nowait_for_pending_blk_quota_req (qctxt_wait_pending_dqacq)** | On the MDS or OSTs, there is one thread sending a quota request for a specific UID/GID for block quota at any time. When threads enter qctxt_wait_pending_dqacq, they do not need to wait. This is done in the qctxt_wait_pending_dqacq function. |
| **nowait_for_pending_ino_quota_req (qctxt_wait_pending_dqacq)** | On the MDS, there is one thread sending a quota request for a specific UID/GID for inode quota at any time. When threads enter qctxt_wait_pending_dqacq, they do not need to wait. This is done in the qctxt_wait_pending_dqacq function. |
| **quota_ctl**                                                | The quota_ctl statistic is generated when lfs `setquota`, `lfs quota` and so on, are issued. |
| **adjust_qunit**                                             | Each time qunit is adjusted, it is counted.                  |

### Interpreting Quota Statistics

Quota statistics are an important measure of the performance of a Lustre file system. Interpreting these statistics correctly can help you diagnose problems with quotas, and may indicate adjustments to improve system performance.

For example, if you run this command on the OSTs:

```
lctl get_param lquota.testfs-OST0000.stats
```

You will get a result similar to this:

```
snapshot_time                                1219908615.506895 secs.usecs
async_acq_req                              1 samples [us]  32 32 32
async_rel_req                              1 samples [us]  5 5 5
nowait_for_pending_blk_quota_req(qctxt_wait_pending_dqacq) 1 samples [us] 2\
 2 2
quota_ctl                          4 samples [us]  80 3470 4293
adjust_qunit                               1 samples [us]  70 70 70
....
```

In the first line, `snapshot_time` indicates when the statistics were taken. The remaining lines list the quota events and their associated data.

In the second line, the `async_acq_req` event occurs one time. The `min_time`, `max_time` and `sum_time` statistics for this event are 32, 32 and 32, respectively. The unit is microseconds (μs).

In the fifth line, the quota_ctl event occurs four times. The `min_time`, `max_time` and `sum_time` statistics for this event are 80, 3470 and 4293, respectively. The unit is microseconds (μs).

## Pool Quotas

OST Pool Quotas feature gives an ability to limit user's (group's/project's) disk usage at OST pool level. Each OST Pool Quota (PQ) maps directly to the OST pool of the same name. Thus PQ could be tuned with standard lctl pool_new/add/remove/erase commands. All PQ are subset of a global pool that includes all OSTs and MDTs (DOM case). It may be initially confusing to be prevented from using "all of" one quota due to a different quota setting. In Lustre, a quota is a limit, not a right to use an amount. You don't always get to use your quota - an OST may be out of space, or some other quota is limiting. For example, if there is an inode quota and a space quota, and you hit your inode limit while you still have plenty of space, you can't use the space. For another example, quotas may easily be overallocated: everyone gets 10PB of quota, in a 15PB system. That does not give them the right to use 10PB, it means they cannot use more than 10PB. They may very well get ENOSPC long before that - but they will not get EDQUOT. This behavior already exists in Lustre today, but pool quotas increase the number of limits in play: user, group or project global space quota and now all of those limits can also be defined for each pool. In all cases, the net effect is that the actual amount of space you can use is limited to the smallest (min) quota out of everything that is applicable. See more details in OST Pool Quotas HLD [http://wiki.lustre.org/OST_Pool_Quotas_HLD]

### DOM and MDT pools

From Quota Master point of view, "data" MDTs are regular members together with OSTs. However Pool Quotas support only OSTs as there is currently no mechanism to group MDTs in pools.

### Lfs quota/setquota options to setup quota pools

The same long option `--pool` is used to setup and report Pool Quotas with `lfs setquota` and `lfs setquota`.

`lfs setquota --pool` pool_name is used to set the block and soft usage limit for the user, group, or project for the specified pool name. 

`lfs quota --pool` pool_name shows the user, group, or project usage for the specified pool name.

### Quota pools interoperability

Both client and server should have at least Lustre 2.14 to support Pool Quotas.

**Note**

Pool Quotas may be able to work with older clients if server supports Pool Quotas. Pool quotas cannot be viewed or modified by older clients. Since the quota enforcement is done on the servers, only a single client is needed to configure the quotas. This could be done by mounting a client directly on the MDS if needed.

###  Pool Quotas Hard Limit setup example

Let's imagine you need to setup quota usage for already existed OST pool flash_pool:

```
# it is a limit for global pool. PQ don't work properly without that
lfs setquota -u ivan -B100T /mnt/testfs
# set 1TiB block hard limit for ivan in a flash_pool
lfs setquota -u ivan --pool flash_pool -B1T /mnt/testfs
```

**Note**

System-side hard limit is required before setting Quota Pool limit. If you do not need to limit user at all OSTs and MDTs at system, only per pool, it is recommended to set some unrealistic big hard limit. Without a global limit in place the Quota Pool limit will not be enforced. No matter hard or soft global limit - at least one of them should be set.

### Pool Quotas Soft Limit setup example

```
# notify OSTs to enforce quota for ivan
lfs setquota -u ivan -B10T /mnt/testfs
# soft limit 10MiB for ivan in a pool flash_pool
lfs setquota -u ivan --pool flash_pool -b1T /mnt/testfs
# set block grace 600 s for all users at flash_pool
lfs setquota -t -u --block-grace 600 --pool flash_pool /mnt/testfs
```



