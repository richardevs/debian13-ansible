# Verify against current sources; never blind-copy install methods or configs

Do not reuse an existing installation method or a copied "recommended" config
just because it worked before. Before porting anything third-party, check the
authoritative current source and update to present practice.

**Install methods** (apt repos, signing keys, CLIs like gcloud / ossutil,
runtimes): check the vendor's current docs and latest stable version. Verify:
- correct keyring handling (dearmored `signed-by=` keyrings, not legacy
  `apt-key` or armored bytes in a `.gpg` path),
- current package/binary names (they get renamed, e.g. `ossutil64` → `ossutil`),
- current auth model — **prefer instance/workload roles over static keys**
  (e.g. ECS RAM role instead of an AccessKey on disk),
- breaking changes (new mandatory params, new signature versions).

**Security / hardening configs** (TLS, `php.ini`, fail2ban, DB): cross-check the
current authoritative source — OWASP Cheat Sheets, Mozilla/TLSRef — and adopt
only what fits the actual application. Do not apply a generic hardening item
that breaks the app (e.g. disabling PHP functions WordPress needs for uploads
and self-updates; COOP/COEP headers that break embeds; DML-only DB grants when
the app performs schema changes). State which items were rejected and why.

**Why:** copy-forward optimizes for "don't rewrite what works" but silently
imports dead assumptions — a broken keyring, a renamed binary, a missing
mandatory `region`, a config tuned for a context that no longer exists. The
config often still parses and the playbook runs green while doing the wrong
thing (or nothing). Verifying against current docs turns a latent outage into a
one-line update.
