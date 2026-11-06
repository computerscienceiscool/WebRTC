# TURN/SFU custom-implementation options

**Date:** 2026-11-05
**Status:** Decision support only. No provider, public deployment, custom fork,
or PromiseGrid media protocol is selected here.

## The important distinction

“Build our own” can mean four very different things:

| Choice | What is custom | What remains standard |
|---|---|---|
| Managed service | Application integration only | TURN/SFU operation and media stack |
| Self-host standard service | Deployment, configuration, credentials, monitoring | Coturn or LiveKit protocol implementation |
| Custom control layer | Policy, access decisions, short-lived credentials, deployment automation | Coturn/LiveKit media-plane behavior |
| Custom media implementation | TURN relay or SFU source itself | Only the published Internet standards |

The first three are realistic stages of one project. The fourth is a separate
networking/security product.

## Options at a glance

| Option | What the team owns | Advantages | Costs and risks | Best fit |
|---|---|---|---|---|
| **Managed TURN/SFU** | Application integration, provider account, spend limits | Fastest path; provider runs network edge, updates, capacity, and availability | Ongoing usage cost; provider dependency; less control over operations and data location | Need reliable evidence quickly, with little operations capacity |
| **Self-host standard Coturn + LiveKit** | Server(s), DNS/TLS, firewall, credentials, upgrades, monitoring, bandwidth bill | Control of deployment, data location, versions, and operating policy; no per-seat media vendor | Real 24/7 operational responsibility; public UDP/TCP/TLS exposure; capacity and incident work | A team has an operator and wants control without rewriting media protocols |
| **Standard services + custom control layer** | Everything above, plus policy/credential issuer and deployment automation | Adds room-specific authorization, short-lived access, local audit policy, and Grid-facing management without replacing proven media code | More application/security code; must protect credential secrets and correctly handle expiry/failure | There is a concrete local policy or deployment need that standard configuration cannot express alone |
| **Fork Coturn or LiveKit** | Upstream code changes, rebasing, custom builds, patches, regression/security maintenance | Can address a demonstrated missing capability while retaining most mature implementation | Becomes a long-term maintenance commitment; forks drift; upstream fixes need review and merge | A specific, repeatable requirement cannot be met through configuration, APIs, or a sidecar/control layer |
| **Write TURN or an SFU** | Full standards implementation and all operations | Maximum theoretical control | Highest interoperability, security, bandwidth, abuse, and maintenance risk; delays the actual meeting goal | Research project only, after explicit authorization and a compelling unmet requirement |

### The plain recommendation

For a first cross-network call: use a managed TURN service or self-host
standard Coturn. For a 10+ person meeting: add a standard SFU such as LiveKit.
Only add custom control code when there is a named policy need. Do not start by
forking or rewriting the media plane.

### Current working hypothesis

JJ expects Steve to prefer **self-hosted standard services**. Treat Coturn plus
a standard SFU such as LiveKit as the candidate worth understanding now. This
is not a selection. Managed media, forks, custom media code, and decentralized
SFU designs remain brief deferred alternatives unless one becomes a specific
concern or Steve asks for research.

## TURN: what custom implementation would mean

