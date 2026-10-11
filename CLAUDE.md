# debian13-ansible

Ansible proof-of-concept for a hardened, multi-tenant LNMP server (nginx + MariaDB +
PHP-FPM) on **Debian 13 (trixie)**, hosting WordPress sites. It demonstrates a
security model: per-site UID isolation, unix-socket php-fpm, ACL-based nginx read,
nftables + fail2ban, no secrets on the control machine, encrypted off-site backups, monitored crons.
Treat it as public: no real hostnames, IPs, bucket names, or account details in tracked files.

## Layout

- `main.yaml` — imports the numbered core playbooks `1-install` … `7-validate` in order.
- `standalone_playbooks/` — opt-in extras run individually (backups, monitoring,
  migration, malware scan); their scripts are Jinja templates in `standalone_playbooks/templates/`.
- `templates/`, `files/` — config for the core playbooks.
- `roles/repo_defaults/defaults/main.yml` — defaults for every play (each play lists the
  role; lowest precedence, so any inventory value overrides it).
- `inventory/` — gitignored; the user's private inventory repo is cloned here
  (`hosts.yml` + `group_vars/webservers/`, optional `secrets/`). `inventory.example/`
  documents every file's schema; per-site data (`users:` list) is in `users.yml`.
- `README.md` — usage and the permission model. `ACL_DESIGN.md` — ACL deep dive.
  `CHANGELOG.md` — factual change index. `7-validate.yaml` (last step of `main.yaml`) —
  read-only checks that the isolation model holds for every site.

## Working here

- **No secrets on the control machine.** A secret lives where it is used — on the server
  (e.g. DB passwords in `/root/.db/` and `wp-config.php`) or in a cloud secret manager — and
  the repo and inventory hold only config and references. Prefer not storing a secret at
  all over storing it securely; a playbook must not need a local secret to re-run safely.
- Follow `.claude/rules/` (changelog entry with every functional change, terse comments,
  drop-ins over stock config, verify third-party installs against current upstream).
- Never run playbooks against a host, or anything that touches the server, unless asked.
  Local checks are fine: `ansible-playbook --syntax-check <playbook>`.
- Pin downloaded third-party artifacts by version + sha256 (see `4-cronitor.yaml`).
- Cron jobs: a script in `/usr/local/sbin`, one cron entry, optional
  `cronitor exec --no-stdout <key>` wrapper gated on `cronitor_enabled`; list new keys in
  the `4-cronitor.yaml` header.
- Commit messages and the CHANGELOG are public: describe the technical change only — no
  hostnames, site names, IPs, account details or incident narrative.
- Personal, machine-local instructions live in the gitignored `CLAUDE.local.md`.
- Run `betterleaks git .` (and `betterleaks dir inventory`) before pushing; mark a false
  positive with a trailing `betterleaks:allow` comment, never by weakening the scan.
