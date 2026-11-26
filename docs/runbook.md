# Runbook

## Change the Wazuh config or the notification script

1. Edit the file in the Git repo. For XML pasted into `wazuh_manager.conf`, check the diff before committing: a stray duplicated line can leave an unclosed tag and put the manager in a restart loop.
2. In the deploy tool, pull the repo.
3. Deploy the `wazuh` stack. Pre Deploy sets the script's executable bit before the containers start.
4. Check for errors, then run the matching checks in [`verification.md`](verification.md):
   ```bash
   sudo docker logs --tail 50 wazuh-manager
   sudo docker exec wazuh-manager ls -l /var/ossec/integrations/custom-ntfy    # -rwxr-xr-x
   ```

## Change a custom rule or an agent group config

Make the change in the dashboard (rules: Server management → Rules; groups: Agents management → Groups), then update the reference copy in [`files/`](../files). The procedure and its pitfalls are in [`tuning.md`](tuning.md).

## Update Wazuh

Merge the grouped Renovate pull request, then deploy the `wazuh` stack by hand. The three images always move together. Agents are upgraded separately, from the manager with `agent_upgrade` ([`agents.md`](agents.md)), and must never be newer than the manager.

## Test the alert path

**The script only**, as the container's `wazuh` user, with a fake alert:

```bash
sudo docker exec -u wazuh wazuh-manager sh -c 'echo "{\"rule\":{\"id\":\"999999\",\"level\":10,\"description\":\"ntfy integration test\",\"groups\":[\"test\"]},\"agent\":{\"id\":\"000\",\"name\":\"wazuh.manager\"},\"full_log\":\"manual test\"}" > /tmp/ntfy-test.json && /var/ossec/integrations/custom-ntfy /tmp/ntfy-test.json "" https://ntfy.example.net/wazuh-alerts; rm -f /tmp/ntfy-test.json'
sudo tail -n 3 /srv/wazuh-data/manager-logs/integrations.log
```

**End to end** (the integration daemon calls the script on a real alert): add a temporary rule to `local_rules.xml`, restart the manager, fire it from an agent, then remove the rule.

```xml
<group name="local,ntfy_test,">
  <rule id="100199" level="10">
    <match>WAZUH_NTFY_TEST</match>
    <description>ntfy integration end-to-end test</description>
  </rule>
</group>
```

On any agent:

```bash
logger "WAZUH_NTFY_TEST end-to-end"
```

## Rebuild the Wazuh VM

Restoring the VM from backup is the normal recovery. The data disk then starts empty or is reattached as it was.

1. Create the VM with Debian, both disks (the second with the backup flag off) and the static address. Follow the new-host steps for the fleet: hardening, the SSH CA, the host certificate and the baseline.
2. Format the second disk as ext4 with the label `wazuh-data`, add `LABEL=wazuh-data /srv/wazuh-data ext4 defaults,nofail 0 2` to `/etc/fstab`, mount it, and create the five folders with the owners and modes in [`design.md`](design.md#data-disk).
3. Add the Docker drop-in at `/etc/systemd/system/docker.service.d/wazuh-data.conf`:
   ```ini
   [Unit]
   RequiresMountsFor=/srv/wazuh-data
   ```
   and set `vm.max_map_count=262144` in `/etc/sysctl.d/`.
4. Install Docker and run the Docker standard for the host before starting any containers.
5. Restore `/opt/appdata/wazuh/` and the stack's env files from the backup. If they are lost, generate new certificates and the security config with Wazuh's single-node tooling, and re-enroll every agent.
6. Clone the Git repo to the deploy tool's stacks folder, start its agent by hand, and reconnect the server in the deploy tool.
7. In the deploy tool, pull the repo and deploy `wazuh`.
8. Install the host's own Wazuh agent.
9. Run every check in [`verification.md`](verification.md).

## Recovery

- **A config change breaks the manager** (a restart loop, `mismatched tag` in `docker logs wazuh-manager`): revert the file in Git, pull, and deploy `wazuh` again.
- **Docker doesn't start after a boot:** the data disk didn't mount. Check `findmnt /srv/wazuh-data` and the disk on the node. Once it is mounted, run `sudo systemctl start docker`.
- **The data disk is lost:** attach a new disk to the VM, format and mount it as in the rebuild above, then start Docker. The manager downloads the vulnerability feed again and alert history starts over. Agents reconnect on their own.
- **The VM is lost:** restore it from backup and reattach or recreate the data disk.
- **ntfy alerts stop:** see the table in [`verification.md`](verification.md).
