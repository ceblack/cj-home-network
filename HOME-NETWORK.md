# Black family home network — handoff

Design record for the home build, written 2026-10-31. **Nothing below is built
yet.** It is the target state, the decisions behind it, and the commands that
get there.

Modelled on `~/Code/network-1770` (Oscillas office, OSC-ER-001 / ES-24-Lite /
UAP-AC-LR / Mac mini management box / AWS WireGuard hub). The conventions carry
over. Most of the machinery does not: one appliance is the router, the switch
is unmanaged, and there is no EdgeOS, no separate controller, no Lab, no IoT
VLAN, no NAS, no cameras, no EC2.

`~/.claude/CLAUDE.md` holds the principle — reads are free, a write to a named
system needs a named person. This repository's two columns:

- **CJ's:** the UX7 (`BF-UX-001`), the ISP router and ONT, and every AWS write.
  Produce paste-ready commands; he runs them and reports back.
- **Not his:** the Pi (`bf-home-001`). It is a box being built from scratch that
  holds nothing irreplaceable (§14), so act on it; nothing on it is production.
- **No CloudFormation `deploy --execute`.** §10.1 deliberately uses
  `--no-execute-changeset`; CJ reviews and executes the change set.
- **Secrets never enter git** — the UniFi local-admin password, `.unf` files,
  anything matching `*.env`. `.gitignore` already covers `.ssh/`, `*.unf`,
  `*.unifi`, `backups/` and `*.env`. Do not weaken it.

What changed and when is in `CHANGELOG.md`.

**Git.** The remote is `git@github.com:ceblack/cj-home-network.git` over SSH,
switched from HTTPS because a deploy key only authenticates over SSH. The key is
`.ssh/deploy_key` (4096-bit RSA, `cj@OSCILLAS-0010`); `.gitignore` blocks
`.ssh/`, so **git does not carry it** — it exists only on `OSCILLAS-0010`, and
`core.sshCommand` is local config that does not survive a clone either. Cloning
over SSH with CJ's own GitHub key needs nothing further. To use the deploy key
instead, copy it out of band, `chmod 600` it, and
`git config --local core.sshCommand 'ssh -i .ssh/deploy_key -o IdentitiesOnly=yes'`.
Commit and push only when CJ asks.

**AWS.** `black-family` is account `883357539412`, reached by chaining off the
`oscillas` SSO profile through `OrganizationAccountAccessRole`, region
`us-east-1`. Log in with `aws sso login --sso-session default-sso`, then confirm
with `aws sts get-caller-identity --profile black-family`.

---

## 1. Decisions

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Networks | Mgmt (10), Main (20), AV/Accessory (30), Guest (50) | Four, keeping the office VLAN IDs so the two networks read alike. IoT and Lab are deliberately absent — nothing at home justifies them, and the TV is the whole of "untrusted device". |
| D2 | Address space | `172.21.<VLAN>.0/24` | Third octet = VLAN ID, same convention as the office. `172.21` overlaps nothing CJ connects to: office `172.20/16`, AWS `10.40/16`, `10.44/24`, `10.56/16`, `172.31/16`. Avoiding `192.168/16` means hotel and café Wi-Fi never collides with a tunnel into the house. |
| D3 | Where the tagged VLANs live | UX7 LAN port native = AV (30), tagged = Mgmt (10) only. Main and Guest are wireless-only. | Only two wired devices exist. The TV cannot tag, so it gets the untagged network; the Pi can tag, so it gets Mgmt. Carrying Main and Guest across an unmanaged switch buys nothing and risks the one thing that can go wrong here (§4). |
| D4 | Remote access | The Pi as an AWS SSM hybrid managed node, reached with `connect.sh` | No inbound WAN port, no DDNS, no EC2, no WireGuard to maintain, rotating credentials instead of keys on disk. It reuses the office's `connect.sh` muscle memory exactly. The AWS Advanced Instances tier — the thing that used to make this cost ~$5/month — was removed 2026-06-30. |
| D5 | Remote-access fallback | WireGuard VPN server on the UX7 + UniFi DDNS, armed but idle | D4 inverts the office's D1: at the office the tunnel lives on the router *because* the management box must not be in the availability path. Here the Pi is the only box that can run an SSM agent, so if it dies the primary path dies with it. The UX7 can serve WireGuard natively; standing it up now, with DDNS, means the fallback is already proven when it is needed. Requires a real public IPv4 — check for CGNAT before relying on it. |
| D6 | Break-glass | UniFi Teleport | Zero configuration, no port, no DDNS, works from the phone. Costs nothing but routes through Ubiquiti's cloud and needs console remote access enabled — a trade CJ should make knowingly, not by default. |
| D7 | Pi OS | Raspberry Pi OS Lite 64-bit (Debian 13) | 1 GB of RAM on a 3B+. No snapd, vendor kernel and firmware for this board, and the SSM agent and AWS CLI both ship usable arm64 builds. Ubuntu Server 24.04 LTS arm64 works on a 3B+ if shop consistency matters more; it is heavier and pulls snaps. |
| D8 | Backup credentials | SSM hybrid activation, no access keys | Same as the office. The agent keeps the role's rotating credentials in `/root/.aws/credentials`; the role may write and read only its own S3 prefix and cannot delete anything. |
| D9 | IPv6 | Off | Same posture as the office. Enabling it means mirroring every firewall policy in v6 on day one or quietly having none. Revisit as a decision, not as a default. |

---

## 2. Hardware

| Role | Model | Address | Notes |
|---|---|---|---|
| Router + AP + controller | UniFi Express 7, `BF-UX-001` | `.1` on every VLAN; UI on `172.21.10.1` | Tri-band Wi-Fi 7 (2.4/5/6 GHz, 2×2 per band). **1 × 10 GbE WAN, 1 × 2.5 GbE LAN.** USB-C 5 V/5 A, 22 W max, no PoE in or out. Runs UniFi OS with the Network application on-box; 30+ managed devices, 300+ clients. Needs Network 9.0.108 or later. |
| Switch | TP-Link unmanaged, `BF-SW-001` | none | No management plane, no address, nothing to back up. Its one job is to split the single LAN port between the Pi and the TV. See §4 for the one thing it has to do correctly. |
| Management appliance | Raspberry Pi 3B+, `bf-home-001` | `172.21.10.5` (UX7 fixed-IP reservation) | BCM2837B0, 4 × A53 1.4 GHz, 1 GB RAM. Gigabit Ethernet **over USB 2.0 — ~300 Mbps ceiling**, irrelevant for this role. 2.4/5 GHz 802.11ac onboard, used as the standby path (§8.2). microSD boot; see §8.3. |
| TV | `BF-TV-001` | `172.21.30.10` (reservation) | Wired to the switch, untagged, AV network. Cannot tag — which is what fixes the LAN port's native network (D3). |
| WAN | ISP router/ONT | per ISP | If the ISP device cannot be bridged, put it in passthrough/bridge if offered, or accept double NAT. Note whether the WAN address is public and static; D5 depends on it. |

