# Wazuh-Linux-Kernel-Module-Loading-Investigation
## Overview
### What Are Linux Kernel Modules?

Linux kernel modules are pieces of code that extend the functionality of the Linux kernel. They can be loaded into or removed from the running kernel without requiring the entire operating system to restart.

Modules are commonly used to provide hardware drivers, filesystem support, networking functionality, and other kernel-level features.

Examples include:

- Device drivers for network adapters, storage controllers, and sound devices.
- Filesystem modules such as XFS and ext4.
- Networking modules and protocol support.
- Kernel components that provide additional system functionality.

Kernel modules are generally stored under:

    /lib/modules/<kernel-version>/

### How Are Kernel Modules Managed?

Linux provides several commands for inspecting and managing kernel modules.

| Command | Purpose |
|---|---|
| `lsmod` | Lists currently loaded kernel modules. |
| `modinfo` | Displays module metadata and dependencies. |
| `modprobe` | Loads or removes modules by name and handles dependencies. |
| `insmod` | Inserts a module directly into the kernel. |
| `rmmod` | Removes a loaded module. |
| `dmesg` | Displays kernel ring-buffer messages. |
| `journalctl -k` | Displays kernel-related journal messages. |

For example:

    lsmod

Displays the currently loaded modules.

    modinfo nft_fib

Displays metadata for the `nft_fib` module.

    modprobe <module_name>

Requests the kernel to load a module by name.

### Why Is Kernel Module Loading Security-Relevant?

Kernel modules operate with kernel-level privileges. An unexpected or malicious module can potentially affect the operating system at a highly privileged level.

An attacker with sufficient privileges may attempt to load a malicious kernel module to:

- Hide processes, files, or network activity.
- Manipulate kernel behaviour.
- Establish persistence.
- Interfere with security monitoring.
- Modify system functionality.

Malicious kernel modules may be associated with rootkit activity. However, legitimate administrators, system services, and hardware drivers also load modules during normal system operations.

A module-loading event should therefore be investigated in context rather than automatically classified as malicious.

### Kernel Module Loading System Calls

Linux provides system calls that allow kernel modules to be loaded.

Two important system calls are:

- `init_module`: Loads a kernel module from a memory image.
- `finit_module`: Loads a kernel module using a file descriptor.

These system calls can be monitored through Linux Audit.

Monitoring them provides visibility into attempts to load kernel modules, including information about the process and user context.

### Linux Audit

Linux Audit provides a framework for recording security-relevant system activity.

The `auditd` service collects audit events, while `auditctl` can be used to configure runtime audit rules.

The following rule monitors module-loading system calls on a 64-bit system:

    auditctl -a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_module

The rule components have the following purposes:

- `-a always,exit`: Records matching system calls when they exit.
- `-F arch=b64`: Applies the rule to 64-bit system calls.
- `-S init_module`: Monitors the `init_module` system call.
- `-S finit_module`: Monitors the `finit_module` system call.
- `-k kernel_module`: Assigns an identifiable audit key to matching events.

The rule can be checked using:

    auditctl -l

Events associated with the audit key can be searched using:

    ausearch -k kernel_module -i

The audit records can contain information such as the timestamp, process ID, executable, user ID, system call, and operation result.

### Wazuh Integration

Wazuh can collect and analyse Linux Audit events when the relevant audit log collection is configured.

The typical investigation flow is:

1. Linux Audit records a system event.
2. The event is written to the audit log.
3. Wazuh collects the configured audit log.
4. Wazuh decodes the event.
5. Matching Wazuh rules generate alerts.
6. Analysts investigate the event using the available audit and system evidence.

Wazuh may also generate alerts when audit configuration changes occur.

An audit configuration-change alert and a kernel module-loading alert represent different activities. Analysts must examine the event type and system call before determining what actually happened.

### Understanding Module Metadata

The `modinfo` command provides information that can help analysts assess a kernel module.

Important fields include:

- `filename`: Location of the module file.
- `name`: Module name.
- `description`: Module functionality.
- `author`: Module author information, when available.
- `license`: Module license.
- `depends`: Required module dependencies.
- `vermagic`: Kernel version and build characteristics associated with the module.
- `signer`: Module signing identity, when available.
- `sig_id`: Signature format.

