# DUB Unternehmenswertrechner

Single-File-Webapp zur Unternehmensbewertung + Nachfolge-Lead-Generierung.
**Live:** https://wertrechner.dub.de

## Überblick / Architektur
- **Eine Datei:** `index.html` enthält alles — Markup, Styles (inline) und die komplette App in einem `<script type="text/babel">`.
- **Stack:** React 18 + ReactDOM + **Babel Standalone** (alle via CDN/cdnjs). JSX wird **im Browser** kompiliert — **kein Build-Step**, kein Bundler, keine `node_modules`.
- **Ablauf (State-Machine über `_step`/`setStep`):** Hero → Schnellcheck (`r1`–`r3`) → E-Mail-Gate (`email`) → Ergebnis (`result`) → optional vertiefte Analyse (`d1`–`d3` → `resultDeep`) → Zukunfts-Check (`check` → `checkResult`) → Bridge (Registrierung). Darunter dauerhaft FAQ + Footer.
- **Branding:** Post-Merger DUB — Navy `#0B2340` (dominant), Orange `#F07E23` (Akzent/CTA), Schrift Inter (Fallback für TWK Lausanne), Ansprache durchgängig „du". Das Logo ist als base64-PNG direkt in `index.html` eingebettet.
- **Stil-Warnung:** Der Code ist bewusst sehr dicht (oft eine Komponente/Funktion pro Zeile). Es gibt **kein Linting und keine Build-Validierung** — ein JSX-Syntaxfehler führt zu einer **komplett weißen Seite**, die Ursache steht nur in der **Browser-Konsole**. Nach jeder Änderung lokal laden und die Konsole prüfen. (Risiko: Die CDN-Abhängigkeiten sind nicht gepinnt/SRI-geschützt — fällt cdnjs aus, ist die Seite tot.)

## Branches
- **`main` = Live-Stand** (Source of Truth für Production). Vor jeder Arbeit: `git checkout main`. Verifikation: `main` enthält den Commit „Redesign als Live-Stand übernommen".
- `redesign` = inhaltlich identische Kopie aus der Redesign-Phase, **obsolet** — sollte gelöscht/archiviert werden, damit kein Doppelstand bleibt. (Der zuletzt ausgecheckte Arbeits-Branch hieß `redesign`; nicht von dort deployen.)

## Lokale Entwicklung
Kein Build nötig:
- `index.html` direkt im Browser öffnen, **oder**
- statischen Server starten, z. B. `npx serve` → `http://localhost:3000`.
- Nach Änderungen **immer die Browser-Konsole prüfen** (weiße Seite = JSX-Fehler).

Das **Usercentrics-Cookie-Banner** erscheint nur auf `wertrechner.dub.de` (Domain-Allowlist), nicht auf localhost — normal.

## ⚠️ Gefahrlos testen (keine echten Leads erzeugen!)
**Es gibt keinen Test-/Staging-Modus und keine Hostname-Weiche** — jeder abgesendete Submit geht an das **echte Produktiv-Portal 4804905**, auch von localhost. So testest du, ohne echte Kontakte anzulegen:
- **fetch blocken:** DevTools → Network → Rechtsklick auf den `api.hsforms.com`-Request → „Block request URL" (oder Network auf „Offline" stellen), dann durchklicken.
- **oder fetch stubben** (Konsole, vor dem Test):
  ```js
  const _f = window.fetch;
  window.fetch = (u,o) => (typeof u==='string' && u.includes('hsforms'))
    ? Promise.resolve({ok:true, status:200, json:()=>({})}) : _f(u,o);
  ```
- **oder** mit klar erkennbarer Test-Adresse testen (z. B. `…@example.com`) und den Kontakt danach in HubSpot **löschen**.

Dauerhafte Lösung (empfohlen): eine Hostname-Weiche einbauen (`const TEST = location.hostname==='localhost'`) und die `fetch`-Calls bei `TEST` überspringen.

