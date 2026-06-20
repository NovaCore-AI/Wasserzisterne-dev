# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Sprache: Deutsch (Projektkonvention, Team-OS „Deutsch für alle Artefakte").

---

## 0. Projektstatus — Greenfield

Dieses Repo enthält **noch keinen Code**. Aktuell existiert nur `Dokumente/` mit den Planungsunterlagen.
Es ist **noch kein Git-Repo** und **noch nicht scaffolded** (kein `package.json`, keine Next.js-App).
Der nächste reale Bau-Schritt ist Phase 0 (Infrastruktur) — **aber durch Kunden-Blocker gegated** (siehe §10).

**Konsequenz für dich:** Es gibt noch keine Build-/Lint-/Test-Befehle. Werden sie gebraucht, ergeben sie
sich aus dem Zielstack (Next.js → `npm`/`pnpm`); erfinde keine spezifischen Skripte, bevor das Projekt
aufgesetzt ist.

---

## 1. Source of Truth — WICHTIG

- **Einziges verbindliches Dokument:** `Dokumente/Projektplan Empfehlungssystem.md` (v1.1, Produktionsstand).
  Alle fachlichen Fakten (Datenmodell, API, Attribution, Reward-Logik, Phasen) kommen **ausschließlich** von dort.
- `Dokumente/Empfehlungssystem.pdf` = Original-Vorgabe des Kunden (19 Folien); der Plan löst sie auf und entscheidet.
  Traceability PDF-Folie ↔ Anforderung steht in §14 des Plans.

> ⚠️ **Abgrenzung zu den globalen Team-OS-Regeln:** Die globale `~/.claude/`-Instruktion nennt als
> Use-Case-Quelle das Repo **`Alarmsystem-Dev`** (ein Flughafen-Vereisungs-Alarmsystem) mit Domänenregeln
> wie **RB-01** und **Fail-safe GRÜN/GELB**. Das ist ein **anderes Projekt**. Für dieses Repo gelten diese
> Domänenregeln **nicht**. Übernimm hier ausschließlich die *Methodik/Workflow/Genehmigungs*-Teile der
> globalen Regeln (Gates, Human-in-the-loop, Git-Freigabepflicht), nicht deren fachliche Werte.

---

## 2. Was das System ist (in einem Absatz)

Ein eigenentwickeltes **Empfehlungs-/Referral-System**, das Bestandskunden zu Multiplikatoren macht.
Nach einem Kauf bekommt der Kunde per KlickTipp eine Mail → mobil-first Landing Page → **NPS-Gate**
(9–10 = empfehlen, 0–8 = Feedback) → erzeugt einen **persönlichen Short-Link** → teilt ihn an **mehrere**
Kontakte via **WhatsApp (`wa.me`, mobil primär)** oder E-Mail (Desktop). Neukunde über den Link → E1-Infopage
(Vimeo-Video + consent-gated Lead-Feld) → Termin/WhatsApp-CTA. Kauft ein empfohlener Neukunde (Mindestbestellwert/
Produktkopplung), erhält der Promotor **25 € Amazon-Gutschein per E-Mail**. Ersetzt das gekündigte **TELLSCALE**.
Zieldomain: `empfehlung.wasserzisterne.de`. Team: 2 Personen, ~15–20 h/Woche.

---

## 3. Architektur — der Verantwortungs-Schnitt (Kern)

Die zentrale Architekturentscheidung ist die Aufteilung zwischen eigener App und n8n. **Halte sie ein** —
sie verläuft quer durch Datenmodell, API und Integrationen:

- **Referral-App (Next.js + eigene API) besitzt den State:** Token-/Short-Link-Generierung, Click-/View-Tracking,
  Lead-Capture, Consent-Records, **Attribution-Matching**, Admin-Daten, **Reward-State-Machine**. PostgreSQL = State-Hoheit.
- **n8n besitzt alle Fremdsystem-Verbindungen:** KlickTipp-Tag-Push, WooCommerce-Order-Ingest (Webhook),
  Alerts, Gutschein-Schritte (Streamendous → Amazon) **und Versand der Gutschein-Code-Mail** an den Promotor.
- **KlickTipp** behält Kampagnen-/Analyse-Hoheit; Promotor per `klicktipp_subscriber_id` gespiegelt.

