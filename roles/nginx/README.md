# Ansible Role: ahmz.server_setup.nginx

A production-grade, highly declarative Nginx reverse proxy and web server role.

## Description

This role completely automates the deployment of a hardened Nginx edge server. It features an advanced Jinja macro-based virtual host generator, automated OWASP ModSecurity WAF integration, dynamic trusted proxy resolution (for CDNs like Cloudflare), and a built-in JSON-to-Prometheus observability pipeline.

### Core Capabilities
* **Declarative VHosts:** Generate complex server blocks, including WebSocket proxies and location-specific rate limits, using simple YAML lists.
* **Dynamic Real-IP:** Fetch trusted proxy CIDRs from URLs, IPsets, or static lists to securely parse `X-Forwarded-For` headers.
* **Security First:** Automatically strips `Server` headers (via `headers-more`), applies strict CSP/HSTS/X-Frame headers, and configures modern TLS ciphers.
* **Observability:** Formats access logs in JSON and configures Grafana Promtail to extract Prometheus metrics (`request_duration_seconds`, `requests_total`) without exploding label cardinality.
* **State Safety:** Automatically retains rolling backups of all configuration files before applying updates.

---

## Role Variables: Global Configuration

| Variable | Description | Default |
|---|---|---|
| `nginx_managed` | Master toggle to execute the role. | `false` |
| `nginx_waf_install` | Install and configure ModSecurity & OWASP CRS. | `true` |
| `nginx_exporter_install` | Install `nginx-prometheus-exporter`. | `true` |
| `nginx_promtail_install` | Install Grafana Promtail for Loki log shipping. | `false` |
| `nginx_remove_default_site`| Deletes the default `/var/www/html` welcome site. | `true` |
| `nginx_backup_enabled` | Retain rolling backups in `/etc/nginx/backups`. | `true` |
| `nginx_worker_connections` | Max simultaneous connections per worker. | `65535` |

### Trusted Proxies (Real IP)
Define IPs that Nginx should trust to provide the real client IP. Supports static CIDRs, local IPset files, and remote URL fetching.

```yaml
nginx_real_ip_header: "X-Forwarded-For"
nginx_trusted_proxies:
  - "10.0.0.0/8"                           # Internal network
  - { ipset: "cloudflare-v4" }             # Read from /var/lib/ipset/cloudflare-v4.save
  - { url: "[https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4)" } # Fetched at runtime
```

---

## Role Variables: Site Definitions (`nginx_sites`)

The `nginx_sites` variable is a list of dictionaries representing virtual hosts. The template engine automatically handles HTTP -> HTTPS redirects and certificate paths.

### Site Object Properties
| Key | Type | Description |
|---|---|---|
| `domain` | String | **(Required)** Primary domain name (e.g., `api.example.com`). |
| `aliases` | List | Alternative domains to listen on. |
| `enabled` | Bool | Whether the site symlink should exist. (Default: `true`) |
| `ssl_enabled` | Bool | Enables port 443 and SSL/TLS configuration. (Default: `false`) |
| `ssl_cert` / `ssl_key` | String | Absolute paths to SSL files. Defaults to `/etc/ssl/certs/<domain>/...` |
| `acme_challenge_enabled`| Bool | Serves `/.well-known/acme-challenge/` from a local webroot. |
| `upstream` | String | URL to proxy root (`/`) requests to (e.g., `http://127.0.0.1:8080`). |
| `redirect_to` | String | If set, issues a 301 redirect to this exact URL/domain. |
| `waf_enabled` | Bool | Enables ModSecurity for this specific site. |
| `waf_extra_rules` | String | Raw ModSecurity rules (ideal for `SecRuleRemoveById` false positive tuning). |
| `header_*` | String | Override global security headers (e.g., `header_x_frame_options: "DENY"`). |
| `nginx_config` | Dict | Key-value mapping of standard Nginx directives (e.g., `client_max_body_size: 50M`). |
| `extra_config` | String | Raw Nginx configuration string injected into the server block. |

### Location Object Properties (`locations`)
You can define specific behavior for sub-paths under a site.

| Key | Type | Description |
|---|---|---|
| `path` | String | **(Required)** The location path (e.g., `/api/v1/`). |
| `upstream` | String | Backend URL to proxy this specific path to. |
| `rate_limit` | Dict | Keys: `zone`, `burst`, `nodelay`. Applies rate limiting to this path. |
| `cache_control` | String | Sets the `Cache-Control` header for this path. |
| `log_prefix` | String | Isolates access/error logs to `<domain>_<log_prefix>_access.log`. |
| `nginx_config` | Dict | Location-specific Nginx directives. |

---

## Example Configurations

### 1. Simple Reverse Proxy (Backend Application)
```yaml
nginx_sites:
  - domain: "app.internal.local"
    upstream: "[http://127.0.0.1:3000](http://127.0.0.1:3000)"
    nginx_config:
      client_max_body_size: "100M"
```

### 2. High-Security Public API
Full SSL, ModSecurity WAF enabled, strict rate limits on specific endpoints, and false-positive WAF rules bypassed.
```yaml
nginx_sites:
  - domain: "api.example.com"
    ssl_enabled: true
    ssl_cert: "/etc/ssl/certs/api.cert"
    ssl_key: "/etc/ssl/private/api.key"
    waf_enabled: true
    waf_extra_rules: |
      # Bypass false positive on JSON payload rule
      SecRuleRemoveById 942100 
    upstream: "[http://127.0.0.1:8080](http://127.0.0.1:8080)"
    locations:
      - path: "/auth/login"
        upstream: "[http://127.0.0.1:8080](http://127.0.0.1:8080)"
        rate_limit:
          zone: "global_limit"
          burst: 5
          nodelay: true
        log_prefix: "auth"
```

### 3. Static Site & 301 Redirect
```yaml
nginx_sites:
  - domain: "old-domain.com"
    redirect_to: "[https://new-domain.com](https://new-domain.com)"
    
  - domain: "new-domain.com"
    ssl_enabled: true
    nginx_config:
      root: "/var/www/new-domain/html"
      index: "index.html"
    locations:
      - path: "/assets/"
        cache_control: "public, max-age=31536000, immutable"
```

## Advanced: Observability Pipeline
When `nginx_promtail_install` is `true`, the role configures Nginx to output logs in a custom JSON format (`json_analytics`). Promtail parses this JSON and automatically builds Prometheus metrics. 

Because URLs can have high cardinality (e.g., `/user/123`, `/user/456`), the macro automatically injects `set $endpoint_name "<path>";` into location blocks. Promtail groups metrics by this `$endpoint_name` rather than the raw `$request_uri`, keeping your Prometheus server safe from cardinality explosions.

## License
MIT