## Deployment (Vercel)
Vercel-Projekt: **`dub-wertrechner`**. Aktuell manuell per CLI.
**Voraussetzungen:** Vercel-Zugang zum Projekt (Team-Einladung), `vercel login`, ggf. `vercel link` auf das Projekt.
- Production: `vercel --prod`
- Vorschau (eigene URL, Live unberührt): `vercel`
- Rollback: `vercel rollback` oder im Dashboard ein früheres Deployment „promoten" (Sekunden, kein Downtime). `wertrechner.dub.de` ist ein **Alias** auf das jeweils aktuelle Prod-Deployment — Rollback = anderes Deployment zum Alias machen.

**`.vercelignore` ist eine TYP-basierte Blockliste, KEINE Allowlist.** Sie blockt `*.docx`, `*.png`, `*.html` (außer `index.html`) und `.claude`. Konsequenzen:
- Eine **neue `.html`-Datei** wird **stumm vom Deploy ausgeschlossen** → per `!name.html` freigeben.
- **Andere Dateitypen** (`.js`, `.css`, `.svg`, `.woff` …) gehen **ungefiltert live**.
Aktuell gewollt (Single-File-App, nur `index.html`). Bei neuen Assets `.vercelignore` bewusst anpassen.

**Empfehlung:** Vercel-Git-Integration einrichten → Auto-Deploy bei Push auf `main` + Preview-Deploys pro PR. (Derzeit keine Git-Integration; Deploys laufen rein über die CLI von lokal.)

## ⚠️ Beim Bearbeiten unverändert lassen (Fachlogik & Integrationen)
- `doCalc()` + `calcQA()`/`calcDA()` (Bewertungslogik), `BRANCHEN` (Marktdaten-Multiples je Branche/Größe), `questions`/`getRecs`/`getCR` (Zukunfts-Check-Scoring & Empfehlungen).
- `_UTM` (UTM-Parsing aus der URL → `wertrechner_quelle`).
- Die `window._dub*`-Flags (Doppel-Submit-/Event-Schutz — siehe Einschränkung unten).

## HubSpot-Anbindung (wichtig)
Leads über die **HubSpot Forms API v3** (`integration/submit`):
- **Portal:** `4804905` · **Form-GUID:** `e157bf3d-d4c9-44ee-80c8-27315fb500ff`

**Drei Submits entlang der Journey:**

| # | Auslöser | Doppel-Schutz | `wertrechner_modus` | Schwerpunkt-Felder |
|---|---|---|---|---|
| A | „Weiter zur Bewertung" im E-Mail-Gate | **nur indirekt** (s. u.) | `schnell` | Identität, `inhaber`, Schnell-Ergebnis, **SOI-Opt-in** |
| B | Erreichen von `resultDeep` (vertiefte Analyse) | `window._dubDeepFired` (gelesen) | `vertieft` | verfeinertes Ergebnis + Konfidenz |
| C | Abschluss Zukunfts-Check (Frage 8) | `window._dubZukunftFired` (gelesen) | – | `wertrechner_zukunft_score`, `wertrechner_zukunft_level` |

**Achtung Submit A:** Der Gate-Submit ist **nicht** per Flag gegen Doppelung geschützt — `window._dubHsResult` wird zwar gesetzt, aber **nirgends gelesen** (Altlast aus dem Umbau). Schutz nur dadurch, dass der Button erst bei gültigem Formular **und** gesetztem `consent` aktiv ist und der Schritt nach dem Submit verlassen wird. Ein schneller Doppelklick könnte zwei POSTs auslösen. **Optionale Härtung:** `if(window._dubHsResult)return;` vor den `fetch` in A setzen.

**Feldnamen:** `email`, `salutation`, `firstname`, `lastname`, `quelle_kontakt`,
`wertrechner_branche`, `wertrechner_groesse`, `wertrechner_umsatz`, `wertrechner_ebit`,
`wertrechner_ergebnis_min`, `wertrechner_ergebnis_max`, `wertrechner_konfidenz`,
`wertrechner_modus`, `wertrechner_inhaber`, `wertrechner_quelle`,
`wertrechner_zukunft_score`, `wertrechner_zukunft_level`,
`wertrechner_opt_in`, `wertrechner_opt_in_quelle`, `wertrechner_opt_in_datum_z`.