TURN is a standards-track relay protocol, specified by [RFC 8656](https://www.rfc-editor.org/rfc/rfc8656.html).
It is normally used through ICE, which combines direct candidates with STUN and
TURN candidates to establish a usable WebRTC path. [RFC 8445](https://datatracker.ietf.org/doc/rfc8445/)

A standard Coturn deployment already implements TURN/STUN, authenticated relay
allocations, UDP/TCP/TLS/DTLS transport variants, relay-port management, and
operational interfaces. [Coturn source and support matrix](https://github.com/coturn/coturn)

Writing or deeply forking the relay would make the team responsible for:

- browser interoperability across ICE, STUN, TURN, NATs, and restrictive
  firewalls;
- authenticated allocation, credential expiry, quotas, abuse prevention, and
  bandwidth accounting;
- public UDP/TCP/TLS listener and relay-port policy;
- TLS/DTLS security maintenance, logging redaction, metrics, incident response,
  and upgrades;
- proving that a relay failure does not become a call failure or an open relay.

The better first custom boundary is a **credential/control service**: the
application decides whether a participant may receive a short-lived TURN
credential; Coturn remains the standards-compliant media relay. This keeps the
hard network protocol in a mature service while leaving room for local policy.

## What a custom Coturn control layer actually does

The custom layer is a small service beside the application, not a replacement
TURN server:

```text
browser ── asks to join room ──> application
                                     │ checks identity, room rule, local policy
                                     v
                         credential issuer (custom code)
                                     │ creates short-lived TURN credentials
                                     v
browser <── iceServers + temporary username/password
   │
   └── standard TURN protocol ──> Coturn ──> relayed browser media
```

### What the issuer owns

1. **Eligibility:** decide whether this browser is allowed to receive media
   connectivity for this room/session.
2. **Lifetime:** set a short expiry, so a copied browser credential has a
   limited useful life.
3. **Configuration:** return the concrete `stun:` and `turn:`/`turns:` URLs,
   plus the temporary username and credential, in the browser's ordinary
   WebRTC `iceServers` configuration.
4. **Policy/audit:** record only the operational facts needed for support,
   quotas, and abuse response; do not record media frames or permanent TURN
   secrets in application evidence or browser diagnostics.
5. **Secret protection:** hold the shared Coturn authentication secret in a
   host-controlled secret store. The browser receives a derived, expiring
   credential, never that shared secret.

Coturn documents this time-limited mode as `use-auth-secret`: its username can
contain an expiry timestamp and user identifier, and the credential is derived
from the server secret. It also supports relay-port ranges, per-user/session
limits, TLS/DTLS, and metrics. Sources: [Coturn configuration example](https://github.com/coturn/coturn/blob/master/examples/etc/turnserver.conf) and [Coturn TURN-server options](https://github.com/coturn/coturn/wiki/turnserver).

### What Coturn still owns

- validating the temporary TURN credential;
- allocating relay addresses and ports;
- carrying TURN/STUN protocol traffic over its supported transports;
- enforcing its configured relay, bandwidth, and session limits;
- forwarding relay traffic and exposing service-level logs/metrics.

The issuer does not inspect or forward the actual call media. A valid browser
credential will necessarily be visible to that browser for its short lifetime;
the security goal is limited scope/lifetime and a protected shared secret, not
pretending that a browser can use TURN without any credential material.

### Failure behavior to design deliberately

| Failure | Safe outcome |
|---|---|
| Issuer unavailable before call setup | Browser cannot obtain new relay credentials; show a clear connectivity failure rather than silently using permanent credentials. |
| Coturn unavailable | ICE may succeed directly, but restrictive networks fail clearly; do not misreport this as room authorization failure. |
| Credential expired mid-call | Decide and test renewal before expiry; otherwise reconnect/re-negotiate deliberately. |
| Suspected abuse | Stop issuing credentials, narrow limits, rotate the shared secret under an operator procedure, and preserve only safe operational evidence. |

### Where Grid could fit — provisional only

An implementation could use local Grid-related evidence or a Grid-operated
agent to authorize deployment, issue a local policy decision, or publish the
chosen service endpoint to an approved application. That is an
implementation-specific control-plane choice. The browser still needs explicit
standard WebRTC `iceServers` data, and Coturn still speaks ordinary TURN.

The current PromiseGrid Guide does not define a universal relay-discovery,
credential-issuance, or browser-media protocol. Under `DR-tuhaz`, presenting
this control layer as a settled PromiseGrid requirement would be inaccurate.

## SFU: what custom implementation would mean

An SFU receives each participant's WebRTC tracks and forwards the appropriate
tracks to other participants. It solves group topology; it is not TURN.

LiveKit is an open-source WebRTC SFU that can be self-hosted. Its documentation
describes a single-node deployment and multi-node deployments that use Redis to
place every participant in a room on the same node. [LiveKit SFU architecture](https://docs.livekit.io/reference/internals/livekit-sfu/)

A custom SFU would inherit responsibility for:

- WebRTC session setup and encrypted media transport;
- track publication, subscription, forwarding, and quality/layer selection;
- bandwidth adaptation, CPU pressure, muted/paused tracks, reconnection, and
  diagnostics;
- room placement, node failure, migration/rejoin behavior, and multi-region
  routing if more than one machine participates;
- authorization, moderation, recordings, observability, resource limits, and
  media-service upgrades.

That is why a decentralized SFU is not merely “several SFUs.” It also needs a
reliable answer to which node owns a room, where a participant connects, how
failure is detected, and how a room recovers without duplicating or losing
media state.

## Recommended learning/deployment order

1. Run a standard Coturn service with short-lived credentials and prove one
   cross-network two-person call.
2. Operate one standard SFU for the anticipated 10+ person call shape.
3. Add a custom application/control layer only where it provides a real policy,
   deployment, or observability benefit.
4. Consider a fork only after a concrete unmet requirement is demonstrated.
5. Treat a custom TURN implementation or decentralized SFU as separately
   authorized research, not as the first operational path.

## PromiseGrid boundary

The **Bundled PromiseGrid Development Guide — App Devs** describes the
application surface as provisional under `DR-tuhaz`; it does not settle a
universal browser-media, TURN-discovery, TURN-credential, or SFU protocol.

Therefore a Grid-facing control layer is an **implementation-specific candidate**:
it may manage a chosen deployment, but it must not be presented as a PromiseGrid
requirement or as a replacement for standard browser `turn:`/`turns:` and
WebRTC behavior.

### Grid-facing functions that can remain host-local

The following may be informed by a Grid app or its evidence, but can remain
local implementation and operations policy. None needs a new browser-media
protocol:

| Function | Sensible boundary | Must not be claimed |
|---|---|---|
| Admit a participant to a room | Application/control service checks its room rule before issuing session or TURN credentials. | That TURN itself understands PromiseGrid. |
| Issue short-lived TURN credentials | Host-controlled credential issuer derives temporary credentials for standard Coturn. | That a Grid rule replaces TURN authentication. |
| Select an SFU endpoint | Application returns the approved URL/token for the selected standard SFU deployment. | That WebRTC discovers or negotiates an SFU through a settled Grid protocol. |
| Apply quota or budget policy | An operator-owned policy can cap rooms, relay lifetime, concurrent users, or spend. | That a universal PromiseGrid economic/budget rule exists for media. |
| Retain operational evidence | Keep safe audit facts such as deployment version, authorization decision, and aggregate usage. | That media frames, permanent secrets, or browser diagnostics belong in durable Grid evidence. |

The relevant PromiseGrid status is still provisional: the Bundled Development
Guide's App Dev guidance says no final universal app API is fixed while
`DR-tuhaz` remains open. These are therefore local implementation promises,
not protocol requirements.

## Deployment decision gate for Steve

Before any public TURN or SFU deployment, custom fork, or decentralized-SFU
experiment, record a short decision with these answers:

| Decision | What needs an explicit owner/answer |
|---|---|
| **Purpose** | Which bounded outcome is authorized: two-person cross-network proof, group-call proof, ordinary self-hosted operation, or cross-SFU research? |
| **Operator** | Who has authority to configure public DNS/firewall/TLS, hold TURN/SFU secrets, apply upgrades, and take the service offline during an incident? |
| **Admission authority** | Who may create rooms, obtain credentials, invite participants, and revoke access? |
| **Operating budget** | What monthly infrastructure/bandwidth ceiling is acceptable, who monitors it, and what automatic limit or shutdown behavior applies when it is reached? No number is assumed here. |
| **Security posture** | Required credential lifetime, rate/port/session limits, log-redaction rules, patch cadence, and incident contact. |
| **Success evidence** | The exact test, networks/devices, expected behavior, and measured signals—join success, direct versus TURN path, loss/bitrate, CPU/bandwidth, and recovery behavior. |
| **Stop rule** | What result ends the trial or returns the team to a non-public/local test rather than silently expanding a prototype? |
| **Custom-code threshold** | The specific demonstrated requirement that configuration, standard service APIs, and a sidecar/control layer cannot meet. |

For the current direction, the smallest responsible authorization is: one
self-hosted standard TURN/SFU trial, a named operator, a bounded budget, and
a written success/failure test. A custom fork or decentralized media mesh
needs a *new* authorization after the baseline results are available.

## Questions for Steve

- Does “custom” mean self-operating existing open-source services, adding a
  control layer, or writing/forking media-plane code?
- Is the first success criterion a two-person cross-network call, a 10+ person
  meeting, or a decentralized-media experiment?
- Who owns public network exposure, credentials, bandwidth cost, monitoring,
  security updates, and incident response?
- What spending ceiling and automatic stop/alert behavior are acceptable for
  the first public trial?
- What evidence would demonstrate that a custom fork or cross-SFU design is
  warranted rather than continued use of standard services?
- What concrete need would justify a fork instead of configuration and normal
  service integration?
