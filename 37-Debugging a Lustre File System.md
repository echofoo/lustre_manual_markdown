# Debugging a Lustre File System

- [Debugging a Lustre File System](#debugging-a-lustre-file-system)
  * [Diagnostic and Debugging Tools](#diagnostic-and-debugging-tools)
    + [Lustre Debugging Tools](#lustre-debugging-tools)
    + [External Debugging Tools](#external-debugging-tools)
      - [Tools for Administrators and Developers](#tools-for-administrators-and-developers)
      - [Tools for Developers](#tools-for-developers)
  * [Lustre Debugging Procedures](#lustre-debugging-procedures)
    + [Understanding the Lustre Debug Messaging Format](#understanding-the-lustre-debug-messaging-format)
      - [Lustre Debug Messages](#lustre-debug-messages)
      - [Format of Lustre Debug Messages](#format-of-lustre-debug-messages)
      - [Lustre Debug Messages Buffer](#lustre-debug-messages-buffer)
    + [Using the lctl Tool to View Debug Messages](#using-the-lctl-tool-to-view-debug-messages)
      - [Sample `lctl` Run](#sample-lctl-run)
    + [Dumping the Buffer to a File (`debug_daemon`)](#dumping-the-buffer-to-a-file-debug_daemon)
      - [`lctl debug_daemon` Commands](#lctl-debug_daemon-commands)
    + [Controlling Information Written to the Kernel Debug Log](#controlling-information-written-to-the-kernel-debug-log)
    + [Troubleshooting with `strace`](#troubleshooting-with-strace)
    + [Looking at Disk Content](#looking-at-disk-content)
    + [Finding the Lustre UUID of an OST](#finding-the-lustre-uuid-of-an-ost)
    + [Printing Debug Messages to the Console](#printing-debug-messages-to-the-console)
    + [Tracing Lock Traffic](#tracing-lock-traffic)
    + [Controlling Console Message Rate Limiting](#controlling-console-message-rate-limiting)
  * [Lustre Debugging for Developers](#lustre-debugging-for-developers)
    + [Adding Debugging to the Lustre Source Code](#adding-debugging-to-the-lustre-source-code)
    + [Accessing the `ptlrpc` Request History](#accessing-the-ptlrpc-request-history)
    + [Finding Memory Leaks Using `leak_finder.pl`](#finding-memory-leaks-using-leak_finderpl)


This chapter describes tips and information to debug a Lustre file system, and includes the following sections:

- [the section called “ Diagnostic and Debugging Tools”](#diagnostic-and-debugging-tools)
- [the section called “Lustre Debugging Procedures”](#lustre-debugging-procedures)
- [the section called “Lustre Debugging for Developers”](#lustre-debugging-for-developers)

## Diagnostic and Debugging Tools

A variety of diagnostic and analysis tools are available to debug issues with the Lustre software. Some of these are provided in Linux distributions, while others have been developed and are made available by the Lustre project.

### Lustre Debugging Tools

The following in-kernel debug mechanisms are incorporated into the Lustre software:

- **Debug logs** - A circular debug buffer to which Lustre internal debug messages are written (in contrast to error messages, which are printed to the syslog or console). Entries in the Lustre debug log are controlled by a mask set by `lctl set_param debug=*mask*`. The log size defaults to 5 MB per CPU but can be increased as a busy system will quickly overwrite 5 MB. When the buffer fills, the oldest log records are discarded.
- **lctl get_param debug** - This shows the current debug mask used to delimit the debugging information written out to the kernel debug logs.
- **lctl debug_kernel file** - Dump the Lustre kernel debug log to the specified file as ASCII text for further debugging and analysis.
- **lctl set_param debug_mb=size** - This sets the maximum size of the in-kernel Lustre debug buffer, in units of MiB.
- **Debug daemon** - The debug daemon controls the continuous logging of debug messages to a log file in userspace.

The following tools are also provided with the Lustre software:

- **lctl** - This tool is used with the debug_kernel option to manually dump the Lustre debugging log or post-process debugging logs that are dumped automatically. For more information about the lctl tool, see [*the section called “Using the lctl Tool to View Debug Messages”*](#using-the-lctl-tool-to-view-debug-messages)and [*the section called “ lctl”*](06.07-System%20Configuration%20Utilities.md#lctl).
- Lustre subsystem asserts - A panic-style assertion (LBUG) in the kernel causes the Lustre file system to dump the debug log to the file `/tmp/lustre-log.*timestamp*` where it can be retrieved after a reboot. For more information, see [*the section called “Viewing Error Messages”*](05.01-Lustre%20File%20System%20Troubleshooting.md#viewing-error-messages).
- *`lfs `*- This utility provides access to the layout of of a Lustre file, along with other information relevant to users. For more information about lfs, see [*the section called “ `lfs` ”*](06.03-User%20Utilities.md#lfs).

### External Debugging Tools

The tools described in this section are provided in the Linux kernel or are available at an external website. For information about using some of these tools for Lustre debugging, see [*the section called Lustre Debugging Procedures”*](#lustre-debugging-procedures) and [*the section called “Lustre Debugging for Developers”*](#lustre-debugging-for-developers).

#### Tools for Administrators and Developers

Some general debugging tools provided as a part of the standard Linux distribution are:

- **strace** . This tool allows a system call to be traced.
- **/var/log/messages** . `syslogd` prints fatal or serious messages at this log.
- **Crash dumps** . On crash-dump enabled kernels, sysrq c produces a crash dump. The Lustre software enhances this crash dump with a log dump (the last 64 KB of the log) to the console.
- **debugfs** . Interactive file system debugger.

The following logging and data collection tools can be used to collect information for debugging Lustre kernel issues:

- **kdump** . A Linux kernel crash utility useful for debugging a system running Red Hat Enterprise Linux. For more information about kdump, see the Red Hat knowledge base article How to troubleshoot kernel crashes, hangs, or reboots with kdump on [Red Hat Enterprise Linux](https://access.redhat.com/solutions/6038). To download `kdump`, install the RPM package via yum install kexec-tools.
- **netconsole** . Enables kernel-level network logging over UDP. A system requires (SysRq) allows users to collect relevant data through `netconsole`.
- *wireshark* . A network packet inspection tool that allows debugging of information that was sent between the various Lustre nodes. This tool is built on top of tcpdump and can read packet dumps generated by it. There are plug-ins available to dissassemble the LNet and Lustre protocols. They are included with wireshark since version 2.6.0. See also [Wireshark Website](http://www.wireshark.org/) for more details.

#### Tools for Developers

The tools described in this section may be useful for debugging a Lustre file system in a development environment.

Of general interest is:

- `*leak_finder.pl* `. This program provided with the Lustre software is useful for finding memory leaks in the code.

A virtual machine is often used to create an isolated development and test environment. Some commonly-used virtual machines are:

- **VirtualBox Open Source Edition** . Provides enterprise-class virtualization capability for all major platforms and is available free at [Get Sun VirtualBox](https:// www.virtualbox.org/wiki/Downloads).
- **VMware Server** . Virtualization platform available as free introductory software at [Download VMware Server](https://my.vmware.com/web/vmware/downloads/).
- **Xen**. A para-virtualized environment with virtualization capabilities similar to VMware Server and Virtual Box. However, Xen allows the use of modified kernels to provide near-native performance and the ability to emulate shared storage. For more information, go to [xen.org](https://xen.org/).

A variety of debuggers and analysis tools are available including:

- **kgdb** . The Linux Kernel Source Level Debugger kgdb is used in conjunction with the GNU Debugger `gdb` for debugging the Linux kernel. For more information about using `kgdb` with `gdb`, see [Chapter 6. Running Programs Under gdb](https://www.linuxtopia.org/online_books/redhat_linux_debugging_with_gdb/running.html) in the *Red Hat Linux 4 Debugging with GDB* guide.
- **crash** . Used to analyze saved crash dump data when a system had panicked or locked up or appears unresponsive. For more information about using crash to analyze a crash dump, see:
  - Overview on how to use crash by the author: White Paper: [Red Hat Crash Utility](https://crashutility.github.io/crash_whitepaper.html)

## Lustre Debugging Procedures

The procedures below may be useful to administrators or developers debugging a Lustre files system.

### Understanding the Lustre Debug Messaging Format

Lustre debug messages are categorized by originating subsystem, message type, and location in the source code. For a list of subsystems and message types, see [*the section called “Lustre Debug Messages”*](#lustre-debug-messages).

**Note**

For a current list of subsystems and debug message types, see`libcfs/include/libcfs/libcfs_debug.h` in the Lustre software tree

The elements of a Lustre debug message are described in [*the section called “Format of Lustre Debug Messages”*](#format-of-lustre-debug-messages) Format of Lustre Debug Messages.

#### Lustre Debug Messages

Each Lustre debug message has the tag of the subsystem it originated in, the message type, and the location in the source code. The subsystems and debug types used are as follows:

- Standard Subsystems:

  mdc, mds, osc, ost, obdclass, obdfilter, llite, ptlrpc, portals, lnd, ldlm, lov

- Debug Types:

- | **Types**    | **Description**                                              |
  | ------------ | ------------------------------------------------------------ |
  | **trace**    | Function entry/exit markers                                  |
  | **dlmtrace** | Distributed locking-related information                      |
  | **inode**    |                                                              |
  | **super**    |                                                              |
  | **malloc**   | Memory allocation or free information                        |
  | **cache**    | Cache-related information                                    |
  | **info**     | Non-critical general information                             |
  | **dentry**   | kernel namespace cache handling                              |
  | **mmap**     | Memory-mapped IO interface                                   |
  | **page**     | Page cache and bulk data transfers                           |
  | **info**     | Miscellaneous informational messages                         |
  | **net**      | LNet network related debugging                               |
  | **console**  | Significant system events, printed to console                |
  | **warning**  | Significant but non-fatal exceptions, printed to console     |
  | **error**    | Critical error messages, printed to console                  |
  | **neterror** | Significant LNet error messages                              |
  | **emerg**    | Fatal system errors, printed to console                      |
  | **config**   | Configuration and setup, enabled by default                  |
  | **ha**       | Failover and recovery-related information, enabled by default |
  | **hsm**      | Hierarchical space management/tiering                        |
  | **ioctl**    | IOCTL-related information, enabled by default                |
  | **layout**   | File layout handling (PFL, FLR, DoM)                         |
  | **lfsck**    | Filesystem consistency checking, enabled by default          |
  | **other**    | Miscellaneious other debug messages                          |
  | **quota**    | Space accounting and management                              |
  | **reada**    | Client readahead management                                  |
  | **rpctrace** | Remote request/reply tracing and debugging                   |
  | **sec**      | Security, Kerberos, Shared Secret Key handling               |
  | **snapshot** | Filesystem snapshot management                               |
  | **vfstrace** | Kernel VFS interface operations                              |

#### Format of Lustre Debug Messages

The Lustre software uses the `CDEBUG()` and `CERROR()` macros to print the debug or error messages. To print the message, the `CDEBUG()` macro uses the function `libcfs_debug_msg()` (`libcfs/libcfs/tracefile.c`). The message format is described below, along with an example.

| **Description**                   | **Parameter**                                   |
| --------------------------------- | ----------------------------------------------- |
| **subsystem**                     | 800000                                          |
| **debug mask**                    | 000010                                          |
| **smp_processor_id**              | 0                                               |
| **seconds.microseconds**          | 1081880847.677302                               |
| **stack size**                    | 1204                                            |
| **pid**                           | 2973                                            |
| **host pid (UML only) or zero**   | 31070                                           |
| **(file:line #:function_name())** | (obd_mount.c:2089:lustre_fill_super())          |
| **debug message**                 | kmalloced '*obj': 24 at a375571c (tot 17447717) |

#### Lustre Debug Messages Buffer

Lustre debug messages are maintained in a buffer, with the maximum buffer size specified (in MBs) by the `debug_mb` parameter (`lctl get_param debug_mb`). The buffer is circular, so debug messages are kept until the allocated buffer limit is reached, and then the first messages are overwritten.

### Using the lctl Tool to View Debug Messages

The `lctl` tool allows debug messages to be filtered based on subsystems and message types to extract information useful for troubleshooting from a kernel debug log. For a command reference, see [*the section called “ lctl”*](06.07-System%20Configuration%20Utilities.md#lctl).

You can use `lctl` to:

- Obtain a list of all the types and subsystems:

  ```
  lctl > debug_list subsystems|types
  ```

- Filter the debug log:

  ```
  lctl > filter subsystem_name|debug_type
  ```

**Note**

When `lctl` filters, it removes unwanted lines from the displayed output. This does not affect the contents of the debug log in the kernel's memory. As a result, you can print the log many times with different filtering levels without worrying about losing data.

- Show debug messages belonging to certain subsystem or type:

  ```
  lctl > show subsystem_name|debug_type
  ```

  `debug_kernel` pulls the data from the kernel logs, filters it appropriately, and displays or saves it as per the specified options

  ```
  lctl > debug_kernel [output filename]
  ```

  If the debugging is being done on User Mode Linux (UML), it might be useful to save the logs on the host machine so that they can be used at a later time.

- Filter a log on disk, if you already have a debug log saved to disk (likely from a crash):

  ```
  lctl > debug_file input_file [output_file] 
  ```

  During the debug session, you can add markers or breaks to the log for any reason:

  ```
  lctl > mark [marker text] 
  ```

  The marker text defaults to the current date and time in the debug log (similar to the example shown below):

  ```
  DEBUG MARKER: Tue Mar 5 16:06:44 EST 2002 
  ```

- Completely flush the kernel debug buffer:

  ```
  lctl > clear
  ```

**Note**

Debug messages displayed with `lctl` are also subject to the kernel debug masks; the filters are additive.

#### Sample `lctl` Run

Below is a sample run using the `lctl` command.

```
bash-2.04# ./lctl 
lctl > debug_kernel /tmp/lustre_logs/log_all 
Debug log: 324 lines, 324 kept, 0 dropped. 
lctl > filter trace 
Disabling output of type "trace" 
lctl > debug_kernel /tmp/lustre_logs/log_notrace 
Debug log: 324 lines, 282 kept, 42 dropped. 
lctl > show trace 
Enabling output of type "trace" 
lctl > filter portals 
Disabling output from subsystem "portals" 
lctl > debug_kernel /tmp/lustre_logs/log_noportals 
Debug log: 324 lines, 258 kept, 66 dropped. 
```

### Dumping the Buffer to a File (`debug_daemon`)

The `lctl debug_daemon` command is used to continuously dump the `debug_kernel` buffer to a user-specified file. This functionality uses a kernel thread to continuously dump the messages from the kernel debug log, so that much larger debug logs can be saved over a longer time than would fit in the kernel ringbuffer.

The `debug_daemon` is highly dependent on file system write speed. File system write operations may not be fast enough to flush out all of the `debug_buffer` if the Lustre file system is under heavy system load and continues to log debug messages to the `debug_buffer`. The `debug_daemon` will write the message `DEBUG MARKER: Trace buffer full` into the `debug_buffer` to indicate the `debug_buffer` contents are overlapping before the `debug_daemon`flushes data to a file.

Users can use the `lctl debug_daemon` command to start or stop the Lustre daemon from dumping the `debug_buffer` to a file.

#### `lctl debug_daemon` Commands

To initiate the `debug_daemon` to start dumping the `debug_buffer` into a file, run as the root user:

```
lctl debug_daemon start filename [megabytes]
```

The debug log will be written to the specified filename from the kernel. The file will be limited to the optionally specified number of megabytes.

The daemon wraps around and dumps data to the beginning of the file when the output file size is over the limit of the user-specified file size. To decode the dumped file to ASCII and sort the log entries by time, run:

```
lctl debug_file filename > newfile
```

The output is internally sorted by the `lctl` command.

To stop the `debug_daemon` operation and flush the file output, run:

```
lctl debug_daemon stop
```

Otherwise, `debug_daemon` is shut down as part of the Lustre file system shutdown process. Users can restart `debug_daemon` by using start command after each stop command issued.

This is an example using `debug_daemon` with the interactive mode of `lctl` to dump debug logs to a 40 MB file.

```
lctl
```

```
lctl > debug_daemon start /var/log/lustre.40.bin 40 
```

```
run filesystem operations to debug
```

```
lctl > debug_daemon stop 
```

```
lctl > debug_file /var/log/lustre.bin /var/log/lustre.log
```

To start another daemon with an unlimited file size, run:

```
lctl > debug_daemon start /var/log/lustre.bin 
```

The text message `*** End of debug_daemon trace log ***` appears at the end of each output file.

### Controlling Information Written to the Kernel Debug Log

The `lctl set_param subsystem_debug=*subsystem_mask*` and `lctl set_param debug=*debug_mask*` are used to determine which information is written to the debug log. The subsystem_debug mask determines the information written to the log based on the functional area of the code (such as lnet, osc, or ldlm). The debug mask controls information based on the message type (such as info, error, trace, or malloc). For a complete list of possible debug masks use the `lctl debug_list types` command.

To turn off Lustre debugging completely:

```
lctl set_param debug=0 
```

To turn on full Lustre debugging:

```
lctl set_param debug=-1 
```

To list all possible debug masks:

```
lctl debug_list types
```

To log only messages related to network communications:

```
lctl set_param debug=net 
```

To turn on logging of messages related to network communications and existing debug flags:

```
lctl set_param debug=+net 
```

To turn off network logging with changing existing flags:

```
lctl set_param debug=-net 
```

The various options available to print to kernel debug logs are listed in `libcfs/include/libcfs/libcfs.h`

### Troubleshooting with `strace`

The `strace` utility provided with the Linux distribution enables system calls to be traced by intercepting all the system calls made by a process and recording the system call name, arguments, and return values.

To invoke `strace` on a program, enter:

```
$ strace program [arguments] 
```

Sometimes, a system call may fork child processes. In this situation, use the `-f` option of `strace` to trace the child processes:

```
$ strace -f program [arguments] 
```

To redirect the `strace` output to a file, enter:

```
$ strace -o filename program [arguments] 
```

Use the `-ff` option, along with `-o`, to save the trace output in `filename.pid`, where `pid` is the process ID of the process being traced. Use the `-ttt` option to timestamp all lines in the strace output, so they can be correlated to operations in the lustre kernel debug log.

### Looking at Disk Content

In a Lustre file system, the inodes on the metadata server contain extended attributes (EAs) that store information about file striping. EAs contain a list of all object IDs and their locations (that is, the OST that stores them). The `lfs`tool can be used to obtain this information for a given file using the `getstripe` subcommand. Use a corresponding `lfs setstripe` command to specify striping attributes for a new file or directory.

The `lfs getstripe` command takes a Lustre filename as input and lists all the objects that form a part of this file. To obtain this information for the file `/mnt/testfs/frog` in a Lustre file system, run:

```
$ lfs getstripe /mnt/testfs/frog
lmm_stripe_count:   2
lmm_stripe_size:    1048576
lmm_pattern:        1
lmm_layout_gen:     0
lmm_stripe_offset:  2
        obdidx           objid          objid           group
             2          818855        0xc7ea7               0
             0          873123        0xd52a3               0
        
```

The `debugfs` tool is provided in the `e2fsprogs` package. It can be used for interactive debugging of an `ldiskfs` file system. The `debugfs` tool can either be used to check status or modify information in the file system. In a Lustre file system, all objects that belong to a file are stored in an underlying `ldiskfs` file system on the OSTs. The file system uses the object IDs as the file names. Once the object IDs are known, use the `debugfs` tool to obtain the attributes of all objects from different OSTs.

A sample run for the `/mnt/testfs/frog` file used in the above example is shown here:

```
$ debugfs -c -R "stat O/0/d$((818855 % 32))/818855" /dev/vgmyth/lvmythost2

debugfs 1.41.90.wc3 (28-May-2011)
/dev/vgmyth/lvmythost2: catastrophic mode - not reading inode or group bitmaps
Inode: 227649   Type: regular    Mode:  0666   Flags: 0x80000
Generation: 1375019198    Version: 0x0000002f:0000728f
User:  1000   Group:  1000   Size: 2800
File ACL: 0    Directory ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x4e177fe5:00000000 -- Fri Jul  8 16:08:37 2011
 atime: 0x4d2e2397:00000000 -- Wed Jan 12 14:56:39 2011
 mtime: 0x4e177fe5:00000000 -- Fri Jul  8 16:08:37 2011
crtime: 0x4c3b5820:a364117c -- Mon Jul 12 12:00:00 2010
Size of extra inode fields: 28
Extended attributes stored in inode body: 
  fid = "08 80 24 00 00 00 00 00 28 8a e7 fc 00 00 00 00 a7 7e 0c 00 00 00 00 00
 00 00 00 00 00 00 00 00 " (32)
  fid: objid=818855 seq=0 parent=[0x248008:0xfce78a28:0x0] stripe=0
EXTENTS:
(0):63331288
```

### Finding the Lustre UUID of an OST

To determine the Lustre UUID of an OST disk (for example, if you mix up the cables on your OST devices or the SCSI bus numbering suddenly changes and the SCSI devices get new names), it is possible to extract this from the last_rcvd file using debugfs:

```
debugfs -c -R "dump last_rcvd /tmp/last_rcvd" /dev/sdc
strings /tmp/last_rcvd | head -1
myth-OST0004_UUID
      
```

It is also possible (and easier) to extract this from the file system label using the `dumpe2fs` command:

```
dumpe2fs -h /dev/sdc | grep volume
dumpe2fs 1.41.90.wc3 (28-May-2011)
Filesystem volume name:   myth-OST0004
      
```

The debugfs and dumpe2fs commands are well documented in the `debugfs(8)` and `dumpe2fs(8)` manual pages.

### Printing Debug Messages to the Console

To dump debug messages to the console (`/var/log/messages`), set the corresponding debug mask in the `printk`flag:

```
lctl set_param printk=-1 
```

This slows down the system dramatically. It is also possible to selectively enable or disable this capability for particular flags using:`lctl set_param printk=+vfstrace` and `lctl set_param printk=-vfstrace `.

It is possible to disable warning, error, and console messages, though it is strongly recommended to have something like `lctl debug_daemon` running to capture this data to a local file system for failure detection purposes.

### Tracing Lock Traffic

The Lustre software provides a specific debug type category for tracing lock traffic. Use:

```
lctl> filter all_types 
lctl> show dlmtrace 
lctl> debug_kernel [filename]  
```

### Controlling Console Message Rate Limiting

Some console messages which are printed by Lustre are rate limited. When such messages are printed, they may be followed by a message saying "Skipped N previous similar message(s)," where N is the number of messages skipped. This rate limiting can be completely disabled by a libcfs module parameter called `libcfs_console_ratelimit`. To disable console message rate limiting, add this line to `/etc/modprobe.d/lustre.conf` and then reload Lustre modules.

```
options libcfs libcfs_console_ratelimit=0
```

It is also possible to set the minimum and maximum delays between rate-limited console messages using the module parameters `libcfs_console_max_delay` and `libcfs_console_min_delay`. Set these in `/etc/modprobe.d/lustre.conf` and then reload Lustre modules. Additional information on libcfs module parameters is available via `modinfo`:

```
modinfo libcfs
```

## Lustre Debugging for Developers

The procedures in this section may be useful to developers debugging Lustre source code.

### Adding Debugging to the Lustre Source Code

The debugging infrastructure provides a number of macros that can be used in Lustre source code to aid in debugging or reporting serious errors.

To use these macros, you will need to set the `DEBUG_SUBSYSTEM` variable at the top of the file as shown below:

```
#define DEBUG_SUBSYSTEM S_PORTALS
```

A list of available macros with descriptions is provided in the table below.

| **Macro**                                | **Description**                                              |
| ---------------------------------------- | ------------------------------------------------------------ |
| **LBUG()**                               | A panic-style assertion in the kernel which causes the Lustre file system to dump its circular log to the `/tmp/lustre-log` file. This file can be retrieved after a reboot. `LBUG()`freezes the thread to allow capture of the panic stack. A system reboot is needed to clear the thread. |
| **LASSERT()**                            | Validates a given expression as true, otherwise calls LBUG(). The failed expression is printed on the console, although the values that make up the expression are not printed. |
| **LASSERTF()**                           | Similar to `LASSERT()` but allows a free-format message to be printed, like `printf/printk`. |
| **CDEBUG()**                             | The basic, most commonly used debug macro that takes just one more argument than standard `printf()` - the debug type. This message adds to the debug log with the debug mask set accordingly. Later, when a user retrieves the log for troubleshooting, they can filter based on this type.                                                                                                                                                   `CDEBUG(D_INFO, "debug message: rc=%d\n", number);` |
| **CDEBUG_LIMIT()**                       | Behaves similarly to `CDEBUG()`, but rate limits this message when printing to the console (for `D_WARN`, `D_ERROR`, and `D_CONSOLE` message types. This is useful for messages that use a variable debug mask:                                                                                                                                                   `CDEBUG(mask, "maybe bad: rc=%d\n", rc);` |
| **CERROR()**                             | Internally using `CDEBUG_LIMIT(D_ERROR, ...)`, which unconditionally prints the message in the debug log and to the console. This is appropriate for serious errors or fatal conditions. Messages printed to the console are prefixed with `LustreError:`, and are rate-limited, to avoid flooding the console with duplicates.                                                                                                                                                   `CERROR("Something bad happened: rc=%d\n", rc);` |
| **CWARN()**                              | Behaves similarly to `CERROR()`, but prefixes the messages with `Lustre:`. This is appropriate for important, but not fatal conditions. Messages printed to the console are rate-limited. |
| **CNETERR()**                            | Behaves similarly to `CERROR()`, but prints error messages for LNet if `D_NETERR` is set in the `debug` mask. This is appropriate for serious networking errors. Messages printed to the console are rate-limited. |
| **DEBUG_REQ()**                          | Prints information about the given `ptlrpc_request` structure.                                                                                                                                                   `DEBUG_REQ(D_RPCTRACE, req, ""Handled RPC: rc=%d\n", rc);` |
| **ENTRY**                                | Add messages to the entry of a function to aid in call tracing (takes no arguments). When using these macros, cover all exit conditions with a single `EXIT`, `GOTO()`, or `RETURN()` macro to avoid confusion when the debug log reports that a function was entered, but never exited. |
| **EXIT**                                 | Mark the exit of a function, to match `ENTRY` (takes no arguments). |
| **GOTO()**                               | Mark when code jumps via `goto` to the end of a function, to match `ENTRY`, and prints out the goto label and function return code in signed and unsigned decimal, and hexadecimal format. |
| **RETURN()**                             | Mark the exit of a function, to match `ENTRY`, and prints out the function return code in signed and unsigned decimal, and hexadecimal format. |
| **LDLM_DEBUG()** **LDLM_DEBUG_NOLOCK()** | Used when tracing LDLM locking operations. These macros build a thin trace that shows the locking requests on a node, and can also be linked across the client and server node using the printed lock handles. |
| **OBD_FAIL_CHECK()**                     | Allows insertion of failure points into the Lustre source code. This is useful to generate regression tests that can hit a very specific sequence of events. This works in conjunction with "`lctl set_param fail_loc=*fail_loc*`" to set a specific failure point for which a given `OBD_FAIL_CHECK()` will test. |
| **OBD_FAIL_TIMEOUT()**                   | Similar to `OBD_FAIL_CHECK()`. Useful to simulate hung, blocked or busy processes or network devices. If the given `fail_loc` is hit, `OBD_FAIL_TIMEOUT()` waits for the specified number of seconds. |
| **OBD_RACE()**                           | Similar to `OBD_FAIL_CHECK()`. Useful to have multiple processes execute the same code concurrently to provoke locking races. The first process to hit `OBD_RACE()` sleeps until a second process hits `OBD_RACE()`, then both processes continue. |
| **OBD_FAIL_ONCE**                        | A flag set on a `fail_loc` breakpoint to cause the `OBD_FAIL_CHECK()` condition to be hit only one time. Otherwise, a `fail_loc` is permanent until it is cleared with "`lctl set_param fail_loc=0`". |
| **OBD_FAIL_RAND**                        | A flag set on a `fail_loc` breakpoint to cause `OBD_FAIL_CHECK()` to fail randomly; on average every (1 / fail_val) times. |
| **OBD_FAIL_SKIP**                        | A flag set on a `fail_loc` breakpoint to cause `OBD_FAIL_CHECK()` to succeed `fail_val`times, and then fail permanently or once with `OBD_FAIL_ONCE`. |
| **OBD_FAIL_SOME**                        | A flag set on `fail_loc` breakpoint to cause `OBD_FAIL_CHECK` to fail `fail_val` times, and then succeed. |



### Accessing the `ptlrpc` Request History

Each service maintains a request history, which can be useful for first occurrence troubleshooting.

`ptlrpc` is an RPC protocol layered on LNet that deals with stateful servers and has semantics and built-in support for recovery.

The ptlrpc request history works as follows:

1. `request_in_callback()` adds the new request to the service's request history.
2. When a request buffer becomes idle, it is added to the service's request buffer history list.
3. Buffers are culled from the service request buffer history if it has grown above `req_buffer_history_max` and its reqs are removed from the service request history.

Request history is accessed and controlled using the following parameters for each service:

- `req_buffer_history_len`

  Number of request buffers currently in the history

- `req_buffer_history_max`

  Maximum number of request buffers to keep

- `req_history`

  The request history

Requests in the history include "live" requests that are currently being handled. Each line in `req_history` looks like:

```
sequence:target_NID:client_NID:cliet_xid:request_length:rpc_phase service_specific_data 
```

| **Parameter**    | **Description**                                              |
| ---------------- | ------------------------------------------------------------ |
| **seq**          | Request sequence number                                      |
| `*target NID*`   | Destination `NID` of the incoming request                    |
| `*client ID*`    | Client `PID` and `NID`                                       |
| `*xid*`          | `rq_xid`                                                     |
| **length**       | Size of the request message                                  |
| **phase**        | New (waiting to be handled or could not be unpacked)Interpret (unpacked or being handled)Complete (handled) |
| **svc specific** | Service-specific request printout. Currently, the only service that does this is the OST (which prints the opcode if the message has been unpacked successfully |

### Finding Memory Leaks Using `leak_finder.pl`

Memory leaks can occur in code when memory has been allocated and then not freed once it is no longer required. The `leak_finder.pl` program provides a way to find memory leaks.

Before running this program, you must turn on debugging to collect all `malloc` and free entries. Run:

```
lctl set_param debug=+malloc 
```

Then complete the following steps:

1. Dump the log into a user-specified log file using lctl (see [*the section called “Using the lctl Tool to View Debug Messages”*](#using-the-lctl-tool-to-view-debug-messages)).

2. Run the leak finder on the newly-created log dump:

   ```
   perl leak_finder.pl ascii-logname
   ```

The output is:

```
malloced 8bytes at a3116744 (called pathcopy) 
(lprocfs_status.c:lprocfs_add_vars:80) 
freed 8bytes at a3116744 (called pathcopy) 
(lprocfs_status.c:lprocfs_add_vars:80) 
```

The tool displays the following output to show the leaks found:

```
Leak:32bytes allocated at a23a8fc(service.c:ptlrpc_init_svc:144,debug file line 241)
```

 
