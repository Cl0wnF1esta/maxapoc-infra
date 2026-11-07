# Decisions (ADR)

[Русский](../DECISIONS.md) | **English**

Each decision is recorded in the format "context → decision → alternatives → consequences → how it was verified". The status shows whether the decision has been applied and where; anything not verified is marked explicitly.

| # | Decision | Status |
|---|---|---|
| [001](#adr-001) | Server configuration as code | accepted, applied to the production server 2026-10-28 |
| [002](#adr-002) | Docker from the official repository, pinned versions | accepted, applied |
| [003](#adr-003) | Docker bypasses ufw: access is restricted by the bind address | accepted, applied |
| [004](#adr-004) | Secrets: Ansible Vault, separate passwords per environment, files instead of environment variables | accepted, applied |
| [005](#adr-005) | Environments: separate inventories, shared variables next to the playbook | accepted, applied |
| [006](#adr-006) | A separate monitoring stack for the project; disabled on the production server for now | accepted; disabled on the production server |
| [007](#adr-007) | The boundary between Ansible and the deploy | accepted, implemented (production server — 2026-11-02) |
| [008](#adr-008) | `gateways.json` is application configuration, its source is the project | accepted, applied |
| [009](#adr-009) | Production server firewall | accepted, applied 2026-10-28 |
| [010](#adr-010) | Migrating the running proxy to Ansible | accepted, done 2026-10-28 |
| [011](#adr-011) | Docker daemon `live-restore` | accepted; production server — 2026-10-30 (no container restarts) |
| [012](#adr-012) | External dependencies: retries and the test environment proxy | accepted, applied |
| [013](#adr-013) | The test environment mirrors the production server but shares no secrets with it | accepted |
| [014](#adr-014) | The proxy CA private key is kept off the server | accepted, done |
| [015](#adr-015) | The proxy image is built in CI, the server pulls it by digest | accepted, production server — 2026-11-02 |
| [016](#adr-016) | CI access to the server: a separate user, a forced command, the `production` environment | accepted, applied 2026-10-30 |
| [017](#adr-017) | The image version lives in `release.env`, owned by the deploy | accepted, applied |
| [018](#adr-018) | Deploy script: pull before switching, wait for healthy, roll back | accepted, verified (including with a broken deploy) |
| [019](#adr-019) | The source of files for the servers is a clean clone of `main` | accepted, applied 2026-11-02 |
| [020](#adr-020) | The fallback build-on-server mode was removed | accepted, done 2026-11-02 |
| [021](#adr-021) | Application and proxy in one repository; `dev` → `main` flow | accepted |
| [022](#adr-022) | Post-deploy verification is a process, not code | accepted (accepted risk) |
| [023](#adr-023) | Backups: restic to the hosting provider's S3 and a monthly secondary backup copy | accepted, applied 2026-11-04 |
| [024](#adr-024) | State is backed up, not code | accepted |
| [025](#adr-025) | Bucket versioning, no object lock | accepted |
| [026](#adr-026) | fail2ban is not part of Ansible | accepted |
| [027](#adr-027) | Recovery: RTO/RPO targets and drills | accepted, drill 2026-11-04 |

---

<a id="adr-001"></a>
## ADR-001. Server configuration as code

**Status:** accepted. Verified on a test virtual machine (VM) restored from a snapshot; the production server was migrated on 2026-10-28.

**Context.** The server was configured by hand: packages, the firewall, Docker and the proxy were installed with commands, the proxy was updated with `rsync` + `docker compose up -d --build`. The default ufw policy was "allow". Restoring the server after a failure or handing the project over to someone else was possible only from the author's memory. Resources are limited: 1 GB RAM.

**Decision.** Describe the server as Ansible code and deploy it with one command.

1. **Roles by area of responsibility:** `common` (base), `security` (SSH and firewall), `docker`, `backup` (backups), `maxapoc_proxy` (application), `maxapoc_deploy_access` (deploy access), `maxapoc_monitoring` (monitoring stack). Role variables are prefixed with the role name. A role is shared by all environments; differences are only in inventory data.
2. **Environments differ in data, not in code.** Defaults are in the role's `defaults`; addresses, ports, swap size, proxy are in the host's `host_vars`. Role code does not know the environment name. A variable needed by several roles (the shared network name) lives in the shared variables, not in the `defaults` of one of the roles: otherwise the second role would work "by accident", thanks to the order of roles in the playbook.
3. **Idempotency as a requirement.** Modules that describe state instead of commands; where commands are unavoidable — a check of the current state before acting (not "the file was just created" but "the file is formatted as swap"). Restarts go through handlers and only on changes.
4. **The supported platform is declared explicitly:** on any OS other than Ubuntu 24.04 the playbook stops at the first task with a clear message.
5. **Ansible logs in as a separate user with password-protected sudo,** not as root and not in the `docker` group.

**Alternatives**
- *Shell scripts.* Easier to start, but idempotency and repeat runs would have to be handled by hand everywhere.
- *Manual setup following instructions.* Not checked automatically and drifts away from reality.
- *`synchronize` (rsync) for file delivery.* Rejected: it runs `sudo` in a separate connection with no way to enter a password and disables host key checking by default (`verify_host: false`).
- *Passwordless sudo.* Rejected: a stolen SSH key would immediately give root.

**Consequences**
- (+) The server can be reproduced from scratch; changes go through git.
- (+) Secrets are stored in plain text neither in the repository nor in the output (see [ADR-004](#adr-004)).
- (−) A manual change on the server is now drift: the next run either restores the state from code or (for what the role does not describe) does not notice the change.
- (−) Roles are additive only: a ufw rule or a file removed from a list stays on the server until removed by hand.

**How it was verified**
- Every check starts from the test VM rolled back to the snapshot "clean OS + user + key".
- Clean VM → production server: 6 min 26 s (2026-11-04, all roles, including backups; through the test environment proxy; the proxy image is pulled pre-built), `failed=0`; repeat run `changed=0`.
- An interrupted run (swap file without formatting) — the role recovers by itself.
- Changing one code file → image rebuild → the next run `changed=0`.
- No leaks: a search for made-up secret values in the run log with `--diff` and across the repository — empty; positive control on the server — the values are in place.
- Production server: after the migration a repeat run gives `changed=0`.

---

<a id="adr-002"></a>
## ADR-002. Docker from the official repository, pinned versions

**Status:** accepted, applied to both environments.

**Context.** On the production server Docker was installed from the official repository (`docker-ce` 29.8.2 and related packages), but the versions were not held: any `apt upgrade` could upgrade the daemon and restart all containers.

**Decision**
- The `download.docker.com` repository in deb822 format with a `signed-by` key (the apt chain of trust verifies package signatures).
- All related packages (`docker-ce`, `docker-ce-cli`, `containerd.io`, `buildx`, `compose`) are installed in one transaction at the same versions as on the production server and held (`hold`). Upgrading is deliberate: raise the versions in the role's `defaults` and allow changes to held packages.
- The role removes the old `docker.list` (created when installing per the instructions on the Docker website): otherwise the same repository is described twice.
- The role does not install `docker-model-plugin`. On the production server it is preinstalled by the hosting provider and unused; we do not remove it — this is a recorded difference between environments.

**Alternatives**
- *The `docker.io` package from the Ubuntu repository.* Rejected: different versions from the production server.
- *The latest version without pinning.* Rejected: the server stops being reproducible, upgrades happen uncontrolled.

**Consequences**
- (+) The test environment and the production server run the same versions.
- (−) Docker security updates do not arrive by themselves; they have to be tracked and versions raised by hand.

**How it was verified:** `apt-mark showhold` on the production server after the migration — all Docker packages are held; `apt-get update` without errors and without duplicate source warnings.

---

<a id="adr-003"></a>
## ADR-003. Docker bypasses ufw: access is restricted by the bind address

**Status:** accepted, applied.

**Context.** Ports published by Docker (`ports:`) are opened by forwarding rules in iptables, bypassing ufw rules. An experiment in the test environment: a port published without an address is reachable from outside while the port is closed in ufw. The decorative `DENY` rules for 3000/9090 on the production server protected nothing.

**Decision**
- Services that must not be public are published on a specific address only:
  - `127.0.0.1` — this repository's monitoring stack, access via SSH tunnel;
  - the WireGuard address — proxy metrics for the `hobot-metrics` stack.
- Services for which the Docker network is enough (proxy metrics for this stack's Prometheus) do not publish a port at all.
- Protection comes from the bind address, not the firewall; no ufw rules are written for such ports.

**Alternatives**
- *Rules in the `DOCKER-USER` chain.* They work, but it is a second place where access is described, easily out of sync with ufw.
- *Disable Docker's iptables management.* Rejected: it breaks container networking.

**Consequences**
- (+) A monitoring port without a password is unreachable from the internet regardless of ufw.
- (−) Every new `ports:` has to be checked for an address; a port without an address is open to everyone.

**How it was verified:** `ss -tlnp` in the test environment — 9090, 9093, 3000, 9101 only on `127.0.0.1`; on the production server 9101 only on the WireGuard address.

---

<a id="adr-004"></a>
## ADR-004. Secrets

**Status:** accepted, applied.

**Decision**
1. **Ansible Vault.** Each host has an encrypted `vault.yml` in git; its variables are prefixed with `vault_`, roles read them through references in the plain `vars.yml` — names are searchable, values are not.
2. **A separate password for each environment** (vault IDs `test` and `prod`). A leak of the test environment password does not reveal the production server secrets. The passwords are in files outside git (permissions `0600`) and in a password manager.
3. **`no_log`** on all tasks with secrets: values do not reach the output, including with `--diff` and in future CI logs.
4. **Secrets are passed to containers as files where the application supports it** (the Telegram token for Alertmanager, the Grafana password): permissions `0400`, owned by the container process user. The file does not show up in `docker inspect`, in process environments or in child processes. The proxy still reads secrets from `.env` (`root`, `0600`): switching to files requires an application change.
5. **Comparing secrets without revealing them** — by SHA-256 hashes (short values are not compared this way: they can be brute-forced).

**Alternatives**
- *One vault password for everything.* Rejected: the test environment password is stored on a lab machine.
- *Secrets in container environment variables.* Visible in `docker inspect` and in the environment of all processes.
- *An external secrets store (HashiCorp Vault and the like).* Excessive for a single server.

**Consequences**
- (+) Only ciphertext in the repository.
- (−) Vault password files are in plain text: access to the control node as the owner's user reveals both passwords.
- (−) Losing an environment password = reissuing all its secrets (`rekey` requires the old password).
- (−) A secret can leak through the application itself: on a network error Alertmanager logs the Telegram request URL together with the token.

**How it was verified:** `vault.yml` headers — `$ANSIBLE_VAULT;1.2;AES256;test` and `…;prod`; secret wiring checked with `is defined` without printing values; `docker inspect` of Alertmanager and Grafana — only the file path in the environment; the production server secret hashes matched vault before the migration.

---

<a id="adr-005"></a>
## ADR-005. Environments: separate inventories

**Status:** accepted, applied.

**Context.** Ansible asks for the sudo password once per run, and hosts run in parallel. A run without a limit, out of habit, would hit the production server together with the test environment. `ansible-core` has no setting like "forbid runs without `--limit`".

**Decision**
- Two inventories: `inventory/test` (default in `ansible.cfg`) and `inventory/prod` (explicit only, `-i inventory/prod`).
- Shared variables are in `ansible/group_vars/all.yml` next to the playbook: they apply to any inventory without duplication.
- Both inventories have the `maxapoc_servers` group that the playbook refers to.

**Alternatives**
- *One inventory and `--limit`.* Only discipline protects.
- *Checking `ansible_limit` at the start of the play.* Works, but a "default" run would still target both environments.

**Consequences**
- (+) The production server cannot be hit without specifying its inventory.
- (−) Shared variables live outside the inventory; this needs to be known when reading.

**How it was verified:** `ansible-inventory --graph` without `-i` shows only the test environment, with `-i inventory/prod` — only the production server; after the files were moved the test environment gives `changed=0`.

---

<a id="adr-006"></a>
## ADR-006. A separate monitoring stack for the project; disabled on the production server for now

**Status:** accepted. The stack is verified in the test environment; on the production server it is disabled with the switch `maxapoc_monitoring_enabled: false`.

**Context.** MaxApoc metrics and alerts on the production server are currently handled by the `hobot-metrics` stack (the author's personal project): MaxApoc files are placed into its directory. The project infrastructure should be described without ties to other services. At the same time the production server has 1 GB of memory: reconnaissance showed ~300 MB available, ~400 MB of swap in use, while this stack's Grafana takes ~300 MB in the test environment.

**Decision**
- **The `maxapoc_monitoring` role:**
  - Prometheus, Alertmanager, Grafana in the shared network with the proxy;
  - rules and dashboards from the project's `observability/`;
  - interfaces on `127.0.0.1`, access via SSH tunnel;
  - pinned image versions, memory limits;
  - an HTTP health check at the end of the role.
- Alertmanager is a separate container, not the one built into Grafana: Grafana's built-in Alertmanager does not accept alerts from an external Prometheus, and delivery must not depend on Grafana.
- On the production server the role is disabled until a second server or a plan with more memory appears. Proxy metrics for `hobot-metrics` are published on the WireGuard address.

**Alternatives**
- *The full stack on the production server now.* Rejected because of memory.
- *Only Prometheus and Alertmanager, dashboards in the `hobot-metrics` Grafana.* Keeps the dependency on another stack for viewing.

**Consequences**
- (+) The project stack is ready and verified; enabling it is one variable.
- (−) On the production server MaxApoc observability still depends on `hobot-metrics`; port 9101 stays published on the WireGuard address.

**How it was verified (test environment):** target `maxapoc-proxy` `up`; 20 rules loaded; a test alert and drills with the proxy stopped — 🔴 after ~3.5 minutes, 🟢 after recovery; the Grafana data source and two dashboards via provisioning; changing the Grafana password through vault (old `401`, new `200`); the HTTP health check stops the run if a service did not come up.

---

<a id="adr-007"></a>
## ADR-007. The boundary between Ansible and the deploy

**Status:** accepted; production server — 2026-11-02. The decision was refined during implementation: the compose file, rules and dashboards stayed with Ansible (see "Deviation from the original decision").

**Context.** The server state changes rarely, the application version changes often. If they are mixed, either every deploy runs the whole playbook with access to the entire server, or the server gets changed "quickly" around the code.

**Decision**

| What | Owner | How |
|---|---|---|
| OS, hardening, Docker, directories, shared network, deploy user | Ansible | roles `common`, `security`, `docker`, `maxapoc_deploy_access` |
| `.env` (secrets and environment), certificates, `gateways.json` | Ansible | from vault and from a clean clone of `main` ([ADR-019](#adr-019)) |
| The proxy `docker-compose.yml`, alert rules and dashboards | Ansible | from a clean clone of `main` ([ADR-019](#adr-019)) |
| The initial `release.env` (only on a server that does not have one) | Ansible | from the environment's latest backup; no backup — `maxapoc_proxy_image` from the inventory ([ADR-017](#adr-017)) |
| **Image version** (`release.env`) | **deploy from CI** | `deploy <digest>` through a restricted key ([ADR-016](#adr-016), [ADR-017](#adr-017)) |
| Prometheus data, Alertmanager silences, the Grafana database | the service itself | not in the repository → backups ([ADR-023](#adr-023)) |

- **The interface is a naming contract:**
  - `.env` variables;
  - the path to `release.env`;
  - certificate paths;
  - the network name.
- **A contract change takes two steps.** First compose and `.env` compatible with the **old** image (merge to `main` + an Ansible run), then the code that uses them. One commit will not do: a merge to `main` triggers the deploy immediately, while Ansible has not run yet at that moment. If it is merged in one commit anyway, the new code without the variable will not become healthy, and the deploy rolls back by itself ([ADR-018](#adr-018)).
- **The deploy CI does not know the application secrets:** it needs only the deploy key, the server address and `known_hosts`.
- **Clean server:** Ansible takes the version from the environment's latest backup, and without a backup — the initial one from the inventory; the first deploy then brings the current one. Verified: clean VM → one run (6 min 05 s) → deploy (1 min 09 s — before the health check was sped up, now ~7 s) → repeat run `changed=0`; the version from the backup — by deleting `release.env` in the test environment.

**Deviation from the original decision.** Originally the compose file, rules and dashboards were meant to ship "together with the release". That would have required either passing a configuration file through the CI key (the key would then be able to change the container's ports, volumes and privileges — rejected) or extracting compose from the image on every deploy (extra code). Compose changes rarely, so it stayed with Ansible, but its source is `main` only ([ADR-019](#adr-019)). The price is the two-step rule above; an "image rolled back without rolling back compose" mismatch is possible if compatibility is not maintained.

**Consequences**
- (+) A leak from CI does not reveal application secrets; the CI key does not change the run configuration.
- (+) Ansible does not take the version back from the deploy: a repeat run after a deploy gives `changed=0` (verified).
- (−) Two owners of one directory on the server; the boundary relies on `force: false` and the two-step rule.

---

<a id="adr-008"></a>
## ADR-008. `gateways.json` is application configuration

**Status:** accepted, applied.

**Context.** The file sets the order of LLM gateways, daily caps and route disabling parameters. The weekly `gateway-bench` workflow (on Mondays and manually) runs the routes from `proxy/vps/gateways.example.json` and puts the suggested order (`gateways.suggested.json`) and a results table into the run summary and an artifact. It writes nothing to the repository or the server (the workflow has read-only permissions). Previously the suggested file was supposed to be placed on the server by hand. The production server had a file identical to `gateways.example.json`. The file contains no keys, only variable names.

**Decision.** The source of truth is `proxy/vps/gateways.example.json` in the project repository.

1. The owner reviews the benchmark artifact and, if they agree, carries the changes over to `gateways.example.json` (checking `limits` so they are not lost) and commits.
2. The file is delivered to the server as `gateways.json` by the `maxapoc_proxy` role from a clean clone of `main` ([ADR-019](#adr-019)).
3. No manual edits are made on the server.

**Alternatives**
- *A host file in the inventory.* Rejected: a second copy that will diverge from the project.
- *Leave the existing file alone and edit it by hand.* Rejected: the correct version would exist only on the server, with no history and outside backups.

**Consequences**
- (+) Every change to the gateway order and caps has a history in git.
- (−) Caps cannot be changed "on the fly" on the server — only through the repository and a delivery run.
- (−) The file is mounted into the container as a single file (a file bind mount). Ansible writes the new file via rename, and the running container keeps seeing the old one (verified in the test environment: the hashes on the host and in the container diverged). Therefore a change to `gateways.json` restarts the proxy (a short interruption). Mounting the directory instead of the file has not been done and remains an improvement.
- (−) The name `…example.json` is misleading: it is the working configuration.

---

<a id="adr-009"></a>
## ADR-009. Production server firewall

**Status:** accepted, applied 2026-10-28.

**Context.** Before the migration: ufw `default allow incoming`, rules `22 ALLOW`, `51820/udp`, `8443`, decorative `DENY` for 3000/9090. Also listening externally: xray (inbound ports in a range), the x-ui panel and subscription, the hosting provider's Zabbix agent.

**Decision**
- `default deny` for incoming; the `routed` policy is not changed (WireGuard).
- SSH — a separate task with `LIMIT` (6 connections per 30 s per address).
- Other ports — as a list in the inventory, with the purpose in the rule comment:
  - 8443 — proxy;
  - 51820/udp — WireGuard;
  - `33000:38000` — xray inbounds (the range is in code, specific ports are in the panel);
  - 2096 — x-ui subscription;
  - 4467 — x-ui panel;
  - 10050 — only from the three addresses of the hosting provider's Zabbix.
- Old rules are removed by hand after the migration (the role only adds). ufw updated the existing `22 ALLOW` in place when `limit` was added.

**Accepted risk:** the x-ui panel is open to the internet (random path, strong password, TLS).

**Alternatives**
- *`ALLOW` instead of `LIMIT` on 22 "for CI".* Rejected after measuring: the deploy makes **one** SSH connection per run, `LIMIT` (6 per 30 s) does not get in its way.

**How it was verified:** reconnaissance with `ss -tulpn` before the migration — everything listening externally is in the list; after the migration `ufw status verbose` — `deny (incoming)`, the list matches the code; xray, WireGuard, Zabbix and the application work.

---

<a id="adr-010"></a>
## ADR-010. Migrating the running proxy to Ansible

**Status:** accepted, done 2026-10-28.

**Context.** The proxy ran from `/root/maxapoc-proxy/vps` (compose project `vps`), as root (`PROXY_UID=0`), the image `vps-maxapoc-proxy:latest` was built on the server. The role deploys it to `/opt/maxapoc/proxy/vps`, the process runs as uid 1000.

**Decision**
1. **Comparison before the migration:** secrets, repository names and certificate fingerprints — by hash against vault (they matched); code — by file hashes (only `docker-compose.yml` differed). The migration does not change the application version.
2. **Rehearsal in the test environment** with a copy "as on the production server" (the same directory, root, the old code).
3. **A dedicated image name `maxapoc-proxy:local`.** The rehearsal showed: the old and new directories share one compose project name, and therefore one image name. The role picked up the image built by the old directory, and a rollback would have brought back the config but not the code.
4. **Order:** the role recreates the container (compose "adopts" the old container by project name), downtime ~1 s for the recreation.
5. **Rollback** in one line: `docker compose -f /opt/... down && docker compose -f /root/maxapoc-proxy/vps/docker-compose.yml up -d` (1–2 s); the old image is tagged `pre-ansible`.
6. **Cleanup** after 1–2 weeks — without `docker compose down` from the old directory: because of the shared project name it would stop the current proxy.

**Consequences**
- (+) The proxy does not run as root.
- (−) The decision to rebuild the image based on changes to the delivered files did not notice that someone else had replaced the image under the same name. **Closed:** the image is built in CI and run by digest ([ADR-015](#adr-015)); building on the server was removed ([ADR-020](#adr-020)).

**How it was verified:** rehearsal (recreation without errors, downtime ~1 s, uid 1000, rollback); on the production server — `maxapoc-proxy:local`, `healthy`, uid 1000, no permission errors, total downtime 4 s, metrics in the `hobot-metrics` Grafana, application requests and the hub work.

---

<a id="adr-011"></a>
## ADR-011. Docker daemon `live-restore`

**Status:** accepted. Test environment — enabled and verified; production server — 2026-10-30, without restarting containers.

**Context.** During the production server migration the `docker` role created `daemon.json` and restarted the daemon. All containers came back except the Prometheus of the neighbouring `hobot-metrics` stack (policy `unless-stopped`, clean stop, no memory shortage): after starting, the daemon did not attempt to start it, and the log does not explain this. ~16 minutes without metrics, detected by empty dashboards.

**Decision.** `"live-restore": true` in `daemon.json` (role `docker`, variable `docker_live_restore`): containers keep running while the daemon restarts.

**Consequences**
- (+) A daemon restart no longer stops containers — the whole class of failures disappears regardless of the cause of the incident.
- Enabling it does not require a daemon restart: `live-restore` is applied by reloading the configuration (`systemctl reload docker`). On the production server `daemon.json` was written by hand exactly as the role template, checked with `dockerd --validate`, applied with reload; the role then saw a matching file and did not restart the daemon.

**How it was verified:** in the test environment `docker info` → `LiveRestoreEnabled: true`; `systemctl restart docker` — the start time and restart count of all containers before and 2 minutes after matched. Production server (2026-10-30): after reload `LiveRestoreEnabled: true`, the start times and counters of all containers did not change; the Ansible preview no longer shows a Docker restart.

---

<a id="adr-012"></a>
## ADR-012. External dependencies

**Status:** accepted, applied.

**Context.** Deployment depends on Ubuntu mirrors, the Docker repository, Docker Hub, GitHub and Telegram — they can become unavailable without any change on our side. From the test environment network `download.docker.com`, Docker Hub and Telegram are not reachable directly.

**Decision**
- Network tasks are retried several times, then the run stops with an error.
- The test environment goes through the control node's HTTP proxy, which is open only to the test environment network. Everything that needs the proxy gets it explicitly:
  - Ansible tasks — `environment:`;
  - the Docker daemon — `daemon.json`;
  - Alertmanager — the receiver's `proxy_url`.

  Processes inside containers do not know about the proxy by themselves.
- The production server has no proxy.

---

<a id="adr-013"></a>
## ADR-013. The test environment mirrors the production server but shares no secrets with it

**Status:** accepted.

**Decision**
- The same OS (Ubuntu 24.04, not newer), the same amount of memory (1 GB), the same pinned Docker versions.
- **The test environment's own secrets:**
  - CA and certificates;
  - vault password;
  - **Telegram bot and chat** — test alerts do not mix with real ones, and the production bot token is not stored outside the production environment.
- VM snapshots — only of a powered-off machine (otherwise the clock is wrong after a rollback).

---

<a id="adr-014"></a>
## ADR-014. The proxy CA private key is kept off the server

**Status:** accepted, done.

**Context.** The application trusts only certificates signed by the project CA. `ca.key` was stored on the production server.

**Decision.** `ca.key` was moved to the owner's password manager (the manager's database has a copy on a second computer) and deleted from the server; the hosting provider backups taken before the move were deleted. The server certificate is issued off the server and delivered through vault.

**Accepted risk:** the CA was not reissued — there are no signs of compromise, and reissuing requires rebuilding the application for every user.

---

<a id="adr-015"></a>
## ADR-015. The proxy image is built in CI, the server pulls it by digest

**Status:** accepted. Test environment — 2026-10-29, production server — 2026-11-02.

**Context.** The image was built on the server (1 GB RAM) from files on disk under a single name `maxapoc-proxy:local`. The name can be overwritten by a new build, so "rolling back to the previous image" did not guarantee the previous code ([ADR-010](#adr-010)).

**Decision**
- **The `image` job** in the `Proxy` workflow (after the tests):
  - builds the image and publishes it to a private GHCR: `ghcr.io/cl0wnf1esta/maxapoc-proxy:sha-<commit>`;
  - labels `org.opencontainers.image.source` and `revision`;
  - job permissions — only `contents: read` and `packages: write`; the token is the short-lived `GITHUB_TOKEN`.
- The job outputs the **digest** (`sha256:…`) — an immutable reference to the content. The server runs the image only by digest (`name@sha256:…`), not by tag: a tag is overwritten by a repeat build run, a digest is not.
- The server logs in to the registry with a **read-only** token (classic PAT `read:packages`, 90 days); the test environment and the production server have **different** tokens (in each environment's vault).

**Alternatives**
- *Deploy by the `sha-<commit>` tag.* Rejected: a repeat build run overwrites the tag with a new digest (observed: three runs on one commit — three digests).
- *A fine-grained token for one repository.* Not available: GitHub Packages accepts only classic tokens.
- *A repository deploy key.* Not suitable: it gives SSH access to git, the registry accepts only tokens.
- *A separate "machine" GitHub account with access to the package only.* Postponed (see consequences).

**Consequences**
- (+) The same verified image (the same digest) goes to any server; rolling back to a digest brings back exactly the previous content.
- (+) The server spends no memory or time on building: a clean VM comes up in 6 min 05 s instead of 15 min 25 s (the same set of roles, without backups; with them — 6 min 26 s).
- (−) The classic token can read all packages of the account (there is one package at the moment). Improvement — a machine account with access to this package only.
- (−) The token is stored on the server in `/root/.docker/config.json` (`0600`, base64 — not encryption). 90-day lifetime — a reminder in the password manager 7 days ahead; an expired token breaks only pulling a new image, not the running proxy.
- (−) Rebuilding the same commit gives a different digest (build time, the moving base image `node:22-alpine`).

**How it was verified:** the image's `revision` label matches the commit; the package is private and linked to the repository; the server runs `…@sha256:…` and is `healthy`; `/root/.docker/config.json` — `root`, `0600`.

---

<a id="adr-016"></a>
## ADR-016. CI access to the server

**Status:** accepted. Test environment and production server — 2026-10-30.

**Context.** CI must roll out new versions, but a leak of what it uses for that must not give access to the server.

**Decision**
- **A separate user `maxapoc-deploy`** (role `maxapoc_deploy_access`):
  - password locked;
  - not in the `docker` group;
  - no general sudo.
- **A restricted key** in `authorized_keys`:
  - `restrict` — no terminal, forwarding or agent;
  - `command="/usr/local/bin/maxapoc-deploy-gate"` — whatever the client asks for, this script runs;
  - only this key in the file (`exclusive`);
  - separate keys for the test environment and the production server.
- **Filter script** (`root`, `0755`):
  - allows only `status` and `deploy sha256:<64 characters 0-9a-f>` (the pattern matches the whole string);
  - logs the client address and the decision (`logger -t maxapoc-deploy-gate`);
  - unvalidated input is logged escaped (`printf '%q'`), so a line break cannot forge an entry.
- **sudo for a single script:**
  - `/etc/sudoers.d/maxapoc-deploy` (`0440`, checked with `visudo -cf`) allows only `/usr/local/sbin/maxapoc-deploy-run` without a password;
  - the script is `root`, `0700`: the deploy user can neither modify nor read it;
  - the script re-validates the format itself; the image name is hard-coded in it.
- **GitHub environment `production`:**
  - branch rule — `main` only;
  - `DEPLOY_SSH_KEY`, `DEPLOY_HOST`, `DEPLOY_KNOWN_HOSTS` — environment secrets, not repository secrets;
  - a job from another branch is rejected before its steps run, no secrets are issued to it.
- **Host key via a trusted channel:**
  - the `known_hosts` line was read from the server over an already verified SSH connection, the fingerprint was compared;
  - CI uses `StrictHostKeyChecking=yes`, without `ssh-keyscan`.

**Alternatives**
- *Application secrets in CI (the vault password or a secrets store).* Rejected: workflow code that receives a secret can send it out; the deploy does not need the application secrets at all.
- *CI sends the compose file to the server.* Rejected: the CI key would be able to set `privileged`, volumes and ports, i.e. get root.
- *A self-hosted runner in the test environment* (so that CI can reach it behind NAT). Rejected: the control node holds the vault passwords; the deploy logic is exercised in the test environment by hand with the same commands.
- *`ALLOW` instead of `LIMIT` on port 22.* Not needed: one SSH connection per deploy ([ADR-009](#adr-009)).

**Consequences**
- (+) A stolen CI key only allows checking the status and rolling out an image from our registry; there is no shell and no reading of `.env` or certificates.
- (−) The CI key protects the "repository → server" link, but not what lands in `main`: whoever can push to `main` controls production. `main` is protected by account access (2FA); branch protection for a private repository is available only on a paid plan.
- (−) When the server host key changes, the deploy fails (as it should); the new fingerprint is verified through the hosting provider's console, not over SSH.

**How it was verified:** `status` — image and `healthy`; login without a command — no terminal and `denied`; `status; id`, a short digest and a digest with a line break — `denied`, code 1; port forwarding `-R` — refused; the test environment key on the production server with `IdentitiesOnly=yes` — rejected; `sudo -l` — a single command; the `deploy-check` job from a branch — rejected by the environment, from a temporarily allowed branch — `healthy`, secrets in the log `***`, after the rule was removed — rejected again.

---

<a id="adr-017"></a>
## ADR-017. The image version lives in `release.env`, owned by the deploy

**Status:** accepted. Test environment — 2026-10-30, production server — 2026-11-02.

**Context.** If the version is in `.env`, which Ansible writes, then any playbook run after a deploy silently brings back the version from the inventory — while looking like a success.

**Decision**
- The `MAXAPOC_PROXY_IMAGE` variable lives only in `release.env`. It is written by the deploy script; Ansible creates the file **only if it does not exist** (`force: false`) and never touches it again.
- **A server without `release.env`** (clean, or the file was lost):
  - first — the version from this environment's latest backup (`restic restore … --include …/release.env`; the `backup` role runs before the proxy role);
  - if that fails — the initial version from the inventory, with a warning in the output;
  - the task runs only when the file is missing (`creates:`).
- Compose reads both files: `--env-file .env --env-file release.env` (in Ansible — `env_files`, including in the restart handler). `release.env` is not added to the container environment.
- In compose: `image: "${MAXAPOC_PROXY_IMAGE:?…}"` — without the version file compose stops with a hint instead of substituting a default.

**Consequences**
- (+) A repeat Ansible run after a deploy gives `changed=0`, the version is not rolled back (verified by writing a "deploy" into `release.env` by hand).
- (+) A manual `docker compose` without `--env-file` fails loudly instead of starting a different version.
- (−) The `maxapoc_proxy_image` value in the inventory is a fallback initial version (when there is no backup), not "what is running now" (see `status`). It must be fresh and compatible with the current compose.
- (−) Coupling between roles: the proxy role uses the restic wrapper from the `backup` role; without backups — the fallback path.
- The backup brings the version that was running at the time of the latest backup (≤ a day), not the one before the file was lost.
- Ansible keeps no state between runs; a lost `release.env` comes back from the backup ([ADR-024](#adr-024)), without a backup — the initial version.

**How it was verified:** a deleted `release.env` was restored from the backup (the file holds the version from the snapshot, not from the inventory), proxy `healthy`; with backups disabled — the initial version from the inventory; the version variable is neither in `.env` nor in the container environment; `docker compose config` without `--env-file` — an error with a hint; a rebuild from scratch found a handler without `env_files` — fixed and verified with another rebuild.

---

<a id="adr-018"></a>
## ADR-018. Deploy script: pull before switching, wait for healthy, roll back

**Status:** accepted. Verified in the test environment, including with a deliberately broken image; on the production server — by every deploy from CI since 2026-11-02.

**Decision** (`maxapoc-deploy-run deploy <digest>`, under a `flock` lock — a second deploy is refused, not queued):
1. `docker pull` of the new image. Not pulled — exit, nothing changed on the server (code 2).
2. The current version is copied to `release.env.prev`.
3. The new version is written atomically (temporary file + `mv`).
4. `docker compose up --no-build --wait --wait-timeout 60`. Healthy — `OK`, then registry images other than the current and the previous one are removed (code 0).
5. Not healthy within 60 s — `release.env.prev` is put back, `up --wait` on the previous version (code 4); if that fails too — code 5.

In CI the codes become annotations (`::error::…`); the `deploy` job runs only on `push` to `main`, with `concurrency` without cancellation (an SSH drop in the middle of a deploy is more dangerous than waiting) and a check that the commit is in the history of `main`.

**Proxy health check:** `start_period: 30s`, `start_interval: 2s` (with `interval: 60s`, `retries: 3`) — frequent checks at the start, the first successful one makes it healthy immediately. The wait timeout is aligned with `start_period`, not with the time until `unhealthy` is declared (~210 s): a version that needs more than a minute is considered broken.

**Alternatives**
- *A separate "rollback" command and a `stable` tag.* Rejected: a tag is a moving pointer; a second code path would run only during an incident. Rollback is a deploy of the previous digest along the same path.
- *Blue-green (two instances and a traffic switch).* Rejected on cost: memory (1 GB) and complexity with a single user. The price is ~1 s of downtime for the recreation and up to 60 s of the broken version running before the rollback.
- *`docker image prune -a`.* Forbidden on the server: it would delete the previous image that rollback without the registry relies on.

**Consequences**
- (+) An expired token or an unavailable registry breaks the deploy **before** the old version is stopped.
- (+) Rolling back one step does not depend on the registry or the network.
- (−) `release.env.prev` holds one step; after a rollback it equals the current one. Further back — deploy an old digest from the `maxapoc-deploy-run` log or the CI summary.
- (−) A timeout that is too short under memory pressure would cause a false rollback of a working version.

**How it was verified:** a non-existent digest — code 2, the version unchanged; deploy — `OK`, `.prev` = the previous version; confirmation time in the test environment 1 min 08 s → 6.4 s, on the production server 7 s (2026-11-03); an image whose process runs but whose health check always fails (published by a manual run from a separate branch, could not reach production) — rollback after 60 s, healthy on the previous version in 4 s, code 4, 1 min 10 s in total; the broken image was removed by the next successful deploy; on the production server 2 of 4 images were left after a deploy.

---

<a id="adr-019"></a>
## ADR-019. The source of files for the servers is a clean clone of `main`

**Status:** accepted, applied 2026-11-02.

**Context.** The roles took compose, `gateways.json` and observability from the MaxApoc working copy on disk — from whatever branch was checked out. An experimental change could reach the production server bypassing CI.

**Decision.** The first task of the playbook (`pre_tasks`, on the control node, once, without sudo) updates a separate clone `~/.cache/maxapoc/app` to the `main` branch (`ansible.builtin.git`, `force: false`). The `maxapoc_proxy_src` and `maxapoc_monitoring_stack_src` paths point into this clone. Manual changes in the clone are a loud error before any role runs.

**Consequences**
- (+) Only what has passed through `main` reaches the servers.
- (−) The clone is shared by the environments: the production server gets `main` as of its own run, not as of the check in the test environment. To get exactly what was verified — `-e maxapoc_repo_version=<commit hash>`.
- (−) Urgent compose rollback: `-e maxapoc_repo_version=<hash>` (temporary, server ≠ `main`), then `git revert` through the regular flow.

**How it was verified:** an experimental compose change in the working copy on a separate branch — `changed=0`; a change in the clone — the error `Local modifications exist`, the play stopped before the roles.

---

<a id="adr-020"></a>
## ADR-020. The fallback build-on-server mode was removed

**Status:** accepted, done 2026-11-02 (after two deploys from CI to the production server).

**Decision.** Removed the `maxapoc_proxy_build_on_server` switch, the build task file, the list of code files and the conditions in the role; compose runs with `build: never`, `pull: missing`. The code files and the `maxapoc-proxy:local` image were deleted on both servers.

**Why.** A path that never runs breaks unnoticed and fails exactly when it is needed. One verified delivery path is more reliable than two.

**Consequences**
- (−) With the registry unavailable a new version cannot be rolled out. The emergency procedure (break-glass) is in [OPERATIONS](OPERATIONS.md#break-glass).

**How it was verified:** a deploy to the test environment without the code files — `OK`; repeat runs on both servers — `changed=0`.

---

<a id="adr-021"></a>
## ADR-021. Application and proxy in one repository; `dev` → `main` flow

**Status:** accepted.

**Context.** `main` is the application release branch, and a merge into it also rolls out the proxy. Moving the proxy into a separate repository was considered.

**Decision.** Keep one repository. A change to the protocol between the application and the proxy is made in one commit. All changes go `dev` (checks) → `main` (release; for the proxy — auto-deploy). Changes only in `proxy/**`, `observability/**`, `proxy.yml` do not trigger the APK build (`paths-ignore`), changes in `observability/**` do not rebuild the proxy image.

**Consequences**
- (+) No second repository, second environment or version coordination between them.
- (−) `dev` holds only what can be rolled out: the proxy goes to production with the next merge.
- (−) The proxy is rolled out immediately, while the user updates the application whenever they like → the proxy must be backward compatible with the released application versions (otherwise rolling back the proxy is unsafe as well).
- Exception: if `dev` holds something that must not be released, an urgent proxy fix goes branch from `main` → `main`, then `main` → `dev`.

**How it was verified:** the first deploy from CI (2026-11-02): `main` fast-forwarded to `dev`, Build APK disabled for the merge (before `observability/**` was added to the exclusions) and re-enabled; no application release was created.

---

<a id="adr-022"></a>
## ADR-022. Post-deploy verification is a process, not code

**Status:** accepted (accepted risk).

**Context.** The deploy checks only healthy at startup. It does not catch a failure a few minutes after healthy. CI has no access to the production server's Prometheus.

**Decision.** After a deploy — watch the alerts and the proxy dashboard for 15 minutes ([OPERATIONS](OPERATIONS.md#deploy)). There is no repeat check from CI and no queries to Prometheus. A manual workflow run only reads (`deploy-check`); only a `push` to `main` rolls out.

**Rationale.** One developer and one user; the release time is chosen by the same person; a failed release carries no reputational or financial risk. A new path for CI into the server and repeat-check code cost more than the risk.

**Consequences.** A late failure is detected by the `MaxapocServerErrors` / `MaxapocProxyDown` alerts in the Prometheus of the `hobot-metrics` stack (rules loaded, verified 2026-11-03) in about 11–20 minutes; at night the reaction takes hours. The proxy rules exist in two places (`observability/` in MaxApoc and a copy in `hobot-metrics`) — synced by hand.

**Revisit if:** releases are made by someone other than the owner; the product gets real users; availability commitments appear.

---

<a id="adr-023"></a>
## ADR-023. Backups: restic to the hosting provider's S3 and a monthly secondary backup copy

**Status:** accepted. Test environment and production server — 2026-11-04.

**Decision**
- **The `backup` role:**
  - restic, daily on a systemd timer (`03:30` + up to 30 min, `Persistent=true`) to the hosting provider's private S3 bucket (standard class);
  - each environment has its own repository (prefix `test/`, `prod/`) and its own password;
  - the password and S3 keys are in the environment's vault, the password is also in the password manager.
- Retention: 7 daily, 4 weekly, 6 monthly snapshots (`forget --prune`); `restic check` on Sundays.
- Backup failure → `OnFailure=` → a message in Telegram (the environment's bot and chat).
- Once a month, by hand from the control node: `restic copy` from S3 to a local repository (`--copy-chunker-params`), pruning and checking there as well ([OPERATIONS](OPERATIONS.md#backup)).

**Alternatives**
- *A cold storage class.* Rejected: restic reads the index on every backup, `forget`/`prune` delete data; cold classes have a minimum storage duration, read charges and sometimes "thawing". At ~20 MB there are no savings.
- *Hosting provider disk snapshots.* Not a replacement: a crash-consistent copy (a SQLite database in WAL mode may be inconsistent) and the same account.
- *Pulling the secondary copy automatically (pull).* Postponed: another user and a filter on the server; the home computer is not always on.

**Consequences**
- (+) Loss of the VPS — restore from S3 losing up to a day; loss of the hosting account — from the secondary backup copy losing up to a month.
- (+) Size: 16.8 MiB of production server data takes 3.5 MiB in the storage (compression); a backup takes ~10 s.
- (−) One S3 key per bucket (the hosting provider does not issue others): the test environment can delete production server backups but cannot decrypt them (different passwords).
- (−) Root on the production server = the backup password and keys: read, delete, replace. The protected copy lags by up to a month (the secondary backup copy).
- (−) The monthly secondary backup copy depends on a person. Improvement — a user timer on the control node.
- (−) A disabled timer is not caught by an alert (only a failed backup is).

**How it was verified:** a backup through the service; without the password file the service failed and the alert arrived by itself; the restored x-ui and Grafana databases — `integrity_check ok`, 3 inbounds in x-ui; the secondary backup copy and a restore from it — `ok`.

---

<a id="adr-024"></a>
## ADR-024. State is backed up, not code

**Decision.** Only what cannot be recreated from git, vault and the image registry is backed up:

| What | Why |
|---|---|
| the x-ui database (VPN users and keys) | exists only on the server |
| `/etc/wireguard` | other peers know this public key |
| `/root/cert`, `/root/.acme.sh` | the x-ui panel certificate and its renewal settings |
| `/root/hobot-metrics` in full | secrets that git ignores and uncommitted changes |
| the proxy's `release.env` | the current version — right at restore |
| the `hobot-metrics` Alertmanager and Prometheus volumes, the Grafana database | silences, metrics history, users and settings |

Not backed up: the MaxApoc proxy (compose, `.env`, certificates, `gateways.json` — git and vault), Grafana plugins (512 MB, reinstalled), fail2ban and the Zabbix agent (installed by installers).

- SQLite databases — via `sqlite3 -readonly <database> ".backup …"`: a consistent snapshot without stopping the service. `-readonly` makes a missing database an error: without the flag `sqlite3` creates an empty database, and the backup would silently turn out empty (verified on the production server).
- The Prometheus history is copied live: blocks are copied whole, the last ~2 hours (the write-ahead log) may be incomplete and are truncated at startup. Acceptable for metrics history.

**Consequences**
- (−) The list of paths is an inventory variable: new state on the server (a new service) has to be added by hand. Drills and checking the backup contents catch an omission; database integrity does not.

---

<a id="adr-025"></a>
## ADR-025. Bucket versioning, no object lock

**Decision.** Versioning is enabled in the bucket; lifecycle rule: noncurrent versions are deleted after 30 days, incomplete uploads after 7. Expiry of current objects is not enabled (it would destroy the repository).

**Alternatives**
- *Object lock (WORM).* Rejected: restic deletes its lock files on every run, `forget`/`prune` delete data — under a lock the backups would fail every night.

**Consequences**
- (+) For 30 days an object deleted by mistake or by `prune` can be brought back.
- (−) The key owner can delete versions too: only the secondary backup copy protects against deliberate deletion.

**How it was verified:** backups with versioning enabled run without errors.

---

<a id="adr-026"></a>
## ADR-026. fail2ban is not part of Ansible

**Context.** fail2ban runs on the production server, but not from this code: it was installed by the 3x-ui panel installer (limiting the number of addresses per VPN user) together with the package's standard sshd protection.

**Decision.** Do not add fail2ban to the `security` role. Login by key only, password and root disabled, `LIMIT` on port 22 — password brute-forcing is impossible, and the connection rate is already limited. fail2ban remains a dependency of x-ui and is installed together with it during a restore.

---

<a id="adr-027"></a>
## ADR-027. Recovery: RTO/RPO targets and drills

**Targets**
- **RPO — up to a day** (a backup once a day plus up to 30 minutes of delay). The frequency barely affects the size (deduplication) but increases the load and the number of snapshots; for rarely changing data a day is enough.
- **RTO — under an hour when reinstalling the same VPS (the address is kept):**
  - ~7 minutes measured for the automated part: clean VM → Ansible 6 min 26 s, backup restore 2.7 s;
  - the manual steps (x-ui, WireGuard, `hobot-metrics`) follow the procedure and have not been measured.
- **A new address — days, backups do not help.** The address is baked into:
  - the proxy certificate — an application rebuild and an update for users;
  - the x-ui panel certificate;
  - the settings of VPN clients and WireGuard peers.

  Therefore in an incident — reinstall the same VPS, not a new server.

**Drills**
- Monthly (together with the secondary backup copy) — a light check: the x-ui database from the secondary copy, integrity and the number of inbounds.
- Once a quarter and on upgrades or rollbacks of x-ui, Grafana, Prometheus versions or changes to what gets backed up — full drills: clean VM → Ansible → restore of the production server backup into a separate directory → comparison.
- WireGuard, x-ui and the `hobot-metrics` Alertmanager from the production server backup are **not started** in the test environment: the same tunnel key, the enabled x-ui bot and the alert settings would interfere with the production server.

**How it was verified (2026-11-04):** the production server backup was restored to a clean VM; 7 files matched the live server by hash, `count|max(id)` matched in the x-ui database; the Prometheus history from the backup — the same 18 blocks, readable by the same version; Grafana of the same version started with the restored database (`database: ok`, no migrations needed).

**Not verified:** reinstalling at the hosting provider and the first login, putting files in place, starting x-ui, WireGuard and `hobot-metrics` from the backup.
