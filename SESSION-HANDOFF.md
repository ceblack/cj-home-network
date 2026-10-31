# Session handoff: HOME-01 → HOME-02

Written 2026-10-31 UTC by `HOME-01`. That session ran on `OSCILLAS-0010`
(Oscillas office, `172.20.60.11/24`, Ubuntu 26.04, user `cj`, repo
`~/Code/cj-home-network`). It never had a path to the home network and never
will — see §3.

`HOME-02` runs on a machine of CJ's at the house. It starts with none of
`HOME-01`'s Claude Code memory. Everything durable is in this file.

**`HOME-02`: fill in §2 as found on first run.** Hostname, address, whether
the box is on the house LAN, and the Docker check in §4.1.

Read order: `~/.claude/CLAUDE.md` (global rules — this repo has no CLAUDE.md
of its own yet), this file, then `HOME-NETWORK.md` (the design record,
§1 decisions D1–D9, §15 punch list).

---

## 1. Execution boundary

CJ's global `CLAUDE.md` holds the hard rule. The specifics for this repo:

- **Network infrastructure is CJ's alone.** The UX7 (`BF-UX-001`), the ISP
  router/ONT, and AWS writes. Produce paste-ready commands; CJ runs them and
  reports back. Read-only `aws` describe/get/list is fine.
- **The Pi (`bf-home-001`) is not on that list.** It is a box being built
  from scratch that holds nothing irreplaceable (`HOME-NETWORK.md` §14).
  Act on it with CJ's grant. Nothing on it is production.
- **No CloudFormation `deploy --execute`.** §10.1 deliberately uses
  `--no-execute-changeset`; CJ reviews and executes the change set.
- **Secrets never enter git.** The UniFi local-admin password, `.unf` files,
  anything matching `*.env`. `.gitignore` already covers `.ssh/`, `*.unf`,
  `*.unifi`, `backups/`, `*.env`. Do not weaken it.
- **Don't re-pitch.** CJ has been told about the Docker collision (§4.1) and
  the git push question (§2.2). Raise them once more only if he asks.

---

## 2. State

### 2.1 Nothing is built

`HOME-NETWORK.md` is target state, not a record. No UX7 configured, no Pi
imaged, no `home-backups` stack, no SSM node, no `connect.sh` entries. Every
box in §15 is unticked and every one of them is accurate.

`HOME-02` is the first session that can touch any of it, because it is the
first session on a machine that can reach the hardware.

| Field | Value |
|---|---|
| `HOME-02` host | *(fill in: hostname, OS)* |
| Its address | *(fill in: `ip -br addr`)* |
| On the house LAN? | *(fill in — wired, Wi-Fi, or neither)* |
| Docker check (§4.1) | *(fill in)* |

### 2.2 Git

`HOME-01` made the initial commit and pushed it, so `HOME-02` can clone.

```
remote origin:   git@github.com:ceblack/cj-home-network.git   (SSH)
core.sshCommand: ssh -i .ssh/deploy_key -o IdentitiesOnly=yes
```

The remote was HTTPS at handoff and was switched to SSH, because a deploy key
only authenticates over SSH.

The key is `.ssh/deploy_key` (4096-bit RSA, comment `cj@OSCILLAS-0010`).
`.gitignore` blocks `.ssh/`, so **git does not carry it** — it exists only on
`OSCILLAS-0010`. `core.sshCommand` is local config and does not survive a
clone either.

On a fresh clone, cloning over SSH with CJ's own GitHub key needs nothing
further. To use the deploy key instead, copy it over out of band, `chmod 600`
it, and:

```bash
git config --local core.sshCommand 'ssh -i .ssh/deploy_key -o IdentitiesOnly=yes'
```

Commit only when CJ asks. Push only when CJ asks.

### 2.3 AWS

`black-family` is account `883357539412`, reached by chaining off the
`oscillas` SSO profile via `OrganizationAccountAccessRole`, region
`us-east-1`. Already in `~/.aws/config` on `OSCILLAS-0010`; `HOME-02` must
confirm the same profile exists on its own machine.

**The SSO token was expired at handoff.** Both `oscillas` and `black-family`
failed with `Token has expired and refresh failed`. Nothing AWS-side moves
until:

```bash
aws sso login --sso-session default-sso
aws sts get-caller-identity --profile black-family
```

---

## 3. The tunnel question, settled

CJ asked `HOME-01` to confirm it could see home devices over the tunnel he
was connected on. It could not, and the reason is structural.

