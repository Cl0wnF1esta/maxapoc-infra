# Operations

[Русский](../OPERATIONS.md) | **English**

Procedures for the test environment and the production server. Installation from scratch and inventory setup are described in detail in the [README](../../README.en.md); this document covers how to use it and what to do when something goes wrong. The reasons behind decisions are in [DECISIONS.md](DECISIONS.md).

## 1. The main rule for runs

- All commands are run from the `ansible/` folder.
- **Test environment:** `ansible-playbook site.yml` (default inventory).
- **Production server:** only `ansible-playbook -i inventory/prod site.yml`, first with `--check --diff`.
- Environments are run **separately**: the sudo password is asked once per run.
- Ansible takes the MaxApoc files (compose, `gateways.json`, observability) **from a clean clone of `main`** (`~/.cache/maxapoc/app`), which it updates itself as the first task; the working copy is not involved ([ADR-019](DECISIONS.md#adr-019)). Manual changes in the clone stop the run: `git -C ~/.cache/maxapoc/app checkout -- .`.
- The production server must receive **exactly what was verified in the test environment**: after the test environment run, note `git -C ~/.cache/maxapoc/app rev-parse HEAD` and run the production server with `-e maxapoc_repo_version=<hash>`.
- **Ansible does not change the proxy version:** it is rolled out by the deploy from CI (section 5).
- Every unexpected `changed` is investigated **before** the production server.

## 2. Preparing a server from scratch

A clean Ubuntu 24.04 machine becomes a production server in one command; a repeat run changes nothing.

**What is needed in advance**
- **Control node:**
  - `ansible-core` 2.18+;
  - collections (`ansible-galaxy collection install -r requirements.yml`);
  - SSH access to the MaxApoc repository on GitHub (the playbook maintains the `main` clone itself);
  - the vault password for the environment.
- **Target server:**
  - Ubuntu Server 24.04, 1 GB RAM or more, `openssh-server`;
  - **a separate user with password-protected sudo** and the control node's SSH key; root login is not needed and will be disabled after the run.
- **Environment data:**
  - the inventory;
  - `vault.yml` with secrets;
  - a server certificate from the project CA (`ca.key` never reaches the server).

**Run**
```bash
ansible-playbook site.yml --check --diff     # preview (forecast)
ansible-playbook site.yml                    # apply
ansible-playbook site.yml                    # repeat: changed=0 expected
```

**Timing** (test environment, through its HTTP proxy): a full run on a clean virtual machine (VM) — **6 min 26 s** (2026-11-04, all roles, including backups; 2026-11-02 without them — 6 min 05 s; earlier, with the image built on the server — 15 min 25 s); a repeat run without changes — ~1.5 minutes.

**Verifying the result** — README, section "Verification".

**Why a repeat run is safe.** Tasks describe state, not actions. Where commands are needed (swap), they are guarded by a check of the current state. sshd, Docker and stack services restart only when their configs change; monitoring stack restarts run **before** its health check (`flush_handlers`), so the final state is what gets checked. Ansible creates the proxy version (`release.env`) only on a server that does not have one.

**Clean server** ([ADR-007](DECISIONS.md#adr-007), [ADR-017](DECISIONS.md#adr-017)): Ansible takes the proxy version from the **latest backup** of this environment (the `backup` role runs before the proxy role). No backup (new environment, storage unavailable) — the **initial** version from `maxapoc_proxy_image` in the inventory, with a warning in the output; the first deploy then brings the current one. Keep the initial version fresh anyway: a very old image may not match the current compose.

```bash
ssh -q -o IdentitiesOnly=yes -i <environment_deploy_key> maxapoc-deploy@<server_address> 'deploy sha256:<digest of the current version>'
```

Verified 2026-11-02: clean VM → one run without errors → first deploy (1 min 09 s — before the health check was sped up, now ~7 s) → repeat run `changed=0`. 2026-11-04: a deleted `release.env` came back from the backup (the version from the snapshot, not from the inventory); with backups disabled — from the inventory; on a running server the task is skipped.

## 3. Migrating a running server to this code

The first run on a server where something is already running is a **migration**: for every task you need to know what will match, what will be overwritten and what will restart. The procedure used to migrate the production server on 2026-10-28:

**Preparation (in advance, during the day)**
1. **Read-only reconnaissance:** OS, swap, `ufw status numbered` and `verbose`, `ss -tulpn`, `sshd -T`, apt sources, `daemon.json`, `apt-mark showhold`, Docker packages, containers, `docker stats`, Docker networks.
2. **A user instead of root:** create it, add the key, verify **key-only** login from a second session; do not close the first one. Check the hosting provider's console (fallback access) and the root password in the password manager.
3. **Compare secrets** with vault by hash (secrets are not printed), **compare code** by file hashes. If the code differs, the migration also becomes an application update.
4. **Rehearsal** in the test environment with a copy "as on the production server" (same directory, user, code); measure downtime and test rollback.
5. **Preview** `-i inventory/prod --check --diff`: every `changed` is explained. The preview does not see conflicts between files (apt, sshd) — they are found during reconnaissance.
6. A VPS snapshot or backup at the hosting provider.

**During the work** (at minimum load hours according to the request graph)
- **Terminals:**
  - a working session on the server (keep it open);
  - the playbook run;
  - a proxy availability counter;
  - new SSH connections for checks.
- Container list **before**: `sudo docker ps --format '{{.Names}}' | sort > ~/containers-before.txt`.
- Rollback safeguards: a tag on the old image, copies of files being replaced.
- Run with a log written **outside the repository**.

**Checks afterwards — in order, roll back that layer on failure**
1. SSH: the key works; password and root are rejected; `sshd -T` → `no` for `permitrootlogin`, `passwordauthentication`, `kbdinteractiveauthentication`.
2. ufw: policies, rule list; removal of old rules; a new SSH connection right away.
3. Docker and apt: `daemon.json`, `apt-mark showhold`, `apt-get update` without errors.
4. **All containers are back:** `sudo docker ps --format '{{.Names}}' | sort | diff ~/containers-before.txt - && echo ALL_BACK`.
5. Proxy: image, `healthy`, uid, log without permission errors, downtime by the counter, TLS with `--cacert`.
6. Everything that depends on the server: VPN, panel, WireGuard, the hosting provider's Zabbix, monitoring, a request from the application.
7. Repeat run — `changed=0`.

**Rollback by layer**

| What broke | Rollback (from the working session or the hosting provider's console) |
|---|---|
| Proxy | `sudo docker compose -f /opt/maxapoc/proxy/vps/docker-compose.yml down && sudo docker compose -f <old directory>/docker-compose.yml up -d` |
| A needed port was closed | `sudo ufw allow <port>/<proto>`; as a last resort `sudo ufw disable` from the console |
| No SSH login | `sudo rm /etc/ssh/sshd_config.d/00-hardening.conf && sudo systemctl reload ssh` |
| Docker after `daemon.json` | `sudo rm /etc/docker/daemon.json && sudo systemctl restart docker` |
| apt | restore the saved source file |

After rolling back a layer — find the cause first, then run again: otherwise the run brings back the same breakage.

**Cleanup after 1–2 weeks** (if there were no issues during that time)
```bash
sudo docker ps --format '{{.Names}} {{.Label "com.docker.compose.project.working_dir"}}' | grep maxapoc-proxy   # /opt/...
sudo rm -rf <old proxy directory> <saved file copies>
sudo docker network rm vps_default
sudo docker image rm vps-maxapoc-proxy:pre-ansible vps-maxapoc-proxy:latest
```
**No** `docker compose down` from the old directory: the old and new directories share one compose project name, so the command would stop the current proxy.

Cleanup on the production server was done on 2026-11-02 (change log).

## 4. Docker work on the server

- Any work that may restart the Docker daemon (changing `daemon.json`, package upgrades): container list before and comparison after (section 3, step 4).
- `live-restore` (role `docker`): a daemon restart does not stop containers. Enabled on both servers; on the production server — 2026-10-30 without a daemon restart: `daemon.json` written by hand exactly as the role template, `sudo dockerd --validate --config-file /etc/docker/daemon.json`, `sudo systemctl reload docker` ([ADR-011](DECISIONS.md#adr-011)).
- **Images — no `docker image prune -a`:** it would delete the previous image that rollback without the registry relies on. The deploy itself keeps only the current and the previous image; delete manually only by an explicit list (`docker image rm <image>`).
- **Manual proxy compose — only with both variable files:** `cd /opt/maxapoc/proxy/vps && sudo docker compose --env-file .env --env-file release.env <command>`. Without them compose stops with a hint.
- Check: `sudo docker info --format '{{.LiveRestoreEnabled}}'`.
- Compare container restarts by `docker inspect --format '{{.State.StartedAt}} {{.RestartCount}}'`, not by the `Up …` line in `docker ps`: it is rounded.

<a id="deploy"></a>
## 5. Proxy deploy

**Flow.** All changes go `dev` (checks) → `main`. A merge to `main` that changes `proxy/**` or `.github/workflows/proxy.yml` **rolls the proxy out to the production server by itself**: `test` → `image` (image in GHCR, digest in the run summary) → `deploy` (environment `production`, check "the commit is in `main`", `deploy <digest>` over SSH). Changes only in `observability/**` do not rebuild the image. The APK build is not triggered by proxy changes. Therefore `dev` holds only what can be rolled out ([ADR-021](DECISIONS.md#adr-021)).

**Urgent proxy fix** when `dev` holds something that must not be released: branch from `main` → merge to `main` (deploy) → merge `main` into `dev`.

**What the server does** ([ADR-018](DECISIONS.md#adr-018)): pulls the image → saves the previous version to `release.env.prev` → writes the new one to `release.env` → recreates the container and waits up to 60 s for healthy → `OK` and removal of extra images, or automatic rollback. Downtime is about 1 s for the recreation; confirmation takes a few seconds.

**Deploy result** (CI annotation and script exit code):

| Code | Meaning | What to do |
|---|---|---|
| 0 | rolled out, healthy | post-deploy check (below) |
| 1 | invalid input (did not pass the filter) | check that the digest from the `image` job is passed |
| 2 | the image was not pulled, **nothing changed** | GHCR access and the registry token lifetime on the server |
| 3 | another deploy is in progress | wait and retry |
| 4 | the new version did not become healthy, **rollback done** | `docker logs`, fix, release a new version |
| 5 | rollback failed — **production needs attention** | log in to the server, section 6 |
| 255 | SSH did not connect | `DEPLOY_HOST`, `DEPLOY_KNOWN_HOSTS` (did the host key change?), ufw `LIMIT` |

**Post-deploy check** ([ADR-022](DECISIONS.md#adr-022)) — for 15 minutes after the rollout, watch the Telegram alerts and the proxy dashboard. On the production server the proxy alerts (`MaxapocServerErrors`, `MaxapocProxyDown` and others) are evaluated by the `hobot-metrics` stack: its copy of the rules is updated manually on every change to `observability/prometheus/` in MaxApoc.

**What is running now** — a manual run of the `Proxy` workflow from the `main` branch (Actions → Run workflow): the `deploy-check` job prints the image and state from the server; a manual run rolls nothing out. On the server:
```bash
sudo docker inspect maxapoc-proxy --format '{{.Config.Image}} {{.State.Health.Status}}'
sudo cat /opt/maxapoc/proxy/vps/release.env /opt/maxapoc/proxy/vps/release.env.prev
sudo journalctl -t maxapoc-deploy-gate -t maxapoc-deploy-run --since today --no-pager
```

**The test environment** is not reachable from CI (NAT); there the deploy is done manually from the control node with the same command CI uses:
```bash
ssh -q -o IdentitiesOnly=yes -i ~/.ssh/maxapoc_deploy_test maxapoc-deploy@192.168.56.10 'deploy sha256:<digest>'
```
`IdentitiesOnly=yes` is mandatory for key tests: without it the client also offers other keys from the agent, and the "a foreign key is rejected" test gives a false result.

## 6. Rollback

**Image one step back.** Deploy the previous digest the same way — there is no separate command. The previous digest is in `release.env.prev`, the image is already on the server (the registry is not needed).

**Further back than one step.** The digest of the needed version: the server log (`journalctl -t maxapoc-deploy-run` — every `deploy … OK` line is a version that actually ran), the `Proxy` run summaries in GitHub, the package page. Then `deploy <digest>`; the image is pulled from the registry.

> Rolling back the proxy is safe only if the proxy is backward compatible with the released application versions.

**Compose file** (delivered by Ansible from `main`):
1. Urgent: `ansible-playbook -i inventory/prod site.yml -e maxapoc_repo_version=<hash of a good commit>` — temporary, the server differs from `main`, the next regular run brings back the bad compose.
2. Make it stick: `git revert <bad commit>` → `dev` → `main` → a regular Ansible run.

**Contract change** (a new variable in `.env` and compose + code that uses it) — two releases: first compose and `.env` compatible with the old image (merge + Ansible), then the code ([ADR-007](DECISIONS.md#adr-007)).

<a id="break-glass"></a>
## 7. Emergency release with the registry unavailable (break-glass)

When GHCR is unavailable (or the server's token has expired) and a fix is needed now. The CI key is not suitable for this and must not be: the procedure runs under a personal user with sudo, every `sudo` is logged.

1. On a machine with Docker, in the MaxApoc working copy, on the commit with the fix: `cd proxy && node --test` (the image bypasses CI — run the tests by hand).
2. Build from the repository root: `docker build -f proxy/vps/Dockerfile -t maxapoc-proxy:breakglass-<commit> proxy`.
3. Transfer without a file: `docker save maxapoc-proxy:breakglass-<commit> | ssh <user>@<server> 'sudo docker load'`.
4. On the server, in `/opt/maxapoc/proxy/vps`: save the current `release.env` to `release.env.prev`, write `MAXAPOC_PROXY_IMAGE=maxapoc-proxy:breakglass-<commit>` (a loaded image has no registry digest — only a tag).
5. `sudo docker compose --env-file .env --env-file release.env up -d --no-build --wait`.
6. When the registry is back: the same fix the regular way `dev` → `main` → deploy by digest; delete the emergency image (`docker image rm`).

<a id="backup"></a>
## 8. Backups

**What and where** ([ADR-023](DECISIONS.md#adr-023), [ADR-024](DECISIONS.md#adr-024)): the `backup` role, restic → the hosting provider's private S3 bucket, each environment has its own repository (`test/`, `prod/`). What gets backed up is set by the `backup_sqlite_dbs` and `backup_paths` variables in the inventory. A new service or state file on the server must be added there, otherwise it is not backed up.

**When:** daily, `backup.timer` (03:30 + up to 30 min, server time); on Sundays additionally `restic check`. On failure — the message "Резервная копия на … НЕ выполнена" ("Backup on … FAILED") in the environment's Telegram.

**On the server:**
```bash
systemctl list-timers backup.timer --no-pager                         # next and last run
sudo systemctl start backup.service && echo ok                        # off-schedule backup
journalctl -u backup.service -n 30 --no-pager                         # result of the last backup
sudo /usr/local/sbin/restic-env snapshots                             # snapshots of this environment
sudo /usr/local/sbin/restic-env ls latest                             # what is in the latest snapshot
sudo /usr/local/sbin/restic-env stats --mode raw-data                 # space used in the storage
```

**Monthly, from the control node** — the secondary backup copy and a light check. Settings are in `~/.config/restic/prod.env` and `prod.pass` (`0600`; S3 keys and the production server password from the password manager), the local repository is `~/backups/restic-prod`.
```bash
( set -a; . ~/.config/restic/prod.env; set +a; restic snapshots --latest 1 )   # when the secondary copy was last made
( set -a; . ~/.config/restic/prod.env; set +a; restic copy )
( set -a; . ~/.config/restic/prod.env; set +a; restic forget --keep-monthly 12 --prune && restic check )
( set -a; . ~/.config/restic/prod.env; set +a; restic restore latest --target /tmp/home-restore --include /var/backups/restic-staging/x-ui.db )
sqlite3 -readonly /tmp/home-restore/var/backups/restic-staging/x-ui.db 'PRAGMA integrity_check; SELECT count(*) FROM inbounds;'
rm -rf /tmp/home-restore
```
Expected: `ok` and the number of inbounds as in the panel. The commands in parentheses run in a subshell: the secrets do not stay in the terminal's variables.

**Bucket:** versioning is enabled; noncurrent versions are deleted after 30 days ([ADR-025](DECISIONS.md#adr-025)). A deleted or corrupted object can be restored from a previous version within that period using the hosting provider's S3 tools.

**What not to do:** enable expiry of **current** objects or object lock in the bucket rules — the first destroys the repository, the second stops the backups.

<a id="restore"></a>
## 9. Restoring the server from a backup

Scenario — reinstalling **the same** VPS (the address is kept). A new address — see below. Targets: RPO up to a day, RTO under an hour ([ADR-027](DECISIONS.md#adr-027)).

> Practised in drills: clean VM → Ansible → backup restored into a separate directory → comparison. Steps 1 and 4–7 (reinstall, putting files in place, starting x-ui, WireGuard and `hobot-metrics`, bringing back the latest version) have not been drilled end to end; picking the proxy version from the backup in step 2 has been verified.

1. **Hosting provider:** reinstall Ubuntu 24.04 on the same VPS. First login — the hosting provider's console; create a user with a key and sudo (README, step 2 "Target server").
2. **Ansible:** `ansible-playbook -i inventory/prod site.yml`. It brings up the base, hardening, Docker, the `backup` role (the repository already exists — no initialisation), the proxy and deploy access. The role takes the proxy version (`release.env`) **from the latest backup** of this environment by itself; if the backup is unavailable — from the initial version in the inventory, with a warning in the output.
3. **Stop backups until the restore is finished:** `sudo systemctl stop backup.timer`. (A backup of an empty server fails with an alert — there is no x-ui database — but better not to let it happen.)
4. **Extract the backup** into a separate directory:
   ```bash
   sudo /usr/local/sbin/restic-env snapshots --latest 3
   sudo /usr/local/sbin/restic-env restore latest --target /srv/restore
   ```
5. **Put in place and start** (all as root, source — `/srv/restore`):
   - **proxy:** already on the version from the backup (step 2). If the step 2 output had the warning "release.env не получен из копии" ("release.env not obtained from the backup") — copy `release.env` from `/srv/restore/opt/maxapoc/proxy/vps/` to `/opt/maxapoc/proxy/vps/` and run Ansible again;
   - **WireGuard:**
     1. `apt install wireguard`;
     2. `/etc/wireguard` from the backup;
     3. `chmod 600 /etc/wireguard/vps_private.key /etc/wireguard/wg0.conf` (`wg0.conf` also contains the private key);
     4. `systemctl enable --now wg-quick@wg0`;
     5. `wg show` — handshakes with the peers.
   - **x-ui:**
     1. with the 3x-ui installer of **the same version (3.10.0)**;
     2. `systemctl stop x-ui`;
     3. `/etc/x-ui/x-ui.db` from `var/backups/restic-staging/x-ui.db`; delete the old `x-ui.db-wal` and `-shm`;
     4. `/root/cert` and `/root/.acme.sh` from the backup;
     5. `systemctl start x-ui`; the panel shows the same inbounds. The x-ui bot reconnects by itself (single instance).
   - **`hobot-metrics`:**
     1. `/root/hobot-metrics` from the backup;
     2. `docker compose up -d` (creates the volumes), then `docker compose stop`;
     3. Alertmanager and Prometheus data — into the volume directories; `grafana.db` — into the Grafana volume, owner `472`;
     4. `docker compose up -d`; Prometheus targets `up`. Grafana plugins are installed on startup.
   - **The hosting provider's Zabbix agent** — following the provider's instructions.
6. **Bring back the latest proxy version.** The backup brought the version that was running at backup time; if there was a deploy after it, the server now runs the previous version. Bring it back in one of two ways:
   - Actions → the latest `Proxy` run on `main` → Re-run all jobs (a `push` event — the deploy runs; the image is rebuilt from the same commit);
   - `deploy <digest>` with the deploy key (digest — from that run's summary or the package page).

   Forgetting this is not dangerous: the next merge to `main` rolls out the current version, and the previous version is compatible with the released application.
7. **Verify and re-enable backups:**
   - proxy healthy on the right version;
   - the x-ui panel and a VPN client connection;
   - `wg show`;
   - `hobot-metrics` targets and alerts;
   - `sudo systemctl start backup.service` → `backup OK`;
   - `sudo systemctl start backup.timer`.
8. **Remove** `/srv/restore` (it contains keys and the VPN database): `sudo rm -rf /srv/restore`.

**New address** (a new VPS): additionally a new proxy server certificate for the new IP, a new `PROXY_URL` and an application release (users have to update), the x-ui panel certificate, new settings for VPN clients and WireGuard peers, `DEPLOY_HOST` and `DEPLOY_KNOWN_HOSTS` in GitHub. This takes days, not hours.

**Full drills** — once a quarter and on upgrades or rollbacks of x-ui, Grafana, Prometheus versions or changes to what gets backed up: as steps 2 and 4 on a clean test environment VM, with the production server password in a temporary file (`0400`, then `shred -u`) and comparison of hashes and database contents with the live server. WireGuard, x-ui and the `hobot-metrics` Alertmanager from the backup are **not started** in the test environment.

## 10. Rotating secrets

A secret is changed in the environment's `vault.yml` (`ansible-vault edit`), followed by a playbook run for that environment.

| Secret | What to do | Consequences |
|---|---|---|
| Telegram bot token | new token from @BotFather (`/revoke` for the old one) → vault → run | Alertmanager restarts by itself; check with a test alert |
| Grafana password | vault → run (the Grafana container must be running) | the role changes the password with `grafana cli admin reset-admin-password`; check: old password `401`, new `200` |
| Proxy GitHub token (`GH_TOKEN`, `HUB_TOKEN`) | new token → vault → run → revoke the old one | the container is recreated with the new `.env`; the application is not rebuilt |
| Application key `APP_KEY` | vault → run **and** the application repository secret → rebuild | installed applications with the old key stop working with the proxy |
| LLM gateway keys | vault → run | the container is recreated |
| Server certificate (renewal, address change) | `ca.key` from the password manager into a temporary folder → `make-certs.sh <address> <folder>` → `server.crt`/`server.key` into vault → run → `shred -u <folder>/ca.key`; without `ca.key` in the folder the script creates a new CA | the handler restarts the proxy; on an address change — a new `PROXY_URL` and an application rebuild |
| Registry token on the server (`vault_maxapoc_proxy_registry_token`, classic PAT `read:packages`, 90 days; reminder 7 days ahead) | new token → the environment's vault → run → revoke the old one | without the token the server cannot pull a new image (deploy — code 2), the running proxy is not affected |
| Deploy key (CI) | `ssh-keygen` a new key → public part into `maxapoc_deploy_pubkey` in the inventory → run (the old key goes away, `exclusive`) → private part into the environment secret `DEPLOY_SSH_KEY` → manual `Proxy` run from `main` (`deploy-check`) | the test environment and the production server have different keys |
| Server host key (reinstall) | verify the new fingerprint **through the hosting provider's console** (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`), then the line in `DEPLOY_KNOWN_HOSTS` | until updated, the deploy fails with code 255 — by design |
| The environment's restic repository password | new password: `restic-env key add` (new key) → password into vault and the password manager → Ansible run → `restic-env key list`, `key remove` the old one | without the password backups cannot be decrypted — a copy in the password manager is mandatory |
| S3 bucket keys | new keys from the hosting provider → vault of **both** environments and `~/.config/restic/prod.env` on the control node → Ansible run on both | one key per bucket |
| The environment's vault password | `ansible-vault rekey --vault-id <id>@<old> --new-vault-id <id>@<new> <files>`, then replace the password file | re-encrypt first, replace afterwards |

**Losing an environment's vault password.** Re-encrypting without it is impossible. All secrets of that environment have to be reissued (GitHub tokens, gateway keys, `APP_KEY` with an application rebuild, the certificate, the bot token, the Grafana password) and a new `vault.yml` created. That is why a copy of the password in the password manager is mandatory.

**A Grafana password changed in the UI** goes unnoticed by the playbook (the password file does not change) — this is drift. It is fixed by changing the password again through vault.

**Checking alert delivery** (after changing the token or the chat): put the test rule from the MaxApoc repository (`observability/prometheus/maxapoc-test-alerts.yml.example`, saved as `maxapoc-test-alerts.yml`) into the rules directory and reload the configuration. A 🔴 arrives within 1–2 minutes; after removal and reload — 🟢 (within 5–10 minutes). The role does not delete the file — it must be deleted.
```bash
sudo install -o root -g root -m 0644 maxapoc-test-alerts.yml /opt/maxapoc/monitoring/prometheus/rules/
sudo docker kill -s HUP maxapoc-prometheus
sudo rm /opt/maxapoc/monitoring/prometheus/rules/maxapoc-test-alerts.yml
sudo docker kill -s HUP maxapoc-prometheus
```

## 11. Common problems

| Symptom | Cause and action |
|---|---|
| `Incorrect sudo password` on the first task | `BECOME password` entered incorrectly (environments have different passwords) |
| `Host key verification failed` | the server address changed or the server was reinstalled → log in over SSH manually and verify the key fingerprint; do not delete the `known_hosts` entry blindly |
| `Decryption failed` for `vault.yml` | wrong password file for the vault label (`…;test` / `…;prod`) or the file was replaced |
| `'…' is undefined` for a secret | `vault.yml` is not in `host_vars/<host name>/` (the folder name must match the host alias) |
| timeout when downloading the Docker key, images, sending to Telegram | no access to the external source → `curl` from the server; in the test environment — the proxy via `proxy_env` |
| container `Restarting (1)` / `unhealthy` | the application crashes at startup → `sudo docker logs <name> --tail 30` (example: `APP_KEY` shorter than 16 characters) |
| target `down`, `connection refused` | the process is not listening on the port: check the container startup log and which image is running (`docker inspect --format '{{.Config.Image}}'`) |
| `curl … Expecting value` / `Failed to connect` to 9090, 9093, 3000 | the SSH tunnel is not open or does not forward this port → `ss -tlnp \| grep ssh`; or check directly on the server via `127.0.0.1` |
| no messages in Telegram | `docker logs maxapoc-alertmanager --tail 30`: `timeout` — no network to Telegram, `401` — token, `400 chat not found` — chat ID |
| `MAXAPOC_PROXY_IMAGE is not set` | compose started without the variable files → `--env-file .env --env-file release.env` |
| `Local modifications exist` on the first task | something in the `~/.cache/maxapoc/app` clone was edited by hand → `git -C ~/.cache/maxapoc/app checkout -- .` |
| `deploy-check` / `deploy` "not allowed to deploy to production" | the run is not from `main` — the `production` environment rule |
| deploy: code 2, 4, 5, 255 | section 5, results table |
| a key from the "other" environment suddenly lets you onto the server | test without `IdentitiesOnly=yes`: the agent offered another key; check with `-v` (`Server accepts key`) |
| a handler failed, the repeat run is green | a handler notification is lost when a run fails → test a fixed handler so that it fires again |
| "ИИ и хаб" ("AI and hub") dashboards are partly empty, `MaxapocHubSnapshotMissing` fires | the hub repository is not configured (`HUB_REPO`, `HUB_TOKEN`) |
| The Grafana password was changed in the UI, the playbook does not see it | the password file does not change; the source of truth is vault: restore the password through vault and a run |
| "Резервная копия … НЕ выполнена" ("Backup … FAILED") | `journalctl -u backup.service -n 50`: no database (`unable to open database`), no access to S3, bucket full, password |
| `sudo du /var/lib/docker/volumes/…_*` — "No such file" | the glob is expanded by the unprivileged user's shell → `sudo sh -c 'du -sh …_*'` |
| `cd` into the restored directory — Permission denied | the directory belongs to root; `cd` is a shell builtin and does not work under sudo → `sudo sh -c 'cd … && …'` |
| `promtool tsdb list` — read-only file system | it writes its working directory into the data directory even when reading → only on a disposable copy, without `:ro` |
| ad-hoc `ansible -m debug` does not see a role variable | role `defaults` are visible only when the role runs in a playbook; ad-hoc sees only the inventory |

## 12. Change log

| Date | Environment | What | Result |
|---|---|---|---|
| 2026-10-28 | production server | migration to Ansible: sshd, ufw `default deny`, Docker (version hold, `daemon.json`), proxy in `/opt` as uid 1000 | done; proxy downtime 4 s; repeat run `changed=0`; incident below |
| 2026-10-30 | production server | planned window: `live-restore` via reload (no daemon restart); proxy recreated for a new `.env` line | done; proxy downtime < 1 s (not captured by a 1 s step measurement); `hobot-metrics` containers not affected |
| 2026-10-30 | production server | deploy user `maxapoc-deploy` (role `maxapoc_deploy_access`), separate key; `production` environment in GitHub | done; containers not affected |
| 2026-11-02 | production server | cleanup of the old proxy (directory in `/root`, network `vps_default`, images `vps-maxapoc-proxy:*`, file backups); switch to pulling the image from GHCR by digest | done; downtime ~1 s; `hobot-metrics` not affected |
| 2026-11-02 | production server | first deploy from CI (`main` fast-forwarded to `dev`, Build APK disabled for the merge) | `OK`, ~1 s downtime; no application release created |
| 2026-11-02 | both | build-on-server mode removed; code files and `maxapoc-proxy:local` deleted | done; repeat runs `changed=0` |
| 2026-11-03 | both | proxy health check: `start_period` 30 s, `start_interval` 2 s (Ansible, proxy recreated) | deploy on the production server — 7 s instead of ~65 s |
| 2026-11-03 | production server | deploy only on `push` to `main`; two images kept | verified: 2 of 4 images left; a manual run only reads |
| 2026-11-04 | both | `backup` role: restic → S3, timer, failure alert; bucket — versioning, lifecycle 30/7 days | first production server backup 10 s, 3.5 MiB in the storage; restore verified |
| 2026-11-04 | production server | permissions of `/etc/wireguard/vps_private.key` and `wg0.conf` (it also contains the private key) → `0600` (by hand: WireGuard is outside Ansible) | done |
| 2026-11-04 | test environment | drill: clean VM (6 min 26 s) → restore of the production server backup (2.7 s) → comparison | 7 files, the x-ui database and Prometheus history matched; Grafana started with its database |
| 2026-11-04 | both | initial proxy version in the inventory → current (`f16b4f2a…`) | `changed=0` (the version on the servers does not change) |

### Incident 2026-10-28: the neighbouring stack's Prometheus did not come back after a Docker restart

- **What happened.** The `docker` role created `daemon.json` and restarted the daemon. All containers came back except `hobot-prometheus` (the `hobot-metrics` stack).
- **Impact.** ~16 minutes without `hobot-metrics` metrics. The proxy and users were not affected.
- **Detection.** By empty Grafana dashboards, not by the plan's check: the "all containers are up" check was done by eye.
- **Resolution.** `docker start hobot-prometheus`.
- **Cause.** Not established: the container stopped cleanly (code 0, no out-of-memory), policy `unless-stopped`; after starting, the daemon did not attempt to start it, and the daemon log does not explain this.
- **Measures.** Comparing the container list before and after in the work plan (section 3); `live-restore` ([ADR-011](DECISIONS.md#adr-011)) — a daemon restart no longer stops containers.
