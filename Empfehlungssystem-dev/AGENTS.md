# AGENTS.md — Empfehlungssystem wasserzisterne.de

> Agenten-Anweisung für dieses Repository.  
> Stand: 2026-06-20  
> Sprache der Projektdokumentation: Deutsch.

---

## 1. Projektübersicht

Dieses Repository enthält das **Empfehlungssystem für wasserzisterne.de**, das die bestehende TELLSCALE-Lösung (KIWUS CONSULTING) ablösen soll.

Kernziel: Bestandskunden werden nach einem Kauf über KlickTipp zu Multiplikatoren. Sie bewerten das Unternehmen (NPS-Gate), erstellen einen persönlichen Empfehlungslink und teilen ihn per WhatsApp (mobil, primär) oder E-Mail (Desktop, sekundär). Empfohlene Neukunden landen auf einer personalisierten Infoseite, hinterlassen ein minimales Kontaktfeld und können einen Beratungstermin buchen. Kauft der Neukunde (Mindestbestellwert / Produktkopplung), erhält der Promotor einen 25 € Amazon-Gutschein per E-Mail.

Produktive Domain: `empfehlung.wasserzisterne.de`

---

## 2. Was in diesem Repo liegt — und was nicht

**Vorhanden in diesem Verzeichnis:**

- `Dokumente/Empfehlungssystem.pdf` — Kunden-Präsentation (19 Folien), Ausgangsbriefing.
- `Dokumente/Projektplan Empfehlungssystem.md` — Verbindlicher Projektplan v1.1, abgeleitet aus PDF + Kundengespräch + Architektur-Review. Enthält Anforderungen, Architektur, Datenmodell, API-Oberfläche, Phasenplan, Risiken und offene Blocker.
- Diese `AGENTS.md`.

**Nicht vorhanden (Stand heute):**

- Kein Anwendungscode (Frontend/Backend).
- Keine `package.json`, `pyproject.toml`, `Cargo.toml` oder ähnliche Build-Dateien.
- Kein Git-Repository.
- Keine CI/CD-Konfiguration.
- Keine Secrets, keine `.env`-Dateien.

Das Repository befindet sich aktuell in der **Planungs- und Dokumentationsphase**. Code wird in späteren Iterationen hinzugefügt.

---

## 3. Geplante Technologie & Architektur

Die folgenden Angaben stammen ausschließlich aus `Dokumente/Projektplan Empfehlungssystem.md`.

| Schicht | Technologie |
|---------|-------------|
| Hosting | Hetzner (eigener Server) |
| Frontend + Backend | Next.js (App Router), self-hosted Node.js |
| Datenbank | PostgreSQL (self-hosted) |
| Integration / Orchestration | n8n (bereits bestehend) |
| E-Mail-Marketing + Kampagnen | KlickTipp REST API |
| Admin-Authentifizierung | NextAuth.js (Credentials) |
| Video-Einbettung | Vimeo |
| CI/CD | GitLab auf Hetzner (neu) |
| WhatsApp-Share | `wa.me` Click-to-Chat + dynamischer Text |

**Architektur-Skizze (aus dem Projektplan):**

```
WordPress (Marketing-Site)
    │
    ▼
Referral-App (Next.js) auf empfehlung.wasserzisterne.de
    ├── NPS-Gate, Promotor-Seiten, Neukunden-Seiten, Admin-Panel
    ├── API für Token, Short-Links, Tracking, Lead-Capture, Consent, Attribution
    └── PostgreSQL (State-Hoheit)
    │
    ├── Webhooks/Events → n8n
    │       ├── KlickTipp-Tags
    │       ├── WooCommerce-Order-Ingest
    │       ├── Alerts
    │       └── Gutschein-Trigger (Streamendous → Amazon-Code → E-Mail)
    │
    └── Externe Systeme: KlickTipp, WooCommerce, SafeDesk, etermin, Vimeo
```

**Verantwortungs-Schnitt:**

- Referral-App besitzt: Token-/Short-Link-Generierung, Click-/View-Tracking, Lead-Capture, Consent-Records, Attribution-Match, Admin-Daten, Reward-State-Machine.
- n8n besitzt: Verbindungen zu Fremdsystemen (KlickTipp, WooCommerce, Streamendous) und den Versand der Gutschein-Code-Mail.

---

## 4. Source-of-Truth

Bevor du fachliche oder technische Änderungen vornimmst, lies immer in dieser Reihenfolge:

1. **Diese `AGENTS.md`** — Projekt-Kontext und Arbeitsregeln.
2. **`Dokumente/Projektplan Empfehlungssystem.md`** — Verbindliche Anforderungen, Architektur, Datenmodell, API-Oberfläche, Phasenplan. Enthält explizite Blocker (§11.A) und Annahmen (§11.B).
3. **`Dokumente/Empfehlungssystem.pdf`** — Kundenvorgabe, falls der Projektplan eine Details nicht ausreichend abbildet.