What `HOME-01` found on `OSCILLAS-0010`:

| Check | Result |
|---|---|
| Host | `OSCILLAS-0010`, `172.20.60.11/24` — Oscillas **office** LAN (`172.20/16`) |
| CJ's session | `SSH_CONNECTION=172.20.60.1` — the office EdgeRouter `OSC-ER-001` (`b4:fb:e4`, Ubiquiti). His tunnel terminates on the router, which NATs him onto the office LAN: the `office-link-hub` WireGuard path in `cj-local`'s `tunnel.env` |
| Tunnel interfaces | none — no `tun`/`tap`/`wg`; `wg`, `tailscale`, `zerotier-cli`, `cloudflared`, `ngrok` all absent |
| Visible neighbours | three, all office: `.1` gateway, `.10`, `.12` |

So the tunnel CJ is on is the **office** tunnel. There is no home segment
behind it, and there is no home network to reach yet regardless.

**This does not change once the build is finished.** `connect.sh` (§9.2)
creates loopback port-forwards — `127.0.0.1:15120` → the Pi's `:22`,
`127.0.0.1:15121` → `172.21.10.1:443` *through* the Pi — over an SSM session.
It is not an L3 route into the house. `172.21.x` addresses are resolved on
the Pi, never on the client. Any device discovery (`arp-scan`, `nmap`,
`ip neigh`) runs **on `bf-home-001`**, with the client shelled in over
`homepi`. That is the design in D4, not a gap in it.

`HOME-02` is different: it is physically at the house, so it can see the LAN
directly. That is the whole reason to start there.

---

## 4. Corrections to the design record

Found by `HOME-01`. Neither is written into `HOME-NETWORK.md` yet — CJ was
asked and the session handed off before he answered. **Record them in §12 and
§15 once he confirms.**

### 4.1 D2's `172.21.0.0/16` collides with Docker

On `OSCILLAS-0010`, `172.21.0.0/16` is already a Docker bridge:

```
$ docker network ls   (subnets)
artemis-cloud_internal   172.21.0.0/16   (br-a24e8411b1da)
oscillas-crm_default     172.18.0.0/16
artemis_dev_proxy        172.19.0.0/16
artemis-artc-…_module    172.22.0.0/16
artemis-artc-…_internal  172.23.0.0/16
bridge                   172.17.0.0/16

$ ip route get 172.21.10.1
172.21.10.1 dev br-a24e8411b1da src 172.21.0.1
```

`BF-UX-001`'s management address resolves to a Docker bridge on that machine.
Docker's default pool is `172.17.0.0/12` and it has already taken six /16s
from it; it will take more as compose projects appear. D2's rationale checks
office (`172.20/16`), AWS (`10.40/16`, `10.44/24`, `10.56/16`, `172.31/16`)
and `192.168/16` — it does not consider Docker.

Scope it correctly, and do not overstate it:

- **D4 is unaffected.** SSM port-forwards resolve `172.21.10.1` on the Pi.
  The client's own routing table never sees the address. The primary remote
  access path works regardless.
- **D5 is affected.** A WireGuard tunnel into the house with
  `AllowedIPs = 172.21.0.0/16` ties with the Docker route on prefix length.
  Per-VLAN /24s (`172.21.10.0/24` etc.) would win on specificity, but it is
  brittle and the next compose project may grab another /16.

**`HOME-02`: run the same check on your own machine first.** CJ runs
Sail/Docker stacks routinely, so his laptop is likely to have the same
collision — and the laptop is the machine that would terminate the D5 tunnel.

```bash
ip -br addr; docker network ls; ip route get 172.21.10.1
```

Remedies, for CJ to choose between (infrastructure — advise only):

1. Pin Docker's pool away from `172.21` in `/etc/docker/daemon.json`
   (`default-address-pool`, e.g. base `10.210.0.0/16`, size 24). Smaller
   change, keeps D2's reasoning intact. **Requires recreating existing
   Docker networks** — it does not apply to ones already created.
2. Move the home base off `172.21`. Larger blast radius: every address in
   `HOME-NETWORK.md` changes.

### 4.2 This repo has no `CLAUDE.md`

`network-1770` has one holding its hard rule, map and conventions, and its
`SESSION-HANDOFF.md` lists it first in the read order. This repo relies on
CJ's global `~/.claude/CLAUDE.md` alone. Worth adding for parity — ask before
creating it.

