# Changelog

What changed in this repository and on the home network, newest first, with the
decisions made or deliberately deferred along the way. This repository commits
directly to `main`, so entries are keyed by date rather than by pull request.

This is a historical record. It holds no rules, no approvals and no standing
grants, and it is not indexed in any Claude Code memory store — read it when you
need to know what happened and why, and add to it when something lands.

`HOME-NETWORK.md` is the design record and the only authority on target state,
addressing, the firewall, the build order and the punch list.

---

## 2026-10-31 — the design record, written from the office

`84f933e`. The initial commit: the home network design record, decisions D1–D9,
the Pi build, remote access, the backup job, DR, and the §15 punch list.

**Nothing is built.** No UX7 configured, no Pi imaged, no `home-backups` stack,
no SSM node, no `connect.sh` entries. Every box in §15 is unticked and every one
of them is accurate.

Two findings from that session:

**The tunnel CJ was connected on is the office tunnel, and that does not change
when the build finishes.** The session ran on `OSCILLAS-0010`
(`172.20.60.11`, the Oscillas office LAN) with `SSH_CONNECTION=172.20.60.1` —
the office EdgeRouter, which NATs the connection onto the office LAN. The host
had no tunnel interfaces at all: no `tun`, `tap` or `wg`, and `wg`, `tailscale`,
`zerotier-cli`, `cloudflared` and `ngrok` all absent. There is no home segment
behind that path and there never will be: `connect.sh` (§9.2) creates loopback
port-forwards over an SSM session, not an L3 route into the house, so `172.21.x`
addresses resolve on the Pi and never on the client. Device discovery —
`arp-scan`, `nmap`, `ip neigh` — runs **on `bf-home-001`**, with the client
shelled in. That is D4's design, not a gap in it. A session physically at the
house can see the LAN directly, which is the whole reason to start there.

**D2's `172.21.0.0/16` collides with Docker.** On `OSCILLAS-0010` that prefix is
already a Docker bridge (`artemis-cloud_internal`), and `ip route get
172.21.10.1` resolves through it. Docker's default pool is `172.17.0.0/12`, it
had already taken six /16s from it, and it takes more as compose projects
appear. D2's rationale checks the office (`172.20/16`), AWS and `192.168/16` —
it does not consider Docker. Scoped correctly: **D4 is unaffected**, because SSM
port-forwards resolve the address on the Pi and the client's routing table never
sees it, so the primary remote-access path works regardless. **D5 is affected**:
a WireGuard tunnel into the house with `AllowedIPs = 172.21.0.0/16` ties with the
Docker route on prefix length, and per-VLAN /24s would win on specificity but
are brittle against the next compose project. Recorded in `HOME-NETWORK.md` §12
with the two remedies, both CJ's to choose between.
