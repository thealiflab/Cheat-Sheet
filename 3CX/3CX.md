# 3CX Architecture Cheatsheet

A reference for understanding how the system actually fits together, aimed at whoever has to administer and troubleshoot it. Current as of 3CX V20, Update 9 (2026).

---

## 1. What 3CX actually is

3CX is a **software IP PBX**. It is not a box. It is a set of Linux services (Debian 12 in V20) that route calls using **SIP** for signalling and **RTP** for audio. Everything else (web client, mobile apps, desk phones, trunks, contact centre features) hangs off that core.

If you internalise one thing: **signalling and media are two separate flows on two separate protocols.** Most of the confusing failures in 3CX come from one of those two flows breaking while the other works. A call that connects but has no audio is a media (RTP) problem, not a signalling (SIP) problem. A phone that will not register is a signalling problem, and audio never even gets a chance.

---

## 2. Deployment models

| Model | Where it runs | Who patches it | Notes |
|-------|---------------|----------------|-------|
| Self-hosted on-premise | Your own VM or server on site | You | Full control, full responsibility. You own the firewall exposure. |
| Self-hosted cloud | Your cloud tenant (AWS, Google, Azure, Vultr, etc.) | You | 3CX provides marketplace images. You still own the OS and firewall. |
| 3CX-hosted (managed) | 3CX infrastructure | 3CX | Least control, least maintenance. |
| Partner-hosted | A partner such as an ITSP | The partner | Someone else certifies each update before it reaches you. |

The deployment model does not change the internal architecture below. It only changes who is responsible for the operating system, the firewall, and applying updates. Be honest with yourself about which of those you are actually doing, because an unpatched self-hosted PBX exposed to the internet is a liability, not a phone system.

---

## 3. Core internal components

These are the services running inside a single 3CX install.

- **Call Manager** — the SIP server. Handles registration, call setup, routing, transfers, queues, ring groups. Rewritten from the ground up in V20 for higher capacity (roughly 50 percent more users on the same hardware). This is the brain.
- **Media Server** — handles RTP audio streams, conference mixing, recording, and media relay when endpoints cannot talk directly.
- **Nginx** — the web server, now embedded directly into the 3CX install in V20. Serves the Web Client, the Admin Console, and phone provisioning.
- **PostgreSQL** — the database (v15 in current builds). Holds configuration and call detail records (CDRs).
- **Provisioning service** — pushes configuration to IP phones over HTTP/HTTPS so you do not hand-configure each handset.
- **Queue and reporting engine** — call centre logic, wait times, callbacks, statistics.
- **AI services (AI Edition)** — transcription, AI Receptionist, AI Personal Assistant, AI Agents. Provider is pluggable (OpenAI, Grok, or self-hosted). Not part of the call path unless you enable it.

In V20 the old separate **Management Console** is gone. Admin configuration now lives **inside the Web Client** at the same URL with the same login. One FQDN for both users and admins.

---

## 4. The two protocols (the part that matters most)

### SIP — signalling
Sets up, controls, and tears down calls. Registration, "phone is ringing", "call answered", "call transferred", "hung up". SIP carries **no audio**. It is text-based and small.

### RTP — media
The actual audio (and video). Once SIP has negotiated the call, the two endpoints stream RTP to each other. This is where bandwidth and quality live.

**Why this split causes grief:** SIP negotiates *which IP and port* each side will send RTP to. Behind NAT, an endpoint often advertises its private IP address inside SIP, which the far side cannot reach. Signalling succeeds, the call connects, and then RTP goes nowhere. That is the classic **one-way audio or no audio** symptom. NAT traversal (Section 7) exists entirely to fix this.

### Codecs (media, bandwidth planning)
- **G.711 (PCMU / PCMA, a-law / u-law)** — uncompressed, ~64 kbps per stream plus overhead (~85 to 100 kbps in practice). Best quality, most bandwidth. LAN default.
- **G.729** — compressed, ~8 kbps plus overhead. Used to save bandwidth on constrained links, at some quality cost.
- **Opus** — used by WebRTC and the Web Client. Adaptive, good quality.

