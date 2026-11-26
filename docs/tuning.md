# Rule tuning

Custom rules live in the manager's `etc/rules/local_rules.xml`, edited in the dashboard (Server management → Rules → Manage rules files → Custom rules). They are not in Git, so [`files/local_rules.xml`](../files/local_rules.xml) is the reference copy and is kept in step by hand.

**Goal:** anything at level 10 or above should be worth looking at. Expected noise is lowered, not deleted, unless it is pure junk.

## The rules

| Rule | Parent | Level | What it does |
|---|---|---|---|
| 100100 | 510 rootcheck | 0 | Ignore LXC runtime files in `/dev/.lxc` (Proxmox containers) |
| 100101 | 510 rootcheck | 0 | Ignore the link-count check on the FAT `/boot/efi` partition (FAT has no hard links) |
| 100111 | 87701 firewall block | 0 | Ignore internal out-of-state TCP teardown blocks (FIN and RST after state expiry, such as a long media stream) |
| 100112 | 87701 firewall block | 5, `no_log` | External-source blocks get their own rule, so they never count toward 87702 (level 10) |
| 100113 | frequency on 100112 | 5 | 18 external blocks from one source in 45 seconds: internet scanning, visible but not sent to ntfy |
| 100114 | 87701 firewall block | 0 | Ignore internal broadcast blocks (`x.x.x.255` and `255.255.255.255`), such as device discovery chatter |
| 100115 | 87701 firewall block | 0 | Ignore IP cameras probing their gateway on UDP 21065 |
| 100116 | 87701 firewall block | 0 | Ignore a phone app's local control attempts on a smart plug (blocked by policy; the app falls back to the cloud) |
| 100120 | 80710 auditd promisc | 3 | Promiscuous mode on bridge ports (Docker `veth<hex>`, LXC `veth<vmid>i<n>`, VM `tap<vmid>i<n>`, Proxmox `fwln`, `fwbr<vmid>i<n>`, `fwpr<vmid>p<n>`). Real NICs (`ens*`, `eth*`, `eno*`, `vmbr*`) still alert at 10 |
| 100130 | 80700 auditd | 3 | Laptop: denied file operations by a logged-in user (`ws-unsuccessful-*`) |
| 100131 | 80700 auditd | 3 | Laptop: routine activity kept on record (installers, module load and unload, logins, sessions, time changes, network config) |
| 100132 | 80700 auditd | 3 | Laptop: running as another user (`sudo`, `pkexec`, `systemd-run`) |
| 100133 | 80700 auditd | 5 | Laptop: host identity and banner changes, reads of the audit log |
| 100134 | 80700 auditd | 7 | Laptop: auth stack, accounts, sudoers, SELinux policy, root touching user homes |
| 100135 | 100134 | 3 | Laptop: routine sources of 100134 (password checks at unlock and `sudo`, `auditctl`) |
| 100136 | 100134 | 3 | Laptop: root touching a user-owned file via Python (an Ansible run) |
| 100140 to 100170 | | 10 and 0 | The sources raised to level 10 and the Docker allowlist ([`sources.md`](sources.md)) |
| 100180 | 23505, 23506 vulnerability | 7 | Kernel package vulnerabilities, dashboard only ([`sources.md`](sources.md)) |

Rule 87702 (level 10) still fires for repeated blocks from internal sources: a host on our own VLANs hammering something it isn't allowed to reach. That is the signal worth keeping.

## Why the firewall rules are split this way

The stock rule 87702 is a frequency rule: it counts blocks (87701) from one source and goes to level 10. External scanners would reach level 10 constantly. The obvious fix is a child rule of 87702, but that never runs, because frequency rules don't evaluate `if_sid` children.

So the split happens one level lower. 100112 takes external-source blocks at the single-block level and ends them there (`no_log`), so 87702 never counts them. 100113 then watches that quieter stream with its own frequency, at level 5, so scanning is visible without being pushed.

## Laptop audit keys

