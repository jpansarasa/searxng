# searxng

[SearXNG](https://docs.searxng.org/) privacy-respecting metasearch engine, run
as a Docker container under systemd.

## Layout

| File | Role |
| --- | --- |
| `install` | Idempotent installer. Run it for the first deploy and after every change. |
| `compose.yml` | Container definition. Bind-mounts `/tank/searxng` at `/etc/searxng`. |
| `settings.yml.tmpl` | SearXNG config, with the one secret as `${SEARXNG_SECRET_KEY}`. |
| `limiter.toml` | Bot-detection / rate-limit config. No secrets. |
| `searxng.service` | Owns the container. No pull on start, so boot is fast and network-order-independent. |
| `searxng-update.service` | Oneshot wrapper around `check-update`. No `[Install]` section — the timer resolves it by name. |
| `searxng-update.timer` | Daily update check, off the boot critical path. |
| `check-update` | Pulls `:latest`, compares to the running image, **stages** (never applies) and pushes an ntfy notification. |

Paths are pinned: the units hardcode `/opt/searxng` and `/tank/searxng`, and
`install` refuses to run from anywhere else.

## The secret

`settings.yml` needs a `secret_key`. It is templated out of this repo and kept
in `/tank/searxng/secrets.env` (root-only, `0600`).

**This secret is disposable** — it only signs preference URLs and limiter
tokens. So `install` generates one with
`openssl rand -hex 32` if the file is missing, and never regenerates it once
present. There is no manual step on a rebuild, and nothing to back up.

`install` re-asserts `0600 root:root` on it after the recursive `chown`, which
would otherwise hand it to the `searxng` user.

## Backups

**`tank/searxng` is not replicated, deliberately.** Replication to the backup
host is opt-in per dataset (`com.sun:auto-snapshot:*` is false on the pool),
and searxng is not opted in. That is fine here: everything on the dataset is
either in this repo (`settings.yml`, `limiter.toml`) or regenerable (the
secret, the container's own caches). A rebuild is a clone plus `./install`.

`check-update` does push an ntfy notification when an update is staged. It
reads the token from `/tank/ntfy/notify.env` — ntfy's dataset, since that is
an ntfy credential and is backed up there. The push silently no-ops if that
file is unreadable.

## Recreating on a fresh machine

Prerequisites: Docker with the compose plugin, ZFS with a pool named `tank`,
`envsubst` (`gettext-base`), `openssl`, systemd.

```bash
sudo git clone https://github.com/jpansarasa/searxng.git /opt/searxng

# Creates the searxng user and dataset, generates a secret, renders
# settings.yml, registers the units, and starts the container.
sudo /opt/searxng/install
```

That is the whole procedure — there is no secret to supply by hand.

## Day-to-day

```bash
# Change config: edit settings.yml.tmpl, then re-run. Restarts searxng.
sudo /opt/searxng/install

# Check for an image update by hand (stages, does not apply)
sudo systemctl start searxng-update.service
journalctl -u searxng-update.service -n 20

# Apply a staged update
sudo systemctl restart searxng.service

# Is an update waiting?
cat /run/searxng-update-available
```

`install` is safe to run repeatedly; it converges rather than erroring on
things that already exist. The only side effect of a no-change run is the
service restart in the final step.

## Notes

- Upstream ships a large `settings.yml`; `settings.yml.tmpl` is that file with local engine choices applied and the secret templated out. Reconcile it against upstream when a major version lands — `check-update` says as much when it stages an update.
- The container is unprivileged: `cap_drop: ALL`, then only `CHOWN`/`SETGID`/`SETUID`/`DAC_OVERRIDE` back, running as uid/gid 2100.
- `searxng.service` deliberately does **not** pull on start. Pulling is the update timer's job; a start that waits on the network makes boot order fragile.
- `limiter: false` and `public_instance: false` in settings — this instance is private, behind HAProxy, with RFC1918 ranges in the limiter pass list.
