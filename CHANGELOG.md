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
