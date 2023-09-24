# Linux security audit configurations
Three configuration files for my favorite tools, aimed to provide enough visibility to detect common attacks
while not overwhelming SIEM with tons of noisy events. The tools and their unique features are:

- Auditd (https://linux.die.net/man/8/auditd)
  - Immutable config until reboot (-e 2 flag)
  - Works everywhere as it is kernel-native
  - Supports all syscalls + auth events
- OSquery (https://www.osquery.io)
  - Can "snaphsot" listening ports, users, packages, etc.
  - Logs actual changes of SSH keys, cronjobs, systemd services
  - Can join data from multiple sources for sequence-like rules
  - Can calculate file hash for every process or file event
- Falco (https://falco.org)
   - Powerful config language to create filters (glob, wildcards)
   - Context of parent process, shell session, container info
   - Great logging format, allowing tags and list of fields to include
 
| Item | Auditd | OSquery | Falco |
| ----- | ----- | ----- | ----- |
| Supported Kernel | All kernels | All popular distros | 3.4+ kernels
| Working Mode | Kernel-native | User-space | Kernel module / eBPF
| Dependencies | None | None | Linux headers to build kernel module
| Performance Impact | Minimal | High for evented tables | Minimal
| Logging Format | Unreadable | Pretty JSON | Pretty KV or JSON
| Config Format | Specific format | JSON with SQL-like rules | YAML with KV rules, like in Kibana
| Config Filters | No wildcards | SQL "where" and "like" | Glob, wildcards, macros
| Parent Context | Only PPID | Only PPID | PPID, image, cmdline, everything
| Container Context | None | None | ID, name, image, everything

## Usage
Logging is divided into multiple categories and two types: file integrity monitoring (FIM) and process auditing.
- **FIM:** New cronjobs, systemd services, SSH keys, bash profile, and many other persistence methods
- **Processes:** discovery commands, ingress tool transfer, user changes, rootkit-related commands and so on

Note: The listed tools can not coexist together on a single server or VM. Only a single tool from the list must be enabled, otherwise you will face buggy logging, performance issues or other problems that are hard to debug.


### Auditd
Install Auditd, put the config into `/etc/audit/rules.d` as "audit.rules", and restart the Auditd service.
You should see the logs in `/var/log/audit/audit.log`, or query them via `ausearch -i [-k key]`.

You may see output example in [auditd/audit.log](https://github.com/maxvarm/linux-siem-audit-configs/auditd/audit.log).

### OSquery
Install OSquery from the official website, put the JSON config in `/etc/osquery/osquery.conf`, and setup flagfile.
Flagfile is a file in `/etc/osquery/osquery.flags` containing OSquery parameters. You will have to supply at least
flags described [here](https://osquery.readthedocs.io/en/stable/deployment/process-auditing/) to support evented tables.

Note that process event are not divided into groups due to an increased performance impact. Restart the service, and you
will see the logs in `/var/log/osquery/osqueryd.results.log`, accessible via interactive `sudo osqueryi` shell.

You may see output example in [osquery/osquery.log](https://github.com/maxvarm/linux-siem-audit-configs/osquery/osquery.log).

## Falco
Install Falco from the official website either as kernel module, requiring multiple dependecies to build it locally, or
as eBPF program if your kernel is of 4.18+ version and supports eBPF. Then, setup JSON log format and enable logfile output
by configuring `/etc/falco/falco.yaml` as described in [docs](https://falco.org/docs/reference/daemon/config-options/).

Move the config file to `/etc/falco/falco_rules.local.yaml` and optionally disable the default ones in `/etc/falco/falco_rules.yaml`. After rebooting the service, you should see the logs inside a configured logfile.
You may also want to tune event buffering if amount of generated alerts is very low, causing delayed write to the logfile.

You may see output example in [falco/falco.log](https://github.com/maxvarm/linux-siem-audit-configs/falco/falco.log).
