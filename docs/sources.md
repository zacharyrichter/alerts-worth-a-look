# What reaches your phone

Anything at level 10 or above is pushed to ntfy ([`design.md`](design.md)). Most of what the stock ruleset sends is left alone. These are the sources added or raised on purpose, plus the one stock rule that already earns its level.

| Rule | Source | Fires on |
|---|---|---|
| 100140 | Suricata (router agent) | An ET alert with `alert.severity` 1 (trojan, C2 and similar). Severity 2 and 3 stay at the stock level 3 (rule 86601) |
| 100150 | Docker listener (`docker-hosts` group) | Any `exec` into a container, including `docker exec -it x sh` (also chained off the stock shell-session rule 87908) |
| 100151 to 100160 | Docker listener | Level 0: exact-match allowlist of known healthcheck commands, one rule per command |
| 100161 | Docker listener | Level 0: Gitea Actions job containers (`GITEA-ACTIONS-TASK-*`), matched by container name |
| 100170 | File integrity (`default` group agent.conf) | A file added, changed or deleted on the critical paths below |
| 87202 (stock) | Proxmox VE | Brute force: 8 failed web UI logins from one IP within 2 minutes. Single failures are 87201, level 6 |

## Suricata (100140)

- On the router agent, **Services → Wazuh Agent → Settings → Intrusion detection events** is enabled. The plugin adds a `localfile` for `/var/log/suricata/eve.json` (json).
- Suricata's eve HTTP and TLS logging stays off. Otherwise `eve.json` grows large and floods the index.
- To test without touching Suricata, paste a sample eve alert line into `wazuh-logtest`. `"severity":1` should hit 100140, and `2` should stop at 86601.

## Docker exec (100150 to 100161)

Every exec into a container is worth knowing about, so every exec alerts. The hard part is healthchecks, which run as `exec` too.

- **Hosts:** every Docker host, all in the `docker-hosts` agent group. That group's agent.conf holds the `docker-listener` wodle ([`files/agent-conf-docker-hosts.xml`](../files/agent-conf-docker-hosts.xml)).
- **Dependency:** Debian's `python3-docker`, installed by the second play of [`files/wazuh-agent.yml`](../files/wazuh-agent.yml). It is installed **before** a host is added to `docker-hosts`.
- **Why an allowlist:** without it, every container healthcheck would alert on every interval.
- **A new or changed healthcheck alerts on every interval** until it is allowlisted.

To allowlist a healthcheck:

1. Copy the exact `data.docker.Action` from the alert. Trailing spaces count, for example `exec_start: /healthchecker `.
2. Add a level-0 rule with `<if_sid>100150</if_sid>` and `<field name="docker.Action" type="pcre2">^\Qexec_start: ...\E$</field>`.
3. Write `$ & > <` as `\E\x24\Q`, `\E\x26\Q`, `\E\x3e\Q` and `\E\x3c\Q`. The rule loader does **not** decode `&amp;` or `&gt;`, and `$NAME` can be read as a rule variable.
4. Match on `docker.Action`, not `docker.status`. Current Docker no longer sends the deprecated `status` field.
5. Test with a line shaped like a real event (no `status` field) and expect level 0, then restart the manager:
   ```json
   {"integration":"docker","docker":{"Type":"container","Action":"exec_start: ak healthcheck","Actor":{"Attributes":{"name":"authentik-server"}}}}
   ```

The Gitea Actions rule (100161) works differently. The CI runner execs every workflow step into the job container with commands that change per action and per run, so these are silenced by container name (`^GITEA-ACTIONS-TASK-`), not by command. Creating the container still logs at the stock level 3 (87901).

Expected alerts: running `wazuh-logtest` (it is a `docker exec` into the manager) and any manual exec. These are left alerting on purpose.

**If it floods:** set 100150 to `level="3"`, restart the manager, investigate, then set it back.

## Critical-file integrity (100170)

[`files/agent-conf-default.xml`](../files/agent-conf-default.xml) is the `default` group's agent.conf. It applies to Linux only, so not the router. It watches these paths in real time (inotify, no auditd or whodata):

- `/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d`
- `/etc/ssh/ca` and `/etc/ssh/ssh_host_ed25519_key-cert.pub`
- `/etc/sudoers` and `/etc/sudoers.d`
- `/root/.ssh` and `/home/*/.ssh`
- `/etc/ntfy-sendmail.env`

- **`report_changes` (diffs) only on the SSH config and sudoers paths.** Never on `.ssh` or `ntfy-sendmail.env`, because keys and a token must not be copied to the manager.
- **`known_hosts` is excluded in the rule,** since it changes on normal outbound SSH.
- **Expected alerts:** hardening and host-certificate playbook runs, and the yearly host certificate renewal ([expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose)).
- **Everything else** under `/etc` and the binary directories is still covered by the stock 12-hour scan at level 7, which is dashboard only.
- **Test:**
  ```bash
  sudo touch /etc/sudoers.d/wazuh-test && sleep 5 && sudo rm /etc/sudoers.d/wazuh-test
  ```
  This gives two alerts, added and deleted.

## Kernel vulnerabilities (100180, lowered)

The stock vulnerability rules send a push per open CVE on every host each time a kernel package changes, and most of those CVEs aren't fixed upstream yet. Kernel package alerts are dropped to level 7, which is still indexed and visible in the dashboard but below ntfy. Exposure is handled by policy instead: unattended upgrades with a reboot, and unused kernel modules blocked. Alerts for every other package stay at the stock level.
