# Lab 131 – Linux Kernel Module Loading Investigation

## Overview

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

## Investigation Objectives

- Understand Linux kernel modules and their security implications.
- Identify currently loaded kernel modules using `lsmod`.
- Examine module metadata using `modinfo`.
- Monitor kernel module loading system calls.
- Configure an audit rule for `init_module` and `finit_module`.
- Review Linux Audit events using `ausearch`.
- Examine how Wazuh detects and reports audit configuration changes.
- Distinguish between audit-rule configuration activity and actual module loading.
- Document investigation findings and telemetry limitations.

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

## Conclusion

This lab demonstrated how Linux Audit can monitor kernel module loading system calls and how Wazuh can process related audit telemetry.

The investigation successfully validated audit-rule configuration monitoring and Wazuh alert generation. However, the supplied evidence did not confirm an actual kernel module loading event.

The main takeaway is to distinguish between monitoring configuration activity, attempted operations, and successfully executed kernel module loading operations.

## Skills Demonstrated

- Linux System Administration
- Linux Kernel Module Analysis
- Linux Audit Framework
- Audit Rule Configuration
- Syscall Monitoring
- Wazuh SIEM
- Security Event Analysis
- Log Correlation
- Evidence-Based Investigation
- SOC Documentation
