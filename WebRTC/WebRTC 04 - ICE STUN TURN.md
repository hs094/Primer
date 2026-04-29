# 04 — ICE, STUN, TURN

🔑 Most peers sit behind NATs. [[ICE]] is the algorithm that finds *some* path between them, using [[STUN]] to learn public addresses and [[TURN]] as a relay when nothing else works.

Source: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity

## Candidate Types
| Type | Source | Cost |
|---|---|---|
| **host** | local interface IP | free, often unreachable across NAT |
| **srflx** (server reflexive) | learned from [[STUN]] | free, works across most NATs |
| **prflx** (peer reflexive) | discovered mid-connectivity-check | free |
| **relay** | allocated on a [[TURN]] server | bandwidth $ — relays all media |

## STUN
A tiny UDP server. Peer asks "what IP/port did you see this packet from?" — answer is the peer's public mapping. Lets two peers behind cone NATs find each other directly.

⚠️ STUN fails for symmetric NATs (different mapping per destination) and many corporate firewalls — that's where TURN earns its keep.

## TURN
Full media relay. Peer allocates a TURN address; both sides send to it; server forwards. Always works, but you pay for every byte. Production WebRTC apps budget ~10–20% of sessions falling back to TURN.

## ICE Process
1. **Gather** — for each network interface, collect host + srflx (+ relay) candidates.
2. **Exchange** — send candidates over [[signaling]] (trickled, not batched).
3. **Pair & check** — every local × remote pair runs STUN connectivity checks.
4. **Nominate** — best succeeding pair wins; media flows.
5. **Keepalive** — periodic checks; **ICE restart** if path dies.

## Trickle ICE
💡 Send each candidate the instant it's gathered. Connect time drops from "wait for full gather (~1s)" to "first working pair wins (~100ms)."

## Configuring
```js
new RTCPeerConnection({
  iceServers: [
    { urls: "stun:stun.l.google.com:19302" },
    { urls: "turn:turn.example.com", username: "u", credential: "p" }
  ]
});
```

🧪 Test TURN-only with `iceTransportPolicy: "relay"` — simulates the worst-case network.

## Tags
[[ICE]] [[STUN]] [[TURN]] [[WebRTC]]
