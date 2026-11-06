# Decentralized SFU exploration

**Status:** exploratory architecture note for TODO 100. It describes options;
it does not select a media stack, authorize a build, or define a PromiseGrid
protocol.

## Short answer

Steve's stated priority is decentralized computing, and JJ understands his
November 5 idea as the stronger version below: **one conference distributed
across several SFUs**. That is a reasonable technical direction to explore,
but the exact topology still needs Steve's confirmation before it becomes a
build decision.

The smaller meaning is multiple self-hosted SFU nodes that can serve different
rooms. That is already a distributed deployment, but it still needs a shared
way to decide which node owns each room.

The more ambitious definition is a *single room* spread across regional SFUs
which forward media between themselves. Jitsi calls its bridge-to-bridge mode
"relays" (previously called Octo). It can reduce the long-distance leg for
participants near different regions, but it adds a control plane and an
inter-SFU media network. It is not a collection of independent machines that
can work without coordination.

For a first real deployment, operate one ordinary self-hosted SFU and TURN
service first. That establishes the baseline for connection failures,
bandwidth, room lifecycle, and monitoring. It does not reject Steve's
decentralized-computing direction; it gives a future multi-SFU experiment a
comparison point and prevents inventing a mesh without knowing whether it
improves the call.

## Three different ideas that can be confused

| Meaning | What happens to a room | Coordination still required | Practical reading |
|---|---|---|---|
| **One ordinary SFU** | Every participant connects to one SFU. | Signaling, authentication, TURN credential issuance, and the room's state. | The correct baseline. |
| **Many SFUs, one home per room** | Different rooms can land on different nodes; all participants in a particular room use its chosen node. | A room-to-node directory, load/capacity information, health, join routing, and graceful drain. | Distributed capacity, not a cross-node room. |
| **Cascaded regional SFUs** | Participants may connect to nearby SFUs; the SFUs relay a conference's selected media between regions. | Everything above, plus region choice, peer-SFU membership, secure bridge links, failure handling, and cross-SFU metrics. | A genuine multi-node conference design. |

A fourth, much stronger meaning—each person's machine independently acting as
part of the media network—is not an SFU deployment pattern. It resembles a
browser peer mesh or a new distributed-media protocol. It would need separate
authorization, threat modeling, NAT traversal work, and a reason to prefer it
over known SFU designs.

## What a multi-machine system must decide

### Room ownership and join routing

There must be one answer to “which node owns room `R` now?” A client that
reaches the wrong ingress must be redirected or bridged to the owner. This is
not optional bookkeeping: two nodes independently creating the same room can
produce split calls.

LiveKit's self-hosted multi-node documentation is a concrete example: Redis is
the shared room store and message bus; nodes report statistics, choose a host
for a new room, and route joining clients to that room's node. Its current
self-hosted design keeps each individual room on one node.

### Capacity and placement

Placement needs more than a count of open rooms. It should consider available
CPU, packet-processing headroom, outbound bandwidth, expected subscriptions,
region/latency, and whether a node is draining. A busy room cannot safely be
moved like an ordinary web request while browser transport sessions are live.

### Failure and recovery

The team must explicitly choose the failure behavior:

- A node fails: do clients reconnect and rejoin on another node, accepting a
  short interruption?
- A control-plane store fails: can active rooms continue, and can new joins be
  admitted safely?
- A regional relay fails: do affected participants fall back to another region
  or reconnect to the room's home node?

Jitsi's relay documentation illustrates why this matters. It describes bridge
relay expiration and recovery behavior when its coordinator loses state;
orphaned relay-only conferences can require an operator-visible cleanup path.

### Security, admission, and abuse control

Every node must enforce the same room-admission and media-authorization
decision. Inter-SFU links need authenticated, encrypted transport and a
bounded membership rule. The control plane also needs audit records for who
created a room, who was admitted, which node served it, and why a routing or
capacity choice was made.

### Observability

A call must be diagnosable end-to-end: client connection type (direct or
TURN), assigned SFU/region, room owner, node and relay health, packet loss,
round-trip time, bitrate, join/reconnect failures, and capacity. Without this,
more nodes make a bad call harder—not easier—to locate.

## What it can solve—and what it cannot

| Potential benefit | When it is real | What it does not solve |
|---|---|---|
| More total concurrent rooms | There are enough independent rooms to spread over nodes. | A single oversized room, unless the SFU supports splitting one room. |
| Regional latency and data-locality choices | Participants and infrastructure are genuinely far apart and regional placement is measured. | Last-mile loss, a participant's restrictive firewall, or missing TURN. |
| Better resilience for *new* rooms and node maintenance | Nodes can be drained and new rooms placed elsewhere. | Seamless survival of an active room whose only owner fails, unless explicitly designed and tested. |
| Isolation | A busy or faulty node affects fewer rooms. | The operational burden of secure shared identity, routing, and monitoring. |

