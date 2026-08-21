---
name: firewalla-ops
description: >
  Read and analyze the Firewalla firewall: box and WAN status, security alarms,
  traffic flows, firewall rules and blocklists, and every device on the network.
  Use this whenever Evan asks about the firewall, the Firewalla, security
  alarms, whether something was blocked, what a device is talking to, who is
  using bandwidth, a new or unknown device on the network, or whether a block
  or parental-control rule is actually in effect ("did Firewalla catch that",
  "why is this site blocked", "what is that IoT thing phoning home to",
  "is anything suspicious happening"). This is READ-ONLY: it can report and
  explain and recommend, but it cannot change any Firewalla setting, and it
  must never claim otherwise.
---

# Firewalla firewall (read-only)

The Firewalla is registered like any other host, but it is reached over the
Firewalla MSP API instead of SSH, so `hh run` does not apply to it. Everything
goes through one command:

    hh firewalla <op> [alias] [args]

The alias can be omitted when only one box is registered, which is the normal
case. `hh list` shows it with platform `firewalla` and auth `apitoken`.

## THIS IS READ-ONLY, AND HERE THAT MATTERS MORE THAN USUAL

Say it plainly to yourself before every Firewalla task: **you can look, you
cannot touch.** This is not a preference or a default that a good enough reason
can override. It is the point of the integration.

Read this next part carefully, because it is the one place where Firewalla is
genuinely more dangerous than UniFi:

- The UniFi integration has **two** independent locks. The broker only issues
  GET, and the API key is minted under a View Only admin, so the console itself
  refuses writes even if something got past the broker.
- Firewalla has **no View Only token tier at all.** The personal access token
  in the vault carries every permission the MSP account has. The MSP API really
  does expose creating rules, deleting rules, pausing and resuming them,
  renaming devices, and deleting or muting alarms.
- So there is exactly one lock: the broker behind `hh firewalla` issues HTTP
  GET and physically cannot issue anything else, and every write-shaped op name
  (`pause`, `resume`, `block`, `delete`, `create`, `mute`, ...) is refused by
  name. That single lock is the whole guarantee. Respect it accordingly.
- Do not try to route around this. Do not look for another path to the
  firewall: not SSH to the box, not the Firewalla mobile app, not `curl`
  against the MSP API yourself, not a script that "just" pauses one device for
  a second. If you catch yourself planning any of that, stop.
- If Evan asks you to change a Firewalla setting, do not refuse flatly and do
  not quietly do nothing. Say that HomelabHero's Firewalla access is read-only
  on purpose, then give him the exact steps to do it himself in the Firewalla
  app: which screen, which setting, which value, and what he should expect to
  see afterwards. Offer to re-read the state once he has done it and confirm it
  took effect. That is the whole workflow, and it is a good one.

Why it is built this way: the firewall is the one device whose failure takes
away the access you would need to fix it. A bad rule, an accidental device
pause, or a blocked subnet can cut off every host, the command center, and
Evan's own way in, all at once, with no way back except physically standing in
front of the hardware. Nothing an agent could usefully automate here is worth
that risk.

## What you can read

    hh firewalla summary        the one-screen picture: model, firmware, mode,
                                WAN IP, device/rule/alarm counts, recent alarms
    hh firewalla alarms [n]     security events, newest first (default 20)
    hh firewalla flows [n]      recent connections: device, destination,
                                category, blocked or allowed, MB (default 30)
    hh firewalla bandwidth [n]  top devices by traffic, last 24h (default 10)
    hh firewalla trends <type>  daily flows/alarms/rules counts over time
    hh firewalla devices        every client: name, IP, MAC, network, state
    hh firewalla device <q>     one device in full, by MAC, name, or IP
    hh firewalla rules          firewall rules: action, target, scope, status
    hh firewalla targetlists    blocklists and allowlists rules are built from
    hh firewalla boxes          every box on the account; the one this alias
                                reads is marked "<- this alias"
    hh firewalla info           full box record as JSON
    hh firewalla stats [type]   account rollup, or a leaderboard: type is
                                topBoxesByBlockedFlows, topBoxesBySecurityAlarms,
                                or topRegionsByBlockedFlows
    hh firewalla ping           reachability check
    hh firewalla get <path>     any other GET under /v2/ ({box} is expanded)

Start with `hh firewalla summary` for almost anything. It is one call and it
usually contains the answer or tells you which of the above to run next.

## What this is uniquely good for

Firewalla answers questions nothing else in the homelab can:

- **"Is anything suspicious happening?"** -> `hh firewalla alarms`. Port scans,
  abnormal uploads, intrusions blocked, unfamiliar devices joining. This is a
  better first question than any per-host check.
- **"What is that device actually talking to?"** -> `hh firewalla flows`, then
  read the destination and category columns. Especially useful for IoT gear
  that has no shell to log into.
- **"Did the block actually work?"** -> `hh firewalla rules`. Check the rule
  exists AND that its status is `active` rather than `paused`. A paused rule
  explains a surprising amount of "I thought I blocked that".
- **"What is on my network?"** -> `hh firewalla devices` sees phones, TVs,
  cameras, and everything else that never appears in a host inventory.

## The one number you must not misreport

`hh firewalla trends` is the only op here that is NOT scoped to this box. The
MSP trends endpoint has no box filter - it takes a box *group* at best and is
account-wide otherwise. The command prints what its figures actually cover on
the first line.

Read that line before repeating a number. On a multi-box account, saying "your
firewall blocked 71 flows yesterday" when the figure covers every box on the
account is a wrong answer delivered confidently, which is worse than saying you
cannot break it down per box. Every other op - bandwidth, flows, alarms, rules,
devices - is genuinely this box only.

## Reading the results well

- An alarm is a signal, not a verdict. "New device joined" is routine when Evan
  just set something up and alarming when he did not. Ask before concluding.
- A blocked flow is Firewalla working, not Firewalla failing. Say so plainly;
  people often read a wall of BLOCKED rows as an attack in progress when it is
  the normal background noise of the internet.
- Check timestamps before drawing conclusions. Alarms and flows default to
  recent windows, and flow retention depends on the MSP plan (some tiers keep
  24 hours, others 30 days), so "nothing there" can mean "aged out".
- Cross-check against the hosts before blaming the firewall. If a service is
  unreachable, confirm with `hh run <alias> "..."` whether the host itself is
  healthy first.

## The one failure mode that will confuse you

HomelabHero reaches the Firewalla through the **MSP cloud**, not over the LAN.
Firewalla publishes no supported local API, so this is the only documented path.

That means: if the internet is down, `hh firewalla` fails even though the box
is fine and sitting three feet away. Do not report that as "the firewall is
down". Distinguish the two before saying anything:

    ping -c3 192.168.2.1        # or whatever the box's LAN IP is - is it alive?
    hh firewalla ping           # is the MSP path working?

If the first works and the second does not, the box is healthy and the cloud
path is not. Likewise, a 401 means the token is wrong, expired, or the MSP plan
lapsed - it says nothing about the box.

## What Firewalla cannot see

It knows the perimeter, the clients behind it, and what crossed it. It does not
know about the NetBird mesh, internal DNS, Cloudflare tunnels, or anything
happening inside a host. For those, use the network-diag skill and `hh run`.

See `capabilities/firewalla.md` for the full op reference and the raw-GET query
syntax.
