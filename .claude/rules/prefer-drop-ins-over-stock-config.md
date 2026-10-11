# Prefer drop-ins; never overwrite package-shipped config

When configuring a system service, add a **higher-numbered drop-in** in the
relevant `.d` directory containing ONLY the settings that differ from the
distro default. Do not overwrite a file that a package ships.

Applies to (non-exhaustive): `apt.conf.d/` (e.g. `50unattended-upgrades`),
`sysctl.d/`, `sshd_config.d/`, `mysql/mariadb.conf.d/` (e.g. `50-server.cnf`),
PHP FPM `conf.d/` (vs `php.ini`), `sudoers.d/`, `systemd` unit `*.d/` overrides.

**Why:** overwriting a stock file (a) silently discards every other default it
carries, pinning unrelated settings to whatever stale values you copied; (b)
triggers dpkg conffile prompts on upgrade because you modified a package
conffile; (c) reusing the package's own filename slot (e.g. writing your own
`50unattended-upgrades`) collides with the package.

**How to apply:**
- Before writing a drop-in, confirm the stock file's current contents (install
  the package on a vanilla box if needed) so your file carries only the true
  diff. Often the diff is a single line.
- Pick an unused higher number than the package's (`52…` over the shipped
  `50…`) so it's read later and, for list-type options, appends.
- Know whether the option is list-type (accumulates across files, e.g. APT
  `Origins-Pattern`) or scalar (last write wins, e.g. most `.ini`/`.cnf` keys).
- Only fall back to editing the stock file if the setting genuinely cannot be
  expressed as a drop-in — and say so explicitly.

**When a drop-in is structurally impossible — use a managed edit, not a clobber.**
Some settings can't live in a drop-in because of *ordering*: e.g. an nginx
`log_format` must be defined at http level before the http-level `access_log`
that uses it, and both precede the `include conf.d/*.conf` line — so no conf.d
file can change them. In these cases edit the stock file **surgically and
idempotently** with `ansible.builtin.blockinfile` (marked `# BEGIN/END ANSIBLE
MANAGED` block) or `lineinfile` (a format-agnostic regexp so re-runs don't
duplicate), never a whole-file `copy`. This preserves every other stock default,
stays idempotent, and makes the change auditable. Prefer this over scattering the
same override across many generated files. Validate afterwards (e.g. `nginx -t`)
before reload, since some services reload successfully on a bad config.