Not present, and deliberately: a managed switch, a separate AP, a separate
controller host, PoE, a NAS, an EC2 hub.

---

## 3. Topology

```
            Internet
                │
        ┌───────┴────────┐
        │   ISP ONT /    │
        │   modem        │
        └───────┬────────┘
                │ 10 GbE WAN
        ┌───────┴────────────────────────────┐
        │  BF-UX-001  (UniFi Express 7)      │
        │  router + AP + UniFi OS console    │
        │  .1 on VLANs 10 / 20 / 30 / 50     │
        └───────┬────────────────────────────┘
                │ 2.5 GbE LAN
                │   native: VLAN 30 (AV)
                │   tagged: VLAN 10 (Mgmt)
        ┌───────┴────────┐
        │  BF-SW-001     │  unmanaged TP-Link
        └───┬────────┬───┘
            │        │
   untagged │        │ tagged VLAN 10 on eth0.10
     VLAN30 │        │ (plus Wi-Fi standby on Main)
        ┌───┴───┐  ┌─┴──────────────┐
        │BF-TV- │  │ bf-home-001    │
        │001    │  │ Raspberry Pi   │
        └───────┘  │ 3B+            │
                   └────────────────┘

Wireless only: Main (20) and Guest (50), from the UX7's own radios.
```

Everything else in the house is wireless. There is no wired path to Main or
Guest, and nothing needs one today.

---

## 4. The one real risk: tags across an unmanaged switch

The UX7 has a single LAN port. Two wired devices need two different networks.
That forces 802.1Q tagging across an unmanaged switch, and **unmanaged switches
are not required to pass tagged frames**. Most modern ones do — they are
transparent to the 802.1Q header — but cheap or old models either strip the tag
or drop the frame because the 1522-byte tagged frame exceeds their buffer
assumptions. It is model-specific and not something a datasheet usually states.

Test it before building anything on top of it:

```bash
# On the Pi, once the VLAN interface exists (§8.2):
ip -br addr show eth0.10          # expect an address in 172.21.10.0/24
ping -c3 172.21.10.1              # expect replies from the UX7's Mgmt gateway
```

If the tagged path does not come up:

1. Confirm the UX7 LAN port is configured as D3 says (native AV, VLAN 10 tagged).
2. Plug the Pi straight into the UX7 LAN port, bypassing the switch, and retest.
   If it works there and not through the switch, the switch is the problem.
3. Fall back: leave the Pi untagged on AV with a reservation, and reach Mgmt
   another way. The Pi only needs *internet* for SSM, not Mgmt — so this
   degrades management convenience, not remote access.
4. The durable fix is a cheap VLAN-capable switch (TP-Link TL-SG105E or
   similar, ~$25). That is the one upgrade this design would ever need.

Carried from the office: **wait ~45 seconds after any cable move before calling
a port dead.** Spanning tree cost real diagnostic time during the office build.
The unmanaged switch here does not run MSTP, but the UX7 does.

---

## 5. Addressing

`172.21.0.0/16`, third octet = VLAN ID.

| VLAN | Name | Subnet | Gateway | DHCP pool |
|---|---|---|---|---|
| 10 | Mgmt | `172.21.10.0/24` | `.1` | `.100–.149` |
| 20 | Main | `172.21.20.0/24` | `.1` | `.100–.199` |
| 30 | AV / Accessory | `172.21.30.0/24` | `.1` | `.100–.149` |
| 50 | Guest | `172.21.50.0/24` | `.1` | `.100–.199` |

`.2`–`.99` is static infrastructure space in every subnet, as at the office.
Nothing outside that range is ever referenced by a script.

### Reservations

| Host | Address | Set where |
|---|---|---|
| `BF-UX-001` | `172.21.10.1` (and `.1` on 20/30/50) | gateway, by definition |
| `bf-home-001` | `172.21.10.5` | UX7 fixed IP, keyed on the Pi's `eth0` MAC |
| `BF-TV-001` | `172.21.30.10` | UX7 fixed IP |

The Pi takes a reservation rather than a static address for the same reason the
office management box does: a reservation is one leaf to change from the UX7 UI
when hardware is swapped, and it is reversible remotely. A VLAN subinterface
inherits the parent NIC's MAC, so the reservation is made against the Pi's
`eth0` MAC and simply observed on VLAN 10.

`172.34.0.0` is public IP space. The office nearly shipped it. Do not
reintroduce it here either.

---

## 6. Firewall

UniFi Network 9.0+ uses a zone-based firewall. The intent below is the office's
direction of trust, unchanged: **Main initiates to everything; nothing
initiates back into Main.**

| From ↓ To → | Mgmt | Main | AV | Guest | Gateway | Internet |
|---|---|---|---|---|---|---|
| Mgmt | — | allow | allow | allow | allow | allow |
| Main | allow | — | allow | drop | allow | allow |
| AV | drop | est/rel only | allow | drop | DHCP, DNS, NTP, ICMP | allow |
| Guest | drop | drop | drop | isolated | DHCP, DNS | allow |
| VPN clients | allow | allow | allow | drop | allow | — (split tunnel) |

Notes on the matrix:

- **Main → Mgmt is open**, where the office restricts it to an `ADMIN_HOSTS`
  address group. At home CJ's laptop is the only thing on Main that matters and
  it moves between wired and wireless. If that ever stops being true, the office
  pattern — an address group of exactly the admin machines, plus reservations
  for them — is the tightening to copy.
- **AV → Main is drop, not est/rel-by-rule.** Zone policies are stateful, so
  "Main → AV allow" already returns the traffic. There is nothing to write.
- **Guest** uses UniFi's Guest Network type, which brings client isolation with
  it. Do not apply egress port blocks to Guest; visitors legitimately use SSH,
  RDP and VPNs. (The office learned this on its IoT VLAN, which does block
  those — there is no IoT network here to apply it to.)
- **Gateway zone.** Keep Main → Gateway allowed or you lock yourself out of the
  UX7 UI from the only network you use. AV and Guest get DHCP and DNS and
  nothing else.

### Zone mapping