**Wichtig:** Generiere keine Geschäftslogik, Schwellenwerte oder Datenfelder selbst. Wenn etwas im Projektplan nicht definiert ist, kennzeichne es als offen / zu klären und verändere nichts an bestehenden Annahmen.

---

## 5. Build-, Test- & Startbefehle

**Aktueller Stand:** Keine Build- oder Testbefehle verfügbar, da noch kein Code existiert.

**Sobald Code hinzukommt (Next.js), werden voraussichtlich folgende Befehle gelten:**

```bash
# Abhängigkeiten installieren
npm install

# Entwicklungsserver starten
npm run dev

# Produktionsbuild
npm run build

# Start des Produktionsservers
npm start

# Linting
npm run lint

# Tests (voraussichtlich mit Playwright / Jest / Vitest — noch nicht festgelegt)
npm run test
```

> Hinweis: Der Projektplan nennt den Stack (Next.js, Node.js), aber noch keine konkrete Test-Toolbox. Wähle diese gemeinsam mit dem Projektteam, sobald Phase 0/1 startet.

---

## 6. Geplante Code-Organisation

Bislang liegt kein Code vor. Die folgende Modulstruktur leitet sich aus dem Datenmodell und der API-Oberfläche des Projektplans ab und soll als Orientierung für zukünftige Erweiterungen dienen:

```
empfehlung.wasserzisterne.de/      # Next.js App Router
├── app/
│   ├── (public)/                   # Öffentliche Seiten
│   │   ├── nps/                    # NPS-Gate
│   │   ├── c1/                     # Mobile Promotor-Info
│   │   ├── c2/                     # Desktop Promotor-Info
│   │   ├── d1/                     # Mobile Link-Generator + wa.me-Share
│   │   ├── d2/                     # Desktop Link-Generator + E-Mail-Share
│   │   ├── e1/                     # Neukunden-Landingpage
│   │   └── feedback/               # Feedback-Flow
│   ├── r/[short_code]/             # Short-Link-Redirect
│   ├── admin/                      # Geschütztes Admin-Panel
│   └── api/                        # Backend-API-Routen
│       ├── nps/route.ts
│       ├── promotor/route.ts
│       ├── referral/route.ts
│       ├── consent/route.ts
│       ├── interest/route.ts
│       ├── feedback/route.ts
│       ├── webhooks/woocommerce/route.ts
│       ├── webhooks/klicktipp/route.ts
│       └── internal/match/route.ts
├── lib/
│   ├── db/                         # Datenbank-Client, Schema, Migrations
│   ├── auth/                       # NextAuth-Konfiguration
│   ├── tracking/                   # Tracking-Logik (Idempotenz, Bot-Filter, Session)
│   ├── attribution/                # Matching-Algorithmus
│   ├── rewards/                    # Reward-State-Machine
│   └── audit/                      # Audit-Log-Hilfsfunktionen
└── components/
    └── ui/                           # Wiederverwendbare UI-Komponenten
```

Diese Struktur ist ein Vorschlag. Passe sie an die tatsächlichen Next.js-Conventions und Team-Entscheidungen an, sobald Code entsteht.

---

## 7. Konventionen & Styleguide

Da noch kein Code vorhanden ist, gelten vorläufig folgende Richtlinien, die aus dem Projektplan und gängigen Next.js-Praktiken abgeleitet sind:

- **Sprache:** Kommentare, Commit-Messages und Dokumentation auf Deutsch (wie im Projektplan).
- **Code-Sprache:** TypeScript wird für Next.js empfohlen, ist aber noch nicht endgültig festgelegt.
- **Namensgebung:**
  - Dateien/Routen: kebab-case (`referral-link.ts`, `[short-code]`).
  - Funktionen/Komponenten: camelCase / PascalCase nach React/Next.js-Convention.
  - Datenbanktabellen/Spalten: snake_case (vgl. Projektplan §5).
- **Kleine, fokussierte Module:** Eine Route/Datei pro Verantwortlichkeit.
- **Explizites Error-Handling:** Keine stillen Fehler. Fail-safe-Verhalten bevorzugen.
- **Keine Secrets im Code:** API-Keys, Datenbank-URLs, Webhook-Secrets ausschließlich über Umgebungsvariablen.
- **DSGVO-konforme Consent-Records:** Tracking-Cookie und Datenverarbeitung getrennt erfassen (vgl. Projektplan §5 `consents`).

Sobald Code existiert, sollten konkrete Formatter-/Linter-Regeln (z. B. ESLint, Prettier, TypeScript strict) in einer `package.json` / `.eslintrc` / `prettier.config.js` hinterlegt werden.

---

## 8. Testing-Strategie

**Aktueller Stand:** Keine Tests vorhanden.

**Geplante Teststufen (aus Projektplan §9 Phase 5):**

1. **Unit-/Integrationstests** für kritische Backend-Logik:
   - Token-/Short-Link-Generierung
   - Click-Tracking (Idempotenz, Bot-Filter, Rate-Limit)
   - Attribution-Matching (hart/fuzzy, Zeitfenster, Mehrfach-Match-Guard)
   - Reward-State-Machine (Doppelzahlungs-Guard, Idempotenz)
