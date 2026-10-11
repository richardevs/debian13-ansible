# debian13-ansible

These playbooks install and configure an LNMP server (nginx, MariaDB and PHP) on
**Debian 13 (trixie)**. This repository replaces `debian12-ansible` and uses stricter
permission boundaries. It is a proof of concept. Read the playbooks before you run
them on a server with important data. Some optional steps are specific to one
environment, for example the backup and monitoring playbooks.

## Differences from debian12-ansible

| Area | debian12-ansible | debian13-ansible |
|---|---|---|
| OS and software | Debian 12, PHP 8.2, MariaDB 11.4 (external repository) | Debian 13, PHP 8.4, MariaDB 11.8 LTS (Debian package) |
| Web root | `/home/<user>/public_html` | `/srv/www/<site>` (the home directory is for login only) |
| php-fpm connection | TCP `127.0.0.1:2<uid>` | Unix socket `/run/php/<site>-fpm.sock` (mode 0600, nginx only) |
| nginx read access | nginx is a member of each site group | POSIX **default ACLs** (`u:nginx:rX`) |
| Web permissions | 750, the group can read | **750 for directories, 640 for files, 440 for wp-config** (no world read, no read between sites) |
| TLS | Mozilla v5.7, dhparam | **TLSRef v6.0** (post-quantum `X25519MLKEM768`, ECDHE only, no dhparam) |
| Firewall and bans | firewalld, then nftables, then ipset (optional) | **nftables with the fail2ban nftables action** (always used) |
| sshd | Only LogLevel is set | `PermitRootLogin prohibit-password`, `PasswordAuthentication no` |
| Site template | `if (-f) rewrite`, fixed HSTS values | `try_files`, structured `hsts:` settings, and a per-site `include` file for local changes |

## Design principles

- **The control machine holds no secrets.** A secret stays where the server uses it: on
  the server, or in a cloud secret manager. The repository and the inventory contain
  only configuration and references to secrets.
- **Do not store a secret when you can avoid it.** This is better than a secure copy.
  For example, the server makes each database password and keeps it. Ansible never
  stores the password on the control machine.
- **A playbook must run again safely from a different device.** It must not need a
  local copy of a secret for this.

## Permission model

