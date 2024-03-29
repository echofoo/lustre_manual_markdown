# Lustre File System Recovery

- [Lustre File System Recovery](#lustre-file-system-recovery)
  * [Recovery Overview](#recovery-overview)
    + [Client Failure](#client-failure)
    + [Client Eviction](#client-eviction)
    + [MDS Failure (Failover)](#mds-failure-failover)
    + [OST Failure (Failover)](#ost-failure-failover)
    + [Network Partition](#network-partition)
    + [Failed Recovery](#failed-recovery)
  * [Metadata Replay](#metadata-replay)
    + [XID Numbers](#xid-numbers)
    + [Transaction Numbers](#transaction-numbers)
    + [Replay and Resend](#replay-and-resend)
    + [Client Replay List](#client-replay-list)
    + [Server Recovery](#server-recovery)
    + [Request Replay](#request-replay)
    + [Gaps in the Replay Sequence](#gaps-in-the-replay-sequence)
    + [Lock Recovery](#lock-recovery)
    + [Request Resend](#request-resend)
  * [Reply Reconstruction](#reply-reconstruction)
    + [Required State](#required-state)
    + [Reconstruction of Open Replies](#reconstruction-of-open-replies)
      - [Finding the File Handle](#finding-the-file-handle)
      - [Finding the Resource/fid](#finding-the-resourcefid)
      - [Finding the Lock Handle](#finding-the-lock-handle)
    + [Multiple Reply Data per Client](#multiple-reply-data-per-client)L 2.8
  * [Version-based Recovery](#version-based-recovery)
    + [VBR Messages](#vbr-messages)
    + [Tips for Using VBR](#tips-for-using-vbr)
  * [Commit on Share](#commit-on-share)
    + [Working with Commit on Share](#working-with-commit-on-share)
    + [Tuning Commit On Share](#tuning-commit-on-share)
  * [Imperative Recovery](#imperative-recovery)
    + [MGS role](#mgs-role)
    + [Tuning Imperative Recovery](#tuning-imperative-recovery)
      - [ir_factor](#ir_factor)
      - [Disabling Imperative Recovery](#disabling-imperative-recovery)
      - [Checking Imperative Recovery State - MGS](#checking-imperative-recovery-state---mgs)
      - [Checking Imperative Recovery State - client](#checking-imperative-recovery-state---client)
      - [Target Instance Number](#target-instance-number)
    + [Configuration Suggestions for Imperative Recovery](#configuration-suggestions-for-imperative-recovery)
  * [Suppressing Pings](#suppressing-pings)
    + ["suppress_pings" Kernel Module Parameter](#suppress_pings-kernel-module-parameter)
    + [Client Death Notification](#client-death-notification)


This chapter describes how recovery is implemented in a Lustre file system and includes the following sections:

- [the section called “ Recovery Overview”](#recovery-overview)
- [the section called “Metadata Replay”](#recovery-overview)
- [the section called “Reply Reconstruction”](#recovery-overview)
- [the section called “Version-based Recovery”](#version-based-recovery)
- [the section called “Commit on Share”](#commit-on-share)
- [the section called “Imperative Recovery”](#imperative-recovery)

## Recovery Overview

The recovery feature provided in the Lustre software is responsible for dealing with node or network failure and returning the cluster to a consistent, performant state. Because the Lustre software allows servers to perform asynchronous update operations to the on-disk file system (i.e., the server can reply without waiting for the update to synchronously commit to disk), the clients may have state in memory that is newer than what the server can recover from disk after a crash.

A handful of different types of failures can cause recovery to occur:

- Client (compute node) failure
- MDS failure (and failover)
- OST failure (and failover)
- Transient network partition

For Lustre, all Lustre file system failure and recovery operations are based on the concept of connection failure; all imports or exports associated with a given connection are considered to fail if any of them fail.  [*the section called “Imperative Recovery”*](#imperative-recovery) feature allows the MGS to actively inform clients when a target restarts after a failure, failover, or other interruption to speed up recovery.

For information on Lustre file system recovery, see [*the section called “Metadata Replay”*](#metadata-replay). For information on recovering from a corrupt file system, see [*the section called “Commit on Share”*](#commit-on-share). For information on resolving orphaned objects, a common issue after recovery, see [*the section called “ Working with Orphaned Objects”*](05.02-Troubleshooting%20Recovery.md#working-with-orphaned-objects). For information on imperative recovery see [*the section called “Imperative Recovery”*](#imperative-recovery).

### Client Failure

Recovery from client failure in a Lustre file system is based on lock revocation and other resources, so surviving clients can continue their work uninterrupted. If a client fails to timely respond to a blocking lock callback from the Distributed Lock Manager (DLM) or fails to communicate with the server in a long period of time (i.e., no pings), the client is forcibly removed from the cluster (evicted). This enables other clients to acquire locks blocked by the dead client's locks, and also frees resources (file handles, export data) associated with that client. Note that this scenario can be caused by a network partition, as well as an actual client node system failure. [*the section called “Network Partition”*](#network-partition) describes this case in more detail.

### Client Eviction

If a client is not behaving properly from the server's point of view, it will be evicted. This ensures that the whole file system can continue to function in the presence of failed or misbehaving clients. An evicted client must invalidate all locks, which in turn, results in all cached inodes becoming invalidated and all cached data being flushed.

Reasons why a client might be evicted:

- Failure to respond to a server request in a timely manner
  - Blocking lock callback (i.e., client holds lock that another client/server wants)
  - Lock completion callback (i.e., client is granted lock previously held by another client)
  - Lock glimpse callback (i.e., client is asked for size of object by another client)
  - Server shutdown notification (with simplified interoperability)
- Failure to ping the server in a timely manner, unless the server is receiving no RPC traffic at all (which may indicate a network partition).

### MDS Failure (Failover)

Highly-available (HA) Lustre file system operation requires that the metadata server have a peer configured for failover, including the use of a shared storage device for the MDT backing file system. The actual mechanism for detecting peer failure, power off (STONITH) of the failed peer (to prevent it from continuing to modify the shared disk), and takeover of the Lustre MDS service on the backup node depends on external HA software such as Heartbeat. It is also possible to have MDS recovery with a single MDS node. In this case, recovery will take as long as is needed for the single MDS to be restarted.

When [*the section called “Imperative Recovery”*](#imperative-recovery) is enabled, clients are notified of an MDS restart (either the backup or a restored primary). Clients always may detect an MDS failure either by timeouts of in-flight requests or idle-time ping messages. In either case the clients then connect to the new backup MDS and use the Metadata Replay protocol. Metadata Replay is responsible for ensuring that the backup MDS re-acquires state resulting from transactions whose effects were made visible to clients, but which were not committed to the disk.

The reconnection to a new (or restarted) MDS is managed by the file system configuration loaded by the client when the file system is first mounted. If a failover MDS has been configured (using the `--failnode=` option to `mkfs.lustre` or `tunefs.lustre`), the client tries to reconnect to both the primary and backup MDS until one of them responds that the failed MDT is again available. At that point, the client begins recovery. For more information, see [*the section called “Metadata Replay”*](#metadata-replay).

Transaction numbers are used to ensure that operations are replayed in the order they were originally performed, so that they are guaranteed to succeed and present the same file system state as before the failure. In addition, clients inform the new server of their existing lock state (including locks that have not yet been granted). All metadata and lock replay must complete before new, non-recovery operations are permitted. In addition, only clients that were connected at the time of MDS failure are permitted to reconnect during the recovery window, to avoid the introduction of state changes that might conflict with what is being replayed by previously-connected clients.

If multiple MDTs are in use, active-active failover is possible (e.g. two MDS nodes, each actively serving one or more different MDTs for the same filesystem). See [*the section called “ MDT Failover Configuration (Active/Active)”*](02-Introducing%20the%20Lustre%20File%20System.md#mdt-failover-configuration-activepassive) for more information.

### OST Failure (Failover)

When an OST fails or has communication problems with the client, the default action is that the corresponding OSC enters recovery, and I/O requests going to that OST are blocked waiting for OST recovery or failover. It is possible to administratively mark the OSC as *inactive* on the client, in which case file operations that involve the failed OST will return an IO error (`-EIO`). Otherwise, the application waits until the OST has recovered or the client process is interrupted (e.g. ,with *CTRL-C*).

The MDS (via the LOV) detects that an OST is unavailable and skips it when assigning objects to new files. When the OST is restarted or re-establishes communication with the MDS, the MDS and OST automatically perform orphan recovery to destroy any objects that belong to files that were deleted while the OST was unavailable. For more information, see [*Troubleshooting Recovery*](05.02-Troubleshooting%20Recovery.md) (Working with Orphaned Objects).

While the OSC to OST operation recovery protocol is the same as that between the MDC and MDT using the Metadata Replay protocol, typically the OST commits bulk write operations to disk synchronously and each reply indicates that the request is already committed and the data does not need to be saved for recovery. In some cases, the OST replies to the client before the operation is committed to disk (e.g. truncate, destroy, setattr, and I/O operations in newer releases of the Lustre software), and normal replay and resend handling is done, including resending of the bulk writes. In this case, the client keeps a copy of the data available in memory until the server indicates that the write has committed to disk.

To force an OST recovery, unmount the OST and then mount it again. If the OST was connected to clients before it failed, then a recovery process starts after the remount, enabling clients to reconnect to the OST and replay transactions in their queue. When the OST is in recovery mode, all new client connections are refused until the recovery finishes. The recovery is complete when either all previously-connected clients reconnect and their transactions are replayed or a client connection attempt times out. If a connection attempt times out, then all clients waiting to reconnect (and their transactions) are lost.

**Note**

If you know an OST will not recover a previously-connected client (if, for example, the client has crashed), you can manually abort the recovery using this command:

```
oss# lctl --device lustre_device_number abort_recovery
```

To determine an OST's device number and device name, run the `lctl dl` command. Sample `lctl dl`command output is shown below:

```
7 UP obdfilter ddn_data-OST0009 ddn_data-OST0009_UUID 1159 
```

In this example, 7 is the OST device number. The device name is `ddn_data-OST0009`. In most instances, the device name can be used in place of the device number.

### Network Partition

Network failures may be transient. To avoid invoking recovery, the client tries, initially, to re-send any timed out request to the server. If the resend also fails, the client tries to re-establish a connection to the server. Clients can detect harmless partition upon reconnect if the server has not had any reason to evict the client.

If a request was processed by the server, but the reply was dropped (i.e., did not arrive back at the client), the server must reconstruct the reply when the client resends the request, rather than performing the same request twice.

### Failed Recovery

In the case of failed recovery, a client is evicted by the server and must reconnect after having flushed its saved state related to that server, as described in [*the section called “Client Eviction”*](#client-eviction), above. Failed recovery might occur for a number of reasons, including:

- Failure of recovery
  - Recovery fails if the operations of one client directly depend on the operations of another client that failed to participate in recovery. Otherwise, Version Based Recovery (VBR) allows recovery to proceed for all of the connected clients, and only missing clients are evicted.
  - Manual abort of recovery
- Manual eviction by the administrator

## Metadata Replay

Highly available Lustre file system operation requires that the MDS have a peer configured for failover, including the use of a shared storage device for the MDS backing file system. When a client detects an MDS failure, it connects to the new MDS and uses the metadata replay protocol to replay its requests.

Metadata replay ensures that the failover MDS re-accumulates state resulting from transactions whose effects were made visible to clients, but which were not committed to the disk.

### XID Numbers

Each request sent by the client contains an XID number, which is a client-unique, monotonically increasing 64-bit integer. The initial value of the XID is chosen so that it is highly unlikely that the same client node reconnecting to the same server after a reboot would have the same XID sequence. The XID is used by the client to order all of the requests that it sends, until such a time that the request is assigned a transaction number. The XID is also used in Reply Reconstruction to uniquely identify per-client requests at the server.

### Transaction Numbers

Each client request processed by the server that involves any state change (metadata update, file open, write, etc., depending on server type) is assigned a transaction number by the server that is a target-unique, monotonically increasing, server-wide 64-bit integer. The transaction number for each file system-modifying request is sent back to the client along with the reply to that client request. The transaction numbers allow the client and server to unambiguously order every modification to the file system in case recovery is needed.

Each reply sent to a client (regardless of request type) also contains the last committed transaction number that indicates the highest transaction number committed to the file system. The `ldiskfs` and `ZFS` backing file systems that the Lustre software uses enforces the requirement that any earlier disk operation will always be committed to disk before a later disk operation, so the last committed transaction number also reports that any requests with a lower transaction number have been committed to disk.

### Replay and Resend

Lustre file system recovery can be separated into two distinct types of operations: *replay* and *resend*.

*Replay* operations are those for which the client received a reply from the server that the operation had been successfully completed. These operations need to be redone in exactly the same manner after a server restart as had been reported before the server failed. Replay can only happen if the server failed; otherwise it will not have lost any state in memory.

*Resend* operations are those for which the client never received a reply, so their final state is unknown to the client. The client sends unanswered requests to the server again in XID order, and again awaits a reply for each one. In some cases, resent requests have been handled and committed to disk by the server (possibly also having dependent operations committed), in which case, the server performs reply reconstruction for the lost reply. In other cases, the server did not receive the lost request at all and processing proceeds as with any normal request. These are what happen in the case of a network interruption. It is also possible that the server received the request, but was unable to reply or commit it to disk before failure.

### Client Replay List

All file system-modifying requests have the potential to be required for server state recovery (replay) in case of a server failure. Replies that have an assigned transaction number that is higher than the last committed transaction number received in any reply from each server are preserved for later replay in a per-server replay list. As each reply is received from the server, it is checked to see if it has a higher last committed transaction number than the previous highest last committed number. Most requests that now have a lower transaction number can safely be removed from the replay list. One exception to this rule is for open requests, which need to be saved for replay until the file is closed so that the MDS can properly reference count open-unlinked files.

### Server Recovery

A server enters recovery if it was not shut down cleanly. If, upon startup, if any client entries are in the `last_rcvd`file for any previously connected clients, the server enters recovery mode and waits for these previously-connected clients to reconnect and begin replaying or resending their requests. This allows the server to recreate state that was exposed to clients (a request that completed successfully) but was not committed to disk before failure.

In the absence of any client connection attempts, the server waits indefinitely for the clients to reconnect. This is intended to handle the case where the server has a network problem and clients are unable to reconnect and/or if the server needs to be restarted repeatedly to resolve some problem with hardware or software. Once the server detects client connection attempts - either new clients or previously-connected clients - a recovery timer starts and forces recovery to finish in a finite time regardless of whether the previously-connected clients are available or not.

If no client entries are present in the `last_rcvd` file, or if the administrator manually aborts recovery, the server does not wait for client reconnection and proceeds to allow all clients to connect.

As clients connect, the server gathers information from each one to determine how long the recovery needs to take. Each client reports its connection UUID, and the server does a lookup for this UUID in the `last_rcvd` file to determine if this client was previously connected. If not, the client is refused connection and it will retry until recovery is completed. Each client reports its last seen transaction, so the server knows when all transactions have been replayed. The client also reports the amount of time that it was previously waiting for request completion so that the server can estimate how long some clients might need to detect the server failure and reconnect.

If the client times out during replay, it attempts to reconnect. If the client is unable to reconnect, `REPLAY` fails and it returns to `DISCON` state. It is possible that clients will timeout frequently during `REPLAY`, so reconnection should not delay an already slow process more than necessary. We can mitigate this by increasing the timeout during replay.

### Request Replay

If a client was previously connected, it gets a response from the server telling it that the server is in recovery and what the last committed transaction number on disk is. The client can then iterate through its replay list and use this last committed transaction number to prune any previously-committed requests. It replays any newer requests to the server in transaction number order, one at a time, waiting for a reply from the server before replaying the next request.

Open requests that are on the replay list may have a transaction number lower than the server's last committed transaction number. The server processes those open requests immediately. The server then processes replayed requests from all of the clients in transaction number order, starting at the last committed transaction number to ensure that the state is updated on disk in exactly the same manner as it was before the crash. As each replayed request is processed, the last committed transaction is incremented. If the server receives a replay request from a client that is higher than the current last committed transaction, that request is put aside until other clients provide the intervening transactions. In this manner, the server replays requests in the same sequence as they were previously executed on the server until either all clients are out of requests to replay or there is a gap in a sequence.

### Gaps in the Replay Sequence

In some cases, a gap may occur in the reply sequence. This might be caused by lost replies, where the request was processed and committed to disk but the reply was not received by the client. It can also be caused by clients missing from recovery due to partial network failure or client death.

In the case where all clients have reconnected, but there is a gap in the replay sequence the only possibility is that some requests were processed by the server but the reply was lost. Since the client must still have these requests in its resend list, they are processed after recovery is finished.

In the case where all clients have not reconnected, it is likely that the failed clients had requests that will no longer be replayed. The VBR feature is used to determine if a request following a transaction gap is safe to be replayed. Each item in the file system (MDS inode or OST object) stores on disk the number of the last transaction in which it was modified. Each reply from the server contains the previous version number of the objects that it affects. During VBR replay, the server matches the previous version numbers in the resend request against the current version number. If the versions match, the request is the next one that affects the object and can be safely replayed. For more information, see [*the section called “Version-based Recovery”*](#version-based-recovery).

### Lock Recovery

If all requests were replayed successfully and all clients reconnected, clients then do lock replay locks -- that is, every client sends information about every lock it holds from this server and its state (whenever it was granted or not, what mode, what properties and so on), and then recovery completes successfully. Currently, the Lustre software does not do lock verification and just trusts clients to present an accurate lock state. This does not impart any security concerns since Lustre software release 1.x clients are trusted for other information (e.g. user ID) during normal operation also.

After all of the saved requests and locks have been replayed, the client sends an `MDS_GETSTATUS` request with last-replay flag set. The reply to that request is held back until all clients have completed replay (sent the same flagged getstatus request), so that clients don't send non-recovery requests before recovery is complete.

### Request Resend

Once all of the previously-shared state has been recovered on the server (the target file system is up-to-date with client cache and the server has recreated locks representing the locks held by the client), the client can resend any requests that did not receive an earlier reply. This processing is done like normal request processing, and, in some cases, the server may do reply reconstruction.

## Reply Reconstruction

When a reply is dropped, the MDS needs to be able to reconstruct the reply when the original request is re-sent. This must be done without repeating any non-idempotent operations, while preserving the integrity of the locking system. In the event of MDS failover, the information used to reconstruct the reply must be serialized on the disk in transactions that are joined or nested with those operating on the disk.

### Required State

For the majority of requests, it is sufficient for the server to store three pieces of data in the `last_rcvd` file:

- XID of the request
- Resulting transno (if any)
- Result code (`req->rq_status`)

For open requests, the "disposition" of the open must also be stored.

### Reconstruction of Open Replies

An open reply consists of up to three pieces of information (in addition to the contents of the "request log"):

- File handle
- Lock handle
- `mds_body` with information about the file created (for `O_CREAT`)

The disposition, status and request data (re-sent intact by the client) are sufficient to determine which type of lock handle was granted, whether an open file handle was created, and which resource should be described in the `mds_body`.

#### Finding the File Handle

The file handle can be found in the XID of the request and the list of per-export open file handles. The file handle contains the resource/FID.

#### Finding the Resource/fid

The file handle contains the resource/fid.

#### Finding the Lock Handle

The lock handle can be found by walking the list of granted locks for the resource looking for one with the appropriate remote file handle (present in the re-sent request). Verify that the lock has the right mode (determined by performing the disposition/request/status analysis above) and is granted to the proper client.

Introduced in Lustre 2.8

### Multiple Reply Data per Client

Since Lustre 2.8, the MDS is able to save several reply data per client. The reply data are stored in the `reply_data`internal file of the MDT. Additionally to the XID of the request, the transaction number, the result code and the open "disposition", the reply data contains a generation number that identifies the client thanks to the content of the `last_rcvd` file.

## Version-based Recovery

The Version-based Recovery (VBR) feature improves Lustre file system reliability in cases where client requests (RPCs) fail to replay during recover[1].

In pre-VBR releases of the Lustre software, if the MGS or an OST went down and then recovered, a recovery process was triggered in which clients attempted to replay their requests. Clients were only allowed to replay RPCs in serial order. If a particular client could not replay its requests, then those requests were lost as well as the requests of clients later in the sequence. The ''downstream'' clients never got to replay their requests because of the wait on the earlier client's RPCs. Eventually, the recovery period would time out (so the component could accept new requests), leaving some number of clients evicted and their requests and data lost.

With VBR, the recovery mechanism does not result in the loss of clients or their data, because changes in inode versions are tracked, and more clients are able to reintegrate into the cluster. With VBR, inode tracking looks like this:

- Each inod[2]e stores a version, that is, the number of the last transaction (transno) in which the inode was changed.
- When an inode is about to be changed, a pre-operation version of the inode is saved in the client's data.
- The client keeps the pre-operation inode version and the post-operation version (transaction number) for replay, and sends them in the event of a server failure.
- If the pre-operation version matches, then the request is replayed. The post-operation version is assigned on all inodes modified in the request.

**Note**

An RPC can contain up to four pre-operation versions, because several inodes can be involved in an operation. In the case of a ''rename'' operation, four different inodes can be modified.

During normal operation, the server:

- Updates the versions of all inodes involved in a given operation
- Returns the old and new inode versions to the client with the reply

When the recovery mechanism is underway, VBR follows these steps:

1. VBR only allows clients to replay transactions if the affected inodes have the same version as during the original execution of the transactions, even if there is gap in transactions due to a missed client.
2. The server attempts to execute every transaction that the client offers, even if it encounters a re-integration failure.
3. When the replay is complete, the client and server check if a replay failed on any transaction because of inode version mismatch. If the versions match, the client gets a successful re-integration message. If the versions do not match, then the client is evicted.

VBR recovery is fully transparent to users. It may lead to slightly longer recovery times if the cluster loses several clients during server recovery.

----------------------------------------------------------------------------------------------------------------------------------------

[1] There are two scenarios under which client RPCs are not replayed: (1) Non-functioning or isolated clients do not reconnect, and they cannot replay their RPCs, causing a gap in the replay sequence. These clients get errors and are evicted. (2) Functioning clients connect, but they cannot replay some or all of their RPCs that occurred after the gap caused by the non-functioning/isolated clients. These clients get errors (caused by the failed clients). With VBR, these requests have a better chance to replay because the "gaps" are only related to specific files that the missing client(s) changed.

[2] Usually, there are two inodes, a parent and a child.

----------------------------------------------------------------------------------------------------------------------------------------

### VBR Messages

The VBR feature is built into the Lustre file system recovery functionality. It cannot be disabled. These are some VBR messages that may be displayed:

```
DEBUG_REQ(D_WARNING, req, "Version mismatch during replay\n");
```

This message indicates why the client was evicted. No action is needed.

```
CWARN("%s: version recovery fails, reconnecting\n");
```

This message indicates why the recovery failed. No action is needed.

### Tips for Using VBR

VBR will be successful for clients which do not share data with other client. Therefore, the strategy for reliable use of VBR is to store a client's data in its own directory, where possible. VBR can recover these clients, even if other clients are lost.

## Commit on Share

The commit-on-share (COS) feature makes Lustre file system recovery more reliable by preventing missing clients from causing cascading evictions of other clients. With COS enabled, if some Lustre clients miss the recovery window after a reboot or a server failure, the remaining clients are not evicted.

**Note**

The commit-on-share feature is enabled, by default.

### Working with Commit on Share

To illustrate how COS works, let's first look at the old recovery scenario. After a service restart, the MDS would boot and enter recovery mode. Clients began reconnecting and replaying their uncommitted transactions. Clients could replay transactions independently as long as their transactions did not depend on each other (one client's transactions did not depend on a different client's transactions). The MDS is able to determine whether one transaction is dependent on another transaction via the [*the section called “Version-based Recovery”*](#version-based-recovery) feature.

If there was a dependency between client transactions (for example, creating and deleting the same file), and one or more clients did not reconnect in time, then some clients may have been evicted because their transactions depended on transactions from the missing clients. Evictions of those clients caused more clients to be evicted and so on, resulting in "cascading" client evictions.

COS addresses the problem of cascading evictions by eliminating dependent transactions between clients. It ensures that one transaction is committed to disk if another client performs a transaction dependent on the first one. With no dependent, uncommitted transactions to apply, the clients replay their requests independently without the risk of being evicted.

### Tuning Commit On Share

Commit on Share can be enabled or disabled using the `mdt.commit_on_sharing` tunable (0/1). This tunable can be set when the MDS is created (`mkfs.lustre`) or when the Lustre file system is active, using the `lctl set/get_param` or `lctl conf_param` commands.

To set a default value for COS (disable/enable) when the file system is created, use:

```
--param mdt.commit_on_sharing=0/1
```

To disable or enable COS when the file system is running, use:

```
lctl set_param mdt.*.commit_on_sharing=0/1
```

**Note**

Enabling COS may cause the MDS to do a large number of synchronous disk operations, hurting performance. Placing the `ldiskfs` journal on a low-latency external device may improve file system performance.

## Imperative Recovery

Large-scale Lustre filesystems will experience server hardware failures over their lifetime, and it is important that servers can recover in a timely manner after such failures. High Availability software can move storage targets over to a backup server automatically. Clients can detect the server failure by RPC timeouts, which must be scaled with system size to prevent false diagnosis of server death in cases of heavy load. The purpose of imperative recovery is to reduce the recovery window by actively informing clients of server failure. The resulting reduction in the recovery window will minimize target downtime and therefore increase overall system availability.

Imperative Recovery does not remove previous recovery mechanisms, and client timeout-based recovery actions can occur in a cluster when IR is enabled as each client can still independently disconnect and reconnect from a target. In case of a mix of IR and non-IR clients connecting to an OST or MDT, the server cannot reduce its recovery timeout window, because it cannot be sure that all clients have been notified of the server restart in a timely manner. Even in such mixed environments the time to complete recovery may be reduced, since IR-enabled clients will still be notified to reconnect to the server promptly and allow recovery to complete as soon as the last non-IR client detects the server failure.

### MGS role

The MGS now holds additional information about Lustre targets, in the form of a Target Status Table. Whenever a target registers with the MGS, there is a corresponding entry in this table identifying the target. This entry includes NID information, and state/version information for the target. When a client mounts the file system, it caches a locked copy of this table, in the form of a Lustre configuration log. When a target restart occurs, the MGS revokes the client lock, forcing all clients to reload the table. Any new targets will have an updated version number, the client detects this and reconnects to the restarted target. Since successful IR notification of server restart depends on all clients being registered with the MGS, and there is no other node to notify clients in case of MGS restart, the MGS will disable IR for a period when it first starts. This interval is configurable, as shown in [*the section called “Tuning Imperative Recovery”*](#tuning-imperative-recovery).

Because of the increased importance of the MGS in recovery, it is strongly recommended that the MGS node be separate from the MDS. If the MGS is co-located on the MDS node, then in case of MDS/MGS failure there will be no IR notification for the MDS restart, and clients will always use timeout-based recovery for the MDS. IR notification would still be used in the case of OSS failure and recovery.

Unfortunately, it’s impossible for the MGS to know how many clients have been successfully notified or whether a specific client has received the restarting target information. The only thing the MGS can do is tell the target that, for example, all clients are imperative recovery-capable, so it is not necessary to wait as long for all clients to reconnect. For this reason, we still require a timeout policy on the target side, but this timeout value can be much shorter than normal recovery.

### Tuning Imperative Recovery

Imperative recovery has a default parameter set which means it can work without any extra configuration. However, the default parameter set only fits a generic configuration. The following sections discuss the configuration items for imperative recovery.

#### ir_factor

Ir_factor is used to control targets’ recovery window. If imperative recovery is enabled, the recovery timeout window on the restarting target is calculated by: *new timeout = recovery_time \* ir_factor / 10* Ir_factor must be a value in range of [1, 10]. The default value of ir_factor is 5. The following example will set imperative recovery timeout to 80% of normal recovery timeout on the target testfs-OST0000:

```
lctl conf_param obdfilter.testfs-OST0000.ir_factor=8
```

**Note**

If this value is too small for the system, clients may be unnecessarily evicted

You can read the current value of the parameter in the standard manner with *lctl get_param*:

```
# lctl get_param obdfilter.testfs-OST0000.ir_factor
# obdfilter.testfs-OST0000.ir_factor=8 
```

#### Disabling Imperative Recovery

Imperative recovery can be disabled manually by a mount option. For example, imperative recovery can be disabled on an OST by:

```
# mount -t lustre -onoir /dev/sda /mnt/ost1
```

Imperative recovery can also be disabled on the client side with the same mount option:

```
# mount -t lustre -onoir mymgsnid@tcp:/testfs /mnt/testfs
```

**Note**

When a single client is deactivated in this manner, the MGS will deactivate imperative recovery for the whole cluster. IR-enabled clients will still get notification of target restart, but targets will not be allowed to shorten the recovery window.

You can also disable imperative recovery globally on the MGS by writing `state=disabled’ to the controlling procfs entry

```
# lctl set_param mgs.MGS.live.testfs="state=disabled"
```

The above command will disable imperative recovery for file system named *testfs*.

#### Checking Imperative Recovery State - MGS

You can get the imperative recovery state from the MGS. Let’s take an example and explain states of imperative recovery:

```
[mgs]$ lctl get_param mgs.MGS.live.testfs
...
imperative_recovery_state:
    state: full
    nonir_clients: 0
    nidtbl_version: 242
    notify_duration_total: 0.470000
    notify_duation_max: 0.041000
    notify_count: 38
```

| **Item**                  | **Meaning**                                                  |
| ------------------------- | ------------------------------------------------------------ |
| **state**                 | **full:** IR is working, all clients are connected and can be notified.**partial:** some clients are not IR capable.**disabled:** IR is disabled, no client notification.**startup:** the MGS was just restarted, so not all clients may reconnect to the MGS. |
| **nonir_clients**         | Number of non-IR capable clients in the system.              |
| **nidtbl_version**        | Version number of the target status table. Client version must match MGS. |
| **notify_duration_total** | [Seconds.microseconds] Total time spent by MGS notifying clients |
| **notify_duration_max**   | [Seconds.microseconds] Maximum notification time for the MGS to notify a single IR client. |
| **notify_count**          | Number of MGS restarts - to obtain average notification time, divide `notify_duration_total` by `notify_count` |

#### Checking Imperative Recovery State - client

A `client’ in IR means a Lustre client or a MDT. You can get the IR state on any node which running client or MDT, those nodes will always have an MGC running. An example from a client:

```
[client]$ lctl get_param mgc.*.ir_state
mgc.MGC192.168.127.6@tcp.ir_state=
imperative_recovery: ON
client_state:
    - { client: testfs-client, nidtbl_version: 242 }
	
```

An example from a MDT:

```
mgc.MGC192.168.127.6@tcp.ir_state=
imperative_recovery: ON
client_state:
    - { client: testfs-MDT0000, nidtbl_version: 242 }
	
```

| **Item**                         | **Meaning**                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| **imperative_recovery**          | `imperative_recovery`can be ON or OFF. If it’s OFF state, then IR is disabled by administrator at mount time. Normally this should be ON state. |
| **client_state: client:**        | The name of the client                                       |
| **client_state: nidtbl_version** | Version number of the target status table. Client version must match MGS. |

#### Target Instance Number

The Target Instance number is used to determine if a client is connecting to the latest instance of a target. We use the lowest 32 bit of mount count as target instance number. For an OST you can get the target instance number of testfs-OST0001 in this way (the command is run from an OSS login prompt):

```
$ lctl get_param obdfilter.testfs-OST0001*.instance
obdfilter.testfs-OST0001.instance=5
```

From a client, query the relevant OSC:

```
$ lctl get_param osc.testfs-OST0001-osc-*.import |grep instance
    instance: 5
```

### Configuration Suggestions for Imperative Recovery

We used to build the MGS and MDT0000 on the same target to save a server node. However, to make IR work efficiently, we strongly recommend running the MGS node on a separate node for any significant Lustre file system installation. There are three main advantages of doing this:

1. Be able to notify clients when MDT0000 recovered.
2. Improved load balance. The load on the MDS may be very high which may make the MGS unable to notify the clients in time.
3. Robustness. The MGS code is simpler and much smaller compared to the MDS code. This means the chance of an MGS downtime due to a software bug is very low.

## Suppressing Pings

On clusters with large numbers of clients and OSTs, OBD_PING messages may impose significant performance overheads. There is an option to suppress pings, allowing ping overheads to be considerably reduced. Before turning on this option, administrators should consider the following requirements and understand the trade-offs involved:

- When suppressing pings, a server cannot detect client deaths, since clients do not send pings that are only to keep their connections alive. Therefore, a mechanism external to the Lustre file system shall be set up to notify Lustre targets of client deaths in a timely manner, so that stale connections do not exist for too long and lock callbacks to dead clients do not always have to wait for timeouts.
- Without pings, a client has to rely on Imperative Recovery to notify it of target failures, in order to join recoveries in time. This dictates that the client shall eargerly keep its MGS connection alive. Thus, a highly available standalone MGS is recommended and, on the other hand, MGS pings are always sent regardless of how the option is set.
- If a client has uncommitted requests to a target and it is not sending any new requests on the connection, it will still ping that target even when pings should be suppressed. This is because the client needs to query the target's last committed transaction numbers in order to free up local uncommitted requests (and possibly other resources associated). However, these pings shall stop as soon as all the uncommitted requests have been freed or new requests need to be sent, rendering the pings unnecessary.

### "suppress_pings" Kernel Module Parameter

The new option that controls whether pings are suppressed is implemented as the ptlrpc kernel module parameter "suppress_pings". Setting it to "1" on a server turns on ping suppressing for all targets on that server, while leaving it with the default value "0" gives previous pinging behavior. The parameter is ignored on clients and the MGS. While the parameter is recommended to be set persistently via the modprobe.conf(5) mechanism, it also accept online changes through sysfs. Note that an online change only affects connections established later; existing connections' pinging behaviors stay the same.

### Client Death Notification

The required external client death notification shall write UUIDs of dead clients into targets' "evict_client" procfs entries like

```
/proc/fs/lustre/obdfilter/testfs-OST0000/evict_client
/proc/fs/lustre/obdfilter/testfs-OST0001/evict_client
/proc/fs/lustre/mdt/testfs-MDT0000/evict_client
      
```

Clients' UUIDs can be obtained from their "uuid" procfs entries like

```
/proc/fs/lustre/llite/testfs-ffff8800612bf800/uuid
```