---

## 5. What only `HOME-02` can answer

These gate decisions in the design record. All are read-only observations;
none requires changing anything.

| # | Question | How | Gates |
|---|---|---|---|
| Q1 | Is the WAN a real public IPv4, or CGNAT? | Compare the UX7's WAN address against what a public IP checker reports. `100.64.0.0/10` on the WAN interface is CGNAT outright; a private address means double NAT behind the ISP box. | **D5 entirely.** WireGuard + UniFi DDNS is useless behind CGNAT. §15 Open, item 1. |
| Q2 | UX7 firmware / Network application version | UniFi OS → Settings → System | §2 requires Network ≥ 9.0.108. The zone-based firewall in §6 does not exist below 9.0. |
| Q3 | Does the TP-Link pass 802.1Q tags? | §4's test, from the Pi: `ip -br addr show eth0.10` then `ping -c3 172.21.10.1` | **D3 and the entire Pi-on-Mgmt design.** §4 has the full fallback ladder. |
| Q4 | TP-Link model number | The label on the box | Determines whether Q3's answer was predictable. Record it in §2. |
| Q5 | Does this 3B+ support USB mass-storage boot? | Board revision; the 3B+ does without the OTP fuse the plain 3B needs — confirm on the actual board | §8.3, §15 Open. A $20 SSD removes the SD-card failure mode. |
| Q6 | Is the ISP device bridgeable? | ISP box's own UI | §2 WAN row. Double NAT is survivable; note it either way. |

---

## 6. Build order

`HOME-NETWORK.md` §15 is authoritative. The sequencing that is **not
negotiable**:

1. **UX7 first**, firmware current, before anything else exists.
2. **Networks, SSIDs and the LAN port profile**, verified working.
3. **The §4 tag test, before anything depends on the tagged path.** If the
   switch strips tags, D3 is dead and the fallback in §4 step 3 applies — the
   Pi sits untagged on AV and still does its job, because SSM only needs
   internet. Find this out before building on it, not after.
4. **Pi Wi-Fi standby verified before touching the Pi's wired config** from a
   session on the Pi. §8.2 says this explicitly and it is the difference
   between a recoverable mistake and a trip to the house with a keyboard.
5. **Zone-based firewall last, and only once everything else is confirmed.**
   Enabling it is **one-way** — it cannot be undone. Blocking the Gateway
   zone breaks DHCP and DNS for that network immediately, with no warning.
6. AWS stack → SSM activation → `connect.sh` entries → backup bundle →
   restore rehearsal.

Step 5 is the one that can cost a factory reset. Do not let it run ahead of
steps 1–4 for convenience.

---

## 7. Gotchas that will cost time

From `HOME-NETWORK.md` §12, the ones most likely to bite during the build:

- **mDNS is both-ends or nothing.** Main **and** AV, or casting to the TV
  fails silently. The office lost real time to this.
- **Wait ~45 s after any cable move** before calling a port dead. The UX7
  runs MSTP even though the unmanaged switch does not.
- **Never have exactly one way in.** The Pi's Wi-Fi standby and the UX7
  fallbacks exist only for this.
- **Reservations are not leases.** A fixed IP set on the client never appears
  in the UX7's lease table.
- **`curl -k` against the UX7 on its LAN address is correct**, not a
  shortcut. Self-signed certificate.
- **The automation admin must be local-only with no MFA.** A Ubiquiti SSO
  account fails the API login with an HTTP 200 that is not a session.
- **Change one thing at a time** on anything touching a management address or
  VLAN. The office bricked a switch's reachability by changing both at once.
- **A `.unf` carries Wi-Fi passphrases.** Never commit one. `.gitignore`
  already blocks `*.unf`; `/var/backups/home` is mode 700 for the same reason.

---

## 8. Report back to `HOME-01`

So `HOME-NETWORK.md` can be brought from plan to record:

- The `mi-…` SSM node id → §9.1 and the two `tunnel.env` lines in §9.2, which
  currently read `mi-REPLACE`.
- Q1–Q6 from §5, each as a one-line answer.
- The §4 tag test result, and which branch of the fallback ladder was taken.
- Any step in §8/§10.3 where the documented command did not work as written.
  That document has never been executed; it is a careful plan, not a tested
  runbook. Divergences are expected and they belong in the record.
- The §2.1 table, filled in, once `HOME-02` knows its own machine.

Tick §15 boxes only for work actually completed, per §16.