Budget roughly 100 kbps per concurrent call on G.711 when sizing a branch uplink, then add headroom.

---

## 5. Ports reference

Defaults. Learn these; most firewall and connectivity tickets are one of these ports being blocked, translated, or forwarded wrong.

| Port | Protocol | Purpose |
|------|----------|---------|
| 5060 | UDP / TCP | SIP signalling (unencrypted) |
| 5061 | TCP | SIP over TLS (encrypted signalling) |
| 5090 | UDP / TCP | 3CX Tunnel and SBC (wraps SIP + RTP over one port) |
| 443 | TCP | HTTPS — Web Client, Admin Console, provisioning |
| 80 | TCP | HTTP — Let's Encrypt certificate validation and provisioning fallback |
| 9000–10999 | UDP | RTP media range (configurable) |

The RTP range is the wide UDP block people forget to open. If SIP is fine but audio is not, check this range before anything else.

---

## 6. Extensions and clients

An **extension** is a user identity. Multiple devices can register to the same extension.

- **Web Client / PWA** — browser-based, WebRTC media. 3CX is actively pushing this as the primary client because the browser sandbox is a smaller attack surface than a downloaded binary. Redesigned in Update 9.
- **Windows / Mac desktop app** — Electron app. Can run as a softphone or in CTI mode controlling a desk phone. 3CX is steering people away from it toward the Web Client for security reasons (see Section 10).
- **Mobile apps (iOS / Android)** — rely on **PUSH notifications**. The app does not hold a permanent connection; 3CX wakes it via Apple or Google push when a call arrives. If push is misconfigured, mobile calls silently fail to ring. This is a common and confusing failure.
- **IP desk phones (Yealink, Fanvil, Snom, etc.)** — auto-provisioned by the PBX. Config is pushed, not typed. On the LAN this uses DHCP option 66 or multicast; for remote phones it uses 3CX's Redirection and Provisioning Service (RPS).

---

## 7. NAT, firewall, and FQDN (the real-world pain)

Almost everything hard about 3CX is getting SIP and RTP across NAT boundaries.

- **FQDN** — every install has a Fully Qualified Domain Name (often a 3cx.us or similar subdomain) with an automatic Let's Encrypt TLS certificate. Clients, provisioning, and secure connections all key off this. If the FQDN or certificate breaks, clients cannot connect even though the PBX is "up".
- **STUN** — helps an endpoint discover its own public IP so it advertises a reachable address in SIP instead of its useless private one.
- **Public IP and port forwarding** — a self-hosted PBX behind NAT needs the ports in Section 5 forwarded correctly. Get the RTP range wrong and you get one-way audio.
- **Firewall Checker** — 3CX ships a tool that tests your firewall from outside. Run it before blaming anything else. If it fails, stop and fix the firewall first; downstream symptoms are noise until it passes.

---

## 8. SBC vs Tunnel (remote sites and remote users)

Both wrap SIP and RTP over a single port (**5090**) so you do not have to forward the whole RTP range at every remote location. They solve different problems.

- **3CX SBC (Session Border Controller)** — a small application installed **at a remote site** (Linux, Windows, or Raspberry Pi). Multiple physical desk phones at that branch register back to the central PBX through one secure tunnel. Use this when a branch has several handsets. This is the relevant one for a multi-site setup with desk phones at each location.
- **3CX Tunnel** — built into the client apps for an **individual remote user** whose network is hostile to SIP. Wraps that one client's traffic over 5090.

Rule of thumb: many phones at one remote location means an SBC. One roaming user on a bad network means the client tunnel (or increasingly, just the Web Client over WebRTC).

---

## 9. Trunks and the outside world

- **SIP trunk (ITSP / VoIP provider)** — how the PBX reaches the public phone network (PSTN). This is a SIP connection to a carrier. Inbound numbers (DIDs) land here; outbound calls leave here.
- **Gateway** — hardware bridging analog or ISDN lines into SIP, for legacy connectivity.
- **Outbound rules** — decide which trunk a call uses based on the dialled number, caller, or prefix.
- **Inbound rules / DID routing** — decide where an incoming number lands (an extension, a queue, an IVR, an AI receptionist).

