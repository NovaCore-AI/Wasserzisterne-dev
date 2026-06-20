# Projektplan — Empfehlungssystem wasserzisterne.de

**Version:** 1.1 (Produktionsstand, reviewed & ergebnisgesichert)
**Stand:** 2026-06-16
**Status:** Verbindliche Arbeitsgrundlage — ersetzt `Erster Entwurf Plan.md` (archiviert als `Erster Entwurf Plan.bak.md`).
**Quellen:** `Empfehlungssystem.pdf` (19 Folien, Kunde) + Klärungsantworten aus dem Kundengespräch (eingearbeitet) + `Backend Architecture Review.md`.
**Review-Stand:** v1.0 durch zwei unabhängige Review-Pässe (Architektur/Konsistenz + Anforderungs-Vollständigkeit) geprüft; Findings in dieser v1.1 eingearbeitet (siehe §15 Review-Protokoll).
**Team:** 2 Personen, ~15–20 Std/Woche gesamt.
**Zieldomain:** `empfehlung.wasserzisterne.de`. **Ablösung von:** TELLSCALE (KIWUS CONSULTING) — bereits gekündigt.

> **Source-of-Truth-Regel:** Dieser Plan ist das einzige verbindliche Dokument. `Backend Architecture Review.md` bleibt als Begründungsanhang. Lose Notizen aus dem Entwurf sind hier aufgelöst und entschieden.

---

## 0. Wie dieser Plan zu lesen ist

- **Entschieden** = festgelegt, baubar.
- ⛔ **BLOCKER** = muss VOR dem zugehörigen Bau-Schritt kundenseitig geklärt sein; ohne Klärung baut man falsch (§11.A).
- 🔶 **Annahme (Default)** = von mir gesetzte Vorgabe; baubar, bei Kick-off in einem Satz zu bestätigen (§11.B).
- Jede Anforderung ist in §14 auf ihre PDF-Folie rückverfolgbar (Ergebnissicherung).

---

## 1. Executive Summary

Eigenentwickeltes Empfehlungssystem, das Bestandskunden zu Multiplikatoren macht. Nach einem Kauf erhält der Bestandskunde über die Hauptkampagne (**KlickTipp**) eine automatisierte E-Mail. Über eine mobil-first Landing Page bewertet er uns (NPS-Gate), erstellt seinen **persönlichen, kurzen Empfehlungslink** und teilt ihn **an mehrere Kontakte** per **WhatsApp** (mobil, primär) oder **E-Mail** (Desktop). Neukunden über diesen Link landen auf einer personalisierten Infopage mit Video, geben (consent-gated) einen minimalen Kontakt an und gehen weiter zu Termin-/WhatsApp-CTA. Kauft ein empfohlener Neukunde (Bedingung: Mindestbestellwert/Produktkopplung), erhält der Promotor **25 € Amazon-Gutschein** — der Code wird ihm **per E-Mail** zugestellt.

Das System läuft auf **eigenem Server (Hetzner)**, nutzt das vorhandene **n8n** für Integrations-Orchestrierung und ist mit **WordPress** (Marketing-Site) sowie **KlickTipp** verzahnt.

**Mobil-first-Begründung (Folie 3):** WhatsApp-Öffnungsrate ~90 % vs. E-Mail ~11 %. Desktop ist bewusst der Fallback-Pfad.