2. **API-Tests** für Webhooks (WooCommerce, KlickTipp) mit Signatur- und Idempotenz-Prüfungen.
3. **E2E-Tests** für alle User-Flows:
   - Mobil: NPS → C1 → D1 → wa.me-Share → E1 → Lead-Capture
   - Desktop: NPS → C2 → D2 → E-Mail-Share → E1
   - Admin: Promotor-CRUD, manuelle Zuordnung, Reward-Auslösung
4. **Mobile-/Cross-Browser-Optimierung** vor Go-Live.

**Kritische Testfälle, die später grün sein müssen:**

- NPS 9–10 leitet in Empfehlungsflow, 0–8 in Feedback-Flow.
- Ein Promotor kann denselben Link mehrfach teilen (Multi-Share).
- Jede Order kann maximal einen ausgezahlten Reward erzeugen.
- Webhooks sind idempotent (keine doppelte Verarbeitung).
- Bei Ausfall oder fehlenden Daten wird nie ein unsicherer Zustand erzeugt (Fail-safe).

---

## 9. Sicherheit & Datenschutz

Diese Punkte stammen direkt aus dem Projektplan und sind bei jeder zukünftigen Code-Änderung zu beachten:

- **Keine Secrets im Repository:** KlickTipp-, Streamendous-, WooCommerce- und Datenbank-Zugänge ausschließlich über Hetzner-Secret-Management / Umgebungsvariablen.
- **Webhook-Sicherheit:**
  - WooCommerce- und KlickTipp-Webhooks müssen signiert geprüft werden.
  - Idempotenz über `webhook_events.idempotency_key` sicherstellen.
- **DSGVO:**
  - Getrennte Consent-Typen (`tracking_cookie`, `data_processing`).
  - Opt-out-Endpoint bereitstellen.
  - Löschpfade je Consent-Typ modellieren.
  - IP-Adressen nur gehasht (mit verwaltetem Salt) speichern.
- **Reward-Guardrails:**
  - Partial-Unique-Constraint auf `rewards.referral_id` und `rewards.matched_order_id` für Status `approved` / `sent`.
  - Approval-Gate vor Reward-Auszahlung.
  - Audit-Log für alle manuellen Zuordnungen und Reward-Aktionen.
- **Tracking-Robustheit:**
  - Bot-Filter, Rate-Limit, Doppelklick-Dedup.
  - `session_id` als Stitch-Anker.
- **Admin-Auth:** NextAuth.js Credentials, nur ein interner Admin.

**Wichtig:** Das System darf keine Endpunkte anbieten, die automatisch Gutscheine auslösen oder Zahlungen veranlassen. Reward-Auslösung erfordert immer ein manuelles Approval-Gate.

---

## 10. Offene Blocker & Annahmen

Bevor wesentliche Teile der Anwendung gebaut werden, müssen folgende kundenseitige Klärungen erfolgen (aus Projektplan §11):

**Blocker (§11.A):**

- A1: Promotor-Identität / KlickTipp-Subscriber-ID verfügbar?
- A2: WooCommerce-Order-Webhook + Match-Felder?
- A3: Reward-Bedingung (Mindestbestellwert / Produktkopplung)?
- A4: Streamendous-Workflow für Gutschein-Ausführung?

**Annahmen (§11.B, beim Kick-off zu bestätigen):**

- B3: WordPress-Anbindung reicht als Subdomain-Verlinkung, kein Plugin.
- B4: etermin unterstützt ggf. URL-Prefill.
- B5: Match-Zeitfenster default 90 Tage.
- B6: Transaktionaler Mailversand läuft über KlickTipp.
- B7: GitLab-Hosting auf bestehendem Hetzner-Server.

**Agenten-Verhalten:** Baue nichts, was diese Blocker vorweg nimmt. Wenn du während der Implementierung auf eine ungeklärte Entscheidung stößt, dokumentiere sie und stoppe den betroffenen Baustein.

---

## 11. Nächste Schritte / Onboarding-Checkliste

Beim nächsten Einstieg in dieses Repo:

1. Prüfe, ob Code hinzugekommen ist (`package.json`, `app/`, `lib/` etc.).
2. Lese `Dokumente/Projektplan Empfehlungssystem.md` vollständig.
3. Prüfe den Stand der Blocker (§11.A).
4. Initialisiere bei Bedarf ein Git-Repository und richte GitLab-CI ein.
5. Lege nach Code-Start eine konkrete `README.md` und ein `package.json` an.
6. Aktualisiere diese `AGENTS.md`, sobald Build-Befehle, Ordnerstruktur und Testing-Setup feststehen.

---

*Diese Datei ist lernend zu pflegen: Sobald das Projekt aus der Planungsphase in die Implementierung übergeht, müssen alle platzhalterhaften Abschnitte durch konkrete Befehle, Pfade und Konventionen ersetzt werden.*
