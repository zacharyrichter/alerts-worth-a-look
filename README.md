# alerts-worth-a-look

**A security monitor is only useful if what reaches my phone is worth a look. Alerts at level 10 and above are pushed to my phone. Everything else is browsed in a dashboard, and noise is lowered with rules, never by raising the threshold.**

One single-node Wazuh stack, with an agent on every host that can run one. A short integration script sends level 10+ alerts to my own notification server. Custom rules lower the noise I expect and raise the few things I never want to miss.

```
agents ──▶ Wazuh manager ──▶ rules ──┬─ level 10+ ──▶ ntfy ──▶ phone
 (every host)  (one node)            └─ below 10  ──▶ dashboard only
```

## The ideas that matter

1. **The threshold stays at 10.** A noisy rule is fixed at the rule, so what reaches the phone keeps one meaning: look at this.
2. **Expected noise is lowered, not deleted.** Container starts, firewall chatter and kernel CVEs drop below 10, but they are still indexed and searchable. Only pure junk goes to level 0.
3. **A few things are raised on purpose.** High-severity IDS alerts, any command run inside a container, and changes to SSH, sudo and key files all go to level 10.
4. **Healthchecks are matched exactly, never loosely.** Every container healthcheck is an `exec`, so each known one is allowlisted by its exact command string. A new or changed healthcheck alerts until it is added, which is the safe direction to fail.
5. **The rules are tested before they are trusted.** A rule is tried in the command-line tester, a fake alert proves the integration, and a temporary rule proves the whole path to the phone.
6. **Secrets stay on the host.** The enrollment password and the notification token are not in Git. The dashboard signs in through the identity provider, with a local login as the fallback.

## What's here

| Path | What it is |
|---|---|
| [`docs/design.md`](docs/design.md) | The decisions, the notification pipeline, the stack and where its data lives |
| [`docs/sources.md`](docs/sources.md) | The four things raised to level 10, how each is tested, and how the Docker allowlist grows |
| [`docs/tuning.md`](docs/tuning.md) | Every custom rule, the file-integrity ignores, the reasoning, and how to work on rules safely |
| [`docs/agents.md`](docs/agents.md) | How agents are installed, grouped, upgraded and removed |
| [`docs/runbook.md`](docs/runbook.md) | Changing config, rules and the script; updating; rebuilding; testing the alert path |
| [`docs/verification.md`](docs/verification.md) | The checks, and what each failure means |
| [`files/custom-ntfy`](files/custom-ntfy) | The integration script |
| [`files/integration-block.xml`](files/integration-block.xml) | The manager's integration block |
| [`files/local_rules.xml`](files/local_rules.xml) | The custom rules |
| [`files/agent-conf-default.xml`](files/agent-conf-default.xml) | The critical-file watch for the `default` agent group |
| [`files/agent-conf-docker-hosts.xml`](files/agent-conf-docker-hosts.xml) | The Docker listener for the `docker-hosts` agent group |
| [`files/wazuh-agent.yml`](files/wazuh-agent.yml) | The playbook that installs and enrolls agents on a fresh host |

## Limits

- **The monitor sees hosts that have an agent.** The internet-facing host keeps its own change scanning and has no path to the manager, by design. Appliances that can't run an agent are not covered.
- **File watching is real-time but not attributed.** The critical-file watch uses inotify, so it says what changed and when, not which process changed it.
- **Rules are edited in the dashboard.** [`files/local_rules.xml`](files/local_rules.xml) is the reference copy and is kept in step by hand.
- **The custom-rule example list is representative.** The live rule file has one allowlist rule per healthcheck in the fleet. The copy here keeps one per style.

## Related

- [no-mail-server](https://github.com/zacharyrichter/no-mail-server): the notification shim whose token file is one of the watched paths.
- [expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose): the yearly certificate renewal that raises expected critical-file alerts.
- [backups-nobody-watches](https://github.com/zacharyrichter/backups-nobody-watches): how the Wazuh VM is backed up, system disk only.
- [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit): the fleet audit that checks the Docker hosts' standards.

Everything here is sanitized: domains are `example.net`, addresses are `10.0.x.x`, the human account is `admin`, and tokens and passwords are placeholders.

MIT licensed. See [`LICENSE`](LICENSE).
