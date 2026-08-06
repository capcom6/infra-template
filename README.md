<div align="center">

# infra-template

Docker Swarm infrastructure template: reverse proxy, databases, backups, monitoring, and container management.

</div>

## Table of Contents
- [infra-template](#infra-template)
  - [Table of Contents](#table-of-contents)
  - [About The Project](#about-the-project)
  - [Features](#features)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
  - [Installation](#installation)
    - [1. Clone the repository](#1-clone-the-repository)
    - [2. Configure the environment](#2-configure-the-environment)
    - [3. Initialize Swarm mode](#3-initialize-swarm-mode)
    - [4. Create the external secrets](#4-create-the-external-secrets)
    - [5. Deploy the stacks](#5-deploy-the-stacks)
  - [Usage](#usage)
    - [Service endpoints](#service-endpoints)
    - [Typical workflows](#typical-workflows)
  - [Configuration](#configuration)
    - [Environment variables](#environment-variables)
    - [Docker secrets](#docker-secrets)
    - [Node labels](#node-labels)
    - [Configuration files](#configuration-files)
  - [Contributing](#contributing)
  - [License](#license)
  - [Contact](#contact)
  - [Acknowledgments](#acknowledgments)


## About The Project

Standing up self-hosted infrastructure normally means assembling and wiring many services by hand: a reverse proxy with TLS, a database, backups, monitoring, and admin UIs. This repository packages a working baseline into three versioned Docker Compose stacks that deploy together on a Docker Swarm cluster:

- [compose.infra.yml](compose.infra.yml) - core services: Traefik, Valkey (Redis), MariaDB, S3 backups, admin UIs.
- [compose.monitoring.yml](compose.monitoring.yml) - Prometheus, Alertmanager, cAdvisor, Grafana.
- [compose.portainer.yml](compose.portainer.yml) - Portainer CE and its agent.

Every service behind Traefik is exposed over HTTPS with automatic certificates, and protected by service-specific access controls: basic auth, IP allowlists, or application login (for example, Grafana's own login). The target audience is operators running Docker Swarm, single node or multi-node, who want a proven starting point for hosting applications.

## Features

- **Traefik v3 reverse proxy** with HTTP to HTTPS redirect, automatic Let's Encrypt certificates (TLS-ALPN-01 and Cloudflare DNS challenge resolvers), JSON access logs, and a dashboard behind basic auth.
- **Valkey 9** (Redis-compatible) cache with LRU eviction and AOF persistence, plus a **Redis Commander** web UI behind basic auth.
- **MariaDB LTS** database with a healthcheck, and **phpMyAdmin** behind basic auth.
- **Scheduled MariaDB backups to S3** via `mariadb-backup-s3`, driven by `swarm-cronjob` with a configurable cron schedule.
- **Prometheus** with Docker Swarm service discovery and cAdvisor metrics, plus alert rules for down instances, high CPU, memory, disk, and network usage.
- **Alertmanager** routing: critical alerts to Telegram, warning and info alerts to email, using credentials from Docker secrets.
- **Grafana 13** dashboards exposed at `mon.<ROOT_DOMAIN>`.
- **Portainer CE + agent** for cluster management over a dedicated overlay network, published on port 9443.
- **Swarm-native patterns** throughout: overlay networks, node-label placement constraints, versioned Docker configs, and external Docker secrets.

## Getting Started

### Prerequisites

- Docker Engine with Swarm mode enabled. Single node (`docker swarm init`) or a multi-node cluster; see the node labels required in [Configuration](#configuration).
- Inbound TCP ports 80 and 443 routed to the node running Traefik, and 9443 for Portainer.
- DNS records for `ROOT_DOMAIN` pointing at the Traefik node for the hostnames listed in [Usage](#usage).
- Docker 20.10+ or newer with Compose v2 support for `docker stack deploy` (no explicit version pin is required by the stacks).

## Installation

### 1. Clone the repository

```bash
git clone git@github.com:capcom6/infra-template.git
cd infra-template
```

### 2. Configure the environment

```bash
cp .env.example .env
```

Edit `.env` and set at least `ROOT_DOMAIN` (the base domain for all services) and the credentials for your environment. See [Configuration](#configuration) for the full variable reference.

### 3. Initialize Swarm mode

```bash
# Skip if the node is already part of a Swarm cluster
docker swarm init
```

### 4. Create the external secrets

The infra and monitoring stacks reference Docker secrets by fixed names, so create them before deploying. Replace the example values.

```bash
printf 'change-me-db-root-password' | docker secret create mariadb_root_password -
printf 'change-me-telegram-bot-token' | docker secret create telegram_bot_token -
printf 'change-me-email-password' | docker secret create email_password -
printf 'change-me-grafana-admin-password' | docker secret create grafana_admin_password -
```

The remaining secrets are files:

- `users.htpasswd` - basic-auth file used by Redis Commander (htpasswd format, the same format shown in `DB_ADMIN_AUTH` in `.env.example`).
- `metrics.htpasswd` - basic-auth file used for Prometheus/Alertmanager metrics endpoints.

```bash
docker secret create users.htpasswd ./users.htpasswd
docker secret create metrics.htpasswd ./metrics.htpasswd
```

### 5. Deploy the stacks

Deploy the infra stack first: it creates the `internal` and `public` overlay networks that the monitoring stack references as external.

```bash
docker stack deploy -c compose.infra.yml infra
docker stack deploy -c compose.monitoring.yml monitoring
docker stack deploy -c compose.portainer.yml portainer
```

The stack names (`infra`, `monitoring`, `portainer`) are examples and can be renamed; the network names, secret names, and config names are fixed by the compose files.

## Usage

### Service endpoints

Replace `example.com` with your `ROOT_DOMAIN`. All Traefik-routed HTTPS endpoints are served on ports 80/443. Portainer is the exception: it is published directly on port 9443, not routed through Traefik.

| Service            | URL                            | Access control                              |
| ------------------ | ------------------------------ | ------------------------------------------- |
| Traefik dashboard  | https://admin.example.com      | basic auth (`DASHBOARD_AUTH`)               |
| phpMyAdmin         | https://pma.example.com        | basic auth (`DB_ADMIN_AUTH`)                |
| Redis Commander    | https://redis.example.com      | basic auth (`users.htpasswd` secret)        |
| Prometheus         | https://prometheus.example.com | IP allowlist (`PROMETHEUS__IP_ALLOWLIST`)   |
| Alertmanager       | https://alerts.example.com     | IP allowlist (`ALERTMANAGER__IP_ALLOWLIST`) |
| Grafana            | https://mon.example.com        | login (`grafana_admin_password` secret)     |
| Portainer (direct) | https://<node-ip>:9443         | Portainer admin account                     |

### Typical workflows

```bash
# List services of a stack
docker stack services infra

# Follow logs of a service
docker service logs -f infra_traefik
```

**Update a versioned config.** Traefik and Prometheus configs are registered as Docker configs named with a `STACK_VERSION` suffix (for example `prometheus.yml_0`). Editing `traefik/` or `prometheus/` files alone will not be picked up; bump `STACK_VERSION` in `.env` and re-run `docker stack deploy` to publish the new configs.

**Change the backup schedule.** Set `DB_BACKUP__SCHEDULE` in `.env` (cron format, default `@daily`) and redeploy the infra stack. The backup job runs as a scheduled `swarm-cronjob` task.

## Configuration

### Environment variables

All variables are defined in [.env.example](.env.example). Grouping follows the comment headers in that file.

**Common**

| Variable        | Description                                                              | Example         |
| --------------- | ------------------------------------------------------------------------ | --------------- |
| `TIMEZONE`      | Container timezone (compose default `UTC`)                               | `Europe/Berlin` |
| `ROOT_DOMAIN`   | Base domain for all Traefik-routed hostnames                             | `example.com`   |
| `STACK_VERSION` | Bumps versioned Docker config names; increment to publish config changes | `0`             |

**Infra stack**

| Variable                           | Description                                                    | Example                           |
| ---------------------------------- | -------------------------------------------------------------- | --------------------------------- |
| `DB_BACKUP__AWS_REGION`            | AWS region of the backup bucket                                | `us-east-1`                       |
| `DB_BACKUP__AWS_ACCESS_KEY_ID`     | AWS access key for backup uploads                              | `<your-access-key>`               |
| `DB_BACKUP__AWS_SECRET_ACCESS_KEY` | AWS secret key for backup uploads                              | `<your-secret-key>`               |
| `DB_BACKUP__PASSWORD`              | MariaDB password for the `backup` user                         | `<database-backup-user-password>` |
| `DB_BACKUP__OPTIONS`               | Extra mysqldump options                                        | *(empty)*                         |
| `DB_BACKUP__STORAGE_URL`           | S3 storage URL for backups                                     | `s3://bucket-name/backups`        |
| `DB_BACKUP__SCHEDULE`              | Cron schedule for backup jobs (compose default `@daily`)       | `'@daily'`                        |
| `DB_ADMIN_AUTH`                    | htpasswd credentials for phpMyAdmin basic auth                 | `admin:$$apr1$$hashed$$password`  |
| `DASHBOARD_AUTH`                   | htpasswd credentials for the Traefik dashboard                 | `admin:$$apr1$$hashed$$password`  |
| `TRAEFIK__METRICS_IP_WHITELIST`    | IP ranges allowed to reach metrics endpoints (comma-separated) | `127.0.0.1,192.168.0.0/16`        |

**App stack**

| Variable             | Description                                          | Example           |
| -------------------- | ---------------------------------------------------- | ----------------- |
| `HTTP__PROXY_HEADER` | Trusted proxy header for applications behind Traefik | `X-Forwarded-For` |
| `HTTP__PROXIES`      | Trusted proxy CIDR ranges for applications           | `10.0.0.0/16`     |

These variables are carried in `.env.example` for the application stacks you deploy on top of this infrastructure; none of the three compose files in this repository reference them.

**Monitoring stack**

| Variable                     | Description                                                              | Example                    |
| ---------------------------- | ------------------------------------------------------------------------ | -------------------------- |
| `PROMETHEUS__IP_ALLOWLIST`   | IP ranges allowed to reach Prometheus (compose default `127.0.0.1/32`)   | `127.0.0.1,192.168.0.0/16` |
| `ALERTMANAGER__IP_ALLOWLIST` | IP ranges allowed to reach Alertmanager (compose default `127.0.0.1/32`) | `127.0.0.1,192.168.0.0/16` |

### Docker secrets

| Secret                   | Used by         | Purpose                                                       |
| ------------------------ | --------------- | ------------------------------------------------------------- |
| `mariadb_root_password`  | MariaDB         | Root password (file-mounted via `MARIADB_ROOT_PASSWORD_FILE`) |
| `telegram_bot_token`     | Alertmanager    | Telegram bot token for critical alerts                        |
| `email_password`         | Alertmanager    | SMTP password for email alerts                                |
| `grafana_admin_password` | Grafana         | Admin password (via `GF_SECURITY_ADMIN_PASSWORD__FILE`)       |
| `users.htpasswd`         | Redis Commander | Basic-auth users file                                         |
| `metrics.htpasswd`       | Traefik         | Basic-auth users file for metrics endpoints                   |

### Node labels

Services use placement constraints on node labels. Tag nodes before deploying:

```bash
docker node update --label-add redis=true <node>
docker node update --label-add db=true <node>
docker node update --label-add traefik=true <node>
docker node update --label-add cronjob=true <node>
docker node update --label-add prometheus=true <node>
docker node update --label-add grafana=true <node>
docker node update --label-add portainer=true <node>
```

| Label        | Required on  | Services placed there    |
| ------------ | ------------ | ------------------------ |
| `traefik`    | manager node | Traefik                  |
| `cronjob`    | manager node | swarm-cronjob scheduler  |
| `portainer`  | manager node | Portainer                |
| `redis`      | any node     | Valkey                   |
| `db`         | any node     | MariaDB, backup job      |
| `prometheus` | any node     | Prometheus, Alertmanager |
| `grafana`    | any node     | Grafana                  |

cAdvisor (monitoring) and the Portainer agent (portainer) run in `global` mode on every Linux node, so they need no label.

### Configuration files

| File                                                       | Contents                                                                 |
| ---------------------------------------------------------- | ------------------------------------------------------------------------ |
| [traefik/traefik.yml](traefik/traefik.yml)                 | Static Traefik config: providers, entrypoints, ACME resolvers, dashboard |
| [traefik/dynamic.yml](traefik/dynamic.yml)                 | Dynamic Traefik config: metrics auth and IP-whitelist middlewares        |
| [prometheus/prometheus.yml](prometheus/prometheus.yml)     | Scrape configs, Swarm service discovery, Alertmanager target             |
| [prometheus/alert.rules.yml](prometheus/alert.rules.yml)   | Alert rules for instances and containers                                 |
| [prometheus/alertmanager.yml](prometheus/alertmanager.yml) | Routing (Telegram/email) and receiver definitions                        |

## Contributing

This repository has no separate `CONTRIBUTING` guide. To contribute:

1. Fork the repository and create a feature branch.
2. Keep changes scoped to one stack or concern, and keep the compose files deployable as-is.
3. Open a pull request describing the change and how you tested it.

For larger changes or anything that affects the default behavior of a stack, open an issue first to discuss it.

## License

Distributed under the Apache License 2.0. See [LICENSE](LICENSE) for more information.

## Contact

Project repository: [github.com/capcom6/infra-template](https://github.com/capcom6/infra-template)

## Acknowledgments

- [Best README Template](https://github.com/othneildrew/Best-README-Template) for the structure of this document
- [Traefik](https://traefik.io/) - reverse proxy and TLS termination
- [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), [cAdvisor](https://github.com/google/cadvisor), [Grafana](https://grafana.com/) - monitoring stack
- [MariaDB](https://mariadb.org/), [phpMyAdmin](https://www.phpmyadmin.net/), [Valkey](https://valkey.io/), [Redis Commander](https://github.com/joeferner/redis-commander) - data stores and admin UIs
- [Portainer](https://www.portainer.io/) - container management
- [swarm-cronjob](https://github.com/crazy-max/swarm-cronjob) - scheduled tasks in Swarm
- [mariadb-backup-s3](https://github.com/capcom6/mariadb-backup-s3) - S3 backup tooling
