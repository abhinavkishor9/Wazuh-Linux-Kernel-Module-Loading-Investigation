# Investigation Timeline 

| Sequence | Activity | Command / Evidence | Result |
|---|---|---|---|
| 1 | Verified operating system | `cat /etc/os-release` | Rocky Linux 9.8 confirmed |
| 2 | Identified running kernel | `uname -r` | `5.14.0-687.31.1.el9_8.x86_64` |
| 3 | Collected loaded module inventory | `lsmod` | 89 lines returned, including header |
| 4 | Inspected module metadata | `modinfo nft_fib` | Metadata and Rocky Linux signing information observed |
| 5 | Checked audit service | `systemctl status auditd --no-pager` | Active and running |
| 6 | Verified audit utility | `command -v auditctl` | `/usr/sbin/auditctl` |
| 7 | Inspected existing audit rules | `auditctl -l` | No rules initially listed |
| 8 | Added module-loading audit rule | `auditctl -a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_module` | Rule addition command completed |
| 9 | Verified audit rule | `auditctl -l` | `init_module` and `finit_module` rule displayed |
| 10 | Inspected Xen sound module | `modinfo snd_xen_front` using its file path | Module metadata and signing information observed |
| 11 | Verified module file | `ls -l` on `snd_xen_front.ko.xz` | File existed |
| 12 | Attempted module loading | `modprobe` with the full file path | Failed with module-not-found error |
| 13 | Reviewed kernel messages | `dmesg` and `journalctl -k` | VMware graphics warning and other system messages observed |
| 14 | Searched audit records | `ausearch -k kernel_module -i` | Audit rule configuration event returned |
| 15 | Examined Wazuh alert | Rule ID `80705` | Audit configuration-change alert confirmed |