| Network | Zone |
|---|---|
| Main (20) | `Internal` (built-in) |
| Mgmt (10) | custom zone `Mgmt` |
| AV (30) | custom zone `AV` |
| Guest (50) | `Hotspot` (built-in; automatic for Guest-type networks) |
| WireGuard / Teleport clients | `VPN` (built-in) |

Traffic between networks inside one zone is allowed; traffic between zones is
denied unless a policy allows it. That is why Mgmt and AV get their own zones
and Main does not need one.

Two cautions, both from the vendor's own documentation and worth respecting:

- Enabling the zone-based firewall is **one-way**. Existing rules migrate
  automatically; there is no going back.
- Moving a network between zones, or blocking the Gateway zone, breaks DHCP and
  DNS for that network immediately.

**No inbound WAN exceptions.** D4 needs none — the Pi dials out. D5, if armed,
is the single exception: one UDP port for the WireGuard server.

---

## 7. Wireless

Four networks, three of which are wireless. The UX7 supports 4 BSSIDs per
radio, so there is headroom.

| SSID | Network | Bands | Security | Notes |
|---|---|---|---|---|
| `Black` | Main (20) | 5 + 6 GHz | WPA2/WPA3 | Phones, laptops, everything personal |
| `Black-AV` | AV (30) | 2.4 + 5 GHz | WPA2/WPA3 | Wireless AV and accessories; the TV is wired but speakers and the like land here |
| `Black-Guest` | Guest (50) | 2.4 + 5 GHz | WPA2/WPA3 | UniFi Guest Network type |
| `Black-Mgmt` | Mgmt (10) | 5 GHz | WPA3 | Optional. Only purpose is the Pi's standby path (§8.2) and a phone when the wired side is broken. Hide it. |

**mDNS.** Enable Multicast DNS on Main and AV, and only those. Casting and
AirPlay from a phone on Main to the TV on AV do not work without it. This is the
single most expensive lesson from the office build — Chromecast discovery failed
there until an explicit rule admitted UDP 5353 on both networks. UniFi handles
it with a per-network toggle instead of a firewall rule, but the requirement is
identical: **both ends, or neither works.**

Leave Guest and Mgmt without mDNS.

---

## 8. The Pi — `bf-home-001`

### 8.1 Base build

Raspberry Pi OS Lite 64-bit, written with Raspberry Pi Imager. Set in Imager's
advanced options, before first boot: hostname `bf-home-001`, user `cj`, SSH
enabled with a public key, **password authentication off**, locale UTC.

```bash
sudo timedatectl set-timezone Etc/UTC
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y unattended-upgrades curl unzip jq
sudo dpkg-reconfigure -plow unattended-upgrades
```

AWS CLI v2 (arm64; the Debian-packaged v1 is not what the backup job expects):

```bash
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o /tmp/awscliv2.zip
cd /tmp && unzip -q awscliv2.zip && sudo ./aws/install && rm -rf /tmp/aws /tmp/awscliv2.zip
aws --version
```

Time comes from the UX7 (`172.21.10.1`); the Mgmt network has unrestricted
access to the gateway, so nothing special is needed.

### 8.2 Network configuration

Raspberry Pi OS uses NetworkManager. Three connections: the parent with no
address, the tagged Mgmt VLAN, and Wi-Fi as standby.

```bash
# Parent: up, but no IP on either family. The VLAN rides on it.
sudo nmcli con mod "Wired connection 1" ipv4.method disabled ipv6.method disabled \
  connection.autoconnect yes

# Tagged Mgmt. DHCP, so the UX7 reservation decides the address (§5).
sudo nmcli con add type vlan con-name mgmt10 ifname eth0.10 dev eth0 id 10 \
  ipv4.method auto ipv4.route-metric 100 ipv6.method disabled connection.autoconnect yes

# Standby: Wi-Fi on Main, higher metric so it is never the preferred route.
sudo nmcli dev wifi connect 'Black' --ask ifname wlan0
sudo nmcli con mod 'Black' ipv4.route-metric 600 ipv6.method disabled connection.autoconnect yes

sudo nmcli con up mgmt10
ip -br addr; ip route
```

Why the standby matters: SSM is **outbound only**. The Pi stays reachable over
any path that has internet. If the tagged VLAN breaks — a switch that strips
tags, a mis-saved UX7 port profile, a cable — the Pi comes up on Wi-Fi, SSM
reconnects, and CJ can fix the wired side from a shell. The office has no
equivalent: it removed its unfiltered fallback port and a firewall mistake there
now means a factory reset.

Do not change the Pi's network configuration from a session running on the Pi
without the Wi-Fi standby verified first.

### 8.3 Surviving the SD card

A 3B+ running 24/7 off a microSD card will eventually corrupt it. Three cheap
mitigations, in order of value:

1. **Boot from a USB SSD instead.** The 3B+ supports USB mass-storage boot
   without the one-time OTP fuse the plain 3B needs — confirm on the board
   before committing to it. A $20 SSD removes the failure mode entirely.
2. **Cap the journal.** `sudo mkdir -p /etc/systemd/journald.conf.d` and a
   drop-in with `SystemMaxUse=50M`, `RuntimeMaxUse=16M`.
3. **Shrink swap.** `CONF_SWAPSIZE=256` in `/etc/dphys-swapfile`, then
   `sudo systemctl restart dphys-swapfile`. Do not disable it outright on 1 GB.

A UPS shared with the UX7 is worth more than all three. Unclean power-off is how
SD cards actually die.

---

## 9. Remote access

### 9.1 Primary — SSM hybrid managed node (D4)

The Pi registers itself as an SSM managed node in the `black-family` account.
No inbound ports, no DDNS, no tunnel daemon. Session Manager gives a shell;
port forwarding to a remote host gives the UX7 UI through the Pi.

Install the agent (arm64 `.deb`, not the snap the office box uses):

```bash
curl -fsSL "https://s3.us-east-1.amazonaws.com/amazon-ssm-agent/latest/debian_arm64/amazon-ssm-agent.deb" \
  -o /tmp/amazon-ssm-agent.deb
sudo dpkg -i /tmp/amazon-ssm-agent.deb
sudo systemctl stop amazon-ssm-agent
```

Create the activation from the laptop (single-use, valid 24 h; the IAM role
comes from the stack in §10.1):

```bash
aws ssm create-activation --profile black-family --region us-east-1 \
  --default-instance-name bf-home-001 --iam-role home-mgmt-bf-home-001 \
  --registration-limit 1 --query '{Id:ActivationId,Code:ActivationCode}' --output table
```

Register on the Pi, then confirm from the laptop:

