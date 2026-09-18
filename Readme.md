# CNC-Code Simulator — technische Spezifikation

Diese Spezifikation beschreibt ausschließlich den **aktuellen Stand** des Werkzeugs — keine Entwicklungshistorie, keine Änderungsprotokolle. Sie ist so detailliert gehalten, dass sich das Tool anhand dieses Dokuments exakt nachbauen lässt.

## 1. Zweck & Überblick

Der CNC-Code Simulator — gebrandet als „CNC SimX“ — ist ein rein clientseitiges, browserbasiertes Werkzeug zum Laden, Analysieren und 3D-Simulieren von CNC-/ISO-Programmcode aus der Holzbearbeitung (WinPPZ/SCM, HOMAG, FORMAT4/Felder, Moroff, Biesse/CIX und weitere Dialekte, siehe Abschnitt 4/5). Es läuft vollständig im Browser ohne Server-Backend, ohne externe JavaScript-Bibliotheken und ohne Installation — ausgeliefert als eine einzige, in sich geschlossene HTML-Datei (`CNC_Simulation.html`).

Kernfunktionen:
- Laden einer CNC-Programmdatei (Dateiauswahl, Drag & Drop oder direktes Einfügen als Text) mit automatischer Zeichenkodierungserkennung.
- Automatische Erkennung des Maschinen-/Postprozessor-Dialekts und Zerlegung des Programms in einzelne Arbeitsgänge, Zwischenstopps und Programmende-Abschnitte anhand des Kommentarstils.
- Eine per Dropdownmenü wählbare **Maschinentyp-Konfiguration** (Such-/Ersetz-Regeln plus Achsberechnung, je Maschinentyp editierbar), die erst nach expliziter Bestätigung („Ok“) auf den geladenen Programmtext angewendet wird, bevor er angezeigt/simuliert wird (siehe Abschnitt 5) — Kommentare (`;`, `KM="…"`, `{…}`) bleiben dabei von Such-/Ersetz- und Achsberechnungsregeln ausgenommen (siehe 5.11).
- Eine editierbare **Arbeitsgang-Namensliste** für Postprozessoren ohne Trennzeilen-Konvention.
- Eine synchronisierte Zweispalten-Programmansicht (durchgehende Code-Liste + separate Arbeitsgänge-Spalte) mit automatischer Spaltenbreite und Ein-/Ausklapp-Stufen.
- Ein interaktiver 3D-Bahn-Viewer (Zoom zum Mauszeiger, Rotation/Pan, ein anklickbares Rhombenkuboktaeder-Ansichts-Gizmo oben rechts im 3D-Bereich mit „Home“-Knopf daneben, Wiedergabe mit vier Geschwindigkeitsstufen und Leertaste als Play/Pause-Kurzbefehl, Längen- und Winkel-Messwerkzeug mit ΔX/ΔY/ΔZ-Anzeige inklusive eingezeichneter Delta-Linien, beide in einem frei verschiebbaren Info-Fenster), der Kreisbögen (G02/G03, R-Format) als echte tessellierte Bögen statt als gerade Linien darstellt (siehe Abschnitt 9.1).
- Editoren zum direkten Bearbeiten von Programmtext, Maschinentyp-Konfiguration und Arbeitsgang-Namensliste sowie „Speichern als…“ zum Herunterladen des (ggf. bearbeiteten) Programms.
- Export/Import aller Simulationseinstellungen (Arbeitsgang-Namensliste + Maschinentyp-Konfiguration + Farbprofil + Schriftfarbe + Spalteneinstellungen + Menü-Hervorhebungsfarbe, nicht das Programm selbst) als Datei über „Einstellungen exportieren“/„Einstellungen importieren“ (siehe Abschnitt 5.9), zusätzlich zur automatischen localStorage-Persistenz.
- Frei einstellbare Standard-Skalierung (25/50/75/100 %) für die Breite der Programmcode-Spalte sowie ein/aus für die Arbeitsgänge-Spalte, eine frei wählbare Hervorhebungsfarbe für die aktiven Menü-Knöpfe „Datei“/„Bearbeiten“/„Einstellungen“ sowie fünf einzeln festlegbare Schriftfarben für Zeilennummerierung, Programmcode, Arbeitsgänge, Geschwindigkeit und Sprung-in-Zeile (gebündelt unter „Einstellungen“ → „Erscheinungsbild“, siehe Abschnitt 10 und 12.3) sowie ein „Standardwerte wiederherstellen“-Knopf, der alle vier genannten Anzeige-Einstellungen in einem Schritt zurücksetzt — alle dauerhaft persistiert.
- Ein separater, vom CNC-Programm unabhängiger **DXF-Import** („Datei“ → „DXF Importieren“ oder Drag & Drop einer `.dxf`-Datei) für eine verschieb-, spiegel-, dreh- und mit Bauteilstärke versehbare Referenz-Unterlage, die im 3D-Bereich unter der gefrästen Kontur dargestellt wird (siehe Abschnitt 9.8), inklusive echter 3D-Flächenmodelle (3DFACE) und einer Layer-Verwaltung mit einzeln aus-/einblendbaren Layern.

## 2. Architektur

Das Tool besteht aus fünf Quelldateien, die zu einer einzigen HTML-Datei zusammengebaut werden:

| Datei | Inhalt |
|---|---|
| `template_top.html` | Vollständiges HTML-Grundgerüst (`<html><head>…</head><body>…`) inkl. des kompletten `<style>`-Blocks; endet nach dem letzten UI-Panel, **ohne** schließende `</body></html>`-Tags. |
| `parser.js` | Maschinenerkennung, Arbeitsgang-/Zwischenstopp-/Programmende-Erkennung, DXF-Parser. Reine Logik ohne DOM-Zugriff (UMD-Modul, im Browser als `window.CNCParser`, in Node per `module.exports` einbindbar). |
| `swap.js` | Parsen und Anwenden der Maschinentyp-Austauschregeln. Ebenfalls reine Logik ohne DOM-Zugriff (`window.CNCSwap`). |
| `encoding.js` | Robuste Zeichenkodierungserkennung beim Datei-Einlesen (`window.CNCEncoding`). |
| `app.js` | Gesamte UI-Logik, DOM-Verdrahtung, 3D-Rendering, App-Zustand. Läuft als selbstausführende Funktion `(function(){ … })();`, greift auf `window.CNCParser`/`window.CNCSwap`/`window.CNCEncoding` zu. |

**Build-Vorgang:** Die vier Skriptdateien werden jeweils in ein eigenes `<script>…</script>`-Element eingebettet (Reihenfolge: `parser.js`, `swap.js`, `encoding.js`, `app.js`), mit `\n` verbunden, und per **direkter String-Konkatenation** an `template_top.html` angehängt:

```
template_top.html + "\n" + <script>parser.js</script>\n<script>swap.js</script>\n<script>encoding.js</script>\n<script>app.js</script> + "\n</body>\n</html>\n"
```

Kein Ersetzen eines Platzhalters (z. B. `</body>`) — die schließenden Tags werden beim Zusammenbau selbst angehängt, da `template_top.html` sie bewusst nicht enthält. Das Ergebnis ist eine einzige, vollständig eigenständige HTML-Datei ohne externe Skript-Referenzen (einzige externe Ressource: der Google-Fonts-Import für „Open Sans“/„IBM Plex Mono“ im `<head>`).

**Tests:** `test.js` (Node, kein Browser/DOM nötig) lädt `parser.js` und `swap.js` direkt per `require()` und prüft deren Logik anhand synthetischer und teils realer Beispieltexte per einfachem Assertion-Helfer. Ausführung: `node test.js`. Siehe Abschnitt 14.

**Layout:** Die App ist als Einzelbildschirm-Anwendung angelegt (`.app { height: 100vh; min-height: 640px; display: flex; flex-direction: column; }`) — Menüband, Symbolleiste, Transportleiste, Seitenleiste (Programmliste + Arbeitsgänge-Spalte) und 3D-Bereich füllen zusammen den Viewport. Jeder Bereich mit potenziell überlaufendem Inhalt (`.tree`, `.ops-list`, bei schmalen Fenstern zusätzlich die gesamte `.sidebar`, siehe die `@media (max-width: 820px)`-Regel) scrollt zusätzlich **intern** über sein eigenes `overflow-y: auto`; `html`/`body` selbst tragen kein eigenes `overflow: hidden` (nur `margin: 0`), sodass die Gesamtseite bei sehr kleinen Fenstern bzw. durch Symbolleisten weiter verkleinerten Browserfenstern weiterhin ganz normal scrollbar bleibt.


## 3. Datei-Laden & Vorverarbeitung

**Zeichenkodierung** (`encoding.js`, `decodeBuffer(buffer)`): Jede Datei wird unabhängig vom Ladeweg immer als `ArrayBuffer` eingelesen (`FileReader.readAsArrayBuffer`), nie über `readAsText` (dessen UTF-8-Standardannahme bei den in der Praxis häufigen Windows-1252-kodierten Postprozessor-Exporten zu stillschweigend falscher Dekodierung führen würde). Dekodierstrategie: zuerst **strikt** als UTF-8 (`new TextDecoder('utf-8', {fatal:true})`) — schlägt bei einer Windows-1252-Datei mit hohen Bytes zuverlässig fehl statt sie lautlos falsch zu lesen; erst dann Fallback auf `new TextDecoder('windows-1252')`.

**N-Zeilennummern:** `stripN(line)` entfernt eine führende Zeilennummerierung (`/^\s*N\d+\s*/`) ausschließlich für die interne Mustererkennung (Maschinen-/Arbeitsgang-Erkennung) — die Original-Zeilen mit N-Nummer bleiben für Anzeige und Bewegungs-Extraktion unverändert erhalten.

**Ladewege:**
- „Programm laden“ (verstecktes `<input type="file">`, im „Datei“-Menü unter „CNC-Programm“) oder Drag & Drop auf das Fenster (`window.addEventListener('drop', …)`, mit `dragover`-`preventDefault()`), jeweils über `readFileAsText()`/`decodeBuffer()`. Die Dropzone unterscheidet dabei anhand der Dateiendung (case-insensitiv): Eine `.dxf`-Datei geht an den separaten DXF-Import (siehe 9.8), jede andere Endung wird als CNC-Programm behandelt.
- „CNC-Code einfügen“: Textfeld-Panel mit direktem Einfügen von Programmtext (kein Datei-Encoding-Schritt nötig, da bereits als JS-String vorliegend).
- **Host-Hook `window.__hostLoadProgramFromHost(base64Content, filename)`** — vierter Ladeweg für eine optionale native Windows-Hülle, siehe 3.1.
- „CNC-Programm löschen“ setzt Anzeige, Baum, Arbeitsgänge-Spalte, 3D-Szene und Transportleiste auf den Ausgangszustand zurück, **ohne** die Maschinentyp-Konfiguration zu entfernen (die Dropdown-Auswahl selbst fällt dabei auf „ISO“ zurück, siehe Abschnitt 5.1). Ein eventuell geladenes DXF (Abschnitt 9.8) bleibt davon vollständig unberührt.

### 3.1 Host-Hook für eine optionale native Windows-Hülle

Für eine optionale, native C#/WebView2-Hülle um die Simulation herum (Details siehe die Projekt-Dokumente „Konzept_EXE_C-Sharp.md“, „Bauanleitung_EXE_C-Sharp_WebView2.md“ und „Bauanleitung_EXE_fuer_Einsteiger.md“) existiert ein globaler, rein additiver Einstiegspunkt, über den ein per Batch-Datei/Kommandozeilenargument bestimmtes CNC-Programm beim Start automatisch geladen werden kann:

```js
window.__hostLoadProgramFromHost = function (base64Content, filename) {
  var binary = atob(base64Content);
  var bytes = new Uint8Array(binary.length);
  for (var i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
  var text = window.CNCEncoding.decodeBuffer(bytes.buffer);
  loadText(text, filename);
};
```

Ein C#-Host (WebView2) kann diese global erreichbare Funktion per `CoreWebView2.ExecuteScriptAsync(...)` aufrufen. Bewusst werden **Rohbytes** (Base64-kodiert) statt eines vom Host bereits „geratenen“ Texts übergeben, damit dieselbe, bereits bestehende UTF-8/Windows-1252-Erkennung (`window.CNCEncoding.decodeBuffer()`, siehe oben) exakt so greift wie beim regulären Laden im Browser — keine zweite, abweichende Kodierungs-Implementierung auf der C#-Seite nötig. Der übergebene Dateiname wird (anders als bei „CNC-Code einfügen“, das fest „Eingefügter Code“ einträgt) unverändert als `rawProgramFilename` übernommen, sodass `#filenameLabel` den tatsächlichen, vom Host übergebenen Dateinamen anzeigt.

## 4. Parser (`parser.js`)

### 4.1 Maschinenerkennung

`detectMachine(lines)` durchsucht die ersten `HEADER_SCAN_LINES = 60` Zeilen:

| Reihenfolge | Bedingung | Ergebnis |
|---|---|---|
| 1 | `/\.CIX/i` trifft auf Zeile 1 (als Teilstring, beliebiger Dateiname davor/danach zulässig) | `CIX` |
| 2 | `/SCM/i` im Kopfbereich | `SCM` |
| 3 | `/HOMAG/i` im Kopfbereich | `HOMAG` |
| 4 | `/FORMAT/i` im Kopfbereich | `FORMAT4` |
| 5 | `/MOROFF/i` im Kopfbereich | `MOROFF` |
| 6 | `/\(\s*KRC\b/i` im Kopfbereich | `KRC` |
| 7 | Fallback: eine Kopfzeile trifft exakt `/^\{_+\}$/` | `MOROFF` |
| sonst | — | `UNKNOWN` |

Die Prüfung 1 (CIX) läuft **ausschließlich auf Zeile 1**, alle anderen auf den zusammengefügten ersten 60 Zeilen. Das Erkennungsergebnis wird nirgends direkt als Badge angezeigt (siehe Abschnitt 10, „Struktur-Anzeige“) — es dient ausschließlich der Voraberkennung des Maschinentyp-Dropdowns (5.4) sowie als `machine`/`machineLabel` im Parser-Ergebnis.

### 4.2 Arbeitsgang-Kommentarstile

Ein Arbeitsgang wird unabhängig vom per `detectMachine()` erkannten Maschinentyp gefunden — `findGenericOpStart()` durchsucht ab der aktuellen Position **alle** bekannten Stile und liefert den frühesten Treffer (Ausnahme: siehe CIX-Gating unten). So werden auch Dateien mit unbekanntem/fehlendem Kopftext in Arbeitsgänge zerlegt, solange irgendeine der folgenden Kommentarstrukturen vorkommt:

| Stil | Muster (3-zeilig, sofern nicht anders angegeben) | Titel-Extraktion | Ende-Erkennung |
|---|---|---|---|
| `SCM` | `;___` (≥3 `_`) / Titel (keine reine Trennzeile) / `;___` | Semikolon + optionaler führender Unterstrich-Lauf entfernt | `/EndXAxis\s+Ende/i` |
| `HOMAG` | optional `<101\Kommentar\` davor; `KM="[-_=*]+"` / `KM="Titel"` / `KM="[-_=*]+"` | Inhalt des mittleren `KM="…"` | `/ENDLOCAL/i` |
| `FORMAT4` | `FORMAT4_SEP_RE` / `#Titel` (kein `#___`) / `FORMAT4_SEP_RE`, mit `FORMAT4_SEP_RE = /^#\s*_{3,}$/` (Leerzeichen zwischen `#` und den Unterstrichen optional) | führendes `#` entfernt | `/^G60\b/` |
| `MOROFF` | `{___}` / `{Titel}` (keine reine Trennzeile) / `{___}` | Klammerinhalt | vor `park_waste\s*\(\s*laenge` **oder** `{Compass sub Motor OFF}`, je nachdem was zuerst kommt |
| `KRC_BLOCK` | `(___)` (Leerzeichen um die Unterstriche toleriert) / `(Titel)` / `(___)` | Klammerinhalt | kein eigenes Muster — reicht bis zum nächsten Arbeitsgang/Dateiende |
| `CIX` (Biesse) | drei aufeinanderfolgende 4-zeilige Makro-Blöcke `BEGIN MACRO`/`NAME=ISO`/`PARAM,NAME=ISO,VALUE="…"`/`END MACRO` (Leerzeilen dazwischen werden übersprungen), deren `VALUE`-Inhalte dem Schema `;___` / `;Titel` / `;___` folgen | Semikolon + führender Unterstrich-Lauf entfernt | kein eigenes Muster — reicht bis zum nächsten Arbeitsgang/Dateiende |
| `NAMED` | eine einzelne Zeile, deren Inhalt (nach Entfernen eines optionalen führenden `;`, `#` oder `//`) case-insensitiv exakt einem Eintrag der Arbeitsgang-Namensliste entspricht (siehe Abschnitt 6) | die Zeile selbst | kein eigenes Muster — reicht bis zum nächsten Arbeitsgang/Dateiende |

Der früher zusätzlich vorhandene **einzeilige** `KRC`-Stil (`(Titel)` ohne umgebende Trennzeilen) wird **dauerhaft nicht mehr durchsucht**: Bei echten KRC-/MAKA-Programmen kommen runde-Klammer-Kommentare auch ganz gewöhnlich und häufig ohne jeden Bezug zu einem Arbeitsgang vor (z. B. `(Eilgang)`, `(Werkzeug zurück)`) — jeder davon würde sonst fälschlich als eigener Arbeitsgang gewertet. `MACHINES.KRC` bleibt nur noch für die Maschinen-Label-Zuordnung erhalten (`detectMachine()` liefert weiterhin `'KRC'` anhand der Kopfzeile `(KRC …)`); echte Arbeitsgänge dieser Maschinen werden ausschließlich über den bewusst strengen 3-zeiligen `KRC_BLOCK`-Stil erkannt (Trennzeile **davor und danach** verlangt) — strukturell identisch für KRC **und** MAKA.

**CIX-Gating:** Der `CIX`-Stil wird — anders als alle übrigen — **nur** durchsucht, wenn der von `detectMachine()` erkannte Maschinentyp selbst `CIX` ist (Zeile 1 enthält `.CIX`). Bei Gleichstand (zwei Stile treffen an derselben Startzeile) gewinnt der zuerst in der internen Reihenfolge (`STYLE_IDS`, Definitionsreihenfolge oben) stehende Stil; `preferredStyle` (der erkannte Maschinentyp) wird dabei bevorzugt sortiert, `NAMED` steht bewusst an letzter Stelle.

Da der Klammer-Block-Stil (`KRC_BLOCK`) kein eigenes Ende-Muster kennt (reicht immer bis zum nächsten Arbeitsgang bzw. Dateiende), entsteht zwischen zwei aufeinanderfolgenden Arbeitsgängen nie eine „Lücke“ — ein `M00`/`M0` innerhalb eines Arbeitsgangs (z. B. zwischen zwei Klammer-Blöcken) wird deshalb Teil des vorangehenden Arbeitsgangs statt separat als „Zwischenstop KRC“ (siehe 4.3) markiert zu werden; dasselbe gilt für `CIX`/`NAMED`.

### 4.3 Zwischenstopp- und Programmende-Muster

Zwischen zwei Arbeitsgängen (bzw. vor dem ersten/nach dem letzten) wird die Lücke mit `classifyGap()` weiter aufgeschlüsselt: Es wird nach dem frühesten der folgenden Muster gesucht (zur erkannten Maschine passende Muster werden bevorzugt, alle übrigen dienen als Fallback); nicht zugeordnete Bereiche werden als feste Abschnitte („Kopfbereich“, „Fester Zwischenbereich“, „Fester Fußbereich“) übernommen.

| Maschine | Art | Start-Muster | Ende-Muster (Fenster) |
|---|---|---|---|
| SCM | Zwischenstopp | `/^IF\s+DX\s*>/i` | `/^FI\b/i` (20 Zeilen) |
| SCM | Programmende | `/^WEGFAHR(1\|2)\b/i` | `/\.FIN\b/i` (30 Zeilen) |
| HOMAG | Zwischenstopp | `/<117\s*\\NCStop\\/i` | `/KM="([^"]*)"/i` (10 Zeilen) |
| HOMAG | Programmende | `/KM="\s*Ende/i` | sofort (0 Zeilen) |
| FORMAT4 | Zwischenstopp | `/^#\s*MOTOR\s+OFF\s+SP\s*100/i` | `/(#\s*PAUSE\|M100P99)/i` (6 Zeilen) |
| FORMAT4 | Programmende | `/^#\s*ABFAHRWEG/i` | `/#\s*END\s+OF\s+PROGRAM/i` (6 Zeilen) |
| MOROFF | Zwischenstopp | `/park_waste\s*\(\s*laenge/i` | sofort (0 Zeilen) |
| MOROFF | Programmende | `/\{Compass sub Motor OFF\}/i` | `/^#\s*$/` (6 Zeilen) |
| KRC | Zwischenstopp | `/^M0{1,2}\b/` | sofort (0 Zeilen) |

Dieses Muster wird nur innerhalb einer tatsächlichen **Lücke** zwischen zwei Arbeitsgängen (oder vor dem ersten/nach dem letzten) gesucht. Da zwischen zwei KRC-Arbeitsgängen (siehe 4.2) praktisch nie eine solche Lücke entsteht, wird ein `M00` zwischen zwei Arbeitsgängen dort in der Praxis Teil des vorangehenden Arbeitsgangs statt als eigener „Zwischenstop KRC“ markiert — das KRC-Zwischenstopp-Muster bleibt trotzdem wirksam für KRC-Dateien ganz ohne erkanntes Arbeitsgang-Schema (siehe 4.4).

### 4.4 Programm ohne erkanntes Schema

Wird **kein** Arbeitsgang-Kommentarstil gefunden, bleibt das Programm trotzdem gültig und wird als Ganzes simuliert (Zwischenstopp-/Programmende-Muster werden weiterhin gesucht). Für die 3D-Darstellung wird die Datei stattdessen an jeder erkannten **Nullpunktverschiebung** in Farbabschnitte geteilt: `ORIGIN_RESET_RE = /\bO\b|\bG92(?!\d)/i` (führendes `\b` verhindert Treffer in „OPTI“/„OPEN“; **kein** abschließendes `\b` nach `G92`, damit auch das durch eine Austauschregel entstandene zusammengeklebte `G92X100.00` noch erkannt wird; `(?!\d)` verhindert Fehltreffer wie `G920`). `splitAtOriginResets(lines)` liefert die einzelnen Farbabschnitte. Dieselbe Regex wird auch von `extractMotion()` in `app.js` verwendet, um Nullpunktverschiebungszeilen von der Bahn-Darstellung auszunehmen (siehe 9.1).

### 4.5 API

```
CNCParser.parse(text, detectionText, opNames)
```
- `text`: der (ggf. bereits per aktivem Maschinentyp transformierte, siehe Abschnitt 5) anzuzeigende/zu simulierende Programmtext.
- `detectionText` (optional): der **ungetauschte** Original-Programmtext. Ist er gesetzt und zeilengleich zu `text`, läuft die Struktur-Erkennung (Maschinentyp + alle Arbeitsgang-/Zwischenstopp-/Programmende-Muster) auf `detectionText` statt auf `text` — so kann eine SWITCHCASEMACHINE-Regel, die wörtlich mit einer Erkennungsmarke kollidiert (z. B. eine Regel, die `END MACRO` gegen nichts tauscht), die Arbeitsgang-Erkennung nicht zerstören. Angezeigte/simulierte Inhalte kommen davon unberührt weiterhin aus `text`. Bei fehlender oder zeilenzahl-abweichender `detectionText` wird auf `text` zurückgefallen.
- `opNames` (optional): Array von Arbeitsgang-Namen für den `NAMED`-Stil (siehe Abschnitt 6). Ohne Angabe (oder leeres Array) liefert `NAMED` grundsätzlich keine Treffer.
- Rückgabe: `{ machine, machineLabel, sections, lineCount, lines }`. `sections` ist eine geordnete Liste von Abschnitten mit `type` (`'arbeitsgang' | 'zwischenstop' | 'programmende' | 'fixed'`), bei `'arbeitsgang'` zusätzlich `index` (1-basiert, fortlaufend), `title` und `headerEndIdx` (letzter Zeilenindex des erkannten Kommentar-/Titelblocks, siehe 9.7). `machineLabel`: bevorzugt der aus dem Kopftext erkannte Maschinenname; ist der Maschinentyp `UNKNOWN`, aber trotzdem Arbeitsgänge gefunden, wird er aus den tatsächlich genutzten Kommentarstilen zusammengesetzt (mit `+` verbunden).

```
CNCParser.extractToolData(lines)
```
Durchsucht die übergebenen Zeilen (in der Praxis: die Roh-Zeilen eines einzelnen Arbeitsgang-Abschnitts, `sec.startIdx`…`sec.endIdx`) nach Werkzeug-Radius/-Länge und weiteren Werkzeugdaten-Markern und liefert `{radius, length, isBlade, toolMode, toolType}` — siehe 9.7 für die vollständige Beschreibung der erkannten Notationen und die Aufrufstelle in `app.js`.

```
CNCParser.parseDXF(text)
```
Parser für ASCII-DXF-Dateien — siehe Abschnitt 9.8.


## 5. Maschinentyp-Auswahl (`swap.js`)

Statt einer global aktiven Regelliste gibt es mehrere benannte, editierbare **Maschinentyp-Konfigurationen**, von denen genau eine je geladenem Programm ausgewählt und explizit bestätigt wird.

### 5.1 Ablauf

1. Ein CNC-Programm wird geladen (Datei, Drag & Drop, „CNC-Code einfügen“) — der Code bleibt zunächst **unverändert** sichtbar, unabhängig davon, welcher Maschinentyp zuvor für ein anderes Programm aktiv war.
2. Eine Voraberkennung schlägt sofort einen Maschinentyp im Dropdown vor (siehe 5.4) — angewendet ist er **noch nicht**.
3. Erst ein Klick auf „Ok“ wendet den im Dropdown gerade ausgewählten Maschinentyp tatsächlich an: SWITCHCASEMACHINE-Regeln (Suchtext→Ersatztext) und danach CALCMACHINE-Achsberechnung (siehe 5.3) werden auf die Anzeige/Simulation angewendet.
4. `rawProgramText` bleibt dabei unangetastet — die Transformation ist eine reine Anzeige-/Simulationsebene (siehe `displayCurrentProgram()`), „CNC-Programm bearbeiten“/„Speichern als…“ zeigen bzw. speichern weiterhin den unveränderten Originaltext.
5. Der Sondername **„ISO“** steht immer als erste, feste Dropdown-Option für „Original, keine Umrechnung“ — er ist nach jedem neuen Laden eines Programms voreingestellter, tatsächlich angewendeter Zustand (unabhängig vom Vorschlag in Punkt 2).

### 5.2 Maschinentypdatei-Format

Eine Maschinentypdatei enthält beliebig viele Blöcke der Form:

```
[MACHINETYPE Name="Biesse"]
[SWITCHCASEMACHINE]
CR=->R
OPTI->BLA
[/SWITCHCASEMACHINE]
[CALCMACHINE]
X*1
Y*-1
Z*-1
[/CALCMACHINE]
SWIVELANGLE=45
[/MACHINETYPE Name="Biesse"]
```

`CNCSwap.parseMachineTypes(configText)` extrahiert daraus eine Liste `{ name, switchRules, calcRules, rawSwitchText, rawCalcText, swivelAngleDeg }` je Block (`swivelAngleDeg` ist `null`, wenn der Block keine `SWIVELANGLE=<Zahl>`-Zeile enthält — siehe 5.12):

- **Blockgrenzen:** Ein Block beginnt bei `[MACHINETYPE Name="…"]` und reicht bis zum **Beginn** des nächsten solchen Treffers bzw. bis zum Dateiende — der schließende Marker `[/MACHINETYPE Name="…"]` wird für die Blockgrenze nicht ausgewertet.
- **SWITCHCASEMACHINE:** Inhalt zwischen `[SWITCHCASEMACHINE]` und der nächsten Zeile, die (getrimmt) mit `[` beginnt — wird an `parseSwapRules()` (Abschnitt 5.5, Syntax `Suchtext->Ersatztext`/`§Suchtext§Ersatztext§`) übergeben.
- **CALCMACHINE:** Inhalt zwischen `[CALCMACHINE]` und der nächsten Zeile, die mit `[` beginnt — je Zeile eine Regel `Achse*Faktor` (z. B. `Y*-1`, `X*1`), geparst zu `{axis, factor}`. Zeilen außerhalb dieses einfachen Schemas werden stillschweigend ignoriert (siehe 5.10).
- **SWIVELANGLE:** eine optionale, einzelne `SWIVELANGLE=<Zahl>`-Zeile direkt innerhalb des Blocks — siehe 5.12.

**Bewusst tippfehler-tolerant:** Für die Abschnittsgrenzen zählt nur, dass die **nächste** Zeile mit `[` beginnt — der genaue Inhalt der schließenden Markierung wird nicht geprüft (z. B. wird `[/SWITCHCASEMACHIN]` ohne „E“ oder `[/CALCMACHINE ]` mit Leerzeichen genauso behandelt wie die korrekte Schreibweise). Das entspricht der sonstigen Philosophie dieses Projekts (tolerant gegenüber N-Nummern, Leerraum-Varianten etc.).

### 5.3 Achsberechnung (`applyAxisCalc`)

Anders als die reinen Suchtext→Ersatztext-Regeln muss CALCMACHINE tatsächlich mit dem Zahlenwert rechnen. `CNCSwap.applyAxisCalc(text, calcRules)` funktioniert für beliebige Achsbuchstaben und beliebige Faktoren:

- **Faktor 1:** No-op — die Achse wird gar nicht angefasst (auch keine Neuformatierung).
- **Faktor -1:** Rein textuelle Vorzeichenumkehr — Achsbuchstabe, Zwischenraum und die exakte Ziffernfolge/Nachkommastellen bleiben erhalten, kein Umweg über `parseFloat`/`toString` (verlustfrei z. B. für `100.500`). Der Wert `0` bleibt vorzeichenlos (`-0.000` → `0.000`).
- **Jeder andere Faktor** (z. B. `2`, `0.5`, `-2`, …): echte Neuberechnung (`parseFloat(num) * factor`), formatiert mit derselben Anzahl an Nachkommastellen wie im Original; auch hier wird `-0`/`-0.000…` zu vorzeichenloser Null normalisiert.

`negateYZ(text)` ist ein dünner, weiterhin getesteter Wrapper über `applyAxisCalc(text, [{axis:'Y',factor:-1},{axis:'Z',factor:-1}])`.

Wie `applySwapRules` (5.5) lässt auch `applyAxisCalc` einen Achswert unverändert, wenn er (auch nur teilweise) innerhalb eines Kommentarbereichs liegt (siehe 5.11); die Kommentar-Maske wird dabei je Achsregel neu aus dem aktuellen Zwischenstand berechnet, um Positions-Verschiebungen durch bereits vorgenommene Ersetzungen korrekt zu berücksichtigen.

### 5.4 Voraberkennung/Vorschlag

`suggestMachineType(rawText, configuredNames)` (in `app.js`, reine Vorschlagslogik ohne Anwendung) ermittelt beim Laden eines neuen Programms eine Empfehlung für die Dropdown-Vorauswahl:

