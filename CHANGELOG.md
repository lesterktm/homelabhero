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
