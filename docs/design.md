# Design

## Purpose

A single-node Wazuh stack that every host in the lab reports to: file integrity, rootcheck, CIS (SCA) scans, vulnerability detection, auditd, Docker exec events and the router's IDS alerts. Level 10 and above are pushed to ntfy. Everything else is browsed in the dashboard.

## Decisions

**Its own node.** The stack runs on its own Proxmox node, away from the hosts it watches, so a compromised or overloaded app host doesn't take the monitoring with it.

**Docker, deployed from Git.** The three Wazuh containers come from the official single-node compose layout. The files that change (`ossec.conf`, the indexer and dashboard config, the notification script) live in a Git repo, and the deploy tool pulls and redeploys them. The stack is marked critical, so updates are always deployed by hand.

**Secrets stay on the host.** The env files, the enrollment password and the notification token are not in Git. They sit on the host only, under `/opt/appdata/wazuh/`, and are covered by the VM backup.

**Bulk data on its own disk, not backed up.** Alert history, indexes and the vulnerability feed are on a separate data disk on the node's NVMe, which the backup skips. All of it is rebuilt or downloaded again, so backing it up would cost storage for nothing. Agent registrations, certificates and config are on the system disk, which is backed up, so losing the data disk never means re-enrolling agents. See [backups-nobody-watches](https://github.com/zacharyrichter/backups-nobody-watches).

**Docker won't start without the data disk.** The disk is mounted with `nofail`, so the VM still boots without it. A systemd drop-in makes Docker require the mount, so the containers can't start against empty folders on the system disk.

**Only what's worth a look reaches the phone.** Level 10+ goes to ntfy. Expected noise is lowered with custom rules rather than by raising the threshold ([`tuning.md`](tuning.md)).

## The notification pipeline

Alerts at level 10 and above are pushed to a `wazuh-alerts` topic on the notification server by a custom integration, `custom-ntfy`.

| Alert level | ntfy priority | Tag |
|---|---|---|
| 10 to 11 | high (4) | `warning` |
| 12 and above | urgent (5) | `rotating_light` |

Tapping a notification opens the Wazuh dashboard. Anything below level 10 is never sent.

| Piece | Where |
|---|---|
| Script | [`files/custom-ntfy`](../files/custom-ntfy) (Python, no extension; uses the manager's bundled Python and `certifi`) |
| Integration block | [`files/integration-block.xml`](../files/integration-block.xml), placed in `wazuh_manager.conf` between the osquery and syscollector wodles |
| Token | `/opt/appdata/wazuh/secrets/ntfy.token` on the Wazuh VM, owner `root`, group `999` (the container's `wazuh` group), mode `640`. Never in Git |
| Mounts | In the compose file's manager `volumes:`: the script to `/var/ossec/integrations/custom-ntfy` (read-only) and the token to `/var/ossec/etc/ntfy.token` (read-only) |
| Executable bit | The deploy tool's **Pre Deploy** step runs `chmod 755 config/integrations/custom-ntfy`, because Gitea cannot store the executable bit |
| Log | `/srv/wazuh-data/manager-logs/integrations.log`, with `sent rule ... -> HTTP 200` or `send failed ...` |

The ntfy token belongs to the `admin` account on the notification server and is kept in the password manager.

### What the script does

1. Reads the alert JSON that the manager's integration daemon passes in, and the token from the mounted file.
2. Builds the title as `[agent] rule description` and the body from the level, rule ID, agent name and address, rule groups and up to 1500 characters of the original log line.
3. Sends it to the topic with a bearer token. Headers are collapsed to one line and forced to latin-1, which HTTP requires.
4. Logs the result. A failed send is logged with the reason and the script exits non-zero.

The bundled Python has no CA store, so the script loads `certifi` and falls back to the default context only if it is missing.

## The stack

| Container | Published | Config from the repo | Host-only files |
|---|---|---|---|
| manager | 1514 (agents), 1515 (enrollment) | `wazuh_manager.conf` (becomes `ossec.conf`), `custom-ntfy` | env files; `authd.pass` and `ntfy.token` (read-only); the indexer client certificate |
| indexer | none | `wazuh.indexer.yml` (single node) | env file; certificates and the security config |
| dashboard | 443 | `opensearch_dashboards.yml` | env files; certificates and the dashboard config |

The manager's own state (`etc`, including the agent registry and `local_rules.xml`, plus `queue`, `api/configuration` and Filebeat) is in Docker named volumes on the system disk. The indexer needs `vm.max_map_count=262144`.

### Data disk

| Folder | Mounted at | Owner, mode | Contents |
|---|---|---|---|
| `indexer/` | indexer `/var/lib/wazuh-indexer` | 1000:1000, 0750 | Indexes (alerts, vulnerabilities, inventory) |
| `manager-logs/` | manager `/var/ossec/logs` | 999:999, 0770 | `alerts/` (daily `ossec-alerts-DD.json.gz`), `integrations.log`, manager logs |
| `manager-vd/` | manager `/var/ossec/queue/vd` | 0:999, 0660 | The vulnerability feed |
| `manager-tmp/` | manager `/var/ossec/tmp` | 0:999, 1770 | Unpacking space for the feed |

`queue/vd` is mounted, not `queue/vd/feed`. The vulnerability scanner deletes and recreates `feed` on start, which fails on a mount point ("Device or resource busy"), stops the scanner and leaves a large unpacked snapshot in `tmp`.

### Manager settings

| Area | Setting |
|---|---|
| Agent connections | Port 1514/tcp, `secure` |
| Enrollment | Port 1515, password only (`use_password`, read from `authd.pass`); `use_source_ip no`, `purge yes` |
| Alerts | Logged from level 3; JSON output on; no email |
| Notifications | `custom-ntfy`, level 10 and above |
| Vulnerability detection | On, feed updated every 60 minutes |
| SCA | Every 12 hours |
| Syscollector | Every hour (hardware, OS, network, packages, ports, processes) |
| FIM (manager defaults) | `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/boot` every 12 hours. Agent groups add the real-time critical-file set and the ignores |
| Rootcheck | Every 12 hours |
| Cluster | Off (single node) |

### Updates

The three images are grouped into one Renovate pull request, held to the current major release until a deliberate migration, with a minimum release age. Merges never auto-deploy, since the stack is critical. Agents are upgraded separately and must never be newer than the manager ([`agents.md`](agents.md)).

## Sign-in

The dashboard signs in through the identity provider over SAML, which is the normal login. The local login (basic auth) stays enabled as the fallback for when the identity provider is down. Both are set in `opensearch_dashboards.yml`, and the provider's metadata file sits with the dashboard's config on the host. Sessions expire after 15 minutes idle. The dashboard and the indexer use certificates from the stack's own CA.

## Network access

There is no host firewall on the Wazuh VM. The router controls access between VLANs, and the management hosts reach the VM directly.

| Direction | Rule | Port | Purpose |
|---|---|---|---|
| In | Every agent VLAN to the Wazuh VM | 1514 | Agent connections |
| In | The apps VLAN and the user devices alias to the Wazuh VM | 1515 | Enrollment, kept open there by policy. Other VLANs open it only while enrolling |
| Out | MGMT to the notification server | 443 | `custom-ntfy` notifications |
| Out | MGMT to non-private networks | any | Internet only (vulnerability feed, images, apt) |
