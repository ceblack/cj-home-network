# cj-home-network

Black family home network: design record, remote access, config backups to S3.

| File | What |
|---|---|
| `SESSION-HANDOFF.md` | **Read first.** Session-to-session handoff: execution boundary, current state, what the last session verified, open questions only someone at the house can answer |
| `HOME-NETWORK.md` | The design record. Decisions, hardware, addressing, VLANs, firewall, wireless, the Pi, remote access, UX7 → S3 backups, DR, punch list |

Nothing here is built yet. `HOME-NETWORK.md` is the target state and the
commands that get there; it has never been executed, so treat it as a careful
plan rather than a tested runbook.

Built from the structure of `~/Code/network-1770` (Oscillas office): same
addressing and naming conventions, same backup and remote-access patterns, far
less machinery. Where this network departs from that one, the handoff says so
and why — §1 for the decisions, §12 for the gotchas worth carrying across.

Shape of it: a UniFi Express 7 is router, AP and controller in one box; an
unmanaged TP-Link switch splits its single LAN port between a Raspberry Pi 3B+
and the TV; the Pi is an AWS SSM hybrid node that provides remote access and
pushes nightly UX7 config backups to S3 in the `black-family` account.
