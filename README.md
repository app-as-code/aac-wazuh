<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/icon-dark.svg">
    <img src="assets/icon.svg" alt="Wazuh logo" width="128">
  </picture>
</p>

<h1 align="center">aac-wazuh</h1>

<p align="center">Self-hosted deployment definition for the Wazuh SIEM/XDR server stack (single node).</p>

<p align="center">
  <a href="https://github.com/wazuh/wazuh"><img alt="Upstream" src="https://img.shields.io/badge/upstream-wazuh%2Fwazuh-blue?logo=github"></a>
  <img alt="Image" src="https://img.shields.io/badge/image-wazuh%2Fwazuh--dashboard%3A4.14.7-005571?logo=docker&logoColor=white">
  <a href="https://github.com/wazuh/wazuh/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/badge/license-GPL--2.0%20%2F%20AGPL--3.0-green"></a>
</p>

## About

Wazuh is an open-source security platform that combines SIEM and XDR capabilities. Wazuh agents
and agentless log sources send events to a central manager, which analyses them against detection
rules and stores the results in an indexer. A dashboard provides search, alert review, and
compliance and vulnerability views.

This repository defines a single-node Wazuh server stack for Docker Compose. Three containers
(manager, indexer, dashboard) run the same component versions, and a one-shot sidecar generates the
internal TLS certificates on first start.

Key features (as provided by upstream):

- Security event analysis with rule-based detection and alerting
- File integrity monitoring, security configuration assessment, and log analysis
- Vulnerability detection for monitored endpoints
- Agent management, including enrollment and inventory
- Syslog ingestion from agentless devices
- Web dashboard for search, dashboards, and alert review

## Upstream

- Project: <https://github.com/wazuh/wazuh>
- Documentation: <https://documentation.wazuh.com>
- Website: <https://wazuh.com>
- Container images: `wazuh/wazuh-dashboard`, `wazuh/wazuh-manager`, `wazuh/wazuh-indexer`,
  `wazuh/wazuh-certs-generator` (Docker Hub)
- Version: `4.14.7` (set once via `service.image_tag`, used for all Wazuh server images)
- License: GPL-2.0 for most components; AGPL-3.0 for the Wazuh engine (see upstream `LICENSE`)

## Components

| Container | Image | Role |
|---|---|---|
| `aac-wazuh` | `wazuh/wazuh-dashboard` | Web UI, plain HTTP on port 5601 behind a reverse proxy |
| `aac-wazuh.manager` | `wazuh/wazuh-manager` | Agent endpoint, enrollment, syslog, REST API |
| `aac-wazuh.indexer` | `wazuh/wazuh-indexer` | OpenSearch-based storage for alerts and inventory |
| `aac-wazuh-certs-init` | `wazuh/wazuh-certs-generator` | One-shot: generates the TLS certificate set |

Agents must never be newer than the server. Bump the server version first, then the agent definition.

## Quick start

The Compose file is rendered from `service.yml` by the App-as-Code template engine; no rendered
output is stored in this repository. Before the first start:

1. Generate bcrypt hashes for the indexer admin and dashboard server passwords:

   ```bash
   echo -n '<indexer-admin-password>' | scripts/gen-hash.sh   # -> INDEXER_ADMIN_HASH
   echo -n '<dashboard-server-password>' | scripts/gen-hash.sh # -> DASHBOARD_SERVER_HASH
   ```

   The plaintext password and the hash must match. Passwords must not contain `$`.
