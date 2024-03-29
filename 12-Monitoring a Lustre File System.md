# Monitoring a Lustre File System

This chapter provides information on monitoring a Lustre file system and includes the following sections:

- [Monitoring a Lustre File System](#monitoring-a-lustre-file-system)
  * [Lustre Changelogs](#lustre-changelogs)
    + [Working with Changelogs](#working-with-changelogs)
      - [`lctl changelog_register`](#lctl-changelog_register)
      - [`lfs changelog`](#lfs-changelog)
      - [`lfs changelog_clear`](#lfs-changelog_clear)
      - [`lctl changelog_deregister`](#lctl-changelog_deregister)
    + [Changelog Examples](#changelog-examples)
      - [Registering a Changelog User](#registering-a-changelog-user)
      - [Displaying Changelog Records](#displaying-changelog-records)
      - [Clearing Changelog Records](#clearing-changelog-records)
      - [Deregistering a Changelog User](#deregistering-a-changelog-user)
      - [Displaying the Changelog Index and Registered Users](#displaying-the-changelog-index-and-registered-users)
      - [Displaying the Changelog Mask](#displaying-the-changelog-mask)
      - [Setting the Changelog Mask](#setting-the-changelog-mask)
    + [Audit with Changelogs](#audit-with-changelogs)
      - [Enabling Audit](#enabling-audit)
      - [Audit examples](#audit-examples)
        * [`OPEN`](#open)
        * [`GETXATTR`](#getxattr)
        * [`SETXATTR`](#setxattr)
        * [`DENIED OPEN`](#denied-open)
  * [Lustre Jobstats](#lustre-jobstats)
    + [How Jobstats Works](#how-jobstats-works)
    + [Enable/Disable Jobstats](#enabledisable-jobstats)
    + [Check Job Stats](#check-job-stats)
    + [Clear Job Stats](#clear-job-stats)
    + [Configure Auto-cleanup Interval](#configure-auto-cleanup-interval)
  * [Lustre Monitoring Tool (LMT)](#lustre-monitoring-tool-lmt)
  * [`CollectL`](#collectl)
  * [Other Monitoring Options](#other-monitoring-options)


## Lustre Changelogs

The changelogs feature records events that change the file system namespace or file metadata. Changes such as file creation, deletion, renaming, attribute changes, etc. are recorded with the target and parent file identifiers (FIDs), the name of the target, a timestamp, and user information. These records can be used for a variety of purposes:

- Capture recent changes to feed into an archiving system.
- Use changelog entries to exactly replicate changes in a file system mirror.
- Set up "watch scripts" that take action on certain events or directories.
- Audit activity on Lustre, thanks to user information associated to file/directory changes with timestamps.

Changelogs record types are:

| **Value** | **Description**                                |
| --------- | ---------------------------------------------- |
| MARK      | Internal recordkeeping                         |
| CREAT     | Regular file creation                          |
| MKDIR     | Directory creation                             |
| HLINK     | Hard link                                      |
| SLINK     | Soft link                                      |
| MKNOD     | Other file creation                            |
| UNLNK     | Regular file removal                           |
| RMDIR     | Directory removal                              |
| RENME     | Rename, original                               |
| RNMTO     | Rename, final                                  |
| OPEN *    | Open                                           |
| CLOSE     | Close                                          |
| LYOUT     | Layout change                                  |
| TRUNC     | Regular file truncated                         |
| SATTR     | Attribute change                               |
| XATTR     | Extended attribute change (setxattr)           |
| HSM       | HSM specific event                             |
| MTIME     | MTIME change                                   |
| CTIME     | CTIME change                                   |
| ATIME *   | ATIME change                                   |
| MIGRT     | Migration event                                |
| FLRW      | File Level Replication: file initially written |
| RESYNC    | File Level Replication: file re-synced         |
| GXATR *   | Extended attribute access (getxattr)           |
| NOPEN *   | Denied open                                    |

**Note**

Event types marked with * are not recorded by default. Refer to *the section called “Setting the Changelog Mask”* for instructions on modifying the Changelogs mask.

FID-to-full-pathname and pathname-to-FID functions are also included to map target and parent FIDs into the file system namespace.



### Working with Changelogs

Several commands are available to work with changelogs.

#### `lctl changelog_register`

Because changelog records take up space on the MDT, the system administration must register changelog users. As soon as a changelog user is registered, the Changelogs feature is enabled. The registrants specify which records they are "done with", and the system purges up to the greatest common record.

To register a new changelog user, run:

```
mds# lctl --device fsname-MDTnumber changelog_register
```

Changelog entries are not purged beyond a registered user's set point (see `lfs changelog_clear`).



#### `lfs changelog`

To display the metadata changes on an MDT (the changelog records), run:

```
lfs changelog fsname-MDTnumber [startrec [endrec]] 
```

It is optional whether to specify the start and end records.

These are sample changelog records:

```
1 02MKDIR 15:15:21.977666834 2018.01.09 0x0 t=[0x200000402:0x1:0x0] j=mkdir.500 ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x1:0x0] pics
2 01CREAT 15:15:36.687592024 2018.01.09 0x0 t=[0x200000402:0x2:0x0] j=cp.500 ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000402:0x1:0x0] chloe.jpg
3 06UNLNK 15:15:41.305116815 2018.01.09 0x1 t=[0x200000402:0x2:0x0] j=rm.500 ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000402:0x1:0x0] chloe.jpg
4 07RMDIR 15:15:46.468790091 2018.01.09 0x1 t=[0x200000402:0x1:0x0] j=rmdir.500 ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x1:0x0] pics 
```



#### `lfs changelog_clear`

To clear old changelog records for a specific user (records that the user no longer needs), run:

```
lfs changelog_clear mdt_name userid endrec
```

The `changelog_clear` command indicates that changelog records previous to *endrec* are no longer of interest to a particular user *userid*, potentially allowing the MDT to free up disk space. An `*endrec*` value of 0 indicates the current last record. To run`changelog_clear`, the changelog user must be registered on the MDT node using `lctl`.

When all changelog users are done with records < X, the records are deleted.



#### `lctl changelog_deregister`

To deregister (unregister) a changelog user, run:

```
mds# lctl --device mdt_device changelog_deregister userid       
```

`changelog_deregister cl1` effectively does a `lfs changelog_clear cl1 0` as it deregisters.



### Changelog Examples

This section provides examples of different changelog commands.

#### Registering a Changelog User

To register a new changelog user for a device (`lustre-MDT0000`):

```
mds# lctl --device lustre-MDT0000 changelog_register
lustre-MDT0000: Registered changelog userid 'cl1'
```

#### Displaying Changelog Records

To display changelog records on an MDT (`lustre-MDT0000`):

```
$ lfs changelog lustre-MDT0000
1 02MKDIR 15:15:21.977666834 2018.01.09 0x0 t=[0x200000402:0x1:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x1:0x0] pics
2 01CREAT 15:15:36.687592024 2018.01.09 0x0 t=[0x200000402:0x2:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000402:0x1:0x0] chloe.jpg
3 06UNLNK 15:15:41.305116815 2018.01.09 0x1 t=[0x200000402:0x2:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000402:0x1:0x0] chloe.jpg
4 07RMDIR 15:15:46.468790091 2018.01.09 0x1 t=[0x200000402:0x1:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x1:0x0] pics
```

Changelog records include this information:

```
rec# 
operation_type(numerical/text) 
timestamp 
datestamp 
flags 
t=target_FID 
ef=extended_flags
u=uid:gid
nid=client_NID
p=parent_FID 
target_name
```

Displayed in this format:

```
rec# operation_type(numerical/text) timestamp datestamp flags t=target_FID \
ef=extended_flags u=uid:gid nid=client_NID p=parent_FID target_name
```

For example:

```
2 01CREAT 15:15:36.687592024 2018.01.09 0x0 t=[0x200000402:0x2:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000402:0x1:0x0] chloe.jpg
```

#### Clearing Changelog Records

To notify a device that a specific user (`cl1`) no longer needs records (up to and including 3):

```
$ lfs changelog_clear  lustre-MDT0000 cl1 3
```

To confirm that the `changelog_clear` operation was successful, run `lfs changelog`; only records after id-3 are listed:

```
$ lfs changelog lustre-MDT0000
4 07RMDIR 15:15:46.468790091 2018.01.09 0x1 t=[0x200000402:0x1:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x1:0x0] pics
```

#### Deregistering a Changelog User

To deregister a changelog user (`cl1`) for a specific device (`lustre-MDT0000`):

```
mds# lctl --device lustre-MDT0000 changelog_deregister cl1
lustre-MDT0000: Deregistered changelog user 'cl1'
```

The deregistration operation clears all changelog records for the specified user (`cl1`).

```
$ lfs changelog lustre-MDT0000
5 00MARK  15:56:39.603643887 2018.01.09 0x0 t=[0x20001:0x0:0x0] ef=0xf \
u=500:500 nid=0@<0:0> p=[0:0x50:0xb] mdd_obd-lustre-MDT0000-0
```

** Note **

MARK records typically indicate changelog recording status changes.

#### Displaying the Changelog Index and Registered Users

To display the current, maximum changelog index and registered changelog users for a specific device (`lustre-MDT0000`):

```
mds# lctl get_param  mdd.lustre-MDT0000.changelog_users 
mdd.lustre-MDT0000.changelog_users=current index: 8 
ID    index (idle seconds)
cl2   8 (180)
```

#### Displaying the Changelog Mask

To show the current changelog mask on a specific device (`lustre-MDT0000`):

```
mds# lctl get_param  mdd.lustre-MDT0000.changelog_mask 

mdd.lustre-MDT0000.changelog_mask= 
MARK CREAT MKDIR HLINK SLINK MKNOD UNLNK RMDIR RENME RNMTO CLOSE LYOUT \
TRUNC SATTR XATTR HSM MTIME CTIME MIGRT
```

#### Setting the Changelog Mask

To set the current changelog mask on a specific device (`lustre-MDT0000`):

```
mds# lctl set_param mdd.lustre-MDT0000.changelog_mask=HLINK 
mdd.lustre-MDT0000.changelog_mask=HLINK 
$ lfs changelog_clear lustre-MDT0000 cl1 0 
$ mkdir /mnt/lustre/mydir/foo
$ cp /etc/hosts /mnt/lustre/mydir/foo/file
$ ln /mnt/lustre/mydir/foo/file /mnt/lustre/mydir/myhardlink
```

Only item types that are in the mask show up in the changelog.

```
$ lfs changelog lustre-MDT0000
9 03HLINK 16:06:35.291636498 2018.01.09 0x0 t=[0x200000402:0x4:0x0] ef=0xf \
u=500:500 nid=10.128.11.159@tcp p=[0x200000007:0x3:0x0] myhardlink
```

### Audit with Changelogs

A specific use case for Lustre Changelogs is audit. According to a definition found on [Wikipedia](https://en.wikipedia.org/wiki/Information_technology_audit), information technology audits are used to evaluate the organization's ability to protect its information assets and to properly dispense information to authorized parties. Basically, audit consists in controlling that all data accesses made were done according to the access control policy in place. And usually, this is done by analyzing access logs.

Audit can be used as a proof of security in place. But Audit can also be a requirement to comply with regulations.

Lustre Changelogs are a good mechanism for audit, because this is a centralized facility, and it is designed to be transactional. Changelog records contain all information necessary for auditing purposes:

- ability to identify object of action thanks to file identifiers (FIDs) and name of targets
- ability to identify subject of action thanks to UID/GID and NID information
- ability to identify time of action thanks to timestamp

#### Enabling Audit

To have a fully functional Changelogs-based audit facility, some additional Changelog record types must be enabled, to be able to record events such as OPEN, ATIME, GETXATTR and DENIED OPEN. Please note that enabling these record types may have some performance impact. For instance, recording OPEN and GETXATTR events generate writes in the Changelog records for a read operation from a file-system standpoint.

Being able to record events such as OPEN or DENIED OPEN is important from an audit perspective. For instance, if Lustre file system is used to store medical records on a system dedicated to Life Sciences, data privacy is crucial. Administrators may need to know which doctors accessed, or tried to access, a given medical record and when. And conversely, they might need to know which medical records a given doctor accessed.

To enable all changelog entry types, do:

```
mds# lctl set_param mdd.lustre-MDT0000.changelog_mask=ALL
mdd.seb-MDT0000.changelog_mask=ALL
```

Once all required record types have been enabled, just register a Changelogs user and the audit facility is operational.

Note that, however, it is possible to control which Lustre client nodes can trigger the recording of file system access events to the Changelogs, thanks to the `audit_mode` flag on nodemap entries. The reason to disable audit on a per-nodemap basis is to prevent some nodes (e.g. backup, HSM agent nodes) from flooding the audit logs. When `audit_mode` flag is set to 1 on a nodemap entry, a client pertaining to this nodemap will be able to record file system access events to the Changelogs, if Changelogs are otherwise activated. When set to 0, events are not logged into the Changelogs, no matter if Changelogs are activated or not. By default, `audit_mode` flag is set to 1 in newly created nodemap entries. And it is also set to 1 in 'default' nodemap.

To prevent nodes pertaining to a nodemap to generate Changelog entries, do:

```
mgs# lctl nodemap_modify --name nm1 --property audit_mode --value 0
```

#### Audit examples

##### `OPEN`

An OPEN changelog entry is in the form:

```
7 10OPEN  13:38:51.510728296 2017.07.25 0x242 t=[0x200000401:0x2:0x0] \
ef=0x7 u=500:500 nid=10.128.11.159@tcp m=-w-
```

It includes information about the open mode, in the form m=rwx.

OPEN entries are recorded only once per UID/GID, for a given open mode, as long as the file is not closed by this UID/GID. It avoids flooding the Changelogs for instance if there is an MPI job opening the same file thousands of times from different threads. It reduces the ChangeLog load significantly, without significantly affecting the audit information. Similarly, only the last CLOSE per UID/GID is recorded.

##### `GETXATTR`

A GETXATTR changelog entry is in the form:

```
8 23GXATR 09:22:55.886793012 2017.07.27 0x0 t=[0x200000402:0x1:0x0] \
ef=0xf u=500:500 nid=10.128.11.159@tcp x=user.name0
```

It includes information about the name of the extended attribute being accessed, in the form `x=<xattr name>`.

##### `SETXATTR`

A SETXATTR changelog entry is in the form:

```
4 15XATTR 09:41:36.157333594 2018.01.10 0x0 t=[0x200000402:0x1:0x0] \
ef=0xf u=500:500 nid=10.128.11.159@tcp x=user.name0
```

It includes information about the name of the extended attribute being modified, in the form `x=<xattr name>`.

##### `DENIED OPEN`

A DENIED OPEN changelog entry is in the form:

```
4 24NOPEN 15:45:44.947406626 2017.08.31 0x2 t=[0x200000402:0x1:0x0] \
ef=0xf u=500:500 nid=10.128.11.158@tcp m=-w-
```

It has the same information as a regular OPEN entry. In order to avoid flooding the Changelogs, DENIED OPEN entries are rate limited: no more than one entry per user per file per time interval, this time interval (in seconds) being configurable via `mdd.<mdtname>.changelog_deniednext` (default value is 60 seconds).

```
mds# lctl set_param mdd.lustre-MDT0000.changelog_deniednext=120
mdd.seb-MDT0000.changelog_deniednext=120
mds# lctl get_param mdd.lustre-MDT0000.changelog_deniednext
mdd.seb-MDT0000.changelog_deniednext=120
```

## Lustre Jobstats

The Lustre jobstats feature collects file system operation statistics for user processes running on Lustre clients, and exposes on the server using the unique Job Identifier (JobID) provided by the job scheduler for each job. Job schedulers known to be able to work with jobstats include: SLURM, SGE, LSF, Loadleveler, PBS and Maui/MOAB.

Since jobstats is implemented in a scheduler-agnostic manner, it is likely that it will be able to work with other schedulers also, and also in environments that do not use a job scheduler, by storing custom format strings in the `jobid_name`.

### How Jobstats Works

The Lustre jobstats code on the client extracts the unique JobID from an environment variable within the user process, and sends this JobID to the server with the I/O operation. The server tracks statistics for operations whose JobID is given, indexed by that ID.

A Lustre setting on the client, `jobid_var`, specifies which environment variable to holds the JobID for that process Any environment variable can be specified. For example, SLURM sets the `SLURM_JOB_ID` environment variable with the unique job ID on each client when the job is first launched on a node, and the `SLURM_JOB_ID` will be inherited by all child processes started below that process.

Lustre can be configured to generate a synthetic JobID from the client's process name and numeric UID, by setting `jobid_var=procname_uid`. This will generate a uniform JobID when running the same binary across multiple client nodes, but cannot distinguish whether the binary is part of a single distributed process or multiple independent processes.

In Lustre 2.8 and later it is possible to set `jobid_var=nodelocal` and then also set `jobid_name=name`, which all processes on that client node will use. This is useful if only a single job is run on a client at one time, but if multiple jobs are run on a client concurrently, the per-session JobID should be used.

In Lustre 2.12 and later, it is possible to specify more complex JobID values for `jobid_name` by using a string that contains format codes that are evaluated for each process, in order to generate a site- or nodespecific JobID string.

• %e print executable name 

• %g print group ID number 

• %h print fully-qualified hostname 

• %H print short hostname 

• %j print JobID from process environment variable named by the jobid_var parameter 

• %p print numeric process ID 

• %u print user ID number

In Lustre 2.13 and later, it is possible to set a per-session JobID by setting the `jobid_this_session` parameter. This will be inherited by all processes that are started in this login session, but there can be a different JobID for each login session.

The setting of `jobid_var` need not be the same on all clients. For example, one could use `SLURM_JOB_ID` on all clients managed by SLURM, and use `procname_uid` on clients not managed by SLURM, such as interactive login nodes.

It is not possible to have different `jobid_var` settings on a single node, since it is unlikely that multiple job schedulers are active on one client. However, the actual JobID value is local to each process environment and it is possible for multiple jobs with different JobIDs to be active on a single client at one time

### Enable/Disable Jobstats

Jobstats are disabled by default. The current state of jobstats can be verified by checking `lctl get_param jobid_var` on a client:

```
$ lctl get_param jobid_var
jobid_var=disable
      
```

To enable jobstats on the `testfs` file system with SLURM:

```
# lctl conf_param testfs.sys.jobid_var=SLURM_JOB_ID
```

The `lctl conf_param` command to enable or disable jobstats should be run on the MGS as root. The change is persistent, and will be propagated to the MDS, OSS, and client nodes automatically when it is set on the MGS and for each new client mount.

To temporarily enable jobstats on a client, or to use a different jobid_var on a subset of nodes, such as nodes in a remote cluster that use a different job scheduler, or interactive login nodes that do not use a job scheduler at all, run the `lctl set_param`command directly on the client node(s) after the filesystem is mounted. For example, to enable the `procname_uid` synthetic JobID on a login node run:

```
# lctl set_param jobid_var=procname_uid
```

The `lctl set_param` setting is not persistent, and will be reset if the global `jobid_var` is set on the MGS or if the filesystem is unmounted.

The following table shows the environment variables which are set by various job schedulers. Set `jobid_var` to the value for your job scheduler to collect statistics on a per job basis.

| **Job Scheduler**                                    | **Environment Variable** |
| ---------------------------------------------------- | ------------------------ |
| Simple Linux Utility for Resource Management (SLURM) | SLURM_JOB_ID             |
| Sun Grid Engine (SGE)                                | JOB_ID                   |
| Load Sharing Facility (LSF)                          | LSB_JOBID                |
| Loadleveler                                          | LOADL_STEP_ID            |
| Portable Batch Scheduler (PBS)/MAUI                  | PBS_JOBID                |
| Cray Application Level Placement Scheduler (ALPS)    | ALPS_APP_ID              |

There are two special values for `jobid_var`: `disable` and `procname_uid`. To disable jobstats, specify `jobid_var` as `disable`:

```
# lctl conf_param testfs.sys.jobid_var=disable
```

To track job stats per process name and user ID (for debugging, or if no job scheduler is in use on some nodes such as login nodes), specify `jobid_var` as `procname_uid`:

```
# lctl conf_param testfs.sys.jobid_var=procname_uid
```

### Check Job Stats

Metadata operation statistics are collected on MDTs. These statistics can be accessed for all file systems and all jobs on the MDT via the `lctl get_param mdt.*.job_stats`. For example, clients running with `jobid_var=procname_uid`:

```
# lctl get_param mdt.*.job_stats
job_stats:
- job_id:          bash.0
  snapshot_time:   1352084992
  open:            { samples:     2, unit:  reqs }
  close:           { samples:     2, unit:  reqs }
  mknod:           { samples:     0, unit:  reqs }
  link:            { samples:     0, unit:  reqs }
  unlink:          { samples:     0, unit:  reqs }
  mkdir:           { samples:     0, unit:  reqs }
  rmdir:           { samples:     0, unit:  reqs }
  rename:          { samples:     0, unit:  reqs }
  getattr:         { samples:     3, unit:  reqs }
  setattr:         { samples:     0, unit:  reqs }
  getxattr:        { samples:     0, unit:  reqs }
  setxattr:        { samples:     0, unit:  reqs }
  statfs:          { samples:     0, unit:  reqs }
  sync:            { samples:     0, unit:  reqs }
  samedir_rename:  { samples:     0, unit:  reqs }
  crossdir_rename: { samples:     0, unit:  reqs }
- job_id:          mythbackend.0
  snapshot_time:   1352084996
  open:            { samples:    72, unit:  reqs }
  close:           { samples:    73, unit:  reqs }
  mknod:           { samples:     0, unit:  reqs }
  link:            { samples:     0, unit:  reqs }
  unlink:          { samples:    22, unit:  reqs }
  mkdir:           { samples:     0, unit:  reqs }
  rmdir:           { samples:     0, unit:  reqs }
  rename:          { samples:     0, unit:  reqs }
  getattr:         { samples:   778, unit:  reqs }
  setattr:         { samples:    22, unit:  reqs }
  getxattr:        { samples:     0, unit:  reqs }
  setxattr:        { samples:     0, unit:  reqs }
  statfs:          { samples: 19840, unit:  reqs }
  sync:            { samples: 33190, unit:  reqs }
  samedir_rename:  { samples:     0, unit:  reqs }
  crossdir_rename: { samples:     0, unit:  reqs }
    
```

Data operation statistics are collected on OSTs. Data operations statistics can be accessed via `lctl get_param obdfilter.*.job_stats`, for example:

```
$ lctl get_param obdfilter.*.job_stats
obdfilter.myth-OST0000.job_stats=
job_stats:
- job_id:          mythcommflag.0
  snapshot_time:   1429714922
  read:    { samples: 974, unit: bytes, min: 4096, max: 1048576, sum: 91530035 }
  write:   { samples:   0, unit: bytes, min:    0, max:       0, sum:        0 }
  setattr: { samples:   0, unit:  reqs }
  punch:   { samples:   0, unit:  reqs }
  sync:    { samples:   0, unit:  reqs }
obdfilter.myth-OST0001.job_stats=
job_stats:
- job_id:          mythbackend.0
  snapshot_time:   1429715270
  read:    { samples:   0, unit: bytes, min:     0, max:      0, sum:        0 }
  write:   { samples:   1, unit: bytes, min: 96899, max:  96899, sum:    96899 }
  setattr: { samples:   0, unit:  reqs }
  punch:   { samples:   1, unit:  reqs }
  sync:    { samples:   0, unit:  reqs }
obdfilter.myth-OST0002.job_stats=job_stats:
obdfilter.myth-OST0003.job_stats=job_stats:
obdfilter.myth-OST0004.job_stats=
job_stats:
- job_id:          mythfrontend.500
  snapshot_time:   1429692083
  read:    { samples:   9, unit: bytes, min: 16384, max: 1048576, sum: 4444160 }
  write:   { samples:   0, unit: bytes, min:     0, max:       0, sum:       0 }
  setattr: { samples:   0, unit:  reqs }
  punch:   { samples:   0, unit:  reqs }
  sync:    { samples:   0, unit:  reqs }
- job_id:          mythbackend.500
  snapshot_time:   1429692129
  read:    { samples:   0, unit: bytes, min:     0, max:       0, sum:       0 }
  write:   { samples:   1, unit: bytes, min: 56231, max:   56231, sum:   56231 }
  setattr: { samples:   0, unit:  reqs }
  punch:   { samples:   1, unit:  reqs }
  sync:    { samples:   0, unit:  reqs }
```

### Clear Job Stats

Accumulated job statistics can be reset by writing proc file `job_stats`.

Clear statistics for all jobs on the local node:

```
# lctl set_param obdfilter.*.job_stats=clear
```

Clear statistics only for job 'bash.0' on lustre-MDT0000:

```
# lctl set_param mdt.lustre-MDT0000.job_stats=bash.0
```

### Configure Auto-cleanup Interval

By default, if a job is inactive for 600 seconds (10 minutes) statistics for this job will be dropped. This expiration value can be changed temporarily via:

```
# lctl set_param *.*.job_cleanup_interval={max_age}
```

It can also be changed permanently, for example to 700 seconds via:

```
# lctl conf_param testfs.mdt.job_cleanup_interval=700
```

The `job_cleanup_interval` can be set as 0 to disable the auto-cleanup. Note that if auto-cleanup of Jobstats is disabled, then all statistics will be kept in memory forever, which may eventually consume all memory on the servers. In this case, any monitoring tool should explicitly clear individual job statistics as they are processed, as shown above.

## Lustre Monitoring Tool (LMT)

The Lustre Monitoring Tool (LMT) is a Python-based, distributed system that provides a top-like display of activity on server-side nodes (MDS, OSS and portals routers) on one or more Lustre file systems. It does not provide support for monitoring clients. For more information on LMT, including the setup procedure, see:

[https://github.com/chaos/lmt/wiki](https://github.com/chaos/lmt/wiki)

## `CollectL`

`CollectL` is another tool that can be used to monitor a Lustre file system. You can run `CollectL` on a Lustre system that has any combination of MDSs, OSTs and clients. The collected data can be written to a file for continuous logging and played back at a later time. It can also be converted to a format suitable for plotting.

For more information about `CollectL`, see:

<http://collectl.sourceforge.net>

Lustre-specific documentation is also available. See:

<http://collectl.sourceforge.net/Tutorial-Lustre.html>

## Other Monitoring Options

A variety of standard tools are available publicly including the following:

- `lltop` - Lustre load monitor with batch scheduler integration. <https://github.com/jhammond/lltop>
- `tacc_stats` - A job-oriented system monitor, analyzation, and visualization tool that probes Lustre interfaces and collects statistics. <https://github.com/jhammond/tacc_stats>
- `xltop` - A continuous Lustre monitor with batch scheduler integration. <https://github.com/jhammond/xltop>

Another option is to script a simple monitoring solution that looks at various reports from `ipconfig`, as well as the `procfs` files generated by the Lustre software.