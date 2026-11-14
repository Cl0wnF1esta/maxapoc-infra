# maxapoc-infra

[Русский](README.md) | **English**

Ansible configuration for the MaxApoc server. It sets up Ubuntu 24.04, SSH and firewall rules, Docker, the application proxy, CI deployment access, monitoring, and encrypted backups.

The goal is to configure test and production environments consistently and make server recovery reproducible. Ansible manages server configuration. The application repository’s CI releases new proxy versions.

- [Architecture](ARCHITECTURE.en.md) — components, how they connect, and who manages them.
- [Server setup](ARCHITECTURE.en.md#setup) — prerequisites, inventory, and the first playbook run.
- [Operations](docs/en/OPERATIONS.md) — deployment, rollback, backups, and recovery.
- [Architecture decisions](docs/en/DECISIONS.md) and [threat model](docs/en/THREAT_MODEL.md).