2. Provide all secrets and hashes listed under [Configuration](#configuration) through the
   deployment pipeline (or your own secret store). Do not commit them.
3. Deploy the rendered stack. On first start `aac-wazuh-certs-init` creates the certificates, and the
   indexer and manager restart until those files exist.
4. Run `scripts/bootstrap.sh` once after the first start. It creates the agent groups and sets the
   index retention policy.

Host prerequisites:

- `vm.max_map_count` must be at least `262144` on the container host.
- About 4 GB RAM headroom beyond the OS (the indexer heap is 2 GB), plus disk for the indices.

## Configuration

Non-secret values come from `service.yml` (`vars`). Values marked `CHANGE_ME` must be set per
deployment. Values marked (secret) are injected from the secret store.

### Secrets

| Variable | Default | Description |
|---|---|---|
| `INDEXER_ADMIN_PASSWORD` | `CHANGE_ME` (secret) | Password of the indexer `admin` user; the manager and dashboard use it to connect |
| `DASHBOARD_SERVER_PASSWORD` | `CHANGE_ME` (secret) | Password of the internal `kibanaserver` user used by the dashboard backend |
| `WAZUH_API_PASSWORD` | `CHANGE_ME` (secret) | Password of the Wazuh REST API user `wazuh-wui` (at least 8 characters, upper and lower case, digit, symbol) |
| `WAZUH_AUTHD_PASSWORD` | `CHANGE_ME` (secret) | Agent enrollment password; the agent definition needs the same value |

### Non-secret variables

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Time zone for all containers |
| `INDEXER_ADMIN_HASH` | `CHANGE_ME` | bcrypt hash of `INDEXER_ADMIN_PASSWORD` for the indexer's internal users |
| `DASHBOARD_SERVER_HASH` | `CHANGE_ME` | bcrypt hash of `DASHBOARD_SERVER_PASSWORD` for the indexer's internal users |
| `INDEXER_JAVA_OPTS` | `-Xms2g -Xmx2g` | JVM heap of the indexer (keep min and max equal) |
| `CERT_TOOL_VERSION` | `4.14` | Certificate tool version, matching the server major.minor |
| `CERT_OU` / `CERT_O` | `Wazuh` | Certificate subject organizational unit and organization |
| `CERT_L` / `CERT_C` | `California` / `US` | Certificate subject locality and country |
| `SYSLOG_ALLOWED_IPS` | `CHANGE_ME` | Comma-separated networks allowed to send raw syslog to the manager |
| `AGENTS_DISCONNECTION_TIME` | `10m` | Time without contact after which an agent is marked disconnected |
| `AGENTS_DISCONNECTION_ALERT_TIME` | `1h` | Time after which a disconnect alert is raised |
| `VD_FEED_UPDATE_INTERVAL` | `60m` | Interval for refreshing the vulnerability feed |
| `LOG_ALERT_LEVEL` | `3` | Minimum rule level written to the alert log |
| `TRUENAS_SYSLOG_HOSTNAME` | `CHANGE_ME` | Hostname token the appliance syslog decoder matches on (the short hostname the sending device puts in its syslog lines) |

### Ports

| Port | Protocol | Exposed | Purpose |
|---|---|---|---|
| `5601` | TCP | via reverse proxy | Dashboard (HTTP; TLS is terminated by the proxy) |
| `1514` | TCP | host | Agent communication |
| `1515` | TCP | host | Agent enrollment |
| `514` | TCP and UDP | host | Syslog from agentless devices |
| `55000` | TCP | via reverse proxy only | Wazuh REST API (not published on the host) |

### Volumes

| Volume | Mount | Description |
|---|---|---|
| `certs` | `/etc/ssl/wazuh`, `/usr/share/wazuh-indexer/config/certs`, `/usr/share/wazuh-dashboard/certs` | TLS certificate set, generated once by `certs-init` |
| `certs_ca` | `/ca-private` (certs-init only) | CA private keys, readable only by the sidecar |
| `indexer_data` | `/var/lib/wazuh-indexer` | Indexed alerts and inventory |
| `dashboard_data` | dashboard config directory | Dashboard application configuration |
| `dashboard_custom` | dashboard custom assets | Custom branding assets |
| `dashboard_osd_yml` | `opensearch_dashboards.yml` | Rendered dashboard server configuration |
| `manager_*` | `/var/ossec/...` | Manager state: configuration, logs, queue, integrations, wodles, active response |
| `filebeat_etc`, `filebeat_var` | `/etc/filebeat`, `/var/lib/filebeat` | Filebeat configuration and state inside the manager container |

## Rules and decoders

Local rules and appliance decoders are part of the configuration templates in
`custom_templates/files/manager_config/`. They are copied into the manager on every start.
Run the rule test before committing changes:

```bash
scripts/test-rules.sh   # loads rules and decoders into a throwaway container and runs analysisd -t
```

Do not edit files inside a running manager container; a rollout recreates it.

## Deploy with App-as-Code

1. Edit `service.yml`. It is the single source of truth for image, ports, environment,
   volumes, dependencies and per-stage overrides. Do not edit the rendered output.
2. Pushing to `main` (or a configured stage branch) runs the service pipeline, which renders
   the manifests with `aac-template-engine` and triggers the deployment.
3. Releases of this definition are published under <https://github.com/app-as-code/aac-wazuh>.

Configuration files are rendered from `custom_templates/files/`. The manager copies them over
`/var/ossec` on every start, so changes made in the Wazuh UI are not persistent.

## Credits and license

Wazuh is developed by Wazuh Inc. and the open-source community and is licensed under GPL-2.0 and
AGPL-3.0 (see the upstream `LICENSE`). This repository only contains the deployment definition for
it and is not a fork of the application. The logo is provided by the selfh.st icon collection; see
[`assets/ICON_SOURCE.md`](assets/ICON_SOURCE.md). Wazuh is a trademark of Wazuh Inc.