1. `CNCParser.detectMachine()` liefert `'SCM'|'HOMAG'|'FORMAT4'|'MOROFF'|'KRC'|'CIX'|'UNKNOWN'`; wird auf einen passenden konfigurierten Namen gemappt, falls einer existiert (`CIX` → Programme mit Biesse-typischer `.CIX`-Kopfzeile).
2. Fallback, falls (1) nichts Passendes liefert: einfache, case-insensitive Teilstring-Suche jedes konfigurierten Maschinentyp-Namens in den ersten 100 Zeilen des Programmtexts.
3. Sonst „ISO“ (kein Vorschlag).

Der Vorschlag setzt nur die Dropdown-Auswahl, **nicht** den tatsächlich angewendeten Zustand — dieser bleibt bis zum Klick auf „Ok“ bei „ISO“ (siehe 5.1).

### 5.5 Regel-Syntax (SWITCHCASEMACHINE)

Eine Regel je Zeile, in einem von zwei Schemata (auch gemischt):

1. **Altes Schema:** `§Suchtext§Ersatztext§`
2. **Neues Schema:** `Suchtext->Ersatztext` — steht nach `->` nichts oder nur `/`, wird der Suchtext gegen nichts getauscht (entfernt). Führende/nachgestellte Leerzeichen in Such- und Ersatztext werden **nicht** getrimmt (bedeutsam, z. B. `O ->G92`).

`parseSwapRules(text)` verwendet eine `Map` (Suchtext → Ersatztext): Kommt derselbe Suchtext mehrfach vor, gewinnt der **letzte** Eintrag im jeweiligen SWITCHCASEMACHINE-Abschnitt. `applySwapRules(text, rules)` wendet alle Regeln in **einem** Durchlauf von links nach rechts an — an jeder Position wird unter allen passenden Regeln die mit dem **längsten** Suchtext gewählt („Maximal-Munch“, nicht die Datei-Reihenfolge); ersetzter Text wird nicht erneut gescannt. Die Regeln werden auf die **gesamte** Rohdatei angewendet, **bevor** der Arbeitsgang-Parser läuft (Ausnahme: die Struktur-Erkennung selbst läuft dank `detectionText`, siehe 4.5, auf dem ungetauschten Text). Ein an sich passender Treffer wird zusätzlich übersprungen, wenn er (auch nur teilweise) innerhalb eines Kommentarbereichs liegt (siehe 5.11).

### 5.6 Standard-Maschinentypdatei

Beim ersten Programmstart (bzw. wenn noch nichts in localStorage gespeichert ist, siehe 5.8) ist eine fest eingebettete Standard-Konfiguration aktiv (`DEFAULT_MACHINETYPES_TEXT`, alphabetisch nach Name sortiert). Aktueller Stand — neun Maschinentyp-Blöcke:

| Maschinentyp | SWITCHCASEMACHINE | CALCMACHINE |
|---|---|---|
| Biesse | 14 Regeln (u. a. `C1=->C`, `B1=->A`, `CR=->R`, `OPTI->BLA`, MACRO-Block-Entfernung, `G2`↔`G3`-Tausch, `G42`↔`G41`-Tausch) | `X*1`, `Y*-1`, `Z*-1`, `C*-1` |
| FORMAT4 | 2 Regeln (`G77H->M6 T`, `G59->G92`) | `X*1`, `Y*1`, `Z*1` |
| HolzHer | 4 Regeln (`CALL _DINISO ( VAL CODE:='->/`, `')->/`, `G03->G3`, `G02->G2`) | `X*1`, `Y*1`, `Z*1` |
| HOMAG | 16 Regeln (u. a. `CR=->R`, `SUPA/G153/…A->/`, `G03->G3`, `G02->G2`) | `X*1`, `Y*1`, `Z*1` |
| Houfek | 5 Regeln (u. a. `Z1=->Z`, `Z2=->Z`) | `X*1`, `Y*1`, `Z*1` |
| MAKA | 4 Regeln (`G03->G3`, `G02->G2`, `Y->X`, `X->Y`) | `X*1`, `Y*1`, `Z*1` |
| MKM | 2 Regeln (`G03->G3`, `G02->G2`) | `X*1`, `Y*1`, `Z*1` |
| Reichenbacher | 1 Regel (`TRANS->G92`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `SWIVELANGLE=45` außerhalb von CALCMACHINE, siehe 5.12) |
| SCM | 15 Regeln (u. a. `G13D->G1`, `G03D->G0`, `H->Z`, `N X->` [entfernt „N X“], `Q->C`, `R->A`, `C=2->G41`, `PB->;PB`, `XN X=->;X=`, `X=-->;X=`) | `X*1`, `Y*1`, `Z*1` |

Nur **Biesse** hat aktuell eine von 1 abweichende CALCMACHINE-Berechnung (Y/Z/C-Vorzeichenumkehr) — bei allen übrigen acht Maschinentypen dient CALCMACHINE (Stand der Standard-Datei) noch keiner tatsächlichen Umrechnung. Der Benutzer kann Regeln jederzeit über „Maschinentypen bearbeiten“ erweitern.

Beim Biesse-Block bilden die beiden ersten Regeln `C1=->C`/`B1=->A` die Kippachse des zugehörigen Sägeaggregats rein textuell VOR dem eigentlichen Parsen ab — aus „B1=-90.01“ wird „A-90.01“, aus „C1=-12.49“ wird „C-12.49“ (klassische, direkt unterstützte Achsworte-Notation, siehe `AXIS_RE`/9.1). Laut dieser Maschinentyp-Definition entspricht der Kippkanal „B1“ dieses Aggregats kinematisch der Kipp-Achse **A** (Rotation um X, siehe `toolAxisDirection()`) des Simulators, nicht der Kipp-Achse B — eine reine Konvention dieser konkreten Maschine, die der generischen Fallback-Interpretation ohne gewählten Maschinentyp (dort wird „B1=“ mangels besseren Wissens direkt auf die gleichnamige Achse B abgebildet) widerspricht; sobald „Biesse“ im Dropdown gewählt und mit „Ok“ bestätigt wird, gewinnt diese textuelle Vorab-Übersetzung, da `applySwapRules()` vor dem eigentlichen Parsen läuft.

Beim SCM-Block kommentieren die drei letzten Regeln `PB->;PB`, `XN X=->;X=` und `X=-->;X=` bestimmte Zeilen(-anfänge) gezielt aus, statt sie zu ersetzen — z. B. wird aus „XN X=0 Q=0 IF=FLD=14“ ein „;X=0 C=0 IF=FLD=14“. `XN X=->;X=` ist dabei bewusst **länger und spezifischer** als die bereits bestehende, kürzere Regel `N X->` (entfernt „N X“): Ohne die längere Regel würde bei „XN X=…“ wegen „Maximal-Munch an der am weitesten links liegenden Position“ (siehe 5.5) die kürzere `N X`-Regel zuerst zuschlagen und dabei das „X“ „aufessen“, das eine reine `X=->;X=`-Regel an dieser Stelle zum Feuern noch gebraucht hätte — ein Beispiel für die in 5.10 beschriebene Klasse von Regel-Kollisionen durch literale Teilstring-Überschneidung.

### 5.7 Dropdown-Inhalt, Sortierung, Duplikat-Warnung

Der Dropdown-Inhalt wird bei jedem Neuaufbau (`populateMachineTypeSelect()`) **dynamisch** aus den `Name="…"`-Werten der aktuellen Konfiguration gebildet — kein hartkodierter, fester Namenskatalog. „ISO“ steht dabei immer fest an erster Stelle, alle übrigen Namen folgen automatisch alphabetisch sortiert (case-insensitiv), unabhängig von ihrer Reihenfolge in der Konfigurationsdatei. Das Dropdown samt „Ok“-Knopf steht als hervorgehobene `.machine-type-quick-group` (fett/großgeschrieben in Akzentfarbe) direkt im Menüband, siehe Abschnitt 10.

Beim Bearbeiten der Konfiguration (Panel „Maschinentypen bearbeiten“) prüft `CNCSwap.findDuplicateMachineTypeNames()` auf mehrfach vergebene Namen (Vergleich case-insensitiv) und zeigt bei einem Treffer eine sichtbare, aber **nicht blockierende** Warnung im Panel an (`#machineTypeDupWarning`) — das Speichern selbst wird dadurch nicht verhindert.

**Alphabetische Sortierung des rohen Konfigurationstexts selbst** (zu unterscheiden von der oben beschriebenen Dropdown-Sortierung): `CNCSwap.sortMachineTypeBlocksAlphabetically(configText)` ordnet die kompletten Block-**Text-Ausschnitte** (vom öffnenden `[MACHINETYPE Name="…"]`-Marker bis zum nächsten Block bzw. Dateiende, jeweils inklusive ihres gesamten Inhalts byte-identisch) alphabetisch nach Name um (case-insensitiv) — reine Verschiebung ganzer Blöcke, kein Eingriff in deren Inhalt. Etwaiger Text vor dem allerersten Block bleibt unverändert oben stehen; bei weniger als zwei Blöcken bzw. leerem/fehlendem Text bleibt der Text unverändert. Angewendet wird diese Umsortierung beim Import einer Einstellungsdatei über „Einstellungen importieren“ (5.9 unten) auf das darin enthaltene `machineTypesConfigText`. **Nicht** automatisch umsortiert wird dagegen der aus localStorage wiederhergestellte Text bei einem gewöhnlichen Seiten-Reload sowie der Text, während er im „Maschinentypen bearbeiten“-Panel von Hand editiert wird.

Die frühere Möglichkeit, die komplette Maschinentyp-Konfiguration über einen eigenen „Maschinentypen laden“-Dateidialog zu ersetzen, entfällt — dieselbe Funktion ist bereits vollständig über „Einstellungen importieren“ (siehe unten) abgedeckt, dessen JSON-Datei `machineTypesConfigText` vollständig enthält. Das Untermenü „Maschinentypen“ (Abschnitt 10) besteht deshalb nur noch aus dem einen Eintrag „Maschinentypen bearbeiten“.

### 5.8 localStorage-Persistenz

Gespeichert werden der Text der Maschinentyp-Konfiguration (`cncsim.machineTypesConfigText`), der Name des zuletzt per „Ok“ bestätigten Maschinentyps (`cncsim.activeMachineTypeName`) sowie die Arbeitsgang-Namensliste (`cncsim.opNameListText`, siehe Abschnitt 6) — **nicht** das geladene CNC-Programm selbst, das geht beim Neuladen der Seite weiterhin verloren. Alle Lese-/Schreibzugriffe sind per `try`/`catch` abgesichert und fallen bei fehlender Verfügbarkeit (z. B. mancher privater Browser-Modus) einfach auf Standardwerte zurück, ohne die Anwendung zu beeinträchtigen.

### 5.9 Einstellungen exportieren/importieren

„Einstellungen exportieren“ (`#btnSaveSettings`) und „Einstellungen importieren“ (`#btnLoadSettings`, beide im „Einstellungen“-Menü, siehe Abschnitt 10) ergänzen die localStorage-Persistenz um einen expliziten Datei-Export/-Import, z. B. um dieselben Einstellungen auf einen anderen Rechner/Browser zu übertragen.

„Einstellungen exportieren“ lädt sofort (ohne Zwischendialog) eine JSON-Datei `cncsim_einstellungen.json` mit dem aktuellen Stand herunter — analog zu „Speichern als…“ über einen unsichtbaren `<a download>`-Link:

```json
{
  "version": 1,
  "opNameListText": "…",
  "machineTypesConfigText": "…",
  "activeMachineTypeName": "Biesse",
  "colorProfileText": "…",
  "columnScaleCode": 100,
  "columnScaleOps": 1,
  "menuHighlightBg": "#ad1f2d",
  "menuHighlightInk": "#ffffff",
  "fontColors": { "code": "#00ff00" }
}
```

Enthalten sind die Arbeitsgang-Namensliste (Abschnitt 6), die komplette Maschinentyp-Konfiguration (Abschnitt 5), der Name des zuletzt bestätigten Maschinentyps, das Farbprofil (12.1), die beiden „Spalteneinstellungen“-Werte (`columnScaleCode`/`columnScaleOps`, siehe 7.3a), die Menü-Hervorhebungsfarbe (`menuHighlightBg`/`menuHighlightInk`, siehe 12.2) und die gesetzten Schriftfarben (`fontColors`, siehe 12.3) — bewusst **nicht** das geladene CNC-Programm oder eine importierte DXF-Unterlage (dieselbe Abgrenzung wie bei der localStorage-Persistenz).

„Einstellungen importieren“ öffnet eine solche Datei wieder (Dateiauswahl über ein verstecktes `<input type="file">`) und wendet alle enthaltenen Teile über dieselben Pfade an, die auch bei manueller Änderung benutzt werden (`applyOpNameList()`, `CNCSwap.parseMachineTypes()`, `applyColorProfile()`, `normalizeColumnScale()`+`applyProgCollapseStage()`/`setOpsWidth()`, `applyMenuHighlightColor()`, `applyFontColors()` — jeweils + Persistieren) — die importierten Werte werden dadurch sofort wirksam und zusätzlich in localStorage übernommen. Fehlt ein Feld in der geladenen Datei, bleibt der jeweils aktuell aktive Wert unverändert, statt auf einen Standardwert zurückzufallen. Ist die Datei kein gültiges JSON, erscheint ein einfacher Warnhinweis, ohne etwas zu verändern.

### 5.10 Bekanntes Restrisiko (SWITCHCASEMACHINE/CALCMACHINE)

Da die Regeln rein literal auf Zeichenketten wirken (kein G-Code-Tokenizer) und auf die gesamte Rohdatei angewendet werden, kann eine kurze/generische Regel (z. B. ein einzelner Buchstabe wie `H->Z`) zufällig auch mitten in einem eigentlich unbeteiligten Wort oder einer Erkennungsmarke treffen. Für die in Abschnitt 4 beschriebenen strukturellen Erkennungsmarken schützt der `detectionText`-Mechanismus (4.5); für sonstigen Freitext gilt seit der in 5.11 beschriebenen Kommentar-Erkennung ein reduziertes Restrisiko.

Konkretes Beispiel (Sinumerik/Reichenbacher-WinPPZ-Programm mit Unterprogrammaufrufen `H340`/`H342`): Eine Regel `H->Z` ist für WinPPZ-Dialekte gedacht, die die Z-Achse mit dem Buchstaben `H` statt `Z` bezeichnen. In einem Programm, das `H` stattdessen (auch) als Adresse für Unterprogramm-/Technologiezyklus-Aufrufe verwendet, erzeugt dieselbe Regel aus `H340`/`H342` die Achswörter `Z340`/`Z342` — zwei zusätzliche, nicht real gefräste Bahnpunkte. Da die Regel literal auf Zeichenketten wirkt, kann dieser Fall nicht generisch unterschieden werden; betroffen sind nur Programme, die `H` gleichzeitig als Achs- und als Adressbuchstaben verwenden. Dasselbe Restrisiko gilt analog für CALCMACHINE: eine zufällige `Achsbuchstabe`+Zahl-Zeichenfolge außerhalb eines echten Achswortes wird ebenso mit umgerechnet.

**Durch die Kommentar-Erkennung (5.11) eingeschränkt:** Dieses Restrisiko besteht nur noch für Freitext, der in **keiner** der drei erkannten Kommentar-Arten (`;`, `KM="…"`, `{…}`) sowie außerhalb runder Klammern steht — läge das `H340`/`H342`-Beispiel etwa hinter einem `;` oder in `{…}`/`KM="…"` eingeschlossen, würde es nicht mehr angefasst.

CALCMACHINE unterstützt zudem ausdrücklich nur das einfache Schema `Achse*Faktor` — komplexere Formeln (z. B. eine trigonometrische Winkelberechnung) sind nicht umgesetzt; Zeilen außerhalb des einfachen Schemas werden von `parseCalcRules()` stillschweigend ignoriert statt einen Fehler zu werfen.

### 5.11 Kommentar-Erkennung beim Austausch (SWITCHCASEMACHINE & CALCMACHINE)

Sowohl `applySwapRules()` (SWITCHCASEMACHINE, 5.5) als auch `applyAxisCalc()` (CALCMACHINE, 5.3) berechnen vor der eigentlichen Anwendung einmal eine Kommentar-Maske über den (Zwischen-)Text (`computeCommentMask()`) und lassen einen an sich passenden Treffer **unverändert stehen**, wenn er (auch nur teilweise) innerhalb eines erkannten Kommentarbereichs liegt (`isRangeUncommented()` — ein Treffer wird nur ersetzt, wenn er **vollständig** außerhalb jedes Kommentarbereichs liegt).

Erkannt werden drei Kommentar-Arten:

1. **`;` bis zum Zeilenende** (Sinumerik/WinPPZ-Standard, u. a. SCM/HOMAG/Houfek) — wirkt auch **mitten** in einer Zeile (Inline-Kommentar), nicht nur am Zeilenanfang.
2. **`KM="…"`** inklusive der Anführungszeichen (HOMAG-Kommentar-/Struktur-Tag).
3. **`{…}`** (Moroff-Stil, nicht verschachtelt).

**Bewusst NICHT ausgenommen: runde Klammern `(…)`.** Mehrere Standard-Maschinentypen (5.6) nutzen runde Klammern nicht als reinen Kommentar, sondern als Teil des eigentlichen, funktionalen Zielmusters ihrer SWITCHCASEMACHINE-Regeln — z. B. **HolzHer** (`CALL _DINISO ( VAL CODE:='->/`, `')->/`, zielt auf den funktionalen Unterprogrammaufruf) und **Biesse** (`BEGIN MACRO->/`, `NAME=ISO->/`, `PARAM,NAME=ISO,VALUE="->/`, `END MACRO->/`, entfernen klammerförmige Makro-Blöcke). Würden runde Klammern ebenfalls als Kommentar ausgenommen, griffe HolzHers gesamtes Regelwerk gar nicht mehr und bei Biesse mehrere Regeln nicht mehr. Runde Klammern bleiben daher ein normaler, austauschbarer Zeichenbereich.

**Bekannter Sonderfall:** Ein `;` **innerhalb** eines `{…}`- oder `KM="…"`-Bereichs verlängert die `;`-Maskierung regelbasiert bis zum Zeilenende, auch über das eigentliche Ende des `{…}`/`KM="…"`-Bereichs hinaus (keine echte verschachtelte Klammerbalance-Prüfung) — ein bewusst in Kauf genommener Grenzfall.

### 5.12 Konfigurierbarer Reichenbacher-Schwenkwinkel (`SWIVELANGLE`)

Die Reichenbacher-Kardanwinkelkorrektur (9.7) ist eine nichtlineare, trigonometrische Formel, die B UND C zu einem gemeinsamen 3D-Richtungsvektor koppelt und auf den intern berechneten 3D-Werkzeugachsenvektor wirkt — das lässt sich in CALCMACHINEs `Achse*Faktor`-Schema nicht ausdrücken. Der einzige darin frei wählbare, skalare Parameter der Formel ist der Schwenkwinkel des Aggregats selbst — bei Reichenbachers Bauart baubedingt 45°, aber im Prinzip eine reine Konstruktionsgröße. Deshalb ist nur dieser eine Winkel konfigurierbar, während die eigentliche trigonometrische Berechnung fest in `app.js` verdrahtet bleibt.

**Syntax:** Eine einzelne, einfache `SWIVELANGLE=<Zahl>`-Zeile (Groß-/Kleinschreibung des Schlüsselworts egal, Leerzeichen um `=` toleriert, Dezimal- und Minuswerte erlaubt) **direkt innerhalb** eines `[MACHINETYPE …]`-Blocks (an beliebiger Position). `parseSwivelAngle(blockLines)` (`swap.js`, `SWIVEL_ANGLE_RE = /^SWIVELANGLE\s*=\s*(-?\d+(?:\.\d+)?)$/i`) durchsucht alle Zeilen des Blocks nach dem ersten Treffer und liefert dessen Zahlenwert, bzw. `null`, wenn keine Zeile passt. `parseMachineTypes()` hängt das Ergebnis als `swivelAngleDeg` an jeden Block-Eintrag an.

**Konsumiert von `reichenbacherToolAxisDirection(p)`** (`app.js`, 9.7):

```
x'' = sin(S) · sin(B)
y'' = -sin(S) · cos(S) · (1 − cos(B))
z   = cos(S)² + cos(B) · sin(S)²
vx  = cos(C)·x'' − sin(C)·y''
vy  = sin(C)·x'' + cos(C)·y''
vz  = z
```

`S` liefert `getReichenbacherSwivelAngleDeg()`: Sucht den aktuell **aktiven** Maschinentyp (`activeMachineTypeName`) in `machineTypes` und liest dessen `swivelAngleDeg` — ist der Typ nicht auffindbar oder `swivelAngleDeg` `null`/keine gültige Zahl, wird auf **45°** zurückgefallen. Eine geänderte `SWIVELANGLE`-Zeile wirkt sich unmittelbar nach „Übernehmen“ in „Maschinentypen bearbeiten“ auf die 3D-Darstellung aus, ohne dass das Programm neu geladen werden muss.

**Namenserkennung des Reichenbacher-Schalters — Teilstring, nicht Exaktvergleich:** `isReichenbacherTypeName(name)` (`app.js`) prüft `name.toLowerCase().includes('reichenbacher')` — case-insensitive Teilstring-Suche, egal ob „Reichenbacher“ am Anfang, in der Mitte oder am Ende des Namens steht (z. B. „Reichenbacher 2026“, „Neu Reichenbacher“). `getReichenbacherSwivelAngleDeg()` selbst sucht dagegen exakt nach dem konkreten, aktiven Namen, welcher Name das im Einzelfall auch ist.

### 5.13 „Simulation speichern“ — komplette Seite mit eingebackenen Einstellungen

Anders als „Einstellungen exportieren“/„Einstellungen importieren“ (5.9, muss vom Benutzer aktiv wieder importiert werden) erzeugt „Simulation speichern“ (Menü „Datei“, siehe Abschnitt 10) eine **vollständig eigenständige, sofort lauffähige Kopie der kompletten Simulations-Seite selbst** als `.html`-Datei (`CNC_Simulation.html`) — der Benutzer öffnet diese Datei beim nächsten Mal einfach direkt (z. B. Doppelklick), ganz ohne Datei-Import-Schritt, und landet dort bereits mit seinen aktuellen Maschinentyp-/Arbeitsgang-Namen-Einstellungen als neuen Standardwerten, statt wieder bei den eingebauten Werksvorgaben zu starten. Diese Funktion ist eine reine Zwischenlösung, bis eine „richtige“ Programm-Installation mit echter Einstellungs-Persistenz existiert.

**Umsetzung:** `PRISTINE_PAGE_HTML` (app.js, ganz oben) ist ein Schnappschuss von `document.documentElement.outerHTML`, genommen als allererste Anweisung des Scripts — also bevor auch nur eine einzige DOM-Änderung stattfinden konnte. Dieser Schnappschuss entspricht dadurch exakt der ursprünglich ausgelieferten Seite, unabhängig vom aktuellen Laufzeitzustand. `buildSavedSimulationHtml()` ersetzt darin drei mit `SIM_SAVE:*:BEGIN`/`SIM_SAVE:*:END`-Kommentaren fest markierte Konstanten-Deklarationen durch die aktuell aktiven Werte:

- `DEFAULT_MACHINETYPES_TEXT` ← aktueller `machineTypesConfigText`
- `DEFAULT_OPNAME_TEXT` ← aktueller `opNameListText`
- `DEFAULT_ACTIVE_MACHINETYPE_NAME` ← aktueller `activeMachineTypeName` (der zuletzt per „Ok“ bestätigte Maschinentyp — eine im Dropdown nur ausgewählte, aber noch nicht per „Ok“ bestätigte Auswahl zählt nicht)

Jeder Wert wird über `JSON.stringify()` sicher als einzeiliges, syntaktisch gültiges JS-String-Literal eingebettet. Ein fehlender Marker bricht den Vorgang nicht ab, sondern lässt den betroffenen Abschnitt einfach unverändert. Die BEGIN/END-Markerzeilen selbst bleiben in der gespeicherten Kopie erhalten, sodass auch aus einer bereits gespeicherten Kopie heraus erneut „Simulation speichern“ funktioniert (verkettbar).

**Enthält bewusst NICHT** das aktuell geladene CNC-Programm (`rawProgramText`, geparste Arbeitsgänge, 3D-Ansicht-Zustand usw.) — unabhängig davon, ob beim Klick gerade ein Programm geladen ist oder nicht, da `PRISTINE_PAGE_HTML` der Schnappschuss vor jeder Programmladung ist. Der Dateiname wird beim Herunterladen identisch zum Original gewählt (`CNC_Simulation.html`).


## 6. Arbeitsgang-Namensliste (NAMED-Stil)

Editierbare Liste (ein Name je Zeile) für Postprozessoren ohne Trennzeilen-Konvention (siehe `MACHINES.NAMED` in Abschnitt 4.2). `parseOpNameList(text)` entfernt Leerzeilen und exakte Duplikate (Reihenfolge des ersten Vorkommens bleibt erhalten); Groß-/Kleinschreibung und Umlaute bleiben im gespeicherten Text unverändert, der Abgleich selbst erfolgt case-insensitiv. Über „Arbeitsgang-Namen bearbeiten“ jederzeit erweiterbar. Standardliste, 92 eindeutige Einträge (Auszug):

```
Nivellieren
Abräumen
Stabseite 1
Stabseite 2
Seitenbahnen 1
Seitenbahnen 2
Stirnseiten schruppen 1
Stirnseiten schruppen 2
Nut
Minizinken Antritt
Minizinken Austritt
Rechteckstab
Rundstab
Geländerstab Dübelbohrungen
Abstandhalter
Handlaufstütze
Dübelbohrungen
Schraube (Loch)
Schraube mit Doppeltasche
Profil 1 … Profil 6
Stirnseiten schlichten 1
Stirnseiten schlichten 2
Trennschnitt
Schruppen
Schlichten
Bohrung Stab
Bohrung Rechteckstab (Schruppen/Schlichten)
Bohrung Dübel
Bohrung Bodenschraube (Senkloch/Abdeckkappe)
Außen (großer Radius)/… (Stufen/Setzstufen/Ober-Unterkante schruppen/schlichten 5-Achs, Rechteckstab, Rundstab, Stirn schruppen, Abrichten, Ausräumen, Oberfläche/5-Achs, Handlaufprofil 5-Achs)
Innen (kleiner Radius)/… (Abrichten, Ausräumen, Oberfläche/5-Achs, Dübel Stirnseite, Bolzen, Spannschraube, Doppelschraube, Stufen/Setzstufen schruppen/schlichten 5-Achs, Handlaufprofil 5-Achs)
… (weitere Bohrungs-/Kontur-/Profilvarianten)
```

Bewusst **kein** Klammer-Stripping in diesem Stil (`(Titel)` wird nicht zusätzlich als NAMED-Treffer gewertet), um keine Doppelerkennung mit dem KRC-Block-Stil zu erzeugen.

**Persistenz:** Eine Bearbeitung über „Arbeitsgang-Namen bearbeiten“ → „Übernehmen“ übersteht auch ein Schließen/Neuladen der Seite (`cncsim.opNameListText` im localStorage, genau wie die Maschinentyp-Konfiguration, siehe 5.8).

## 7. Programm-Ansicht

### 7.1 Code-Liste

Das gesamte geladene Programm wird links als **eine durchgehende**, nummerierte Zeilenliste angezeigt (keine einzeln aufklappbaren Abschnitts-Boxen je Arbeitsgang/Zwischenstopp). Jede Zeile ist eingefärbt (`computeLineColors()`): sind Arbeitsgänge erkannt, erhält jede Zeile die Farbe ihres Arbeitsgangs (`pathColor(index)`, siehe Abschnitt 12); ohne erkanntes Schema greift stattdessen die Farbaufteilung nach Nullpunktverschiebungen (`splitAtOriginResets`, siehe 4.4). Sämtliche Leerzeilen (nach Trimmen, nach Anwendung des aktiven Maschinentyps) werden vollständig ausgeblendet — unabhängig davon, ob sie bereits im Original leer waren oder erst durch die Transformation entstanden sind.

Die aktuell simulierte/gescrubbte Zeile wird zusätzlich hervorgehoben (`.active-line`: Akzentfarbe + `--accent-soft`-Hintergrund) — **ohne** einen eigenen Dreh-/Kippwinkel-Hinweis direkt an dieser Zeile: Ein früher hier per `<span class="line-angle-badge">` eingefügter Text saß als normaler Textknoten im Programmcode-Element selbst und wurde dadurch beim Markieren/Kopieren einer Codezeile ungewollt mit ausgewählt. Die Dreh-/Kippwinkel-Anzeige gibt es stattdessen ausschließlich im Info-Fenster des 3D-Bereichs (`#readout`/`AXIS_META`, siehe 9.6) — komplett unabhängig von der Programmcode-Spalte.

Aus Performance-Gründen trägt jede `.code-line` `content-visibility: auto` mit `contain-intrinsic-size: auto 18.5px` als Platzhalterhöhe UND eine strukturell feste Höhe `flex: 0 0 18.5px; overflow: hidden` (`#tree` ist als `display: flex; flex-direction: column` mit potenziell zehntausenden `.code-line`-Elementen als Flex-Kindern aufgebaut) — dadurch muss der Flex-Container die Größenverteilung der Geschwister-Zeilen nicht neu berechnen, egal welcher Inhalt sich innerhalb einer einzelnen Zeile ändert (z. B. die Hervorhebung selbst). Die aktive Zeile wird ausschließlich über Farbe, Hintergrund und linken Rahmen hervorgehoben, nicht über `font-weight` (eine Textbreiten-/Fettschrift-Änderung in dieser großen Flex-Liste würde ansonsten einen teuren Neuaufbau der gesamten Liste erzwingen).

Ein Zeilensprung (Klick in der Arbeitsgänge-Spalte oder „Sprung in Zeile“) schaltet `#tree` beim ersten Sprung dauerhaft auf die CSS-Klasse `force-layout` um (`.tree.force-layout .code-line { content-visibility: visible }`) und erzwingt einen synchronen Reflow, bevor das native `scrollIntoView({block:'center'})` springt — nötig, da Chromium bei zuvor nie gerenderten `content-visibility:auto`-Zeilen die `.tree`-Scrollhöhe sonst nur unvollständig berechnet und der Sprung an der falschen Stelle landet. Die `content-visibility:auto`-Optimierung bleibt für den Normalfall (Laden, Scrubben/Abspielen ohne Zeilensprung) vollständig erhalten; erst der erste tatsächliche Sprung „bezahlt“ einmalig den vollen Layout-Preis und bleibt danach dauerhaft exakt.

`updateLineHighlight()`/`updateActiveOpHighlight()` überspringen Klassenwechsel/`scrollIntoView()` vollständig, wenn sich die betroffene Zeile bzw. der betroffene Arbeitsgang seit dem letzten Wiedergabe-Tick nicht geändert hat (z. B. bei einem tessellierten Kreisbogen mit mehreren Punkten auf derselben Quelltextzeile) — sie prüfen zuerst `lineEl !== lastActiveLineEl` bzw. `opIndex !== lastActiveOpIndex`.

