# BunkerWeb Config Generator

A static single-page web app that generates hardened [BunkerWeb](https://www.bunkerweb.io) configuration files for all official templates. Select a template, fill in your service details, and get a ready-to-use config with one click.

**Live demo:** deployed automatically to GitHub Pages on every push to [Live Demo](https://bunkerweb-config.palmcoasty.com).

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Supported Templates](#supported-templates)
- [How to Use](#how-to-use)
  - [1. Select a Template](#1-select-a-template)
  - [2. Fill in Your Settings](#2-fill-in-your-settings)
  - [3. Generate & Copy](#3-generate--copy)
- [Output Format](#output-format)
- [Template Reference](#template-reference)
  - [WordPress](#wordpress)
  - [Nextcloud](#nextcloud)
  - [Drupal](#drupal)
  - [Jellyfin](#jellyfin)
  - [Tomcat](#tomcat)
  - [NetBird](#netbird)
  - [Xen Orchestra](#xen-orchestra)
- [Field Types Explained](#field-types-explained)
- [Locked vs Editable Fields](#locked-vs-editable-fields)
- [Deploying to GitHub Pages](#deploying-to-github-pages)
- [Template Caching](#template-caching)
- [Project Structure](#project-structure)
- [Adding New Templates](#adding-new-templates)

---

## Overview

BunkerWeb is a security-focused reverse proxy built on NGINX. It uses environment variables to configure everything — SSL, rate limiting, ModSecurity WAF, allow/block lists, and more. While powerful, writing these configs by hand is tedious and error-prone.

This generator reads template definitions directly from the upstream [bunkerity/bunkerweb-templates](https://github.com/bunkerity/bunkerweb-templates) repository on GitHub, presents a guided form for each template, and outputs a clean `KEY=value` config you can paste straight into your deployment.

**No build step. No server. No dependencies.** It is a single `index.html` file.

---

## Features

| Feature | Details |
|---|---|
| **All official templates** | WordPress, Nextcloud, Drupal, Jellyfin, Tomcat, NetBird, Xen Orchestra |
| **Auto-fetches from GitHub** | Always reflects the latest upstream templates |
| **localStorage cache** | Templates cached for 1 hour — no repeated downloads on every page load |
| **Smart field types** | Toggles for yes/no, dropdowns for enums, textarea for long values |
| **Conditional fields** | DNS challenge fields only appear when DNS-01 is selected; custom SSL fields only appear when custom SSL is enabled |
| **Security-locked fields** | Hardened defaults shown read-only with an explanation — you can't accidentally weaken them |
| **WordPress REST API toggle** | One toggle adds/removes PUT and DELETE from `ALLOWED_METHODS` |
| **ModSec config files** | Inline preview and copy button for each ModSecurity `.conf` file |
| **RAW output** | Clean `KEY=value` output — no comments, no blank lines, ready to paste |
| **One-click copy** | Copy button on every output block |
| **GitHub Pages ready** | Included GitHub Actions workflow deploys automatically |

---

## Supported Templates

| Template | Use Case | Config Files |
|---|---|---|
| **WordPress** | Self-hosted WordPress behind BunkerWeb | `modsec/wordpress_false_positives.conf` |
| **Nextcloud** | Nextcloud file sync with WebDAV | `modsec/nextcloud_false_positives.conf` |
| **Drupal** | Drupal CMS | *(none)* |
| **Jellyfin** | Jellyfin media server with WebSocket | `modsec-crs/jellyfin_false_positives.conf` |
| **Tomcat** | Apache Tomcat Java servlet container | *(none)* |
| **NetBird** | NetBird self-hosted VPN with gRPC | `modsec/netbird_false_positives.conf` |
| **Xen Orchestra** | Xen Orchestra virtualization UI | `modsec/xen-orchestra_false_positives.conf` |

---

## How to Use

### 1. Select a Template

The landing page shows a card for each available template. Each card shows:
- The template name and a short description
- Number of configuration steps
- Whether the template includes additional ModSecurity config files

Click a card to open the configuration form for that template.

A cache status indicator below the heading shows when the templates were last fetched from GitHub. Click **Refresh from GitHub** to force a fresh fetch.

---

### 2. Fill in Your Settings

The form is divided into **collapsible steps** matching the template's original structure. Click a step header to expand or collapse it. The first step is open by default.

#### Fields you must fill in

| Field | Description |
|---|---|
| **Server Name** | The public hostname(s) your service is reachable at. Space-separated for multiple domains. Example: `example.com www.example.com` |
| **Backend URL** | The internal URL BunkerWeb will proxy requests to. Example: `http://wordpress:80` or `http://192.168.1.10:8096` |

Required fields are marked with a green asterisk (`*`). The generator will not produce output until all required fields have a value.

#### SSL / TLS settings

By default, all templates use **Auto Let's Encrypt** (the `AUTO_LETS_ENCRYPT=yes` setting). This means BunkerWeb will automatically obtain and renew a certificate for your domain.

**To use DNS-01 challenge** (required for wildcards or when port 80 is not reachable):
1. Change **Challenge Type** to `dns`
2. Three additional fields appear: **DNS Provider**, **DNS Propagation**, and **DNS Credential**
3. Fill in your DNS provider name (e.g. `cloudflare`) and the credential item name

**To use a custom certificate** instead of Let's Encrypt:
1. Enable the **Custom SSL Cert** toggle
2. Choose whether to provide the certificate as a **file path** or **inline data**
3. Fill in the path or paste the PEM content

#### Template-specific fields

Each template exposes only the fields relevant to that application. See the [Template Reference](#template-reference) section for a per-template breakdown.

---

### 3. Generate & Copy

Click **Generate Configuration** at the bottom of the form.

The output section appears below with:

- **RAW tab** — a clean `KEY=value` config ready to paste into a `.env` file or a `docker-compose.yml` environment block
- **ModSec config tabs** — one tab per additional ModSecurity `.conf` file the template requires (e.g. `wordpress_false_positives.conf`)

Every output block has a **Copy** button that copies the full content to the clipboard.

---

## Output Format

### RAW config

```
SERVER_NAME=example.com
AUTO_LETS_ENCRYPT=yes
USE_LETS_ENCRYPT_STAGING=no
USE_LETS_ENCRYPT_WILDCARD=no
LETS_ENCRYPT_CHALLENGE=http
USE_REVERSE_PROXY=yes
REVERSE_PROXY_URL=/
REVERSE_PROXY_HOST=http://wordpress:80
MAX_CLIENT_SIZE=50m
ALLOWED_METHODS=GET|POST|HEAD|OPTIONS
...
```

No comments. No blank lines. Every key that has a non-empty value is included exactly once.

### Using the output

**Docker Compose** — paste into the `environment:` block of your BunkerWeb service:

```yaml
services:
  bunkerweb:
    image: bunkerity/bunkerweb:latest
    environment:
      - SERVER_NAME=example.com
      - AUTO_LETS_ENCRYPT=yes
      - REVERSE_PROXY_HOST=http://wordpress:80
      # ... rest of the output
```

**`.env` file** — if you use `env_file:` in Compose or pass a `.env` file directly, paste the RAW output as-is into that file.

### ModSecurity config files

Templates that include a ModSecurity false-positive config (most of them) will show an additional tab in the output. The content of that tab must be saved as a `.conf` file and mounted into the BunkerWeb container at the path shown in the tab header.

Example for WordPress:

```
configs/modsec/wordpress_false_positives.conf
```

Mount it in your Compose file:

```yaml
volumes:
  - ./configs/modsec/wordpress_false_positives.conf:/etc/bunkerweb/configs/modsec/wordpress_false_positives.conf:ro
```

---

## Template Reference

### WordPress

**Steps:** Front service → Upstream server → HTTP settings → SEO whitelisting → Performance → Security

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `www.example.com` | Your WordPress domain(s) |
| `REVERSE_PROXY_HOST` | `http://mywp` | Internal URL of your WordPress container |
| `MAX_CLIENT_SIZE` | `50m` | Maximum upload size — raise this if your WordPress media uploads are larger |
| `WHITELIST_IP` | *(empty)* | IPs to always allow through (e.g. your office IP) |
| `WHITELIST_ASN` | *(empty)* | ASNs to always allow through (e.g. for CDN access) |

#### WordPress REST API toggle

The **WordPress REST API** toggle in the HTTP step controls `ALLOWED_METHODS`:

| Toggle state | `ALLOWED_METHODS` value | Effect |
|---|---|---|
| Off (default) | `GET\|POST\|HEAD\|OPTIONS` | REST API endpoints that require PUT/DELETE are blocked |
| On | `GET\|POST\|HEAD\|OPTIONS\|PUT\|DELETE` | Full REST API access enabled |

Enable this if you use plugins or clients that rely on the WP REST API (e.g. mobile apps, headless frontends, the block editor's REST features).

#### Config file

`modsec/wordpress_false_positives.conf` — enables the `wordpress-rule-exclusions` CRS plugin and adds exclusions for the admin AJAX endpoint, the heartbeat API, and XML-RPC.

---

### Nextcloud

**Steps:** Front door → Upstream service → HTTP / WebDAV → Performance → Security

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `www.example.com` | Your Nextcloud domain |
| `REVERSE_PROXY_HOST` | `http://nextcloud` | Internal URL of your Nextcloud container |
| `MAX_CLIENT_SIZE` | `10G` | Maximum upload size — the default allows 10 GB file uploads |
| `LIMIT_REQ_RATE_1` | `5r/s` | Rate limit for `/apps` |
| `LIMIT_REQ_RATE_2` | `8r/s` | Rate limit for `/apps/text/session/sync` (Nextcloud Text editor) |
| `LIMIT_REQ_RATE_3` | `5r/s` | Rate limit for `/core/preview` (thumbnail generation) |

#### Config file

`modsec/nextcloud_false_positives.conf` — enables the `nextcloud-rule-exclusions` CRS plugin and permits WebDAV methods (`PROPFIND`, `MKCOL`, `LOCK`, etc.) required by sync clients.

---

### Drupal

**Steps:** Front service → Upstream server → Performance → Security

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `www.example.com` | Your Drupal domain |
| `REVERSE_PROXY_HOST` | `http://mydrupal` | Internal URL of your Drupal container |
| `LIMIT_REQ_RATE` | `2r/s` | Global rate limit per client |
| `LIMIT_REQ_RATE_1` | `5r/s` | Separate rate limit for the Drupal installer (`/core/install.php`) |

---

### Jellyfin

**Steps:** Front service → Upstream service → Streaming → Security / Headers

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `www.example.com` | Your Jellyfin domain |
| `REVERSE_PROXY_HOST` | `http://jellyfin:8096` | Main Jellyfin backend URL |
| `REVERSE_PROXY_HOST_1` | `http://jellyfin:8096` | WebSocket backend URL (same host, separate route) |
| `MAX_CLIENT_SIZE` | `20M` | Upload size limit |
| `REVERSE_PROXY_SEND_TIMEOUT` | `300s` | Send timeout — increase for slow uploads |
| `REVERSE_PROXY_READ_TIMEOUT` | `300s` | Read timeout — increase for large media streams |
| `CONTENT_SECURITY_POLICY` | *(preset)* | CSP header — the default allows YouTube embeds and blob: workers |
| `PERMISSIONS_POLICY` | *(preset)* | Permissions Policy header — disables most browser APIs |

#### WebSocket routing

Jellyfin requires a dedicated WebSocket route (`/socket`) for real-time updates. This route is pre-configured and locked — you only need to set the backend host.

#### Config file

`modsec-crs/jellyfin_false_positives.conf` — adds CRS exclusions for Jellyfin's config API and bundle files.

---

### Tomcat

**Steps:** Front service → Upstream server → Performance → HTTP / Servlet tuning

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `www.example.com` | Your Tomcat domain |
| `REVERSE_PROXY_HOST` | `http://mytomcat:8080` | Internal URL of your Tomcat instance |
| `MAX_CLIENT_SIZE` | `100m` | Upload size limit — Tomcat apps often handle larger payloads |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|OPTIONS` | Permitted HTTP methods — edit to restrict for your specific app |

---

### NetBird

**Steps:** Domain & HTTPS → Backend services → HTTP & WebSocket routes → gRPC & long-lived connections → Request limits & ModSecurity

NetBird has the most complex routing of all templates because it requires multiple backend services, WebSocket connections, and gRPC proxying.

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `netbird.example.com` | Your NetBird domain |
| `REVERSE_PROXY_HOST` | `http://netbird-server` | NetBird server — relay route |
| `REVERSE_PROXY_HOST_1` | `http://netbird-server` | NetBird server — WebSocket proxy route |
| `REVERSE_PROXY_HOST_2` | `http://netbird-server` | NetBird server — API route |
| `REVERSE_PROXY_HOST_3` | `http://netbird-server` | NetBird server — OAuth2 route |
| `REVERSE_PROXY_HOST_999` | `http://netbird-dashboard` | NetBird dashboard UI (catch-all route) |
| `GRPC_HOST` | `grpc://netbird-server:80` | gRPC backend for the signal exchange service |
| `GRPC_HOST_1` | `grpc://netbird-server:80` | gRPC backend for the management service |
| `GRPC_READ_TIMEOUT` | `1d` | Read timeout for persistent gRPC connections |
| `GRPC_SEND_TIMEOUT` | `1d` | Send timeout for persistent gRPC connections |
| `LIMIT_CONN_MAX_HTTP1` | `30` | Max simultaneous HTTP/1.1 connections per IP |
| `LIMIT_CONN_MAX_HTTP2` | `200` | Max simultaneous HTTP/2 streams per IP |

#### Route map

| Path | Backend | Protocol |
|---|---|---|
| `/relay` | `REVERSE_PROXY_HOST` | HTTP + WebSocket |
| `/ws-proxy/` | `REVERSE_PROXY_HOST_1` | WebSocket |
| `/api/` | `REVERSE_PROXY_HOST_2` | HTTP |
| `/oauth2/` | `REVERSE_PROXY_HOST_3` | HTTP |
| `/signalexchange.SignalExchange/` | `GRPC_HOST` | gRPC |
| `/management.ManagementService/` | `GRPC_HOST_1` | gRPC |
| `/` (catch-all) | `REVERSE_PROXY_HOST_999` | HTTP |

#### Config file

`modsec/netbird_false_positives.conf` — adds CRS exclusions for the OAuth2 endpoints (`/oauth2/auth` and `/oauth2/token`) which trigger SQL injection false positives.

---

### Xen Orchestra

**Steps:** Front service → Reverse proxy → Performance → Rate limiting → Security

#### Key editable settings

| Setting | Default | Description |
|---|---|---|
| `SERVER_NAME` | `xo.example.com` | Your Xen Orchestra domain |
| `REVERSE_PROXY_HOST` | `https://192.168.1.1` | Direct HTTPS IP of your Xen Orchestra instance |
| `MAX_CLIENT_SIZE` | `100m` | Upload size limit for VM disk imports |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|OPTIONS\|PUT\|DELETE\|PATCH` | Full method set required by the JSON-RPC API |
| `LIMIT_REQ_RATE` | `6r/s` | Global rate limit |
| `LIMIT_REQ_RATE_1` | `15r/s` | Higher limit for `/jsonrpc` (API polling) |
| `LIMIT_CONN_MAX_HTTP1` | `50` | Max HTTP/1.1 connections per IP |
| `HTTP2` | `yes` | Enable HTTP/2 |
| `HTTP3` | `yes` | Enable HTTP/3 / QUIC |
| `SSL_PROTOCOLS` | `TLSv1.2 TLSv1.3` | Minimum TLS version |
| `SECURITY_MODE` | `block` | Set to `detect` for monitoring-only mode |
| `USE_ANTIBOT` | `no` | Bot detection (disabled by default — can be enabled) |
| `USE_DNSBL` | `no` | DNS blacklist checking (disabled by default — can be enabled) |

#### Notes

- Xen Orchestra is typically accessed via HTTPS directly to its IP with a self-signed cert. The `REVERSE_PROXY_HOST` should be `https://` + your XO IP.
- `USE_BAD_BEHAVIOR` is locked to `no` — Xen Orchestra's JSON-RPC API generates request patterns that trigger false positives in the bad behavior detector.
- `USE_BLACKLIST` is locked to `yes` — IP threat intelligence feeds are always active.

#### Config file

`modsec/xen-orchestra_false_positives.conf` — adds CRS exclusions for the `/jsonrpc` endpoint, which generates complex JSON bodies that trigger SQL injection rules.

---

## Field Types Explained

| Type | Appearance | Used for |
|---|---|---|
| **Text input** | Single-line text box | Hostnames, URLs, sizes, timeouts |
| **Toggle** | Green/grey switch | Any `yes`/`no` setting |
| **Select** | Dropdown menu | Fields with a fixed set of valid values (e.g. `http`/`dns`, `block`/`detect`) |
| **Textarea** | Multi-line text box | Long values — CSP headers, Permissions Policy, certificate data |
| **Locked** | Grey read-only display + lock icon | Security-hardened defaults that must not be changed |

---

## Locked vs Editable Fields

The generator divides all settings into two categories.

### Locked fields (read-only)

These are shown with a lock icon and a brief explanation. They represent security defaults that are intentionally hardened and should not be changed for normal deployments.

| Field | Locked value | Reason |
|---|---|---|
| `USE_REVERSE_PROXY` | `yes` | All templates are reverse-proxy setups |
| `SERVE_FILES` | `no` | Files are served by the backend, not BunkerWeb directly |
| `USE_LIMIT_REQ` | `yes` | Rate limiting must always be active |
| `LIMIT_REQ_URL` / `LIMIT_REQ_URL_N` | *(various paths)* | URL patterns are pre-tuned per template |
| `REVERSE_PROXY_URL` / `REVERSE_PROXY_URL_N` | *(various paths)* | Routing paths are pre-configured per template |
| `REVERSE_PROXY_WS` / `REVERSE_PROXY_WS_1` | `yes` | WebSocket flag is set where required by the app |
| `WHITELIST_RDNS` | *(search engine list)* | Pre-configured for Google, Bing, Yandex, Baidu crawlers |
| `WHITELIST_RDNS_GLOBAL` | `yes` | Required for crawler verification to work |
| `MODSECURITY_CRS_PLUGINS` | *(app-specific)* | The correct CRS plugin is pre-selected per template |
| `USE_MODSECURITY` | `yes` | WAF must always be enabled |
| `USE_MODSECURITY_CRS` | `yes` | OWASP CRS must always be active |
| `USE_BLACKLIST` | `yes` | IP threat intel feeds are always active |
| `GRPC_URL` / `GRPC_URL_N` | *(service paths)* | Fixed gRPC method prefixes for NetBird |
| `GRPC_SOCKET_KEEPALIVE` / `_1` | `on` | Required for persistent gRPC connections |

### Editable fields

Everything not in the locked list is editable in the form. This includes all backend URLs, domain names, upload sizes, rate limits, timeouts, SSL options, security headers, and template-specific toggles.

---

## Deploying to GitHub Pages

The repository includes a GitHub Actions workflow at [.github/workflows/deploy.yml](.github/workflows/deploy.yml).

### Setup steps

1. **Push this repository to GitHub.**

2. **Enable GitHub Pages** in your repository settings:
   - Go to **Settings → Pages**
   - Under **Source**, select **GitHub Actions**

3. **Push to `main`.** The workflow will run automatically and deploy `index.html` as a static site.

4. The deployed URL will be:
   ```
   https://<your-username>.github.io/<repo-name>/
   ```

### Manual trigger

The workflow also supports `workflow_dispatch`, meaning you can re-run it manually from the **Actions** tab in your repository without pushing a commit.

### What the workflow does

```yaml
- Checkout the repository
- Configure GitHub Pages
- Upload the root directory as the Pages artifact
- Deploy to GitHub Pages
```

No build tools, no Node.js, no package manager. The entire deployment is the `index.html` file.

---

## Template Caching

To avoid hitting the GitHub API and downloading all template JSON files on every page load, templates are cached in the browser's `localStorage`.

| Detail | Value |
|---|---|
| Cache key | `bwcg_cache_v2` |
| TTL | 1 hour |
| What is cached | All template JSON + all referenced ModSecurity `.conf` file contents |

A **cache status indicator** is shown on the template selection page:
- Green dot — cache is fresh (loaded within the last hour)
- Yellow dot — cache is stale (older than 1 hour, will be refreshed on next load)

Click **Refresh from GitHub** to force an immediate re-fetch regardless of cache age. This is useful when a new template has been added to the upstream repository.

---

## Project Structure

```
bunkerweb-conf-generator/
├── index.html                          # The entire application (HTML + CSS + JS)
└── .github/
    └── workflows/
        └── deploy.yml                  # GitHub Actions — deploy to GitHub Pages
```

Everything lives in `index.html`. There is no build step, no `node_modules`, no bundler. Open the file directly in a browser to run it locally.

---

## Adding New Templates

New templates are picked up **automatically** — no changes to this repository are required.

When a new template is added to [bunkerity/bunkerweb-templates](https://github.com/bunkerity/bunkerweb-templates):

1. The generator fetches the updated directory listing from the GitHub API on the next cache refresh (within 1 hour, or immediately after clicking **Refresh from GitHub**)
2. The new template's `template.json` and any referenced config files are downloaded and cached
3. A new card appears in the template grid

The only case where a new template might need attention is if it introduces a setting key not covered by `FIELD_META` in `index.html`. In that case the field will still appear and work — it will just show the raw key name as its label rather than a human-readable one. To add metadata for a new key, add an entry to the `FIELD_META` object in [index.html](index.html).
