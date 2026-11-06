# Jitsi and WorkAdventure media-architecture review

**Date:** 2026-11-05
**Source:** Steve's November 5 suggestion, recorded in TODO 098.
**Status:** Source-backed architecture comparison; not a product choice,
adoption recommendation, or PromiseGrid protocol proposal.

## Short answer

Jitsi and WorkAdventure are useful examples of a boundary the meeting prototype
also needs: the application/room layer is separate from the WebRTC media
infrastructure. They do **not** support the conclusion that a small prototype
should fork either project.

For a first cross-network, two-person proof, the immediate missing
infrastructure category is an authenticated TURN service with a reachable
browser-facing deployment. An SFU/media server is a later topology decision,
primarily justified by group size, bandwidth, and reliability requirements.

## What Jitsi separates

Jitsi documents these distinct components:

| Responsibility | Jitsi component |
|---|---|
| Browser client | Jitsi Meet web/mobile application |
| Signaling | Prosody (XMPP) |
| Conference coordination and bridge selection | Jicofo |
| Forwarding conference media | Jitsi Videobridge (JVB) |

That separation is the useful idea. Jicofo manages a conference and assigns a
videobridge; JVB routes streams rather than making the application server carry
all media. Jitsi's scalable deployment also separates the web/signaling/Jicofo
node from one or more videobridges. Its region setup adds explicit participant
region detection and bridge selection, which illustrates that a multi-machine
media mesh needs routing and operational coordination, not merely more media
machines.

Sources: [Jitsi architecture](https://jitsi.github.io/handbook/docs/architecture/),
[Jitsi scalable deployment](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-scalable/),
and [Jitsi region-based setup](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-region/).

## What WorkAdventure separates

WorkAdventure's production documentation separates its room/world application
from its media infrastructure. Its self-hosting guide calls for a browser-facing
HTTPS application deployment, Coturn for restrictive networks, and LiveKit for
audio/video groups larger than four. Its Docker documentation says Coturn is
still needed for one-to-one and small-group calls when networks block direct
connections. The same documentation describes Jitsi as a large-room option.

This is a useful correction to an oversimplified “WorkAdventure is Jitsi”
picture: its current materials show a composed stack, not one indivisible
Jitsi-based system. The precise media service varies by feature and deployment
version.

Source: [WorkAdventure's self-hosting Docker guide](https://github.com/workadventure/workadventure/blob/master/contrib/docker/README.md).

## Reusable lessons for the prototype

1. Keep room/application state, signaling, TURN, and any SFU as distinct
   operational responsibilities.
2. Treat TURN as the cross-network fallback for a two-person call; do not wait
   for an SFU to validate that basic case.
3. Issue time-limited TURN credentials rather than long-lived browser-visible
   credentials. WorkAdventure explicitly recommends its static-secret
   time-limited-credential path over a fixed username/password.
4. Treat an SFU as a deliberate media service with public-network, bandwidth,
   monitoring, privacy, and failure-recovery duties.
5. A distributed/decentralized SFU would add placement, routing, failure, and
   observability problems. Jitsi's coordinated bridge selection is evidence of
   those extra responsibilities, not proof that a decentralized SFU is the
   right next step.

## Questions to take to Steve

- Is the next bounded goal a reliable two-person cross-network call, or a
  multi-person room? The first establishes the TURN requirement; the second
  may justify selecting an SFU.
- Who is authorized to operate the public DNS, TLS, TURN relay ports,
  credentials, bandwidth budget, and monitoring?
- Is the decentralized-SFU idea a long-range research topic, or should the
  team first operate one ordinary, observable SFU as a baseline?

## Boundary

This review does not select Jitsi, WorkAdventure, LiveKit, coturn, or a hosted
provider. It does not claim a settled PromiseGrid media protocol: browser-media
deployment is an application/operator concern, separate from any future
PromiseGrid-facing application protocol.
