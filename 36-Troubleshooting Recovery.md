# Troubleshooting Recovery

- [Troubleshooting Recovery](#troubleshooting-recovery)
  * [Recovering from Errors or Corruption on a Backing ldiskfs File System](#recovering-from-errors-or-corruption-on-a-backing-ldiskfs-file-system)
  * [Recovering from Corruption in the Lustre File System](#recovering-from-corruption-in-the-lustre-file-system)
    + [Working with Orphaned Objects](#working-with-orphaned-objects)
  * [Recovering from an Unavailable OST](#recovering-from-an-unavailable-ost)
  * [Checking the file system with LFSCK](#checking-the-file-system-with-lfsck)
    + [LFSCK switch interface](#lfsck-switch-interface)
      - [Manually Starting LFSCK](#manually-starting-lfsck)
        * [Description](#description)
        * [Usage](#usage)
        * [Options](#options)
      - [Manually Stopping LFSCK](#manually-stopping-lfsck)
        * [Description](#description-1)
        * [Usage](#usage-1)
        * [Options](#options-1)
    + [Check the LFSCK global status](#check-the-lfsck-global-status)
      - [Description](#description-2)
      - [Usage](#usage-2)
      - [Options](#options-2)
    + [LFSCK status interface](#lfsck-status-interface)
      - [LFSCK status of OI Scrub via `procfs`](#lfsck-status-of-oi-scrub-via-procfs)
        * [Description](#description-3)
        * [Usage](#usage-3)
        * [Output](#output)
      - [LFSCK status of namespace via `procfs`](#lfsck-status-of-namespace-via-procfs)
        * [Description](#description-4)
        * [Usage](#usage-4)
        * [Output](#output-1)
      - [LFSCK status of layout via `procfs`](#lfsck-status-of-layout-via-procfs)
        * [Description](#description-5)
        * [Usage](#usage-5)
        * [Output](#output-2)
    + [LFSCK adjustment interface](#lfsck-adjustment-interface)
      - [Rate control](#rate-control)
        * [Description](#description-6)
        * [Usage](#usage-6)
        * [Values](#values)
      - [Auto scrub](#auto-scrub)
        * [Description](#description-7)
        * [Usage](#usage-7)
        * [Values](#values-1)

This chapter describes what to do if something goes wrong during recovery. It describes:

- [the section called “ Recovering from Errors or Corruption on a Backing ldiskfs File System”](#recovering-from-errors-or-corruption-on-a-backing-ldiskfs-file-system)
- [the section called “ Recovering from Corruption in the Lustre File System”](#recovering-from-corruption-in-the-lustre-file-system)
- [the section called “ Recovering from an Unavailable OST”](#recovering-from-corruption-in-the-lustre-file-system)
- [the section called “ Checking the file system with LFSCK”](#checking-the-file-system-with-lfsck)

## Recovering from Errors or Corruption on a Backing ldiskfs File System

When an OSS, MDS, or MGS server crash occurs, it is not necessary to run e2fsck on the file system. `ldiskfs`journaling ensures that the file system remains consistent over a system crash. The backing file systems are never accessed directly from the client, so client crashes are not relevant for server file system consistency.

The only time it is REQUIRED that `e2fsck` be run on a device is when an event causes problems that ldiskfs journaling is unable to handle, such as a hardware device failure or I/O error. If the ldiskfs kernel code detects corruption on the disk, it mounts the file system as read-only to prevent further corruption, but still allows read access to the device. This appears as error "-30" ( `EROFS`) in the syslogs on the server, e.g.:

```
Dec 29 14:11:32 mookie kernel: LDISKFS-fs error (device sdz):
            ldiskfs_lookup: unlinked inode 5384166 in dir #145170469
Dec 29 14:11:32 mookie kernel: Remounting filesystem read-only 
```

In such a situation, it is normally required that e2fsck only be run on the bad device before placing the device back into service.

In the vast majority of cases, the Lustre software can cope with any inconsistencies found on the disk and between other devices in the file system.

For problem analysis, it is strongly recommended that `e2fsck` be run under a logger, like `script`, to record all of the output and changes that are made to the file system in case this information is needed later.

If time permits, it is also a good idea to first run `e2fsck` in non-fixing mode (-n option) to assess the type and extent of damage to the file system. The drawback is that in this mode, `e2fsck` does not recover the file system journal, so there may appear to be file system corruption when none really exists.

To address concern about whether corruption is real or only due to the journal not being replayed, you can briefly mount and unmount the `ldiskfs` file system directly on the node with the Lustre file system stopped, using a command similar to:

```
mount -t ldiskfs /dev/{ostdev} /mnt/ost; umount /mnt/ost
```

This causes the journal to be recovered.

The `e2fsck` utility works well when fixing file system corruption (better than similar file system recovery tools and a primary reason why `ldiskfs` was chosen over other file systems). However, it is often useful to identify the type of damage that has occurred so an `ldiskfs` expert can make intelligent decisions about what needs fixing, in place of`e2fsck`.

```
root# {stop lustre services for this device, if running}
root# script /tmp/e2fsck.sda
Script started, file is /tmp/e2fsck.sda
root# mount -t ldiskfs /dev/sda /mnt/ost
root# umount /mnt/ost
root# e2fsck -fn /dev/sda   # don't fix file system, just check for corruption
:
[e2fsck output]
:
root# e2fsck -fp /dev/sda   # fix errors with prudent answers (usually yes)
```

## Recovering from Corruption in the Lustre File System

In cases where an ldiskfs MDT or OST becomes corrupt, you need to run `e2fsck` to ensure local filesystem consistency, then use `LFSCK` to run a distributed check on the file system to resolve any inconsistencies between the MDTs and OSTs, or among MDTs.

1. Stop the Lustre file system.

2. Run `e2fsck -f` on the individual MDT/OST that had problems to fix any local file system damage.

   We recommend running `e2fsck` under script, to create a log of changes made to the file system in case it is needed later. After `e2fsck` is run, bring up the file system, if necessary, to reduce the outage window.

### Working with Orphaned Objects

The simplest problem to resolve is that of orphaned objects. When the LFSCK layout check is run, these objects are linked to new files and put into `.lustre/lost+found/MDT*xxxx*` in the Lustre file system (where MDTxxxx is the index of the MDT on which the orphan was found), where they can be examined and saved or deleted as necessary.

Introduced in Lustre 2.7With Lustre version 2.7 and later, LFSCK will identify and process orphan objects found on MDTs as well.

## Recovering from an Unavailable OST

One problem encountered in a Lustre file system environment is when an OST becomes unavailable due to a network partition, OSS node crash, etc. When this happens, the OST's clients pause and wait for the OST to become available again, either on the primary OSS or a failover OSS. When the OST comes back online, the Lustre file system starts a recovery process to enable clients to reconnect to the OST. Lustre servers put a limit on the time they will wait in recovery for clients to reconnect.

During recovery, clients reconnect and replay their requests serially, in the same order they were done originally. Until a client receives a confirmation that a given transaction has been written to stable storage, the client holds on to the transaction, in case it needs to be replayed. Periodically, a progress message prints to the log, stating how_many/expected clients have reconnected. If the recovery is aborted, this log shows how many clients managed to reconnect. When all clients have completed recovery, or if the recovery timeout is reached, the recovery period ends and the OST resumes normal request processing.

If some clients fail to replay their requests during the recovery period, this will not stop the recovery from completing. You may have a situation where the OST recovers, but some clients are not able to participate in recovery (e.g. network problems or client failure), so they are evicted and their requests are not replayed. This would result in any operations on the evicted clients failing, including in-progress writes, which would cause cached writes to be lost. This is a normal outcome; the recovery cannot wait indefinitely, or the file system would be hung any time a client failed. The lost transactions are an unfortunate result of the recovery process.

**Note**

The failure of client recovery does not indicate or lead to filesystem corruption. This is a normal event that is handled by the MDT and OST, and should not result in any inconsistencies between servers.

**Note**

The version-based recovery (VBR) feature enables a failed client to be ''skipped'', so remaining clients can replay their requests, resulting in a more successful recovery from a downed OST. For more information about the VBR feature, see *Lustre File System Recovery*(Version-based Recovery).

## Checking the file system with LFSCK

LFSCK is an administrative tool for checking and repair of the attributes specific to a mounted Lustre file system. It is similar in concept to an offline fsck repair tool for a local filesystem, but LFSCK is implemented to run as part of the Lustre file system while the file system is mounted and in use. This allows consistency checking and repair of Lustre-specific metadata without unnecessary downtime, and can be run on the largest Lustre file systems with minimal impact to normal operations.

LFSCK can verify and repair the Object Index (OI) table that is used internally to map Lustre File Identifiers (FIDs) to MDT internal ldiskfs inode numbers, in an internal table called the OI Table. An OI Scrub traverses the OI table and makes corrections where necessary. An OI Scrub is required after restoring from a file-level MDT backup ( [*the section called “ Backing Up and Restoring an MDT or OST (ldiskfs Device Level)”*](03.07-BackingUp%20and%20Restoring%20a%20File%20System.md#backing-up-and-restoring-an-mdt-or-ost-ldiskfs-device-level)), or in case the OI Table is otherwise corrupted. Later phases of LFSCK will add further checks to the Lustre distributed file system state. LFSCK namespace scanning can verify and repair the directory FID-in-dirent and LinkEA consistency.

Introduced in Lustre 2.6In Lustre software release 2.6, LFSCK layout scanning can verify and repair MDT-OST file layout inconsistencies. File layout inconsistencies between MDT-objects and OST-objects that are checked and corrected include dangling reference, unreferenced OST-objects, mismatched references and multiple references.

Introduced in Lustre 2.7

In Lustre software release 2.7, LFSCK layout scanning is enhanced to support verify and repair inconsistencies between multiple MDTs.

Control and monitoring of LFSCK is through LFSCK and the  `lctl get_param` command. LFSCK supports three types of interface: switch interface, status interface, and adjustment interface. These interfaces are detailed below.

### LFSCK switch interface

#### Manually Starting LFSCK

##### Description

LFSCK can be started after the MDT is mounted using the `lctl lfsck_start` command.

##### Usage

```
lctl lfsck_start <-M | --device [MDT,OST]_device> \
                    [-A | --all] \
                    [-c | --create_ostobj on | off] \
                    [-C | --create_mdtobj on | off] \
                    [-d | --delay_create_ostobj on | off] \
                    [-e | --error {continue | abort}] \
                    [-h | --help] \
                    [-n | --dryrun on | off] \
                    [-o | --orphan] \
                    [-r | --reset] \
                    [-s | --speed ops_per_sec_limit] \
                    [-t | --type check_type[,check_type...]] \
                    [-w | --window_size size]
```

##### Options

The various `lfsck_start` options are listed and described below. For a complete list of available options, type `lctl lfsck_start -h`.

| **Option**                   | **Description**                                              |
| ---------------------------- | ------------------------------------------------------------ |
| `-M | --device`              | The MDT or OST target to start LFSCK on.                     |
| `-A | --all`                 | Introduced in Lustre 2.6                                                                                        Start LFSCK on all targets on all servers simultaneously. By default, both layout and namespace consistency checking and repair are started. |
| `-c | --create_ostobj`       | Introduced in Lustre 2.6                                                                                            Create the lost OST-object for dangling LOV EA, `off`(default) or `on`. If not specified, then the default behaviour is to keep the dangling LOV EA there without creating the lost OST-object. |
| `-C | --create_mdtobj`       | Introduced in Lustre 2.7                                                                                           Create the lost MDT-object for dangling name entry, `off`(default) or `on`. If not specified, then the default behaviour is to keep the dangling name entry there without creating the lost MDT-object. |
| `-d | --delay_create_ostobj` | Introduced in Lustre 2.9                                                                                                Delay creating the lost OST-object for dangling LOV EA until the orphan OST-objects are handled. `off`(default) or `on`. |
| `-e | --error`               | Error handle, `continue`(default) or `abort`. Specify whether the LFSCK will stop or not if fails to repair something. If it is not specified, the saved value (when resuming from checkpoint) will be used if present. This option cannot be changed while LFSCK is running. |
| `-h | --help`                | Operating help information.                                  |
| `-n | --dryrun`              | Perform a trial without making any changes. `off`(default) or `on`. |
| `-o | --orphan`              | Introduced in Lustre 2.6                                                                                                     Repair orphan OST-objects for layout LFSCK. |
| `-r | --reset`               | Reset the start position for the object iteration to the beginning for the specified MDT. By default the iterator will resume scanning from the last checkpoint (saved periodically by LFSCK) provided it is available. |
| `-s | --speed`               | Set the upper speed limit of LFSCK processing in objects per second. If it is not specified, the saved value (when resuming from checkpoint) or default value of 0 (0 = run as fast as possible) is used. Speed can be adjusted while LFSCK is running with the adjustment interface. |
| `-t | --type`                | The type of checking/repairing that should be performed. The new LFSCK framework provides a single interface for a variety of system consistency checking/repairing operations including:                                                            Without a specified option, the LFSCK component(s) which ran last time and did not finish or the component(s) corresponding to some known system inconsistency, will be started. Anytime the LFSCK is triggered, the OI scrub will run automatically, so there is no need to specify OI_scrub in that case.       Introduced in Lustre 2.4                                                                               `namespace`: check and repair FID-in-dirent and LinkEA consistency.              Introduced in Lustre 2.7                                                                                      Lustre-2.7 enhances namespace consistency verification under DNE mode.Introduced in Lustre 2.6`layout`: check and repair MDT-OST inconsistency. |
| `-w | --window_size`         | Introduced in Lustre 2.6                                                                                                      The window size for the async request pipeline. The LFSCK async request pipeline's input/output may have quite different processing speeds, and there may be too many requests in the pipeline as to cause abnormal memory/network pressure. If not specified, then the default window size for the async request pipeline is 1024. |

#### Manually Stopping LFSCK

##### Description

To stop LFSCK when the MDT is mounted, use the `lctl lfsck_stop` command.

##### Usage

```
lctl lfsck_stop <-M | --device [MDT,OST]_device> \
                    [-A | --all] \
                    [-h | --help]
```

##### Options

The various `lfsck_stop` options are listed and described below. For a complete list of available options, type `lctl lfsck_stop -h`.

| **Option**      | **Description**                                          |
| --------------- | -------------------------------------------------------- |
| `-M | --device` | The MDT or OST target to stop LFSCK on.                  |
| `-A | --all`    | Stop LFSCK on all targets on all servers simultaneously. |
| `-h | --help`   | Operating help information.                              |

### Check the LFSCK global status

#### Description

Check the LFSCK global status via a single `lctl lfsck_query` command on the MDS.

#### Usage

```
lctl lfsck_query <-M | --device MDT_device> \
                    [-h | --help] \
                    [-t | --type lfsck_type[,lfsck_type...]] \
                    [-w | --wait]
```

#### Options

The various `lfsck_query` options are listed and described below. For a complete list of available options, type `lctl lfsck_query -h`.

| **Option**      | **Description**                                              |
| --------------- | ------------------------------------------------------------ |
| `-M | --device` | The device to query for LFSCK status.                        |
| `-h | --help`   | Operating help information.                                  |
| `-t | --type`   | The LFSCK type(s) that should be queried, including: layout, namespace. |
| `-w | --wait`   | will wait if the LFSCK is in scanning.                       |

### LFSCK status interface

#### LFSCK status of OI Scrub via `procfs`

##### Description

For each LFSCK component there is a dedicated procfs interface to trace the corresponding LFSCK component status. For OI Scrub, the interface is the OSD layer procfs interface, named `oi_scrub`. To display OI Scrub status, the standard `lctl get_param` command is used as shown in the usage below.

##### Usage

```
lctl get_param -n osd-ldiskfs.FSNAME-[MDT_target|OST_target].oi_scrub
```

##### Output

| **Information**     | **Detail**                                                   |
| ------------------- | ------------------------------------------------------------ |
| General Information | - Name: OI_scrub.                                                                                                                                               - OI scrub magic id (an identifier unique to OI scrub).                                                                                              - OI files count.                                                                                                                            --- Status: one of the status - `init`, `scanning`, `completed`, `failed`, `stopped`, `paused`, or`crashed`.                                                                                                                           - Flags: including - `recreated`(OI file(s) is/are removed/recreated), `inconsistent`(restored from file-level backup), `auto`(triggered by non-UI mechanism), and `upgrade`(from Lustre software release 1.8 IGIF format.)                              - Parameters: OI scrub parameters, like `failout`.                                                                 - Time Since Last Completed.                                                                                                            - Time Since Latest Start.                                                                                                              - Time Since Last Checkpoint.                                                                                                               - Latest Start Position: the position for the latest scrub started from.                                        - Last Checkpoint Position.                                                                                                                - First Failure Position: the position for the first object to be repaired.                                  - Current Position. |
| Statistics          | - `Checked` total number of objects scanned.                                                                              - `Updated` total number of objects repaired.                                                                              - `Failed` total number of objects that failed to be repaired.                                                  - `No Scrub` total number of objects marked `LDISKFS_STATE_LUSTRE_NOSCRUB and skipped`.                                                                                                                                             - `IGIF` total number of objects IGIF scanned.                                                                                        - `Prior Updated` how many objects have been repaired which are triggered by parallel RPC.                                                                                                                                                        - `Success Count` total number of completed OI_scrub runs on the target.                           - `Run Time` how long the scrub has run, tally from the time of scanning from the beginning of the specified MDT target, not include the paused/failure time among checkpoints.                                                                                                                                           - `Average Speed` calculated by dividing `Checked` by `run_time`.                                               - `Real-Time Speed` the speed since last checkpoint if the OI_scrub is running.                   - `Scanned` total number of objects under /lost+found that have been scanned.                   - `Repaired` total number of objects under /lost+found that have been recovered.               - `Failed` total number of objects under /lost+found failed to be scanned or failed to be recovered. |



Introduced in Lustre 2.4

#### LFSCK status of namespace via `procfs`

##### Description

The `namespace` component is responsible for checks described in [the section called “ Checking the file system with LFSCK”](#checking-the-file-system-with-lfsck). The `procfs` interface for this component is in the MDD layer, named `lfsck_namespace`. To show the status of this component, `lctl get_param` should be used as described in the usage below.

The LFSCK namespace status output refers to phase 1 and phase 2. Phase 1 is when the LFSCK main engine, which runs on each MDT, linearly scans its local device, guaranteeing that all local objects are checked. However, there are certain cases in which LFSCK cannot know whether an object is consistent or cannot repair an inconsistency until the phase 1 scanning is completed. During phase 2 of the namespace check, objects with multiple hard-links, objects with remote parents, and other objects which couldn't be verified during phase 1 will be checked.

##### Usage

```
lctl get_param -n mdd. FSNAME-MDT_target.lfsck_namespace
```

##### Output

| **Information**     | **Detail**                                                   |
| ------------------- | ------------------------------------------------------------ |
| General Information | - Name: `lfsck_namespace`                                                                                                           - LFSCK namespace magic.                                                                                                              - LFSCK namespace version..                                                                                                           - Status: one of the status - `init`, `scanning-phase1`, `scanning-phase2`, `completed`, `failed`,`stopped`, `paused`, `partial`, `co-failed`, `co-stopped` or `co-paused`.                                                                                                                        - Flags: including - `scanned-once`(the first cycle scanning has been completed),`inconsistent`(one or more inconsistent FID-in-dirent or LinkEA entries that have been discovered), `upgrade`(from Lustre software release 1.8 IGIF format.)                                                                                                                - Parameters: including `dryrun`, `all_targets`, `failout`, `broadcast`, `orphan`, `create_ostobj`and `create_mdtobj`.                                                                                             - Time Since Last Completed.                                                                                                                - Time Since Latest Start.                                                                                                                  - Time Since Last Checkpoint.                                                                                                               - Latest Start Position: the position the checking began most recently.                                                                                                                                                - Last Checkpoint Position.                                                                                                             - First Failure Position: the position for the first object to be repaired.                                      - Current Position. |
| Statistics          | - `Checked Phase1` total number of objects scanned during `scanning-phase1`.                - `Checked Phase2` total number of objects scanned during `scanning-phase2`.                - `Updated Phase1` total number of objects repaired during `scanning-phase1`.                                                                                                                              - `Updated Phase2` total number of objects repaired during `scanning-phase2`.                  - `Failed Phase1` total number of objets that failed to be repaired during `scanning-phase1`.                                                                                                                                               - `Failed Phase2` total number of objets that failed to be repaired during `scanning-phase2`.                                                                                                                                                 - `directories` total number of directories scanned.                                                                                                                                                                          - `multiple_linked_checked` total number of multiple-linked objects that have been scanned.                                                                                                                                                  - `dirent_repaired` total number of FID-in-dirent entries that have been repaired.                                                                                                          - `linkea_repaired` total number of linkEA entries that have been repaired.                                                                                                          - `unknown_inconsistency` total number of undefined inconsistencies found in scanning-phase2.                                                                                                                            - `unmatched_pairs_repaired` total number of unmatched pairs that have been repaired.                                                                                                                                                      - `dangling_repaired` total number of dangling name entries that have been found/repaired.                                                                                                                                          - `multi_referenced_repaired` total number of multiple referenced name entries that have been found/repaired.                                                                                                                                   - `bad_file_type_repaired` total number of name entries with bad file type that have been repaired.                                                                                                                                           - `lost_dirent_repaired` total number of lost name entries that have been re-inserted.                                                                                                                                                   - `striped_dirs_scanned` total number of striped directories (master) that have been scanned.                                                                                                                                               - `striped_dirs_repaired` total number of striped directories (master) that have been repaired.                                                                                                                                                    - `striped_dirs_failed` total number of striped directories (master) that have failed to be verified.                                                                                                                                              - `striped_dirs_disabled` total number of striped directories (master) that have been disabled.                                                                                                                                                - `striped_dirs_skipped` total number of striped directories (master) that have been skipped (for shards verification) because of lost master LMV EA.                                                                                                          - `striped_shards_scanned` total number of striped directory shards (slave) that have been scanned.                                                                                                                                         - `striped_shards_repaired` total number of striped directory shards (slave) that have been repaired.                                                                                                                                          - `striped_shards_failed` total number of striped directory shards (slave) that have failed to be verified.                                                                                                                                      - `striped_shards_skipped` total number of striped directory shards (slave) that have been skipped (for name hash verification) because LFSCK does not know whether the slave LMV EA is valid or not.                                                                                                          - `name_hash_repaired` total number of name entries under striped directory with bad name hash that have been repaired.                                                                                                          - `nlinks_repaired` total number of objects with nlink fixed.                                                                                                          - `mul_linked_repaired` total number of multiple-linked objects that have been repaired.                                                                                                                                                        - `local_lost_found_scanned` total number of objects under /lost+found that have been scanned.                                                                                                                                                                            - `local_lost_found_moved` total number of objects under /lost+found that have been moved to namespace visible directory.                                                                                                       - `local_lost_found_skipped` total number of objects under /lost+found that have been skipped.                                                                                                                                                                                    - `local_lost_found_failed` total number of objects under /lost+found that have failed to be processed.                                                                                                                                                                   - `Success Count` the total number of completed LFSCK runs on the target.                                                                                                          - `Run Time Phase1` the duration of the LFSCK run during `scanning-phase1`. Excluding the time spent paused between checkpoints.                                                                                                          - `Run Time Phase2` the duration of the LFSCK run during `scanning-phase2`. Excluding the time spent paused between checkpoints.                                                                                                          - `Average Speed Phase1` calculated by dividing `checked_phase1` by `run_time_phase1`.                                                                                                                                       - `Average Speed Phase2` calculated by dividing `checked_phase2` by `run_time_phase1`.                                                                                                                                                  - `Real-Time Speed Phase1` the speed since the last checkpoint if the LFSCK is running`scanning-phase1`.                                                                                                                                        - `Real-Time Speed Phase2` the speed since the last checkpoint if the LFSCK is running`scanning-phase2`. |



Introduced in Lustre 2.6

#### LFSCK status of layout via `procfs`

##### Description

The `layout` component is responsible for checking and repairing MDT-OST inconsistency. The `procfs` interface for this component is in the MDD layer, named `lfsck_layout`, and in the OBD layer, named `lfsck_layout`. To show the status of this component `lctl get_param` should be used as described in the usage below.

The LFSCK layout status output refers to phase 1 and phase 2. Phase 1 is when the LFSCK main engine, which runs on each MDT/OST, linearly scans its local device, guaranteeing that all local objects are checked. During phase 1 of layout LFSCK, the OST-objects which are not referenced by any MDT-object are recorded in a bitmap. During phase 2 of the layout check, the OST-objects in the bitmap will be re-scanned to check whether they are really orphan objects.

##### Usage

```
lctl get_param -n mdd.
FSNAME-
MDT_target.lfsck_layout
lctl get_param -n obdfilter.
FSNAME-
OST_target.lfsck_layout
```

##### Output

| **Information**     | **Detail**                                                   |
| ------------------- | ------------------------------------------------------------ |
| General Information | - Name: `lfsck_layout`                                                                                                                                                   - LFSCK namespace magic.LFSCK namespace version..                                                                                                                                                   - Status: one of the status - `init`, `scanning-phase1`, `scanning-phase2`, `completed`, `failed`,`stopped`, `paused`, `crashed`, `partial`, `co-failed`, `co-stopped`, or `co-paused`.                                                                                                                                                   - Flags: including - `scanned-once`(the first cycle scanning has been completed),`inconsistent`(one or more MDT-OST inconsistencies have been discovered),`incomplete`(some MDT or OST did not participate in the LFSCK or failed to finish the LFSCK) or `crashed_lastid`(the lastid files on the OST crashed and needs to be rebuilt).                                                                                                                                                   - Parameters: including `dryrun`, `all_targets` and `failout`.                                                                                                                                                   - Time Since Last Completed.                                                                                                                                                   - Time Since Latest Start.                                                                                                                                                   - Time Since Last Checkpoint.                                                                                                                                                   - Latest Start Position: the position the checking began most recently.                                                                                                                                                   - Last Checkpoint Position.                                                                                                                                                   - First Failure Position: the position for the first object to be repaired.                                                                                                                                                   - Current Position. |
| Statistics          |                                                                                                                                                    - `Success Count:` the total number of completed LFSCK runs on the target.                                                                                                                                                   - `Repaired Dangling:` total number of MDT-objects with dangling reference have been repaired in the scanning-phase1.                                                                                                                                                   - `Repaired Unmatched Pairs` total number of unmatched MDT and OST-object pairs have been repaired in the scanning-phase1`Repaired Multiple Referenced` total number of OST-objects with multiple reference have been repaired in the scanning-phase1.                                                                                                                                                   - `Repaired Orphan` total number of orphan OST-objects have been repaired in the scanning-phase2.                                                                                                                                                   - `Repaired Inconsistent Owner` total number.of OST-objects with incorrect owner information have been repaired in the scanning-phase1.                                                                                                                                                   - `Repaired Others` total number of.other inconsistency repaired in the scanning phases.`Skipped` Number of skipped objects.                                                                                                                                                   - `Failed Phase1` total number of objects that failed to be repaired during `scanning-phase1`.                                                                                                                                                   - `Failed Phase2` total number of objects that failed to be repaired during `scanning-phase2`.                                                                                                                                                   - `Checked Phase1` total number of objects scanned during `scanning-phase1`.                                                                                                                                                   - `Checked Phase2` total number of objects scanned during `scanning-phase2`.                                                                                                                                                   - `Run Time Phase1` the duration of the LFSCK run during `scanning-phase1`. Excluding the time spent paused between checkpoints.                                                                                                                                                   - `Run Time Phase2` the duration of the LFSCK run during `scanning-phase2`. Excluding the time spent paused between checkpoints.                                                                                                                                                   - `Average Speed Phase1` calculated by dividing `checked_phase1` by `run_time_phase1`.                                                                                                                                                   - `Average Speed Phase2` calculated by dividing `checked_phase2` by `run_time_phase1`.                                                                                                                                                   - `Real-Time Speed Phase1` the speed since the last checkpoint if the LFSCK is running`scanning-phase1`.                                                                                                                                                   - `Real-Time Speed Phase2` the speed since the last checkpoint if the LFSCK is running`scanning-phase2`. |

### LFSCK adjustment interface

Introduced in Lustre 2.6

#### Rate control

##### Description

The LFSCK upper speed limit can be changed using `lctl set_param` as shown in the usage below.

##### Usage

```
lctl set_param mdd.${FSNAME}-${MDT_target}.lfsck_speed_limit=
N
lctl set_param obdfilter.${FSNAME}-${OST_target}.lfsck_speed_limit=
N
```

##### Values

| 0                | No speed limit (run at maximum speed.)        |
| ---------------- | --------------------------------------------- |
| positive integer | Maximum number of objects to scan per second. |

#### Auto scrub

##### Description

The `auto_scrub` parameter controls whether OI scrub will be triggered when an inconsistency is detected during OI lookup. It can be set as described in the usage and values sections below.

There is also a `noscrub` mount option (see [*the section called “ mount.lustre”*](06.07-System%20Configuration%20Utilities.md#mountlustre)) which can be used to disable automatic OI scrub upon detection of a file-level backup at mount time. If the `noscrub` mount option is specified,`auto_scrub` will also be disabled, so OI scrub will not be triggered when an OI inconsistency is detected. Auto scrub can be renabled after the mount using the command shown in the usage. Manually starting LFSCK after mounting provides finer control over the starting conditions.

##### Usage

```
lctl set_param osd_ldiskfs.${FSNAME}-${MDT_target}.auto_scrub=N
```

where *N* is an integer as described below.

Introduced in Lustre 2.5

**Note**

Lustre software 2.5 and later supports `-P` option that makes the `set_param` permanent.

##### Values

| 0                | Do not start OI Scrub automatically.                         |
| ---------------- | ------------------------------------------------------------ |
| positive integer | Automatically start OI Scrub if inconsistency is detected during OI lookup. |

