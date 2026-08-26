# Release Notes (2026-08-26)

GCE VM deployment scripts gained transparent Identity-Aware Proxy (IAP) tunneling support.

## 🛠️ Tooling
* **[Tooling/starter-hub]:** Add transparent IAP tunneling to GCE VM helper scripts. Wraps all `gcloud compute ssh` and `gcloud compute scp` calls inside a local `gcloud` helper function in `hub-config.sh` to automatically append `--tunnel-through-iap` when `SCION_GCE_SSH_TUNNEL=true` is set. This enables seamless remote provisioning, code setup, and certificate updates on corporate networks where direct port 22 access to external IPs is restricted.
