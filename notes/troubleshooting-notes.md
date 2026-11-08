# Troubleshooting Notes – Linux Kernel Module Loading Investigation

## 1. modprobe Returned Module Not Found

### Command Used

    modprobe /lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/sound/xen/snd_xen_front.ko.xz

### Error

    modprobe: FATAL: Module /lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/sound/xen/snd_xen_front.ko.xz not found in directory /lib/modules/5.14.0-687.31.1.el9_8.x86_64

### Cause

`modprobe` expects a module name rather than a complete `.ko.xz` file path.

The module metadata identified the name as `snd_xen_front`.

### Verification

The file was confirmed to exist using:

    ls -l /lib/modules/$(uname -r)/kernel/sound/xen/snd_xen_front.ko.xz

The module metadata was inspected using:

    modinfo /lib/modules/$(uname -r)/kernel/sound/xen/snd_xen_front.ko.xz

### Resolution and Limitation

The command syntax issue was identified.

However, the module was not loaded as part of the supplied evidence. The module is intended for a Xen virtual sound device, and its suitability for the VMware environment was not established.

No further loading attempt was made.

## 2. Initial Audit Rule Listing Returned No Rules

### Command

    auditctl -l

### Output

    No rules

### Cause

No active audit rules were listed at the time of the initial inspection.

### Action Taken

A runtime rule was added to monitor the 64-bit `init_module` and `finit_module` system calls:

    auditctl -a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_module

The configuration was verified using:

    auditctl -l

### Result

The expected audit rule appeared in the rule listing.

## 3. Wazuh Alert Was a Configuration-Change Event

### Observed Alert

| Field | Value |
|---|---|
| Rule ID | `80705` |
| Description | Auditd: Configuration changed. |
| Severity | 3 |
| Event type | `CONFIG_CHANGE` |
| Process | `auditctl` |
| Executable | `/usr/sbin/auditctl` |

### Investigation

The raw audit record contained:

    op=add_rule key="kernel_module" list=4 res=1

This showed that the audit rule was successfully added.

### Interpretation

The alert represented a change to the audit configuration.

It was not evidence of a kernel module-loading operation.

The actual module-loading system calls to investigate are:

- `init_module`
- `finit_module`

## 4. Audit Search Returned Only the Rule Configuration Event

### Command

    ausearch -k kernel_module -i

### Observation

The supplied output contained the audit records associated with adding the rule.

No actual module-loading event was shown.

### Interpretation

The monitoring rule was installed successfully, but the available audit search did not demonstrate that a module-loading event had been captured.

A successful rule installation should not be reported as a successful module load.

## 5. Module File Existed but Was Not Confirmed as Loaded

### Module

    snd_xen_front

### File

    /lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/sound/xen/snd_xen_front.ko.xz

### Verification

The file existed, and `modinfo` returned its metadata.

The metadata identified the module as a Xen virtual sound device frontend.

### Interpretation

File existence and module metadata do not establish that the module is currently loaded.

The previous `lsmod | grep` command used the module file path rather than the module name, so its empty output was not a valid module-state check.

A module-state check should use:

    lsmod | grep -w snd_xen_front

No successful module-loading result was supplied for this module.

## 6. Kernel Messages Contained VMware Graphics Warnings

### Observation

The supplied `dmesg` output contained a warning and stack trace involving:

    vmwgfx

It also contained other system messages, including an `ibus-daemon` trap.

### Interpretation

The available evidence did not associate these messages with a kernel module-loading operation.

They were retained as separate system observations and were not classified as evidence of malicious module activity.

## 7. Audit Rule Persistence Not Verified

The rule was added using `auditctl`, which configures runtime audit rules.

The supplied evidence confirmed the active rule listing but did not establish that the rule had been saved in persistent audit configuration.

Persistence across a reboot was not tested.

## 8. Troubleshooting Summary

| Issue | Observation | Status |
|---|---|---|
| `modprobe` file-path error | Full module path supplied instead of module name | Cause identified |
| No initial audit rules | `auditctl -l` returned `No rules` | Runtime rule added |
| Wazuh alert interpretation | Alert represented `CONFIG_CHANGE` | Confirmed |
| Audit search | Returned rule-addition records | No module-load event established |
| Module file verification | `snd_xen_front` file existed | Confirmed |
| Module loading | No successful load evidence | Not established |
| Kernel warnings | VMware graphics warning observed | Separate observation |
| Audit persistence | Persistent configuration not inspected | Not established |

## Conclusion

The main troubleshooting issue was the incorrect `modprobe` argument. The audit monitoring configuration was successfully installed and Wazuh detected the resulting configuration change.

The investigation did not establish a successful module-loading event. Further testing requires a suitable module for the VMware environment and verification of the resulting audit syscall records.