- `/home/<site>` (0750) is the SSH and SFTP login directory only. It contains **no web content**.
- `/srv/www/<site>` (0750, files 0640, wp-config 0440) has the site user as owner.
  php-fpm runs **as the site user**. Thus WordPress updates from the dashboard operate
  correctly, because the updates run as the UID that owns the files. The
  [official WordPress hardening guide](https://developer.wordpress.org/advanced-administration/security/hardening/)
  recommends this method.
- nginx reads the files through an **ACL**, not through world bits. Thus one site
  cannot read the `wp-config.php` of a different site. Because of the **default ACL**,
  new files (from rsync or WordPress updates) automatically get nginx read access.
- Additional controls:
  - nginx blocks requests for `*.php` files under all `wp-content/`, `uploads/` and `files/` directories.
  - nginx blocks requests for `xmlrpc.php` and `wp-config.php`.
  - nginx sets the PHP `open_basedir` value for each site.
- **`wp-config.php` has the mode `440` (read-only)**. The file contains the database
  credentials, and WordPress does not write to it during operation. The read-only mode
  stops simple attacks that try to overwrite `wp-config.php`. The owner can change the
  file: run `chmod u+w wp-config.php`, edit the file, then run `chmod 440`. The database
  migration uses these steps to write the new password.
- Ansible sets the password of a database user only when it makes the user. The server
  makes the password and keeps it in `/root/.db/<site>.psw` (mode 0600). The
  `wp-config.php` file of each site contains the password that WordPress uses.
- `5a-wordpress.yaml` sets `FS_CHMOD_FILE 0640` and `FS_CHMOD_DIR 0750` in `wp-config.php`.
  These values control the mode of
  files that WordPress writes (uploads, core and plugin updates). Without these values,
  WordPress gives files the mode `0644`, and all users can read them. This bypasses
  the ACL.

> **More information:** [ACL_DESIGN.md](ACL_DESIGN.md) gives the full model. It tells
> how to read `ls` and `getfacl` output on this tree (the mask). It also gives the three
> actors that can break the model, and the failure modes. Read it before you change
> ACLs on a live server.

## Usage

1. Install ansible-core 2.18 or later on the control node (for example `brew install ansible`
   or `pipx install ansible-core`). Then install the collections:
   `ansible-galaxy collection install -r requirements.yml`.
   `requirements.yml` gives the tested version ranges.
2. Make your inventory. Run `cp -r inventory.example inventory`, or clone your own
   private inventory repository into `inventory/`. Git ignores this directory.
3. Edit `inventory/hosts.yml`. Add the IP address of the server.
4. Edit the files in `inventory/group_vars/webservers/`. `users.yml` gives the schema
   for the sites.
5. Run `ansible-playbook main.yaml -v`.
6. The last step of `main.yaml` (`7-validate.yaml`) examines the result for each site.
   It does not change the server. To run only the checks, run
   `ansible-playbook 7-validate.yaml`.

### Inventory

The repository contains no host data. `ansible.cfg` reads `inventory/hosts.yml`, and
Ansible loads the files in `inventory/group_vars/` with it. Keep your inventory in a
separate private repository, and clone it into `inventory/` on each device.

`roles/repo_defaults/defaults/main.yml` gives the default values. A value in the
inventory always overrides a default. Optional secret files go in
`inventory/secrets/` (refer to `inventory.example/secrets/README.md`).

### Per-site settings (`inventory/group_vars/webservers/users.yml`)

```yaml
users:
  - name: example          # login user + pool + default db/root identity
    domain: [example.com, www.example.com]
    php_enabled: true       # php-fpm pool + WP nginx block + a database
    tls: true               # acme.sh issues a cert for this site
    db_name: example        # optional, default = name
    root: /srv/www/example  # optional, default = /srv/www/<name>
    hsts: { preload: true } # optional; max_age/include_subdomains/preload
```

To add an nginx setting for one site that the template does not have, edit
`/etc/nginx/snippets/<name>.extra.conf` on the server. The site configuration includes
this file. Ansible **never overwrites** it.

### Drift check (dry run)

This command shows the differences between the live server and the repository. It
does not change the server. It runs `main.yaml` in check mode with diffs, and it hides
the tasks with no change and the skipped tasks. Thus only the differences show:

```bash
ANSIBLE_DISPLAY_OK_HOSTS=false ANSIBLE_DISPLAY_SKIPPED_HOSTS=false ANSIBLE_DIFF_ALWAYS=true \
  ansible-playbook main.yaml --check --diff
```

The environment variables apply only to this command. They do not change
`ansible.cfg`, and they have no effect on normal runs.

The check is reliable for **file differences**: nginx configuration, `jail.local`,
pool files, php.ini and ACLs. The check **does not examine `command` and `shell`
tasks** (in `1-install`, `4-nginx`, `4a-fail2ban` and `6-acme-sh`). `--check` skips
these tasks. Thus the output does not show the acme, wp-cli and `nginx -t` steps.

## Settings to change before use

- **Certificates.** `6-acme-sh` gets Let's Encrypt certificates with HTTP-01 (webroot)
  by default. Each domain must point to the server before you run it. The playbook
  asks before it issues certificates.
- **DNS-01 (optional).** To get certificates before the DNS points to the server, set
  `acme_challenge: dns` for the inventory or for one site. Also set `acme_dns_plugin`
  (an [acme.sh DNS API](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) name)
  and `acme_dns_cred_keys`. `inventory.example/group_vars/webservers/main.yml` shows
  an example.
- **DNS plugin credentials.** These apply only to DNS-01. `inventory/secrets/acme-sh.conf`
  is optional. When the file exists, `6-acme-sh` writes it to the server. Use the file
  to set or change the credentials. When the file does not exist and the server has the
  credentials, the playbook keeps them. When neither has the credentials, the playbook
  asks for each key.
- **Google Cloud key.** `inventory/secrets/gcloud_service_account.json` is optional in the same
  way. When the file does not exist and the server has an active account,
  `1-install-gcloud-cli` keeps that account.
- **WordPress settings.** `5a-wordpress.yaml` sets the `wp-config.php` constants in
  `wp_config_constants`, installs and activates wp-fail2ban, and turns on plugin
  auto-updates. It skips a site that has no `wp-config.php`, and it changes only values that
  differ. To keep auto-updates off for a plugin, add its slug to
  `wp_plugin_auto_updates_exclude`. To turn off the plugin auto-update step, set
  `wp_plugin_auto_updates: false`.
- **WordPress fail2ban filters.** The playbooks download these filters from the SVN
  repository of the plugin. `wpf2b_ref` in `roles/repo_defaults` sets the version
  (default `tags/5.4.2`). Each filter has a version stamp, and it becomes **obsolete**
  when the plugin updates. When fail2ban shows a warning about obsolete filters, set
  `wpf2b_ref` to the installed plugin version, or to `trunk`. `5a-wordpress` also shows a
  notice when the installed plugin version differs from `wpf2b_ref`. The jails read the
  systemd journal (`journalmatch = SYSLOG_IDENTIFIER=wordpress`). This is the standard
  method on Debian 13.

## Standalone playbooks

The playbooks in `standalone_playbooks/` are optional. Run each playbook separately.

Migration from the old server:
- `5-rsync-migrate.yaml` copies the site files into `/srv/www/<name>` with rsync
  (directories 750, files 640). It sets the owner, applies the nginx ACL again, and
  sets the strict mode on `wp-config.php`.
- `6-dbdump-migrate.yaml` exports the databases with mysqldump on the source server and
  imports them on the new server. It then sets the database credentials with
  `wp config set`. After the migration, run `main.yaml` again: `5a-wordpress.yaml`
  applies the WordPress settings to the migrated sites.

Backups:
- `1-install-gcloud-cli.yaml`, `2-install-ossutil.yaml` and `3-backup-init.yaml` make
  encrypted database dumps and copy them and `/srv` to GCS and Alibaba OSS.

Monitoring:
- `4-cronitor.yaml` installs CronitorCLI. On the server, run
  `cronitor auth login --no-browser`. Then set `cronitor_enabled: true` in
  the inventory. Run the standalone playbooks 3 and 7 and the core playbook
  `6-acme-sh` again. The cron entries then report to Cronitor.

Malware scan:
- `7-malware-scan.yaml` installs `/usr/local/sbin/malware-scan`. The scan only
  reports. It does not change or quarantine files.
- The scan compares WordPress core and wordpress.org plugins with the published
  checksums. It reports added PHP files and changed PHP or JS files.
- It reports PHP files in the core and root directories that WordPress does not
  supply, and PHP files in `wp-content/uploads`.
- [Linux Malware Detect](https://github.com/rfxn/linux-malware-detect) (maldet) scans
  `/srv/www` with its native engine and YARA. It does not use ClamAV or the inotify
  monitor. It runs on one CPU only.
- Each day, maldet scans the files that changed in the last 2 days. On the first day
  of each month (`--full`), it scans all files.
- When the scan finds a problem, it exits with a non-zero code, and Cronitor sends an
  alert. The details are in `/var/log/malware-scan.log` and in the output of
  `maldet -e`.
- To skip a file that you examined, add its `sha256sum` output line to
  `malware_scan_allow` in the inventory. The scan skips the file while its content
  stays the same.
- Premium plugins and themes have no published checksums. Only maldet scans them.
