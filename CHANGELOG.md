# Changelog

Factual record of changes — what changed, not why. Rationale lives in commit
messages and design docs (e.g. `ACL_DESIGN.md`).

## [Unreleased]

## [1.0.0] - 2026-10-11

First public release. Successor to `debian12-ansible`. Requires ansible-core 2.18 or later;
collection version ranges are in `requirements.yml`.

### Added
- `main.yaml` — LNMP provisioning for Debian 13: sshd hardening drop-in, BBR and swapfile
  (`/etc/sysctl.d/` drop-in), unattended-upgrades, nginx.org mainline, MariaDB 11.8 LTS,
  PHP 8.4
- Per-site isolation: one login user, web root `/srv/www/<site>` (750/640, `wp-config.php` 440),
  nginx read via POSIX access + default ACLs, one php-fpm pool per site on a unix socket
- nginx: TLSRef v6.0 TLS (post-quantum `X25519MLKEM768`), security headers, `limit_req`,
  structured HSTS settings, per-site `extra.conf` snippet, catch-all default server
- fail2ban with the nftables action: WordPress (journald), nginx scan, wp-login, forbidden and
  `limit_req` jails, with a filter self-test on every run
- MariaDB: one database and scoped user per site; passwords generated on the server
  (`/root/.db/`), set only on create
- WordPress baseline (`5a-wordpress.yaml`) for every site with a `wp-config.php`:
  `wp_config_constants` (file/dir modes for the ACL model, no file editor, wp-fail2ban log
  tags, core auto-updates), wp-fail2ban at the filters' version, plugin auto-updates with an
  exclude list; changes only what differs
- acme.sh with Let's Encrypt: HTTP-01 by default, DNS-01 via `acme_challenge: dns`; optional
  local credentials file, otherwise kept from the server or prompted
- Inventory layout: `inventory.example/`, defaults in `roles/repo_defaults`, optional secret
  files in `inventory/secrets/`; no secrets needed on the control machine
- Standalone playbooks: Google Cloud CLI and ossutil (ECS RAM role); encrypted daily DB dumps
  and weekly `/srv` mirror to GCS and/or OSS; CronitorCLI job monitoring; report-only
  malware scan (wp-cli checksums, PHP in uploads, maldet with YARA); site file and database
  migration with a cutover interlock
- `7-validate.yaml`, the last step of `main.yaml`: read-only checks for every site — web root
  mode and owner, nginx ACLs (read, no write), `wp-config.php` 440, php-fpm socket, no php-fpm
  on TCP, HTTP status, no PHP execution in uploads, nftables; files with world bits, x in the
  ACL mask or a foreign owner are listed as FLAGGED without failing the run
- `ACL_DESIGN.md` permission model