The laptop's audit rules carry a `ws-` prefix on every key. Wazuh's stock auditd rules only know the `audit-wazuh-*` keys, so without a rule of its own an event with a `ws-` key stops at the parent rule 80700 (level 0) and is never kept. The prefix also keeps these rules from matching the Debian fleet, which uses some of the same key names.

Rules 100130 to 100136 raise those events to levels 3 to 7. None reaches 10, so nothing is sent to ntfy: the laptop roams and is used interactively, so its routine activity is recorded for the dashboard and for a review after an incident, not pushed. The two child rules (100135, 100136) drop the routine sources of level 7, because they would otherwise fire on every screen unlock, `sudo` password check and Ansible run.

Two things follow from the design. A key that isn't in the lists above never gets a rule, so a new audit key needs a rule added here. The rules are children of 80700, not frequency rules, so they can be tested by hand with `wazuh-logtest`.

The audit rules and their keys are in the laptop's baseline playbook in [assume-it-walks](https://github.com/zacharyrichter/assume-it-walks).

## File-integrity ignores

Set in the dashboard under Agents management → Groups → `<group>` → agent.conf.

| Group | Ignore | Why |
|---|---|---|
| `debian_fleet` | `<ignore type="sregex">/.git/</ignore>` | Git internals in deploy-tool repo clones churn on every pull and Renovate branch fetch. Files outside `.git` (compose files, configs) are still monitored |
| `proxmox` | The cluster filesystem's constantly rewritten status files (`.rrd`, `.version`, `.clusterlog`, `.vmlist` under `/etc/pve`) and `lrm_status` | pmxcfs and HA status files rewritten constantly. VM configs (`qemu-server/*.conf`), authkeys (rotated every 24 hours by Proxmox) and the rest of `/etc/pve` stay monitored |

Remaining expected changes: `/boot` on kernel updates and package files on updates.

## Working on rules

1. **Find what's noisy.** In Threat Hunting, query `rule.level >= 10`, then click `rule.id`, `agent.name` and the relevant `data.*` field in the field list for top values. For counts across days, the daily files are in `/srv/wazuh-data/manager-logs/alerts/YYYY/Mon/ossec-alerts-DD.json.gz` (today's is `alerts/alerts.json`).
2. **Look up the stock rule before overriding it.**
   ```bash
   sudo docker exec wazuh-manager grep -rn -A12 'id="<id>"' /var/ossec/ruleset/rules/
   ```
3. **Write the rule in the dashboard,** save, and restart the manager.
4. **Test with the CLI,** which always starts a fresh session:
   ```bash
   sudo docker exec -it wazuh-manager /var/ossec/bin/wazuh-logtest
   ```
   The dashboard's Ruleset Test keeps its session, and the ruleset it loaded, across restarts. `firedtimes` above 1 means a stale session.

### Gotchas

- Paste test lines with no leading spaces, or the decoder won't match (`No decoder matched`).
- Never hang a rule off a frequency rule (`frequency=` or `if_matched_sid`) with `if_sid`. Tune the single-event rule underneath it instead.
- Use `<srcip negate="yes">` for "not in this range".
- Frequency rules can't be tested by hand. Confirm them live in Threat Hunting a day later.
- Built-in fields (`srcip`, `dstip`, `dstport` and so on) use their own tags. `<field name="dstip">` is rejected on save. To pattern-match them, use `<regex type="pcre2">` on the raw log line, like 100111 and 100114.

## Considered and left out

- **An agent on the internet-facing host.** It has its own change scanning, and it must never reach into the home network.
- **whodata for file watching.** The audit rules are immutable, so adding watch rules needs a rules change plus a reboot on every host. That isn't worth it with a single admin.
- **Reverse proxy and identity provider logs.** CrowdSec already parses the proxy's logs, and the identity provider has its own event log.
- **Syslog from the switch, access point, home automation and NAS.** MGMT-only, low value, and it would open UDP 514 on the manager.
