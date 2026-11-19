# WordPress Hardened (Helm chart)

A Helm chart that runs WordPress on Kubernetes with hardened **FROM scratch**
images: no shell, no package manager, non-root.

| Container | Image | Role |
|---|---|---|
| `nginx` | [`jbsky/nginx-waf-hardened`](https://github.com/jbsky/nginx-hardened) | serves static files, proxies PHP, ModSecurity + OWASP CRS |
| `php-fpm` | [`jbsky/php-fpm-hardened`](https://github.com/jbsky/php-fpm-hardened) | runs WordPress |
| `nginx-exporter` | `nginx/nginx-prometheus-exporter` | nginx metrics |
| `phpfpm-exporter` | `hipages/php-fpm_exporter` | FPM pool metrics |
| `redis` *(optional)* | `redis:alpine` | WordPress persistent object cache |

An Alpine **initContainer** prepares the volume: it downloads the latest
stable WordPress release and drops it into a shared `emptyDir`. That is what
lets the php-fpm image stay shell-free -- it never has to copy a file.

## What the chart does not do

- **No database.** WordPress expects a reachable MySQL/MariaDB; the
  connection settings come with the site configuration (see below).
- **No ingress.** The `Service` is a `ClusterIP`; routing, TLS termination
  and the edge WAF live upstream.
- **No pinned WordPress version.** The initContainer takes the latest stable
  release on every pod start. That is a choice: no image to rebuild on every
  release, at the cost of reproducibility.

## Installation

The chart carries no site configuration. It is dropped into the chart
directory before installing, under three paths the templates pick up:

```
php/*                   -> ConfigMap (php.ini, www.conf, wp-config.php...)
nginx/*                 -> ConfigMap (nginx.conf, vhost, ModSecurity rules)
files/mu-plugins/*.php  -> ConfigMap, mounted as mu-plugins
```

```bash
cp -a my-site/php my-site/nginx my-site/files .
helm upgrade my-site . --install \
  --namespace my-site --create-namespace \
  --values my-site/values.yaml
```

## Values

Images, resources and security contexts are set per container (`nginx`,
`phpfpm`, `nginxexporter`, `phpfpmexporter`). Pin the hardened images to
their current immutable tag, given in the Tags section of each image's README:
their registries keep only the last six. Three blocks deserve a mention.

### `egress` -- internet access of the initContainer

Default: direct access, no proxy.

```yaml
egress:
  proxy: "http://proxy.example:3128"
  proxyCA: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

`proxyCA` is only needed when the proxy **intercepts TLS**: it then presents
its own authority, and without it the first HTTPS call fails with
`self-signed certificate in certificate chain` -- `apk` included, so before
`curl` is even installed.

### `redis` -- object cache

```yaml
redis:
  enabled: true
```

Disabled by default. When enabled, the initContainer installs the
`redis-cache` plugin if it is missing and **resynchronizes the
`object-cache.php` drop-in on every start**: that is the copy WordPress loads,
and it must never diverge from the plugin version actually installed. The
`WP_REDIS_*` constants stay yours to set, in the site configuration.

### `wordfence` -- extended protection

```yaml
wordfence:
  extendedProtection: true
```

Writes `wordfence-waf.php` then `.user.ini` into the volume, in that order.
An `auto_prepend_file` pointing to a missing file is not a warning but a
**fatal error**: every page returns 500. Only enable it when the plugin is
actually installed.

## Security context

Both application containers run under the same non-root UID (1999 by default)
-- that is what makes the shared volume readable on both sides -- with
`allowPrivilegeEscalation: false` and, for nginx, every capability dropped.

## License

This chart is licensed under **Apache-2.0**, see [`LICENSE`](LICENSE).
Copyright 2026 jbsky.

The images it deploys each have their own, in their own repository; the
software they ship keeps its upstream's.

Contributions: see [`CONTRIBUTING.md`](CONTRIBUTING.md); a `Signed-off-by` is
required.
