# Security

[Русский](/docs/THREAT_MODEL.md) | **English**

What is protected, how, and what is deliberately left open. The document contains only what has been verified; plans are marked.

## What is protected

One public VPS (1 GB RAM) running the MaxApoc application proxy: the proxy holds GitHub tokens, LLM gateway keys and the private key of the server's TLS certificate. There is a single owner. The main threats: brute-forcing access to the server from the internet, secret leaks (through the repository, output, logs, environment variables), accidentally open ports, uncontrolled server changes.

## Server access

- **SSH by key only:**
  - password, root and keyboard-interactive login are disabled in code (`sshd_config.d/00-hardening.conf`);
  - the config is validated (`sshd -t`) before it is applied;
  - production server — since 2026-10-28, verified with `sshd -T` and attempts to log in with a password and as root.
- SSH with connection rate limiting (`ufw limit`: 6 per 30 seconds per address).
- **A separate user for administration:**
  - root privileges via sudo **with a password**;
  - not in the `docker` group (Docker access equals root).
- Fallback access — the hosting provider's console (checked before any work on access).

## Deploy from CI

Goal: a leak of what CI uses does not give access to the server ([ADR-016](/docs/en/DECISIONS.md#adr-016)).

- **A separate user `maxapoc-deploy`:**
  - password locked;
  - not in the `docker` group;
  - no general sudo.
- **A restricted key** in `authorized_keys`:
  - the CI key only; the test environment and the production server have different keys;
  - `restrict` — no terminal, port forwarding or agent;
  - forced command — whatever the client asks for, the filter script runs.
- **Filter script:**
  - allows only `status` and `deploy sha256:<64 characters 0-9a-f>`;
  - the pattern matches the whole string: a space, `;`, `-` or a line break do not pass;
  - every decision is logged with the client address; unvalidated input is logged escaped (`printf '%q'`), so a line break cannot forge an entry.
- **sudo for a single script:**
  - `/etc/sudoers.d/maxapoc-deploy` (`0440`, checked with `visudo -cf`) allows only `maxapoc-deploy-run`;
  - the script is `root`, `0700`: the deploy user can neither modify nor read it;
  - the script re-validates the format; the image name is hard-coded — the client sends only the digest, so another registry cannot be substituted.
- **GitHub environment `production`:**
  - deploy secrets (key, address, `known_hosts`) live only in the environment;
  - branch rule — `main` only: a run from another branch is rejected before the first step, no secrets are issued;
  - the deploy job has `contents: read`, `concurrency` without cancellation;
  - secrets are passed to steps via `env:`, not interpolated into command text.
- **Host key via a trusted channel:**
  - the `known_hosts` line was read over an already verified SSH connection, the fingerprint was compared;
  - CI uses `StrictHostKeyChecking=yes`, without `ssh-keyscan`;
  - after a server reinstall, the new fingerprint is verified through the hosting provider's console.
- **CI does not know the application secrets:** `.env`, certificates, tokens and gateway keys are delivered only by Ansible.

**Verified:**
- login without a command — no terminal and rejected;
- `status; id`, an invalid and a multi-line digest — rejected;
- port forwarding — refused;
- the test environment key on the production server (`IdentitiesOnly=yes`) — rejected;
- `sudo -l` — a single command;
- a run from a disallowed branch — rejected by the environment, secrets masked in the log.

## Network

- **ufw firewall, default-deny for incoming, defined in code** (role `security`; production server — since 2026-10-28):
  - SSH is allowed by a separate rule;
  - other ports — as a list in the inventory, each with its purpose (a comment in the ufw rule);
  - the hosting provider's Zabbix agent — from the provider's addresses only.
- **Ports published by Docker bypass ufw rules** (verified by experiment: a port without a bind address is reachable from outside with ufw closed). That is why services that must not be public are published on a specific address only:
  - the Prometheus, Alertmanager and Grafana interfaces — on `127.0.0.1`, access via SSH tunnel;
  - proxy metrics — inside the Docker network without publishing; on the production server also on the WireGuard address (for the `hobot-metrics` stack).
- Only the proxy port 8443 (TLS) is published to the outside.

## Secrets

- **Ansible Vault:**
  - only ciphertext in the repository;
  - the test environment and the production server have **different** vault **passwords**;
  - the passwords are in files outside git (`0600`) and in a password manager.
- **`no_log`** on tasks with secrets: values do not reach Ansible output, including with `--diff`. Run logs are not kept in git.
- **Secrets are passed to containers as files where the application supports it:**
  - the Telegram token (Alertmanager) and the Grafana password — `0400` files owned by the container process user;
  - they are absent from `docker inspect` and from process environments (verified);
  - the proxy reads its secrets from `.env` (`root`, `0600`).
- Secrets are compared with the server by hash, without printing values.
- **Infrastructure addresses and names are in vault:** server addresses, SSH user and key, the hosting provider's Zabbix addresses, the backup storage, private repository addresses. The public inventory does not reveal the infrastructure.
- **Image registry token on the server:**
  - read-only (classic PAT `read:packages`), 90-day lifetime, reminder 7 days ahead;
  - separate for each environment, in vault;
  - on the server — in `/root/.docker/config.json` (`0600`).
- **The test environment holds no production server secrets:** its own CA, its own alert bot and chat, its own backup repository password.
- **The root of trust is the password manager** (vault passwords, `ca.key`, backup passwords, tokens): its database has a copy on a second computer.

## Proxy

- **TLS with a private CA:**
  - the application trusts only this CA;
  - **the CA private key is not stored on the server**;
  - the server certificate is issued off the server and delivered through vault.
- **Proxy container** (verified on the production server, `docker inspect`: `ReadonlyRootfs`, `CapDrop`, `SecurityOpt`, `Memory`, `PidsLimit`):
  - the process runs **not as root** (uid 1000; production server — since 2026-10-28, ran as root before that);
  - read-only file system;
  - all capabilities dropped, `no-new-privileges`;
  - memory (96 MB) and process count (64) limits.
- The gateway configuration and daily spending caps (`gateways.json`) are mounted into the container read-only; the source is the project repository.
- The container runs by image **digest** (`…@sha256:…`), not by tag: the content behind the same reference cannot be swapped.

## Backups

- **Encryption on the server side** (restic): the S3 storage sees only ciphertext.
  - each environment has its own repository and its own password;
  - the password is in vault and in the password manager, the S3 keys are in vault;
  - on the server — `/etc/restic` (`0700`), `env` (`0600`), `password` (`0400`), owned by root.
- **Secondary backup copy** (once a month) — on the control node: keys and password in `~/.config/restic` (`0600`), the repository is where the vault passwords already are.
- The bucket is private, with versioning (noncurrent versions kept 30 days).
- **Restore drills do not touch the production server:**
  - the backup is restored into a separate directory;
  - the production server password is in a temporary file (`0400`), removed with `shred -u` after the drill; restored data is deleted;
  - WireGuard, x-ui and the `hobot-metrics` Alertmanager from the backup are not started in the test environment: the tunnel key, the x-ui bot and the alert settings belong to the production server.
- WireGuard keys (`vps_private.key`, `wg0.conf`) — `0600` (fixed; backups preserve permissions, so the fix survives a restore).

## Supply chain

- Docker packages come from the official repository with signature verification (`signed-by`); versions are pinned and held from upgrades.
- Monitoring stack images use pinned versions (`prom/prometheus:v3.13.2`, `prom/alertmanager:v0.28.1`, `grafana/grafana:13.2.0`).
- **Proxy image:**
  - built only in CI, after tests, from a commit (`revision` label);
  - published to a private GHCR with the job permission `packages: write`;
  - never built on the server.
- Only files from the `main` branch reach the servers (a clean clone; local changes in it stop the run), not from the working copy.
- Official GitHub actions (`actions/*`, `docker/*`) are pinned to a major version (see limitations).

## Known limitations and accepted risks

| What | Why | Mitigation |
|---|---|---|
| The x-ui panel (VPN) is open to the internet | needed to manage the VPN without a tunnel | random path, strong password, TLS (accepted risk) |
| `APP_KEY` is embedded in the APK and can be extracted | inherent to a client-side key | filters out random requests; daily caps and limits on the proxy protect against targeted use |
| The Telegram token ends up in the Alertmanager log on a network error (part of the Bot API URL) | Alertmanager behaviour | the log is readable by root only; the test environment has a separate bot |
| Vault password files on the control node are in plain text | this is how `vault_identity_list` works | `0600` permissions, directory outside git; separate passwords protect against a leak of one file, not against access to the machine |
| The CA was not reissued after `ca.key` had been stored on the server | no signs of compromise; reissuing means rebuilding the application for every user | the key was moved off the server, hosting provider backups taken before that were deleted |
| The proxy reads secrets from `.env`, not from files | requires an application change | `.env` — `root`, `0600`; no extra users on the server |
| The ufw role only adds rules | how the role works | removal is a manual step in the change plan |
| MaxApoc observability on the production server depends on the `hobot-metrics` stack | not enough memory for a second stack | see [ADR-006](/docs/en/DECISIONS.md#adr-006); the copy of alert rules is synced manually |
| Whoever can push to `main` controls production | this is how auto-deploy works; branch protection for a private repository requires a paid plan | account 2FA, a single collaborator with write access; the CI key limits the damage ([ADR-016](/docs/en/DECISIONS.md#adr-016)) |
| The classic registry token can read all packages of the account | GitHub Packages does not accept fine-grained tokens | read-only, 90-day lifetime, separate per environment; there is one package at the moment. Improvement — a machine account with access to this package only |
| The registry token on the server is base64-encoded (`/root/.docker/config.json`) | this is how `docker login` stores it | `0600`, read-only |
| GitHub actions are pinned to a major version, not to a commit hash | a trade-off for simplicity; only official `actions/*` and `docker/*` | minimal job permissions; the build job sees only `GITHUB_TOKEN` |
| The base image `node:22-alpine` is a moving tag | base image security updates arrive without changes | every image passes tests; it reaches the server by digest |
| Root on the production server = the backup password and keys (read, delete, replace) | the backup is taken on the server | bucket versioning (30 days) — against mistakes; against an attacker — the secondary backup copy, up to a month behind |
| One S3 key per bucket | the hosting provider does not issue separate keys | the test environment can delete production server backups but cannot decrypt them (different passwords); the test virtual machine (VM) is not reachable from the internet |
| The monthly secondary backup copy is a manual step | the simplest option was chosen | the procedure and the backup age check are in OPERATIONS; improvement — a timer on the control node |
| The Telegram token is in the request URL of the backup failure alert (`curl`) | this is how the Bot API works | root process, short-lived; the script does not log the URL |
| A late failure after deploy is caught by alerts, not by CI | single user, the cost of the code outweighs the risk | [ADR-022](/docs/en/DECISIONS.md#adr-022): ~11–20 min until the alert |

## Repository publication

- The public code and the repository history contain no server addresses, private repository names or secrets: all of these are in vault (section "Secrets"). Verified by searching the full history before publication.
- **gitleaks** (pinned version, checksum verified) scans the full history on every push and pull request.
- Not done: pre-commit secret scanning on the control node.
