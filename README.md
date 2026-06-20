# Empfehlungssystem — wasserzisterne.de

Eigenentwickeltes Referral-System, das Bestandskunden zu Multiplikatoren macht. Löst die bestehende TELLSCALE-Lösung ab.

**Produktive Domain:** `empfehlung.wasserzisterne.de`

---

## Was das System tut

1. Bestandskunde kauft → KlickTipp sendet automatisch eine E-Mail
2. Kunde öffnet die Landing Page (mobil-first) → **NPS-Gate**
   - NPS 9–10 → Empfehlungsflow
   - NPS 0–8 → Feedback-Flow
3. Kunde erhält einen **persönlichen Short-Link** und teilt ihn per WhatsApp (`wa.me`, primär) oder E-Mail
4. Neukunde öffnet die personalisierte Infoseite (E1) → Vimeo-Video + Lead-Feld (consent-gated)
5. Kauft der Neukunde (Mindestbestellwert / Produktkopplung) → Promotor bekommt **25 € Amazon-Gutschein per E-Mail**

---

## Tech-Stack

| Schicht | Technologie |
|---|---|
| Hosting | Hetzner (eigener Server, Datensouveränität) |
| Frontend + Backend | Next.js (App Router), self-hosted Node.js |
| Datenbank | PostgreSQL (self-hosted) |
| Orchestrierung / Integrationen | n8n (bestehend) |
| E-Mail / Kampagnen | KlickTipp REST API |
| Admin-Auth | NextAuth.js (Credentials) |
| Video | Vimeo |
| CI/CD | GitLab auf Hetzner |
| WhatsApp-Share | `wa.me` Click-to-Chat + dynamischer Text |

---

## Architektur

```
WordPress (Marketing-Site)
    │
    ▼
Referral-App (Next.js) auf empfehlung.wasserzisterne.de
    ├── NPS-Gate, Promotor-Seiten, Neukunden-Seiten (E1), Admin-Panel
    ├── API: Token, Short-Links, Tracking, Lead-Capture, Consent, Attribution
    └── PostgreSQL (State-Hoheit)
    │
    └── Webhooks/Events → n8n
            ├── KlickTipp-Tags
            ├── WooCommerce-Order-Ingest
            ├── Alerts
            └── Gutschein-Trigger (Streamendous → Amazon-Code → E-Mail an Promotor)
```

**Verantwortungsschnitt:**
- **Referral-App** besitzt: Token-/Short-Link-Generierung, Click-/View-Tracking, Lead-Capture, Consent-Records, Attribution-Matching, Reward-State-Machine, Admin-Daten.
- **n8n** besitzt: alle Verbindungen zu Fremdsystemen (KlickTipp, WooCommerce, Streamendous) und den Gutschein-Versand.

---

## Projektstatus

**Phase: Planung / Dokumentation**

Aktuell existiert kein Anwendungscode. Das Repository enthält die Planungsunterlagen in `Empfehlungssystem-dev/Dokumente/`.

| Phase | Inhalt | Status |
|---|---|---|
| 0 | Infrastruktur (Hetzner, GitLab, PostgreSQL, n8n) | ⏳ Gated durch Kundenblocker |
| 1 | Datenmodell + Core-Backend | ⏳ Ausstehend |
| 2 | Attribution + Reward-State-Machine | ⏳ Ausstehend |
| 3 | Öffentliche Seiten (C1/D1 mobil, E1, C2/D2 Desktop, Feedback) | ⏳ Ausstehend |
| 4 | Admin-Panel | ⏳ Ausstehend |
| 5 | Integration, Test, Launch | ⏳ Ausstehend |

---

## Dokumente

| Datei | Beschreibung |
|---|---|
| `Empfehlungssystem-dev/Dokumente/Projektplan Empfehlungssystem.md` | **Verbindlicher Projektplan v1.1** — Anforderungen, Datenmodell, API, Phasenplan, Blocker |
| `Empfehlungssystem-dev/Dokumente/Empfehlungssystem.pdf` | Original-Kundenvorgabe (19 Folien) |
| `Empfehlungssystem-dev/CLAUDE.md` | KI-Arbeitsanweisung für Claude Code |
| `Empfehlungssystem-dev/AGENTS.md` | Erweiterte Agenten-Anweisung |

---

## Offene Kundenblocker

Vor dem Bau der jeweiligen Phasen kundenseitig zu klären:

- **A1** Promotor-Identität / KlickTipp-Subscriber-ID — vor Phase 1
- **A2** WooCommerce-Order-Webhook + Match-Felder — vor Phase 2/3
- **A3** Reward-Bedingung (Mindestbestellwert / Produktliste) — vor Phase 2 & 3
- **A4** Gutschein-Ausführung Streamendous (API? `failed`-Handling?) — vor Phase 2

---

## Entwicklung starten (sobald Code vorhanden)

```bash
# Abhängigkeiten installieren
npm install

# Entwicklungsserver starten
npm run dev

# Produktionsbuild
npm run build

# Linting
npm run lint

# Tests
npm run test
```

> Diese Befehle sind Platzhalter. Konkrete Skripte werden mit Phase 0/1 ergänzt.

---

## Konventionen

- **Sprache:** Deutsch für alle Artefakte (Kommentare, Commits, Dokumentation)
- **Code:** TypeScript (Next.js-Standard)
- **Namensgebung:** kebab-case für Dateien/Routen, camelCase/PascalCase für Funktionen/Komponenten, snake_case für Datenbank
- **Secrets:** ausschließlich über Umgebungsvariablen / Hetzner-Secret-Management — nie im Repository
- **Git-Workflow:** Feature-Branch → PR → Review → `main`; kein direkter Push auf `main`

---

## Team

2 Personen, ~15–20 h/Woche.
