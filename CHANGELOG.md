# Changelog

Notable changes to HomelabHero, newest first.

The version lives in `HH_VERSION` at the top of `bin/hh` and is printed by
`hh version`. Versioning is semantic: MAJOR for a breaking change to an existing
command, MINOR for new functionality, PATCH for fixes.

To move between versions, re-run the installer (see README). `hh update` pulls
the latest and needs no manual migration for any release below.

## Linking to a release

Every version heading carries a stable anchor of the form
`v<major>-<minor>-<patch>`, so a GitHub release body, an issue, or a commit can
link straight to the notes instead of duplicating them:

    https://github.com/serversathome/homelabhero/blob/main/CHANGELOG.md#v1-1-0

The anchor is written explicitly rather than relying on the heading text,
because GitHub strips the dots out of `## 1.1.0` and generates `#110`, which is
easy to get wrong and changes if a date is added to the heading.

When adding a version, keep the three in sync: the anchor (`v1-1-0`), the git
tag (`v1.1.0`), and `HH_VERSION` in `bin/hh` (`1.1.0`).

<a id="v1-3-0"></a>

## 1.3.0 (2026-08-18)

Firewalla joins UniFi as a network appliance HomelabHero can read.

### Added

- `hh firewalla <op> [alias]` reads a Firewalla box through the Firewalla MSP
  API, with a new `hh-firewalla` broker built on the same shape as `hh-unifi`:
  it runs as the vault user, reads a token the agent user cannot read, passes it
  to curl through a config file on stdin rather than argv, and logs every call
  to the broker audit log.

  Ops: `summary`, `ping`, `info`, `boxes`, `devices`, `device`, `alarms`,
  `flows`, `rules`, `targetlists`, `stats`, and `get` for any other GET
  under `/v2/`. A registered box appears in `hh list`, `hh test`, `hh doctor`,
  `hh overview`, and `hh inventory` like any other host.

  This covers ground UniFi cannot: security alarms (port scans, abnormal
  uploads, new devices), per-device traffic flows with the destination and
  whether it was blocked, and the firewall rules themselves including whether a
  rule is active or quietly paused.

- `hh add-firewalla` registers a box from an admin shell. Like `hh add-unifi`,
  it is operator-only and shell-only, because an API token is a secret being
  typed and a secret typed into an LLM session has already been seen by the LLM.

- A `firewalla-ops` skill and `capabilities/firewalla.md`, plus Firewalla
  sections in the ops brain, `infra/network.md`, and the `network-diag` skill.

- `hh run` on a Firewalla now stops with a pointer to `hh firewalla`, the way it
  already did for a UniFi console. Without it, an API-token host fell through to
  "unknown AUTH type: apitoken" from the bottom of the SSH broker.

`hh scan` deliberately does not learn to find a Firewalla. Registration needs an
MSP portal domain rather than a LAN address, so there is nothing a subnet sweep
could hand to `hh add-firewalla`; the box still shows up in a scan as a gateway.

### Read-only, and why it is enforced differently here

The broker issues HTTP GET and has no code path that can POST, PUT, PATCH, or
DELETE, and every write-shaped op name (`pause`, `resume`, `block`, `delete`,
`create`, `mute`, ...) is refused by name rather than left to fail obscurely.

This matters more than it does for UniFi. The UniFi integration has two
independent locks: a GET-only broker AND an API key minted under a View Only
admin that the console itself refuses writes from. **Firewalla has no View Only
token tier** - an MSP personal access token carries every permission the account
has, and the API really does expose rule creation, deletion, and pausing. So
there is no second lock to fall back on, and the broker's construction is the
whole guarantee. The skill and the ops brain say so explicitly rather than
letting the UniFi wording imply a safety net that is not there.

### Notes on the integration

- It uses the **MSP cloud API** (`https://<yourname>.firewalla.net/v2/...`), not
  a LAN connection. Firewalla publishes no supported local REST API - the on-box
  service on port 8833 is internal, undocumented, and needs an ETP token from
  the app's Additional Pairing flow - so HomelabHero does not build against it.
  The registry entry records the backend as `API=msp` so a local backend can be
  added later without changing the format; `API=local` currently fails with an
  explanation rather than guessing at an undocumented protocol.
- API access requires an MSP account with an active plan.
- There is no TLS pin here, unlike UniFi. The MSP is a public host with an
  ordinary publicly-trusted certificate that Firewalla rotates on its own
  schedule; pinning a certificate we do not control would break on renewal and
  train people to re-accept pins on warning, which is worse than not pinning.
  Normal CA validation applies and is always on.

### Changed

- The CLI now tells API-reached hosts apart by `PLATFORM` rather than by
  `AUTH=apikey`. That test worked only while UniFi was the sole API platform:
  with a second one, every API host looked like a UniFi console, and `hh test`,
  `hh overview`, `hh inventory`, `hh doctor`, and bash completion would all have
  handed a Firewalla to the UniFi broker. `PLATFORM` has been written into every
  registry entry from the beginning, so hosts registered by older versions are
  read correctly with no migration.