Rechts neben der Kopfzeilen-Überschrift „CNC-Programmcode“ (bzw. dem Reiter, siehe 7.1b) sitzt ein Edit-Icon (`#btnEditProgramIcon`, „✎“, 24×24px) — öffnet über `openProgramEditPanel()` denselben Editor wie der Menüpunkt „CNC-Programm bearbeiten“ (siehe Abschnitt 11); beide Auslöser sind an denselben beiden Stellen (de)aktiviert (aktiv nach dem Laden eines Programms, deaktiviert nach „CNC-Programm löschen“).

### 7.1b Reiter „CNC-Programmcode“ / „DXF-Layer“

Die Kopfzeile der Programmcode-Spalte ist ein Reiter-Paar (`#sidebarTabCode`/`#sidebarTabDxf`, siehe 9.8a für die vollständige Umschalt-/Prioritätslogik) statt einer reinen Überschrift — sichtbar/anklickbar abhängig davon, ob ein CNC-Programm und/oder ein DXF geladen sind.

Der Arbeitsgang-Zähler steht in der Kopfzeile der Arbeitsgänge-Spalte selbst (`Arbeitsgänge (N)`, siehe 7.2) statt in der Programmcode-Überschrift; Letztere zeigt nur bei fehlendem Arbeitsgang-Schema einen Hinweistext („Kein Arbeitsgang-Schema erkannt · gesamtes Programm“).

### 7.2 Arbeitsgänge-Spalte

Rechts neben der Code-Liste, ein-/ausklappbar (auf 32px eingeklappt, siehe 7.3) und listet jeden erkannten Arbeitsgang als eigene Zeile mit Farbpunkt, laufender Nummer, Titel, einem „Auge“-Knopf und einem „3D“-Knopf. Die Kopfzeile (`.ops-col-head`) ist zweizeilig: oben die Titelzeile mit „Arbeitsgänge (N)“ (`#opsColTitle`, N = Anzahl erkannter Arbeitsgänge) und dem Ein-/Ausklappen-Knopf, darunter auf voller Breite ein Umschalt-Knopf für „alle ausblenden“/„alle wieder einblenden“.

- Klick auf die Zeile springt im Code zur Startzeile des Arbeitsgangs **und** bewegt die Wiedergabeposition (Scrubber) dorthin (`jumpToCodeLine` → `scrubToLineIdx`), sodass ein anschließendes Abspielen tatsächlich ab dieser Stelle weiterläuft.
- Klick auf das „Auge“-Symbol (`toggleOpVisibility(opIndex)`) blendet die Bahn genau dieses einen Arbeitsgangs im 3D-Bereich aus bzw. wieder ein (diagonaler Strich über dem Auge-Glyph, Zeile zusätzlich gedimmt). Wirkt **ausschließlich** auf das Zeichnen (siehe 9.3/9.5) — Wiedergabe, Scrubber und Scope-Auswahl sind davon unberührt. Die ausgeblendeten Arbeitsgänge (`hiddenOpIndexes`, eine `Set` von 1-basierten Indizes) werden bei jedem neu geladenen bzw. gelöschten Programm zurückgesetzt.
- Der Umschalt-Knopf im Spalten-Kopf (`#btnShowAllOps`) richtet Label/Optik ausschließlich nach `hiddenOpIndexes.size`: leer → Label „Alle AG ausblenden“, ein Klick blendet alle aus; nicht leer (ob einzeln oder vollständig ausgeblendet) → Label „Alle AG anzeigen“, ein Klick setzt alles zurück.
- Klick auf „3D“ setzt den 3D-Scope auf genau diesen Arbeitsgang (`setScope({type:'op', index})`, siehe 9.3) — echter Umschalter: Ist genau dieser Arbeitsgang bereits der aktive Scope, wechselt ein erneuter Klick stattdessen zu `{type:'all'}` („Gesamtes Programm“); ein Klick auf einen anderen Arbeitsgang wechselt direkt zu diesem.
- **Unabhängig davon** wird der Arbeitsgang, dessen Zeilenbereich die aktuell simulierte/gescrubbte Position enthält, laufend mit der eigenen Klasse `.current` hervorgehoben und automatisch in den sichtbaren Bereich gescrollt — auch in der Standardansicht „Gesamtes Programm“. `.current` und `.active` (aktiver Scope) sind unabhängige Klassen und können gleichzeitig zutreffen.

### 7.3 Automatische Spaltenbreiten & Ein-/Ausklappen

Die natürliche Breite der Programmcode-Spalte wird per Canvas-2D `measureText()` aus dem tatsächlichen Inhalt berechnet (`computeTreeContentWidth`: längste Codezeile in Mono-Schrift plus Zeilennummer-Spalte/Ränder/Sicherheitsabstand) und setzt darüber die per Ziehgriff verstellbare `sidebarWidth` neu (Grenzen: `max(360, min(900, innerWidth-260))`).

Die Programm-Spalte kennt **fünf Einklapp-Stufen** (`progCollapseStage`, 0–4): `PROG_STAGE_PCT = [100, 75, 50, 25, 0]` — Stufe 0 = volle automatisch ermittelte Breite, Stufe 4 = vollständig eingeklappt (Mindestbreite in dem Fall 160px statt der sonst geltenden 220px). Jedes (Neu-)Laden eines Programms setzt die Stufe auf die in „Spalteneinstellungen“ (7.3a) hinterlegte, dauerhaft persistierte Standard-Stufe zurück. Zwei feste, gerichtete Knöpfe im Spaltenkopf bewegen die Stufe jeweils nur in ihre eigene Richtung und bleiben am erreichten Ende stehen (kein zyklisches Umlaufen): **Zuklappen** (`#btnProgCollapse`, „«“, physisch links, Stufe steigt 0→4, jeweils −25 Prozentpunkte, bei Stufe 4 deaktiviert) und **Aufklappen** (`#btnProgExpand`, „»“, physisch rechts, Stufe sinkt 4→0, bei Stufe 0 deaktiviert). Ein deaktivierter Knopf zeigt zusätzlich zur Abdunklung (`opacity: .4`) eine diagonale Durchstreichung des Icons.

Diese Stufe wirkt **nur für die laufende Sitzung**: Sie überschreibt die aus „Spalteneinstellungen“ geladene Standard-Stufe temporär, wird aber beim nächsten Laden/Neuladen eines Programms wieder auf die dort persistierte Standard-Stufe zurückgesetzt.

Die Arbeitsgänge-Spalte kennt dagegen **nur zwei Zustände** (`columnScalePreference.ops`, ein strikter Binärwert 0/1) statt Zwischenstufen — ausgeklappt bedeutet immer die volle, automatisch aus dem längsten Arbeitsgang-Eintrag ermittelte Breite (`computeOpsContentWidth()`: berücksichtigt sowohl den breitesten Listeneintrag inkl. Farbpunkt/Nummer/Titel/„Auge“-Knopf/„3D“-Knopf als auch die Kopfzeile selbst inkl. beider möglichen Umschalt-Knopf-Beschriftungen — die größere der beiden Breiten gewinnt), eingeklappt bedeutet 32px. Eine zentrale Funktion `applyOpsScaleToCurrent()` wendet diese Breite überall konsistent an (nach dem Laden, nach „Übernehmen“ im Spalteneinstellungen-Dialog, nach JSON-Import, nach „Standardwerte wiederherstellen“).

Der Ziehgriff links der Arbeitsgänge-Spalte (`#opsResizer`) verändert **nicht** deren eigene Breite, sondern verschiebt die gesamte Programm-Spalte (`setSidebarWidth((e.clientX - rect.left) + opsColSpacePx())` auf `#sidebarCol`) — da die Arbeitsgänge-Spalte als Flex-Kind rechts daneben sitzt und ihre feste, automatisch ermittelte Breite behält, „wandert“ sie beim Breiterwerden der Programmcode-Spalte sichtbar mit nach rechts, statt selbst schmaler zu werden. Im eingeklappten Zustand ist der Ziehgriff inaktiv.

### 7.3a Spalteneinstellungen — persistierte Standard-Skalierung

Menüpunkt „Spalten“ (`#btnColumnScale`, unter „Einstellungen“ → „Erscheinungsbild“, siehe Abschnitt 10) öffnet den Dialog `#columnScalePanel` mit zwei unabhängigen Auswahlgruppen: „CNC-Programmcode“ (`columnScaleCode`, vier Werte 25/50/75/100 %) und „Arbeitsgänge“ (`columnScaleOps`, zwei Radio-Buttons „Ausgeklappt“ `value="1"`/„Eingeklappt“ `value="0"`) — sowie „Abbrechen“/„Übernehmen“.

**Bedeutung:** Die gewählten Werte legen die **Standard-Skalierung** fest, die bei jedem (Neu-)Laden eines Programms angewendet wird — für „CNC-Programmcode“ als Startwert der Einklapp-Stufe (`PROG_STAGE_PCT.indexOf(columnScalePreference.code)`), für „Arbeitsgänge“ als Ausgeklappt/Eingeklappt-Zustand. Ein Klick auf „Übernehmen“ wendet die Werte zusätzlich sofort auf ein bereits geladenes Programm an; dabei wird zwingend zuerst `setOpsWidth()` mit dem neuen Arbeitsgänge-Zustand aufgerufen und **erst danach** `applyProgCollapseStage()`, da Letzteres den verfügbaren Platz der Programmcode-Spalte anhand der aktuellen `opsWidth` berechnet.

**Persistenz:** `columnScalePreference = { code, ops }` (Standard `{ code: 100, ops: 1 }`) wird über `localStorage` gespeichert (`persistColumnScale()`) und beim Programmstart geladen; ein ungültiger/veralteter (z. B. noch prozentbasierter, aus einer älteren Einstellungsdatei stammender) `ops`-Wert wird über `normalizeColumnScale()` auf `1` (ausgeklappt) abgebildet, exakte `0` bleibt `0`. Zusätzlich Teil des JSON-Exports/-Imports (5.9).

## 8. Wiedergabe-Steuerung

Transportleiste (volle Fensterbreite, direkt unter der Symbolleiste, siehe Abschnitt 10) mit Play/Pause (ein Knopf, Symbol wechselt zwischen ▶/❚❚), Stopp (springt zurück auf Position 0, im Gegensatz zu Pause), Geschwindigkeitswahl **1x/5x/10x/20x** sowie Scrubber-Schieberegler und Positionsanzeige („n / gesamt“). Wiedergabe-Takt: alle ~130ms ein Tick (`requestAnimationFrame`-basiert); die Geschwindigkeitsstufe multipliziert dabei die Anzahl der pro Tick übersprungenen Punkte (nicht die Taktdauer) — die Wiedergabe bleibt dadurch gleichmäßig flüssig statt ruckartiger bei höherer Geschwindigkeit.

Zusätzliche Navigationswege, die alle auf denselben zentralen Positions-/Anzeige-Update-Pfad (`updateReadoutAndHighlight()` — aktualisiert Readout, aktive Code-Zeile **und** die aktuell hervorgehobene Arbeitsgang-Zeile in der Ops-Spalte, siehe 7.2) münden:
- Pfeiltasten (← / →) bewegen die Position um einen Punkt.
- „Sprung in Zeile:“-Eingabefeld (Transportleiste) springt auf den nächsten simulierten Punkt ab der eingegebenen Quelltextzeile (bewegt nur die Wiedergabeposition, scrollt die Code-Liste nicht automatisch mit).
- Klick auf eine Zeile in der Arbeitsgänge-Spalte (siehe 7.2) — dies ist der einzige Navigationsweg, der zusätzlich auch die Code-Liste selbst scrollt (`jumpToCodeLine`, siehe 7.1).

**Leertaste als Play/Pause-Kurzbefehl:** Ein `keydown`-Listener auf `window` reagiert auf `e.code === 'Space'` bzw. `e.key === ' '` und ruft `btnPlay.click()` auf — dieselbe Aktion wie ein Mausklick auf den Play/Pause-Knopf. Zwei Schutzmaßnahmen: Der Kurzbefehl greift nur, wenn tatsächlich ein Programm geladen ist (`points.length`), und er greift NICHT, während der Eingabefokus auf einem Formularelement liegt (`document.activeElement.tagName` ∈ {INPUT, TEXTAREA, BUTTON, SELECT} oder `isContentEditable`) — sonst würde die Leertaste in einem fokussierten Textfeld eines der Editoren (Abschnitt 11) versehentlich die Wiedergabe umschalten statt ein Leerzeichen einzufügen.


## 9. 3D-Viewer

### 9.1 Bewegungs-Extraktion

Generischer ISO-Achsen-Parser, unabhängig vom erkannten Maschinentyp. Zwei unterstützte Achswort-Notationen, in dieser Reihenfolge geprüft:

```js
const AXIS_RE = /(?<![A-Za-z])(?:([XYZABC])\d{0,2}\s*=\s*(-?\d+(?:\.\d+)?)|([XYZABC])\s*(-?\d+(?:\.\d+)?))/g;
```

1. **„=“-Notation mit optionaler Kanalnummer:** direkt (ohne Leerzeichen) 0–2 Ziffern (optionale Kanalnummer, wird verworfen) und danach ein „=“, erst danach der eigentliche Zahlenwert — z. B. „B1=-90.01“, „C1=-12.49“, „X=780.3“ (auch ganz ohne Kanalnummer).
2. **Klassische Notation** (Fallback, falls kein „=“ gefunden): Achsbuchstabe, optional Leerraum, Zahl direkt danach — „X780.27“, „B-90“.

Das vorangestellte `(?<![A-Za-z])` (negativer Lookbehind) verhindert, dass ein Achsbuchstabe erkannt wird, dem unmittelbar ein weiterer Buchstabe vorausgeht — nötig, damit Parameterzeilen wie „TRSX=0 TRSY=0 TRSZ=0“ oder „ROTX=0 ROTY=0 ROTZ=0“ (Verschiebungs-/Rotationsversatz-Parameter, keine echten Achsbewegungen) nicht fälschlich als eigene Achsbewegung gelesen werden — ein echtes Achswort beginnt stets nach einem Nicht-Buchstaben.

**Kommentar-Erkennung auch bei der Bewegungs-Extraktion:** `extractMotion()` berechnet für jede Zeile einzeln dieselbe Kommentar-Maske wie SWITCHCASEMACHINE/CALCMACHINE (`CNCSwap.computeCommentMask()`/`isRangeUncommented()`, dieselben drei Kommentar-Arten `;`/`KM="…"`/`{…}`, siehe 5.11) und verwirft jeden AXIS_RE-Treffer, der (auch nur teilweise) in einem erkannten Kommentarbereich liegt — eine per SWITCHCASEMACHINE-Regel oder im Original bereits auskommentierte Zeile wie `;X=0 C=0 IF=FLD=14` erzeugt dadurch weder einen Bahnpunkt noch aktualisiert sie den modalen Bewegungszustand (Release 82).

Werte werden **modal** (jede Achse behält ihren letzten Wert, bis ein neuer folgt) in den Bewegungszustand fortgeschrieben. Eilgang-/Vorschub-Erkennung (`scanRapid`, `G_RE = /G0*([0-9]{1,2})(?![0-9.])/g`): `G0` → Eilgang, `G1`/`G2`/`G3` → Vorschub; für KUKA-Robotersyntax (KRC) zusätzlich `PTP` → Eilgang, `LIN`/`CIRC` → Vorschub; ohne erkennbares Wort bleibt der zuletzt bekannte Zustand (`prevRapid`) erhalten.

**Nullpunktverschiebungen (`G92`/`O`/getauschtes `TRANS`, `ORIGIN_RESET_RE`, siehe 4.4):** Eine Zeile, die dieses Muster matcht, erzeugt **keinen eigenen Bahnpunkt** (`extractMotion()`: `if (found && !isOriginReset)`) — sie legt lediglich ein neues Koordinatensystem fest, der Fräser bewegt sich dabei nicht wirklich zu den angegebenen Werten. Der modale Zustand führt dafür einen eigenen Versatz je Achse (`state.offsetX/offsetY/offsetZ`, nur X/Y/Z, da eine Nullpunktverschiebung per Definition keine Rundachsen betrifft): Auf einer Nullpunktverschiebungszeile setzt ein gefundenes X-/Y-/Z-Achswort **nur** den jeweiligen Versatz neu (modal je Achse, fehlt eine Achse in der Zeile, bleibt ihr bisheriger Versatz unverändert) und lässt `state.x/y/z` unangetastet. Jede normale Bewegungszeile aktualisiert weiterhin `state.x/y/z` mit dem programmierten (absoluten) Wert; ein erzeugter Bahnpunkt trägt `x: state.x + state.offsetX` (entsprechend y/z). `scrubToLineIdx()` findet für eine übersprungene Nullpunktverschiebungszeile automatisch den nächsten **echten** Bahnpunkt danach.

`toRender(p) = {x: p.x, y: p.z, z: p.y}` — die Render-Y-Achse (Bildschirm-„hoch“, vor Anwendung der Kamera-Basis) entspricht der CNC-Z-Achse (Werkstückhöhe), die Render-Z-Achse (Tiefe) der CNC-Y-Achse.

**Bogeninterpolation G02/G03 (R-Format):** Trägt eine Zeile ein **explizites** G02 (im Uhrzeigersinn) oder G03 (gegen den Uhrzeigersinn) — ermittelt über `scanArcDirection(line)`, „letzter G-Code auf der Zeile gewinnt“, aber bewusst **ohne modales Fortschreiben** auf Folgezeilen ohne eigenes G02/G03 (siehe unten) — UND einen erkennbaren Bogen-Parametersatz im **R-Format** (`G02/G03 X.. Y.. R<Radius>`, per `CNCParser.parseArcParams(line)`), erzeugt `extractMotion()` mehrere tessellierte Zwischenpunkte entlang des tatsächlichen Kreisbogens (`CNCParser.interpolateArc(...)`) anstelle eines einzelnen Zielpunkts. Positiver Radius wählt den „kurzen“ Bogen (Schwenkwinkel ≤180°), negativer Radius den „langen“ Bogen (>180°). Ein Vollkreis (Start=Ziel) ist im R-Format nicht darstellbar (mehrdeutig) — `interpolateArc()` liefert dann `null` und die Bahn fällt auf eine gerade Linie zurück. **I/J-Format wird bewusst nicht unterstützt** — `parseArcParams()` erkennt ausschließlich `R`; eine reine I/J-Bogenzeile (ohne R) wird als gerade Linie vom vorherigen zum programmierten Zielpunkt dargestellt.

Bewusste Einschränkung: Es wird ausschließlich die XY-Ebene (G17) unterstützt — in der Holzbearbeitung der ganz überwiegende Regelfall. Ein eventuell vorhandenes „K“ (Z-Mittelpunktversatz für XZ-/YZ-Ebenen-Bögen unter G18/G19) wird von `parseArcParams()` nicht ausgewertet. Z selbst wird linear zwischen Start- und Endpunkt über die Bogenpunkte interpoliert — das erlaubt auch helixförmige Bögen (Z ändert sich während des Bogens) ohne Mehraufwand.

Ein G02/G03 muss auf **jeder** Bogenzeile explizit erneut stehen, damit ein „R“-Wert auf einer späteren, eigentlich unbeteiligten Zeile (z. B. die Rückzugsebene `R..` eines Bohrzyklus wie `G81`) nicht fälschlich als fortgesetzter Bogen-Radius fehlinterpretiert wird.

Segmentanzahl der Tessellierung (`tessellateArc()`, gemeinsam mit dem DXF-Bogenparser über `computeSegmentCount()` verwendet, siehe 9.8) richtet sich nach der Bogenlänge (ca. 10mm je Segment), begrenzt auf 4–180 Segmente je Bogen. Bei geometrisch nicht auflösbaren/unplausiblen Eingaben (z. B. R-Format mit zu kleinem Radius für den Punktabstand, unbekannter Parametersatz, ungültige Richtung) liefert `interpolateArc()` `null` — `extractMotion()` fällt dann auf das bisherige Verhalten (gerade Linie) zurück.

Die Wiedergabe (`stepPlayback()`) rückt pro Tick um eine feste ANZAHL PUNKTE weiter (`scrubIdx += playSpeed`), nicht um eine feste Strecke — ein Geradenstück (G0/G1) besteht unabhängig von seiner Länge immer aus genau 2 Punkten (Start/Ziel) und wird dadurch praktisch instantan durchlaufen, während ein tessellierter Bogen viele kurze Segmente und entsprechend viele Punkte erzeugt.

### 9.2 Kamera

Sphärische Kamera um einen Zielpunkt (`cam = {theta, phi, dist, minEyeDist, maxDist, target, roll}`):

```
cameraPosition():
  eyeDist = max(dist, minEyeDist)
  x = target.x + eyeDist·cos(phi)·sin(theta)
  y = target.y + eyeDist·sin(phi)
  z = target.z + eyeDist·cos(phi)·cos(theta)
```

`dist` ist der vom Nutzer per Mausrad gesteuerte Zoom-Wert, geklemmt auf `[30, maxDist]`. `minEyeDist` ist die Mindest-Augdistanz, mit der die Kamera tatsächlich positioniert wird (`frameCamera()` setzt sie auf `max(120, halbeBoundingBoxDiagonale + 60)` des aktuell angezeigten Programms/Arbeitsgangs/DXF, ohne Geometrie 220) — sie verhindert, dass die Kamera beim Heranzoomen physisch näher an den Zielpunkt heranfährt, als der nächstgelegene Bahnpunkt selbst entfernt ist (was sonst zu fälschlich als „hinter der Kamera“ ausgeblendeten, nahen Punkten führen würde). Die Blickrichtung (und damit `right`/`up`/`fwd`, siehe `buildBasis()`) hängt nur von `theta`/`phi` ab, nicht von der tatsächlich verwendeten Distanz.

