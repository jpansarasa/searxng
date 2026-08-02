# searxng

[SearXNG](https://docs.searxng.org/) privacy-respecting metasearch engine, run
as a Docker container under systemd.

## Layout

| File | Role |
| --- | --- |
| `install` | Idempotent installer. Run it for the first deploy and after every change. |
| `compose.yml` | Container definition. Bind-mounts `/tank/searxng/{config,cache}`. |
| `settings.yml.tmpl` | SearXNG config, as a *delta* on the image's own defaults. Two templated values. |
| `limiter.toml` | Bot-detection / rate-limit policy. Deployed but **inert** while `limiter: false`. |
| `searxng.service` | Owns the container. No pull on start, so boot is fast and network-order-independent. |
| `searxng-update.service` | Oneshot wrapper around `check-update`. No `[Install]` section — the timer resolves it by name. |
| `searxng-update.timer` | Daily update check, off the boot critical path. |
| `check-update` | Pulls the configured tag, compares to the running image, **stages** (never applies) and pushes an ntfy notification. |

Paths are pinned: the units hardcode `/opt/searxng` and `/tank/searxng`, and
`install` refuses to run from anywhere else.

## Where things live

This machine splits storage on purpose, and that rule decides every path below.

**ZFS carries services and their data.** The pool is portable: the drives can be
pulled into a new chassis, or `zfs send` to a new box. It survives a dead board,
a dead CPU, a dead OS drive.

**The OS is a separate NVMe and is disposable.** It is rebuilt from an ISO. It
is not where anything you would miss belongs.

So service configuration lives on the dataset, with the service it configures.

| File | Contents | Created by |
| --- | --- | --- |
| `/tank/searxng/secrets.env` | `SEARXNG_SECRET_KEY` (generated), `SEARXNG_BASE_URL` (**you**) | `install` seeds it; you fill in the base URL |
| `/tank/searxng/deploy.env` | `SEARXNG_BIND_ADDR` (**you**), `SEARXNG_TAG` (optional pin) | `install` seeds it; you fill in the address |

Both are on the dataset. Both are `root:root`, as is the dataset root, and
neither is ever sourced — both are parsed for known keys only.

### The one exception, and why

`install` renders a copy of `deploy.env` to **`/etc/searxng/deploy.env`**, and
that generated copy is the only one systemd ever reads. Do not edit it; it is
overwritten on every run.

The reason is measured rather than assumed: systemd reads `EnvironmentFile=` in
**PID 1 itself**. An `strace` of pid 1 shows it opening the file five times
during a single `systemctl restart searxng`. Since `tank` is `failmode=wait`, an
`EnvironmentFile=` pointing at the pool means a faulted pool can put PID 1 into
uninterruptible sleep — no `systemctl`, no logins, no clean shutdown — and the
`ExecStartPre` guards written to survive exactly that outage sit unreachable
behind it. Rendering to the root filesystem keeps the pool out of PID 1's path
while leaving the source of truth on the half that travels.

The render also **filters**: only `SEARXNG_BIND_ADDR` and `SEARXNG_TAG` cross
over. That is load-bearing, not tidiness. systemd applies `EnvironmentFile=`
*after* `Environment=`, so anything else appearing in the source file — most
pointedly `DOCKER_UID=0` — would otherwise put the container back to running as
root, the one state in which it can read `secrets.env` by ownership alone.
Verified by injecting exactly that and confirming it does not cross.

`install` generates what it can and refuses to invent what it cannot. It names
every unset value at once and exits, so a fresh machine takes exactly one
correction round rather than one per variable.

**The recovery set is this repo plus the dataset.** Move the drives to a new
chassis, install an OS, clone this repo, run `./install`. The only value that
does not survive that move intact is `SEARXNG_BIND_ADDR`, which is the address
of *this* host and is expected to change — `install` refuses to start without
it, and a wrong one makes compose fail loudly to bind rather than quietly
publishing somewhere else.

`secrets.env` is `0600 root:root` and sits at the **dataset root**, which
`compose.yml` does not bind-mount — only `config/` and `cache/` are — so the
container never sees it. `install` re-asserts that mode after its recursive
`chown`.

That split is load-bearing, not tidiness. When the dataset root *was* mounted,
the container's own uid owned the directory holding `secrets.env`, and directory
write permission is all it takes to unlink and replace a root-owned file — which
`install` then read as root on the next run. Keeping credentials and the render
staging path outside the mount is what closes that path.

### The secret

`settings.yml` needs a `secret_key`. It is templated out of this repo and kept
in `secrets.env`.

**This secret is disposable**, which is why `install` generates one with
`openssl rand -hex 32` rather than making you supply it, and never regenerates
it once present. Rotating it costs saved preference links, in-flight
image-proxy URLs, and the encrypted engine cache — all self-healing. (It is not
*only* preference URLs: it is also Flask's session key, the image-proxy URL
HMAC, and the engine cache password.)

## Backups

`tank/searxng` is opted into `zfs-auto-snapshot` at the monthly label
(`com.sun:auto-snapshot:monthly=true`, set locally on the dataset), and the
snapshots are picked up by the next sync to the backup host. `install` sets that
property and takes a first snapshot when it creates the dataset from scratch, so
a rebuild is protected immediately rather than at whenever-you-remember.

It does that **only on creation**, deliberately: on a restore the property
arrives with the received dataset, and a dataset opted out on purpose must not
be silently opted back in by a later run.

Note what a stale snapshot actually costs here, which is very little: the engine
caches are regenerable, `settings.yml` and `limiter.toml` are rendered from this
repo, and the secret is generated. The exposure is a hand edit to `secrets.env`
or `deploy.env` made since the last snapshot — the two values you supply.
Snapshot by hand after changing one of those if you care:

```bash
sudo zfs-auto-snapshot --label=monthly --keep=12 tank/searxng
```

`check-update` pushes an ntfy notification when an update is staged. It reads
the token from `/tank/ntfy/notify.env` — ntfy's dataset, since that is an ntfy
credential and is backed up there. It parses that file rather than sourcing it,
and says on stderr when it skips or fails, so a revoked token shows up in the
timer's journal instead of looking like "no updates".

## Recreating on a fresh machine

Prerequisites: Docker with the compose plugin, ZFS with a pool named `tank`,
`envsubst` (`gettext-base`), `openssl`, `findmnt` (`util-linux`), `curl`,
`python3`, systemd.

```bash
sudo git clone https://github.com/jpansarasa/searxng.git /opt/searxng

# First run creates the searxng user and dataset, generates a secret, seeds
# both env files, and then stops and tells you the two values it needs.
sudo /opt/searxng/install

sudo sed -i 's|^SEARXNG_BASE_URL=.*|SEARXNG_BASE_URL=https://searxng.example.com/|' /tank/searxng/secrets.env
sudo sed -i 's|^SEARXNG_BIND_ADDR=.*|SEARXNG_BIND_ADDR=10.0.0.5|'                   /tank/searxng/deploy.env

# Second run converges and does not exit until searxng actually answers.
sudo /opt/searxng/install
```

Do **not** create `/tank/searxng` by hand and write the env files there first —
ZFS mounts over a non-empty directory, so the dataset would hide them. `install`
creates the dataset first and refuses to write into a bare mountpoint.

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

`/run` is tmpfs, so that marker does not survive a reboot. Normally that is
right, because a reboot applies the staged image — but if the reboot did *not*
apply it, the marker is gone while the update is still staged.
`journalctl -u searxng-update.service` is the authoritative record.

`install` is safe to run repeatedly; it converges rather than erroring on things
that already exist. It finishes by polling `/healthz` until searxng actually
answers, so a zero exit means "serving", not merely "systemd accepted the job".
That distinction is the whole point: a bad value in `settings.yml.tmpl` renders
cleanly and starts cleanly, then crash-loops inside the container. Since editing
the template and re-running is the most common operation here, that is the
failure most likely to be introduced. It then asserts that every engine named in
`keep_only` actually came back in `/config`. That check has to be positive
rather than a log grep: `keep_only` is a pure filter, so a name upstream renames
or retires matches nothing and is dropped **silently**, leaving results quietly
thinner while everything stays green. That is exactly how the `reddit` and
`presearch` entries stayed broken for months.

### Pinning a bad release

The normal state is `latest`, and the normal response to a new release is to
take it. Pinning is the exception, kept to a file that is not in git:

```bash
# Which version am I on? (the container logs it at startup, and the unit runs
# attached, so the journal has the whole history)
docker logs searxng | head -1
journalctl -u searxng.service | grep -oE 'SearXNG [0-9.]+-[0-9a-f]+' | uniq

# Pin, then apply
sudo sed -i 's|^SEARXNG_TAG=.*|SEARXNG_TAG=2026.7.1-abc1234|' /tank/searxng/deploy.env
sudo /opt/searxng/install   # re-renders /etc/searxng/deploy.env, then restarts

# Un-pin once upstream supersedes the bad release
sudo sed -i 's|^SEARXNG_TAG=.*|SEARXNG_TAG=latest|' /tank/searxng/deploy.env
sudo /opt/searxng/install
```

Both units set `SEARXNG_TAG=latest` and then read the rendered `/etc/searxng/deploy.env`,
which systemd applies **after** `Environment=` and therefore wins. While pinned,
`check-update` tracks the **pinned** tag — it will not claim an update is staged
that a restart would not actually apply. It still watches `latest` separately
and pushes a one-line "the pin can likely be retired" notice once upstream moves
past the release you pinned away from. The pin lives on the dataset, so it survives a
chassis move; it is deliberately not in git, so it never collides with a pull.

## Configuration model

`settings.yml.tmpl` is a **delta on the image's own `settings.yml`**, via
`use_default_settings: true` — not a copy of it. Any key it does not mention
keeps the upstream default, so an upstream rename or a new setting is a
non-event.

It replaced a 2421-line vendored fork, and the reason is worth recording: that
fork had silently drifted two schema generations behind the running image. It
declared `redis:` where upstream had moved to `valkey:`, carried a commented-out
`enabled_plugins:` block the code no longer reads at all (so editing it to
disable a plugin did nothing, with no error), and named five engine modules the
image had already deleted — which logged a traceback on every start while the
unit stayed green. Four stale half-finished renders on the dataset were the
fossil record of reconciliations that had stalled.

Engines are a **whitelist**, via `use_default_settings.engines.keep_only`. That
is the one property genuinely worth forking for: upstream ships hundreds of
engines and adds more every release, and without a whitelist an update would
silently start fanning queries out to third parties nobody chose. Adding an
engine now means adding its name to that list.

## Notes

- The container's effective capability set is **empty**. `cap_drop: ALL` with no
  `cap_add`, running as uid/gid 2100, plus `no-new-privileges`. The previous
  `cap_add` list was inert anyway — a non-root euid with no file capabilities
  never raises anything into the effective set — but it named `DAC_OVERRIDE`,
  the one capability that bypasses file permission checks, which is a poor thing
  to have written down next to a root-only secret.
- The published port binds **one** address (`SEARXNG_BIND_ADDR`), not `0.0.0.0`.
  This instance has no authentication and `limiter: false`, so every interface it
  is published on is an open search proxy — one that issues arbitrary outbound
  HTTP attributed to this host's IP, with getting that IP blocked by the search
  engines as the practical consequence. Note Docker publishes ports via DNAT
  ahead of the INPUT chain, so a host firewall does **not** cover this without an
  explicit `DOCKER-USER` rule.
- **Expected noise at startup.** The container prints
  `"/etc/searxng" directory is not owned by "searxng:searxng" … Got "UNKNOWN:UNKNOWN"`
  for both mounted directories. That is cosmetic and correct: the tree is owned
  by host uid 2100, and the image has no name for that uid (its own `searxng`
  account is 977), so the entrypoint cannot resolve it and says so. Both
  directories are readable and writable by the running process.
  `FORCE_OWNERSHIP=false` is what stops it from "fixing" this by chowning the
  dataset to 977 — which is the failure it looks like it is warning about.
- Also expected: `X-Forwarded-For nor X-Real-IP header is set!` at ERROR level
  for any request that does not come through the reverse proxy — i.e. a direct
  `curl` from the host, not normal traffic.
- `searxng.service` deliberately does **not** pull on start. Pulling is the
  update timer's job; a start that waits on the network makes boot order fragile.
- `compose.yml` deliberately sets **no** restart policy. systemd is the only
  supervisor; see the comment in that file for what two supervisors cost.

### Waiting, not skipping

Every prerequisite that can be *late* — the ZFS dataset, the rendered config,
dockerd — is checked in a way that **fails and retries**, never one that skips.
That is a deliberate choice, and the reasoning is worth keeping:

`Condition*` directives (and `Assert*`, and a failed `Requires=`) abort the
start *job*. The unit never enters start, so `Restart=` is never armed, the unit
sits at `inactive (dead)` with `Result=success`, and **nothing ever
re-evaluates**. Repairing the prerequisite does not bring it back — only a new
start job does. This unit previously gated the dataset on
`ConditionPathIsDirectory=/tank/searxng`, which is worse still: a **bare
mountpoint satisfies it**, so it passed in exactly the fail-open case it looked
like it was guarding.

So the dataset and config checks are `ExecStartPre=` (a real start failure,
which `Restart=always` retries every 30s, re-running the check each cycle),
docker is `Wants=` rather than `Requires=`, and `StartLimitIntervalSec=0` means
it never gives up. A late pool import, a hand `zfs mount`, or a dockerd that
comes back all recover the service unattended within 30 seconds.

The cost, which you need to know when looking for a sick service: a unit that
retries forever never reaches `failed`, so **`systemctl --failed` stays empty
while searxng is down**. Look instead at:

```bash
systemctl is-active searxng            # "activating" (not "active") while stuck
journalctl -p err -u searxng -n 20     # the guards log why, at priority err, every cycle
```

## Removing this service

Order matters. Removing the tree first strands the container: `ExecStop` runs
`docker compose --file /opt/searxng/compose.yml down`, which cannot work once
that file is gone.

```bash
sudo systemctl disable --now searxng.service searxng-update.timer
sudo systemctl stop searxng-update.service
sudo docker compose --file /opt/searxng/compose.yml --project-name searxng down
sudo docker volume rm searxng_config searxng_cache
sudo rm -f /etc/systemd/system/searxng-update.service
sudo systemctl daemon-reload
sudo rm -rf /opt/searxng /etc/searxng
# The dataset holds the secret and the two deployment values. Deliberate step:
# sudo zfs destroy -r tank/searxng
```