### Fixed

- Errors from the new broker report once, with the real reason. `die()` runs
  `exit`, which ends only the subshell it is in, and bash's `errexit` does not
  propagate out of a function that is itself running inside a command
  substitution - so a failed box lookup nested in an argument let the caller
  carry on with an empty value and fail again later with a worse message. Box
  resolution is hoisted into plain assignments with explicit propagation, so an
  MSP account with no boxes (or with several) now says exactly that, once.

<a id="v1-2-1"></a>

## 1.2.1 (2026-08-15)

Installs on TrueNAS 26 that died on the first line of setup now work.

### Fixed

- The installer no longer aborts when `no_new_privs` is set on the attaching
  shell alone. The preflight read the flag from `/proc/self/status` and stopped
  there, so a TrueNAS web shell or `lxc-attach` session - both of which set the
  flag per session, on containers whose PID 1 is clean - failed immediately
  after `Starting setup...`, with advice to recreate the container as
  privileged. That advice could not have worked: the flag was never a property
  of the container, and on TrueNAS 26 the ID Map Type is fixed at creation, so
  following it cost a full rebuild and changed nothing.

  The flag is now read from `/proc/1/status` and `/proc/self/status`
  separately, because the two mean different things. Set on PID 1, the whole
  container is constrained - including the command center service, which PID 1
  forks - and the install still stops with the privileged-recreate or
  run-in-a-VM advice, which applies there and only there. Set on the shell
  alone, the install continues when it is already root, since nothing has to
  escalate, and stops for a non-root user with the fix that actually works:
  get a shell from PID 1 with `systemd-run --pty --quiet /bin/bash`, or SSH in
  rather than attaching from the host UI. The flag is one-way and cannot be
  cleared, so such a shell has to be replaced, not repaired. If
  `/proc/1/status` cannot be read, the check says nothing rather than guessing.

### Changed

- The README names the web UI's port. It described the installer as finishing
  with "a browser link" without ever saying `3001`, so anyone who scrolled past
  the final banner had nothing to go back to. The address, the `PORT=` override
  in `/etc/homelabhero/cloudcli.env`, and the first-visit steps are now written
  out in the install section.

<a id="v1-2-0"></a>

## 1.2.0 (2026-08-05)

Your router joins the homelab. HomelabHero can now discover a UniFi console
during setup and read it over its local API, so the gateway stops being the one
piece of the network Claude has to guess about.

It is **read-only** and cannot be talked out of it. Reading the router is
enormously useful; letting an agent write to it is not, because the router is
the one device whose failure takes away the access you would need to fix it.

### Added

- `hh scan` looks for your router. Anything at the default gateway or at the
  `.1` of the subnet (`192.168.1.1`, `10.99.0.1`) is fingerprinted, and a UniFi
  console is identified by name and version through its unauthenticated status
  endpoint, falling back to its TLS certificate. The results table gained a
  `ROLE` column, and a found router is called out by name, since it is the one
  device nobody thinks of as "a server to add".
- `hh add-unifi` registers a UniFi console with an API key. Offered directly
  from `hh scan --add` and from step 10 of the installer, so the normal path is
  simply to accept it during setup. Operator- and shell-only, like password
  auth: an API key is a secret being typed, and a secret typed into a chat
  session has already been seen by the LLM.
- `hh unifi <op> [alias]` reads the console: `summary`, `health`, `devices`,
  `clients`, `networks`, `info`, `sites`, `device`, `stats`, `ping`, and a
  read-only `get` escape hatch for any other path under `/proxy/network/`
  (`{site}` and `{siteName}` are substituted for you). The alias may be omitted
  when only one console is registered, which is the usual case.
- `hh overview` and `hh inventory` include the console: WAN and internet health,
  LAN and WiFi status, adopted devices with firmware state, and client counts.
  `hh doctor` and `hh test` check it too.
- A `unifi-ops` skill and a `capabilities/unifi.md` catalog, plus gateway-first
  guidance in `network-diag` and `infra/network.md`. An offline switch or access
  point explains every host behind it, so checking the fabric first saves
  troubleshooting those hosts one at a time.
- `hh repin <alias>` re-pins a console's TLS public key after you legitimately
  change its certificate.

### Security

- **Read-only in three independent places.** The `hh-unifi` broker issues HTTP
  `GET` and has no code path that can `POST`, `PUT`, `PATCH`, or `DELETE`, so no
  prompt and no jailbreak can reconfigure the network through it. Registration
  tells you to mint the key under a **View Only** UniFi admin, so the console
  refuses writes on its own. The ops brain and the skill state the rule plainly
  and redirect to "tell the user what to change, then read it back to confirm".
