Optional local secret files, read only when present (keep this directory out of Git):

- `acme-sh.conf` — DNS-plugin credentials for `6-acme-sh.yaml` (see `acme-sh.conf.example`).
- `gcloud_service_account.json` — service-account key for `standalone_playbooks/1-install-gcloud-cli.yaml`.

Without them, the playbooks keep what the server already has.