**`maxDist` — Bugfix Release 87 (Sebastian: „Bauteil ca. 10000mm groß machen, Home drücken, dann Zoomen -> zoomt ran, man kommt nicht wieder weiter weg"):** Bis Release 86 war die obere Mausrad-Zoom-Grenze im wheel-Handler fest auf 4000 Welteinheiten codiert. Bei einem großen Bauteil setzt `frameCamera()` (u. a. ausgelöst durch den „Home“-Knopf bzw. seit Release 86 automatisch nach Skalieren/Drehen/Spiegeln, siehe 9.8c) `dist` aber proportional zur Bounding-Box-Diagonale — bei z. B. 10000mm deutlich über 4000. Der erste Mausrad-Scroll (in JEDE Richtung) kappte `dist` dadurch abrupt auf 4000, spürbar als ungewollter Ranzoom-Sprung, und von da an verhinderte dieselbe feste Obergrenze jedes weitere Herauszoomen über 4000 hinaus, obwohl das Bauteil einen größeren Abstand braucht, um komplett sichtbar zu sein. Seit Release 87 berechnet `frameCamera()` `maxDist` dynamisch mit `max(4000, dist·8)` (4000 bleibt Untergrenze, damit sich an normal großen Bauteilen nichts ändert) und der wheel-Handler klemmt `dist` gegen `cam.maxDist` statt gegen eine feste 4000. Der über „+Werkzeuge (T)“ gesicherte/wiederhergestellte Kamerazustand (siehe Ende dieses Abschnitts) sichert seither ebenfalls `maxDist` mit, damit ein zwischenzeitlich weiter herausgezoomter Zustand dort nicht verlorengeht.

`buildBasis(camPos)` — eine einzige, durchgehende Formel ohne Sonderfall-Korrektur:

```
fwd   = normalize(target - camPos)
right = normalize(cross(fwd, {x:0,y:1,z:0}))   // Fallback {1,0,0}, falls entartet (|right| < 1e-6)
up    = cross(right, fwd)
cam.roll rotiert right/up anschließend um fwd (Rotation um die eigene Blickachse, siehe Gizmo unten)
```

Diese Formel ist über den gesamten erlaubten `phi`-Bereich (±1,45 rad ≈ ±83°, der exakte Pol bei 90° wird durch das Clamping nie erreicht) für jeden Drehwinkel `theta` stetig. „↑ Oben“ zeigt die Kamera-Position, die von Haus aus „X rechts, Y oben“ liefert; „↓ Unten“ die andere, physisch oberhalb liegende Position, die „Y unten“ liefern würde.

`project(p, camPos, basis, w, h)`: **orthografische (parallele) Projektion**, bewusst nicht perspektivisch — zueinander parallele Fräsbahnen bleiben auf dem Bildschirm exakt parallel und gleich groß, unabhängig von ihrer Entfernung zur Kamera entlang der Blickachse. `cz = dot(p-camPos, fwd)` dient ausschließlich dazu, Punkte hinter der Kamera-Bildebene auszublenden (`cz ≤ 0.01`) — die Bildschirmgröße selbst hängt nicht von `cz` ab. `f = 1/tan(22.5°)` ist ein fester Skalierungsfaktor: `scale = (h/2)·f/dist` (konstant für die gesamte Szene, abhängig vom Zoom-Wert `dist`, nicht von `minEyeDist`).

`panCamera(dx, dy)` verschiebt `cam.target` in Bildschirmrichtung proportional zur Mausbewegung: `worldPerPixel = dist / ((h/2)·f)`, Verschiebung entlang `basis.right`/`basis.up`.

**Ansichts-Presets** (`VIEW_PRESETS`):

| Preset | theta | phi |
|---|---|---|
| Oben | 0 | −1.45 |
| Unten | 0 | 1.45 |
| Links | −π/2 | 0 |
| Rechts | π/2 | 0 |
| Vorn | π | 0 |
| Hinten | 0 | 0 |

„Vorn“/„Hinten“ liegen auf derselben horizontalen Kamera-Höhe wie „Links“/„Rechts“ (`phi=0`), aber um die Y- statt die X-Achse geschwenkt: „hinten“ ist die +Y-Seite, „vorne“ die −Y-Seite (Achs-Konvention „X+ nach rechts, Y+ nach hinten, Z+ nach oben“).

**Bedienung:** Linke Maustaste + Ziehen rotiert (`theta -= dx·0.006`, `phi` analog, geklemmt auf ±1.45), rechte Maustaste + Ziehen verschiebt (`panCamera`, natives Kontextmenü im 3D-Bereich unterdrückt), Mausrad zoomt **zum Mauszeiger** hin/weg. Neues Laden eines Programms setzt die Ansicht immer auf das Oben-Preset zurück (`cam.roll` ebenfalls auf 0); reiner Arbeitsgang-Wechsel (`setScope`) behält den Blickwinkel bei und rahmt nur Distanz/Ziel neu ein (`frameCamera()`: Zielpunkt = Bounding-Box-Mittelpunkt der sichtbaren Punkte, `dist = max(90, diag·1.5+60)`, `minEyeDist = max(120, diag/2+60)`, `maxDist = max(4000, dist·8)`, siehe Bugfix-Hinweis oben).

**Zoom zum Mauszeiger:** Da die Projektion orthografisch ist, hängt die Bildschirmposition eines Weltpunkts nur von seinem Versatz zu `cam.target` in der right/up-Bildebene ab, skaliert mit `scale`. Beim Zoomen wird `cam.target` zusätzlich zur reinen Distanzänderung so verschoben, dass der Punkt unter dem Mauszeiger an derselben Bildschirmposition bleibt:

```
shift = (distAlt − distNeu) / ((h/2)·f)
tr = (mouseX − w/2) · shift   // entlang right
tu = (h/2 − mouseY) · shift   // entlang up
```

Bei Zoom exakt im Bildschirmmittelpunkt ergibt sich `tr = tu = 0` (keine Zielpunkt-Verschiebung); am Zoom-Anschlag (`dist` bereits bei 30 oder `maxDist`) findet ebenfalls keine Verschiebung statt.

**„Home“-Knopf** (frei im 3D-Bereich platziert, links neben dem Ansichts-Gizmo, siehe unten): setzt `theta`/`phi`/`roll` fest auf das Oben-Preset und ruft anschließend `frameCamera()` erneut auf — stellt damit die Ansicht von oben her UND setzt Zoom/Zielpunkt auf das aktuell angezeigte Programm bzw. den aktuell aktiven Arbeitsgang-Scope zurück, **ohne** den 3D-Scope selbst zu verändern (anders als „Gesamtes Programm“, das zusätzlich den Scope auf `{type:'all'}` umschaltet).

**Pos1-Taste (Release 92, Sebastian: „Pos1 drücken ist wie der Homebutton im 3D und setzt den Fokus auf Ansicht von Oben/Zentriert"):** globaler `keydown`-Listener auf `window`, ruft bei `e.key === 'Home'` exakt dieselbe, unveränderte `goHomeView()`-Funktion auf wie ein Klick auf `#btnHomeView` selbst — beide Auslöser sind dadurch garantiert immer exakt gleichwertig. Da `#btnHomeView` kein `disabled`-Attribut besitzt (`goHomeView()`/`frameCamera()` sind auch ganz ohne geladenes Programm/DXF sicher aufrufbar, siehe oben) ist auch die Pos1-Taste ohne jede Vorbedingung nutzbar. Wie bei den bestehenden Tastenkürzeln (Leertaste, Pfeiltasten) wird die Taste ignoriert, solange ein echtes Texteingabefeld (Eingabe/Textarea/Auswahlliste/`contenteditable`) fokussiert ist, damit „Pos1“ dort seine gewohnte Bedeutung (an den Zeilenanfang springen) behält.

### 9.2a Rhombenkuboktaeder-Ansichts-Gizmo

Ein anklickbarer 3D-Körper (Rhombenkuboktaeder) oben rechts im 3D-Bereich (`GIZMO_MARGIN = 18px` Abstand zum Rand, `GIZMO_RADIUS = 34px`) ersetzt textbasierte Ansichts-Knöpfe vollständig als Bedienelement für Kamera-Presets und freies Drehen. `drawViewGizmo()` zeichnet ihn nur, solange tatsächlich Bahnpunkte eines Programms ODER eine geladene und sichtbare DXF-Unterlage vorhanden sind.

**Geometrie:** Rein mathematisch erzeugt (`GIZMO_FACES`, `app.js`) — kein importiertes 3D-Modell. Die 24 Eckpunkte sind alle Permutationen von `(±1, ±1, ±(1+√2))`; daraus ergeben sich 26 Flächen (6 Quadrate an den Hauptachsen, 12 Rechtecke an den Kanten, 8 Dreiecke an den Ecken) und 48 Kanten. Nur die sechs Hauptflächen (an den Achsen ±X/±Y/±Z) tragen eine Beschriftung: Normalenvektor `(1,0,0)` → „Right“ → `VIEW_PRESETS.right`, `(-1,0,0)` → „Left“, `(0,1,0)` → „Bottom“, `(0,-1,0)` → „Top“, `(0,0,1)` → „Rear“, `(0,0,-1)` → „Front“.

**Rendering:** Der Körper ist konvex und um den Ursprung zentriert und wird rein orthografisch (dieselbe Projektionslogik wie die Hauptszene) betrachtet — ein einfacher Rückseiten-Test (`dot(normal, fwd) < -ε`) genügt zur Sichtbarkeitsbestimmung. Die verwendete Basis (`buildBasis(cameraPosition())`) hängt nur von `cam.theta`/`cam.phi` ab, nicht von Zoom oder Zielpunkt — das Gizmo zeigt unabhängig vom aktuellen Zoom/Ausschnitt der Hauptszene stets die korrekte, aktuelle Blickrichtung. Jede sichtbare Fläche wird nach Beleuchtungsstärke unterschiedlich stark eingefärbt (Hauptflächen in der Akzentfarbe, übrige Flächen neutral grau).

**Klickbarkeit — alle 26 Flächen:** Jede aktuell sichtbare Fläche (nicht nur die sechs beschrifteten Hauptseiten) landet bei jedem Frame mit `{points, view, normal}` in `gizmoHitFaces` — bei Kanten/Ecken ist `view: null`, stattdessen ihre eigene `normal`. `hitTestGizmo()` (Standard-Punkt-in-Polygon-Test) liefert die getroffene Fläche und dient sowohl der Cursor-Rückmeldung (Mauszeiger wird zu „pointer“) als auch der Klick-Auswertung. Ein Klick auf eine Hauptseite (`hit.view` gesetzt) ruft `applyViewPreset(hit.view)` auf (setzt `cam.theta`/`cam.phi` auf das exakte, benannte Preset); ein Klick auf eine Kante oder Ecke berechnet die Zielperspektive dagegen direkt aus der Flächennormale: Da `cameraPosition()` die Kamera-Richtung als reine Funktion von `theta`/`phi` ausdrückt, lässt sich diese Formel für eine beliebige Normale umkehren — `phi = asin(ny)`, `theta = atan2(nx, nz)` (`viewAnglesFromNormal()`). Eine Kante (zwei betragsgleiche Normalen-Komponenten) landet dadurch bei `phi ≈ ±45°`, eine Ecke (drei betragsgleiche Komponenten) bei `phi ≈ ±35,26°`. Beide Wege münden in derselben gemeinsamen Kernfunktion `applyViewAngles(theta, phi)`.

**Ein Klick fokussiert zusätzlich neu — Bugfix Release 88 (Sebastian: „Beim Klick auf den Würfel, sollte die angeklickte Seite fokussiert werden. Aktuell kann man die Sim außerhalb des Sichtbereichs schieben und beim Klick drauf, ändert sich die Perspektive, der Fokus wird aber nicht neu gesetzt"):** Bis Release 87 änderte `applyViewAngles(theta, phi)` ausschließlich `cam.theta`/`cam.phi`/`cam.roll` — war die Ansicht zuvor per rechter Maustaste weit weggeschwenkt (`panCamera()`, siehe oben), blieb `cam.target`/`cam.dist` davon unberührt, ein Gizmo-Klick änderte dadurch nur die Blickrichtung, ohne die Geometrie wieder ins Bild zu holen. Seit Release 88 ruft `applyViewAngles()` zusätzlich `frameCamera()` auf (dieselbe Funktion, die auch `goHomeView()`/„Home" für Zielpunkt/Zoom verwendet, siehe 9.2) — ein Klick auf eine beliebige Gizmo-Fläche (Hauptseite über `applyViewPreset()`, Kante/Ecke direkt, beide münden in `applyViewAngles()`) fokussiert die aktuell sichtbare Geometrie dadurch zuverlässig neu, exakt wie „Home", nur aus der jeweils gewählten Perspektive statt zwingend von oben.

**Greif- und drehbar überall, plus Rotation um die eigene Achse (Roll):** Ein auf dem Gizmo (an beliebiger Stelle, auch mitten auf einer Hauptfläche) begonnener Zug dreht die Kamera exakt nach derselben Formel wie ein Zug außerhalb des Gizmos (`cam.theta -= dx·0.006`, `cam.phi` analog, geklemmt auf ±1.45) — ein `gizmoPointerId`-Zustand dient dabei nur noch der „echter Klick vs. Zug“-Unterscheidung (≤6px Bewegung zwischen `pointerdown`/`pointerup` → zusätzlicher Sprung auf die getroffene Voreinstellung). Ein schmaler Ring knapp außerhalb des sichtbaren Würfels (`hitTestGizmoRing()`, Abstand zwischen `GIZMO_RADIUS − 4` und `GIZMO_RADIUS + 14`) erlaubt eine reine Rotation der Kamera um ihre eigene Blickachse (`cam.roll`, über die Winkeländerung zum Gizmo-Mittelpunkt bei jeder Mausbewegung akkumuliert) — ändert nicht, wohin die Kamera schaut, nur welche Bildschirmrichtung dabei „oben“ ist. Ein Sprung auf eine feste Voreinstellung (Preset-Klick, „Home“, Laden eines neuen Programms) setzt `cam.roll` jeweils wieder auf 0 zurück. `cam.roll` ist fester Bestandteil der Bake-Cache-Parameter (siehe 9.5), damit ein reines Rollen zuverlässig einen Voll-Rebake auslöst.

**Passt sich der DXF-Ausdehnung an, auch ohne Programm:** Bodenraster (`drawGrid()`) und Achsen-Gizmo (`drawAxisGizmo()`, die drei farbigen X/Y/Z-Achsen im Ursprung) beziehen ihre Größe aus derselben kombinierten Punktmenge wie `frameCamera()` selbst (`viewPoints` plus, sofern geladen/sichtbar, die DXF-Unterlage über `dxfPointsForBounds()`), nicht nur aus den CNC-Bahnpunkten. Ohne geladenes Programm, nur mit importierter DXF, sind Raster und Achsen dadurch exakt an die tatsächlich angezeigte DXF-Ausdehnung angepasst statt auf einen festen `-100..100`-Fallback zurückzufallen.

**Bugfix Release 94 — `lastFramedBounds`-Cache, statt live bei jedem Frame neu zu rechnen (Sebastian: „Beim ausblenden aller DXF Layer (egal welcher Weg), geht der Fokus verloren und das ‚Gitter‘, was unter der DXF liegt, wird sehr klein.“):** Bis Release 93 lasen `drawGrid()`/`drawAxisGizmo()` `boundsOf(viewPoints.concat(dxfPointsForBounds()))` bei **jedem** gezeichneten Frame frisch — und `dxfPointsForBounds()` liefert bewusst ein leeres Array, sobald **alle** Layer ausgeblendet sind (siehe 9.8a), unabhängig davon, ob dieser Zustand über den Sammel-Knopf „Alle Layer ausblenden“ oder das einzelne Deaktivieren jeder Checkbox erreicht wurde. Weder `toggleAllDxfLayers()` noch die einzelne Checkbox rufen dabei `frameCamera()` auf — bewusst so, analog zum „Auge“ der Arbeitsgänge-Spalte, dessen `hiddenOpIndexes` ebenfalls nicht in die Bounding Box einfließt (siehe 9.3): Ein-/Ausblenden soll nur das Zeichnen betreffen, nicht die Kamera bewegen. Die Kamera selbst (`cam.target`/`cam.dist`) blieb dadurch beim „alle Layer aus“-Zustand unverändert auf der ursprünglichen, vollen DXF-Ausdehnung stehen — Raster/Achsen-Gizmo fielen pro Frame aber plötzlich auf den kleinen, ortsfesten Nichts-geladen-Standard (−100…100 um den Ursprung, bzw. beim Gizmo `diag=100`) zurück: sichtbar als „Fokus verloren“ (Raster sitzt an einer anderen Stelle als die Kamera hinschaut) und „Gitter wird sehr klein“ (200 Einheiten Spannweite statt der tatsächlichen, oft deutlich größeren Bauteil-/DXF-Ausdehnung, auf die die Kamera weiterhin gezoomt blieb).

**Fix:** Eine neue Zustandsvariable `lastFramedBounds` hält das Ergebnis der jeweils letzten ECHTEN `frameCamera()`-Berechnung fest — also genau der Momente, die auch tatsächlich die Kamera selbst bewegen (Laden/Löschen von Programm/DXF, Scope-Wechsel, „Home“, Spiegeln/Drehen/Skalieren, siehe 9.2/9.2a/9.8c). `frameCamera()` schreibt `lastFramedBounds` bei jedem Aufruf neu (auch `null`, wenn wirklich nichts geladen/sichtbar ist). `drawGrid()`/`drawAxisGizmo()` lesen ab sofort ausschließlich diesen Cache statt live neu zu rechnen — ein reines Ein-/Ausblenden von DXF-Layern (über beide Wege) verändert dadurch weder Kamera noch Raster/Gizmo mehr, exakt wie beim „Auge“ der Arbeitsgänge bereits der Fall. Ein bewusster Re-Frame (z. B. „Home“-Klick NACH dem Ausblenden) berücksichtigt weiterhin korrekt nur die aktuell sichtbare Geometrie, da `frameCamera()`/`dxfPointsForBounds()` selbst unverändert bleiben — nur der ungewollte Live-Nachzieheffekt von Raster/Gizmo zwischen zwei echten Reframes wurde entfernt. Verifiziert über eine dedizierte 11-Punkte-Playwright-Testsuite (Gitter pixelidentisch vor/nach Ausblenden über beide Wege, pixelidentisch nach Wieder-Einblenden, bewusste Änderung nach explizitem „Home“, korrekte Rück-Einrahmung nach erneutem „Home“ mit wieder sichtbaren Layern) sowie den vollständigen 281/281-Unit-Testlauf und alle bestehenden Regressionssuiten (unverändert grün).


### 9.3 Szenen-Aufbau je Scope

`setScope(mode)`:
- **`{type:'all'}`** (Standard, „Gesamtes Programm“): Sind Arbeitsgänge erkannt, wird jeder in seiner eigenen, vollfarbigen Bahnfarbe (`pathColor`) in Originalreihenfolge dargestellt. Ohne erkanntes Schema wird stattdessen nach Nullpunktverschiebungen eingefärbt (4.4).
- **`{type:'op', index}`**: Der gewählte Arbeitsgang wird aktiv/vollfarbig dargestellt, alle **vorherigen** Arbeitsgänge bleiben als gedimmter Kontext sichtbar (`--ink-faint`, „bereits gefertigtes Bauteil“), spätere werden ausgeblendet.

Aktive Punkte (Scrubber/Readout/Wiedergabe) sind stets nur die des fokussierten Bereichs (nicht der gedimmte Kontext); für Kamera-Framing und Raster zählen dagegen alle sichtbaren Punkte inklusive Kontext.

**Bahn wächst mit der Wiedergabe:** Jedes NICHT gedimmte Segment (`context: false`) merkt sich in `setScope()` seinen eigenen Start-Index (`seg.rangeStart`) innerhalb des gemeinsamen, fortlaufenden Wiedergabe-Arrays `points`. `drawPolyline()` zeichnet von einem solchen Segment nur die Punkte, die bis zur aktuellen Wiedergabeposition bereits erreicht sind: `revealCount = clamp(scrubIdx − seg.rangeStart + 1, 0, seg.points.length)`. Gedimmte Kontext-Segmente werden davon ausgenommen und immer vollständig gezeichnet. Kamera-Einrahmung und Raster basieren weiterhin auf der vollständigen Bounding Box aller sichtbaren Punkte (`viewPoints`), damit die Ansicht beim Zusehen nicht mitwandert/-zoomt.

Das Wachsen gilt erst **ab dem tatsächlichen Start der Wiedergabe**, nicht bereits ab dem bloßen Laden: Ein Zustand `hasStarted` (`false` direkt nach Laden/Scope-Wechsel bzw. nach „Stop“) merkt sich, ob der Benutzer die Wiedergabe bereits aktiv bewegt hat (Play, Ziehen am Scrubber, Pfeiltasten, Zeilensprung/Klick auf einen Arbeitsgang). `drawPolyline()` zeichnet ein nicht gedimmtes Segment weiterhin sofort vollständig, solange `!hasStarted` gilt, und wechselt erst danach auf die an `scrubIdx` gekoppelte Teilanzeige. So sieht man direkt nach dem Laden (und nach jedem „Stop“) sofort das komplette Programm im Überblick, und erst ein echter Start lässt die Bahn wieder schrittweise nachwachsen.

Ein per „Auge“-Knopf (siehe 7.2) ausgeblendeter Arbeitsgang (`hiddenOpIndexes`) wird beim Rendern komplett übersprungen (`render()`: `if (hiddenOpIndexes.has(seg.opIndex)) return;`) — unabhängig vom „wächst mit der Wiedergabe“-Verhalten und ohne Einfluss auf `points`/`scrubIdx`/Kamera-Framing.

Die frühere Scope-Chip-Anzeige (Text-Hinweis oberhalb des Canvas mit Titel/Punktanzahl des aktiven Arbeitsgangs) ist dauerhaft ausgeblendet — Titel/Punktanzahl werden intern weiterhin berechnet, aber nicht mehr angezeigt.

### 9.4 Messwerkzeug

Zwei Modi, exklusiv zueinander (Aktivieren des einen schaltet den anderen automatisch ab, inkl. Zurücksetzen bereits gesetzter Punkte): **Längenmessung** (`#btnMeasure`, „📏“) und **Winkelmessung** (`#btnMeasureAngle`, „∠“). Beide Auslöser existieren doppelt und gleichwertig — als reine Icon-Knöpfe in der Kopfzeile des Info-Fensters (`#infoPanelHead`, siehe 9.6) sowie als Icon-Knöpfe in der immer sichtbaren Symbolleiste (`#toolbarRow3`, direkt hinter „⛶ Gesamtes Programm“, siehe Abschnitt 10) — eine gemeinsame Zustandslogik (`toggleMeasure()`/`toggleMeasureAngle()`/`syncMeasureButtons()`) hält beide Knopfpaare stets synchron `.active`. Die Platzierung in der immer sichtbaren Symbolleiste (statt nur in der nur bei geladenem Programm sichtbaren Transportleiste) stellt sicher, dass beide Werkzeuge auch beim reinen Betrachten einer DXF-Unterlage ohne geladenes CNC-Programm nutzbar sind.

**„∠“ deutlich größer, kalibriert auf die Höhe von „📏“ (Release 88/89):** Sebastian: „Das ∠ Symbol für die Winkelmessung bitte überall größer machen. Das ist immer noch sehr klein.“ Das Zeichen „∠“ besteht nur aus zwei dünnen Linien und wirkte dadurch bei gleicher `font-size` optisch deutlich kleiner als das benachbarte, flächig gezeichnete Lineal-Emoji „📏“ — eine frühere Vergrößerung beider Mess-Knöpfe (72. Runde, 13px → 18px im Info-Fenster) betraf beide Symbole gleichermaßen und reichte für „∠“ allein nicht aus. Ein erster Versuch (Release 88) mit einer einzigen `font-size: 26px` für beide Vorkommen war in der Symbolleiste (dort sonst 11.5px wie jeder `.toolbar-btn`) deutlich zu groß — Sebastian: „So groß soll das nicht sein, eher so hoch wie das Lineal ist ;)“. Korrektur (Release 89): Statt einer pauschalen Größe wurde die tatsächlich gerenderte Zeichenhöhe („Ink-Height“, per Canvas-`fillText()` + Alpha-Bounding-Box gemessen) von „∠“ auf die von „📏“ **im jeweils selben Kontext** kalibriert: Symbolleiste `#btnMeasureAngleTransport { font-size: 19px; }` (📏 hat dort bei 11.5px eine Ink-Height von 13px, „∠“ bei 19px ebenfalls 13px) und Info-Fenster-Kopfzeile `#btnMeasureAngle { font-size: 27px; }` (📏 hat dort bei 18px eine Ink-Height von 19px, „∠“ bei 27px ebenfalls 19px) — zwei getrennte, ID-spezifische CSS-Regeln in `template_top.html` statt einer gemeinsamen. „📏“ und alle übrigen `.toolbar-btn`/`.info-panel-btn`-Knöpfe bleiben unverändert. Die feste 34×34px-Klickfläche von `#btnMeasureAngle` bleibt dabei ausreichend groß.

**Gleich große Knopf-Rahmen in der Symbolleiste (Release 90):** Sebastian: „Der Rahmen um den Winkel bei ‚Gesamtes Programm‘ ist größer als das der Längenmessung. Das umschließende Viereck soll gleich groß sein.“ Die obige Ink-Height-Kalibrierung (Release 89) hatte zwar die Zeichen selbst optisch angeglichen, aber einen Nebeneffekt übersehen: Anders als `.info-panel-btn` (feste 34×34px-Klickfläche für beide Mess-Knöpfe, siehe oben) hat `.toolbar-btn` **keine** feste Breite/Höhe — der Knopf-Rahmen wächst dort mit seinem Inhalt (`padding: 5px 10px` plus der natürlichen Zeilenhöhe der jeweiligen `font-size`). Die eigene, größere `font-size` von `#btnMeasureAngleTransport` (19px gegenüber 11.5px bei `#btnMeasureTransport`) ließ dadurch dessen Rahmen selbst spürbar größer werden (gemessen 39,05×34px gegen 36,36×26px bei „📏“), obwohl die Zeichenhöhe selbst bereits angeglichen war. Behoben nach demselben Muster wie `.info-panel-btn`: `#btnMeasureTransport, #btnMeasureAngleTransport { width: 36px; height: 26px; padding: 0; display: flex; align-items: center; justify-content: center; line-height: 1; }` — eine feste, für beide Knöpfe identische Klickfläche (36×26px, der bisherigen natürlichen Größe von „📏“ entsprechend, damit sich an dessen Optik nichts ändert) mit Flexbox-Zentrierung statt der sonst variablen Innenabstände. Beide Rahmen sind seither pixelgenau gleich groß (je 36×26px); die Info-Fenster-Variante war davon nie betroffen, da `.info-panel-btn` diese feste Klickfläche bereits seit der 72. Runde besitzt.

**Fangpunkt-Erkennung (`hitTest3D()`):** Ein Klick im 3D-Bereich sucht in dieser Reihenfolge:
1. den nächstgelegenen sichtbaren Bahn- oder DXF-Punkt im Bildschirmabstand `MEASURE_PICK_PX = 14`;
2. liegt keiner nah genug, den nächstgelegenen echten **Schnittpunkt zweier Linien** (Kontur- und/oder DXF-Linien) — `screenSegmentIntersect()` (klassische Zwei-Geraden-Schnittpunktformel über die Parameter t/u) prüft dafür alle Linienpaare, deren Bildschirm-Lot-Fußpunkt zum Klick höchstens 14px entfernt liegt, und akzeptiert nur einen Schnittpunkt, der tatsächlich auf BEIDEN Strecken selbst liegt (nicht nur auf ihrer gedachten Verlängerung); die 3D-Koordinate wird als Mittelwert der beiden getrennt interpolierten Punkte berechnet (bei ebenen 2D-Konturen liegen beide ohnehin exakt aufeinander);
3. liegt auch keiner nah genug, den nächsten Punkt auf einer nahen Bahn-Linie (Lot-Fußpunkt-Berechnung).

Ein Klick zählt nur dann als Auswahl, wenn sich die Maus zwischen Drücken und Loslassen um höchstens 6px bewegt hat (sonst wurde stattdessen die Kamera bedient). Beide Suchstufen beziehen sowohl `viewPoints`/`segments` (CNC-Bahn) als auch die (um `dxfOffset` und Spiegelung/Drehung transformierte) DXF-Geometrie ein, sofern ein DXF geladen UND sichtbar ist (siehe 9.8) — eine Messung funktioniert dadurch auch ganz ohne geladenes CNC-Programm sowie gemischt zwischen einem DXF-Punkt und einem Punkt der Fräskontur.

**Hover-Fangpunkt-Hervorhebung:** Solange das Messwerkzeug aktiv ist und gerade nicht per Drag die Kamera bedient wird, läuft bei jeder Mausbewegung über dem 3D-Bereich derselbe `hitTest3D()` wie beim tatsächlichen Klick, aber ohne bereits einen Mess-Punkt zu setzen — das Ergebnis wird als hohler, halbtransparent gefüllter Ring (Radius 9px) mit kleinem Mittelpunkt gezeichnet, deutlich unterscheidbar von den soliden, gefüllten Kreisen (Radius 5px) für bereits gesetzte Start-/Endpunkte.

**Längenmessung:** Erster Klick setzt den Startpunkt, zweiter den Endpunkt; ein dritter Klick verwirft die bisherige Messung und beginnt mit diesem Klick als neuem ersten Punkt von vorn. Das Ergebnis (ΔX/ΔY/ΔZ, „Ebene (XY)“, „Direkt (3D)“) erscheint im Info-Fenster (siehe 9.6). Zusätzlich zur gestrichelten direkten Verbindungslinie (Luftlinie) werden bis zu drei weitere, ebenfalls gestrichelte „Treppenweg“-Teilstrecken gezeichnet, die den Weg von Start- zu Endpunkt achsenweise nachzeichnen: `start → (endX, startY, startZ)` in der Achsfarbe X, `(endX, startY, startZ) → (endX, endY, startZ)` in der Achsfarbe Y, `(endX, endY, startZ) → end` in der Achsfarbe Z (dieselben CSS-Variablen `--axis-x`/`--axis-y`/`--axis-z` wie beim Achsen-Gizmo) — jede Teilstrecke nur, wenn ihre Länge tatsächlich ungleich null ist.

**Winkelmessung:** Sammelt bis zu drei Fangpunkte; der beim zweiten Klick gesetzte Punkt ist der Scheitelpunkt, an dem der Winkel zwischen den beiden Schenkeln zum ersten und dritten Punkt gemessen wird (ein vierter Klick beginnt neu). Berechnung über das Skalarprodukt der beiden Schenkelvektoren, `Winkel = acos((v1·v2)/(|v1|·|v2|))` in Grad (echte räumliche 3D-Vektoren, nicht auf eine Ebene projiziert); „Gegenwinkel“ wird als Explementärwinkel verstanden (`360° - Winkel`). Das Panel zeigt zusätzlich zu Winkel/Gegenwinkel je einmal pro Schenkel dieselben ΔX/ΔY/ΔZ/„Direkt (3D)“-Zeilen wie bei der Längenmessung, mit Präfix „Schenkel 1“/„Schenkel 2“. Ein kleiner Bildschirm-Bogen am Scheitelpunkt dient als rein optische Orientierungshilfe.

**Sichtbarkeit des Info-Fensters:** `#infoPanel` (umschließt `#readout` und `#measurePanel`) ist sichtbar, sobald entweder Bahnpunkte eines geladenen CNC-Programms existieren **oder** eine der beiden Messfunktionen aktiv ist (`updateInfoPanelVisibility()`, aufgerufen aus `setScope()` sowie aus `toggleMeasure()`/`toggleMeasureAngle()`) — dadurch bleibt das Messergebnis auch beim reinen DXF-Betrachten ohne geladenes CNC-Programm sichtbar. `#readout` selbst bleibt weiterhin ausschließlich an vorhandene Bahnpunkte gekoppelt.

**Tastenkürzel Shift+L / Shift+W (Release 92, Sebastian: „Generell soll Shift + L die Längenmessung starten und Shift + W die Winkelmessung"):** globaler `keydown`-Listener auf `window`, geprüft per `e.code` (`'KeyL'`/`'KeyW'`, unabhängig von Tastaturlayout/Feststelltaste) zusammen mit `e.shiftKey` (und ohne gleichzeitig gedrückte Strg-/Alt-/Cmd-Taste) — ruft dieselbe, unveränderte `toggleMeasure()`/`toggleMeasureAngle()`-Logik auf wie ein Klick auf `#btnMeasure`/`#btnMeasureAngle`, inklusive derselben gegenseitigen Exklusivität der beiden Werkzeuge. Da es sich um einen echten Umschalter handelt, schaltet ein zweites Drücken derselben Kombination das jeweilige Werkzeug wieder aus. Dieselbe Eingabefeld-Ausnahme wie bei der Pos1-Taste oben (kein Auslösen bei fokussiertem Eingabe-/Textfeld).

**Escape bricht eine aktive Messung ab (Release 94, Sebastian: „Wenn die Messfunktion aktiv ist, soll die mit Esc abgebrochen werden können"):** Der bereits bestehende globale Escape-Handler (siehe Abschnitt 10 — schließt Menüs/Kontextmenü und bricht offene Dialoge ab) ruft zusätzlich `toggleMeasure()` bzw. `toggleMeasureAngle()` auf, sofern das jeweilige Werkzeug gerade aktiv ist — exakt derselbe Effekt wie ein erneuter Klick auf `#btnMeasure`/`#btnMeasureAngle` oder ein zweites Drücken von Shift+L/Shift+W: Das Werkzeug wird vollständig deaktiviert (nicht nur die bereits gesetzten Fangpunkte verworfen), der Cursor kehrt zum Normalzustand zurück, und das Info-Fenster blendet sich aus, sofern kein CNC-Programm geladen ist (siehe „Sichtbarkeit des Info-Fensters" oben). Läuft unconditioniert bei jedem Escape-Druck mit, genau wie das bereits bestehende Schließen von Menüs — ein Escape-Druck kann dadurch gleichzeitig ein offenes Menü schließen UND eine laufende Messung beenden. Ein Escape-Druck ohne aktives Mess-Werkzeug bleibt wirkungslos für diesen Teil des Handlers (kein versehentliches erneutes Aktivieren). Verifiziert über eine dedizierte 10-Punkte-Playwright-Testsuite (Deaktivierung für beide Werkzeuge, kein Wiederaktivieren bei erneutem Escape ohne aktive Messung, weiterhin funktionierendes Menü-Schließen, gleichzeitiges Schließen+Abbrechen in einem einzigen Escape-Druck).

### 9.5 Rendering

Canvas-2D, `devicePixelRatio`-bewusst skaliert (`fitCanvas()`), Zeichnung per `requestAnimationFrame`-Batching (`requestRender()`/`rafPending`). Jeder Frame zeichnet in dieser Reihenfolge: Hintergrund, Bodenraster, Achsen-Gizmo, DXF-Unterlage (siehe 9.8, damit sie optisch unter der Fräskontur liegt), alle Bahn-Segmente (Eilgang gestrichelt, Vorschub durchgezogen, sofern nicht per „Eilgang anzeigen“-Kontrollkästchen ausgeblendet), der aktuelle Positionsmarker, das Werkzeug samt Halter (siehe 9.7), zuletzt das Messwerkzeug-Overlay und das Ansichts-Gizmo.

**Performance-Architektur:** Drei zusammenwirkende Techniken halten das Rendering auch bei großen Programmen/Dateien flüssig:

1. **Gebündelte Zeichenaufrufe:** `drawPolyline()` fasst aufeinanderfolgende Punkte mit identischem Zeichenstil (gedimmter Kontext vs. aktive Bahn, Eilgang vs. Vorschub) in einem einzigen zusammenhängenden Pfad (`beginPath()`/mehrere `lineTo()`/ein `stroke()`) statt für jedes Punktepaar neu anzusetzen. Dieselbe Technik verwendet `strokeGroupedByColor()` für die DXF-Unterlage: Alle Strichzüge werden nach ihrer aufgelösten Anzeigefarbe gruppiert (siehe 9.8) und je Farbe in einem Pfad gezeichnet — bei einheitlicher Farbe (kein Programm geladen bzw. nur ein Layer) genügt dadurch ein einziger `stroke()`-Aufruf für die komplette DXF-Datei statt eines Aufrufs je Entität.
2. **Offscreen-„Bake-Canvas“ mit inkrementellem Anhängen:** Zusätzlich zum sichtbaren `<canvas id="canvas3d">` gibt es eine unsichtbare, gleich große Offscreen-Canvas (`bakeCanvas`/`bakeCtx`, `ensureBakeCanvas()`), auf die Raster/Achsen/DXF/Bahn „gebacken“ werden. Bei jedem `render()`-Aufruf prüft `computeBakeParams()`/`bakeParamsEqual()`, ob sich seit dem letzten Bake etwas **Strukturelles** geändert hat (Kamera-Winkel/-Distanz/-Ziel/-Roll, Canvasgröße, „Eilgang anzeigen“, Wiedergabe gestartet/nicht gestartet, Sichtbarkeit einzelner Arbeitsgänge, Hintergrundfarbe, DXF-Geometrie/-Verschiebung/-Sichtbarkeit/-Spiegelung/-Drehung/-Bauteilstärke/-Layer-Sichtbarkeit, `segments`-Referenz nach Programmwechsel). Ist das der Fall (oder wurde rückwärts gescrubbt), wird einmal komplett neu gebacken; hat sich dagegen nur die Wiedergabeposition vorwärts bewegt, wird pro Bahn-Segment lediglich der neu hinzugekommene Punktebereich inkrementell angehängt. Die fertige Bake-Canvas wird per `drawImage()` auf die sichtbare Canvas kopiert; Positionsmarker, Werkzeug und Messwerkzeug werden dagegen **jeden Frame frisch** direkt auf die sichtbare Canvas gezeichnet, damit sie nie „einfrieren“. Rückwärts-Scrubben wird erkannt (Aufdeckungslänge eines Segments unter den zuvor gebackenen Stand gefallen) und erzwingt einen vollen Rebake, da die Bake-Canvas rein additiv ist.
3. **Strukturell feste Zeilenhöhe im Code-Panel** (siehe 7.1) — verhindert teure Flex-Neuberechnungen der gesamten Code-Liste bei jeder Zeilen-Hervorhebung, unabhängig vom Canvas-Rendering selbst.

**Leerer Viewer:** Solange kein CNC-Programm geladen ist, zeigt der 3D-Bereich nur Raster/Achsen (und ggf. eine DXF-Unterlage) ohne jeden Hinweistext. Ein Hinweistext (`#emptyViewerMsg`, „Keine ISO-Bewegungsdaten (X/Y/Z) in diesem Bereich erkannt.“) erscheint ausschließlich, wenn ein Programm zwar geladen ist, aber im aktuell gewählten Anzeigebereich (Scope, siehe 9.3) keine auswertbaren Bewegungsdaten liefert.

### 9.6 Readout & Info-Fenster

Zeigt zur aktuellen Position: Quelltext-Zeilennummer, X/Y/Z (2 Nachkommastellen, „mm“), die im Programm tatsächlich genutzten Dreh-/Kippachsen (`AXIS_META`: `C` = „Dreh C“, `A` = „Kipp A“, `B` = „Kipp B“, jeweils mit Hinweistext „…winkel (typisch, maschinenabhängig)“), bei vorhandenen Werkzeugdaten zusätzlich Werkzeug-Durchmesser (`toolRadius*2`) und -Länge bzw. bei einem Sägeaggregat die Sägeblattstärke (siehe 9.7), sowie ein Eilgang/Vorschub-Tag.

**Gemeinsames, frei verschiebbares Info-Fenster:** `#readout` und `#measurePanel` (siehe 9.4) sitzen gemeinsam in einem Container `#infoPanel`, standardmäßig unten links im 3D-Bereich (oberhalb der Copyright-Zeile). Ein dünner Trennstrich erscheint automatisch nur, wenn beide gleichzeitig sichtbar sind. Die beiden Mess-Icon-Knöpfe (`#btnMeasure`/`#btnMeasureAngle`, siehe 9.4) sitzen als eigene Kopfzeile `#infoPanelHead` oben im Panel, mit einer 34×34px-Klickfläche und zentriertem Icon (`display: flex; align-items: center; justify-content: center`), zeigen nur ihr Icon ohne Textlabel und tragen zusätzlich ein Griff-Symbol „⠿“, das die Ziehbarkeit signalisiert.

Ein Drag-Mechanismus (`initInfoPanelDrag()`, `pointerdown`/`pointermove`/`pointerup` mit `setPointerCapture()`) erlaubt es, das gesamte Panel per Ziehen an der Kopfzeile frei innerhalb des 3D-Bereichs zu verschieben (geklemmt auf den sichtbaren Bereich des Viewers) — ein Ziehvorgang beginnt bewusst nicht, wenn der Klick auf einem der beiden Mess-Icons selbst beginnt, damit diese weiterhin normal anklickbar bleiben. Die aktuelle Position ist reine Sitzungs-/Layout-Kosmetik und wird nicht persistiert. Da „Messen“/„Winkel“ Teil desselben Containers sind, bewegen sie sich bei jedem Drag automatisch mit.


### 9.7 Werkzeug-3D-Darstellung

Erkannte Werkzeugdaten (Radius/Länge, ggf. Sägeblatt-Kennzeichnung) werden als Drahtgitter-Zylinder plus Werkzeughalter an der aktuellen Wiedergabeposition dargestellt, unter Berücksichtigung von Dreh-/Kippwinkeln und Radiuskorrektur.

#### 9.7.1 Werkzeugdaten-Erkennung im Programmtext (`extractToolData()`, `parser.js`)

Läuft je erkanntem Arbeitsgang über dessen eigenen Zeilenbereich, bewusst auf den **ungetauschten** Rohzeilen (`rawProgramText`, vor Anwendung des aktiven Maschinentyps) — analog zur Struktur-Erkennung (`detectionText`, 4.5): Eine SWITCHCASEMACHINE-Regel wie „R->A“ würde sonst auch das Wort „RADIUS“ selbst verstümmeln. Erkannt werden fünf unabhängige, je einmal pro Abschnitt gesuchte Marker (erster Treffer gewinnt):

- **Radius/Länge**, zwei Notationen: Reichenbacher/Sinumerik-Werkzeugparameter `$TC_DP6[110,1]=6.000` (Radius, Parameter 6) / `$TC_DP3[110,1]=133.000` (Länge, Parameter 3); oder ein reines Schlüsselwort `RADIUS`/`LENGTH` (bzw. rückwärtskompatibel `LAENGE`/`LÄNGE`) irgendwo in der Zeile, direkt gefolgt vom Zahlenwert, umgeben von einem beliebigen Kommentarzeichen — `{ RADIUS 9.695 }` (Moroff), `; RADIUS 9.695` (SCM), `( RADIUS 9.695 )` (KRC), `KM="RADIUS 9.695"` (HOMAG), `# RADIUS 9.695` (FORMAT4).
- **`BLADE`** (`TOOL_BLADE_RE = /BLADE\s+(-?\d+(?:[.,]\d+)?)/i`, Komma oder Punkt als Dezimaltrennzeichen): Bei Sägeaggregaten beschreibt `LENGTH` nicht die tatsächliche Werkzeuglänge, sondern eine davon unabhängige Kenngröße — die für die 3D-Darstellung relevante Sägeblattdicke steht stattdessen in einer separaten `BLADE`-Zeile. Ist eine solche Zeile vorhanden (und wurde für diesen Arbeitsgang noch nie ein Werkzeugtyp per Dialog bestätigt, siehe unten), gewinnt ihr Wert als `length`, unabhängig von einer eventuell vorhandenen `LENGTH`-Zeile, und `isBlade` wird `true`.
- **`TOOLMODE`** (`/TOOLMODE\s+(EINGESPANNT|AUSGESPANNT)/i`): reiner Anzeige-/Darstellungs-Schalter, beeinflusst nicht den Zahlenwert selbst (siehe 9.7.4).
- **`TOOLTYPE`** (`/TOOLTYPE\s+(SAEGE|FRAESER)/i`): manuelle Werkzeugtyp-Festlegung, siehe unten.

**Zusammenspiel `BLADE`/`TOOLTYPE`/`LENGTH`:** Ist für einen Abschnitt eine `TOOLTYPE`-Markierung vorhanden (der „+Werkzeuge (T)“-Dialog wurde für diesen Arbeitsgang also mindestens einmal bestätigt), bestimmt sie `isBlade` (`toolType === 'saege'`) mit Vorrang vor einer eventuell noch vorhandenen rohen `BLADE`-Zeile, UND eine vorhandene `LENGTH`-Zeile gewinnt in diesem Fall immer gegen einen `BLADE`-Wert (nur ohne eigene `LENGTH`-Zeile wird auf `BLADE` zurückgegriffen). Fehlt die `TOOLTYPE`-Markierung, gilt weiterhin: eine vorhandene `BLADE`-Zeile gewinnt unbedingt gegen `LENGTH`, und `isBlade` folgt allein aus ihrer Anwesenheit. Dadurch bleibt sowohl das reine Erkennen einer realen Sägeprogramm-Datei (nur `BLADE`, nie manuell bearbeitet) als auch das nachträgliche Umschalten Säge↔Fräser per Dialog (inklusive neu eingegebener Länge) konsistent funktionsfähig.

`extractToolData()` liefert `{radius, length, isBlade, toolMode, toolType}`. `app.js` (`displayCurrentProgram()`) ruft die Funktion pro erkanntem Arbeitsgang auf und hinterlegt das Ergebnis am jeweiligen `sec` (`sec.toolRadius`/`toolLength`/`toolIsBlade`/`toolMode`/`toolType`); ohne Treffer bleiben `toolRadius`/`toolLength` `null` und es wird nichts gezeichnet.

#### 9.7.2 Zylinder-Geometrie (`drawTool(proj)`, direkt nach `drawMarker()`)

Gezeichnet wird — zusätzlich zum Positionsmarker — ein Drahtgitter-Zylinder für den Punkt an der aktuellen Wiedergabeposition (`points[scrubIdx]`), sofern für dessen Arbeitsgang Radius UND Länge gefunden wurden. Darstellung: zwei 16-Segment-Kreise (Grund-/Deckfläche) plus jede 4. Mantellinie, in `--tool-color`.

**Normales Werkzeug (Fräser):**
- **Spitze/Tiefe:** Die Zylinder-Grundfläche liegt exakt auf dem programmierten Punkt (ggf. inkl. Radiuskorrektur-Versatz, siehe unten) — die Tiefe im Programm (Z) gibt immer die Spitze des Fräsers wieder. Der Zylinder erstreckt sich von dort um `toolLength` entlang der Werkzeugachse weg vom Werkstück.
- **Durchmesser:** `toolRadius*2`, in derselben Skala (mm) wie die Bahn selbst.
- **Werkzeugachse / Dreh- & Kippwinkel:** Ausgehend von der unverkippten Achse `(0,0,1)` werden die modal mitgeführten Winkel `state.a/b/c` als Rotationen angewendet, in der Reihenfolge erst Kipp A (um X), dann Kipp B (um Y), zuletzt Dreh C (um Z). Bei nur einem gleichzeitig aktiven Winkel (der praktisch häufigste Fall) ist die Reihenfolge irrelevant.
- **Radiuskorrektur G41/G42:** `state.comp` (`'left' | 'right' | 'none'`) wird unabhängig von der Eilgang-/Vorschub-Erkennung an eigenen Mustern (`/\bG41\b/`, `/\bG42\b/`, `/\bG40\b/`) festgemacht. Der gezeichnete/simulierte Bahnpunkt bleibt immer die tatsächlich gefräste Kontur; nur die Position des Werkzeug-**Körpers** wird zusätzlich senkrecht zur Bewegungsrichtung versetzt — bei G41 („links“) nach links, bei G42 („rechts“) nach rechts, ohne Radiuskorrektur (Mittelpunktsbahn) gar nicht. Die Bewegungsrichtung ermittelt `tangentAt(idx)` aus den Nachbarpunkten desselben Segments (Mittel aus ein- und auslaufender Richtung); „links“ ist die 90°-Drehung gegen den Uhrzeigersinn.

**Reichenbacher-Kardanwinkelkorrektur:** Reichenbacher-Programme geben den Kippwinkel als „Kipp B“ (`state.b`) an — physisch handelt es sich dabei um einen kardanischen Schwenkkopf, dessen Schwenkachse fest um einen konstruktionsbedingten Schwenkwinkel `S` (per `SWIVELANGLE`, siehe 5.12) zwischen der Y- und der Z-Achse geneigt ist, statt parallel zur Y-Achse zu liegen. `reichenbacherToolAxisDirection(p)` ersetzt für diesen Fall die generische Rotationsformel durch die in 5.12 angegebene Kardanformel; `toolAxisDirection()` ruft sie anstelle der generischen Berechnung auf, sobald `reichenbacherCorrectionActive()` `true` liefert.

**Schalter „Kardan-Winkelkorrektur“:** Ein Kippschalter (`#reichenbacherSwitchWrap`/`#chkReichenbacherCorrection`) sitzt in der Kopfzeile direkt rechts neben dem Dateinamen (siehe Abschnitt 10) und ist nur sichtbar, solange der zuletzt per „Ok“ bestätigte Maschinentyp den Teilstring „reichenbacher“ enthält (`isReichenbacherTypeName()`, siehe 5.12). Wird er dabei neu sichtbar, springt er auf „an“ zurück (Standardzustand); bleibt er durchgehend sichtbar, wird ein manuell gesetzter Zustand nicht überschrieben. Ausgeschaltet zeigt er zum Vergleich die unkorrigierte generische Berechnung.

#### 9.7.3 Säge-Spezialbehandlung

Ist ein Werkzeug als Sägeblatt erkannt (`toolIsBlade`, siehe 9.7.1), ersetzt eine physikalisch motivierte Sonderformel die generische Achsenberechnung — aber **nur**, solange das Programm für diesen Punkt keinen von Null verschiedenen Dreh-/Kippwinkel liefert (`hasTiltAngles = (p.a||0)!==0 || (p.b||0)!==0 || (p.c||0)!==0`). Liegen echte A/B/C-Werte vor (echte 5-Achs-Kippung), läuft die Säge exakt wie ein normaler Fräser über die generische Berechnung (inkl. einer eventuell aktiven Reichenbacher-Korrektur).

**Ohne Dreh-/Kippwinkel — automatische, kinematisch korrekte Kippung:** Eine reale Kreissäge schneidet mit der Kante der Scheibe, die Scheibenebene steht deshalb aufrecht und senkrecht zur Vorschubrichtung — diese Kippung übernimmt in der Realität die Maschinensteuerung automatisch, auch wenn der reine 2,5D-CNC-Code selbst keine A/B/C-Werte enthält. Die Simulation rechnet sie deshalb selbst nach:

```js
const t = sawCutDirectionAt(scrubIdx);
axis = { x: -t.y, y: t.x, z: 0 };
const center = { x: p.x, y: p.y, z: p.z + p.toolRadius };
const half = p.toolLength / 2;
tip = { x: center.x - axis.x * half, ... };
top = { x: center.x + axis.x * half, ... };
```

- Die Werkzeugachse steht waagerecht, um 90° gegenüber der (X/Y-)Bewegungsrichtung gedreht — dieselbe 90°-Drehung `(-t.y, t.x)`, die auch für die G41/G42-Radiuskorrektur verwendet wird; da diese Achse in der X/Y-Ebene liegt, steht die Kreisquerschnitt-Basis automatisch senkrecht dazu und liegt in der von Bewegungsrichtung und Höhe aufgespannten, aufrechten Ebene.
- Der Mittelpunkt liegt exakt auf den unveränderten Programmkoordinaten X/Y (**ohne** einen sonst üblichen G41/G42-Seitenversatz — ein solcher Versatz um den vollen, bei Sägen typischerweise über 100mm großen Radius würde den Mittelpunkt weit von der Kontur wegschieben) und exakt `toolRadius` über der programmierten Z-Höhe, sodass der tiefste Punkt der gezeichneten Scheibe exakt auf der Kontur-Höhe liegt.
- Die Sägeblattstärke (`toolLength`) wird **mittig** um den programmierten Bahnpunkt zentriert (anders als bei einem normalen, einseitig gezeichneten Fräser) — die Bahn stellt die Mitte des gesägten Schlitzes dar.

**Schnittrichtung am Schnittstart (`sawCutDirectionAt()`, statt der generischen `tangentAt()`):** Bei einem typischen Sägeschnitt fährt ein Eilgang (G0) das Blatt seitlich der Kontur an, erst der darauffolgende Vorschub (G1) ist der eigentliche Schnitt in die tatsächliche Schnittrichtung. `tangentAt()` würde am Übergangspunkt die (oft ganz andere) Eilgang-Anfahrrichtung mit der nachfolgenden Schnittrichtung mitteln und dadurch ein diagonal „schräg stehendes“ Sägeblatt ergeben. `sawCutDirectionAt(idx)` bevorzugt deshalb gezielt die tatsächliche Vorschub-Richtung:

```js
function sawCutDirectionAt(idx) {
  const p = points[idx];
  const next = (idx < points.length - 1 && points[idx + 1].color === p.color) ? points[idx + 1] : null;
  if (next && !next.rapid) { /* 1. bevorzugt: der GLEICH folgende Vorschub-Schnitt */ }
  const prev = (idx > 0 && points[idx - 1].color === p.color) ? points[idx - 1] : null;
  if (prev && !p.rapid) { /* 2. sonst: die VORAUSGEGANGENE Vorschub-Bewegung */ }
  return tangentAt(idx); // 3. Fallback, z. B. reiner Eilgang ohne Vorschub-Nachbarn
}
```

Der unveränderte `tangentAt(scrubIdx)`-Aufruf am Anfang derselben `drawTool()`-Funktion (für die G41/G42-Versatz-Berechnung bei normalen Fräsern) bleibt davon unberührt — dort ist das Mitteln von Eilgang- und Vorschubrichtung weiterhin korrekt.

#### 9.7.4 Werkzeughalter (HSK63, Sägeblatt-Adapter, T-Längen)

**HSK63-Werkzeughalter** (`drawToolHolder(origin, axis, u, v, proj, profile)`): Ein Drahtgitter-Rotationskörper direkt oberhalb des Zylinders, aus einem festen Profil (`TOOL_HOLDER_HSK63_PROFILE`, 15 Radius/Distanz-Punkte, aus einer realen HSK63-Werkzeugaufnahme-DXF hergeleitet) — flache Stirnfläche am schmalen Ende (Ø36mm, am Zylinder befestigt), Flansch (Ø61,7mm), zwei Einstiche/Nuten, Übergang zum breiteren Ende (Ø51,7mm), insgesamt 103mm lang. Für ein erkanntes Sägeblatt (`toolIsBlade`) wird stattdessen die **verlängerte** Variante verwendet (`TOOL_HOLDER_HSK63_EXTENDED_PROFILE`): Der breiteste zylindrische Abschnitt des Standardprofils ist um `HSK63_EXTENDED_EXTRA_LENGTH = 40` mm verlängert (Gesamtlänge damit 143mm), alle danach folgenden Profilpunkte verschieben sich entsprechend.

Bei erkanntem Sägeblatt wird zusätzlich, **zwischen** Werkzeugspitze und HSK63-Halter, ein einfacher zylindrischer **Adapterkegel** gezeichnet (`BLADE_ADAPTER_PROFILE`: Ø30mm, Länge 55mm, konstanter Durchmesser statt echter Verjüngung) — derselbe generische `drawToolHolder()`-Mechanismus mit eigenem Profil-Parameter. Der HSK63-Halter setzt in diesem Fall erst am oberen Ende des Adapters an, statt direkt am Zylinderende.

Halter und Adapter übernehmen automatisch jede Dreh-/Kippwinkel-Berechnung (inklusive einer aktiven Reichenbacher-Korrektur), da sie exakt dieselben `axis`/`u`/`v`-Basisvektoren wie der Zylinder verwenden.

**„T-Längen“ — Eingespannte/Ausgespannte Länge:** `TOOLMODE` (siehe 9.7.1) steuert, wie weit die Halter-Baugruppe (HSK63-Halter, ggf. Adapterkegel) den ansonsten unveränderten Zylinder überlappt:

```js
const clampInset = (p.toolMode === 'eingespannt') ? HSK63_CLAMP_INSET /* = 24.98mm */ : 0;
const holderAttach = clampInset > 0
  ? { x: top.x - axis.x*clampInset, y: top.y - axis.y*clampInset, z: top.z - axis.z*clampInset }
  : top;
```

Bei „Ausgespannt“ (Standard, `clampInset = 0`) sitzt die Halter-Baugruppe wie bisher direkt am oberen Zylinderende; bei „Eingespannt“ rückt sie um 24,98mm (die Länge des schmalen Ansatzstücks des HSK63-Profils) über das letzte Stück des unverändert vollen Zylinders. Der Zylinder selbst hat in beiden Fällen exakt die volle eingegebene Länge.

**Unabhängige Ein-/Ausblendbarkeit:** `showTool` (Werkzeug-Zylinder, Knopf „👁 Werkzeug“) und `showToolHolder` (komplette Spannfutter-Baugruppe — HSK63-Halter samt ggf. vorhandenem Adapterkegel, Knopf „👁 Spannfutter“) sind zwei unabhängige Zustandsvariablen; `drawTool()` berechnet die gemeinsam benötigte Geometrie (Achse/Ansatzpunkte) unbedingt weiter, zeichnet aber Zylinder und Halter-Baugruppe in getrennten `if`-Blöcken — jeweils eines der beiden kann unabhängig vom anderen ausgeblendet bleiben.

#### 9.7.5 Bedienung — „+Werkzeuge (T)“-Dialog

Werkzeugleisten-Knopf „+Werkzeuge (T)“ (`#btnInsertToolData`) ist nur aktiv, solange der 3D-Scope genau auf einen einzelnen Arbeitsgang gesetzt ist (über dessen „3D“-Knopf, siehe 7.2 — ein bloßer Klick auf die Arbeitsgang-Zeile selbst genügt nicht). Öffnet `#toolDataPanel` mit vier Feldern, in dieser Reihenfolge:

1. **Werkzeugtyp** (`#toolTypeSelect`, „Fräser“/„Säge“) — vorbelegt mit „Säge“, sobald `sec.toolIsBlade` für diesen Arbeitsgang bereits gesetzt ist, sonst „Fräser“. Ein Umschalten ändert **sofort**, noch ohne „Ok“, die Beschriftung des zweiten Felds (`applyToolTypeUI()`, bei `change` UND beim Öffnen aufgerufen).
2. **Radius (R)** / **Länge (L)** bzw. **Sägeblattstärke** (Feldbeschriftung abhängig vom Werkzeugtyp) — vorbelegt mit den für diesen Arbeitsgang bereits erkannten Werten, sonst Standardwerte **8**/**200**. Komma als Dezimaltrennzeichen wird akzeptiert und beim Übernehmen zu einem Punkt normalisiert (`normalizeToolValue()`, verlustfreie Ziffernübernahme ohne Umweg über `parseFloat`/`toString`).
3. **T-Längen:** (`#toolModeSelect`, „Ausgespannte Länge“/„Eingespannte Länge“) — vorbelegt mit `sec.toolMode`, Standard „Ausgespannte Länge“.

**Einfügeposition:** Direkt unter dem gesamten erkannten Arbeitsgang-Kommentar-/Titelblock (`sec.headerEndIdx`, siehe 4.5), in `rawProgramText`, als bis zu vier eigene Kommentarzeilen `; RADIUS <Wert>` / `; LENGTH <Wert>` / `; TOOLMODE …` / `; TOOLTYPE …` — dieselbe Notation, die `extractToolData()` ohnehin erkennt. Stehen an der Einfügeposition bereits Zeilen desselben Musters (von einem vorherigen Klick), werden deren Werte aktualisiert statt eine weitere Kopie einzufügen. Der 3D-Scope bleibt nach dem Bestätigen auf demselben Arbeitsgang, und die Kamera (`cam.target`/`cam.dist`/`cam.minEyeDist`/`cam.maxDist`) wird explizit gesichert und nach dem Neuanzeigen unverändert zurückgeschrieben, damit ein zuvor manuell eingestellter Zoom nicht durch den ansonsten bei jedem Neuanzeigen ausgelösten `frameCamera()`-Aufruf verloren geht.


### 9.8 DXF-Import

Ein separater, vom CNC-Programm unabhängiger Import einer DXF-Zeichnung als Referenz-/Unterlage im 3D-Bereich (z. B. um die simulierte Fräskontur gegen eine vorgegebene Zeichnung zu prüfen, auch direkt anmessbar, siehe 9.4). Laden, Bearbeiten oder Löschen des einen (CNC-Programm bzw. DXF) wirkt sich nicht auf den Zustand des anderen aus — `dxfGeometry`/`dxfFilename`/`dxfOffset`/`dxfVisible` u. a. werden weder von `loadText()`/`clearProgram()`/`displayCurrentProgram()` noch umgekehrt `loadDxfText()`/`clearDxf()` angefasst.

**Ladewege:** Menüpunkt „DXF Importieren“ (`#btnDxfImport`, im „Datei“-Menü) bzw. Symbolleisten-Icon „📂 DXF“ (`#btnDxfImportIcon`) öffnen denselben nativen Dateiauswahl-Dialog; zusätzlich per Drag & Drop derselben fensterweiten Dropzone wie das CNC-Programm, unterschieden anhand der Dateiendung (`/\.dxf$/i`, case-insensitiv). Die gewählte Datei wird wie ein CNC-Programm über `readFileAsText()`/`decodeBuffer()` eingelesen (automatische Kodierungserkennung, siehe Abschnitt 3) und per `parseDXF()` geparst. Ein erneuter Import ersetzt eine zuvor geladene DXF-Unterlage vollständig (inkl. Zurücksetzen von Verschiebung, Spiegelung, Drehung, Bauteilstärke, Layer-Sichtbarkeit und Gesamt-Sichtbarkeit auf den Ausgangszustand).

**Unterstützte Entitätstypen (`parseDXF()`, `parser.js`):** Ein eigenständiger Parser für das ASCII-DXF-Gruppencode-Format, beschränkt auf den `ENTITIES`-Abschnitt (Geometrie in `BLOCKS` ohne `INSERT`-Platzierung wird ignoriert, Block-Referenzen `INSERT` selbst werden nicht aufgelöst):

- **LINE** — ein Strichzug mit den beiden unveränderten Endpunkten.
- **POINT** — ein Ein-Punkt-Strichzug (wird beim Zeichnen übersprungen, nichts zu verbinden).
- **CIRCLE** — als geschlossener Strichzug tesselliert; Radius ≤ 0 wird übersprungen.
- **ARC** — wie CIRCLE, nicht geschlossen; der „kurze“ Weg von Start- zu Endwinkel wird genommen, auch über den 0°/360°-Wraparound hinweg.
- **LWPOLYLINE** / **Alt-Style POLYLINE/VERTEX/SEQEND** — Vertices geradlinig verbunden, außer ein Vertex trägt einen **Bulge**-Wert (42) — dann wird das Segment als Kreisbogen tesselliert (`bulgeSegmentPoints()`, DXF-Standardformel `bulge = tan(θ/4)`; positiver Bulge wölbt nach links der Bewegungsrichtung, negativer nach rechts). Bei geschlossenen Polylinien wird zusätzlich das Schlusssegment zurück zum ersten Vertex gezeichnet. Bei der Alt-Style-Variante wird ein fehlendes `SEQEND` trotzdem beim Dateiende finalisiert, eine `VERTEX` außerhalb einer offenen `POLYLINE` wird ignoriert und nur gezählt.
- **3DFACE** — eine ebene Fläche aus bis zu vier Eckpunkten (Gruppencodes 10/20/30, 11/21/31, 12/22/32, 13/23/33), direkt in Weltkoordinaten (keine OCS-/Extrusionsrichtungs-Umrechnung nötig). Fehlt der vierte Eckpunkt oder wiederholt er exakt den dritten, wird die Fläche korrekt als Dreieck (3 statt 4 Eckpunkte) erkannt. Praktisch alle CAD-Programme exportieren echte 3D-Flächen-/Volumenmodelle als tausende bis zehntausende einzelne 3DFACE-Entitäten — Unterstützung dieses Typs macht dadurch reale 3D-Modelle (z. B. Treppenhaus-/Bauteilmodelle mit zehntausenden Flächen) grundsätzlich importierbar.
- **Nicht unterstützt** (werden erkannt, gezählt — `skippedTypes`, Tooltip der Statusanzeige — aber nicht gezeichnet): u. a. TEXT/MTEXT, HATCH, SPLINE, ELLIPSE. Extrusionsrichtung/OCS (Gruppencode 210/220/230) wird bei CIRCLE/ARC/LWPOLYLINE/POLYLINE nicht berücksichtigt — eine gedrehte, nicht in der Standard-XY-Ebene liegende Entität dieser Typen kann falsch orientiert erscheinen (3DFACE ist davon nicht betroffen, siehe oben).

Segmentanzahl der Bogen-/Bulge-Tessellierung: dieselbe `computeSegmentCount()`-Formel wie bei der G02/G03-Bogeninterpolation (9.1, 4–180 Segmente, ca. 10mm pro Segment).

### 9.8a Layer-Verwaltung

`parseDXF()` liest zusätzlich zur reinen Geometrie die Layer-Zuordnung jeder Entität (Gruppencode 8, Standardlayer `"0"`, falls nicht angegeben) sowie — aus der `TABLES`/`LAYER`-Sektion — eine Layer-Farbtabelle (siehe 9.8b), und liefert `layers` (eine numerisch-alphabetisch sortierte Liste aller tatsächlich gezeichneten Layer-Namen, `localeCompare(..., {numeric:true})`, sodass „LIST2“ vor „LIST10“ einsortiert wird) sowie `layerColors`.

**Reiter „DXF-Layer“:** Die Kopfzeile der Programmcode-Spalte (siehe 7.1b) zeigt je nach Ladezustand entweder nur „CNC-Programmcode“, nur „DXF-Layer“ oder beide als echtes, anklickbares Reiter-Paar (`updateSidebarHeader()`, Zustand `sidebarView`: `'code'` | `'dxfLayers'`):

- Weder Programm noch DXF geladen, oder nur ein Programm: ausschließlich „CNC-Programmcode“, nicht anklickbar.
- Nur ein DXF geladen: ausschließlich „DXF-Layer“, nicht anklickbar — `#tree` wird ausgeblendet, stattdessen erscheint `#dxfLayerPanel` mit der Layer-Liste.
- Beide gleichzeitig geladen: beide Reiter sichtbar und anklickbar.

**Priorität beim Laden hat immer das CNC-Programm:** Das Laden eines Programms setzt `sidebarView` immer auf `'code'`, unabhängig vom zuvor aktiven Reiter; das Laden eines DXF setzt `sidebarView` nur dann auf `'dxfLayers'`, wenn noch kein Programm geladen ist. „CNC-Programm löschen“ bei weiterhin geladenem DXF schaltet auf „DXF-Layer“ zurück.

**Einzeln aus-/einblendbar:** `#dxfLayerPanel` zeigt je Layer eine Checkbox-Zeile mit Farb-Schwatch (dieselbe Farbe, die `drawDxf()` für diesen Layer beim Zeichnen verwenden würde) und Name. Ein Zustand `dxfHiddenLayers` (Set der ausgeblendeten Layer-Namen, leer = alle sichtbar) wird beim Umschalten aktualisiert und an allen drei Stellen gefiltert, an denen die DXF-Geometrie sonst durchlaufen wird: Zeichnen, Kamera-Framing (`dxfPointsForBounds()`) und Mess-Fangpunkte (`hitTest3D()`) — ein ausgeblendeter Layer ist dadurch weder sichtbar noch anmessbar noch beeinflusst er die automatische Kamera-Rahmung. `dxfHiddenLayers` wird bei jedem neuen DXF-Import zurückgesetzt (alle Layer sichtbar), bleibt aber über einen Reiterwechsel oder ein zusätzlich geladenes CNC-Programm hinweg unverändert erhalten.

**„Alle Layer ausblenden/anzeigen“-Knopf (Release 91):** Sebastian: „Bei dem Fenster ‚DXF Layer‘, wo auch die Anzeige für die Layer ist, bitte einen Knopf einbauen, der alle Layer auf einmal deaktiviert. Beim nochmaligen draufklicken, wenn alle inaktiv, sind wieder alle aktiv. Wenn Layer einzeln angezeigt worden sind, blendet ein Klick auch wieder alle Layer ein.“ Direkt oberhalb der Checkbox-Liste sitzt ein zusätzlicher, volle Breite einnehmender Umschalt-Knopf (`#btnShowAllDxfLayers`, `.ops-showall-btn`), nach exakt demselben Muster wie der bereits bestehende „Alle AG ausblenden/anzeigen“-Knopf der Arbeitsgänge-Spalte (`#btnShowAllOps`, siehe 7.2) — dieselbe Zustandslogik (`toggleAllDxfLayers()`/`updateShowAllDxfLayersButton()`) wurde eins zu eins auf den bereits vorhandenen `dxfHiddenLayers`-Set übertragen, statt eine eigene, parallele Zustandsverwaltung einzuführen. Label/Optik richten sich ausschließlich danach, ob `dxfHiddenLayers` leer ist oder nicht:
- **Leer** (kein Layer ausgeblendet, auch ein zuvor gemischter Zustand zählt hier nicht hinein) → Label „Alle Layer ausblenden“; ein Klick blendet **alle** Layer auf einmal aus (`dxfHiddenLayers` wird mit sämtlichen Layer-Namen befüllt).
- **Nicht leer** (ob einzelne oder bereits alle Layer ausgeblendet sind) → Label „Alle Layer anzeigen“; ein Klick setzt `dxfHiddenLayers` vollständig zurück (leere Set) und blendet dadurch wieder **alle** Layer ein — auch aus einem zuvor nur teilweise/gemischt ausgeblendeten Zustand heraus (Sebastians dritte Anforderung).

Jeder Klick erhöht zusätzlich `dxfHiddenLayersVersion`, ruft `renderDxfLayerList()` (baut die komplette Checkbox-Liste inkl. neuem Anzeigezustand neu auf, siehe oben) und `requestRender()` (3D-Ansicht) auf — identisch zum bestehenden Verhalten eines einzelnen Checkbox-Klicks. Der Knopf bleibt vollständig synchron zu individuellen Checkbox-Klicks: `renderDxfLayerList()`s Checkbox-`change`-Listener ruft nach jeder Einzeländerung ebenfalls `updateShowAllDxfLayersButton()` auf, sodass das Label auch bei einer rein manuellen, gemischten Auswahl sofort korrekt auf „Alle Layer anzeigen“ wechselt, ohne dass der Knopf selbst geklickt wurde. Der Knopf ist nur sichtbar, solange tatsächlich mindestens ein Layer vorhanden ist (`layers.length > 0`), und wird bei jedem neuen DXF-Import zusammen mit `dxfHiddenLayers` auf den Ausgangszustand (alle Layer sichtbar, Label „Alle Layer ausblenden“) zurückgesetzt. Verifiziert über eine dedizierte Playwright-Testsuite (16 Prüfpunkte: Anfangszustand, alle-ausblenden, erneuter Klick bei „alle inaktiv“ → alle wieder aktiv, Einzel-Deaktivierung erzeugt automatisch „Alle Layer anzeigen“, Klick im gemischten Zustand blendet wieder alle ein, sowie eine echte Pixel-Sichtprüfung der 3D-Ansicht vor/nach dem vollständigen Ausblenden) sowie den vollständigen 281/281-Unit-Testlauf.

### 9.8b Darstellung: Farbe und Strichelung

**Strichelung** hängt ausschließlich davon ab, ob gerade ein CNC-Programm geladen ist — live geprüft bei jedem Zeichnen (`rawProgramText != null`), nicht nur einmalig beim DXF-Import festgelegt: Ist keines geladen, wird die DXF-Unterlage durchgezogen gezeichnet; ist eines geladen (unabhängig von der Reihenfolge, in der DXF und Programm geladen wurden), gestrichelt (`ctx.setLineDash([6, 4])`).

**Farbe** wird je Entität über `resolveEntityColor()` aufgelöst, in dieser Rangfolge: (1) eine direkte True-Color an der Entität (Gruppencode 420), (2) ein direkter ACI-Farbindex an der Entität (Gruppencode 62, Werte 1–255; die Sonderwerte 0/256/257 zählen nicht als eigene Farbe), (3) die Farbe des Layers, auf dem die Entität liegt (aus der `TABLES`/`LAYER`-Sektion, ebenfalls bevorzugt 420, sonst 62), (4) keine — Standardfarbe `--dxf-color`. Ein ACI-Index wird über eine intern hinterlegte 256-Farben-Tabelle (`ACI_RGB`) in RGB umgerechnet.

**Farbe wird nur angezeigt, wenn beide Bedingungen erfüllt sind** (`showLayerColors = multiLayer && !dashed`):
- **`multiLayer`** — die DXF-Datei nutzt tatsächlich **mindestens zwei** unterschiedliche Layer (`layerCount >= 2`, gezählt über die tatsächlich gezeichneten Strichzüge, nicht über alle in `TABLES`/`LAYER` definierten Einträge). Bei genau einem (oder keinem) tatsächlich genutzten Layer bleibt die DXF unabhängig von einer eventuell dennoch auflösbaren Farbe einheitlich in `--dxf-color` — eine einfache Einzel-Layer-DXF (der weit überwiegende Normalfall, da viele CAD-Programme einem neuen Layer automatisch eine Farbe zuweisen) sieht dadurch unverändert neutral aus statt unerwartet bunt zu werden.
- **`!dashed`** — es ist gerade **kein** CNC-Programm geladen. Sobald ein Programm geladen ist, zeigt die DXF-Unterlage unabhängig von der Layer-Anzahl ausschließlich die einheitliche `--dxf-color` (und gestrichelt) — Layerfarben erscheinen nur beim reinen DXF-Betrachten ohne geladenes Programm.

Die reine Layer-**Erkennung**/-Zählung (`layerCount`, `layers`, für die „DXF-Layer“-Liste, siehe 9.8a) ist von dieser Anzeige-Einschränkung unberührt — nur die tatsächliche farbliche Darstellung wird zusätzlich unterdrückt.

### 9.8c Verschieben, Spiegeln, Drehen, Skalieren, Bauteilstärke

Fünf Transformationen, jeweils über einen eigenen Werkzeugleisten-Knopf bedient (`#toolbarRow2`, siehe Abschnitt 10; alle deaktiviert ohne geladenes DXF), zentral in `dxfTransformPoint(p)` zusammengefasst und in dieser Reihenfolge angewendet: Skalierung → Spiegelung → Drehung → Anker-Korrektur → `dxfOffset`. Dieselbe Funktion wird von `drawDxf()`, `dxfPointsForBounds()` (Kamera-Framing) und `hitTest3D()` (Messfunktion) gleichermaßen verwendet, damit Zeichnung, Kamera-Framing und Fangpunkt-Ermittlung immer exakt dieselbe Geometrie sehen.

**Automatisches „Home“ nach Spiegeln/Drehen/Skalieren (Release 86):** Sebastian: „Nach dem Skalieren, Drehen, Spiegeln sollte der Fokus immer wieder neu mit ‚Home‘ gesetzt werden. So kann man direkt weiterarbeiten.“ `goHomeView()` (aus dem ursprünglichen „Home“-Knopf-Klick-Handler herausgelöst, siehe 9.2/„Home“-Knopf) setzt Blickrichtung auf das Oben-Preset zurück und ruft `frameCamera()` erneut auf; sie wird jetzt zusätzlich automatisch aufgerufen, sobald Spiegeln, Drehen oder Skalieren (inkl. des „Originalgröße“-Knopfs) tatsächlich etwas ändern — unabhängig davon, wie die Kamera zuvor per Hand gedreht/gezoomt wurde, landet man danach sofort wieder bei einer passend eingerahmten Oben-Ansicht der neuen Kontur, statt sie erst manuell wiederfinden zu müssen (insbesondere nach einer starken Skalierung oder einer Drehung kann die Kontur sonst weit außerhalb des bisherigen Bildausschnitts liegen). **Bewusst ausgenommen bleiben Verschieben und Bauteilstärke** — siehe deren jeweilige Beschreibung unten für die Begründung.

- **Verschieben** (`#btnDxfMoveIcon`, dazu gleichwertig der Menüpunkt „DXF Verschieben“ im „Bearbeiten“-Menü): Dialog `#dxfMovePanel` mit drei Feldern X/Y/Z, vorbelegt mit dem aktuellen `dxfOffset` (Ausgangswert 0/0/0). „Ok“ übernimmt und zeichnet sofort neu, **ohne** die Kamera zu verändern (kein `goHomeView()`/`frameCamera()`-Aufruf) — bewusst weiterhin unverändert, damit sich die Verschiebung visuell gegen die feststehende Kontur beurteilen lässt (anders als bei Spiegeln/Drehen/Skalieren bleibt die Kontur bei einer Verschiebung typischerweise im selben Bildausschnitt sichtbar, ein Home-Sprung würde hier eher den direkten Vorher/Nachher-Vergleich stören als helfen).
- **Spiegeln** (`#btnDxfMirrorX`/`#btnDxfMirrorY`, „⬍ DXF“/„⬌ DXF“) — reine Umschalter ohne Dialog, sofortige Wirkung, unabhängig voneinander gleichzeitig aktivierbar (beide zusammen ergeben eine Punktspiegelung am Ursprung). Wirken um den DXF-eigenen Koordinatenursprung (0,0) der Rohgeometrie, vor `dxfOffset`. Löst danach `goHomeView()` statt eines reinen `requestRender()` aus (siehe oben).
- **Drehen** (`#btnDxfRotate`, „↻ DXF“) — Dialog mit einem Winkel-Feld (°), wirkt ebenfalls um den DXF-eigenen Ursprung. „Ok“ löst danach `goHomeView()` aus (siehe oben).
- **Skalieren** (`#btnDxfScale`, „🔍 DXF“, Release 84/85/86, rechts neben „🧱 DXF“/Bauteilstärke) — Dialog `#dxfScalePanel` mit einem Multiplikator-Feld sowie (Release 85) einem informativen Hinweistext `#dxfScaleCurrentHint` und einem „Originalgröße“-Knopf. Intern hält `dxfScale` (Ausgangswert 1) weiterhin den **Gesamtfaktor relativ zur Rohgeometrie** beim Import, angewendet auf **X, Y und Z** gleichermaßen (nicht nur auf die Ebene), da eine falsch exportierte Einheit typischerweise alle drei Achsen gleichermaßen betrifft (u. a. die Z-Werte von 3DFACE-Entitäten, siehe 9.8). Die separat vom Benutzer in mm eingegebene Bauteilstärke (siehe unten) bleibt davon unberührt — sie ist eine eigenständige, von der DXF-Einheit unabhängige physische Maßangabe.
  - **Eingabe ist relativ zur aktuell sichtbaren Größe (Release 85):** In Release 84 wurde die Dialog-Eingabe direkt als neuer, absoluter Gesamtfaktor übernommen, UND das Feld beim Öffnen mit dem aktuellen Faktor vorbefüllt — dadurch sah es wie eine Fortsetzung vom aktuellen Zustand aus, tatsächlich wurde aber bei jeder Eingabe der Gesamtfaktor bezogen auf die Rohgeometrie neu gesetzt (Sebastian, Bugmeldung: eine 1000mm-Linie wurde mit 0,1 auf 100mm skaliert, eine anschließende Eingabe von 10 ergab 10000mm statt der erwarteten 1000mm). Seit Release 85 ist die Eingabe ein reiner **Multiplikator auf die aktuell sichtbare Größe**: `confirmDxfScale()` multipliziert den bestehenden `dxfScale` mit dem eingegebenen Faktor (`dxfScale *= Eingabe`), statt ihn zu ersetzen — mehrfache Skalierungen verketten sich dadurch wie erwartet (0,1 gefolgt von 10 ergibt wieder Faktor 1, also die Originalgröße). `openDxfScalePanel()` befüllt das Eingabefeld deshalb bei jedem Öffnen neutral mit „1“ statt mit dem aktuellen Gesamtfaktor, damit sich ein versehentlich erneut eingetippter Wert nicht ungewollt aufmultipliziert. Der seit dem Import angesammelte Gesamtfaktor wird stattdessen informativ im Hinweistext „Aktuelle Gesamtskalierung seit Import: ×…“ angezeigt.
  - **„Originalgröße“-Knopf (Release 85, NEU):** setzt `dxfScale` direkt und absolut auf 1 zurück, ohne dass der Kehrwert der bisher verketteten Faktoren von Hand ausgerechnet werden muss. Löst ebenfalls `goHomeView()` aus (Release 86).
  - Eine ungültige oder nicht positive Eingabe (0, negativ, leer) wird beim Bestätigen wie ein Faktor 1 behandelt, verändert den Gesamtfaktor also **nicht** (No-op) — statt ihn wie in Release 84 auf 1 zurückzusetzen —, da 0 die Geometrie auf einen Punkt kollabieren ließe und ein negativer Faktor eine versteckte zusätzliche Spiegelung wäre.
  - „Ok“ übernimmt und zeichnet sofort neu, löst dabei seit Release 86 `goHomeView()` aus (siehe oben) — bis Release 85 blieb die Kamera hier noch unverändert, exakt wie bei „Verschieben“.
- **Bauteilstärke** (`#btnDxfThickness`, „🧱 DXF“) — Dialog mit einem Stärke-Feld (mm). Da die DXF-Unterlage die Aufstandsfläche/Oberseite des Werkstücks darstellt, wird eine Stärke > 0 nach unten (kleineres Z) extrudiert: `drawDxf()` zeichnet zusätzlich zur Kontur dieselbe Kontur nochmal um die Stärke tiefer (etwas transparenter, als „Unterseite“ erkennbar) sowie dünne senkrechte Verbindungslinien an jedem Konturpunkt — ein reines Drahtgitter-Volumen. Ober- und Unterseite werden aus demselben, bereits transformierten Punkte-Array gebaut und folgen dadurch automatisch jeder Skalierung/Spiegelung/Drehung/Verschiebung (die Unterseite ist rechnerisch identisch zur Oberseite, nur um die Stärke in Z versetzt). `dxfPointsForBounds()` bezieht bei gesetzter Stärke auch die Unterseiten-Punkte mit ein. Bewusst **ohne** automatisches „Home“ (nicht Teil von Sebastians Anfrage, siehe oben) — eine reine Z-Extrusion verändert die von oben sichtbare Ausdehnung der Kontur ohnehin nicht.

**Untere linke Ecke bleibt bei (0,0):** Da Skalieren/Spiegeln/Drehen um den DXF-eigenen Ursprung wirken, könnte eine Kontur, deren untere linke Ecke ursprünglich nicht bei (0,0) liegt, danach an anderer Stelle (z. B. in negativen Koordinaten) landen. Ein Korrekturversatz `dxfAnchorCorrection = {x, y}` (`recomputeDxfAnchorCorrection()`) durchläuft dafür die komplette Rohgeometrie, wendet dieselbe Skalierungs-/Spiegel-/Dreh-Rechnung wie `dxfTransformPoint()` an und ermittelt daraus `minx`/`miny` der resultierenden (noch nicht per `dxfOffset` verschobenen) Punktwolke; `dxfAnchorCorrection = {-minx, -miny}` wird vor `dxfOffset` addiert. Die untere linke Ecke der transformierten Kontur liegt dadurch unabhängig von ihrer ursprünglichen Lage und unabhängig von der gewählten Kombination aus Skalieren/Spiegeln/Drehen immer exakt bei (0,0), genauso wie beim ersten, unveränderten Import — ein zusätzlich über „DXF verschieben“ gesetzter `dxfOffset` wirkt weiterhin als reine zusätzliche Verschiebung ab dieser 0/0-Ankerposition. Da eine gleichmäßige Skalierung (derselbe Faktor für X und Y) mit Spiegelung/Rotation um denselben Ursprung mathematisch vertauschbar ist, spielt es für das Ergebnis keine Rolle, dass die Skalierung in `dxfTransformPoint()`/`recomputeDxfAnchorCorrection()` als erster Schritt vor Spiegeln/Drehen angewendet wird.

Alle fünf Transformationswerte gehören zur jeweils geladenen DXF-Datei und werden bei jedem (Neu-)Import sowie bei „DXF löschen“ auf den Ausgangszustand zurückgesetzt (kein Spiegeln, 0°, Skalierung 1, 0mm, `dxfOffset` 0/0/0).

### 9.8d Rendering, Kamera-Framing, Performance

`drawDxf(proj)` wird in `render()` bewusst **vor** der Segment-Schleife der Fräskontur aufgerufen, sodass die DXF-Unterlage an jeder Überlappungsstelle optisch unter der Kontur liegt (der Canvas arbeitet ohne Tiefenpuffer nach dem Malprinzip). `frameCamera()` bezieht die transformierten DXF-Punkte über `dxfPointsForBounds()` in die Bounding-Box-Berechnung mit ein (zusammen mit den CNC-Bahnpunkten, sofern ein Programm geladen ist) — ein DXF-Import ganz ohne geladenes Programm zentriert/zoomt die Kamera also korrekt allein auf die DXF-Geometrie; `dxfPointsForBounds()` liefert ein leeres Array, wenn `dxfVisible === false` oder ein Layer ausgeblendet ist, damit unsichtbare Geometrie die automatische Rahmung nicht beeinflusst.

Für die Zeichenperformance großer DXF-Dateien (insbesondere reale 3D-Flächenmodelle mit zehntausenden 3DFACE-Entitäten) gilt dieselbe gebündelte Zeichenaufruf-Technik wie für die Fräsbahn — siehe 9.5, `strokeGroupedByColor()`.

### 9.8e Ein-/Ausblenden, Löschen, Persistenz

Werkzeugleisten-Knopf „👁 DXF“ (`#btnToggleDxf`) schaltet `dxfVisible` um (deaktiviert ohne geladenes DXF); `drawDxf()` zeichnet bei `dxfVisible === false` nichts. „DXF löschen“ (Menüpunkt `#btnDxfDeleteMenu` im „Bearbeiten“-Menü, gleichwertiges Symbolleisten-Icon „✕ DXF“) entfernt die importierte DXF-Unterlage vollständig; ein evtl. geladenes CNC-Programm bleibt unberührt.

Wie das CNC-Programm selbst wird eine importierte DXF-Unterlage **nicht** über `localStorage`, „Einstellungen exportieren“/„Einstellungen importieren“ oder „Simulation speichern“ gespeichert — ein Reload der Seite entfernt ein geladenes DXF vollständig, es muss danach erneut importiert werden (siehe auch Abschnitt 13).

Statusanzeige neben dem Dateinamen (`#dxfStatusGroup`, in der Kopfzeile, siehe Abschnitt 10): nur sichtbar, solange ein DXF geladen ist, zeigt ausschließlich den Dateinamen (Tooltip zusätzlich mit Entitäts-/Punktanzahl sowie ggf. nicht unterstützten/übersprungenen Entitätstypen). Ein Trennstrich (`#dxfFilenameSep`) erscheint zwischen dem Programmnamen und dem DXF-Dateinamen, aber nur, wenn tatsächlich beide gleichzeitig geladen sind.


## 10. Menüband (Topbar)

Von links nach rechts:

1. **„Datei“-Menü** (`#menuFileTrigger`, `.menu-trigger`) — öffnet `#menuFileDropdown`:
   - **„CNC-Programm“** (`#submenuFileCncProgramTrigger`/`#submenuFileCncProgramDropdown`, Untermenü-Flyout) → **Programm laden** (`#btnLoadFile`, nativer Dateiauswahl-Dialog), **Programm speichern als…** (`#btnSaveAsProgram`, deaktiviert ohne geladenes Programm)
   - **DXF Importieren** (`#btnDxfImport`, immer aktiv; importiert eine DXF-Datei als eigenständige, vom CNC-Programm unabhängige Referenz-Unterlage im 3D-Bereich, siehe 9.8)
   - **Simulation speichern** (`#btnSaveSimulation`, immer aktiv; lädt eine komplette, eigenständig lauffähige Kopie dieser Seite herunter, mit den aktuellen Maschinentyp-/Arbeitsgang-Namen-Einstellungen als neuer Standard eingebacken, aber ohne ein eventuell geladenes CNC-Programm, siehe 5.13)

2. **„Bearbeiten“-Menü** (`#menuEditTrigger`) — öffnet `#menuEditDropdown` mit drei Untermenü-Flyouts:
   - **„CNC“** (`#submenuCncTrigger`/`#submenuCncDropdown`) → **CNC-Code einfügen** (`#btnPaste`), **CNC-Programm bearbeiten** (`#btnEditProgram`, deaktiviert ohne geladenes Programm), **CNC-Programm löschen** (`#btnClearProgram`, deaktiviert ohne geladenes Programm)
   - **„DXF“** (`#submenuDxfTrigger`/`#submenuDxfDropdown`) → **DXF Verschieben** (`#btnDxfMoveMenu`, deaktiviert ohne importiertes DXF; zweiter, gleichwertiger Auslöser: Toolbar-Icon „✥ DXF“), **DXF löschen** (`#btnDxfDeleteMenu`, deaktiviert ohne importiertes DXF; zweiter, gleichwertiger Auslöser: Toolbar-Icon „✕ DXF“)
   - **„Arbeitsgänge“** (`#submenuOpsTrigger`/`#submenuOpsDropdown`) → **Arbeitsgang-Namen bearbeiten** (`#btnEditOpNames`, immer aktiv)

3. **„Einstellungen“-Menü** (`#menuViewTrigger`) — öffnet `#menuViewDropdown`:
   - **„Erscheinungsbild“** (`#submenuAppearanceTrigger`/`#submenuAppearanceDropdown`) → fasst zusammen:
     - **„Farben“** (`#submenuColorsTrigger`/`#submenuColorsDropdown`, zweite Verschachtelungsebene) → **Buttonfarbe** (`#btnMenuColor`, öffnet den Dialog aus 12.2), **Hintergrundfarbe** (`#btnEditColors`, öffnet das Farbprofil-Panel aus 12.1), **Schriftfarbe** (`#btnFontColor`, öffnet das Panel aus 12.3)
     - **Spalten** (`#btnColumnScale`, kein eigenes Flyout; öffnet den Dialog aus 7.3a)
     - **Standardwerte wiederherstellen** (`#btnResetDisplayDefaults`; setzt Hintergrundfarbe/Buttonfarbe/Schriftfarbe/Spaltenbreiten in einem Schritt zurück, mit Ja/Nein-Sicherheitsabfrage, siehe 12.3)
   - **„Maschinentypen“** (`#submenuMachineTypesTrigger`/`#submenuMachineTypesDropdown`) → **Maschinentypen bearbeiten** (`#btnEditMachineTypes`, siehe 5.7/11)
   - **Einstellungen importieren** (`#btnLoadSettings`) / **Einstellungen exportieren** (`#btnSaveSettings`) — flache, direkte Einträge ganz am Ende, in dieser Reihenfolge

4. **„Info“-Menü** (`#menuInfoTrigger`) — öffnet `#menuInfoDropdown`, ganz rechts nach „Einstellungen“. Enthält statt anklickbarer Einträge zwei reine Anzeige-Zeilen (`.menu-info-row`, `cursor: default`, kein Hover-Zustand):
   - **Version** — zeigt „Release “ + `APP_RELEASE_NUMBER` (Konstante ganz oben in `app.js`, aktuell 96)
   - **Letzte Releasezeit:** — zeigt `APP_RELEASE_TIMESTAMP` (Konstante direkt daneben, deutsche Zeit)
   - **„Versionsupdate prüfen“** (`#btnCheckForUpdate`, Release 93) — einziger echter, anklickbarer Eintrag in diesem Menü, unterhalb der beiden Anzeige-Zeilen per Trennlinie abgesetzt. Siehe 10.3 für die vollständige Funktionsbeschreibung.

   Beide Konstanten werden von Hand bei jeder ausgelieferten Version aktualisiert. `APP_RELEASE_NUMBER` speist zusätzlich den „r96“-Zusatz im Markennamen ganz rechts (siehe Punkt 9 unten) sowie den Vergleichswert für die Update-Prüfung (siehe 10.3).

5. **„Maschinentyp“-Label** (fett, Akzentfarbe, Großbuchstaben, siehe 5.7) **+ Maschinentyp-Dropdown** (`<select>`, dynamisch befüllt, „ISO“ immer zuerst) **+ Ok** (deaktiviert ohne geladenes Programm; wendet den gewählten Maschinentyp tatsächlich an, siehe 5.1) — zusammen `.machine-type-quick-group`, direkt neben dem „Info“-Menü.

6. *(vertikaler Trennstrich, `.topbar-sep`)*

7. **Dateiname-Anzeige** (`.file-status-row`, immer sichtbar — reine Status-Anzeige, kein Bestandteil eines Menüs), direkt gefolgt vom **„Kardan-Winkelkorrektur“-Schalter** (`#reichenbacherSwitchWrap`, siehe 9.7) — nur sichtbar, solange der bestätigte Maschinentyp „Reichenbacher“ ist — sowie, unabhängig davon, der **DXF-Statusgruppe** (`#dxfStatusGroup`, siehe 9.8) — nur sichtbar, solange ein DXF importiert ist, und zeigt ausschließlich den reinen Dateinamen, ohne jeden Knopf (die DXF-bezogenen Aktionen sitzen vollständig im „Bearbeiten“-Menü bzw. in der Werkzeugleiste, siehe unten). Unmittelbar vor der DXF-Statusgruppe steht zusätzlich ein eigener Trennstrich (`#dxfFilenameSep`), aber nur, wenn tatsächlich sowohl ein CNC-Programm als auch ein DXF gleichzeitig geladen sind.

8. *(Flex-Abstandshalter)*

9. **Markenname** — „CNC SimX“ (`.brand-mark`), mit dem `r`+`APP_RELEASE_NUMBER`-Zusatz (`#brandReleaseTag`, z. B. „CNC SimX r96“), ganz rechts, mit einem kleinen Marken-Icon davor (`<img class="brand-icon">`, dasselbe 32px-Hexagon-Symbol wie das Favicon). `<title>` des Browser-Tabs sowie ggf. Fenstertitel/MessageBox-Titel einer WebView2-exe sind ebenfalls auf „CNC SimX“ eingestellt.

10. **Favicon/Tab-Icon** — ein `<link rel="icon">` mit einem eingebetteten 32×32px-PNG (Base64-`data:`-URI direkt in `template_top.html`, keine externe Bilddatei nötig, passend zur einzigen, in sich geschlossenen HTML-Datei) erscheint sowohl als Browser-Tab-Icon als auch direkt neben dem Markennamen.

**Menü-Verhalten** (`.menu`/`.menu-trigger`/`.menu-dropdown` in `template_top.html`, Verdrahtung in `app.js`): Klick auf einen Menü-Trigger öffnet dessen Dropdown und schließt dabei automatisch das jeweils andere (höchstens ein Hauptmenü gleichzeitig offen, `closeAllMenus()`/`openMenu()`); ein erneuter Klick auf denselben, bereits offenen Trigger schließt ihn wieder (Umschalter). Ein Klick auf einen beliebigen Menüeintrag löst zusätzlich zu seiner eigenen Aktion das Schließen des Menüs aus. Ein Klick außerhalb eines offenen Menüs (`document`-weiter Klick-Listener, geprüft per `!e.target.closest('.menu')`) sowie die Escape-Taste (`window`-weiter `keydown`-Listener) schließen ebenfalls. `aria-expanded` am jeweiligen Trigger wird passend mitgeführt. Deaktivierte Einträge (`:disabled`, dieselbe CSS-Regel wie bei den 3D-Werkzeugleisten-Knöpfen, siehe 9.7) bleiben klar erkennbar (blasser, „verboten“-Cursor) statt wirkungslos anklickbar zu wirken.

**Untermenü-Flyout-Mechanik:** Ein `.menu-submenu` (z. B. `#submenuCnc`, `#submenuColors`) ist ein normaler `.menu-item`-Auslöser (zusätzliche Klasse `.submenu-trigger`, mit angehängtem „▸“-Pfeilglyph) mit einem direkt daran hängenden, seitlich (`left: 100%`) ausklappenden zweiten Dropdown (`.submenu-dropdown`) — optisch dieselbe `.menu-dropdown`-Bauweise (Karte, Schatten, `.menu-item`-Einträge), nur seitlich statt unterhalb positioniert. Ein Klick auf einen Flyout-Auslöser öffnet/schließt nur sein eigenes Flyout (`openSubmenu()`/`closeSubmenu()`) und schließt dabei automatisch jedes Geschwister-Flyout im selben übergeordneten Dropdown — das übergeordnete Hauptmenü selbst bleibt dabei geöffnet, da `e.stopPropagation()` verhindert, dass der Klick beim „Klick auf Menüeintrag schließt alles“-Listener des Hauptmenüs ankommt. Ein Klick auf einen tatsächlichen Aktions-Eintrag innerhalb eines Flyouts (z. B. „CNC-Code einfügen“) schließt dagegen weiterhin alles — Hauptmenü und alle offenen Flyouts. Escape sowie ein Klick außerhalb schließen ebenfalls alles.

Die gesamte Flyout-Mechanik ist rein rekursiv aufgebaut: `allSubmenus` sammelt alle `.menu-submenu`-Elemente im Dokument per `document.querySelectorAll('.menu-submenu')`, unabhängig von ihrer Verschachtelungstiefe, und jedes bekommt dieselbe, unabhängige `position: relative`-Verankerung für sein direkt angehängtes `.submenu-dropdown` (`position: absolute; left: 100%`). Die Geschwister-Schließlogik in `openSubmenu()` vergleicht dabei ausschließlich `wrapper.parentElement` (das jeweils unmittelbare Elternelement), nie die absolute Tiefe — dadurch funktioniert „ein Klick auf einen anderen Flyout-Auslöser im selben übergeordneten Dropdown schließt die Geschwister“ automatisch korrekt auf jeder Ebene, auch bei der zwei Ebenen tiefen Verschachtelung „Erscheinungsbild“ → „Farben“.

Ein früherer Hinweistext direkt unterhalb des Menübands wurde entfernt; die einstige Maschinen-/Struktur-Badge-Anzeige (`#machineBadge`) ist per `hidden`-Attribut dauerhaft ausgeblendet — sichtbar bleibt ausschließlich der Dateiname (`#filenameLabel`).

### 10.1 Symbolleisten (drei Zeilen)

`#viewerToolbar` ist der äußere, senkrecht stapelnde Rahmen (`display:flex; flex-direction:column`) um drei `.toolbar-row`-Zeilen (`#toolbarRow1`/`#toolbarRow2`/`#toolbarRow3`), jede mit `display:flex; align-items:center; gap:8px; flex-wrap:wrap`. Beide Leisten (`#viewerToolbar`, `#transport`) sitzen als eigene Zeilen zwischen dem Menüband (`.topbar`) und dem Hauptbereich (`.main`, die Zwei-Spalten-Aufteilung Programmcode/3D) und spannen über die volle Fensterbreite.

- **Zeile 1** (`#toolbarRow1`): „📂 CNC“ (`#btnLoadFileIcon`, Datei-Auswahl), „✎ CNC“ (`#btnEditProgramToolbarIcon`, öffnet `openProgramEditPanel()` — dritter, gleichwertiger Auslöser neben dem Menüpunkt `#btnEditProgram` und dem Icon-Knopf neben der Überschrift „CNC-Programmcode“, siehe 7.1), Trennstrich, „✕ CNC“ (`#btnClearProgramIcon`, ruft `clearProgram()` auf, dieselbe Funktion wie der Menüpunkt „CNC-Programm löschen“; nur das „✕“-Zeichen ist rot eingefärbt, `.dxf-delete-glyph`/`--danger`) — alle drei deaktiviert ohne geladenes Programm (außer dem Lade-Icon selbst).

- **Zeile 2** (`#toolbarRow2`): die komplette, zusammenhängende Gruppe der DXF-Symbolleisten-Icons: „📂 DXF“ (`#btnDxfImportIcon`, Laden — ruft dieselbe Datei-Auswahl wie der Menüpunkt „DXF Importieren“ auf, `loadDxfText()`; bewusst ohne eigenes `disabled`-Verhalten, ein erneuter Import ersetzt das vorherige DXF), „👁 DXF“ (`#btnToggleDxf`, Sichtbarkeit), „✥ DXF“ (`#btnDxfMoveIcon`, Verschieben), „⬍ DXF“/„⬌ DXF“ (Spiegeln an X-/Y-Achse), „↻ DXF“ (Drehen), „🧱 DXF“ (Bauteilstärke), „🔍 DXF“ (`#btnDxfScale`, Skalieren, Release 84/85/86), „✕ DXF“ (`#btnDxfDeleteIcon`, Löschen, ruft `clearDxf()` auf; nur das „✕“-Zeichen ist rot eingefärbt) — siehe 9.8 für die vollständige Funktionsbeschreibung jedes einzelnen Knopfs. Alle DXF-spezifischen Knöpfe außer dem Lade-Icon sind ohne importiertes DXF deaktiviert.

- **Zeile 3** (`#toolbarRow3`): „+Werkzeuge (T)“ (`#btnInsertToolData`), „👁 Werkzeug“ (`#btnToggleTool`, Werkzeug-Zylinder ein-/ausblenden), „👁 Spannfutter“ (`#btnToggleToolHolder`, Werkzeughalter ein-/ausblenden, siehe 9.7.4), Trennstrich, „⛶ Gesamtes Programm“ (`#btnWholeProgram`), Trennstrich, „📏 Messen“ (`#btnMeasureTransport`) + „∠ Winkel“ (`#btnMeasureAngleTransport`) — das Mess-Knopfpaar sitzt hier statt (nur) in der Transportleiste, damit die Messfunktion auch beim reinen DXF-Betrachten ohne geladenes CNC-Programm nutzbar ist (die Symbolleiste ist stets sichtbar, die Transportleiste nur bei geladenem Programm, siehe unten).

Die „Perspektive:“-Beschriftung samt der sechs Ansichts-Pfeil-Icons existiert nicht mehr als eigene Symbolleisten-Gruppe — sie wurde vollständig durch das Rhombenkuboktaeder-Ansichts-Gizmo direkt im 3D-Bereich ersetzt (siehe 9.2a). Der „⌂ Home“-Knopf (`#btnHomeView`) sitzt als freies Overlay unmittelbar links neben dem Gizmo im 3D-Bereich selbst (`position: absolute; top: 18px; right: 112px`).

Direkt darunter folgt die volle-Fensterbreite-Transportleiste (`#transport`, Play/Stopp/Scrubber/„Eilgang anzeigen“-Kontrollkästchen/Geschwindigkeit/Sprung-in-Zeile). Die DOM-Reihenfolge direkt unter `#app` lautet `.topbar` → `#viewerToolbar` → `#transport` → `.main` → `.footer-credit`.

**Einheitliche Knopfgröße je Gruppe (Release 96, Sebastian: „Alle Icons bei CNC (öffnen, bearbeiten, löschen) und DXF (öffnen, ausblenden, verschieben usw.) sollen dieselbe Größe kriegen.“ + „Die Icons +Werkzeuge (T), Werkzeugausblenden und Spannfutter ausblenden sollen auch gleich groß werden.“):** Bis Release 95 hatte `.toolbar-btn` (siehe oben) keine feste Breite — der Rahmen wuchs mit seinem Inhalt (Innenabstand + Textbreite), und da jeder Knopf ein anderes Unicode-Symbol vor demselben „CNC“-/„DXF“-Textsuffix trägt (z. B. „📂“ gegen „✎“ gegen „✕“), rendern die Symbole selbst bei gleicher Schriftgröße unterschiedlich breit — die Knöpfe derselben Gruppe wichen dadurch sichtbar in der Breite voneinander ab (gemessen z. B. 65px „📂 CNC“ gegen 60px „✎ CNC“/„✕ CNC“). Behoben nach demselben Muster wie bereits bei den Mess-Knöpfen (Release 90, siehe 9.4): eine feste, gruppenweit gemeinsame Breite (an der jeweils längsten Beschriftung der Gruppe gemessen + kleinem Rand) mit Flexbox-Zentrierung statt der sonst variablen Innenabstände. Drei getrennte Gruppen, da sich die Beschriftungslängen zwischen ihnen stark unterscheiden:

- **Zeile 1 (CNC)** — `#btnLoadFileIcon`/`#btnEditProgramToolbarIcon`/`#btnClearProgramIcon`: einheitlich 68×26px.
- **Zeile 2 (DXF)** — alle neun DXF-Symbolleisten-Icons (`#btnDxfImportIcon` … `#btnDxfDeleteIcon`): einheitlich 66×26px.
- **Zeile 3 (Werkzeuge)** — `#btnInsertToolData`/`#btnToggleTool`/`#btnToggleToolHolder`: einheitlich 118×26px (die deutlich längere Beschriftung „+Werkzeuge (T)“ bestimmt hier die gemeinsame Breite).

Die Höhe war je Gruppe bereits einheitlich 26px (natürliche `.toolbar-btn`-Zeilenhöhe bei 11.5px Schriftgröße) und wird seither zur Klarheit ebenfalls explizit festgeschrieben. Das Mess-Knopfpaar (`#btnMeasureTransport`/`#btnMeasureAngleTransport`, ebenfalls in Zeile 3) sowie „⛶ Gesamtes Programm“ (`#btnWholeProgram`) bleiben unverändert außerhalb dieser drei Gruppen — sie hatten bereits vorher (Release 90 bzw. von Natur aus) eine passende, konsistente eigene Größe bzw. wurden von dieser Anfrage nicht erfasst. Verifiziert über Playwright (`offsetWidth`/`offsetHeight` je Gruppe identisch, kein abgeschnittener/umgebrochener Beschriftungstext, Werkzeug-Sichtbarkeits-Knopf bleibt nach der Größenänderung weiterhin funktional).

### 10.2 Rechtsklick-Kontextmenü (Release 92)

Sebastian, wörtlich: „Wenn man Rechts irgendwo im Programm klickt, geht ein Kontextmenü auf, in dem folgende Funktionen angeboten: Längenmessung / Winkelmessung ————— Programm laden / Programm löschen / Alle Arbeitsgänge ausblenden ————— DXF laden / DXF löschen / DXF-Layer ausblenden / DXF verschieben.“ Auf Rückfrage zum Umfang des Rechtsklick-Bereichs („irgendwo im Programm“ – ganze App vs. nur bestimmte Bereiche): „Überall in der App“; auf Rückfrage zur Beschriftung der beiden Ausblenden-Einträge (statisch vs. dynamisch wie die gleichnamigen Knöpfe): „Dynamisch, wie bestehende Knöpfe“.

**Markup & Optik (`#appContextMenu`, `template_top.html`):** Ein einzelnes, fest positioniertes (`position: fixed`) Panel ganz am Ende des Dokuments, technisch dieselbe `.menu-item`-Bauweise wie die vier Hauptmenüs (siehe 10, Hover-/Fokus-/`disabled`-Optik identisch), aber mit eigenem `z-index: 500` (deutlich über `.menu-dropdown`/`.submenu-dropdown`, damit es auch über bereits offenen Dialogen/Panels liegt). Neun Einträge in exakt der von Sebastian vorgegebenen Reihenfolge, per `.context-menu-divider` (dünne Trennlinie) in drei Gruppen geteilt:

1. Längenmessung (`#ctxMeasureLength`) / Winkelmessung (`#ctxMeasureAngle`)
2. Programm laden (`#ctxLoadProgram`) / Programm löschen (`#ctxClearProgram`) / Alle Arbeitsgänge ausblenden (`#ctxToggleAllOps`)
3. DXF laden (`#ctxLoadDxf`) / DXF löschen (`#ctxClearDxf`) / DXF-Layer ausblenden (`#ctxToggleDxfLayers`) / DXF verschieben (`#ctxMoveDxf`)

Jeder Eintrag ruft in `app.js` dieselbe, unveränderte Funktion bzw. denselben Datei-Dialog-Auslöser wie sein bereits bestehendes Gegenstück auf: `toggleMeasure()`/`toggleMeasureAngle()` (siehe 9.4), `#fileInput`/`#dxfFileInput`-Klick (wie „Programm laden“/„DXF Importieren“ im Menüband), `clearProgram()`/`clearDxf()` (wie „CNC-Programm löschen“/„DXF löschen“), `toggleAllOps()`/`toggleAllDxfLayers()` (wie die gleichnamigen Knöpfe im Arbeitsgänge-/DXF-Layer-Fenster, siehe 7.2/9.8a) und `openDxfMovePanel()` (wie „DXF Verschieben“).

**Öffnen/Position (`openContextMenu(x, y)`):** Blendet das Panel zunächst an der Klickposition ein, liest danach `offsetWidth`/`offsetHeight` aus und rückt es bei Bedarf so weit nach links/oben, dass es nie über den sichtbaren Fensterrand hinausragt (4px Mindestabstand zum Rand). Ruft vor dem Einblenden `closeAllMenus()` (schließt ein eventuell offenes Hauptmenü/Flyout) sowie `updateContextMenuState()` auf.

**Zustand bei jedem Öffnen neu ermittelt (`updateContextMenuState()`), nicht laufend mitgeführt** — spiegelt exakt denselben Zustand wie die jeweils bestehenden Knöpfe/Menüpunkte an ihrer gewohnten Stelle:
- „Programm löschen“ deaktiviert ⟺ `#btnClearProgram` deaktiviert (kein Programm geladen).
- „Alle Arbeitsgänge ausblenden“/„…anzeigen“ deaktiviert, solange keine Arbeitsgänge existieren; Label wechselt dynamisch auf „…anzeigen“, sobald `hiddenOpIndexes` nicht leer ist (Sebastian, auf Rückfrage: „Dynamisch, wie bestehende Knöpfe“) — exakt dieselbe Abfrage wie `updateShowAllOpsButton()` (siehe 7.2), nur mit dem vollen Wort „Arbeitsgänge“ statt der Abkürzung „AG“ des kompakten Knopfs.
- „DXF löschen“/„DXF verschieben“ deaktiviert ⟺ `#btnDxfDeleteMenu`/`#btnDxfMoveMenu` deaktiviert (kein DXF importiert).
- „DXF-Layer ausblenden“/„…anzeigen“ deaktiviert, solange die aktuelle DXF-Datei keine Layer hat; Label wechselt dynamisch nach derselben `dxfHiddenLayers.size`-Logik wie `updateShowAllDxfLayersButton()` (siehe 9.8a).
- „Längenmessung“/„Winkelmessung“/„Programm laden“/„DXF laden“ sind ohne Vorbedingung immer aktiv.

Ein deaktivierter Eintrag (`:disabled`, `pointer-events: none`) löst schon aus sich heraus kein `click`-Ereignis aus — die Klick-Handler selbst brauchen deshalb keinen zusätzlichen Enable-Guard.

**Auslöser — „überall in der App“ (Sebastian, auf Rückfrage):**

- Ein globaler `document`-weiter `'contextmenu'`-Listener öffnet das Menü an der Klickposition, außer das Ziel ist ein echtes Texteingabefeld (`INPUT`/`TEXTAREA`/`SELECT`/`contenteditable`) — dort wird bewusst **kein** `preventDefault()` aufgerufen, damit das native Browser-Kontextmenü (Markieren/Kopieren/Einfügen) dort erhalten bleibt.
- Der 3D-Bereich (`#canvas3d`) hat einen eigenen `'contextmenu'`-Handler mit `e.stopPropagation()` (der globale Listener bekommt Rechtsklicks auf dem Canvas dadurch nie zu Gesicht) UND einer eigenen Klick-vs-Zieh-Unterscheidung, damit das bestehende Rechtsklick-Ziehen zum Kamera-Schwenken (`panCamera()`, siehe 9.2) weiterhin funktioniert, ohne dass danach ungewollt das Menü aufspringt.

**Klick-vs-Zieh-Erkennung im 3D-Bereich — zweistufig, wegen Chromiums Ereignisreihenfolge:** Per Playwright-Test verifiziert, feuert Chromium bei der rechten Maustaste `'contextmenu'` bereits **unmittelbar nach `'pointerdown'`** — also noch bevor `'pointerup'` überhaupt stattfindet — und zwar immer mit den Koordinaten der Drück-Position, unabhängig von einer eventuell folgenden Zieh-Bewegung. Im `'contextmenu'`-Handler selbst lässt sich eine Zieh-Bewegung deshalb noch gar nicht erkennen. Die eigentliche Entscheidung fällt daher zweistufig: `'contextmenu'` merkt nur die Position (`pendingContextMenuAt = {x, y}`) und ruft `preventDefault()`/`stopPropagation()` auf; erst im nachfolgenden `'pointerup'`-Handler, wenn die tatsächliche Loslass-Position feststeht, wird geprüft, ob sich die Maus seit `pointerdown` um höchstens 6px bewegt hat (dasselbe Kriterium wie beim Gizmo-/Messwerkzeug-Klick, siehe 9.2a/9.4) — nur dann öffnet `openContextMenu()` tatsächlich das Menü; bei größerer Bewegung (= Kamera-Schwenken per `panCamera()`) wird die gemerkte Position kommentarlos verworfen. `pendingContextMenuAt` wird zusätzlich bei `'pointercancel'`/`'pointerleave'` zurückgesetzt, damit kein verwaister Zustand einen späteren, unzusammenhängenden Rechtsklick fälschlich beeinflusst.

**Schließen:** Klick auf einen Menüeintrag selbst (schließt zusätzlich zu seiner eigenen Aktion), Klick irgendwo außerhalb des Menüs (`document`-weiter Klick-Listener, `!e.target.closest('#appContextMenu')`) sowie die Escape-Taste (dieselbe bestehende globale Escape-Behandlung wie bei den Hauptmenüs, siehe 10, um `closeContextMenu()` ergänzt).

Verifiziert über eine dedizierte 46-Punkte-Playwright-Testsuite (alle neun Menüpunkte samt Aktivierbarkeit/Label-Dynamik in allen Programm-/DXF-Zustandskombinationen, Schließverhalten, Rechtsklick überall inkl. 3D-Bereich mit expliziter Klick-vs-Zieh-Unterscheidung, Ausnahme für echte Texteingabefelder) sowie den vollständigen 281/281-Unit-Testlauf und alle bestehenden Release-87/88/91-Regressionssuiten (unverändert grün).

### 10.3 Versionsupdate prüfen (Release 93)

Sebastian hatte die Idee zunächst auf Nachfrage zurückgestellt („Lieber nicht umsetzen“), griff sie dann aber unaufgefordert wieder auf, wörtlich: „Die Idee lässt mich nicht los: Was ich brauche: der Benutzer klickt auf „Versionsupdate prüfen“ (unter Info) und das Programm schaut, ob es eine aktuellere Version gibt. Dann kommt die Frage, ob man jetzt updaten möchte und wenn ja, startet der Download der aktuell(st)en HTML. Der Benutzer muss es dann händisch an den richtigen Ort kopieren, damit könnte ich leben oder geht das auch automatisiert dann, also die Datei lädt herunter und beim nächsten Start, ist sie an dem Ort?! Die HTML kann da bei Github liegen und du kopierst die aktuellste Version immer dort hin und das Programm lädt die sich von da runter?“ Auf Rückfrage bestätigt: Sebastian hat bereits ein GitHub-Konto (Benutzername `Sebastian-CNCSimX`, Repository `CNC_SimX`, Branch `main`), und es soll **„Gleich vollautomatisch für die exe mit einbauen“** umgesetzt werden — nicht nur die manuelle Download-und-Ersetzen-Variante.

Zusätzliche, proaktiv nachgereichte Anforderung, wörtlich: „Bevor ich es vergesse: das Programm muss die Einstellungen, also ggf. geänderte Austauschdatei für Maschinentypen, Farbeinstellungen, Arbeitsgang Namen, also quasi alles was konfiguriert werden kann automatisch irgendwo bei einer Änderung schreiben, so dass optisch nach einem Update alles beim Alten ist“ — siehe dazu den eigenen Absatz „Einstellungen bleiben erhalten“ weiter unten.

**Auslöser (`#btnCheckForUpdate`, `app.js`):** Einzelner Knopf im Info-Menü (siehe 10, Punkt 4), unterhalb der beiden Anzeige-Zeilen per Trennlinie (`margin-top`/`padding-top`/`border-top`) optisch abgesetzt, technisch derselbe `.menu-item`-Knopf wie jeder andere Menüeintrag.

**Ablauf (`checkForUpdate()`):**

1. Der Knopf wird während der Prüfung deaktiviert und sein Text auf „Prüfe…“ umgestellt (verhindert Doppelklicks während eines laufenden Netzwerk-Aufrufs), am Ende (Erfolg **und** Fehler, per `.finally()`) wird beides zurückgesetzt.
2. `fetch(UPDATE_MANIFEST_URL, { cache: 'no-store' })` lädt ein kleines JSON-Manifest von Sebastians eigenem, öffentlichen GitHub-Repo — die neue Konstante `UPDATE_MANIFEST_URL` (direkt neben `APP_RELEASE_NUMBER`/`APP_RELEASE_TIMESTAMP` ganz oben in `app.js`) zeigt fest auf `https://raw.githubusercontent.com/Sebastian-CNCSimX/CNC_SimX/main/latest.json`. `cache: 'no-store'` stellt sicher, dass wirklich jedes Mal der aktuelle Stand geprüft wird, nicht eine zwischengespeicherte alte Antwort.
3. Das Manifest hat die Form `{"release": <Zahl>, "timestamp": "<deutsche Zeit>", "htmlUrl": "<Adresse der aktuellen CNC_Simulation.html>"}`. `release` wird direkt mit `APP_RELEASE_NUMBER` verglichen (`Number.isFinite`-Prüfung gegen einen kaputten/fehlenden Wert im Manifest).
4. Ist `release` nicht größer als die eigene `APP_RELEASE_NUMBER`, erscheint lediglich ein `alert()` „Sie verwenden bereits die aktuellste Version (Release N)“ — kein weiterer Netzwerkzugriff.
5. Ist eine neuere Version verfügbar, fragt ein natives `confirm()` „Eine neuere Version ist verfügbar: Release N (Zeitstempel). Jetzt aktualisieren?“ — bei „Nein“ endet der Vorgang hier, ohne dass die eigentliche HTML-Datei überhaupt abgerufen wird.
6. Bei „Ja“ wird `manifest.htmlUrl` per `fetch()` abgerufen (derselbe „raw.githubusercontent.com“-Host, keine zusätzliche Adresse einzurichten) und der komplette HTML-Text an `installUpdate()` übergeben.

**Installation (`installUpdate(htmlText, latestRelease)`) — zwei Wege, automatisch anhand der Laufzeitumgebung gewählt:**

- **`isRunningInHostWebView()`** prüft `window.chrome && window.chrome.webview && typeof window.chrome.webview.postMessage === 'function'` — genau dieses Objekt existiert ausschließlich innerhalb einer WebView2-Hülle (siehe `Bauanleitung_EXE_C-Sharp_WebView2.md`), niemals in einem gewöhnlichen Chrome-/Edge-Browser-Tab. Dieselbe Erkennung wie beim bereits bestehenden `window.__hostLoadProgramFromHost`-Hook (siehe 3.1), nur in umgekehrter Richtung.
- **Innerhalb der exe (WebView2 erkannt):** `window.chrome.webview.postMessage({ type: 'cncsim-install-update', html: htmlText })` — die neue HTML wird direkt an den C#-Host übergeben, der sie an derselben Stelle abspeichert, an der die exe die Datei ohnehin navigiert, und die Seite danach neu lädt (vollautomatisch, kein Benutzereingriff mehr nötig). Der dazugehörige `WebMessageReceived`-Handler auf der C#-Seite ist in `Bauanleitung_EXE_C-Sharp_WebView2.md`, Abschnitt „Schritt 3a“, dokumentiert. Ein kurzer `alert()` informiert währenddessen, dass die Installation läuft.
- **Im gewöhnlichen Browser-Tab (kein WebView2):** Fallback auf dasselbe Blob+`<a download>`-Muster wie bei „Simulation speichern“ (siehe 5.13) — die heruntergeladene Datei heißt bewusst exakt `CNC_Simulation.html`, damit ein Speichern am selben Ort wie bisher die alte Datei ersetzt. Ein abschließender `alert()` bittet darum, die Datei „händisch an derselben Stelle wie bisher“ abzulegen (vorhandene Datei ersetzen) und weist darauf hin, dass die Einstellungen dabei automatisch erhalten bleiben.

**Fehlerbehandlung:** Netzwerkfehler, ein nicht erreichbares Manifest (z. B. HTTP 404, noch keine Datei hochgeladen), kaputtes JSON im Manifest sowie ein fehlendes `htmlUrl`-Feld führen alle einheitlich zu einer verständlichen Fehlermeldung („Versionsprüfung nicht möglich (keine Internetverbindung oder das Update-Verzeichnis ist gerade nicht erreichbar)“ + technische Detailmeldung) statt zu einem Absturz — der Knopf wird in jedem Fall wieder aktiviert.

**Einstellungen bleiben erhalten:** Diese Garantie war eine ausdrückliche Vorbedingung Sebastians für die gesamte Funktion (siehe Zitat oben) und wurde **nicht nur angenommen, sondern gezielt geprüft**: Ein dedizierter Playwright-Test (`chromium.launchPersistentContext()`) legte eine Datei an einem festen Pfad an, setzte einen `localStorage`-Wert, überschrieb dieselbe Datei anschließend mit komplett anderem Inhalt und öffnete sie erneut — der `localStorage`-Wert blieb dabei vollständig erhalten. Grund: `localStorage` wird bei `file://`-Seiten in Chromium/WebView2 über den **Dateipfad** verankert, nicht über den Dateiinhalt. Da sowohl der WebView2-Auto-Install-Pfad als auch ein manuell ersetzter Browser-Download die Datei exakt an ihrem bisherigen Pfad/Namen belassen (nie umbenennen oder verschieben), bleiben alle sieben in 5.9 gelisteten `localStorage`-Schlüssel (Maschinentypdatei, Farbeinstellungen, Arbeitsgang-Namen usw. — exakt der Umfang von „Einstellungen exportieren“) automatisch erhalten, ganz ohne eigene Migrations-Logik.

**Was Sebastian bei jeder Release selbst pflegen muss:** In seinem GitHub-Repo (`Sebastian-CNCSimX/CNC_SimX`, Branch `main`) müssen bei jeder neuen Version zwei Dateien aktualisiert werden — die neue `CNC_Simulation.html` sowie eine `latest.json` mit passender Releasenummer/Zeitstempel/`htmlUrl`. Automatisierung von meiner Seite (etwa automatisches Hochladen ins Repo) ist nicht möglich, da kein GitHub-Werkzeug/-Zugang zur Verfügung steht — bestätigt per gezielter Prüfung der verfügbaren Werkzeuge/Konnektoren.

Verifiziert über eine dedizierte 7-Szenarien-Playwright-Testsuite (bereits aktuell/keine Aktualisierung nötig, Ablehnung durch Benutzer, Zustimmung mit Browser-Download-Fallback inkl. exaktem Dateinamen, Netzwerkfehler, kaputtes JSON, HTTP 404, simulierter WebView2-Host mit Prüfung der exakten `postMessage()`-Nutzlast) sowie den vollständigen 281/281-Unit-Testlauf und alle bestehenden Regressionssuiten (Release 87/88/91/92, unverändert grün).

## 11. Editoren

**Enter bestätigt, Escape bricht ab — global, für alle Eingabe-Dialoge (Release 87):** Sebastian: „In allen Menüs/Dialogen der Sim wo ich Dinge eingeben kann, möchte ich mit Enter rausgehen können (Bestätigung der Eingabe) oder mit Escape (Abbruch).“ Praktisch alle Eingabe-Dialoge der Anwendung — die fünf Editoren dieses Abschnitts, „Werkzeugdaten einfügen“ und „DXF verschieben“, die übrigen DXF-Dialoge „Drehen“/„Bauteilstärke“/„Skalieren“ (9.8c) sowie „Farben einstellen“/„Schriftfarbe einstellen“/„Spalteneinstellungen“/„Buttonfarbe einstellen“ (12) — teilen sich dieselbe `.paste-panel`-Struktur (sichtbar via CSS-Klasse `.open`) mit genau einem primären Bestätigen-Knopf (`.paste-actions .btn.primary`) und einem „Abbrechen“-Knopf, dessen ID stets auf „Cancel“ endet (`.paste-actions button[id$="Cancel"]`). Ein einziger globaler `window`-`keydown`-Listener (direkt neben der bestehenden Escape-Behandlung für die Menüs, siehe Abschnitt 10) deckt dadurch automatisch **alle** diese Dialoge ab, auch künftig neu hinzukommende, ohne pro Dialog eigens verdrahtet werden zu müssen:

- **Escape** klickt (sofern ein `.paste-panel.open` existiert) dessen Cancel-Knopf — exakt dieselbe Wirkung wie ein Klick auf „Abbrechen“ bzw. auf den Panel-Hintergrund: der Dialog schließt sich, ohne die Eingabe zu übernehmen. Funktioniert auch mit Fokus innerhalb eines mehrzeiligen Textfelds (z. B. „Programm bearbeiten“).
- **Enter** klickt den primären Bestätigen-Knopf des aktuell offenen Panels — mit zwei Ausnahmen: Innerhalb eines `<textarea>` (die vier mehrzeiligen Editoren „NC-Code einfügen“, „Programm bearbeiten“, „Maschinentypen bearbeiten“, „AG-Namen bearbeiten“) fügt Enter weiterhin ganz normal einen Zeilenumbruch ein, statt den Dialog zu bestätigen. Und Felder, die bereits einen eigenen Enter-Handler besitzen (die „Zeile:“-Sprungfelder, sowie die einzelnen DXF-/Werkzeug-/Dateiname-Eingabefelder, die direkt ihre jeweilige `confirm…()`-Funktion aufrufen) lösen dabei bereits selbst `e.preventDefault()` aus — der globale Listener erkennt das über `e.defaultPrevented` und tut in diesem Fall nichts zusätzlich (z. B. bestätigt „zur Zeile springen“ dadurch NICHT zugleich den ganzen Dialog).
- Bewusst NICHT über `.btn.ghost` (statt `button[id$="Cancel"]`) angesteuert: Der Skalieren-Dialog hat mit „Originalgröße“ einen zweiten ghost-Knopf, der von Escape nicht gemeint ist.

Fünf identisch aufgebaute Overlay-Panels („Paste-Panels“), jeweils mit „Abschicken“ (primär) und „Abbrechen“ (ghost):

| Panel | Zweck |
|---|---|
| CNC-Code einfügen | Programmtext direkt einfügen statt Datei zu laden |
| Maschinentypen bearbeiten | Komplette Maschinentyp-Konfiguration (alle Blöcke, Abschnitt 5) direkt als Text bearbeiten, inkl. nicht-blockierender Duplikat-Namen-Warnung |
| Arbeitsgang-Namen bearbeiten | Aktive Namensliste (Abschnitt 6) direkt bearbeiten |
| CNC-Programm bearbeiten | Geladenen Programmtext direkt bearbeiten (Änderungen fließen sofort in Anzeige/Simulation ein) |
| Speichern als… | Dateiname-Eingabe zum Herunterladen des aktuellen (ggf. bearbeiteten) Programmtextes |

(Die Panels selbst tragen ihre eigenen Überschriften „NC-Code einfügen“ bzw. „Programm bearbeiten“, unabhängig von den Menüpunkt-Beschriftungen aus Abschnitt 10.)

Zusätzlich zwei weitere `.paste-panel`-Dialoge nach demselben Grundmuster: „Werkzeugdaten einfügen“ (`#toolDataPanel`, siehe 9.7) sowie „DXF verschieben“ (`#dxfMovePanel`, drei Felder X/Y/Z statt zwei, siehe 9.8) — beide eher Werkzeug-/Eingabedialoge als reine Text-Editoren, folgen aber derselben Öffnen/Abbrechen/Übernehmen/Klick-außerhalb-schließt/Enter-bestätigt/Escape-bricht-ab-Technik (siehe Kasten oben).

„CNC-Code einfügen“ und „CNC-Programm bearbeiten“ besitzen zusätzlich eine Zeilennummer-Spalte (Gutter, synchron zum Scrollen des Textfelds) sowie ein eigenes „Zeile:“-Sprungfeld (`jumpToTextareaLine`) — das bewegt Cursor/Auswahl/Scrollposition innerhalb des Textfelds und ist unabhängig von der „Sprung in Zeile:“-Funktion der Transportleiste (die stattdessen die Wiedergabeposition im 3D-Viewer bewegt).

**Arbeitsgang-Sprung in „CNC-Programm bearbeiten“:** Neben dem Zeilensprung-Feld sitzt (durch eine dünne Trennlinie abgesetzt) ein zusätzliches Dropdown „Arbeitsgang:“ mit jedem per `current.sections` bereits erkannten Arbeitsgang (`index`. `title`, dieselbe Liste wie in der Arbeitsgänge-Spalte, siehe 7.2/9.3) — `populateProgramJumpOps()` füllt es beim Öffnen des Editors (`btnEditProgram`-Klick) frisch aus dem zu diesem Zeitpunkt aktuellen `current`. Eine Auswahl springt sofort per `jumpToTextareaLine()` zur Startzeile des gewählten Arbeitsgangs (`sec.startIdx + 1`, da 1-indexiert erwartet) — die gesamte Startzeile wird markiert und mittig in den sichtbaren Bereich gescrollt. Das Dropdown springt nach jeder Auswahl selbst sofort auf den (deaktivierten) Platzhalter „Springen zu…“ zurück, damit sich derselbe Arbeitsgang erneut auswählen lässt (ein `<select>` feuert sonst bei zweimaliger Wahl desselben Werts kein `change`-Ereignis). Dropdown und Trennlinie bleiben verborgen, wenn das aktuell geladene Programm kein erkanntes Arbeitsgang-Schema enthält — dieselbe Fallback-Logik wie bei der Arbeitsgänge-Spalte selbst (7.2/7.3). Im Editor selbst vorgenommene Einfügungen/Löschungen verschieben die ursprünglich ermittelten Startzeilen — dasselbe Verhalten wie beim reinen Zeilensprung.

„Speichern als…“ lädt den aktuellen `rawProgramText` als `Blob` (`type: 'text/plain;charset=utf-8'`) über einen unsichtbaren `<a download>`-Link herunter, ohne systemeigenen Speichern-Dialog.


## 12. Farbschema

Alle Farben sind zentral als CSS-Custom-Properties in `template_top.html` definiert (kein hartkodiertes Farbmaterial im übrigen Markup) — helles Theme (`:root`), System-Dark-Mode (`@media (prefers-color-scheme: dark) :root:not([data-theme="light"])`) und expliziter Dark-Mode-Toggle (`:root[data-theme="dark"]`).

**Hell:**
```
--bg:#eef2f4  --panel:#ffffff  --panel-2:#eef4f6  --border:#b7c2c9  --border-soft:#ccd6db
--ink:#17324a  --ink-muted:#58707e  --ink-faint:#8a9ba6
--accent:#ad1f2d  --accent-ink:#ffffff  --accent-soft:#f8e1e2
--ok:#157a4d  --ok-soft:#e1f3ea  --warn:#a85a00  --warn-soft:#fbead2  --danger:#b3261e  --danger-soft:#fbe2e0
--muted-soft:#eef2f4
--axis-x:#c73a3a  --axis-y:#1f8a52  --axis-z:#2b62c9  --grid-line:rgba(23,50,74,0.08)
--path-1:#ad1f2d  --path-2:#2b6674  --path-3:#1f8a52  --path-4:#8a4bb0  --path-5:#c9922a  --path-6:#1f9e9e
--tool-color:#6b7f8f  --dxf-color:#7d8f9c
```

**Dunkel:**
```
--bg:#0d1218  --panel:#141b23  --panel-2:#101620  --border:#263241  --border-soft:#1c2530
--ink:#e8edf1  --ink-muted:#93a3b0  --ink-faint:#62727e
--accent:#ef5b56  --accent-ink:#2a0806  --accent-soft:#3a1512
--ok:#49d191  --ok-soft:#123425  --warn:#f0b429  --warn-soft:#3a2b0a  --danger:#ff7a72  --danger-soft:#3a1614
--muted-soft:#1a232d
--axis-x:#ff7a72  --axis-y:#5fdb9a  --axis-z:#7ab0ff  --grid-line:rgba(232,237,241,0.08)
--path-1:#ef5b56  --path-2:#5fc2cf  --path-3:#5fdb9a  --path-4:#cf9bf0  --path-5:#e8b04a  --path-6:#7fd8d8
--tool-color:#b7c6d2  --dxf-color:#9fb3c2
```

`--path-1` … `--path-6` sind die zyklisch verwendeten Bahn-/Zeilenfarben je Arbeitsgang (`pathColor(i)` liest `--path-{(i%6)+1}`).

**Trennlinien im Standardmodus dunkler (Release 95, Sebastian: „Beim Standardmodus der Darstellung sind die hellgrauen Linien zum Aufteilen der Bereiche zu hell. Die sollten etwas dunkler werden.“):** Betroffen sind ausschließlich die beiden Variablen `--border`/`--border-soft` im hellen `:root`-Theme (siehe „Hell“ oben) — sie liefern flächendeckend jede Trennlinie zwischen Programmcode-/Arbeitsgänge-/3D-Bereich, jeden Panel-/Dialog-Rahmen, den Menüband-Trennstrich und ähnliches (`border`/`border-*`/`background`-Verwendungen von `--border`/`--border-soft` in `template_top.html`). Bis Release 94: `--border:#d6dee3`, `--border-soft:#e6ecef` (sehr hell, wenig Kontrast zu `--bg:#eef2f4`/`--panel:#ffffff`). Seit Release 95: `--border:#b7c2c9`, `--border-soft:#ccd6db` — deutlich sichtbarerer, aber weiterhin dezent-neutraler Grauton, gleiche Blaugrau-Nuance beibehalten. Bewusst **nicht** angefasst: der System-Dark-Mode (`@media (prefers-color-scheme: dark)`) sowie der explizite Dark-Mode-Toggle (`:root[data-theme="dark"]`, siehe „Dunkel“ oben) — beide verwenden ohnehin eigene, unverändert bereits kontrastreiche `--border`/`--border-soft`-Werte. Betrifft auch nicht die „Benutzerfarben“-Option (12.1) — diese überschreibt ausschließlich die drei Bereichs-Hintergründe, nicht `--border`/`--border-soft`. Verifiziert per Playwright (`getComputedStyle(document.documentElement).getPropertyValue('--border')` im hellen Farbschema).

**Schriftarten:** „IBM Plex Mono“ bleibt ausschließlich für tatsächlich angezeigten/bearbeiteten CNC-Programmcode reserviert — die Code-Anzeige selbst (`.code-line`, `.tree`), ihre Zeilennummern-Gutter (`.editor-gutter`) sowie die beiden Textfelder, die rohen Programmtext bearbeiten: „Code einfügen“ (`#pasteArea`) und „Programm bearbeiten“ (`#programEditArea`). Alle übrigen Schriftstellen der gesamten Oberfläche — Arbeitsgang-Index/-„3D“-Knopf, „Alle AG ausblenden“, Readout-/Mess-Panel, Transport-/Sprung-Eingaben, Dateiname-Anzeige, Werkzeugleisten-Knöpfe sowie die Textfelder „Maschinentypen bearbeiten“/„Arbeitsgang-Namen bearbeiten“/„CNC Sim“-Dateiname — verwenden „Open Sans“ (`:root`s Basis-`font-family`). Die per Canvas-2D `measureText()` gemessenen Hilfskonstanten für die automatische Spaltenbreiten-Berechnung (`OPIDX_FONT`, `OPTITLE_FONT`, `OP3D_FONT`, `OPS_HEAD_TITLE_FONT`, `OPS_SHOWALL_FONT` in `app.js`) sowie die Achsenbeschriftung „X“/„Y“/„Z“ im 3D-Canvas selbst sind entsprechend gesetzt, damit Layout-Messung und tatsächliche Darstellung übereinstimmen; `TREE_FONT` (echter Programmcode) bleibt „IBM Plex Mono“. Der Google-Fonts-`<link>` lädt „Open Sans“ (Gewichte 400/500/600/700) und „IBM Plex Mono“.

### 12.1 Farbprofile

Menüpfad: „Einstellungen“ → „Erscheinungsbild“ → „Farben“ → „Hintergrundfarbe“ (`#btnEditColors`), öffnet das Panel `#colorProfilePanel`.

**Drei Modi** (Radio-Auswahl im Panel, `applyColorProfile()` in `app.js`):
- **Standard** — erzwingt per `data-theme="light"` die eingebaute helle Palette, unabhängig von der Hell-/Dunkel-Einstellung des Browsers/Betriebssystems.
- **Dark Mode** — erzwingt per `data-theme="dark"` die eingebaute dunkle Palette.
- **Benutzerfarben** — lässt die übrige Oberfläche (Menüband, Knöpfe, Editoren, Buttons …) bewusst unangetastet weiter der Browser-/System-Einstellung folgen (kein `data-theme` gesetzt) und überschreibt ausschließlich drei Hintergründe per Hexcode: „Hintergrund CNC-Programmcode“ (Hintergrund der Programmcode-Spalte `#sidebarCol`), „Hintergrund Arbeitsgänge“ (`#opsCol`) und „Hintergrund 3D Simulation“ (der 3D-Canvas selbst — malt seinen Hintergrund pro Frame per `ctx.fillRect()` aus der globalen CSS-Variable `--viewer-bg`, siehe `render()`). Jede Zeile hat sowohl ein Text- als auch ein natives `<input type="color">`-Feld, die sich gegenseitig synchron halten; ein ungültiger Hexcode (weder 3- noch 6-stellig) zeigt eine Warnung und verhindert „Übernehmen“, statt kommentarlos zu übernehmen oder abzustürzen.
- Solange „Hintergrundfarbe“ noch nie geöffnet/bestätigt wurde (weder in dieser Sitzung noch per `localStorage`), bleibt die Seite beim automatischen Verhalten — kein `data-theme` gesetzt, keine Hintergründe überschrieben.

**Automatische Lesbarkeit:** `deriveReadableInk(bgHex)` berechnet aus der relativen Luminanz (WCAG-Formel) der gewählten Hintergrundfarbe, ob eine helle oder dunkle Haupt-Schriftfarbe nötig ist (Schwelle 0.5), und blendet daraus zwei abgestufte Varianten (35 %/60 % Richtung Hintergrundfarbe) für „gedämpfte“/„blasse“ Schrift — dieselbe optische Hierarchie wie bei den beiden eingebauten Paletten, nur dynamisch statt fest kodiert. Für Programmcode- und Arbeitsgänge-Spalte werden Hintergrund und die drei Schriftfarben-Variablen (`--ink`/`--ink-muted`/`--ink-faint`) direkt per Inline-Style auf genau dem jeweiligen Spalten-Element gesetzt (bewusst nicht als selbstreferenzierende CSS-Regel im Stylesheet — eine Formel wie `--ink: var(--code-ink, var(--ink))` im Stylesheet selbst würde einen CSS-Abhängigkeitszyklus erzeugen und `--ink` komplett ungültig machen; per Inline-Style gesetzte konkrete Werte umgehen dieses Risiko vollständig) — dank normaler CSS-Vererbung wirkt das automatisch auf jeden Text innerhalb der jeweiligen Spalte. Für den 3D-Bereich wird zusätzlich die Gitterlinienfarbe (`--grid-line`) passend zur neuen Kontrastfarbe eingefärbt (gleiche leichte Transparenz wie bei den eingebauten Paletten); ebenso folgen die Umrandung des Mess-Punkts und das „Loch“ des Positionsmarkers `--viewer-bg` statt der fest verwendeten `--panel`, damit beide optisch zum tatsächlichen 3D-Hintergrund passen.

**Persistenz:** Das gewählte Profil (Modus + bei Benutzerfarben die drei Hexcodes) wird in `localStorage` gespeichert und beim nächsten Seitenaufruf automatisch wieder angewendet — analog zur Maschinentyp-Konfiguration (Abschnitt 5).

**„Einstellungen exportieren“/„Einstellungen importieren“:** Die JSON-Einstellungsdatei (Abschnitt 5.9) enthält ein Textfeld `colorProfileText` in folgender Klammer-Syntax mit den drei tatsächlich aktiven (aufgelösten) Hintergrundfarben — unabhängig davon, ob gerade „Standard“, „Dark Mode“ oder „Benutzerfarben“ aktiv ist:
```
[COLORPROGRAMCODE]
#ffffff
[/COLORPROGRAMCODE]
[COLORPROCESSINGSTEPS]
#eef4f6
[/COLORPROCESSINGSTEPS]
[COLOR3DSIM]
#eef4f6
[/COLOR3DSIM]
```
Da nur konkrete Hexcodes gespeichert werden (kein separates Modus-Feld), wendet „Einstellungen importieren“ diese drei Werte beim Wiedereinlesen immer als „Benutzerfarben“ an — das stellt das exakte sichtbare Ergebnis wieder her, auch wenn die ursprüngliche Auswahl „Standard“ oder „Dark Mode“ war. Fehlt das Feld (ältere/handgeschriebene Datei), bleibt das aktuell aktive Farbprofil unangetastet, dieselbe Regel wie bei den übrigen Einstellungsteilen.

### 12.2 Buttonfarbe — Menü-Hervorhebungsfarbe

Menüpunkt „Buttonfarbe“ (`#btnMenuColor`, „Einstellungen“ → „Erscheinungsbild“ → „Farben“) öffnet den Dialog `#menuColorPanel` mit zwei Hexcode-Feldern (je mit synchronisiertem `<input type="color">`-Farbwähler, dieselbe Text/Picker-Kopplungstechnik wie bei den Farbprofil-Feldern, 12.1): „Hintergrundfarbe“ (`#menuColorBgHex`/`#menuColorBgPicker`) und „Schriftfarbe“ (`#menuColorInkHex`/`#menuColorInkPicker`), sowie „Abbrechen“/„Übernehmen“.

**Wirkung:** Die gewählte Farbe färbt ausschließlich den Zustand „Menü gerade geöffnet“ der drei Menü-Trigger „Datei“/„Bearbeiten“/„Einstellungen“ (`.menu.open .menu-trigger`) — eine Farbe für alle drei gemeinsam. Der Standardzustand (Menü geschlossen) bleibt unberührt.

**Eigene CSS-Variablen statt Wiederverwendung von `--accent`:** Die Farbe wird über zwei eigens dafür angelegte CSS-Variablen `--menu-active-bg`/`--menu-active-ink` gesetzt, nicht über die bereits bestehenden `--accent`/`--accent-ink`, die an vielen weiteren Stellen der Oberfläche wiederverwendet werden (Fokusringe, Kippschalter, aktive Arbeitsgang-Hervorhebung, Badges u. a.) — eine direkte Umfärbung von `--accent` hätte all diese Stellen ungewollt mit ausgeweitet. `.menu.open .menu-trigger { background: var(--menu-active-bg, var(--accent)); color: var(--menu-active-ink, var(--accent-ink)); border-color: transparent; }` fällt über die CSS-`var()`-Fallback-Syntax automatisch auf die bisherige Optik (`--accent`/`--accent-ink`, Rot) zurück, solange keine eigene Menüfarbe gesetzt ist.

**Standard:** `DEFAULT_MENU_COLOR = { bg: '#ad1f2d', ink: '#ffffff' }` — dieselbe Rot-/Weiß-Kombination, die zuvor implizit über `--accent`/`--accent-ink` galt.

**Validierung „Schrift- und Hintergrundfarbe dürfen nicht identisch sein“:** `applyMenuHighlightColor(color)` prüft Gültigkeit beider Hexcodes und dass sie sich voneinander unterscheiden (case-insensitiver String-Vergleich der normalisierten Hexcodes); ist eine der beiden Bedingungen verletzt, bleibt die zuvor aktive Farbe vollständig unverändert, der Dialog zeigt stattdessen den Hinweistext `#menuColorWarning` („Schrift- und Hintergrundfarbe dürfen nicht identisch sein.“) und bleibt geöffnet.

**Persistenz:** `activeMenuColor = { bg, ink }` wird über `localStorage` (`LS_KEY_MENU_COLOR`) dauerhaft gespeichert (`persistMenuColor()`, try/catch-abgesichert wie alle übrigen Persistenz-Mechanismen, siehe 5.8) und beim Programmstart mit `loadPersistedMenuColor()` geladen und sofort angewendet. Zusätzlich Teil des JSON-Exports/-Imports „Einstellungen exportieren“/„Einstellungen importieren“ (`menuHighlightBg`/`menuHighlightInk`, siehe 5.9).

### 12.3 Schriftfarbe einzeln festlegen + „Standardwerte wiederherstellen“

Menüpunkt „Schriftfarbe“ (`#btnFontColor`) im „Farben“-Untermenü des „Einstellungen“-Menüs (`#submenuColorsDropdown`), direkt nach „Hintergrundfarbe“ (12.1). Öffnet das Panel `#fontColorPanel` mit fünf unabhängigen Zeilen — für jeden Bereich eine eigene Zeile mit Häkchen + Hexcode-Textfeld + `<input type="color">`-Farbwähler (dieselbe Text/Picker-Kopplungstechnik wie bei „Hintergrundfarbe“, 12.1, und „Buttonfarbe“, 12.2):

- **CNC-Programmcode Zeilennummerierung** — die Zeilennummern-Spalte der Code-Anzeige (`.code-line .ln`)
- **CNC-Programmcode Code** — der eigentliche Programmcode-Text daneben (`.code-line .tx`)
- **Arbeitsgänge** — Index und Titel jedes Eintrags in der Arbeitsgänge-Spalte (`.op-item .op-idx`/`.op-title`)
- **Geschwindigkeit** — Beschriftung und Wert der vier Geschwindigkeitsstufen-Knöpfe im Transportbereich (`.speed-caption`, `.speed-btn`)
- **Sprung in Zeile** — Beschriftung und Eingabefeld des „Sprung in Zeile“-Reglers (`.jump-group label`, `.jump-group input[type="number"]`)

**Jede Zeile unabhängig ein-/ausschaltbar:** Ein aktiviertes Häkchen setzt für genau diesen Bereich die daneben gewählte Farbe fest; bleibt es deaktiviert, verwendet dieser Bereich unverändert automatisch die zum aktiven Hintergrund passende Farbe (siehe „Hintergrundfarbe“/`deriveReadableInk()`, 12.1) — die Funktion ist eine reine, optionale Ergänzung „obendrauf“, keine Ablösung der automatischen Lesbarkeits-Logik. Ein ungültiger Hexcode bei einer aktivierten Zeile zeigt eine Warnung (`#fontColorHexWarning`) und verhindert „Übernehmen“.

**Technische Umsetzung — zweiargumentiger `var()`-Fallback statt neuer `:root`-Variablen:** Eine neue Variable wie `--font-code-text: var(--ink);` direkt am `:root` zu deklarieren würde die „Benutzerfarben“-Funktion (12.1) brechen: Dort werden `--ink`/`--ink-muted`/`--ink-faint` bewusst nicht global am `:root`, sondern per Inline-Style direkt auf den einzelnen Spalten-Elementen (`#sidebarCol`/`#opsCol`) gesetzt, damit jede Spalte ihre eigene, zu ihrem eigenen Hintergrund passende Schriftfarbe erhält — eine globale `:root`-Variable, die sich beim Deklarieren bereits auf den globalen `--ink`-Wert bezieht, würde diese Spalten-genaue Auflösung umgehen. Stattdessen wird der Fallback direkt an jeder betroffenen CSS-Verwendungsstelle selbst eingebaut, z. B.:
```css
.code-line .ln { color: var(--font-code-linenum, var(--ink-faint)); }
.code-line .tx { color: var(--font-code-text, var(--ink)); }
```
Solange `--font-code-text` nirgends gesetzt ist, löst der Browser den inneren Fallback `var(--ink)` genau dort im Baum auf, wo die Regel tatsächlich greift — innerhalb von `#sidebarCol` bekommt er also weiterhin den dortigen Inline-Style-Wert von „Benutzerfarben“ zu sehen. Erst wenn `applyFontColors()` (`app.js`) die jeweilige `--font-*`-Variable per `root.style.setProperty()` global am `:root` setzt, gewinnt sie überall — unabhängig vom lokalen `--ink`. Dieselbe Technik wird an allen fünf betroffenen Stellen angewendet, für „Arbeitsgänge“ und „Sprung in Zeile“ jeweils mit zwei eigenen Variablen (`--font-ops-idx`/`--font-ops-title` bzw. `--font-jump-label`/`--font-jump-input`), da Index/Titel bzw. Beschriftung/Eingabefeld unterschiedliche Standard-Fallbacks haben (`--ink-faint` bzw. `--ink-muted` statt `--ink`), beim Setzen einer eigenen Farbe aber beide gemeinsam dieselbe gewählte Farbe erhalten.

**Bewusste Ausnahme — „aktueller Arbeitsgang“ behält seine Akzentfarbe:** Der jeweils aktuell aktive Arbeitsgang wird unverändert in der Akzentfarbe hervorgehoben (`.op-item.current .op-idx, .op-item.current .op-title { color: var(--accent); font-weight: 600; }`) — diese Regel ist spezifischer als die `.op-item .op-idx`-Regel und gewinnt daher weiterhin, auch wenn für „Arbeitsgänge“ eine eigene Schriftfarbe gesetzt ist. Das ist beabsichtigt: Die Akzent-Hervorhebung ist ein Status-Hinweis, keine normale Textfarbe, und soll unabhängig von der gewählten Schriftfarbe erkennbar bleiben.

**„Standardwerte wiederherstellen“** — flacher Menüpunkt `#btnResetDisplayDefaults` direkt im „Einstellungen“-Menü (nicht im „Farben“-Untermenü), unmittelbar nach „Spalteneinstellungen“ (`#btnColumnScale`). Ein Klick fragt zuerst per nativem `window.confirm()`-Dialog nach („Standardwerte werden wiederhergestellt, alle vorherigen Einstellungen zurückgesetzt. Fortführen?“); nur bei „OK“ (Ja) ruft `resetAllDisplaySettingsToDefaults()` (`app.js`) tatsächlich alle vier Anzeige-Einstellungen in einem Schritt zurück, bei „Abbrechen“ (Nein) bricht die Funktion sofort ab, ohne dass irgendeine der vier Einstellungen angerührt wird:
- **Hintergrundfarbe** — entfernt den persistierten Farbprofil-Eintrag, wodurch das automatische Standard-/Dark-Mode-Verhalten (kein `data-theme` erzwungen) wieder greift.
- **Buttonfarbe** — setzt die Menü-Hervorhebungsfarbe zurück auf `DEFAULT_MENU_COLOR` (Rot/Weiß, siehe 12.2).
- **Schriftfarbe** — entfernt alle fünf `--font-*`-Variablenpaare wieder vollständig vom `:root` (identisch zum Zustand „noch nie geöffnet“) und löscht den zugehörigen `localStorage`-Eintrag.
- **Spaltenbreiten** — setzt `columnScalePreference` zurück auf die eingebauten Standardwerte (je 100 %, siehe 7.3a) und wendet diese sofort auf ein gerade geladenes Programm an, falls eines aktiv ist.

Alle vier zugehörigen Bedienpanels (sofern gerade geöffnet oder als Nächstes geöffnet) zeigen unmittelbar danach wieder ihren jeweiligen Ausgangszustand, ohne dass ein erneutes Neuladen der Seite nötig ist. Bewusst der native `window.confirm()` statt eines eigenen `.paste-panel`-Dialogs: Der native Dialog blockiert zuverlässig jede weitere Interaktion bis zur Entscheidung, benötigt kein eigenes Markup/CSS und bildet die geforderte Ja/Nein-Logik direkt 1:1 ab.

**JSON-Export/-Import („Einstellungen exportieren“/„Einstellungen importieren“, Abschnitt 5.9):** Die exportierte Einstellungsdatei enthält ein Feld `fontColors` mit genau den aktuell aktivierten Schriftfarben-Zeilen (z. B. `{"code":"#00ff00","speed":"#fedcba"}` — nur tatsächlich gesetzte Zeilen, keine leeren/deaktivierten). Beim Wiedereinlesen wendet „Einstellungen importieren“ diese Werte über dieselbe `applyFontColors()`-Funktion wie beim regulären „Übernehmen“ an und persistiert sie ebenso in `localStorage`; fehlt das Feld (ältere/handgeschriebene Datei), bleiben die aktuell aktiven Schriftfarben unangetastet — dieselbe Rückwärtskompatibilitäts-Regel wie bei den bereits bestehenden Einstellungsteilen (Farbprofil, Spaltenbreite, Menü-Hervorhebungsfarbe).


## 13. Bekannte Einschränkungen

1. Kein automatisches Laden aus einem lokalen Dateipfad (z. B. `C:\Temp`) — aus Browser-Sicherheitsgründen nicht möglich; Dateien müssen aktiv ausgewählt, per Drag & Drop gezogen oder eingefügt werden.
2. Die Dreh-/Kippwinkel-Zuordnung (A/B/C → „Kipp A“/„Kipp B“/„Dreh C“) ist eine verbreitete, aber maschinenabhängige Konvention — keine maschinenspezifische Achsbelegung ist hinterlegt.
3. Der Bahn-Viewer nutzt einen generischen ISO-Achsen-Parser; für Dialekte ohne erkennbare G0/G1-Wörter wird die Bahn standardmäßig als Eilgang (gestrichelt) dargestellt, sofern kein anderslautendes Muster (z. B. KUKA PTP/LIN/CIRC) erkannt wird.
4. SWITCHCASEMACHINE-Regeln (Maschinentyp-Auswahl, Abschnitt 5) matchen rein literal auf Zeichenketten (kein G-Code-Tokenizer) und wirken auf die gesamte Rohdatei, bevor der Arbeitsgang-Parser läuft — kurze/generische Suchtexte (insbesondere einzelne Buchstaben) können theoretisch auch unbeteiligten Text treffen. Der `detectionText`-Mechanismus (4.5) schützt gezielt die strukturellen Erkennungsmarken; für sonstigen Freitext gilt ein reduziertes Restrisiko (nur noch außerhalb der drei erkannten Kommentar-Arten `;`/`KM="…"`/`{…}` sowie innerhalb runder Klammern, siehe 5.11). CALCMACHINE (5.3) unterstützt zudem nur das einfache Schema `Achse*Faktor`, keine komplexeren Formeln.
5. Gemischte Kommentarstile im selben Programm werden zwar unterstützt (jeder Arbeitsgang merkt sich seinen eigenen erkannten Stil), ein sehr kurzes/generisches Muster eines Stils könnte in einer Datei eines anderen Stils theoretisch zufällig zutreffen. Der Biesse/CIX-Stil ist davon bewusst ausgenommen (nur bei erkannter `.CIX`-Kopfzeile aktiv). Der listenbasierte `NAMED`-Stil trägt ein analoges Restrisiko: je mehr generische/kurze Einträge die editierbare Liste enthält, desto eher könnte eine eigentlich unbeteiligte Zeile fälschlich als Arbeitsgang erkannt werden.
6. Eine Nullpunktverschiebung (`G92`/`O`/getauschtes `TRANS`, siehe 9.1) wird ausschließlich literal an `ORIGIN_RESET_RE` erkannt — sie muss als eigenes Achswort/eigene Zeile im (ggf. bereits getauschten) Programmtext vorkommen. Enthält ein und dieselbe Zeile sowohl eine Nullpunktverschiebung als auch eine tatsächliche Werkzeugbewegung (in der Praxis unüblich), wird für diese Zeile ebenfalls kein Bahnpunkt gezeichnet.
7. „↑ Oben“ zeigt bewusst die Kamera-Position, die „X rechts, Y oben“ liefert (siehe 9.2) — bei einem Programm, dessen Z-Achse invertiert vorliegt (Z+ nach unten statt oben, in der Praxis selten), würde dieselbe Ansicht dann fälschlich als von unten wirken. Es ist keine maschinenspezifische Achsrichtungs-Erkennung hinterlegt; die Zuordnung geht von der gängigen Konvention „X rechts, Y nach hinten, Z nach oben“ aus.
8. Die Werkzeugachsen-Rotationsreihenfolge in `toolAxisDirection()` (siehe 9.7) — erst Kipp A, dann Kipp B, zuletzt Dreh C — ist eine begründete, aber nicht verifizierte Annahme über die tatsächliche Schwenkkopf-Kinematik. Bei nur einem gleichzeitig aktiven Winkel spielt sie keine Rolle; bei mehreren gleichzeitig aktiven Winkeln könnte die dargestellte Werkzeugneigung von der realen Maschine abweichen.
9. Ein gemeldetes Problem („Das ‚Auge‘ zum Anzeigen/Verstecken funktioniert noch nicht bei allen Arbeitsgängen/Maschinen“, zur Datei `PROG.CNC`) konnte trotz umfangreicher, gezielter Tests (Einzeltoggle aller Arbeitsgänge, kumulatives Aus-/Wiedereinblenden, Toggle bei aktivem wie bei Kontext-Arbeitsgang) nicht reproduziert werden — alle getesteten Kombinationen funktionierten korrekt. Es fehlt eine genauere Fehlerbeschreibung (welcher Arbeitsgang, welcher Build-Stand) für eine erneute Untersuchung.
10. Die Voraberkennung/Vorschlag im Maschinentyp-Dropdown (5.4) ist nur eine unverbindliche Empfehlung: Sie erkennt weder eine völlig neue, noch nicht konfigurierte Maschine, noch garantiert sie bei mehrdeutigem Kopftext/Freitext den „richtigen“ Vorschlag — sie ersetzt daher nicht die Prüfung durch den Benutzer vor dem Klick auf „Ok“.
11. Die Reichenbacher-Dreh-/Kippwinkel-Umrechnung (siehe 9.7, „Reichenbacher-Kardanwinkelkorrektur“, sowie die Projektnotiz „Notiz_Reichenbacher_Aggregat_Winkelkorrektur“ für die vollständige Herleitung) ist ein eigener, abschaltbarer Schalter neben dem Dateinamen, Standardzustand „an“. Die Formel ist an drei Kippwinkeln (30°/45°/90°) aus realen Beispielpaaren exakt verifiziert; ein noch feineres/weiteres Beispiel (insbesondere ein „krummer“ Winkel abseits von 30°/45°/90°) wäre trotzdem hilfreich, um sie endgültig abzusichern. Der eine darin frei wählbare, skalare Parameter — der Schwenkwinkel des Aggregats, baubedingt 45° — ist über den Parameter `SWIVELANGLE=<Zahl>` im jeweiligen Maschinentyp-Block konfigurierbar (Fallback 45°, siehe 5.12); die eigentliche trigonometrische Formel selbst bleibt fest in `app.js` verdrahtet, da CALCMACHINE (5.3) dafür strukturell nicht ausreicht (siehe Punkt 4). Der Reichenbacher-Maschinentyp-Block (SWITCHCASEMACHINE/CALCMACHINE, Abschnitt 5) bleibt davon ansonsten unberührt — CALCMACHINE enthält nur Platzhalter-Werte (`X*1`, `Y*1`, `Z*1`, keine Ersetzungsregeln), die Winkelkorrektur läuft komplett unabhängig davon in `toolAxisDirection()`.
12. Die G02/G03-Bogeninterpolation (9.1) unterstützt ausschließlich die XY-Ebene (G17, in der Holzbearbeitung der Regelfall) — ein Bogen in der XZ-/YZ-Ebene (G18/G19) wird als gerade Linie dargestellt. Ein G02/G03 muss zudem auf jeder Bogenzeile explizit erneut stehen (kein modales Fortschreiben, siehe 9.1) — ein Programm, das G02/G03 mehrzeilig-modal ohne Wiederholung nutzt, bekommt für die entsprechenden Folgezeilen eine gerade statt einer gebogenen Linie. Bei Bögen mit leicht unstimmigem Start-/Zielabstand vom Mittelpunkt (Rundungsungenauigkeit realer Postprozessoren) wird der Radius ausschließlich aus dem Startpunkt berechnet; der letzte tessellierte Punkt landet trotzdem immer exakt auf dem programmierten Zielpunkt, wodurch das allerletzte Bogensegment in diesem seltenen Fall minimal von einem exakten Kreisbogen abweichen kann. Nur das R-Format wird unterstützt, ein I/J-Format liefert keine Bogeninterpolation (siehe 9.1).
13. **DXF-Import (9.8):** Block-Referenzen (`INSERT`) werden nicht aufgelöst — Geometrie, die nur innerhalb eines `BLOCK`s definiert, aber nie per `INSERT` platziert wird, erscheint dadurch gar nicht. Nicht unterstützte Entitätstypen (u. a. TEXT/MTEXT, HATCH, SPLINE, ELLIPSE) werden erkannt/gezählt, aber nicht gezeichnet (siehe Tooltip der Statusanzeige) — 3DFACE ist davon ausgenommen und wird unterstützt (siehe 9.8). Extrusionsrichtung/OCS (Gruppencode 210/220/230) wird bei CIRCLE/ARC/LWPOLYLINE/POLYLINE nicht berücksichtigt — eine gedrehte, nicht in der Standard-XY-Ebene liegende Kreis-/Bogen-/Polylinien-Entität kann dadurch falsch orientiert erscheinen, reine 2D-Zeichnungen in der Standard-XY-Ebene sind davon nicht betroffen. 3DFACE-Entitäten sind von dieser Einschränkung nicht betroffen — ihre vier Eckpunkte liegen laut DXF-Format immer direkt in Weltkoordinaten vor, ohne OCS-Umrechnung. Die Werkzeugachsen-Rotationsreihenfolge/Kinematik-Annahmen (Punkt 8) betreffen die DXF-Unterlage nicht, da sie rein statisch (ohne Rotation) dargestellt wird. Wie das CNC-Programm selbst wird eine importierte DXF-Unterlage nicht über `localStorage`, „Einstellungen exportieren“/„Einstellungen importieren“ oder „Simulation speichern“ persistiert — sie muss nach einem Seiten-Reload erneut importiert werden; dasselbe gilt für die einzeln aus-/eingeblendete Layer-Auswahl (9.8a).


## 14. Automatisierte Tests

`test.js` (Node, `node test.js`) bindet `parser.js` und `swap.js` direkt per `require()` ein und prüft ausschließlich deren reine Logik (kein Browser/DOM nötig). Abgedeckt sind unter anderem: Maschinenerkennung für alle unterstützten Dialekte; alle Arbeitsgang-Kommentarstile (SCM, HOMAG, FORMAT4, MOROFF, KRC_BLOCK, CIX/Biesse, NAMED); Zwischenstopp-/Programmende-Muster; Nullpunktverschiebungs-Erkennung (`splitAtOriginResets`/`ORIGIN_RESET_RE`); SWITCHCASEMACHINE-/CALCMACHINE-Regelanwendung sowie `parseMachineTypes()`/`findDuplicateMachineTypeNames()`/`sortMachineTypeBlocksAlphabetically()`; `extractToolData()` (RADIUS/LENGTH/BLADE-Erkennung über alle unterstützten Kommentarstile); `negateYZ()`/`applyAxisCalc()`; `computeCommentMask()`/`isRangeUncommented()` (Kommentar-Maskierung für `;`, `KM="…"`, `{…}`, bewusste Nicht-Maskierung runder Klammern); `parseArcParams()`/`interpolateArc()` (G02/G03-Bogeninterpolation, ausschließlich R-Format); `parseSwivelAngle()`/`swivelAngleDeg` (Reichenbacher-Schwenkwinkel); sowie ein umfangreicher Testblock für `parseDXF()` (LINE, POINT, CIRCLE, ARC, LWPOLYLINE mit/ohne Bulge, klassisches POLYLINE/VERTEX/SEQEND, 3DFACE, nicht unterstützte Entitätstypen, leere/fehlerhafte Eingabe, SECTION/ENTITIES-Scoping, CRLF-Zeilenenden). Aktueller Stand: **281 grüne Assertionen**, Ausgabe endet mit „Alle Tests erfolgreich.“ bei Erfolg.

Reine `app.js`/DOM-/Canvas-Logik ohne Bezug zu `parser.js`/`swap.js` — Menüband und Symbolleisten, Farbprofile/Schriftfarben, das Messwerkzeug, die 3D-Kamera samt Gizmo, die Werkzeug-3D-Darstellung, der DXF-Import als UI (Menüpunkte, Statusgruppe, Verschieben-Dialog, Ein-/Ausblenden, Kamera-Framing), „Simulation speichern“ sowie der Host-Hook für eine optionale native Windows-Hülle — wird nicht über `test.js`, sondern end-to-end per Playwright/Chromium verifiziert (u. a. reale und synthetische Testprogramme, Pixel-/`getComputedStyle()`-Vergleiche vor/nach einer Aktion, sowie algebraische Node-Kontrollrechnungen für Kamera-/Zoom-Formeln).

**Build-Integritätsprüfung vor jeder Auslieferung:** Das gebaute `CNC_Simulation.html` (Konkatenation aus `template_top.html` + den vier in `<script>`-Tags gewickelten Quelldateien `parser.js`/`swap.js`/`encoding.js`/`app.js` + `</body></html>`, siehe Abschnitt 2) enthält exakt vier `<script>`-Tags und null Debug-Marker; zusätzlich läuft `node --check` einzeln über jede der vier Quelldateien, um Syntaxfehler vor der Auslieferung auszuschließen.


