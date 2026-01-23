PHASE 4a PATCH — FILE: README.md
OP: REPLACE (FULL FILE)
CONTENT:
xp-vpn-stack/README.md
================================================================================
TITLE: xp-vpn-stack (Repository Overview)
VERSION: v1.3
DATUM: 2026-01-23
STATUS: ACCEPTED
OWNER-INTENT: Kurzüberblick: Ziel, Struktur, Einstiegspfade und Verweise auf Master/Blocks/docs als Spezifikation.
OWNER: Marco (Owner) + ChatGPT (Co-Architect)
================================================================================

# xp-vpn-stack

Debian-13 „from scratch“ High-Security VPN ISP-Plattform für Windows XP / Vista (Legacy) plus Modern/Admin Stack.

Design-Highlights (Kanon, Master v2.6)
- Exposure-by-Binding: WAN ist primär „closed“, weil Services nicht an WAN/0.0.0.0/:: binden. nftables-Stealth ist Zusatzschutz.
- Server-Sphäre vs Client-Bubble getrennt: Host-INPUT/OUTPUT vs Client-FORWARD/NAT (+ XFRM bei IPsec/IKEv2).
- WAN-Minimal-Exposure: dauerhaft nur VPN-Endpunkte (IPsec: UDP 500/4500 + ESP; WG: WG-UDP-Port).
- Hybrid Gate-0: Legacy bleibt PSK (IKEv1/L2TP); Modern/Admin (IKEv2) nutzt Server-Zertifikat/Trust (Public CA, konkret Let’s Encrypt), kein PSK.
- Gate-1 bleibt Pflicht: connection_login (User/Pass) via EAP/RADIUS (SQL-First) → fail-closed ohne „unpoliced window“.
- SQL-First: Settings/Policy/State in SQL; keine versteckten TTL/Timeout-Literale außerhalb von B057.

Repository Structure (SOLL)
- 00_MASTER_v2.6.txt — Master/Canon/DoD (lesbar, zentral).
- 000_MASTER-KONZEPT v2.6 (ZEILENFEST).txt — Master mit stabilen Zeilennummern (für Anchors/Patches).
- 01_CHANGELOG.txt — zentraler Master-Changelog (v2.x).
- 02_GLOSSAR.txt — Terminologie/Definitionen.
- 03_ASSUMPTIONS.txt — Annahmen/offene Punkte.
- 04_BLOCK_INDEX.txt — Index der Blocks.
- blocks/ — Kanon-Blocks (B***.txt), jeweils mit TITLE/VERSION/DATUM/STATUS/CHANGELOG.
- docs/ — ergänzende technische Doku, von Blocks referenziert (kein Policy-Leak).
- templates/ — Vorlagen/Patterns (nicht automatisch Kanon).
- tasks/ — optional (kann leer sein; nicht kanonisch).

Schneller Einstieg
1) 00_MASTER_v2.6.txt (Überblick + Regeln + DoD)
2) 02_GLOSSAR.txt (Begriffe/Abkürzungen)
3) 04_BLOCK_INDEX.txt → relevante Blocks öffnen
4) blocks/B310_TESTS_ACCEPTANCE_CATALOG.txt (DoD/Tests)

Status
- Master: v2.6
- Blocks: B010–B344
- Doku: docs/ + templates/ konsistent; tasks/ optional.

================================================================================
CHANGELOG
================================================================================
- 2026-01-23 v1.3: Auf Master v2.6 aktualisiert; Struktur-/Einstiegsdoku bereinigt; Blocks-Range auf B344 erweitert.
- 2026-01-20 v1.2: Service-Bind-Satz im Design-Prinzip auf SEPARATION (Server-Sphäre vs Client-Bubble) präzisiert.
- 2026-01-20 v1.1: K1/B060-SoT übernommen: Segmente (LEGACY/ADMIN/MODERN) konsolidiert.
- 2026-01-13 v1.0: Added Version/Changelog sections (protocol compliance).
================================================================================

## License

This project is licensed under the MIT License.