If external calls fail but internal extension-to-extension calls work, the problem is the trunk or its rules, not the PBX core. That single test isolates the fault fast.

---

## 10. Security realities (read this)

3CX has a track record that a responsible admin should not ignore.

- **2023 supply chain compromise.** The 3CX Desktop App (the Electron build) was trojanised in a supply chain attack ("SmoothOperator"). This is the direct reason 3CX now pushes the browser-based Web Client over the downloaded desktop binary. Prefer the Web Client where you can.
- **Exposed SIP ports get attacked.** Port 5060 open to the internet invites constant registration brute-force and toll-fraud attempts. Do not expose SIP more than you must. Use TLS (5061), strong extension passwords, IP allow-listing, and the built-in anti-fraud and blacklist features.
- **Patching is not optional.** A self-hosted PBX on the public internet running an old build is a real risk. Updates ship regularly (Update 8 in Feb 2026, Update 9 in mid-2026). If you self-host, you own the patch cadence, and "we will do it later" is how systems get compromised.
- **Toll fraud is a financial risk, not just a technical one.** A compromised extension dialling international premium numbers can generate a large bill overnight. Set outbound call limits and destination restrictions.

---

## 11. Call flow, end to end

**Outbound call:**
1. Client registers to Call Manager over SIP (5060 or 5061).
2. User dials. SIP INVITE goes to the Call Manager.
3. Outbound rule picks a trunk. Call Manager sends SIP to the carrier.
4. Once answered, RTP audio flows (direct between endpoints where possible, or relayed through the Media Server across NAT).
5. Hang-up sends SIP BYE. RTP stops.

**Inbound call:**
1. Carrier sends SIP INVITE to the PBX over the trunk.
2. Inbound rule routes it (extension, ring group, queue, IVR, or AI receptionist).
3. Target endpoint rings via SIP; mobiles are woken by PUSH.
4. On answer, RTP flows.

---

## 12. Troubleshooting map (symptom to layer)

| Symptom | Most likely layer | First check |
|---------|-------------------|-------------|
| Phone will not register | SIP signalling | Port 5060/5061, credentials, FQDN, firewall checker |
| Call connects, no audio or one-way audio | RTP media | RTP range 9000–10999, NAT, STUN, direct-media setting |
| Mobile does not ring | PUSH | Push configuration, app rebuild, network sleep |
| External calls fail, internal calls fine | Trunk | SIP trunk registration, outbound rules, carrier |
| Nobody can reach the Web Client | Web / FQDN | FQDN resolution, Let's Encrypt certificate, port 443, Nginx |
| Whole branch of phones down, others fine | SBC | SBC tunnel status on port 5090 at that site |
| Choppy or dropping audio | Bandwidth / QoS | Uplink capacity, codec choice, jitter, QoS on the link |

Work from the layer, not the panic. Isolate whether it is signalling, media, trunk, or web first, and the rest narrows quickly.

---

## 13. V20-specific notes worth remembering

- **Debian 12** is the underlying OS. Windows is being deprioritised; treat Linux as the default going forward.
- **Admin Console is now inside the Web Client.** Same URL, same login. No more separate management console.
- **Departments replaced Groups**, behaving like an organisational unit (similar to an Active Directory OU).
- **Call Manager was rewritten**, which is why capacity and call-centre features improved.
- **Nginx and PostgreSQL are embedded** in the install rather than bolted on.
- **April 2026 licensing:** Free, Basic, PRO, and AI Edition (AI Edition is the renamed Enterprise tier; Enterprise Plus was retired). AI features are gated to AI Edition; core telephony and queues do not need it.

---

*Fundamentals (SIP, RTP, SBC, trunks, NAT traversal) are stable across versions. Version-specific details above reflect V20 Update 9 and will drift as 3CX ships updates, so re-verify licensing tiers and OS support at upgrade time.*