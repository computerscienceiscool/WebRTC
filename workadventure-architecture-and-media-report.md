# WorkAdventure architecture and media report

**Reviewed:** 2026-11-05
**Scope:** Current public WorkAdventure documentation. This describes the
architecture; it does not recommend adopting WorkAdventure or its suppliers.

## Short answer

WorkAdventure is a virtual office/world application. It decides where avatars
are, when people are close enough to talk, and what the room rules are. It
uses different media paths depending on the conversation:

```text
WorkAdventure app: map, avatars, proximity, rooms, chat, permissions
                 │
      ┌──────────┼──────────┐
      │          │          │
 direct WebRTC  TURN     LiveKit SFU
 small group   fallback  group call
```

The important point is that its world logic is separate from the services that
carry media.

## Discussion bubbles

A discussion bubble is WorkAdventure's normal proximity conversation. A person
walks close to another person and stops; a circle appears around the group,
with audio, video, and chat. The default maximum is four people, although an
administrator can change it. A bubble is a product/UI concept, not a server.

Source: [Talking in a discussion bubble](https://docs.workadventu.re/user/proximity-bubble/).

## Small conversations: peer-to-peer WebRTC

For a small bubble, WorkAdventure tries browser-to-browser media first. This
avoids sending every stream through a central media server, but it does not
always work across home, office, school, mobile, and corporate networks.

## STUN and TURN/Coturn

**STUN** helps a browser discover its public-facing network route while trying
a direct connection. It normally does not carry call media.

**TURN** relays media when that direct route cannot be made. Both browsers make
outgoing connections to the relay, which forwards the encrypted WebRTC traffic.
WorkAdventure supports self-hosted Coturn or managed Cloudflare TURN. Its
current self-hosting guide says to provide TURN for ICE connectivity and
supports time-limited credentials generated from a shared secret.

TURN is about reachability, not group-call scaling. It does not turn a mesh of
ten browsers into an efficient conference.

Source: [WorkAdventure administration self-hosting guide](https://docs.workadventu.re/admin/admin-self-hosting/).

## Large conversations: LiveKit SFU

For five or more people in a room or bubble, current WorkAdventure
documentation says media goes through **LiveKit**, a selective forwarding unit
(SFU). Each browser uploads one stream to LiveKit; the SFU forwards the needed
streams to the other participants.

```text
Mesh, 10 people: each browser uploads 9 copies.
SFU,  10 people: each browser uploads 1 copy to LiveKit.
```

This is why an SFU is the sensible topology for a 10+ person meeting. It is not
a different name for TURN. WorkAdventure can run LiveKit in Kubernetes or on
dedicated server(s); its documentation identifies LiveKit as its most
resource-intensive component because of CPU and bandwidth.

Source: [WorkAdventure administration self-hosting guide](https://docs.workadventu.re/admin/admin-self-hosting/).

## Why both TURN and LiveKit can be needed

| Component | Role |
|---|---|
| STUN | Help discover a direct browser route |
| TURN/Coturn | Relay when a browser cannot make that route |
| LiveKit SFU | Forward group media without a browser mesh |

For a large WorkAdventure group, LiveKit is the normal media destination. A
participant behind a restrictive network can still need TURN to obtain a usable
media path. That is why WorkAdventure deploys both kinds of service.

## Jitsi is optional and transitional

WorkAdventure also supports Jitsi rooms. A map can point a meeting room to a
self-hosted Jitsi instance. But newer WorkAdventure administration docs say
Jitsi was the main conference solution in past releases and is being superseded
by LiveKit for a more integrated experience. Other pages still document Jitsi
rooms, so the exact release/deployment configuration matters.

Sources: [Jitsi meeting rooms](https://docs.workadventu.re/map-building/tiled-editor/meeting-rooms/) and [WorkAdventure administration self-hosting guide](https://docs.workadventu.re/admin/admin-self-hosting/).

## Other operational parts

A full deployment can also include HTTPS ingress, DNS names for the application
and media services, OIDC login, Matrix/Synapse chat, object storage for maps
and backups, optional recording, monitoring, upgrades, and incident response.

## Source-code map

These are separate projects; there is not one small “bubble server” repository.

| What to inspect | Repository | Why it matters |
|---|---|---|
| WorkAdventure core: browser world, proximity bubbles, rooms, application services, and integration configuration | [workadventure/workadventure](https://github.com/workadventure/workadventure) | Main monorepo; start here for how the virtual world decides that a bubble or meeting exists. |
| Build a map, attach map scripts, and create a local world | [workadventure/map-starter-kit](https://github.com/workadventure/map-starter-kit) | Practical map-authoring starting point, including TypeScript map scripts. |
| Reusable map-script features | [workadventure/scripting-api-extra](https://github.com/workadventure/scripting-api-extra) | Optional reusable features layered on the map scripting API. |
| SFU for current 5+ person media | [livekit/livekit](https://github.com/livekit/livekit) | LiveKit server source; separate from WorkAdventure. |
| TURN/STUN service commonly self-hosted with WorkAdventure | [coturn/coturn](https://github.com/coturn/coturn) | Standards-based connectivity service; separate from the app and SFU. |
| Optional/legacy WorkAdventure room integration | [jitsi/jitsi-meet](https://github.com/jitsi/jitsi-meet) | Jitsi client and conference stack. |
| Jitsi's media-forwarding component | [jitsi/jitsi-videobridge](https://github.com/jitsi/jitsi-videobridge) | Jitsi media bridge, separate from the Jitsi Meet browser/UI project. |

The first repository is where to inspect **bubbles**. LiveKit and Coturn are
external services that WorkAdventure configures and uses; they are not source
directories inside the WorkAdventure monorepo.

## What the current core source says about bubbles

I inspected the WorkAdventure core repository at commit `323ef90` (a temporary,
read-only checkout on 2026-11-05). Its backend has two explicit media
strategies:

- `back/src/Model/Strategies/WebRTCCommunicationStrategy.ts` creates a separate
  direct WebRTC connection for each pair of people in the communication space.
- `back/src/Model/Strategies/LivekitCommunicationStrategy.ts` creates a
  LiveKit room and issues invitations/tokens for participants instead.
- `back/src/Model/Policies/TransitionPolicy.ts` chooses which strategy to use.
  It switches from direct WebRTC to LiveKit when the participant count exceeds
  the configured `MAX_USERS_FOR_WEBRTC`, when a participant reports CPU
  limitation in a bubble larger than two, or when automatic recording requires
  LiveKit. It can switch back only when the count falls back within the limit
  and no recording/CPU condition remains.

The current `.env.template` sets `MAX_USERS_FOR_WEBRTC=3`, so that checked-in
source configuration would switch a **fourth** participant to LiveKit. This is
more conservative than public documentation that describes LiveKit for five or
more people. The threshold is configurable, so the safe statement is: **the
deployment chooses a threshold; do not infer one fixed threshold from a product
description.**

## What matters here

- Bubble: WorkAdventure's proximity-selected conversation.
- LiveKit: its current group-media direction for five or more people.
- Coturn/TURN: a separate fallback for blocked WebRTC routes.
- Jitsi: supported in meeting rooms, but no longer its sole/current media
  direction.

WorkAdventure is a useful example of separating world logic, network fallback,
and group-media forwarding. It is not a blueprint requiring its map, providers,
thresholds, or feature set.

## Sources

- [WorkAdventure: discussion bubbles](https://docs.workadventu.re/user/proximity-bubble/)
- [WorkAdventure: administration self-hosting](https://docs.workadventu.re/admin/admin-self-hosting/)
- [WorkAdventure: network preparation](https://docs.workadventu.re/admin/preparing-your-network/)
- [LiveKit SFU architecture](https://docs.livekit.io/reference/internals/livekit-sfu/)