```bash
# Pi
sudo amazon-ssm-agent -register -y -region us-east-1 -id <Id> -code <Code>
sudo systemctl enable --now amazon-ssm-agent
sudo ls -l /root/.aws/credentials        # rotating role credentials appear within ~30 s

# Laptop
aws ssm describe-instance-information --profile black-family --region us-east-1 \
  --query 'InstanceInformationList[].{Id:InstanceId,Name:ComputerName,Ping:PingStatus}' --output table
```

The node id is `mi-…` and is stable until the Pi is re-registered. Record it
here when it exists.

**Cost.** The Advanced Instances tier, which used to be required for Session
Manager on non-EC2 nodes and cost about $5/month, was removed on 2026-06-30;
hybrid nodes now register at no per-instance charge. From 2026-09-30, hybrid and
multicloud nodes bill $0.05 per Session Manager session and $0.002 per Run
Command invocation. Each `connect.sh` tunnel is one session, so a handful of
tunnels a day is pennies a month — but it is no longer strictly free, and
leaving tunnels up all day is the thing that would change that.

### 9.2 `connect.sh` entries

Same format and same file as the office, `~/Code/cj-local/gen-tools/connect/tunnel.env`.
Local ports use a `151xx` block so nothing collides with the office's `150xx`.

**Insert these inside the `TUNNELS=( … )` array**, above the closing `)`.
Appending after the `)` makes bash execute them when `connect.sh` sources the
file.

```
  # bf-home-001 itself. The forward target is the Pi's own loopback.
  "homepi|home-pi-ssh|black-family|us-east-1|mi-REPLACE|127.0.0.1|15120|22|tcp"

  # BF-UX-001 UniFi OS / Network UI, forwarded through the Pi (self-signed cert).
  "homeux|home-unifi-gui|black-family|us-east-1|mi-REPLACE|172.21.10.1|15121|443|tcp"
```

The office entries resolve their instance from a CloudFormation stack output
(`stack:office-link-hub`) so instance replacement never edits the file. There is
no stack here — the `mi-…` id goes in literally and changes only on
re-registration (§14).

`tunnel.env` is gitignored in `cj-local`. Back it up with the rest of that repo's
untracked files; it is not reconstructible from this document alone.

### 9.3 Fallbacks

| Path | When to use | Cost of having it |
|---|---|---|
| WireGuard server on the UX7 + UniFi DDNS (D5) | The Pi is dead, or you want full L3 into the house from a laptop | One inbound UDP port on WAN, one DDNS hostname. **Useless behind CGNAT** — verify the WAN address is a real public IPv4 before building it. |
| UniFi Teleport (D6) | Phone, away from home, nothing else works | Needs console remote access and Ubiquiti cloud in the path. Zero configuration otherwise. |

Build D5 while the network is healthy and test it once. A fallback first
exercised during an outage is not a fallback.

---

## 10. Backups — UX7 config to S3

The Pi pulls a UniFi Network backup from the UX7 nightly and pushes it to a
versioned bucket in the `black-family` account. This is the whole reason the Pi
has AWS credentials.

What the backup covers: the Network application's configuration — networks,
VLANs, firewall zones and policies, SSIDs, DHCP reservations, device adoption.
That is the state that took work to build and is painful to rebuild from memory.

What it does not cover: UniFi OS console-level settings (the `.unifi` system
backup). On a UX7 the console and the Network app are the same box, but they
back up separately. The Network `.unf` is the automatable one; the console
backup is a punch-list item (§15).

### 10.1 AWS side

Stack `home-backups`, account `black-family`, `us-east-1`. One versioned,
encrypted, private bucket and one IAM role the Pi assumes as a hybrid node. The
role can write and read only its own prefix and **cannot delete anything** — a
compromised Pi cannot destroy the backup history.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: >-
  home-backups: one versioned, encrypted, private S3 bucket for BF-UX-001
  config backups pushed nightly by bf-home-001, plus the IAM role the Pi
  assumes as an SSM hybrid managed node (rotating credentials, no access keys
  on disk). The role can write and list only its own prefix in this bucket,
  publish backup alerts, and emit one heartbeat metric.
  Account black-family, us-east-1.

Parameters:
  ApplicationTag:
    Type: String
    Default: home-backups
  HostPrefix:
    Type: String
    Default: bf-home-001
    Description: Object key prefix the Pi writes under (its hostname). Also used in the role name.
  NoncurrentVersionDays:
    Type: Number
    Default: 90
    Description: How long superseded object versions are kept before expiry.
  AlarmEmail:
    Type: String
    Default: cj@oscillas.com
    Description: Where backup failures and the missing-heartbeat alarm are sent.