This is why multi-node architecture should follow evidence. It does not replace
TURN: TURN is a relay for a browser that cannot obtain a usable network path to
the SFU or peer. An SFU mesh is server-side media placement and forwarding.
They solve different network problems.

## What existing self-hosted systems show

### LiveKit: distributed cluster, one node per room

LiveKit documents a straightforward multi-node design: Redis holds room data
and relays messages, and a node selects a host based on cluster information.
The room remains on one self-hosted node. Its region-aware mode adds
region-aware ingress and node selection, rather than splitting a self-hosted
room across nodes. That is a good reference for the *first* meaning of
decentralization: a coordinated cluster.

### Jitsi Videobridge: cascaded bridges for a conference

Jitsi supports a more ambitious model. Its region-based configuration lets the
conference coordinator select a bridge based on participant region. Its relay
documentation says secure Octo relays use ICE and DTLS/SRTP between bridges.
The coordinator selects bridges and tracks the conference; the regional
bridges do not independently discover and form the conference. This is a
real-world reference for the *third* meaning: a coordinated, cascaded
conference.

## Boundary: media infrastructure versus application policy

These are media-infrastructure responsibilities:

- WebRTC sessions to browsers; SFU packet forwarding; TURN relay service.
- Node/room assignment mechanism, media relay links, capacity telemetry,
  connection draining, and media health monitoring.

These are application/product decisions that *could* inform the infrastructure
but are not supplied by WebRTC:

- Who may create or join a room, and what room identity means.
- Whether a room must stay in a particular region, organization, or machine.
- Spending/budget policy for relayed bandwidth and what audit history is kept.
- The decision to retry, reconnect, fail visibly, or end a room after a node
  failure.

The **Bundled PromiseGrid Development Guide** (plugin snapshot 0.1.0) does not
settle a media topology here: its app-developer surface remains provisional
under `DR-tuhaz`. If a later application has a Grid control plane, it could
carry an authorized room or placement decision, but that would be an
implementation-specific design question—not evidence that the media plane
becomes decentralized.

## A bounded experiment—only after the baseline

Do **not** begin by building a custom mesh. First run a standard self-hosted
TURN plus one standard self-hosted SFU for a small, real cross-network room and
record: browser path, TURN use, join success, bitrate, loss, CPU, outbound
bandwidth, and reconnect behavior.

If measurements show a need for more capacity, run this limited experiment:

1. Start two same-version SFU nodes in a test environment.
2. Use the stack's documented coordinator/directory; do not invent a new
   browser protocol.
3. Prove that separate rooms land on separate nodes, while every participant
   in one room reaches the same owner.
4. Drain one node and document what happens to active and new rooms.
5. Capture the metrics above and decide whether the added operations justified
   the benefit.

Only if there is measured regional latency or a single-conference scaling need
should the team separately authorize a cascaded-bridge test. A useful success
criterion would be a specific improvement (for example, measured join or
media-latency improvement for two regions) with a documented failure and
recovery behavior.

## Questions for Steve

Use this opening so the discussion starts from the intended idea:

> I understand the goal is one meeting that can span several cooperating SFUs,
> not merely many servers dividing up separate rooms. What properties must that
> decentralized design have for you?

Then ask:

1. Is the first purpose geographic locality, resilience against one machine or
   operator failing, distributing ownership, or all of those?
2. May there be a small shared coordinator/directory for room placement, or
   must membership and placement themselves be decentralized?
3. When a participant joins, may one SFU be the room's temporary coordinator,
   with other SFUs serving regional participants?
4. When one SFU disappears mid-call, is a brief reconnect acceptable, or must
   the call continue without an interruption?
5. What is the first realistic test: two regions, two independently operated
   machines, a target number of participants, or a failure scenario?
6. What measurement would count as success—lower latency, continued service
   after a node failure, local control of media, or a particular capacity?

## Sources

- [LiveKit: SFU architecture](https://docs.livekit.io/reference/internals/livekit-sfu/)
- [LiveKit: distributed multi-region self-hosting](https://docs.livekit.io/transport/self-hosting/distributed/)
- [Jitsi: scalable setup](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-scalable/)
- [Jitsi: region-based setup](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-region/)
- [Jitsi Videobridge: cascaded bridge relays](https://github.com/jitsi/jitsi-videobridge/blob/master/doc/relay.md)
