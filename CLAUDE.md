# Gezeiten – Hinweise für Claude

Kleine Web-App (eine Datei `index.html`, kein Build) mit Tiden-, Wind- und Regenvorhersage für
**Wittdün Hafen, Hooge Anleger (Standard) und Dagebüll**. Ausgelegt für das Smartphone, auch als
App vom Home-Bildschirm. Sprache der Seite und der Kommunikation: **Deutsch**.

## Arbeitsweise

- Änderungen erst als **Vorschaubild** zeigen, wenn es um Aussehen geht; der Nutzer wählt dann aus.
- Commit und Push nach **`main`** nur, wenn der Nutzer es sagt (meist „commit und push“ / „nach main pushen“).
- Vor dem Push testen: Seite mit Playwright in Handy-Breite (390 px, `timezoneId: 'Europe/Berlin'`) laden,
  live gegen die APIs und mit nachgestellten Daten (z. B. Sturmflut-Werte, fehlende Werte).
  `celebrations.csv` wird per `fetch` geladen – dafür über einen lokalen Webserver testen, nicht `file://`.
- Veröffentlichung: GitHub Pages aus `main`. Live unter **https://dorfcode.com/gezeiten/**
  (das Repository heißt `DorfCode/gezeiten`, klein).

## Grundregel zu den Daten

**Nur offizielle Daten abbilden, nichts selbst berechnen oder erfinden.** Fehlt ein Wert im JSON,
steht „–“. Ausdrücklich erlaubte Ausnahmen:
- Umrechnung Wind km/h → **Beaufort** (WMO-Skala in m/s, siehe `BFT_MAX_MS`).
- **Regen**: Mittelwert der zwei Stundenwerte je Spalte (mm/h und %).
- **Nipp/Spring** aus der Mondphase berechnet (steht nicht im BSH-JSON; Nutzer will es behalten).

## Datenquellen

### Wasserstand: BSH
`https://gdi.bsh.de/ldproxy/rest/services/WaterLevelForecast/collections/waterlevelforecastdata/items/<id>?lang=en&f=json`
mit `<id>` = `wittduen_hafen`, `hooge_anleger`, `dagebuell`. Doku der Felder:
https://www2.bsh.de/aktdat/wvd/api/parameter_documentation.pdf

- Zeitstempel haben ein Leerzeichen (`2026-11-02 12:56:00+01:00`) → für Safari/iPhone in `T` umwandeln (`parseTs`).
- `mean_high_water` / `mean_low_water` in cm über Pegelnull (PNP). Fehlt ein Wert, Zeile ausblenden.
- Abweichung ist **Text** und bezieht sich auf MHW/MNW: `forecast_deviation`, sonst MOS-Hauptsystem
  `mos_forecast_r0_deviation`, sonst Ersatzsysteme `r1`…`r5` (erster vorhandener Wert, `DEVIATION_KEYS`).
- Angezeigt wird nur `information`/`information_text` (Betriebshinweise). Der Wasserstandstext
  (`forecast_text`) wird bewusst **nicht** gezeigt.
- Zeitraum der Tabelle: ganzer heutiger Tag + 5 Tage (`DAYS_AHEAD`).

### Wind und Regen: DWD über Bright Sky
- Vorhersage (MOSMIX): `https://api.brightsky.dev/weather?wmo_station_id=<id>&date=…&last_date=…&tz=Europe/Berlin`
  - **Hooge: `A201`** (echte Station Hallig Hooge). **Wittdün und Dagebüll: `A112`** (Wrixum/Föhr).
  - Nicht über Koordinaten abfragen: Das liefert für Hooge den Vorhersagepunkt `P0151`, der im Vergleich
    (Nov. 2026) rund 9 km/h zu niedrig lag; `A201` traf die Messungen fast genau.
  - Nur Einträge mit `observation_type === 'forecast'` verwenden (für vergangene Stunden liefert
    Bright Sky sonst Messwerte anderer, teils entfernter Stationen).
