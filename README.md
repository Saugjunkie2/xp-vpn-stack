xp-vpn-stack/README.md
================================================================================
TITLE: xp-vpn-stack (Repository Overview)
VERSION: v1.2
DATUM: 2026-01-20
STATUS: ACCEPTED
OWNER-INTENT: Kurzüberblick: Ziel, Struktur, Einstiegspfade und Verweis auf Master/Blocks/Decisions als Spezifikation.
OWNER: Marco (Owner) + ChatGPT (Co-Architect)
================================================================================

# xp-vpn-stack

A **Debian 13** “from scratch” **L2TP/IPsec (strongSwan + xl2tpd/pppd)** VPN stack designed specifically for **Windows XP / Vista** clients.

Goal: **Full-Tunnel**, minimal client configuration, deterministic server-side policy, and a production-ready operational model (AAA, accounting, QoS, DNS stack, internal webpanel, blockpage MITM, onboarding: Verify-Wall + Claim Token + UNCLAIMED Grace/Overdue).

---

## What this is

This repository is a **specification** (not an installer script yet).  
It is structured as:

- **MASTER (zeilenfest)**: canonical, versioned master concept
- **0_MASTERKONZEPT_ANALYSE.md**: Arbeits-/Abarbeitungsdatei (Hybrid), verweist auf Blocks/Decisions/Tasks (optional)
- **blocks/**: implementation blocks (B010–B343) defining the system precisely
- **decisions/**: decision records (ADR) explaining *why* choices were made
- **tasks/**: optional workspace files (non-canonical; the specification lives in MASTER/blocks)
- **templates/**: templates for adding future blocks/decisions consistently


The intended outcome is a reproducible server setup where the VPN software provides the tunnel, while **Linux enforces the policies** (nftables / tc / service binding).

---

## Core objectives

- **Windows XP/Vista built-in client compatibility** (no SoftEther client)
- **Full-Tunnel by design** (all client traffic exits through the VPN)
- **“Idiotensicher” client experience**
  - client should only need **FQDN + VPN-credentials (PPP) + PSK**
  - no manual IP/gateway/DNS entry (PPP/IPCP)
- **Segments (SoT: B060)**
  - LEGACY (XP/Vista): `10.77.16.0/20` (IPv4-only; PPP/L2TP path)
  - ADMIN (dualstack, IKEv2): `10.77.2.0/24` + `fd77:0:0:2::/64`
  - MODERN (dualstack, IKEv2): `10.77.32.0/20` + `fd77:0:0:32::/64`
  - SERVICE_NET: `10.77.0.0/28` with fixed service anchors:
    - `SERVICE_DNS 10.77.0.1` (DNS)
    - `SERVICE_WEB 10.77.0.3` (panel/portal/blockpage + /diag)
    - `SERVICE_NTP 10.77.0.4` (NTP)
- **Strict Peer-Isolation** (user clients cannot reach each other)
- **Server exposure minimized**
  - WAN: only IPsec ports (UDP 500/4500 + ESP)
  - UDP 1701 (L2TP) only accepted **via IPsec/XFRM**
- **Per-client QoS**
  - shaping per `pppX` using `tc` (CAKE/fq_codel)
- **DNS stack (enforced)**
  - Unbound local resolver + AdGuardHome (internal)
  - **DNS enforcement is MUST** (DNAT TCP+UDP 53 from `ppp*` → `10.77.0.1:53`)
- **Internal webstack**
  - OpenResty (Nginx+Lua) + PHP-FPM + MySQL/MariaDB + phpMyAdmin (admin-only)
  - webpanel reachable only inside VPN, bound to SERVICE_WEB (10.77.0.3)
- **HTTPS blockpage with internal CA (MITM)**
  - blocked domains resolve to SERVICE_WEB (10.77.0.3) (AdGuard “Custom IP”)
  - OpenResty serves HTTP/HTTPS block pages
  - XP browser target: **MyPal** with NSS trust store handling
- **Time/NTP strategy**
  - XP time problems handled (pre-/post-connect strategy; optional NTP hijack to SERVICE_NTP (10.77.0.4))
## Onboarding (v2.3)
- Verify-Wall (App-Layer): Customer PENDING sieht nach Login nur Code/Resend/Support.
- Verify-Code ist kurzlebig + single-use; persistiert wird nur ein Hash + Ablaufzeit (kein Klartext-Code in SQL).
- claim_token (App-Layer): Claim ordnet eine VPN-Connection einem Customer zu (claim_token als Besitznachweis; nicht für VPN-Login).
- UNCLAIMED Grace/Overdue (Kernel): 30 Tage ab Provisioning/Erstellung Internet frei; danach UNCLAIMED_OVERDUE -> "Nur Panel-Zugriff" (Walled Garden), damit Verify+Claim weiterhin möglich sind.
- Hard-Stop gegen Leichen (Standardbetrieb): claim_deadline immer gesetzt (180 Tage). Nach Ablauf unclaimed -> DISABLED (kein VPN/kein Panel).


---

## Design principle: “VPN is only the tunnel – policies live in Linux”

The stack intentionally avoids relying on “VPN software features” for control:

- Routing/NAT/segmentation: **Linux**
- Policy enforcement: **nftables**
- QoS: **tc**
- Services: bound to SERVICE_* anchors in SERVICE_NET (B060) and firewall-limited (never WAN)
- AAA/accounting: **SQL + FreeRADIUS** (source of truth)
- Determinism: **policy-apply + reconcile** (drift-safe)

This is built for selling/supporting XP systems with minimal support overhead and predictable behavior.

---

## Stack overview

### VPN (XP/Vista compatible)
- strongSwan (IKEv1/IPsec, including legacy crypto requirements)
- xl2tpd + pppd
- IP assignment via PPP/IPCP (no client-side static IPs)

### Policy / Security
- nftables WAN stealth + XFRM-only L2TP
- full-tunnel forward + NAT
- MSS clamping / MTU stability measures
- outbound abuse blocking (SMTP + SMB/NetBIOS)
- IPv6 leak-control (LEGACY v4-only; ADMIN/MODERN dualstack per B060; WAN exposure strictly minimized)

### QoS
- per-PPP interface shaping via tc
- default limits by conn_group / segment (LEGACY/ADMIN/MODERN) and DB-driven parameters

### DNS
- Unbound (local-only)
- AdGuardHome on `10.77.0.1:53` (UI admin-only)
- **DNS enforcement MUST**: DNAT TCP+UDP 53 from `ppp*` → `10.77.0.1:53`

### Webpanel & Blockpage
- OpenResty + PHP-FPM + DB
- panel and diagnostics endpoints internal-only
- MITM HTTPS blockpage using an internal CA
- MyPal NSS trust store integration

### Reliability / Operations (v2.0 hardened)
- systemd auto-restart + healthchecks for key services
- DPD + LCP echo to avoid stale sessions
- offload tuning defaults for virtualized environments
- **Accounting collector + session mapping** (pppX → connection_id)
- **Stale-session janitor + on-demand janitor in login flow** (SimUse lockout-safe)
- **Spool/Retention safety (v2.3)**: SQL-first settings + local safety ceilings (hard max bytes/age). On ceiling hit: **Ring Buffer (Drop Oldest) MUST** + alert (prevents disk-full failure).
- **Policy apply + reconcile (FLUSH+REBUILD)** to prevent drift
- **Hard Cut enforcement (v2.3)**: when a client becomes restricted, established flows are terminated via **conntrack flush** (bidirectional). Fail-safe: **PPP session kill (fail-closed)** if flush fails.

---

## Repository structure (authoritative)

- `00000_Ordnerstruktur.txt` : aktuelle Ordner-/Dateistruktur (Single Source of Truth für den Tree)
- `000_MASTER-KONZEPT vX.X (ZEILENFEST).txt` : canonical “zeilenfest” spec
- `00_MASTER_vX.X.txt` : readable master summary + block map
- `0_MASTERKONZEPT_ANALYSE.md` : Abarbeitungs-/Arbeitsdatei (Hybrid: führt durch Blocks/Decisions/Tasks) (optional)
- `01_CHANGELOG.txt` : version history / deltas
- `02_GLOSSAR.txt` : glossary
- `03_ASSUMPTIONS_AND_RULES.txt` : bindende Annahmen & Regeln
- `04_BLOCK_INDEX.txt` : block index (B010–B343)
- `blocks/` : implementation blocks
- `decisions/` : ADR decision records
- `tasks/` : optional workspace files (non-canonical)

---

## Phased rollout model

The spec is built around phases (VPN first, then QoS/web/MITM/gates), with the key rule:
**VPN stability first, then features**.

See `B290_PHASE_PLAN_ROLLOUT` and the MASTER file for the authoritative phase plan.

---

## Status

- Spec baseline: **MASTER v2.5** (inhaltlich fortgeschrieben; Version bump erfolgt separat im Master)
- Blocks: B010–B343 present (inkl. Session-Control Anchor **B171** sowie Hardening in B150/B165–B169/B330 und Fail2ban/Outcome B340–B343)
- Decision records: present (u.a. D008 Verify-Wall customer scope, **D010/D011** Reason/Emission, **D012** Session-Control A/B)
- Templates: present

---

## Scope / Non-goals

- This repo does **not** ship a one-click installer yet.
- DoH/DoT bypass prevention is out of scope for the “DNS enforcement” mechanism.
- “DNS enforcement” means **port 53 enforcement** (DNAT), not DoH/DoT interception.

---



================================================================================
CHANGELOG
================================================================================
- 2026-01-20 v1.2: Service-Bind-Satz im Design-Prinzip auf SERVICE_* (B060) korrigiert; IPv6-Statement auf Leak-Control/Segment-Scope präzisiert; QoS-Grouping auf conn_group/Segmente (LEGACY/ADMIN/MODERN) gezogen.
- 2026-01-20 v1.1: K1/B060-SoT übernommen: Segmente (LEGACY/ADMIN/MODERN) + SERVICE_* (DNS/Web/NTP) statt „2 Netze + Service-/32“; Blockpage/Web/NTP Targets auf SERVICE_WEB/SERVICE_NTP gezogen; Status-Baseline auf MASTER v2.5 aktualisiert.
- 2026-01-13 v1.0: Added Version/Changelog sections (protocol compliance).
================================================================================

## License

This project is licensed under the [MIT License](LICENSE).