- The API key never reaches the agent. It lives in the vault (mode 600,
  `hhvault`), and `hh-unifi` hands it to curl through a config file on stdin
  rather than a command-line header, so it never appears in `/proc`, which is
  world readable and would otherwise expose it to the very user the vault exists
  to keep it from.
- The console's TLS public key is pinned on first contact (trust on first use,
  like SSH's `accept-new`) and verified on every later call. A UniFi console
  ships a self-signed certificate, so ordinary CA validation cannot apply; a
  changed certificate now fails closed with instructions rather than quietly
  trusting a new identity.
- `hh-connect` refuses an API-key host with a pointer to `hh unifi`, and
  `hh-provision` refuses to register one at all, so there is no path from the
  chat to a stored credential.

### Fixed

- Registry values may now contain `=`. Both brokers split on the first `=` by
  hand instead of using `IFS='='`, which silently dropped a trailing `=` and so
  truncated the base64 padding of a pinned key into one that could never match.
- `hh scan` no longer aborts when the default route cannot be read. With
  `pipefail`, a missing `ip` or an LXC with no default route turned a lookup
  that is allowed to fail into a fatal one, and the scan produced nothing at
  all.

<a id="v1-1-1"></a>

## 1.1.1 (2026-07-27)

### Changed

- TrueNAS apps are formatted as a name/state/version table in `hh inventory`,
  the same treatment VMs and LXCs got in 1.1.0. They were still printing as raw
  JSON truncated at 3000 characters, so a box with a dozen apps produced an
  unreadable blob, and an empty app list printed a bare `[]`.

### Fixed

- An unreadable middleware response is now reported instead of silently
  producing an empty section. Previously any parse failure was swallowed, so a
  changed API shape would look identical to "nothing is running", which is the
  exact blind spot this reporting exists to remove. A namespace the running
  version does not have is still quiet, since that is normal and expected rather
  than a fault.

<a id="v1-1-0"></a>

## 1.1.0 (2026-07-27)

TrueNAS VMs and LXC containers now show up in the inventory, at parity with how
Proxmox guests have always been reported.

### Added

- `hh inventory` reports TrueNAS VMs and LXC containers. It previously listed
  pools, Docker containers, and apps, but no guests at all, so a VM or LXC on
  the NAS was invisible to every command that builds on inventory.
- Guest reporting covers all three TrueNAS virtualization backends, because the
  middleware namespace changed with the engine:
  - 24.10 and earlier: libvirt VMs only, `vm.*`
  - 25.04 (Fangtooth) and 25.10 (Goldeye): Incus for both VMs and LXC,
    `virt.instance.*`
  - 26: Incus removed, libvirt drives VMs and libvirt_lxc containers, `vm.*`
    again
  Both namespaces are queried and whatever answers is reported, so one binary
  works across all of them with no per-host configuration.
- Version-aware TrueNAS virtualization docs in `capabilities/truenas.md`,
  `infra/truenas.md`, and the `truenas-ops` / `truenas-middleware` skills,
  including a live `core.get_methods` probe so the method surface is read off
  the box rather than assumed.
- A documented gotcha for the 26 Incus-to-libvirt migration: orphaned LXCs and
  VMs that vanish from the UI while their zvols survive under the hidden
  `.ix-virt` dataset. The data is usually intact, so the guidance is to reattach
  the existing zvol rather than delete and rebuild.

### Changed

- TrueNAS guests are formatted as a name/type/state table instead of raw
  truncated JSON, matching the readability of the Proxmox `qm list` and
  `pct list` output:

      docker-vm              VM         RUNNING
      pihole-lxc             CONTAINER  STOPPED

- `hh inventory` section labels on TrueNAS: `# Containers` is now `# Docker`,
  and guests appear under `# VMs / LXCs`. The old label was ambiguous once LXC
  containers entered the picture, since it actually meant Docker.
- The `homelab-triage` skill runs `hh inventory` alongside `hh overview` in step
  one, and states that overview lists no guests on any platform. Previously a
  "state of the homelab" answer could report healthy while a VM or LXC was
  stopped, because overview covers vitals and pool health only.

### Notes

- `hh overview` is unchanged and remains a fast vitals sweep. Guest state comes
  from `hh inventory`.
- If TrueNAS 26 ships LXC under a namespace that is neither `virt.instance` nor
  `vm`, those containers will not be listed yet. Confirm the real name on a 26
  box with the `core.get_methods` probe documented in `capabilities/truenas.md`.

<a id="v1-0-0"></a>

## 1.0.0 (2026-07-18)

Initial release. One command turns a fresh LXC into a homelab command center:
Claude Code plus a web UI, preloaded skills for TrueNAS, Proxmox, Docker, and
networking, and a credential broker that connects to managed hosts over SSH
without exposing secrets to the agent.