### ‼️ HubSpot-Gotcha (unbedingt wissen)
Die Forms API speichert **nur Felder, die im HubSpot-Formular `e157bf3d-…` als Formularfeld hinterlegt sind.** Eine Property, die nur als Kontakt-Property existiert, aber **nicht im Formular** angelegt ist, wird **stillschweigend verworfen** — der Submit liefert trotzdem `200`. **→ `200` ≠ gespeichert.**

Neues `wertrechner_*`-Feld hinzufügen — Checkliste:
1. Property in HubSpot anlegen (Kontakt-Properties).
2. Feld im **Formular `e157bf3d-…`** als Formularfeld ergänzen.
3. Im Code zum passenden Submit hinzufügen.
4. Test-Submit (s. „Gefahrlos testen") und **am Test-Kontakt prüfen**, dass der Wert ankam.

### Lead-Eingang prüfen
HubSpot → Portal `4804905` → Kontakte. Nach `quelle_kontakt = "DUB Wertrechner"` (oder der Test-E-Mail) filtern und **am Einzelkontakt** prüfen, ob die erwarteten `wertrechner_*`-Felder gesetzt sind (wegen „200 ≠ gespeichert"). Portal-/Property-/Formular-Zugang vergibt das Marketing-/HubSpot-Admin-Team.

### SOI-Einwilligung (Single Opt-in)
- **Nur beim Gate-Submit (A)**: `wertrechner_opt_in="true"` + `wertrechner_opt_in_quelle="DUB Wertrechner"`. `consent` ist **Pflicht** (Button `disabled={!validForm||!consent}`) — d. h. wenn A überhaupt feuert, ist `opt_in` immer `true`. Der defensive `consent ? [...] : []`-Zweig im Code erreicht den leeren Fall produktiv nie.
- **`wertrechner_opt_in_datum_z`** ist eine **Datum-&-Uhrzeit-Property** → in HubSpot **nicht als Formularfeld** hinzufügbar. Der Code sendet sie zwar als rohen Epoch-Millisekunden-String (`String(Date.now())`) mit, das Formular ignoriert sie aber. Der echte Zeitstempel kommt von einem **HubSpot-Workflow**: Trigger `wertrechner_opt_in = true` (Wiederaufnahme **aus** → nur Erst-Einwilligung), Aktion „Eigenschaft setzen → auf das Ausführungsdatum".

## Tracking & Consent
- **GTM:** Container `GTM-W385QDX` steuert GA4/etracker (keine direkten Snippets im Code). Lead-Event: `dataLayer.push({event:'form_submit_lead'})`; zusätzlich Meta-Pixel-Events via `fbq`, falls geladen.
- **Usercentrics CMP:** Settings-ID `i-OW_OGmKJjg4h` (nur auf der Live-Domain aktiv).

## Bekannte Punkte / vor Änderungen klären
- **FAQ-Widerspruch (Datenschutz/Marketing prüfen):** Die FAQ sagt „Deine Finanzdaten werden nicht auf unseren Servern gespeichert." Der Rechner sendet jedoch `wertrechner_umsatz/ebit/ergebnis_min/max` an HubSpot — diese Werte werden also gespeichert. Aussage und Verhalten passen nicht zusammen.
- **Doppel-Submit-Schutz für A** fehlt (siehe oben) — ggf. härten.
- **Keine Build-/Lint-Validierung** und **ungepinnte CDN-Abhängigkeiten** — siehe Architektur.

## Was bewusst NICHT im Repo liegt
Brand Book, Plattform-Mockup und Logo-Originaldateien (vertraulich) sind **nicht** eingecheckt (auch nicht in der Git-Historie) und werden über `.vercelignore` nie deployed.