Resources:

  BackupBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub black-family-net-backups-${AWS::AccountId}
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
            BucketKeyEnabled: true
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      OwnershipControls:
        Rules:
          - ObjectOwnership: BucketOwnerEnforced
      LifecycleConfiguration:
        Rules:
          - Id: expire-noncurrent
            Status: Enabled
            NoncurrentVersionExpiration:
              NoncurrentDays: !Ref NoncurrentVersionDays
          - Id: abort-incomplete-multipart
            Status: Enabled
            AbortIncompleteMultipartUpload:
              DaysAfterInitiation: 7
      Tags:
        - { Key: application, Value: !Ref ApplicationTag }

  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref BackupBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: DenyInsecureTransport
            Effect: Deny
            Principal: "*"
            Action: s3:*
            Resource:
              - !GetAtt BackupBucket.Arn
              - !Sub ${BackupBucket.Arn}/*
            Condition:
              Bool: { aws:SecureTransport: "false" }

  AlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      DisplayName: home-backups
      Subscription:
        - Protocol: email
          Endpoint: !Ref AlarmEmail

  HybridNodeRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub home-mgmt-${HostPrefix}
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: ssm.amazonaws.com }
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                aws:SourceAccount: !Ref AWS::AccountId
      ManagedPolicyArns:
        - !Sub arn:${AWS::Partition}:iam::aws:policy/AmazonSSMManagedInstanceCore
      Policies:
        - PolicyName: home-backups-write-own-prefix
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ListOwnPrefix
                Effect: Allow
                Action: s3:ListBucket
                Resource: !GetAtt BackupBucket.Arn
                Condition:
                  StringLike:
                    s3:prefix: [!Sub "${HostPrefix}/*", !Sub "${HostPrefix}"]
              - Sid: PutAndReadOwnPrefix
                Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetObject
                Resource: !Sub ${BackupBucket.Arn}/${HostPrefix}/*
              # no s3:DeleteObject, no *Version* actions: the Pi cannot destroy history
              - Sid: PublishAlerts
                Effect: Allow
                Action: sns:Publish
                Resource: !Ref AlertTopic
              - Sid: Heartbeat
                Effect: Allow
                Action: cloudwatch:PutMetricData
                Resource: "*"
                Condition:
                  StringEquals:
                    cloudwatch:namespace: HomeBackup
      Tags:
        - { Key: application, Value: !Ref ApplicationTag }

  NoBackupAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub home-backup-missing-${HostPrefix}
      AlarmDescription: No successful UX7 backup heartbeat in 48 hours.
      Namespace: HomeBackup
      MetricName: BackupSuccess
      Dimensions:
        - { Name: Host, Value: !Ref HostPrefix }
      Statistic: Sum
      Period: 86400
      EvaluationPeriods: 2
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      TreatMissingData: breaching
      AlarmActions: [!Ref AlertTopic]
      OKActions: [!Ref AlertTopic]

Outputs:
  BucketName:
    Description: BACKUP_BUCKET for /etc/home-backup/env on the Pi.
    Value: !Ref BackupBucket
  HybridRoleName:
    Description: --iam-role for `aws ssm create-activation`.
    Value: !Ref HybridNodeRole
  AlertTopicArn:
    Description: ALERT_TOPIC for /etc/home-backup/env on the Pi.
    Value: !Ref AlertTopic
  RestoreHint:
    Value: !Sub aws s3 ls s3://${BackupBucket}/${HostPrefix}/unifi/ --profile black-family
```

Deploy with a change set, never a blind `deploy`:

```bash
aws cloudformation deploy --stack-name home-backups --template-file home-backups.yaml \
  --capabilities CAPABILITY_NAMED_IAM --profile black-family --region us-east-1 --no-execute-changeset
# review the change set URL it prints, then execute it
```

Confirm the SNS subscription from the email that arrives, or no alert will ever
reach anyone.

The two alerting paths are deliberately different in kind: the unit's
`OnFailure` catches a backup that ran and failed, and the heartbeat alarm
catches a Pi that stopped running backups at all. The office learned this
distinction the hard way with its tunnel alarm — a thing that is silently dead
produces no failures to notice.

### 10.2 UniFi side

Create a **local-only admin** on the UX7 for the job. A Ubiquiti SSO account
will not work: those carry MFA and the API login will fail with an HTTP 200 that
is not a session.

UniFi OS → Settings → Admins & Users → Add Admin:

- **Restrict to Local Access Only** — on. This is the setting that matters.
- Role: full management of the Network application.
- Two-factor: off. (It is a local-only account whose password lives in a
  password manager and in one root-owned file on the Pi.)
- Username `backup`.

The office uses exactly this arrangement against UniFi OS Server, and the API
flow below is the one proven there.

### 10.3 Pi side

Four files. `/etc/home-backup/` is root-owned, mode 700; `env` is mode 600 and
holds the UniFi password.

**`/etc/home-backup/env`**

```bash
# /etc/home-backup/env  (mode 600, root)
# Quote every value: KEY='value'. Passwords with $ ! ) } # or spaces break unquoted sourcing.
BACKUP_ROOT='/var/backups/home'
AWS_REGION='us-east-1'
BACKUP_BUCKET=''            # home-backups stack output BucketName; empty = local only
ALERT_TOPIC=''              # home-backups stack output AlertTopicArn; empty = no alert mail
UX_URL='https://172.21.10.1'
UX_SITE='default'
UX_USER=''                  # local-only admin (no UI account, no MFA)
UX_PASS=''                  # leave both empty to skip the backup and log why
```

**`/usr/local/sbin/home-backup-unifi.sh`**

```bash
#!/usr/bin/env bash
# home-backup-unifi.sh -- pull a UniFi Network settings backup from BF-UX-001.
#
# This is what the UI's Download button does, against the UniFi OS console:
#   POST /api/auth/login  {username,password}        -> cookie TOKEN + header x-csrf-token
#   POST /proxy/network/api/s/<site>/cmd/backup  {"cmd":"backup","days":0}   (settings only)
#   GET  /proxy/network<url from the response>       -> the .unf
#   POST /api/auth/logout
# Credentials come from /etc/home-backup/env (root, 600); the password never
# appears on a command line. Keeps the last 30 locally.
set -euo pipefail
# Refuse to source a malformed env quietly: bash would otherwise echo the offending
# line -- and any secret on it -- into the journal.
bash -n /etc/home-backup/env 2>/dev/null || { printf '<3>%s\n' "home-backup: /etc/home-backup/env has a shell syntax error (quote values: KEY='value'); nothing run" >&2; exit 1; }
. /etc/home-backup/env
log() { printf '%s\n' "$*"; }
err() { if [ -n "${JOURNAL_STREAM:-}" ]; then printf '<3>%s\n' "$*" >&2; else printf 'ERROR: %s\n' "$*" >&2; fi; }
fail() { err "unifi: $*"; exit 1; }
[ -n "${BACKUP_ROOT:-}" ] || fail "BACKUP_ROOT empty in /etc/home-backup/env - refusing to run at /"

DEST="$BACKUP_ROOT/unifi"
TS=$(date -u +%Y%m%d-%H%M%S)
umask 077
mkdir -p "$DEST"

if [ -z "${UX_USER:-}" ] || [ -z "${UX_PASS:-}" ]; then
  fail "UX_USER/UX_PASS not set in /etc/home-backup/env"
fi

URL="${UX_URL:-https://172.21.10.1}"; SITE="${UX_SITE:-default}"
WORK=$(mktemp -d); trap 'rm -rf "$WORK"' EXIT
CURL="curl -sk --max-time 300"      # -k: the console serves its own certificate on the LAN address
export UX_USER UX_PASS
python3 -c 'import json,os,sys; sys.stdout.write(json.dumps({"username":os.environ["UX_USER"],"password":os.environ["UX_PASS"],"rememberMe":False}))' > "$WORK/login.json" \
  || fail "could not build the login body"
unset UX_PASS

$CURL -c "$WORK/jar" -D "$WORK/login.h" -o "$WORK/login.body" -H 'Content-Type: application/json' \
  -d @"$WORK/login.json" "$URL/api/auth/login" || fail "login request to $URL failed (unreachable?)"
rm -f "$WORK/login.json"
CODE=$(head -1 "$WORK/login.h" | awk '{print $2}')
[ "$CODE" = "200" ] || fail "login to $URL returned HTTP $CODE (bad credentials, or an SSO account with MFA?)"
CSRF=$(awk 'tolower($1)=="x-csrf-token:"{gsub(/\r/,"",$2); print $2}' "$WORK/login.h" | tail -1)
[ -n "$CSRF" ] || fail "login succeeded but no x-csrf-token header"
printf 'x-csrf-token: %s\n' "$CSRF" > "$WORK/csrf.h"

RESP=$($CURL -b "$WORK/jar" -H @"$WORK/csrf.h" -H 'Content-Type: application/json' \
         -d '{"cmd":"backup","days":0}' "$URL/proxy/network/api/s/$SITE/cmd/backup") \
  || fail "cmd/backup request failed"
DL=$(printf '%s' "$RESP" | python3 -c 'import json,sys
d=json.load(sys.stdin)
if d.get("meta",{}).get("rc")!="ok": sys.exit(1)
print(d["data"][0]["url"])' 2>/dev/null) || fail "cmd/backup did not return ok: $(printf '%s' "$RESP" | head -c 200)"

OUT="$DEST/$TS-network.unf"
$CURL -b "$WORK/jar" -o "$OUT" "$URL/proxy/network$DL" || { rm -f "$OUT"; fail "download of $DL failed"; }
$CURL -b "$WORK/jar" -H @"$WORK/csrf.h" -X POST -o /dev/null "$URL/api/auth/logout" || log "unifi: logout failed (harmless)"

SIZE=$(stat -c %s "$OUT")
[ "$SIZE" -gt 5120 ] || { rm -f "$OUT"; fail "network backup only $SIZE bytes - refusing to keep it"; }
PREV=$(find "$DEST" -maxdepth 1 -type f -name '*-network.unf' ! -name "$TS-network.unf" -printf '%T@ %p\n' | sort -rn | head -1 | cut -d' ' -f2- || true)
if [ -n "$PREV" ] && cmp -s "$PREV" "$OUT"; then CHANGED="unchanged since $(basename "$PREV")"; else CHANGED="changed"; fi
ln -sfn "$(basename "$OUT")" "$DEST/latest-network.unf"
find "$DEST" -maxdepth 1 -type f -name '*-network.unf' -printf '%T@ %p\n' | sort -rn | tail -n +31 | cut -d' ' -f2- | xargs -r rm -f || true
log "unifi: network backup $(basename "$OUT") ($SIZE bytes, settings-only, $CHANGED) from $URL"
```

**`/usr/local/sbin/home-backup-push.sh`**

```bash
#!/usr/bin/env bash
# home-backup-push.sh -- copy /var/backups/home to the home-backups bucket, then
# emit the heartbeat the missing-backup alarm watches.
#
# Credentials: none stored here. The SSM agent keeps the role's rotating
# credentials in /root/.aws/credentials (profile "default"), and the role may
# only write under s3://<bucket>/bf-home-001/. Never --delete: S3 versioning and
# the bucket lifecycle handle retention, and a wiped local dir must not wipe the
# remote copy.
set -euo pipefail
bash -n /etc/home-backup/env 2>/dev/null || { printf '<3>%s\n' "home-backup: /etc/home-backup/env has a shell syntax error (quote values: KEY='value'); nothing run" >&2; exit 1; }
. /etc/home-backup/env
log() { printf '%s\n' "$*"; }
err() { if [ -n "${JOURNAL_STREAM:-}" ]; then printf '<3>%s\n' "$*" >&2; else printf 'ERROR: %s\n' "$*" >&2; fi; }
[ -n "${BACKUP_ROOT:-}" ] || { err "push: BACKUP_ROOT empty in /etc/home-backup/env - nothing pushed"; exit 1; }

if [ -z "${BACKUP_BUCKET:-}" ]; then
  log "push: BACKUP_BUCKET empty in /etc/home-backup/env - local-only backup, nothing pushed"
  exit 0
fi
trap 'err "push: FAILED (exit $?) syncing to s3://$BACKUP_BUCKET"' ERR
aws s3 sync "$BACKUP_ROOT/" "s3://$BACKUP_BUCKET/$(hostname)/" \
  --region "$AWS_REGION" --sse AES256 --no-progress --only-show-errors \
  --exclude '*/latest*'
