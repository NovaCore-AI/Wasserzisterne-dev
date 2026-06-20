# Empfehlungssystem wasserzisterne.de — Kickoff

**Stakeholder:** Hein (Auftraggeber)  ·  **Termin:** Dienstag, 23.06.2026  ·  **Team:** 2 Personen (~15–20 h/Woche gesamt)
**Grundlage:** `Projektplan Empfehlungssystem.md` v1.1 (reviewed & ergebnisgesichert)

> Dies ist der Projekt-Kickoff: Projektplan, Phasen, Milestones, KPIs, Risiken und die Entscheidungen, die wir für den Start brauchen. Das fachliche Konzept (Seiten, Flows, Design) ist aus `Empfehlungssystem.pdf` bekannt und nicht Gegenstand dieses Termins.

---

## 1 · Auftrag & Lieferversprechen

Ablösung von **TELLSCALE** (gekündigt) durch ein **eigenentwickeltes** Empfehlungssystem auf **eurem Hetzner-Server**, verzahnt mit eurem Bestand (KlickTipp, WooCommerce, SafeDesk, n8n). Ziel-Domain: `empfehlung.wasserzisterne.de`.

**Was wir liefern (Lieferumfang nach Priorität):**

| Prio | Lieferpaket |
|:----:|-------------|
| **P0** | Empfehlungs-Engine: NPS-Weiche · Link-/Token-System · Tracking · **Attribution (Empfehlung→Kauf)** · Consent/DSGVO · Gutschein-Versand · Branding-Basis |
| **P1** | Desktop-Pfad · Feedback-Flow · **Admin-Panel** (Promotoren, Status, Lookup, manuelle Zuordnung, Reward-Approval) · Interessenten-Fragebogen |
| **P2** | Admin-Dashboard mit Kennzahlen |

**Nicht im Umfang (bewusst entschieden):** Template-Konfiguratoren · eigenes Mailsystem (KlickTipp übernimmt) · WhatsApp-Business-API (gelöst über `wa.me`-Multi-Share) · Bewertungsplattform-Integration · eigenes WP-Plugin.

---

## 2 · Vorgehen

```
  backend-first  →  erst das verlässliche Fundament (Zuordnung, Gutschein-Sicherheit),
                    dann die Oberflächen. Das Geschäftskritische steht zuerst sicher.

  Qualität:   Staging vor Produktion · CI/CD über GitLab · Tests auf dem kritischen Pfad
  Steuerung:  klare Schnitt-Verantwortung — App besitzt Daten/Logik, n8n besitzt alle
              Verbindungen zu Fremdsystemen, KlickTipp behält die Kampagnen-Hoheit
```

---

## 3 · Phasenplan & Timeline

```
 Woche       1    2    3    4    5    6    7    8    9   10   11   12   13
             ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
 P0 Infra    ████
 P1 Core          ████████
 P2 Attrib.                ███████████
 P3 Seiten                            ███████
 P4 Admin                                     ███████
 P5 Launch                                            ███████
 Meilenstein  ◆M0      ◆M1        ◆M2      ◆M3     ◆M4      ◆M5
```

| Phase | Inhalt | Aufwand |
|-------|--------|:------:|
| **0 · Infrastruktur** | Server, PostgreSQL, Reverse-Proxy/SSL, DNS, GitLab+CI, n8n/Secrets/Backup, Testzugänge | ~22 h |
| **1 · Datenmodell + Core** | Schema/Migrations, Token-/Short-Link + Redirect, Tracking, NPS-API, Promotor+Consent, QR, Branding-Basis | ~37 h |
| **2 · Attribution + Reward** | Lead-Capture, Order-Ingest, bidirektionales Matching, Mehrfach-Match-Guard, Reward-State-Machine, Gutschein-Mail | ~50 h |
| **3 · Öffentliche Seiten** | C1/D1 (mobil), E1 (Neukunde), C2/D2 (Desktop), Feedback, Fragebogen | ~33 h |
| **4 · Admin-Panel** | Auth+Audit, Promotoren CRUD/CSV, Referral-Status, Lookup, manuelle Zuordnung, Reward-Approval, Dashboard | ~27 h |
| **5 · Integration & Launch** | n8n-Flows final, E2E-Tests, Mobile/Cross-Browser, Go-Live + Monitoring | ~22 h |
| | **Gesamt** | **~191 h** |

**Dauer: ~10–13 Wochen** (bei 15–20 h/Woche gesamt; bei höherer Auslastung entsprechend früher).

---

## 4 · Milestones (mit Abnahmekriterium)

| MS | Ergebnis | „Fertig"-Kriterium | Ziel |
|:--:|----------|--------------------|:----:|
| **M0** | Infrastruktur steht | Server/DB/CI/Staging laufen, DNS aktiv, Testzugänge da, **A1–A4 entschieden** | Wo. 1–2 |
| **M1** | Fundament & Tracking | Ein Link wird erzeugt, geteilt und ein Klick sauber & doppel-sicher getrackt; NPS-Weiche live | Wo. 3–4 |
| **M2** | **Attribution & Reward (Herzstück)** | Ein geworbener Kauf wird zugeordnet und löst **genau eine** 25-€-Auszahlung aus (Doppelzahlungs-Schutz, Approval) | Wo. 6–7 |
| **M3** | Öffentliche Seiten live | Alle Kunden-Flows mobil **und** Desktop bedienbar (Empfehlen, Teilen, Neukunde, Feedback) | Wo. 8–9 |
| **M4** | Admin-Panel | Team kann Promotoren, Empfehlungen, Status, Auszahlungen & Lookup verwalten | Wo. 10–11 |
| **M5** | Go-Live | System produktiv auf `empfehlung.wasserzisterne.de`, E2E grün, Monitoring aktiv | Wo. 12–13 |