A valid signature and matching kernel version can provide useful context, but they do not independently prove that a module is safe or authorised.

### SOC Investigation Approach

When investigating kernel module loading activity, analysts should examine:

1. The module name and file path.
2. The module metadata and dependencies.
3. The module signature and signing identity, where available.
4. The timestamp of the loading operation.
5. The initiating process and executable.
6. The user and effective user context.
7. The audit result.
8. Related kernel messages.
9. Whether the activity corresponds to expected administrative or system behaviour.

The investigation should distinguish between:

- A successful module-loading operation.
- A failed module-loading attempt.
- An audit rule configuration change.
- A module that is already loaded.
- A module file that exists but has not been confirmed as loaded.

### Conclusion

Linux kernel module loading is a security-relevant activity because modules operate within the kernel's privileged execution environment.

Linux Audit provides visibility into module-loading system calls, while Wazuh can collect and analyse the resulting audit events.

Combining audit records, module metadata, kernel messages, and process context helps SOC analysts investigate unexpected kernel-level activity.

A detection should always be interpreted according to the actual event recorded. The presence of an audit rule or module file does not establish that a module was successfully loaded.


This lab investigates Linux kernel module activity using Linux Audit and Wazuh. Kernel modules extend the functionality of the Linux kernel and are commonly used for device drivers, filesystems, and networking components.

From a security perspective, unauthorized kernel module loading can indicate suspicious activity, including attempts to introduce rootkits, establish persistence, or interfere with system monitoring.

The investigation focuses on identifying module metadata, configuring audit monitoring for kernel module loading system calls, examining audit events, and validating how Wazuh processes Linux Audit telemetry.

## Lab Environment

| Component | Details |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel Version | 5.14.0-687.31.1.el9_8.x86_64 |
| Wazuh Version | 4.13.1 |
| Audit Framework | Linux Audit (`auditd`) |
| Audit Log | `/var/log/audit/audit.log` |
| Monitoring Tool | Wazuh |
| Environment | VMware virtual machine |

## Lab Objectives

The objectives of this lab are to:

- Understand the purpose and security relevance of Linux kernel modules.
- Identify currently loaded kernel modules and inspect their metadata.
- Understand the role of `init_module` and `finit_module` system calls.
- Verify the availability and operational status of Linux Audit.
- Configure audit monitoring for kernel module-loading system calls.
- Validate audit rules and identify configuration-change events.
- Examine module filenames, dependencies, kernel compatibility, and signing information.
- Investigate audit records to distinguish rule configuration activity from actual module-loading operations.
- Integrate Linux Audit telemetry with Wazuh and examine the resulting alerts.
- Correlate audit events with process information, user context, and kernel messages.
- Document command failures, monitoring limitations, and investigation evidence.
- Develop an evidence-based SOC investigation approach for suspicious kernel module activity.

## Lab Scenario

Linux kernel modules provide essential functionality for hardware support, networking, filesystems, and other operating system components. Because these modules operate with kernel-level privileges, unexpected module-loading activity can present a security concern.

A threat actor with sufficient privileges may attempt to introduce a malicious kernel module to manipulate system behaviour, conceal activity, or interfere with security monitoring. However, legitimate system operations also involve kernel module loading, making contextual investigation necessary.

In this lab, a Rocky Linux 9.8 system running Wazuh 4.13.1 components was examined to understand how Linux Audit can monitor kernel module-loading system calls and how Wazuh can process the resulting audit telemetry.

### Investigation Setup

The system was running kernel version `5.14.0-687.31.1.el9_8.x86_64`.

The initial inspection covered:

- Currently loaded kernel modules.
- Module metadata and dependencies.
- Kernel messages and journal records.
- Linux Audit service status.
- Existing audit rules.
- Wazuh audit event collection.

The audit service was active, but the initial audit rule listing returned no rules.

### Controlled Monitoring Configuration

A runtime audit rule was added to monitor the `init_module` and `finit_module` system calls using the audit key `kernel_module`.

The rule was verified using `auditctl -l`.

Wazuh subsequently generated an audit configuration-change alert when the rule was added. The alert identified the `auditctl` process and recorded the successful rule-addition operation.

### Module Investigation

Module metadata was examined using `modinfo`, including the `nft_fib` and `snd_xen_front` modules.