N=$(aws s3 ls "s3://$BACKUP_BUCKET/$(hostname)/unifi/" --region "$AWS_REGION" | wc -l)
aws cloudwatch put-metric-data --region "$AWS_REGION" --namespace HomeBackup \
  --metric-name BackupSuccess --value 1 --dimensions "Host=$(hostname)" || log "push: heartbeat metric failed (harmless this run)"
log "push: synced $BACKUP_ROOT to s3://$BACKUP_BUCKET/$(hostname)/ ($N unifi objects)"
```

**`/etc/systemd/system/home-backup.service`**

```ini
[Unit]
Description=home backups: BF-UX-001 config export, then push to S3
After=network-online.target
Wants=network-online.target
OnFailure=home-backup-alert.service

[Service]
Type=oneshot
SyslogIdentifier=home-backup
ExecStart=/usr/local/sbin/home-backup-unifi.sh
ExecStart=/usr/local/sbin/home-backup-push.sh
```

**`/etc/systemd/system/home-backup.timer`**

```ini
[Unit]
Description=home backups nightly (08:45 UTC = 03:45 CDT / 02:45 CST)

[Timer]
OnCalendar=*-*-* 08:45:00
RandomizedDelaySec=10min
Persistent=true

[Install]
WantedBy=timers.target
```

**`/etc/systemd/system/home-backup-alert.service`**

```ini
[Unit]
Description=publish a home-backup failure to SNS

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/home-backup-alert.sh
```

**`/usr/local/sbin/home-backup-alert.sh`**

```bash
#!/usr/bin/env bash
# Called by OnFailure=. Sends the last 20 journal lines so the mail says what broke.
set -uo pipefail
. /etc/home-backup/env 2>/dev/null || exit 0
[ -n "${ALERT_TOPIC:-}" ] || exit 0
BODY=$(journalctl -u home-backup.service -n 20 --no-pager 2>/dev/null | tail -c 3000)
aws sns publish --region "$AWS_REGION" --topic-arn "$ALERT_TOPIC" \
  --subject "home-backup FAILED on $(hostname)" --message "$BODY" >/dev/null 2>&1 || true