→ Eigencode baut **nie** direkt gegen Fremdsysteme; alles Externe läuft über n8n-Webhooks/-Flows.

---

## 4. Tech-Stack & bewusst verworfene Alternativen

| Schicht | Technologie |
|---|---|
| Hosting | **Hetzner** (eigener Server, Datensouveränität) |
| Frontend + Backend | **Next.js (App Router)**, self-hosted Node, ein Repo (SSR, mobil-optimiert) |
| Datenbank | **PostgreSQL** (self-hosted) |
| Orchestrierung | **n8n** (bestehend) |
| Repo/CI | **GitLab auf Hetzner** (neu aufzusetzen) |
| WhatsApp-Share | **`wa.me` Click-to-Chat** + dynamischer Text (kein API-Zugang) |
| Mail (Kampagne + transaktional) | **KlickTipp REST API** via n8n |
| Admin-Auth | **NextAuth.js (Credentials)** + Audit-Log |
| Video | **Vimeo** |

**Verworfen (nicht wieder vorschlagen):** Vercel, managed Supabase, Web Share API, eigener Exchange-Mailserver,
WhatsApp-Business-API, Template-Konfiguratoren (Seiten sind fix), eigenes WP-Plugin (Default: nein).

---

## 5. Attribution-Pipeline — das Kernstück des Backends

Der fachlich härteste Teil (Plan §4). Der **Token** verbindet Klick↔Promotor; der **Kauf** passiert später in
WooCommerce mit evtl. abweichenden Daten. Ablauf:

1. **E1-Lead-Capture (consent-gated)** = die **harte Match-Basis**: Neukunde gibt mind. E-Mail **oder** Telefon →
   `referrals` (Status `neu`). Ohne dieses Feld ist Zuordnung nur manuell → das Feld ist **P0**.
2. WooCommerce-Order → n8n → `orders`.
3. **Bidirektionales Matching** im Zeitfenster (Default 90 Tage): E-Mail/Telefon = hart, Name = fuzzy (Vorschlag) →
   `match_confidence`.
4. **Bedingungsprüfung:** Mindestbestellwert/Produktkopplung erfüllt → `reward_condition_met`.
5. **Mehrfach-Match-Guard:** eine Order → genau **ein** Reward. Tie-Break: höchste `match_confidence`, sonst frühestes `referral`.
6. **Manueller Fallback:** kein/unsicherer Match → Admin verknüpft `referral` ↔ `order` von Hand.
7. Statuslauf: `neu → kontaktiert → gekauft → gutschein_versendet`.

---

## 6. Reward-Guardrails — kritische Invariante (Geldfluss)

Verhindert Doppelzahlung (echtes Geld, hohes Risiko). Beim Bauen der `rewards`-Logik **zwingend**:

- **Partial-unique** auf `rewards.referral_id` **und** auf `rewards.matched_order_id`, jeweils **nur für
  `status ∈ {approved, sent}`** → nie 2× auszahlen (auch nicht bei Mehrfach-Empfehlung).
- `failed`-Zeilen erlauben einen **sauberen Neuversuch** (neue Zeile, alte nicht mutieren) — deshalb *partial*-unique,
  nicht voll-unique.
- **Approval-Gate** + **Audit-Log** vor jeder Auszahlung; Gutschein-Code-Versand per E-Mail an Promotor,
  `code_sent_to_email` protokolliert.

---

## 7. Datenmodell (Entitäten-Überblick)

Vollständige Spalten in Plan §5. Entitäten und ihre Rolle:

- `promotors` — Bestandskunde/Multiplikator (+ `klicktipp_subscriber_id`, `safedesk_id`, `woo_customer_id`).
- `referral_links` — 1 Promotor : n Links; 1 Link : n `referrals` (**Multi-Share**); `token` + `short_code` unique.
- `referrals` — Neukunden-Lead; Neukundenfelder nullable (mind. eins gesetzt); `match_method ENUM(hard|fuzzy|manual)`.
- `consents` — **getrennte Typen** `tracking_cookie` vs. `data_processing`; eigener Lösch-Pfad je Typ; `session_id`.
- `orders` — aus WooCommerce; `woo_order_id` unique; `products JSONB`.
- `rewards` — Geld-Ledger (siehe §6).
- `nps_responses` — NPS **vor** Promotor-Anlage (`promotor_id` nullable, `session_id`/`subject_email`-Anker).
- `tracking_events` — `session_id` stitcht Events; `idempotency_key` gegen Doppelklick.
- `webhook_events` — eingehende Idempotenz, `idempotency_key` unique **pro Source**.
- `interest_submissions` — Fragebogen aus E1-CTA.
- `admin_audit_log` — before/after je Admin-Aktion.