> M2 ist der fachlich härteste und geldrelevante Meilenstein — wird vor Abnahme doppelt (adversarial) gegen die dokumentierten Fälle geprüft.

---

## 5 · KPIs / Erfolgsmessung

**Produkt-/Business-KPIs** (das Dashboard aus Phase 4 liefert sie automatisch):

| Messgröße | Was sie zeigt | Zielwert |
|-----------|---------------|:--------:|
| NPS-Promotor-Quote (Anteil 9–10) | Wie viele Kunden überhaupt empfehlen wollen | *KI-Vorschlag* ≥ 40 % |
| Promotor-Aktivierung (Mail → Link erstellt) | Greift die Kampagne? | *KI-Vorschlag* ≥ 15 % |
| Ø Empfänger pro Promotor | **Multiplikator-Effekt** (Kundenfokus) | *KI-Vorschlag* ≥ 3 |
| Klickrate auf geteilte Links | Trägt der WhatsApp-Kanal? | *KI-Vorschlag* ≥ 25 % |
| Lead-Conversion auf E1 (Klick → Kontakt) | Funktioniert die Neukunden-Seite? | *KI-Vorschlag* ≥ 20 % |
| Empfehlungs-Conversion (Lead → Kauf) | Wirtschaftlicher Kern | *KI-Vorschlag* ≥ 5 % |
| Attribution-Match-Rate (auto vs. manuell) | Qualität der Zuordnung | *KI-Vorschlag* ≥ 70 % auto |
| Ausgezahlte Gutscheine / CAC | Kosten je geworbenem Kunden | im Kickoff festlegen |

> Die **Zielwerte sind KI-Vorschläge** zur Diskussion — sie stehen nicht im Plan und sind im Kickoff gemeinsam zu kalibrieren (Baseline TELLSCALE als Vergleich). Die *Messgrößen* selbst sind durch das System abgedeckt.

**Projekt-/Liefer-KPIs:** Phasen on-time · Testabdeckung Attribution/Reward ≥ 80 % · **0** Doppelzahlungen (Guard) · **0** offene kritische Bugs vor Go-Live.

---

## 6 · Top-Risiken & Umgang

| Risiko | Umgang |
|--------|--------|
| Strukturelle Attribution-Verlustrate (Gastkauf, andere E-Mail, wa.me-Sprung) | Lead-Capture auf E1 maximiert harte Treffer; manuelle Zuordnung als Fallback; Restquote ist akzeptiert & transparent |
| Reward-Doppelzahlung (Mehrfach-Empfehlung) | Doppelte Idempotenz + Tie-Break + Approval-Gate (Geld nie 2×) |
| WooCommerce-Webhook liefert zu wenig Match-Felder | In Phase 0 testen (A2); manueller Fallback by-design |
| Streamendous-Workflow unklar | A4 vor Phase 2 klären; sauberer Fehler-/Neuversuch-Pfad modelliert |
| WhatsApp misst „versandt/an wen" technisch nicht | Bewusste Grenze, ehrlich kommuniziert; gemessen werden Teilen-Klick + Link-Öffnung |

---

## 7 · ⭐ Kickoff-Entscheidungen (Gate für den Start)

Vier Punkte müssen entschieden sein, sonst bauen wir an einer Stelle ins Blaue. Für jeden gibt es einen **baubaren Default** — bestätigen oder anders entscheiden.

| # | Entscheidung | Default (baubar) | benötigt vor | ☐ |
|:--:|--------------|------------------|:------------:|:--:|
| **A1** | Promotor-Identität / Datenhoheit | Eigene DB, gespiegelt über KlickTipp-Abonnenten-ID | Phase 1 | ☐ |
| **A2** | Kauf-Trigger + Match-Felder | Woo-Webhook; Match über E-Mail+Telefon (hart), Name (unscharf) | Phase 2 | ☐ |
| **A3** | Gutschein-Bedingung | Fester Mindestbestellwert **oder** definierte Produktliste | Phase 2 **&** 3 | ☐ |
| **A4** | Gutschein-Ausführung (Streamendous) | Backend prüft & triggert, Streamendous führt aus, Code per Mail | Phase 2 | ☐ |

**Zu bestätigende Annahmen:** WordPress nur verlinken (kein Plugin) · Match-Zeitfenster 90 Tage · Gutschein-Mail über KlickTipp · GitLab auf bestehendem Hetzner-Server.

---

## 8 · Rollen & Mitwirkung

| Wer | Verantwortung |
|-----|---------------|
| **Entwicklungsteam (2)** | Bau, Tests, Deployment, Doku — gemäß Phasenplan |
| **n8n / KlickTipp** | Integrations-Orchestrierung bzw. Kampagnen- & Mailversand (Bestand) |
| **Auftraggeber (Hein)** | A1–A4 entscheiden · Zugänge bereitstellen · Branding & Video liefern · Abnahme je Milestone |

**Für den Start benötigte Zugänge:** KlickTipp-API-Key · WooCommerce-Webhook · Streamendous-API/Doku · Branding (Logo SVG + Hex-Farben) · Vimeo-URL · etermin-Prefill-Doku · DNS-Freigabe.

---

## 9 · Nächste Schritte

1. **Heute:** A1–A4 entscheiden, Annahmen bestätigen, Zugänge anstoßen.
2. **Diese Woche:** Phase 0 starten (DNS, Server, DB, CI/Staging) → **M0**.
3. **Danach:** Fundament & Attribution bauen — das Herzstück zuerst.

> Mit den vier entschiedenen Punkten ist das Projekt **kickoffbereit** und startet ohne weitere Rückfragen.