The `snd_xen_front` module file was present under the running kernel's module directory. Its metadata identified it as a Xen virtual sound device frontend, with dependencies and a Rocky Linux kernel signing identity.

An initial attempt to execute `modprobe` using the complete module file path failed. The command expected a module name rather than the supplied path.

The available evidence did not establish a successful module-loading operation. The audit search returned the rule configuration event, not a confirmed `init_module` or `finit_module` event.

### Investigation Focus

The investigation focused on distinguishing audit configuration activity from actual kernel module loading.

The following evidence was examined:

- Operating system and kernel version.
- Loaded module inventory.
- Module metadata and signing information.
- Audit service status and configured rules.
- Audit configuration-change records.
- Wazuh alert details and decoder information.
- Kernel messages and journal output.
- Results of the attempted module-loading command.

### Investigation Outcome

Linux Audit was active, and the kernel module-loading audit rule was successfully added.

Wazuh detected the audit configuration change through rule ID `80705`, with a low-severity alert indicating that the audit configuration had changed.

However, the available evidence did not confirm that a kernel module was successfully loaded during the exercise.

The investigation demonstrated how Linux Audit and Wazuh can provide visibility into security monitoring configuration changes while highlighting the importance of validating the actual system call and event type before concluding that module loading occurred.

## Key Commands

### Review Loaded Modules

```bash
lsmod
lsmod | wc -l
```

### Examine Module Metadata

```bash
modinfo nft_fib
modinfo snd_xen_front
```

### Check Audit Service

```bash
systemctl status auditd --no-pager
auditctl -l
```

### Configure Kernel Module Monitoring

```bash
auditctl -a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_module
```

### Review Audit Events

```bash
ausearch -k kernel_module -i
```

### Review Kernel Messages

```bash
journalctl -k --no-pager
dmesg | tail -50
```

## Investigation Workflow

1. Identify the operating system and running kernel version.
2. Review the currently loaded kernel modules.
3. Examine selected module metadata, including module location, dependencies, version compatibility, and signature information.
4. Verify that the Linux Audit service is running and inspect existing audit rules.
5. Configure runtime monitoring for the `init_module` and `finit_module` system calls.
6. Review audit records associated with the `kernel_module` key.
7. Examine the corresponding Wazuh alert and identify the event type, process, user, and audit details.
8. Correlate audit records with kernel messages and module metadata.
9. Document whether the available evidence establishes an actual module loading event.

## Key Findings

- The system was running Rocky Linux 9.8 with kernel version `5.14.0-687.31.1.el9_8.x86_64`.
- The initial module inventory contained 89 lines, including the header.
- Module metadata for `nft_fib` and `snd_xen_front` was successfully retrieved.
- The inspected modules contained Rocky Linux kernel signing information and matching kernel version metadata.
- The Linux Audit service was active, and the initial audit rule list was empty.
- A runtime audit rule was successfully added to monitor `init_module` and `finit_module`.
- Wazuh generated alert `80705` after the audit configuration change.
- The corresponding audit record identified an `auditctl` operation that added the monitoring rule.
- The observed `sendto` syscall was associated with the audit configuration operation, not a kernel module loading operation.
- The attempted `modprobe` command used a module file path instead of the module name and failed.

## Evidence and Limitations

The investigation confirmed that Linux Audit monitoring was configured and that Wazuh detected the resulting audit configuration change.

However, the available audit output did not establish a successful `init_module` or `finit_module` operation. Therefore, an actual kernel module loading event was not confirmed during the investigation.

The attempted `modprobe` operation also did not establish a successful load. The selected `snd_xen_front` module is a Xen virtual sound frontend, and its suitability for the VMware environment was not established. It was therefore not treated as a confirmed module-loading test.

The audit rule was added at runtime. Persistence across a system reboot was not verified.

## SOC Investigation Perspective

When investigating suspicious kernel module activity, analysts should examine:

- Module name and file location.
- Module signature and signing authority.
- Kernel compatibility and version information.
- Executing process and associated parent process.
- User identity and privilege level.
- Audit syscall, timestamp, and operation result.
- Related kernel messages.
- Whether the module installation or loading was authorized.

A module file existing on disk does not prove that it was loaded. Similarly, an audit configuration-change alert does not prove that a kernel module was loaded.