---

## 8. API-Oberfläche (Plan §6)

- **Öffentlich:** `POST /api/nps`, `POST /api/promotor`, `GET /r/:short_code` (Redirect + Click-Tracking, idempotent,
  Bot-Filter, Rate-Limit), `POST /api/referral`, `POST /api/consent`, `POST /api/interest`, `POST /api/feedback`.
- **Integrationen (n8n):** `POST /api/webhooks/woocommerce` (Signatur, idempotent), `POST /api/webhooks/klicktipp`,
  `GET /api/internal/match`.
- **Admin (auth):** Promotoren CRUD/CSV-Import · Referral-Statuspflege (→ Audit) · Lookup · **manuelle referral↔order-
  Zuordnung** · Reward-Auslösung (Approval) · Dashboard.

---

## 9. Bewusste Grenzen (by design — kein Defekt)

- **`wa.me`** erlaubt **kein** „versandt"-Tracking und **kein** „an wen geteilt". Erfasst wird nur Share-Klick +
  spätere Link-Öffnung. Nicht überversprechen.
- Ein Teil der Käufe ist **strukturell nicht zuordenbar** (Gastkauf, dritte E-Mail). Verlustrate ist akzeptiert;
  manuelle Zuordnung ist der Fallback.

---

## 10. ⛔ Blocker, die den Bau gaten (Plan §11.A)

Vor den zugehörigen Phasen kundenseitig zu klären — jeder hat einen baubaren Default, aber **falsch bauen droht**:

- **A1** Promotor-Identität / KlickTipp-Subscriber-ID — vor Phase 1.
- **A2** WooCommerce-Trigger + Match-Felder — vor Phase 2/3 (bestimmt auch E-Mail vs. Telefon auf E1).
- **A3** Reward-Bedingung (Mindestbestellwert/Produktliste) — vor Phase 2 **und** Phase 3 (steuert Logik **und** Wording).
- **A4** Gutschein-Ausführung Streamendous (API? `failed`-Handling?) — vor Phase 2.

> Reihenfolge-Falle: Die **E1-Lead-/Attribution-Strategie muss vor Phase 3 final sein**, sonst wird E1 zweimal gebaut.

---

## 11. Phasenplan (backend-first, Plan §9, ~191 h / ~10–13 Wochen)

0 Infra → 1 Datenmodell + Core-Backend → 2 **Attribution + Reward** (größter Brocken) → 3 Öffentliche Seiten
(C1/D1 mobil, E1 Neukunde, C2/D2 Desktop, Feedback) → 4 Admin-Panel → 5 Integration/Test/Launch.

---

## 12. Arbeitsweise in diesem Repo

- **Fachliche Fakten nie aus dem Gedächtnis** — immer gegen `Dokumente/Projektplan Empfehlungssystem.md` prüfen.
  Selbst generierte Zahlen/Regeln als KI-Vorschlag kennzeichnen und plausibilisieren.
- **Git ist noch nicht initialisiert.** Sobald es das ist, gilt: Feature-Branch → PR → Review → `main`;
  **kein direkter `main`-Push**; Push/PR/Merge und destruktive Git-Aktionen **nur nach Freigabe durch Lucas**.
- **DSGVO ist Erstklasse-Anforderung:** getrennte Consent-Typen, Opt-out-Endpoint, Lösch-Pfad je Typ, IP-Hash mit
  verwaltetem Salt.
- **Secrets** (KlickTipp/Streamendous/Woo/DB) via Env/Secret-Mgmt auf Hetzner, nie im Repo.
- Beim Scaffolding der Next.js-App: Build-/Lint-/Test-Befehle hier ergänzen.