```

Enable and prove it:

```bash
sudo chmod 700 /usr/local/sbin/home-backup-*.sh
sudo install -d -m 700 /etc/home-backup
sudo chmod 600 /etc/home-backup/env
sudo systemctl daemon-reload
sudo systemctl enable --now home-backup.timer
sudo systemctl start home-backup.service
systemctl status home-backup.service --no-pager
journalctl -u home-backup.service -n 20 --no-pager
```

`inactive (dead)` with exit 0 is success. `failed` means a step logged its
reason — the `<3>` lines are the errors.

### 10.4 What the job does, and what it refuses

- Logs in to the UX7 as the local-only admin, asks the Network application for a
  **settings-only** backup (`days:0` — no historical statistics), downloads the
  `.unf`, logs out.
- **Refuses a file under 5 KB.** A truncated or error-page "backup" is worse than
  no backup because it looks like success.
- Logs whether the file differs from the previous night, so a glance at the
  journal says whether anything on the network actually changed.
- Keeps the last 30 locally under `/var/backups/home/unifi/`, mode 700.
- Syncs to `s3://<bucket>/bf-home-001/unifi/`, **never with `--delete`**.
- Emits one `HomeBackup/BackupSuccess` datapoint, which is what the
  missing-backup alarm watches.

The `.unf` is **not encrypted and contains secrets** — Wi-Fi passphrases among
them. That is why `/var/backups/home` is mode 700, the bucket is SSE-encrypted
and private, and the file is gitignored here.

Volume: roughly 100 KB a night, about 36 MB a year before versioning. There is
no lifecycle rule expiring current objects, and at that size there does not need
to be.

### 10.5 Restore

```bash
# Laptop: find and fetch the newest
BUCKET=$(aws cloudformation describe-stacks --stack-name home-backups --profile black-family \
  --region us-east-1 --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' --output text)
aws s3 ls "s3://$BUCKET/bf-home-001/unifi/" --profile black-family
aws s3 cp "s3://$BUCKET/bf-home-001/unifi/<ts>-network.unf" . --profile black-family
```

Then: UniFi OS → Settings → System → Backups → Restore, and upload the `.unf`.
On a factory-reset UX7 the setup wizard offers "Restore from backup" as the
first step; take it there rather than setting the device up and restoring after.

A settings-only backup restores the configuration, not historical statistics.
That is the intended trade.

---

## 11. Health checks

Read-only, top down, from the laptop:

```bash
P="--profile black-family --region us-east-1"
aws ssm describe-instance-information $P \
  --query 'InstanceInformationList[].{Id:InstanceId,Ping:PingStatus}' --output table   # mi-… Online
aws cloudwatch describe-alarms $P --alarm-names home-backup-missing-bf-home-001 \
  --query 'MetricAlarms[].StateValue' --output text                                    # OK
BUCKET=$(aws cloudformation describe-stacks --stack-name home-backups $P \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' --output text)
aws s3 ls "s3://$BUCKET/bf-home-001/unifi/" $P | tail -3                                # last night's object
```

Interpretation: node Online + alarm OK + a fresh object = every layer works.
Node Online + alarm ALARM = the Pi is up but the backup is failing; read the
journal over `homepi`. Node not Online = the Pi or the house internet is down,
and the alarm will follow within 48 h.

---

## 12. Gotchas worth carrying over

From the office build. These cost real time there and most of them apply here.

- **mDNS is both-ends or nothing.** Casting silently fails until the source
  network and the destination network both have it. §7.
- **Wait ~45 s after a cable move** before concluding a port is dead.
- **Never have exactly one way in.** The office removed its unfiltered fallback
  port and now a firewall mistake costs a factory reset. The Pi's Wi-Fi standby
  (§8.2) and the UX7 fallbacks (§9.3) exist for this reason alone.
- **Reservations are not leases.** A fixed IP set on a client does not show up
  in a lease table. Look for it where it was set.
- **`172.21.0.0/16` collides with Docker on any machine of CJ's that runs
  compose stacks.** Docker's default pool is `172.17.0.0/12` and it allocates
  /16s from it as projects appear; on `OSCILLAS-0010`, `172.21.0.0/16` is
  already a bridge. D4 is unaffected (SSM port-forwards resolve the address on
  the Pi), but D5's WireGuard `AllowedIPs = 172.21.0.0/16` would tie with the
  Docker route on prefix length. Check any machine that would terminate that
  tunnel with `ip -br addr; docker network ls; ip route get 172.21.10.1`.
  Remedies, CJ's to choose between: pin Docker's pool away from `172.21` in
  `/etc/docker/daemon.json` (`default-address-pool`, smaller change, but it does
  not apply to networks already created — they must be recreated), or move the
  home base off `172.21` (larger blast radius: every address in this document
  changes).
- **Self-signed certificates everywhere.** `curl -k` against the UX7 on its LAN
  address is correct, not a shortcut.
- **One local-only admin, no MFA, for automation.** An SSO account with MFA
  cannot drive the API.
- **Change one thing at a time** on network infrastructure, especially anything
  touching the management address or VLAN. The office bricked its switch's
  reachability by changing management VLAN and management IP in one go: one
  applied and the other did not.
- **Config backups never go in git.** A `.unf` carries Wi-Fi passphrases the way
  the office's `config.boot` carries password hashes.

New here:

- **Zone-based firewall is one-way.** Enabling it cannot be undone.
- **Blocking the Gateway zone breaks DHCP and DNS** for that network
  immediately, with no warning.
- **An unmanaged switch may not pass VLAN tags.** §4.
- **A 3B+ on a microSD card will eventually eat it.** §8.3.

---

## 13. DR

| Failure | How you know | What to do |
|---|---|---|
| Pi dead or SD card gone | SSM node offline; missing-backup alarm within 48 h | Remote access falls to D5/D6. Rebuild per §14 — the Pi holds nothing irreplaceable. |
| UX7 dead or factory-reset | The house has no internet | Replace or reset, restore the newest `.unf` during setup (§10.5). The switch, the TV and the Pi need no reconfiguration: addresses are reservations inside the restored config. |
| Backup job failing | SNS mail from `OnFailure` | `journalctl -u home-backup.service` over `homepi`. Most likely the local-only admin's password changed or the account was deleted. |
| Backups silently stopped | Missing-heartbeat alarm after 48 h | The Pi is up but not running the timer, or has no credentials. `systemctl status home-backup.timer`, `aws sts get-caller-identity`. |
| House WAN down | Everything is unreachable | Nothing to do remotely. Both D4 and D5 depend on the house having internet; Teleport does too. There is no out-of-band path and adding one (LTE) is not justified here. |
| Locked out by a firewall policy | The UX7 UI stops answering from Main | From the Pi on Mgmt, over `homepi`. If Mgmt is also cut: the UX7's reset button, then restore the last `.unf`. This is why the backup exists. |
| Laptop lost | — | Needs: AWS access to `black-family`, AWS CLI v2 + `session-manager-plugin`, a clone of `cj-local` for `connect.sh`, a copy of `tunnel.env`, and an SSH key authorised on the Pi. |

