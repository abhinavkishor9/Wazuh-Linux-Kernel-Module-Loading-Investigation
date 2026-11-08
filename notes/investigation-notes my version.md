# Investigation Notes
## 1. Environment Details

| Component | Observed value |
|---|---|
| Operating system | Rocky Linux 9.8 (Blue Onyx) |
| Kernel version | `5.14.0-687.31.1.el9_8.x86_64` |
| Wazuh version | 4.13.1 |
| Wazuh Manager | Active |
| Wazuh Indexer | Installed |
| Wazuh Dashboard | Installed |
| Audit service | Active and running |
| Audit utility | `/usr/sbin/auditctl` |
| Audit log | `/var/log/audit/audit.log` |

## 2. Initial Module Inventory

The loaded module inventory was collected using:

    lsmod

The command returned 89 lines, including the header.

Some observed modules included:

- `vmwgfx`
- `vmw_balloon`
- `vmxnet3`
- `xfs`
- `nvme`
- `nvme_core`
- `snd`
- `snd_pcm`
- `drm`
- `libata`

These modules provide functionality associated with the system's virtual hardware, storage, filesystem, networking, sound, and graphics components.

The inventory established a baseline of loaded modules. It did not independently establish when or how each module had been loaded.

## 3. Module Metadata Examination

### Module: nft_fib

Command:

    modinfo nft_fib

Observed information:

| Attribute | Value |
|---|---|
| Module name | `nft_fib` |
| Filename | `/lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/net/netfilter/nft_fib.ko.xz` |
| Author | Florian Westphal |
| License | GPL |
| Rocky version | 9.8 |
| Dependency | `nf_tables` |
| In-tree module | Yes |
| Signer | Rocky Linux kernel signing key |
| Signature algorithm | SHA-256 |

The metadata showed that the module was associated with the running Rocky Linux kernel release and included a Rocky Linux kernel signing identity.

### Module: snd_xen_front

The module file was inspected using:

    modinfo /lib/modules/$(uname -r)/kernel/sound/xen/snd_xen_front.ko.xz

Observed information:

| Attribute | Value |
|---|---|
| Module name | `snd_xen_front` |
| Description | Xen virtual sound device frontend |
| License | GPL |
| Rocky version | 9.8 |
| Dependencies | `snd-pcm`, `snd`, `xen-front-pgdir-shbuf` |
| In-tree module | Yes |
| Signer | Rocky Linux kernel signing key |
| Signature algorithm | SHA-256 |
| Kernel compatibility | Matching `vermagic` value |

The module file was also confirmed to exist:

    ls -l /lib/modules/$(uname -r)/kernel/sound/xen/snd_xen_front.ko.xz

The file was reported as 20,480 bytes and owned by `root:root`.

The metadata and file existence confirmed that the module was available on disk. They did not establish that it was loaded.

## 4. Linux Audit Configuration

The audit service was checked using:

    systemctl status auditd --no-pager

Observed status:

- Service: `auditd.service`
- State: Active (running)
- Enabled: Yes
- Main process: `auditd`

The audit utility was available at:

    /usr/sbin/auditctl

The initial rule listing returned:

    auditctl -l

    No rules

This established that no active audit rules were listed at the time of the initial check.

## 5. Kernel Module Audit Rule

The following runtime rule was added:

    auditctl -a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_module

The rule was verified using:

    auditctl -l

Observed rule:

    -a always,exit -F arch=b64 -S init_module,finit_module -F key=kernel_module

This confirmed that the runtime audit rule was installed.

The rule monitors 64-bit module-loading system calls. The supplied evidence does not establish that a persistent rule was configured for future reboots.

## 6. Wazuh Alert Analysis

Wazuh generated an alert after the audit rule was added.

### Alert Details

| Field | Observed value |
|---|---|
| Timestamp | Nov 8, 2026 @ 05:56:44.793 |
| Index | `wazuh-alerts-4.x-2026.11.08` |
| Agent ID | `000` |
| Agent name | `localhost.localdomain` |
| Decoder | `auditd` |
| Audit event type | `CONFIG_CHANGE` |
| Audit key | `kernel_module` in the configuration-change record |
| Rule ID | `80705` |
| Rule description | Auditd: Configuration changed. |
| Rule level | 3 |
| Rule groups | `audit`, `audit_configuration` |
| Log source | `/var/log/audit/audit.log` |

### Relevant Audit Evidence

The alert contained the following information:

- `op=add_rule`
- `key="kernel_module"`
- `res=1`
- `comm="auditctl"`
- `exe="/usr/sbin/auditctl"`
- `auid=1000`
- `uid=0`
- `euid=0`
- `tty=pts0`

The event showed that the `auditctl` process successfully added the audit rule.

The process was executed with root privileges, while the audit login identity was recorded as `abhinavmathur`.

### Alert Interpretation

This alert confirms a successful audit configuration change.

It does not confirm that a kernel module was loaded.

The syscall information in the alert relates to the audit rule configuration operation. The event must not be interpreted as a successful `init_module` or `finit_module` operation.

The alert's `data.audit.key` field was displayed as `null`, although the raw configuration-change record contained `key="kernel_module"`. This is a difference between the raw audit record and the decoded alert fields.

## 7. Audit Search Results

The following command was executed:

    ausearch -k kernel_module -i

The returned evidence contained:

- `PROCTITLE`
- `SYSCALL`
- `CONFIG_CHANGE`

The records described the addition of the audit rule by `auditctl`.

No actual module-loading event was present in the supplied search output.

Therefore, the audit search confirmed that the monitoring rule was configured, but did not demonstrate that the rule had captured a module-loading operation.

## 8. Module Loading Attempt

An attempt was made to load the `snd_xen_front` module using its full file path:

    modprobe /lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/sound/xen/snd_xen_front.ko.xz

The command returned:

    modprobe: FATAL: Module /lib/modules/5.14.0-687.31.1.el9_8.x86_64/kernel/sound/xen/snd_xen_front.ko.xz not found in directory /lib/modules/5.14.0-687.31.1.el9_8.x86_64

The error occurred because `modprobe` expects a module name rather than a complete module file path.

The correct module name identified through `modinfo` was:

    snd_xen_front

The failed command did not establish a successful module-loading operation.

The module is also specifically described as a Xen virtual sound device frontend. Its suitability for the VMware-based lab environment was not established, so it was not treated as an appropriate test module.

## 9. Kernel Message Review

The following command was used:

    journalctl -k --since "5 minutes ago" --no-pager

The supplied output contained:

    sched: RT throttling activated

The `dmesg` output also contained a VMware graphics-related warning and stack trace involving `vmwgfx`, along with other system messages.

No supplied kernel message directly confirmed the loading of `snd_xen_front`, `nft_fib`, or another test module during the exercise.

The VMware graphics warning was recorded as separate system evidence and was not attributed to the module-loading investigation.

## 10. Investigation Findings

### Confirmed

- Rocky Linux 9.8 was running kernel version `5.14.0-687.31.1.el9_8.x86_64`.
- Linux Audit was active.
- The initial audit rule listing was empty.
- The `kernel_module` audit rule was successfully added.
- Wazuh detected the audit configuration change.
- The Wazuh alert was generated under rule ID `80705`.
- The `snd_xen_front` module file existed on disk.
- Module metadata showed a Rocky Linux kernel signing identity.
- The attempted `modprobe` command failed.

### Not Established

- Successful loading of `snd_xen_front`.
- Successful loading of `nft_fib` during the exercise.
- A captured `init_module` event.
- A captured `finit_module` event.
- A corresponding Wazuh module-loading alert.
- Persistence of the audit rule across a reboot.