**Multiplikator-Fokus (Kundennotiz „main fokus"):** Ein Promotor soll an **mehrere** Kontakte teilen. Der D1/D2-Flow ist auf wiederholtes Teilen ausgelegt (gleicher Link, mehrere Empfänger), jeder erfasste Neukunde wird ein eigener Lead unter demselben Link.

---

## 2. Systemarchitektur (verbindlich)

```
┌──────────────────────────────────────────────────────────────┐
│  WordPress (bestehend) — Marketing-Site                        │
│     Verlinkt nur: QR-Code / Links → empfehlung.wasserzisterne… │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  REFERRAL-APP  (Hetzner, Subdomain empfehlung.wasserzisterne…) │
│  Frontend + Backend-API  (Next.js, self-hosted Node)           │
│   • Seiten: NPS-Gate, C1/C2, D1/D2, E1, Feedback               │
│   • API: Token-/Short-Link-Gen, Click-Tracking, Lead-Capture,  │
│     Consent, Attribution-Match, Reward-State, Admin            │
│   • Short-Link-Redirect (eigene Domain)                        │
│        └── PostgreSQL (Hetzner, self-hosted) = State-Hoheit    │
│  CI/CD ← GitLab (Hetzner, neu)                                 │
└──────────┬───────────────────────────────────┬───────────────┘
           │ Webhooks/Events                    │ Reads/Writes
┌──────────▼────────────────────┐   ┌───────────▼───────────────┐
│  n8n (bestehend, Hetzner)      │   │  PostgreSQL                │
│   • Referral → KlickTipp-Tag   │   │  Referral-State + Tracking │
│   • WooCommerce-Order-Webhook  │   └────────────────────────────┘
│     → Attribution-Matching     │
│   • Alert bei neuer Empfehlung │   Externe Systeme:
│   • Gutschein-Trigger          │   • KlickTipp (Hauptkampagne + Tags + Analyse)
│     (Streamendous → Amazon)    │   • SafeDesk (Bestandsdaten, API)
│   • Gutschein-Code-Mail        │   • WooCommerce (Käufe)
│     an Promotor                │   • etermin (Termin), Vimeo (Video), wa.me
└────────────────────────────────┘
```

**Verantwortungs-Schnitt (Kern-Architekturentscheidung):**
- **Referral-App/Backend** besitzt: Token-/Short-Link, Click-/View-Tracking, Lead-Capture, Consent-Records, Attribution-Match, Admin-Daten, Reward-State-Machine.
- **n8n** besitzt: alle Verbindungen zwischen Fremdsystemen — KlickTipp-Tag-Push, WooCommerce-Order-Ingest, Alerts, Gutschein-Schritte (Streamendous) **und Versand der Gutschein-Code-Mail** an den Promotor.
- **KlickTipp** behält Kampagnen-/Analyse-Hoheit; Promotor per `klicktipp_subscriber_id` gespiegelt.

**Tech-Stack:**

| Schicht | Technologie | Begründung |
|---------|------------|------------|
| Server/Hosting | **Hetzner (eigener Server)** | Kundenvorgabe; Datensouveränität |
| Frontend+Backend | **Next.js (App Router)**, self-hosted | SSR, mobil-optimiert, ein Repo |
| Datenbank | **PostgreSQL (self-hosted)** | State-Hoheit |
| Orchestrierung | **n8n (bestehend)** | Integrations-Glue ohne Eigencode |
| Repo/CI | **GitLab auf Hetzner** (neu) | existiert noch nicht |
| WhatsApp-Share | **`wa.me` Click-to-Chat** + dynamischer Text | mobil + Desktop-Web, kein API-Zugang |
| E-Mail/Kampagne + transakt. Mails | **KlickTipp REST API** (via n8n) | Hauptkampagne, Tags, **Gutschein-Code-Mail** |
| Admin-Auth | **NextAuth.js (Credentials)** + Audit-Log | 1 interner Admin; Geldfluss → Audit |
| Video | **Vimeo** | bereits vorhanden |

> **Korrektur ggü. Entwurf:** Vercel + managed Supabase + „Web Share API" verworfen (Begründung: `Backend Architecture Review.md` §2). **Exchange als Mailserver entfällt bewusst** — KlickTipp übernimmt Kampagne *und* transaktionale Mails (Gutschein-Code). Falls transaktionaler Versand doch über Exchange laufen soll → 🔶 §11.B6.

---

## 3. User Flows

### Flow A — Bestandskunde Mobil (Primär)
```
KlickTipp-Massenmail → Kunde öffnet auf Handy → QR scannen / Link
  → NPS-Gate "Wie wahrscheinlich empfiehlst du uns?" (0–10)
    ├─ 9–10 → C1 Mobile Info (BENEFIT-fokussiert, kurz; Endkunden-Tonalität,
    │          NICHT die B2B-Makler-Vorlage; Reward-Bedingung im Wording)
    │   → D1 Formular (Promotor: Vorname, Nachname, E-Mail, Telefon)
    │   → System erzeugt Token + KURZEN Short-Link
    │   → wa.me-Share mit vorgefülltem, dynamischem Text (NUR WhatsApp anzeigen)
    │   → MULTI-SHARE: Promotor kann den Link nacheinander an mehrere
    │      Kontakte teilen (Button bleibt nutzbar)
    └─ 0–8  → Feedback-Flow (Kategorien-Mehrfachauswahl + anonymer Freitext → Danke)
```

### Flow B — Bestandskunde Desktop (Sekundär)
```
KlickTipp-Mail → Desktop-Browser (Browser-Erkennung) → NPS-Gate (gleiche Logik)
    ├─ 9–10 → C2 Desktop Info (eigenes Branding, professionell/aufgeräumt,
    │          erklärt das Programm TIEFER als mobil, Vimeo-Video eingebunden)
    │   → D2 Formular + separate E-Mail-Eingabemaske (mehrere Empfänger)
    │   → Übergabe an KlickTipp (via n8n) → E-Mail-Share
    └─ 0–8  → Feedback-Flow (identisch)
```
> Video ist auf der **Desktop-Promotor-Seite (C2/D2)** eingebunden (F11) **und** auf E1.

### Flow C — Neukunde (Link-Empfänger)
```
Öffnet Short-Link → /r/:short_code Redirect (Click-Tracking) → E1 Seite
  → Tracking-/Cookie-Consent (consent_type=tracking_cookie, DSGVO, Opt-out-fähig)
  → Personalisiert: "Empfehlung von [Promotor-Vorname]" + 60s Vimeo-Video
  → MINIMALES Lead-Feld (E-Mail ODER Telefon) — mit EIGENEM data_processing-Consent
     am Feld (getrennt vom Tracking-Cookie; greift auch bei Cookie-Opt-out),
     als Benefit gerahmt ("damit wir dir die Infos / den Termin schicken")
       → schreibt referrals (Status neu) = HARTE Match-Basis für Attribution
  → Doppel-CTA:  primär "Beratungstermin buchen" → etermin.net/wasserzisterne
                 sekundär "Kurz auf WhatsApp schreiben" → wa.me
  → optional: Klick auf "Beratung buchen" → kurzer FRAGEBOGEN (Interessens-
     Qualifizierung) → intern gespeichert + Alert an Team
```
> **Attribution-Anker:** Ohne mind. ein hartes Feld (E-Mail/Telefon) auf E1 ist die spätere Kauf-Zuordnung nur manuell. Das Lead-Feld ist daher P0. Token-Durchreichung an etermin zusätzlich, falls etermin Prefill unterstützt (🔶 §11.B4).

### Flow D — Admin (Backend)
```
Passwortgeschütztes Panel
  → Promotoren-Übersicht (Liste, Suche, CSV-Import)
  → Pro Promotor: Links, Klicks, Empfehlungen
    (Hinweis: "an wen per WhatsApp geteilt" ist via wa.me technisch NICHT
     erfassbar — nur Klicks/geöffnete Leads; siehe §7)
  → Empfehlungs-Liste + Status (neu → kontaktiert → gekauft → gutschein_versendet)
  → Lookup: E-Mail / Telefon(=WhatsApp-Nr) / Name → "über Empfehlung gekommen?"
  → MANUELLE Zuordnung: referral ↔ order von Hand verknüpfen (für No-Match-Fälle)
  → Reward auslösen (Bedingungsprüfung + Idempotenz + Approval + Audit)
  → Dashboard: Klicks, Empfehlungen, Conversions, NPS
```

---

## 4. Attribution-Konzept (Kernstück des Backends)

Der Token verbindet den **Klick** mit dem Promotor. Der **Kauf** passiert später in WooCommerce, evtl. mit abweichenden Daten. Pipeline:

1. **Lead-Capture auf E1** (consent-gated): Neukunde gibt mind. E-Mail oder Telefon an → `referrals` (Status `neu`) mit `referral_link_id` + `promotor_id`. **Dies ist die harte Match-Basis.**
2. **Order-Ingest:** WooCommerce-Order-Webhook → n8n → `orders`.
3. **Bidirektionales Matching** (innerhalb Zeitfenster, 🔶 §11.B5):
   - neue Order → gegen offene `referrals` (E-Mail/Telefon hart, Name fuzzy = Vorschlag)
   - neues `referral` → gegen bereits eingegangene, noch unmatched `orders`
   - Ergebnis: `match_confidence`.
4. **Bedingungsprüfung:** Mindestbestellwert / Produktkopplung (⛔ §11.A3) erfüllt → `reward_condition_met = true`.
5. **Mehrfach-Match-Guard:** Matcht eine Order mehrere `referrals` (zwei Promotoren empfehlen dieselbe Person), greift **eine Order → genau ein Reward** (Unique auf `matched_order_id` für ausgezahlte Rewards). Tie-Break: höchste `match_confidence`, bei Gleichstand frühestes `referral`.
6. **Manueller Fallback:** kein/unsicherer Match (Gastkauf, dritte E-Mail) → Admin verknüpft `referral` ↔ `order` von Hand (Flow D). **Eine strukturelle Attribution-Verlustrate wird akzeptiert** (nicht jeder Kauf ist zuordenbar) — als Risiko dokumentiert (§10).
7. **Statuslauf:** `neu → kontaktiert → gekauft → gutschein_versendet`.
8. **Reward-Zustellung:** nach `gekauft` + Bedingung → n8n triggert Streamendous → Amazon-Code → **Code per E-Mail an Promotor** (KlickTipp/n8n) → Status `gutschein_versendet`.

---

## 5. Datenmodell (vollständig, baubar)

```
promotors
  id, vorname, nachname, email, telefon,
  klicktipp_subscriber_id, safedesk_id, woo_customer_id,
  status ENUM(active|blocked), created_at, last_activity_at

referral_links
  id, promotor_id, token (unique), short_code (unique),
  channel ENUM(whatsapp|email), link_clicks_count, created_at
  -- 1 Promotor : n Links möglich; 1 Link : n referrals (Multi-Share)

referrals
  id, referral_link_id, promotor_id,
  neukunde_vorname (nullable), neukunde_nachname (nullable),
  neukunde_email (nullable), neukunde_telefon (nullable),   -- mind. eins gesetzt (E1-Lead)
  consent_id, status ENUM(neu|kontaktiert|gekauft|gutschein_versendet),
  matched_order_id (nullable), match_confidence, match_method ENUM(hard|fuzzy|manual),
  reward_condition_met BOOL, created_at, updated_at

consents                 -- DSGVO; getrennte Typen
  id, consent_type ENUM(tracking_cookie|data_processing),
  subject_email (nullable),    -- bei reinem Cookie-Consent leer
  session_id, granted BOOL, policy_version, ip_hash, created_at
  -- Lösch-Pfad: data_processing über subject_email/consent_id; tracking über session_id

orders                   -- aus WooCommerce
  id, woo_order_id (unique), email, telefon, name,
  order_total, products JSONB,   -- Produkt-ID/SKU-Feld bei A3 festlegen
  matched BOOL, created_at

rewards                  -- Geld-Ledger
  id, referral_id, matched_order_id,
  -- Guard: PARTIAL UNIQUE auf referral_id UND auf matched_order_id, jeweils nur für
  -- status IN (approved, sent) → nie 2× ausgezahlt; failed-Zeilen erlauben Neuversuch
  amount, provider (streamendous),
  status ENUM(pending|approved|sent|failed),
  amazon_code_ref, code_sent_to_email, approved_by, sent_at, failed_reason, created_at

nps_responses
  id, promotor_id (nullable), session_id, subject_email (nullable),  -- NPS VOR Promotor-Anlage
  score (0–10), routed_to ENUM(referral|feedback),
  feedback_categories TEXT[], feedback_text TEXT, created_at

tracking_events
  id, referral_link_id, referral_id (nullable), session_id,   -- session_id stitcht Events
  event_type ENUM(link_click|whatsapp_share|page_view|consent|booking_click),
  idempotency_key, user_agent, ip_hash, created_at

webhook_events           -- eingehende Idempotenz
  id, source ENUM(klicktipp|woocommerce),
  idempotency_key (unique per source),   -- Woo: order_id; KlickTipp: subscriber+event+zeitfenster
  payload JSONB, processed_at, created_at

interest_submissions     -- Fragebogen aus E1-CTA (Interessenten-Erkennung)
  id, referral_id (nullable), session_id, answers JSONB, created_at

admin_audit_log
  id, admin_user, action, entity, entity_id, before JSONB, after JSONB, created_at
```

**Änderungen ggü. v1.0 (aus Review):** `nps_responses.promotor_id` nullable + `session_id`/`subject_email`-Anker (NPS kommt vor Promotor-Anlage); `consents` mit getrennten Typen + `session_id`; `orders.matched`; `rewards.matched_order_id` unique (Mehrfach-Match-Guard) + `code_sent_to_email` + `failed_reason`; `referrals.match_method` + nullable Neukundenfelder; `tracking_events.session_id` als Stitch-Anker; `webhook_events.idempotency_key` statt `external_id`; neue Entität `interest_submissions` (Fragebogen).

---

## 6. Backend-API-Oberfläche

**Öffentlich:**
- `POST /api/nps` — Score + Routing (9–10 vs 0–8)
- `POST /api/promotor` — Bestandskunde anlegen → Token + short_code
- `GET  /r/:short_code` — Redirect + Click-Tracking (idempotent, Bot-Filter, Rate-Limit)
- `POST /api/referral` — Neukunde-Lead-Capture (consent-gated, E1)
- `POST /api/consent` — Consent-Record (typisiert, versioniert)
- `POST /api/interest` — Fragebogen-Antworten (E1-CTA) → Alert
- `POST /api/feedback` — Kategorien + anonymer Freitext

**Integrationen (n8n):**
- `POST /api/webhooks/woocommerce` — Order-Ingest (Signatur, idempotent)
- `POST /api/webhooks/klicktipp` — Tag-/Subscriber-Events
- `GET  /api/internal/match` — Attribution-Query (Auto-Match + Admin-Lookup)

**Admin (auth):** Promotoren CRUD/CSV-Import · Referral-Liste + Statuspflege (→ Audit) · Lookup · **manuelle referral↔order-Zuordnung** · Reward auslösen (Approval) · Dashboard.

---

## 7. Querschnitt (Pflicht für „produktionsfertig")

| Thema | Maßnahme |
|-------|----------|
| **DSGVO** | Getrennte Consent-Typen (Cookie/Tracking vs. Datenverarbeitung), Opt-out-Endpoint, Lösch-Pfad je Typ, IP-Hash mit verwaltetem Salt. Eigene Vereinbarung neben TELLSCALE; Opt-out + Cookie-Richtlinie. |
| **Reward-Guardrails** | Idempotenz/Guard: **partial-unique** auf `rewards.referral_id` und `rewards.matched_order_id`, jeweils nur für `status ∈ {approved, sent}` → nie 2× zahlen (auch nicht bei Mehrfach-Empfehlung), aber `failed`-Zeilen erlauben einen sauberen Neuversuch (kein Mutieren der alten Zeile). Approval-Gate, Bedingungsprüfung, Audit-Log, `failed`-Recovery-Pfad. |
| **Gutschein-Code-Zustellung** | Nach `sent` Amazon-Code **per E-Mail an Promotor** (über KlickTipp/n8n), `code_sent_to_email` protokolliert (F3-Pflicht). |
| **Webhook-Sicherheit** | Signaturprüfung (Woo/KlickTipp) + Idempotenz über `webhook_events.idempotency_key` (Schlüssel pro Source definiert). |
| **Tracking-Robustheit** | Bot-Filter, Rate-Limit, Doppelklick-Dedup über `idempotency_key`, `session_id`-Stitching. |
| **WhatsApp-Share-Grenze** | `wa.me` ist externer Sprung: „Nachricht versandt" und „an wen geteilt" sind **nicht** zuverlässig messbar. Erfasst wird nur der Share-Klick + spätere Link-Öffnung. Ehrlich kommuniziert (kein Folie-12-Überversprechen). |
| **Secrets** | KlickTipp/Streamendous/Woo/DB-Keys via Env/Secret-Mgmt auf Hetzner, nie im Repo. |
| **Backup/Monitoring** | Eigener Server ⇒ Postgres-Backup (automatisiert) + Uptime-Monitoring selbst verantworten. |
| **Repo/CI** | GitLab auf Hetzner; CI/CD; Staging vor Prod. |

---

## 8. Feature-Scope

**IN SCOPE**

| # | Feature | Prio |
|---|---------|------|
| 1 | NPS-Gate (0–10 + Routing) | P0 |
| 2 | C1 Mobile Info (benefit, Endkunden-Tonalität, Reward-Wording) | P0 |
| 3 | D1 Mobile Link-Generator + Short-Link + wa.me-Share + Multi-Share | P0 |
| 4 | QR-Code-Generierung/-Verwaltung (Kampagnen-Einstieg) | P0 |
| 5 | E1 Neukunden-Page (Vimeo + Lead-Capture + Doppel-CTA + Consent) | P0 |
| 6 | Tracking + Token/Short-Link-System (session-stitched) | P0 |
| 7 | Attribution-Pipeline (Lead→Order→bidirekt. Match→Bedingung→Guard) | P0 |
| 8 | Consent/DSGVO (typisiert, Opt-out, Lösch-Pfad) | P0 |
| 9 | Gutschein-Code-Versand per E-Mail an Promotor | P0 |
| 10 | wasserzisterne-Branding (Farben, Fonts, Logo) | P0 |
| 11 | C2 Desktop-Info (tiefer erklärend, Video) | P1 |
| 12 | D2 Desktop Link-Generator + Multi-Empfänger + KlickTipp-Übergabe (n8n) | P1 |
| 13 | Feedback-Flow (anonym, kategorisiert mit Mehrfachauswahl, Freitext) | P1 |
| 14 | Admin: Promotoren (CRUD + CSV-Import) | P1 |
| 15 | Admin: Empfehlungs-Tracking + Status + Audit + manuelle Zuordnung | P1 |
| 16 | Admin: Lookup-Funktion | P1 |
| 17 | Reward-State-Machine + Bedingungsprüfung + Approval | P1 |
| 18 | Interessenten-Erkennung: CTA-Klick → Fragebogen → speichern + Alert | P1 |
| 19 | Admin-Dashboard (Kennzahlen) | P2 |

> **Reprio ggü. v1.0:** Interessenten-Erkennung **inkl. Fragebogen** von P2 auf P1 (expliziter Kundenwunsch); QR-Generierung und Gutschein-Code-Mail als eigene P0-Features ergänzt (waren in v1.0 nicht eingeplant).

**OUT OF SCOPE** (entschieden)
- Template-Konfigurator Neukunden-/Promoter-Seite (Folien 14–16) — fixe Seiten.
- Vollautomatische Gutschein-Auszahlung — Backend prüft + triggert, Streamendous führt aus, finaler Versandstatus halbautomatisch.
- Eigenes E-Mail-Versandsystem / Exchange — KlickTipp übernimmt (Kampagne + transaktional).
- **WhatsApp-Business-API / Bot-Erstkontakt / Kontakt-Verlinkung** — kein API-Zugang; bewusst über `wa.me`-Re-Share gelöst (Kundenfrage damit beantwortet: nicht via Business-API, sondern Multi-Share-Link).
- Bewertungsplattform-Integration (Google/Trustpilot).
- Eigenes WP-Plugin (nur bei echtem Embedding/SSO-Bedarf, 🔶 §11.B3 — Default: nein).

---

## 9. Phasenplan + realistische Schätzung (backend-first)

> Korrigiert ggü. v1.0 (Review H3): Attribution ist der fachlich härteste Teil und war zu niedrig angesetzt. Realistisch **~185–200 h** → bei 15–20 h/Woche **~10–13 Wochen**.
> Korrigiert ggü. v1.0 (Review H5): Die **E1-Lead-Capture-/Attribution-Strategie (⛔ §11.A1/A2) muss vor Phase 3 final sein**, sonst wird E1 zweimal gebaut.

### Phase 0 — Infrastruktur (~22 h)
| Aufgabe | Std |
|---------|-----|
| Hetzner: Server, PostgreSQL, Reverse-Proxy/SSL, Subdomain-DNS | 5 |
| GitLab auf Hetzner + CI/CD + Staging | 6 |
| n8n-Verdrahtung prüfen, Secret-Mgmt, Backup-Job | 4 |
| KlickTipp-API + WooCommerce-Webhook + Streamendous Test-Zugänge | 5 |
| ⛔-Klärungen A1–A4 final ziehen (Workshop) | 2 |

### Phase 1 — Datenmodell + Core-Backend (~37 h)
| Aufgabe | Std |
|---------|-----|
| Schema + Migrations (alle Entitäten §5) | 7 |
| Token-/Short-Link + Redirect-Service | 6 |
| Tracking-Endpoint (idempotent, Bot-Filter, session-stitch, ip_hash) | 8 |
| NPS-API + Routing | 3 |
| Promotor-Capture + Consent-API (typisiert) | 6 |
| QR-Code-Generierung | 3 |
| Design-System (Branding) Basis | 4 |

### Phase 2 — Attribution + Reward (~50 h)
| Aufgabe | Std |
|---------|-----|
| E1-Lead-Capture-API + Datenfluss | 5 |
| WooCommerce-Order-Ingest (n8n) + webhook_events | 7 |
| Bidirektionales Matching (hart/fuzzy + Confidence + Zeitfenster) | 14 |
| Mehrfach-Match-Guard + Bedingungsprüfung (A3) | 7 |
| Reward-State-Machine + doppelte Idempotenz + Audit + failed-Recovery | 9 |
| Gutschein-Code-Mail-Versand (n8n/KlickTipp) | 4 |
| Manuelle referral↔order-Zuordnung (Backend-Logik) | 4 |

### Phase 3 — Öffentliche Seiten (~33 h)
| Aufgabe | Std |
|---------|-----|
| C1 Mobile Info (benefit) | 3 |
| D1 Formular + wa.me-Share + Multi-Share | 6 |
| E1 Neukunden-Page (Vimeo + Lead-Feld + Doppel-CTA + Consent) | 7 |
| Interessenten-Fragebogen + Alert | 4 |
| C2 Desktop Info (tiefer erklärend, Video, Branding/Responsive) | 5 |
| D2 Desktop + Multi-Empfänger-Maske + KlickTipp-Übergabe (n8n) | 6 |
| Feedback-Flow | 2 |

### Phase 4 — Admin-Panel (~27 h)
| Aufgabe | Std |
|---------|-----|
| Auth (NextAuth) + Audit-Log-Basis | 3 |
| Promotoren-Übersicht + CRUD + CSV-Import | 6 |
| Referral-Liste + Statuspflege | 5 |
| Lookup + manuelle Zuordnung-UI | 5 |
| Reward-Auslösung (Approval-UI) | 5 |
| Dashboard-Kennzahlen | 3 |

### Phase 5 — Integration, Test, Launch (~22 h)
| Aufgabe | Std |
|---------|-----|
| n8n-Flows final (KlickTipp/Woo/Alert/Gutschein/Code-Mail) | 6 |
| E2E-Test aller Flows (Mobil/Desktop/Admin/Attribution/Reward) | 7 |
| Mobile-/Cross-Browser-Optimierung | 3 |
| Bugfixes | 3 |
| Go-Live: DNS, Monitoring, Mini-Doku | 3 |

| Phase | Std |
|-------|-----|
| 0 Infra | ~22 |
| 1 Datenmodell+Core | ~37 |
| 2 Attribution+Reward | ~50 |
| 3 Seiten | ~33 |
| 4 Admin | ~27 |
| 5 Launch | ~22 |
| **GESAMT** | **~191 h (~10–13 Wochen)** |

---

## 10. Risiken

| Risiko | Einschätzung | Maßnahme |
|--------|-------------|----------|
| Strukturelle Attribution-Verlustrate (Gastkauf, dritte E-Mail, wa.me-Sprung) | Hoch — by design | Lead-Capture auf E1 maximiert harte Matches; manuelle Zuordnung als Fallback; Verlustrate als akzeptiert dokumentiert |
| E1-Lead-Feld senkt Neukunden-Conversion (Friction) | Mittel | minimal (1 Feld), als Benefit gerahmt; A/B-fähig |
| Reward-Doppelzahlung (Mehrfach-Empfehlung) | Hoch (Geld) | doppelte Idempotenz (referral_id + matched_order_id) + Tie-Break + Approval |
| WooCommerce-Webhook-Felder unzureichend für Match | Hoch | A2 in Phase 0 testen; manueller Fallback by-design |
| Streamendous-Workflow unklar (manuell?) | Mittel | ⛔ A4 vor Phase 2; failed-Recovery modelliert |
| GitLab/n8n auf gemeinsamem Hetzner-Server | Mittel | Ressourcen prüfen; ggf. separater Worker |
| KlickTipp-API-Limits | Mittel | Phase 0 testen; n8n retry/entkoppelt |

---

## 11. Offene Punkte

### 11.A — ⛔ BLOCKER (vor zugehörigem Bau-Schritt klären)

⛔ **A1 — Promotor-Identität / Source-of-Truth** (vor Phase 1). *Default:* eigene DB führt Referral-State, gespiegelt per `klicktipp_subscriber_id`. *Klären:* KlickTipp-Subscriber-ID verfügbar?

⛔ **A2 — WooCommerce-Kauf-Trigger + Match-Felder** (vor Phase 2/3). *Default:* Order-Webhook → Match über E-Mail+Telefon (hart), Name (fuzzy). *Klären:* Webhook verfügbar? Welche Felder? → bestimmt auch, ob das E1-Lead-Feld E-Mail oder Telefon priorisiert.

⛔ **A3 — Reward-Bedingung** (vor Phase 2 UND Phase 3). *Default:* fester Mindestbestellwert ODER definierte Produktliste (+ SKU-Feld in `orders.products`). *Klären:* welche Regel, welche Werte? Steuert Logik **und** Wording (C1/E1).

⛔ **A4 — Gutschein-Ausführung (Streamendous)** (vor Phase 2). *Default:* Backend prüft + triggert, Streamendous führt Amazon-Kauf aus, Code per Mail an Promotor, Status manuell auf `sent`. *Klären:* API/Webhook von Streamendous? `failed`-Handling?

### 11.B — 🔶 Annahmen mit Default (bei Kick-off bestätigen)

🔶 **B3 — WordPress-Anbindung:** Default Subdomain + Verlinkung, kein Plugin. *Klären:* Embedding/SSO nötig?
🔶 **B4 — etermin Token-Durchreichung:** Default Best-effort (Prefill, falls etermin es kann), sonst Attribution über E1-Lead. *Klären:* unterstützt etermin URL-Prefill/Parameter?
🔶 **B5 — Match-Zeitfenster:** Default 90 Tage zwischen Link-Öffnung und Kauf. *Klären:* realistische Kauf-Latenz?
🔶 **B6 — Transaktionaler Mailversand:** Default KlickTipp (Gutschein-Code-Mail). *Klären:* doch Exchange?
🔶 **B7 — GitLab-Hosting:** Default auf bestehendem Hetzner-Server. *Klären:* Ressourcen/separater Server?

*Sekundär:* Anzahl Bestandskunden, exakte Hex-Farben/Logo-SVG (kommen im Template), Vimeo-URL des 60s-Videos.

---

## 12. Nächste Schritte / Jira-Übergabe

- [ ] Kick-off-Workshop: ⛔ A1–A4 entscheiden, 🔶 B3–B7 bestätigen.
- [ ] Zugänge: KlickTipp-API-Key, WooCommerce-Webhook, Streamendous-API, Branding-Template, Vimeo-URL, etermin-Prefill-Doku.
- [ ] DNS für `empfehlung.wasserzisterne.de`.
- [ ] Phase 0 starten.
- [ ] §8-Features + §9-Phasen + §11-Blocker in Jira spiegeln (mit Kimi abgleichen). Blocker → eigene „Klärung"-Tickets mit Due-Date vor der jeweiligen Phase.

---

## 13. (frei — siehe §14 Traceability)

---

## 14. Anforderungs-Traceability-Matrix (Ergebnissicherung)

| PDF/Quelle | Anforderung | Abgedeckt in |
|------------|-------------|--------------|
| F1 | Ablösung TELLSCALE | §1, §8 |
| F1 | Domain empfehlung.wasserzisterne.de | §1, §2, §12 |
| F1 | Links über eigene Homepage | §2, §11.B3 |
| F1 | Versand über eigenen Server: **KlickTipp gewählt, Exchange bewusst verworfen** | §2 (Korrektur-Box), §8 OUT, §11.B6 |
| F1/F2 | A Kampagne automatisiert | §1, §3 Flow A |
| F1/F2 | B1 QR scannen (mobil) inkl. **QR-Generierung** | §3, §8 #4 |
| F1/F2 | B2 Link (Desktop) | §3 Flow B |
| F3 | C1 Mobile Info, kurz/benefit, **Endkunden-Tonalität** | §3 Flow A, §8 #2 |
| F3 | Mobil/WhatsApp-first (90/11) | §1, §2 |
| F3 | 25 € Amazon-Gutschein | §1, §4, §5 rewards |
| F3 | **Code per E-Mail an Promotor** | §4 Schritt 8, §7, §8 #9 |
| F3 | beide Wege (WhatsApp + Desktop/E-Mail) | §3 Flow A/B |
| F4 | D1 Formular → Token, Zuordnung | §3, §5, §6 |
| F4 | vorgefertigter WhatsApp-Text | §3 (wa.me dynamisch) |
| F5 | kürzerer Link, Logo, nur WhatsApp | §5 short_code, §3, §8 #3 |
| F6 | WhatsApp-Share-Flow; **„versandt"-Status mit wa.me NICHT messbar (Grenze benannt)** | §3, §7 (WhatsApp-Share-Grenze) |
| F6 | Kennung am Link; langer→kurzer Link | §5 short_code, §8 #3 |
| F7 | E1: Tracking-Consent, Info, etermin-Termin | §3 Flow C, §7 |
| F7-Notiz | WhatsApp-Direktkontakt-CTA | §3 Doppel-CTA |
| F8 | C2 Desktop: eigenes Design, professionell, **Programm tiefer erklären**, responsive | §3 Flow B, §8 #11 |
| F9/F10 | D2 Desktop: Link, E-Mail-Weiterleitung | §3, §8 #12 |
| F11 | D2 professioneller, **Video auf Desktop-Promotor-Seite**, KlickTipp, E-Mail-Maske | §3 Flow B (Video-Hinweis), §8 #12 |
| F12 | Backend Promotoren; **„an wen geteilt" = nur soweit via Klick/Lead erfassbar (wa.me-Grenze)**; Kauf-Lookup E-Mail/WhatsApp(=Tel)/Name | §3 Flow D, §4, §6, §7 |
| F13 | Liste Kunden über Link | §5 referrals, §8 #15 |
| F14–F16 | Konfiguratoren nicht nötig | §8 OUT |
| F17 | NPS 9–10 vs 0–8 Routing | §3, §8 #1/#13 |
| F18 | Kategorisieren (für KI), Mehrfachauswahl | §5 TEXT[], §8 #13 |
| F19 | anonymer Freitext | §5 feedback_text, §8 #13 |
| Notiz | Mindestbestellwert/Produktkopplung im Wording | §3, §4, §11.A3 |
| Notiz | Gutschein via Streamendous | §4, §7, §11.A4 |
| Notiz | SafeDesk/WooCommerce Quellen, CRM-Zukunft | §2, §5 Quell-IDs |
| Notiz | n8n vorhanden, GitLab fehlt | §2, §9 Phase 0, §11.B7 |
| Notiz | Interessenten-Erkennung **inkl. Fragebogen** | §3 Flow C, §5 interest_submissions, §8 #18 |
| Notiz | **Multi-Share / mehrere Empfänger (main fokus)**; WhatsApp-Business-Bot-Frage beantwortet (→ wa.me-Re-Share, keine Business-API) | §1, §3 Flow A/B, §8 #3/#12, §8 OUT |
| Notiz | DSGVO Opt-out + Cookie, eigene Vereinbarung | §7 |
| Notiz | Vimeo | §2, §3 |

---

## 15. Review-Protokoll (Ergebnissicherung)

**Vorgehen:** v1.0 wurde durch zwei unabhängige adversariale Review-Pässe geprüft — (1) Architektur/Konsistenz/Baubarkeit, (2) Anforderungs-Vollständigkeit gegen alle 19 PDF-Folien + Kundennotizen.

**Eingearbeitete Findings (v1.0 → v1.1):**
- **Attribution-Strukturfehler behoben:** E1-Lead-Capture als harte Match-Basis ergänzt (§3 Flow C, §4, §8 #5); Flow C macht Datenherkunft jetzt explizit.
- **Doppelzahlung verhindert:** Mehrfach-Match-Guard + `rewards.matched_order_id` unique + Tie-Break (§4, §5, §7).
- **Bidirektionales Matching** + Zeitfenster ergänzt (§4, §11.B5).
- **Gastkauf/No-Match:** manuelle referral↔order-Zuordnung als Pflicht-Feature (§3 Flow D, §6, §8 #15); Attribution-Verlustrate als Risiko akzeptiert (§10).
- **Verlorene Anforderungen ergänzt:** Gutschein-Code-Mail (#9), QR-Generierung (#4), Fragebogen bei Interessenten-Erkennung (#18, P1), „Programm tiefer erklären" auf C2 (#11), Video auch auf D2/C2 (§3/§8 #12), Multi-Share (#3/#12), Exchange-Entscheidung explizit (§2/§8 OUT/§11.B6), WhatsApp-Business-Frage beantwortet.
- **Schema-FK-Fixes:** `nps_responses.promotor_id` nullable + session-Anker; Consent-Typen getrennt; `tracking_events.session_id`-Stitch; `webhook_events.idempotency_key` pro Source.
- **Schätzung korrigiert:** Phase 2 30→50 h, gesamt ~160→~191 h (~10–13 Wochen).
- **Blocker-Härtung:** A3/A4 von „Annahme" zu ⛔ BLOCKER hochgestuft; §11 in A (Blocker) und B (Annahmen) getrennt; Reihenfolge-Abhängigkeit (Attribution vor Phase 3) dokumentiert (§9).
- **§14-Traceability ehrlich gemacht:** weggelassene Teilkriterien (Code-per-Mail, „tiefer erklären", Fragebogen) ergänzt; wa.me-Grenzen bei F6/F12 offen benannt statt Abdeckung zu behaupten.

**Zweite Verifikationsrunde (v1.1):** Ein erneuter adversarialer Pass bestätigte alle 9 v1.0-Findings als gelöst und fand zwei durch die Fixes neu entstandene Inkonsistenzen, die anschließend behoben wurden:
- N1 (HIGH): `rewards.matched_order_id` war als voll-unique beschrieben (§5/§7), was den `failed`-Recovery-Pfad blockiert hätte → auf **partial-unique (status ∈ {approved, sent})** geschärft, deckungsgleich mit §4.5.
- N3 (MEDIUM): Flow C zeigte den `data_processing`-Consent am E1-Lead-Feld nicht → in §3 Flow C ergänzt (getrennt vom Tracking-Cookie).

**Verbleibende bewusste Grenzen (kein Defekt, dokumentiert):** wa.me erlaubt kein „versandt"-Tracking und kein „an wen geteilt"; ein Teil der Käufe ist strukturell nicht zuordenbar. Beides ist in §7/§10 offen kommuniziert.

**Ergebnis:** Plan ist konsistent, vollständig gegen die Quelle (§14), baubar, und ohne weitere Eingaben handlungsfähig — die einzigen echten Abhängigkeiten sind die 4 kundenseitigen ⛔-Blocker (§11.A), jeweils mit baubarem Default.