---

## 14. Rebuilding the Pi

Design rule, borrowed verbatim from the office: **nothing irreplaceable lives on
the box.**

| Lives on the Pi | How it comes back |
|---|---|
| `/etc/home-backup/env` | This document, plus the UniFi local-admin password from the password manager |
| SSM hybrid registration | New activation; the old `mi-…` deregistered |
| `/var/backups/home/*` | Copies; the masters are in S3, versioned |
| OS and packages | Fresh image plus §8 |

Order:

1. Fresh Raspberry Pi OS Lite 64-bit per §8.1. Same hostname `bf-home-001` —
   the S3 prefix and the alarm dimension are both derived from it.
2. If the NIC changed (new board, not just a new card), update the fixed-IP
   reservation on the UX7 to the new MAC.
3. Network per §8.2.
4. `aws ssm deregister-managed-instance --instance-id <old mi-…> --profile black-family`,
   then a new activation and registration per §9.1.
5. Update the `mi-…` id in `tunnel.env` (§9.2) — this is the one place a Pi
   rebuild leaks outside the Pi.
6. Reinstall the backup bundle per §10.3, fill `env`, run once by hand.
7. Verify: `homepi` and `homeux` connect; an object lands in S3; the alarm
   returns to OK.

---

## 15. Punch list

Build order first, then the open items.

**Build**

- [ ] UX7 initial setup, firmware current, Network ≥ 9.0.108.
- [ ] Four networks per §5; Guest as a UniFi Guest Network.
- [ ] LAN port profile: native AV, tagged VLAN 10 only (D3).
- [ ] SSIDs per §7; mDNS on Main and AV only.
- [ ] Zone-based firewall per §6 — **after** confirming the UI matches, since it
      cannot be undone.
- [ ] Pi base build, VLAN, Wi-Fi standby (§8). **Prove the tagged path works
      through the switch before anything else depends on it** (§4).
- [ ] `home-backups` stack; confirm the SNS subscription email.
- [ ] SSM activation and registration; record the `mi-…` id in §9.1.
- [ ] `connect.sh` entries; test `homepi` and `homeux` from outside the house.
- [ ] UniFi local-only admin `backup`; backup bundle; first run; object in S3.
- [ ] Restore rehearsal: download the `.unf` and confirm the UI accepts it on
      the restore screen. Do not actually restore a working network — confirm it
      is accepted, then cancel.

**Open**

- [ ] **ISP WAN**: public IPv4 or CGNAT? D5 is only viable on the former.
      Confirm before building the WireGuard fallback.
- [ ] **Console-level backup.** The `.unf` covers the Network application. The
      UniFi OS console backup (`.unifi`) is separate and its API path is not
      documented. Either confirm an endpoint on the device and add a second step
      to the job, or make it a quarterly manual download — but decide, rather
      than assume the `.unf` covers everything.
- [ ] **UPS** for the UX7 and the Pi. The cheapest protection against SD-card
      corruption there is.
- [ ] **USB SSD boot** for the Pi (§8.3), if the 3B+'s USB boot is confirmed.
- [ ] **DNS filtering.** UniFi's per-network ad blocking is strictly better than
      the office's arrangement — the office runs one dnsmasq policy for every
      client, which is why its blocklists have to stay narrow (vendor TV lists
      only). Here, Guest and AV can be filtered hard while Main is left alone.
      Decide per network.
- [ ] **A wired Main device**, if one ever appears. The unmanaged switch then
      forces a choice: tag it too, or buy the VLAN-capable switch from §4.

**Answerable only from the house** — all read-only observations; none needs
anything changed.

- [ ] **Q1. Is the WAN a real public IPv4, or CGNAT?** Compare the UX7's WAN
      address against a public IP checker. `100.64.0.0/10` on the WAN interface
      is CGNAT outright; a private address means double NAT behind the ISP box.
      **Gates D5 entirely** — WireGuard plus UniFi DDNS is useless behind CGNAT.
- [ ] **Q2. UX7 firmware and Network application version** (UniFi OS → Settings
      → System). §2 requires Network ≥ 9.0.108; the zone-based firewall in §6
      does not exist below 9.0.
- [ ] **Q3. Does the TP-Link pass 802.1Q tags?** §4's test, from the Pi:
      `ip -br addr show eth0.10` then `ping -c3 172.21.10.1`. **Gates D3 and the
      whole Pi-on-Mgmt design**; §4 has the fallback ladder.
- [ ] **Q4. TP-Link model number** — the label on the box. Determines whether
      Q3's answer was predictable. Record it in §2.
- [ ] **Q5. Does this 3B+ support USB mass-storage boot?** The 3B+ does without
      the OTP fuse the plain 3B needs — confirm on the actual board. §8.3; a $20
      SSD removes the SD-card failure mode.
- [ ] **Q6. Is the ISP device bridgeable?** From the ISP box's own UI. Double
      NAT is survivable; note it either way.

**To bring this document from plan to record**, once the build runs: the `mi-…`
SSM node id into §9.1 and the two `tunnel.env` lines in §9.2, which currently
read `mi-REPLACE`; Q1–Q6 each as a one-line answer; the §4 tag-test result and
which branch of the fallback ladder was taken; and any step in §8 or §10.3 where
the documented command did not work as written. **This document has never been
executed** — it is a careful plan, not a tested runbook, so divergences are
expected and they belong in the record.
- [ ] **Tighten Main → Mgmt** to an address group, if the house ever has more
      people with laptops on Main than CJ.

---

## 16. Conventions

Carried from `network-1770` so the two networks stay legible side by side.

- **Hostnames.** `BF-<ROLE>-NNN` for network devices (`BF-UX-001`, `BF-SW-001`,
  `BF-TV-001`); lowercase site-style names for Linux hosts (`bf-home-001`,
  matching `osc-chicago-001`).
- **Addressing.** `172.21.<VLAN>.0/24`, `.1` gateway, `.2–.99` static
  infrastructure, DHCP above `.100`. `172.34.x.x` is public space — never use it.
- **Time.** UTC everywhere: `Etc/UTC` on hosts, UTC in timers and in anything
  written down. Convert to Chicago time only when saying *when* something
  happens to a person.
- **Secrets never enter git.** The UniFi admin password, the `.unf` files and
  anything matching `*.env` stay on the hosts that use them and in the password
  manager.
- **Scripts print results only.** Every instruction lives in a document, not in a
  comment block nobody reads.
- **This file is the record.** When something on the network changes, it changes
  here. Tick punch-list items only once they are actually done.
