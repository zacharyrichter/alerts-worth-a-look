# Verification

## On the Wazuh VM

```bash
# Containers up; Docker tied to the data disk
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
systemctl show docker -p RequiresMountsFor        # RequiresMountsFor=/srv/wazuh-data
findmnt /srv/wazuh-data                           # LABEL=wazuh-data, ext4

# Manager daemons, and every agent connected
sudo docker exec wazuh-manager /var/ossec/bin/wazuh-control status | grep -v 'not running' | head
sudo docker exec wazuh-manager /var/ossec/bin/agent_control -l

# ntfy integration: executable, and recent sends succeeded
sudo docker exec wazuh-manager ls -l /var/ossec/integrations/custom-ntfy   # -rwxr-xr-x
sudo tail -n 3 /srv/wazuh-data/manager-logs/integrations.log              # HTTP 200
```

From the management host, the dashboard answers through its reverse proxy:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://wazuh.example.net/        # 200 or 302
```

The fleet audit ([diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit)) shows `changed=0 failed=0` for the Wazuh host.

## On an agent

```bash
sudo grep ^status /var/ossec/var/run/wazuh-agentd.state                    # status='connected'
```

The rule and source tests are in [`sources.md`](sources.md) and [`tuning.md`](tuning.md), and the alert path tests are in [`runbook.md`](runbook.md).

## When a check fails

| Symptom | Cause and fix |
|---|---|
| `integratord: ERROR: Couldn't execute command ... Check file and permissions` | The script lost its executable bit. Pre Deploy should prevent this. If it recurs, check the stack's Pre Deploy step and run the `chmod` by hand |
| `send failed ... CERTIFICATE_VERIFY_FAILED` | The bundled Python has no CA store, and the script loads `certifi`. Check the `SSL_CTX` block is intact |
| `send failed ... HTTP 401` or `403` | The token was revoked, or the user has the wrong access to the topic on the notification server |
| Nothing in `integrations.log` at all | Run `wazuh-control status` inside the container. `wazuh-integratord` must be running, and it only starts when an `<integration>` block exists |
| The manager restart-loops after a config change | Check `docker logs wazuh-manager` for `mismatched tag`, and revert the file in Git |
| A healthcheck alerts on every interval | It is new or changed. Add its exact `docker.Action` to the allowlist ([`sources.md`](sources.md)) |
| Docker exec alerts flood | Set 100150 to `level="3"`, restart the manager, investigate, then set it back |
