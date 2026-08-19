# Firewalla - capability catalog

For a Firewalla box (Gold, Gold SE, Purple, Blue Plus, Purple SE) registered
with an MSP API token. Everything runs through `hh firewalla <op> [alias]`.
`hh run` does NOT apply here: the box is reached over the Firewalla MSP API,
not SSH.

## Read-only, with no exceptions - and read this part

Every capability below reads. There is no write capability, not because none
was documented but because none exists: the broker issues HTTP GET and has no
code path that can POST, PUT, PATCH, or DELETE.

Firewalla differs from UniFi in a way that matters. The UniFi integration has
two independent locks - a GET-only broker AND a key minted under a View Only
admin that the console itself refuses writes from. **Firewalla has no View Only
token tier.** The personal access token in the vault carries every permission
the MSP account has, and the API really does expose rule creation, rule
deletion, pausing and resuming rules, device renaming, and alarm deletion.

So the broker being read-only is not a convenience here, it is the only thing
standing between a mistake and the firewall. Do not look for a way around it,
and never tell Evan you have changed something on the Firewalla, because you
cannot have.

What there IS to do, when a change is needed, is tell him exactly what to
change in the Firewalla app and then verify the result by reading it back. See
the firewalla-ops skill.

## Overall state

- One-screen picture: `hh firewalla summary`
- Reachability check: `hh firewalla ping`
- Full box record as JSON: `hh firewalla info`
- Every box on the MSP account: `hh firewalla boxes`

`summary` reports the model, firmware version, routing mode, online state,
public (WAN) IP, and the device / rule / active-alarm counts, followed by the
three most recent alarms.

## Devices

- Everything the box sees on the network: `hh firewalla devices`
  (name, IP, MAC, network/VLAN, online state, last seen)
- One device in full: `hh firewalla device <mac|name|ip>`

What to look for: a device that is unexpectedly offline, or one you do not
recognise. Firewalla sees every client on the LAN, including things that never
appear in a host inventory - phones, TVs, cameras, IoT gear.

## Alarms - the thing Firewalla is actually for

- Recent security events: `hh firewalla alarms [limit]` (default 20, max 500)

Alarms are Firewalla's core value and have no UniFi equivalent. They cover new
devices joining, abnormal upload volumes, port scans, blocked intrusions, and
traffic to flagged regions or categories. When Evan asks "is anything wrong on
the network", this is a better first question than any host check.

## Flows - who talked to what

- Recent connections: `hh firewalla flows [limit]` (default 30, max 500)
  (time, device, destination domain or IP, category, blocked/allowed, MB)

Use this to answer "what is that device talking to", "what is using bandwidth",
and "was that connection actually blocked". Note that flow retention depends on
the MSP plan - some tiers keep 24 hours, others 30 days.

## Rules and blocklists

- Configured firewall rules: `hh firewalla rules`
  (action, target type and value, scope, status, id)
- Target lists (blocklists/allowlists that rules build on): `hh firewalla targetlists`

Reading rules is how you check whether a block is actually in place, and
whether a rule is `active` or `paused`. A paused rule explains a surprising
amount of "I thought I blocked that".

## Statistics

- Account-level rollup: `hh firewalla stats`

## Anything else (raw GET)

The named ops cover the common ground. For the rest, a read-only escape hatch
takes any path under `/v2/`, with `{box}` substituted for this box's id:

- Trends over time: `hh firewalla get '/v2/trends?query=box.id:{box}'`
- Top blocked flows: `hh firewalla get '/v2/stats/topBoxesByBlockedFlows'`
- One alarm in full: `hh firewalla get '/v2/alarms/<gid>/<aid>'`
- A filtered flow search: `hh firewalla get '/v2/flows?query=box.id:{box}%20category:malware&limit=50'`

Query syntax: filters are `field:value`, space-separated. Different fields are
ANDed, repeats of the same field are ORed. Useful qualifiers are `ts`, `status`,
`direction`, `box.id`, `device.id`, `device.name`, `category`, `domain`,
`region`, `dport`, `download`, `upload`, `total`. List responses are
`{count, results, next_cursor}`; `/v2/boxes` and `/v2/devices` return a bare
array instead.

## How this is reached, and what that implies

HomelabHero talks to the **Firewalla MSP cloud API** at your portal domain
(`https://<yourname>.firewalla.net/v2/...`), not to the box on the LAN.
Firewalla publishes no supported local REST API - the on-box service on port
8833 is internal and undocumented - so the cloud is the only documented path.

Two consequences worth remembering when troubleshooting:

- If the internet is down, `hh firewalla` is down too, even though the box
  itself is fine and three feet away. A failure here is not evidence about the
  box. `ping 192.168.2.1` from the command center distinguishes the two.
- API access requires an MSP account with an active plan. A 401 usually means
  the token is wrong, expired, or the plan lapsed - not that the box is broken.

## What Firewalla cannot tell you

It knows the perimeter, the clients behind it, and what crossed it. It does not
know about the NetBird mesh, internal DNS resolution, Cloudflare tunnels, or
anything happening inside a host. For those, use the network-diag skill and
`hh run` against the hosts themselves. A service can look perfectly healthy in
Firewalla's view and still be unreachable over the mesh.