- Messung „jetzt“: `https://api.brightsky.dev/current_weather?lat=…&lon=…` (10-Minuten-Werte, nur wenn
  höchstens 90 Minuten alt).
- Open-Meteo wurde verglichen und **nicht** übernommen (ähnlich zu niedrig wie `P0151`; Tageslimit pro IP).

## Aufbau der Seite (von oben)

1. **Ortsauswahl**: drei Knöpfe Wittdün · Hooge · Dagebüll; gewählter Ort dunkel mit dickem weißem Rahmen,
   leicht aufgehellt, **kein Fettdruck**. Ort steht im Link als `#wittduen_hafen` usw. (ohne = Hooge).
2. **Wind & Regen · DWD <Station> · 12 h**: 7 Spalten, ab der aktuellen vollen Stunde alle 2 h.
   Erste Spalte „jetzt“ = Messung, leicht hinterlegt. Zeilen: Pfeil (wohin der Wind weht; schmaler Schaft,
   kleine Spitze), Bft, Böen (Bft), Regen mm/h und %. Fehlende Werte als „–“; ohne messbaren Regen
   (unter 0,1 mm/h) auch keine Prozentangabe.
3. **Eine Zeile Wasser**: `Wasser · BSH <Stand> · Abruf <HH:MM>` (passt auf 320 px). Ortsname nur für
   Bildschirmleser (`.visually-hidden`) und im Seitentitel.
4. **Tidentabelle**: Zeit · Tide · Abw. [m].
   - Tide als Wellen-Symbol: **drei Wellen = Hochwasser, eine Welle = Niedrigwasser**, dahinter ggf. „Nipp“/„Spring“.
   - Hochwasser-Zeilen leicht blau hinterlegt; Datumszeile als blauer Balken, haftet beim Scrollen
     unter dem Tabellenkopf.
   - Markierung **nur für Hochwasser**, bei Bereichen zählt der höchste Wert:
     - ab **+0,75 m**: nur der Wert als gelbe Plakette (`.raised`), Zeile unverändert
     - ab **+1,5 m** gelb (Sturmflut), ab **+2,5 m** amber (schwere), ab **+3,5 m** rot (sehr schwere) –
       BSH-Einstufung Nordsee über MHW (Tab. 5 der BSH-Doku), ganze Zeile
   - Vergangene Tiden von heute blass.
5. **Footer**: MHW/MNW, Legende, Quellen, Link zur offiziellen BSH-Seite des Pegels (`bsh_url_waterlevel`).

Farben: Blau-Palette (`--blue` #005a8c, `--blue-dark` #003e61). Keine bunten Textfarben in der Tabelle.

## Weiteres

- **Aktualisierung**: alle 10 Minuten und beim Zurückkehren zur Seite, ohne Flackern (`loadForecast(…, true)`);
  alle Abrufe mit `cache: 'no-store'`.
- **celebrations.csv**: ein Datum pro Zeile (`TT.MM.` oder `TT.MM.JJJJ`, Rest nach `;` ignoriert, `#` = Kommentar).
  An diesen Tagen bei jedem Öffnen ca. 3,5 s Konfetti, **ohne Text**, auch bei „Bewegung reduzieren“.
  Datei ist öffentlich → keine Namen. Abruf mit `?t=<Zeitstempel>` gegen den Cloudflare-Zwischenspeicher.
- **Home-Bildschirm-App**: Icons in `icons/` (Pegellatte), `favicon.ico`, `site.webmanifest`;
  `viewport-fit=cover` und Safe-Area-Abstände, Statusleiste `black-translucent`.
- **Domain**: `dorfcode.com` über Cloudflare. Redirect-Regeln dort: `gezeiten.dorfcode.com` → `/gezeiten/`,
  alte Schreibweise `/Gezeiten…` → `/gezeiten…`.
