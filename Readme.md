# CNC-Code Simulator — technische Spezifikation

Diese Spezifikation beschreibt ausschließlich den **aktuellen Stand** des Werkzeugs — keine Entwicklungshistorie, keine Änderungsprotokolle. Sie ist so detailliert gehalten, dass sich das Tool anhand dieses Dokuments exakt nachbauen lässt.

## 1. Zweck & Überblick

Der CNC-Code Simulator — gebrandet als „CNC SimX" — ist ein rein clientseitiges, browserbasiertes Werkzeug zum Laden, Analysieren und 3D-Simulieren von CNC-/ISO-Programmcode aus der Holzbearbeitung (WinPPZ/SCM, HOMAG, FORMAT4/Felder, Moroff, Biesse/CIX und weitere Dialekte, siehe Abschnitt 4/5). Es läuft vollständig im Browser ohne Server-Backend, ohne externe JavaScript-Bibliotheken und ohne Installation — ausgeliefert als eine einzige, in sich geschlossene HTML-Datei (`CNC_Simulation.html`).

Kernfunktionen:
- Laden einer CNC-Programmdatei (Dateiauswahl, Drag & Drop oder direktes Einfügen als Text — je immer genau ein Programm) mit automatischer Zeichenkodierungserkennung. Über einen eigenen Menüpunkt/ein eigenes Symbol/einen eigenen Kontextmenü-Eintrag „Neue CNC-Programmliste" können zusätzlich bis zu zehn Programme per Mehrfachauswahl gleichzeitig im Hintergrund geladen werden (**Programmliste**, siehe Abschnitt 3.2) — berechnet/in der 3D-Simulation angezeigt wird dabei stets nur genau ein per Dropdownmenü ausgewähltes, aktives Programm.
- Automatische Erkennung des Maschinen-/Postprozessor-Dialekts und Zerlegung des Programms in einzelne Arbeitsgänge, Zwischenstopps und Programmende-Abschnitte anhand des Kommentarstils.
- Eine per Dropdownmenü wählbare **Maschinentyp-Konfiguration** (Such-/Ersetz-Regeln plus Achsberechnung, je Maschinentyp editierbar), die erst nach expliziter Bestätigung („Ok") auf den geladenen Programmtext angewendet wird, bevor er angezeigt/simuliert wird (siehe Abschnitt 5) — Kommentare (`;`, `KM="…"`, `{…}`) bleiben dabei von Such-/Ersetz- und Achsberechnungsregeln ausgenommen (siehe 5.11).
- Eine editierbare **Arbeitsgang-Namensliste** für Postprozessoren ohne Trennzeilen-Konvention.
- Eine synchronisierte Zweispalten-Programmansicht (durchgehende Code-Liste + separate Arbeitsgänge-Spalte) mit automatischer Spaltenbreite und Ein-/Ausklapp-Stufen.
- Ein interaktiver 3D-Bahn-Viewer (Zoom zum Mauszeiger, Rotation/Pan, ein anklickbares Rhombenkuboktaeder-Ansichts-Gizmo oben rechts im 3D-Bereich mit „Home"-Knopf daneben, Wiedergabe mit vier Geschwindigkeitsstufen und Leertaste als Play/Pause-Kurzbefehl, Längen-, Winkel- und Codesprung-Werkzeug — alle drei gegenseitig exklusiv, Standard ist keines davon aktiv, dann verhält sich die Sim wie eine reine „Hand" zum Drehen/Verschieben —, Längen-/Winkelmessung mit ΔX/ΔY/ΔZ-Anzeige inklusive eingezeichneter Delta-Linien in einem frei verschiebbaren Info-Fenster), der Kreisbögen (G02/G03, R-Format) als echte tessellierte Bögen statt als gerade Linien darstellt (siehe Abschnitt 9.1). Bei aktiviertem Codesprung (eigener Knopf/Kontextmenü-Eintrag, siehe 9.4a) springt ein Klick auf eine Bahnlinie zur zugehörigen Quelltextzeile und markiert Start-/Endpunkt der angeklickten Stelle mit einem kleinen „[]" im 3D.
- Editoren zum direkten Bearbeiten von Programmtext, Maschinentyp-Konfiguration und Arbeitsgang-Namensliste sowie „Speichern als…" zum Herunterladen des (ggf. bearbeiteten) Programms.
- Export/Import aller Simulationseinstellungen (Arbeitsgang-Namensliste + Maschinentyp-Konfiguration + Farbprofil + Schriftfarbe + Arbeitsgangfarbe + Spalteneinstellungen + Menü-Hervorhebungsfarbe, nicht das Programm selbst) als Datei über „Einstellungen exportieren"/„Einstellungen importieren" (siehe Abschnitt 5.9), zusätzlich zur automatischen localStorage-Persistenz.
- Frei einstellbare Standard-Skalierung (25/50/75/100 %) für die Breite der Programmcode-Spalte sowie ein/aus für die Arbeitsgänge-Spalte, eine frei wählbare Hervorhebungsfarbe für die aktiven Menü-Knöpfe „Datei"/„Bearbeiten"/„Einstellungen", fünf einzeln festlegbare Schriftfarben für Zeilennummerierung, Programmcode, Arbeitsgänge, Geschwindigkeit und Sprung-in-Zeile sowie zehn frei festlegbare Arbeitsgangfarben (1. bis 10. Arbeitsgang, erst ab dem 11. Arbeitsgang zyklisch wiederholt, siehe 12.4) (gebündelt unter „Einstellungen" → „Erscheinungsbild", siehe Abschnitt 10 und 12.3/12.4) sowie ein „Standardwerte wiederherstellen"-Knopf, der alle fünf genannten Anzeige-Einstellungen in einem Schritt zurücksetzt — alle dauerhaft persistiert.
- Ein separater, vom CNC-Programm unabhängiger **DXF-Import** („Datei" → „DXF Importieren" oder Drag & Drop einer `.dxf`-Datei) für eine verschieb-, spiegel-, dreh- und mit Bauteilstärke versehbare Referenz-Unterlage, die im 3D-Bereich unter der gefrästen Kontur dargestellt wird (siehe Abschnitt 9.8), inklusive echter 3D-Flächenmodelle (3DFACE) und einer Layer-Verwaltung mit einzeln aus-/einblendbaren Layern.

## 2. Architektur

Das Tool besteht aus fünf Quelldateien, die zu einer einzigen HTML-Datei zusammengebaut werden:

| Datei | Inhalt |
|---|---|
| `template_top.html` | Bewusst gewachsenes „Tag-Soup"-Dokument **ohne** `<!DOCTYPE html>`, `<html>`, `<head>` oder `<body>` — beginnt direkt mit `<meta charset="UTF-8">` (siehe unten) gefolgt von `<title>`/`<link>`/dem kompletten `<style>`-Block, dann der eigentlichen UI-Struktur; endet nach dem letzten UI-Panel, ohne jegliche schließenden Grundgerüst-Tags. Der Browser ergänzt Head/Body beim Parsen implizit selbst (Quirks-Modus mangels `<!DOCTYPE html>`) — das funktioniert zuverlässig; ein `<!DOCTYPE html>` wird bewusst nicht ergänzt, da das den Browser vom bestehenden Quirks- in den Standards-Modus umschalten und dadurch unklare Layout-Nebenwirkungen auslösen könnte. |
| `parser.js` | Maschinenerkennung, Arbeitsgang-/Zwischenstopp-/Programmende-Erkennung, DXF-Parser. Reine Logik ohne DOM-Zugriff (UMD-Modul, im Browser als `window.CNCParser`, in Node per `module.exports` einbindbar). |
| `swap.js` | Parsen und Anwenden der Maschinentyp-Austauschregeln. Ebenfalls reine Logik ohne DOM-Zugriff (`window.CNCSwap`). |
| `encoding.js` | Robuste Zeichenkodierungserkennung beim Datei-Einlesen (`window.CNCEncoding`). |
| `app.js` | Gesamte UI-Logik, DOM-Verdrahtung, 3D-Rendering, App-Zustand. Läuft als selbstausführende Funktion `(function(){ … })();`, greift auf `window.CNCParser`/`window.CNCSwap`/`window.CNCEncoding` zu. |

**Build-Vorgang:** Die vier Skriptdateien werden jeweils in ein eigenes `<script>…</script>`-Element eingebettet (Reihenfolge: `parser.js`, `swap.js`, `encoding.js`, `app.js`) und per **direkter String-Konkatenation** an `template_top.html` angehängt:

```
template_top.html + <script>parser.js</script> + <script>swap.js</script> + <script>encoding.js</script> + <script>app.js</script>
```

Kein Ersetzen eines Platzhalters, kein Anhängen schließender `</body></html>`-Tags — `template_top.html` enthält ohnehin kein Grundgerüst, an das sich sowas anhängen ließe (siehe Abschnitt 2, Tabelle oben), die Datei endet nach dem letzten `</script>` einfach. Das Ergebnis ist eine einzige, vollständig eigenständige HTML-Datei ohne externe Skript-Referenzen (einzige externe Ressource: der Google-Fonts-Import für „Open Sans"/„IBM Plex Mono" ganz am Anfang der Datei).

**Zeichenkodierung des Grundgerüsts selbst:** Ganz an erster Stelle der Datei — noch vor jedem Kommentar, garantiert innerhalb der ersten 1024 Bytes — steht `<meta charset="UTF-8">`. Das stellt sicher, dass die Zeichenkodierung unabhängig vom Ausliefer-/Ladeweg zuverlässig als UTF-8 erkannt wird: Beim Öffnen als `file://` bzw. über die WebView2-Hülle (Chromium) wird die Kodierung ohnehin zuverlässig als UTF-8 erkannt; sobald die Datei aber über einen echten HTTP-Server ohne Charset-Angabe im `Content-Type`-Header ausgeliefert wird (wie bei generischem statischem Hosting, das die Dateiendung nicht besonders behandelt), muss der Browser die Kodierung sonst selbst raten und kann danebenliegen — sichtbar u. a. als Mojibake bei den Symbolleisten-Icons. Bewusst **kein** `<!DOCTYPE html>` ergänzt (siehe Abschnitt 2, Tabelle oben) — das würde den bestehenden Quirks-Modus verlassen und wäre ein unnötig großer Eingriff für ein reines Kodierungsproblem.

**Tests:** `test.js` (Node, kein Browser/DOM nötig) lädt `parser.js` und `swap.js` direkt per `require()` ein und prüft deren Logik anhand synthetischer und teils realer Beispieltexte per einfachem Assertion-Helfer. Ausführung: `node test.js`. Siehe Abschnitt 14.

**Layout:** Die App ist als Einzelbildschirm-Anwendung angelegt (`.app { height: 100vh; min-height: 640px; display: flex; flex-direction: column; }`) — Menüband, Symbolleiste, Transportleiste, Seitenleiste (Programmliste + Arbeitsgänge-Spalte) und 3D-Bereich füllen zusammen den Viewport. Jeder Bereich mit potenziell überlaufendem Inhalt (`.tree`, `.ops-list`, bei schmalen Fenstern zusätzlich die gesamte `.sidebar`, siehe die `@media (max-width: 820px)`-Regel) scrollt zusätzlich **intern** über sein eigenes `overflow-y: auto`; `html`/`body` selbst tragen kein eigenes `overflow: hidden` (nur `margin: 0`), sodass die Gesamtseite bei sehr kleinen Fenstern bzw. durch Symbolleisten weiter verkleinerten Browserfenstern weiterhin ganz normal scrollbar bleibt.

## 3. Datei-Laden & Vorverarbeitung

**Zeichenkodierung** (`encoding.js`, `decodeBuffer(buffer)`): Jede Datei wird unabhängig vom Ladeweg immer als `ArrayBuffer` eingelesen (`FileReader.readAsArrayBuffer`), nie über `readAsText` (dessen UTF-8-Standardannahme bei den in der Praxis häufigen Windows-1252-kodierten Postprozessor-Exporten zu stillschweigend falscher Dekodierung führen würde). Dekodierstrategie: zuerst **strikt** als UTF-8 (`new TextDecoder('utf-8', {fatal:true})`) — schlägt bei einer Windows-1252-Datei mit hohen Bytes zuverlässig fehl statt sie lautlos falsch zu lesen; erst dann Fallback auf `new TextDecoder('windows-1252')`.

**N-Zeilennummern:** `stripN(line)` entfernt eine führende Zeilennummerierung (`/^\s*N\d+\s*/`) ausschließlich für die interne Mustererkennung (Maschinen-/Arbeitsgang-Erkennung) — die Original-Zeilen mit N-Nummer bleiben für Anzeige und Bewegungs-Extraktion unverändert erhalten.

**Ladewege ("klassisch", je immer genau EIN Programm):**
- „Programm laden" (verstecktes `<input type="file">`, im „Datei"-Menü unter „CNC-Programm", ebenso über das Symbolleisten-Icon „📂 CNC" und „Programm laden" im Rechtsklick-Kontextmenü) oder Drag & Drop einer einzelnen Datei auf das Fenster (`window.addEventListener('drop', …)`, mit `dragover`-`preventDefault()`), jeweils über `readFileAsText()`/`decodeBuffer()`. Die Dropzone unterscheidet dabei anhand der Dateiendung (case-insensitiv): Eine `.dxf`-Datei geht an den separaten DXF-Import (siehe 9.8), jede andere Endung wird als CNC-Programm behandelt.
- „CNC-Code einfügen": Textfeld-Panel mit direktem Einfügen von Programmtext (kein Datei-Encoding-Schritt nötig, da bereits als JS-String vorliegend).
- **Host-Hook `window.__hostLoadProgramFromHost(base64Content, filename)`** — weiterer klassischer Ladeweg für eine optionale native Windows-Hülle, siehe 3.1.
- „CNC-Programm löschen" setzt Anzeige, Baum, Arbeitsgänge-Spalte, 3D-Szene und Transportleiste auf den Ausgangszustand zurück, **ohne** die Maschinentyp-Konfiguration zu entfernen (die Dropdown-Auswahl selbst fällt dabei auf „ISO" zurück, siehe Abschnitt 5.1). Ein eventuell geladenes DXF (Abschnitt 9.8) bleibt davon vollständig unberührt. Ein evtl. im Hintergrund bestehender Rest einer Programmliste (Abschnitt 3.2) bleibt dabei bewusst unangetastet — „CNC-Programm löschen" entfernt nur die gerade aktive Anzeige, nicht die Hintergrund-Programme.

Alle klassischen Ladewege laufen über `loadTextClassic()` (`app.js`), die unverändert direkt `loadText()` aufruft — **komplett unabhängig von einer evtl. bestehenden Programmliste** (siehe 3.2): War zu diesem Zeitpunkt eine Zeile der Programmliste als „aktiv" markiert, wird diese Markierung dabei lediglich zurückgesetzt (`activeProgramId = null`), die Liste selbst bleibt im Hintergrund unverändert bestehen und über eine bereits sichtbare Dropdownliste weiterhin erreichbar. `loadText()` selbst bleibt dabei die einzige Funktion, die ein Programm tatsächlich als aktive Anzeige berechnet/parst.

Diese Trennung ist bewusst: Jeder normale Ladeweg (Menü/Symbolleiste/Kontextmenü/Drag & Drop/„CNC-Code einfügen"/Host-Hook) lädt immer nur genau ein Programm und ist komplett unabhängig von einer evtl. bestehenden Programmliste. Mehrfachauswahl ist ausschließlich über die drei dedizierten „Neue CNC-Programmliste"-Auslöser erreichbar (siehe 3.2).

### 3.1 Host-Hook für eine optionale native Windows-Hülle

Für eine optionale, native C#/WebView2-Hülle um die Simulation herum (Details siehe die Projekt-Dokumente „Konzept_EXE_C-Sharp.md", „Bauanleitung_EXE_C-Sharp_WebView2.md" und „Bauanleitung_EXE_fuer_Einsteiger.md") existiert ein globaler, rein additiver Einstiegspunkt, über den ein per Batch-Datei/Kommandozeilenargument bestimmtes CNC-Programm beim Start automatisch geladen werden kann:

```js
window.__hostLoadProgramFromHost = function (base64Content, filename) {
  var binary = atob(base64Content);
  var bytes = new Uint8Array(binary.length);
  for (var i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
  var text = window.CNCEncoding.decodeBuffer(bytes.buffer);
  loadTextClassic(text, filename);
};
```

Ein C#-Host (WebView2) kann diese global erreichbare Funktion per `CoreWebView2.ExecuteScriptAsync(...)` aufrufen. Bewusst werden **Rohbytes** (Base64-kodiert) statt eines vom Host bereits „geratenen" Texts übergeben, damit dieselbe, bereits bestehende UTF-8/Windows-1252-Erkennung (`window.CNCEncoding.decodeBuffer()`, siehe oben) exakt so greift wie beim regulären Laden im Browser — keine zweite, abweichende Kodierungs-Implementierung auf der C#-Seite nötig. Der übergebene Dateiname wird (anders als bei „CNC-Code einfügen", das fest „Eingefügter Code" einträgt) unverändert als `rawProgramFilename` übernommen, sodass `#filenameLabel` den tatsächlichen, vom Host übergebenen Dateinamen anzeigt. Dieser Ladeweg ist ein klassischer, von der Programmliste unabhängiger Einzelladeweg über `loadTextClassic()` (siehe 3./3.2) — exakt dasselbe Verhalten wie „CNC-Code einfügen" und die übrigen klassischen Ladewege.

### 3.2 Mehrere Programme in Programmliste laden

Zusätzlich zu den klassischen Einzelladewegen (siehe oben) können bis zu zehn Programme gleichzeitig im Hintergrund geladen werden (**Programmliste**). Berechnet und in der 3D-Simulation angezeigt wird dabei ausschließlich das eine, per Dropdownmenü ausgewählte aktive Programm — aus reinen Performancegründen: Parsing, Maschinentyp-Austausch und 3D-Berechnung laufen nur für das aktive Programm, alle übrigen Einträge liegen unangetastet im Hintergrund.

**Drei gleichwertige Auslöser**, die alle dieselbe Funktion `startNewProgramList(files)` aufrufen:

- **„Neue CNC-Programmliste"** — Menüpunkt `#btnNewProgramList` unter „Datei" → „CNC-Programm", direkt unter „Programm laden".
- **„☰+ Liste"** — Symbolleisten-Icon `#btnNewProgramListIcon` in `#toolbarRow1`, unmittelbar links von „☰✕ Liste" (`#btnClearProgramListIcon`, siehe unten). Anders als die übrigen CNC-Symbolleisten-Icons ist dieser Knopf nie deaktiviert (auch ohne geladenes Programm nutzbar, um eine erste Liste zu beginnen).
- **„Programmliste erstellen"** — Kontextmenü-Eintrag `#ctxNewProgramList` in `#appContextMenu`, hinter „Programm löschen", direkt gefolgt von „Programmliste löschen" (`#ctxClearProgramList`, siehe 10.2).

Alle drei öffnen denselben, separaten Datei-Dialog `#fileInputList` (`<input type="file" multiple>`, bis zu 10 Dateien) — bewusst getrennt vom „normalen", auf Einzelauswahl beschränkten `#fileInput` (siehe 3.). Erst nachdem im Dialog tatsächlich mindestens eine Datei ausgewählt wurde, ruft der `change`-Handler `startNewProgramList(files)` auf — ein abgebrochener Dialog (Escape/„Abbrechen") verändert eine evtl. bereits bestehende Programmliste dadurch nie.

**`startNewProgramList(files)`** **ersetzt** eine evtl. bereits bestehende Programmliste vollständig durch die neu getroffene Auswahl, statt an sie anzuhängen: War zu diesem Zeitpunkt ein Programm aktiv angezeigt (`rawProgramText != null`), wird zunächst `clearProgram()` aufgerufen, um die Anzeige zu leeren; danach werden `programList` auf ein leeres Array und `activeProgramId` auf `null` zurückgesetzt; abschließend füllt `addFilesToProgramList(files)` (siehe unten) die frische Liste mit der getroffenen Auswahl. Alte Einträge einer zuvor bestehenden Liste sind danach unwiderruflich weg — genau wie es der Name „**Neue** CNC-Programmliste" verspricht.

**Datenmodell (`app.js`):** `programList` ist ein reines Array roher `{id, filename, text}`-Einträge — bewusst **ohne** geparste Arbeitsgänge/Segmente je Eintrag (Performance: nur das aktive Programm wird berechnet/angezeigt). `activeProgramId` verweist auf den gerade in `rawProgramText`/`rawProgramFilename` geladenen Eintrag (`null`, solange nichts aktiv ist). `PROGRAM_LIST_MAX = 10`. Ein Eintrag wird ausschließlich über `activateProgramFromList(id)` aktiv geschaltet, die intern unverändert `loadText(entry.text, entry.filename)` aufruft — genau wie beim klassischen Einzelladefall. Solange ein Eintrag nur in `programList` liegt, aber nicht aktiv ist, wird er **nicht** geparst, nicht mit dem Maschinentyp ausgetauscht und beansprucht keinerlei 3D-Berechnung.

**Aufnahme neuer Dateien** (`addFilesToProgramList(files)`): liest die übergebenen Dateien parallel ein (`readFilesAsText()`), begrenzt sie auf die verbleibende Kapazität bis `PROGRAM_LIST_MAX` (mit Warnhinweis, falls mehr ausgewählt wurden, als noch Platz ist) und hängt sie an `programList` an. **Automatische Aktivierung** erfolgt ausschließlich in genau einem Fall: die Liste war vorher leer, **und** es wurde genau eine Datei hinzugefügt — z. B. bei „Neue CNC-Programmliste" mit nur einer ausgewählten Datei. Diese Funktion wird von zwei Stellen aus aufgerufen: von `startNewProgramList(files)` (Mehrfachauswahl über einen der drei Auslöser oben) sowie von „+ Add" (`#programListAddBtn`, siehe unten, auf Einzelauswahl beschränkt).

**Gemeinsames `#fileInput` für „Programm laden" und „+ Add" (`fileInputIntent`):** Das einzelauswahl-fähige `#fileInput` wird von zwei unterschiedlichen Stellen aus geöffnet — den klassischen „Programm laden"-Auslösern (siehe 3.) und „+ Add" in der Programmlisten-Dropdownliste (siehe unten) — die nach der Dateiauswahl unterschiedlich reagieren müssen (einmal `loadTextClassic()`, einmal `addFilesToProgramList()`). Ein Modul-Zustand `fileInputIntent` (`'load'` als Voreinstellung, `'add'`) hält fest, welcher der beiden Wege gerade gemeint ist: „+ Add" setzt `fileInputIntent = 'add'` unmittelbar bevor es `#fileInput` öffnet, der `change`-Handler liest den Wert aus, setzt ihn sofort wieder auf `'load'` zurück und verzweigt entsprechend.

**Dropdownliste** (`#programListBar`/`#programListDropdown`, eigene Zeile unterhalb der Kopfzeile der Programmcode-Spalte, nicht innerhalb von `.sidebar-head` selbst, um dessen bereits eng bemessene Kopfzeile nicht zusätzlich zu verengen): erscheint automatisch, sobald `programList.length >= 2` ist — bei nur einem oder keinem geladenen Programm (und solange die Liste noch nie mindestens zwei Einträge hatte) bleibt sie verborgen. **Einmal „aktiviert" bleibt die Dropdownliste auch bei nur noch einem einzigen verbliebenen Eintrag sichtbar/bedienbar**, statt unterhalb von zwei Einträgen wieder zu verschwinden — Zustand `programListActivated` (`app.js`): wird `true`, sobald `programList.length` einmal 2 erreicht, und erst wieder auf `false` zurückgesetzt, sobald die Liste komplett geleert wird (0 Einträge, z. B. über „CNC-Programmliste löschen"). Ein danach neu geladenes, einzelnes Programm verhält sich dann wieder exakt wie der unveränderte „normale" Einzeldatei-Ladefall (kein Dropdown, bis erneut mindestens zwei Programme geladen sind). Der Auslöser-Knopf (`#programListTrigger`) zeigt den Dateinamen des aktiven Programms (oder einen Hinweistext, falls keines aktiv ist) sowie den Zähler „x/10" (`#programListCount`). Ein Klick öffnet eine eigene, schlanke Liste (`#programListMenu`, dieselbe generische `.menu-item`-Optik wie die übrigen Menüs der App) statt eines nativen `<select>` — ein natives `<select>` könnte keine eingebetteten Lösch-Knöpfe je Zeile tragen. Jede Zeile besteht aus dem Dateinamen **mit Endung** (Klick aktiviert/berechnet genau dieses Programm über `activateProgramFromList()`, das aktive trägt eine eigene Hervorhebung) und einem roten „x" (`removeProgramFromList(id)`).

**Bleibt nach einer Entfernung genau noch ein einziges Programm in der Liste übrig UND ist zu diesem Zeitpunkt kein Programm (mehr) aktiv, wird dieses eine verbleibende Programm automatisch aktiviert** (`activateProgramFromList()`), statt die Anzeige zu leeren bzw. leer zu lassen. Diese Regel deckt zwei Fälle gleichermaßen ab:
  - Das gelöschte Programm war aktiv, danach bleibt genau ein Programm übrig.
  - Es war von vornherein **kein** Programm aktiv (z. B. direkt nach einer Mehrfachauswahl, bevor eine Auswahl getroffen wurde), trotzdem bleiben nach dem Löschen nur noch zwei Programme, von denen eines entfernt wird.

  In beiden Fällen würde das letzte verbleibende Programm ohne diese Sonderregel in einem unsichtbaren/unerreichbaren Zwischenzustand landen: Es bliebe zwar in `programList` erhalten, aber die Dropdownliste selbst verschwindet ja bereits unterhalb von zwei Einträgen (siehe oben) — ohne sichtbares Dropdown gäbe es dann keinen Weg mehr, dieses eine übrig gebliebene Programm auszuwählen. Bleiben nach dem Löschen dagegen weiterhin **zwei oder mehr** Programme übrig, oder ist weiterhin ein **anderes** Programm aktiv, gilt die normale Regel unverändert — die Auswahl unter mehreren verbliebenen Programmen ist nicht eindeutig, eine evtl. aktive Anzeige wird bei entferntem aktivem Eintrag geleert (`clearProgram()`) und die (weiterhin sichtbare) Dropdownliste überlässt die erneute Auswahl bewusst dem Anwender. Ganz oben in der Liste befindet sich „+ Add" (`#programListAddBtn`), das denselben `#fileInput`-Dialog wie „Programm laden" öffnet, mit dem Unterschied, dass es zuvor `fileInputIntent = 'add'` setzt (siehe oben) und die gewählte Datei dadurch über `addFilesToProgramList()` der bestehenden Liste hinzufügt statt sie klassisch anzuzeigen (deaktiviert, sobald `PROGRAM_LIST_MAX` erreicht ist). Dieser Dialog erlaubt — wie „Programm laden" selbst — nur die Auswahl einer einzelnen Datei (`#fileInput` hat kein `multiple`-Attribut); Mehrfachauswahl ist ausschließlich über „Neue CNC-Programmliste" möglich (siehe oben). Die Liste selbst schließt sich bei Klick außerhalb (`document.addEventListener('click', …)`, prüft `e.target.closest('#programListDropdown')`) oder mit Escape — eine eigene, einfache Logik, bewusst **nicht** die „nur Ok/Übernehmen/Enter/Esc schließt"-Regel der `.paste-panel`-Dialoge aus Abschnitt 11: die Programmliste ist ein reines Auswahlmenü ohne Texteingabe und soll sich wie die übrigen Menüs der App beim Klicken daneben schließen.

**„CNC-Programmliste löschen":** Menüpunkt `#btnClearProgramList` unter „Bearbeiten" → „CNC" (direkt unter „CNC-Programm löschen") sowie ein gleichwertiges Symbolleisten-Icon `#btnClearProgramListIcon` in derselben Zeile wie die übrigen CNC-Symbolleisten-Icons (Laden/Bearbeiten/Löschen, `#toolbarRow1`). Beide rufen `clearProgramList()` auf: leert — im Unterschied zum unveränderten „CNC-Programm löschen" (entfernt nur die aktive Anzeige, lässt Hintergrund-Einträge bewusst unangetastet, siehe 3.) — die **komplette** Programmliste inklusive eines evtl. gerade aktiven Programms; die Maschinentyp-Konfiguration bleibt dabei wie gehabt erhalten. Aktiv, solange `programList.length > 0` ist, unabhängig davon, ob die Dropdownliste selbst gerade sichtbar ist (z. B. bei nur einem Hintergrundprogramm).

**Performance:** Da nur der aktive Eintrag jemals `loadText()`/`displayCurrentProgram()` durchläuft, verursachen im Hintergrund liegende Programme keinerlei Parsing-, Maschinentyp-Austausch- oder 3D-Berechnungsaufwand — das Umschalten in der Dropdownliste kostet exakt so viel wie ein regulärer Neu-Ladevorgang eines einzelnen Programms.

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

Die Prüfung 1 (CIX) läuft **ausschließlich auf Zeile 1**, alle anderen auf den zusammengefügten ersten 60 Zeilen. Das Erkennungsergebnis wird nirgends direkt als Badge angezeigt (siehe Abschnitt 10, „Struktur-Anzeige") — es dient ausschließlich der Voraberkennung des Maschinentyp-Dropdowns (5.4) sowie als `machine`/`machineLabel` im Parser-Ergebnis.

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

Ein einzeiliger `KRC`-Stil (`(Titel)` ohne umgebende Trennzeilen) wird **dauerhaft nicht durchsucht**: Bei echten KRC-/MAKA-Programmen kommen runde-Klammer-Kommentare auch ganz gewöhnlich und häufig ohne jeden Bezug zu einem Arbeitsgang vor (z. B. `(Eilgang)`, `(Werkzeug zurück)`) — jeder davon würde sonst fälschlich als eigener Arbeitsgang gewertet. `MACHINES.KRC` bleibt nur für die Maschinen-Label-Zuordnung erhalten (`detectMachine()` liefert weiterhin `'KRC'` anhand der Kopfzeile `(KRC …)`); echte Arbeitsgänge dieser Maschinen werden ausschließlich über den bewusst strengen 3-zeiligen `KRC_BLOCK`-Stil erkannt (Trennzeile **davor und danach** verlangt) — strukturell identisch für KRC **und** MAKA.

**CIX-Gating:** Der `CIX`-Stil wird — anders als alle übrigen — **nur** durchsucht, wenn der von `detectMachine()` erkannte Maschinentyp selbst `CIX` ist (Zeile 1 enthält `.CIX`). Bei Gleichstand (zwei Stile treffen an derselben Startzeile) gewinnt der zuerst in der internen Reihenfolge (`STYLE_IDS`, Definitionsreihenfolge oben) stehende Stil; `preferredStyle` (der erkannte Maschinentyp) wird dabei bevorzugt sortiert, `NAMED` steht bewusst an letzter Stelle.

Da der Klammer-Block-Stil (`KRC_BLOCK`) kein eigenes Ende-Muster kennt (reicht immer bis zum nächsten Arbeitsgang bzw. Dateiende), entsteht zwischen zwei aufeinanderfolgenden Arbeitsgängen nie eine „Lücke" — ein `M00`/`M0` innerhalb eines Arbeitsgangs (z. B. zwischen zwei Klammer-Blöcken) wird deshalb Teil des vorangehenden Arbeitsgangs statt separat als „Zwischenstop KRC" (siehe 4.3) markiert zu werden; dasselbe gilt für `CIX`/`NAMED`.

### 4.3 Zwischenstopp- und Programmende-Muster

Zwischen zwei Arbeitsgängen (bzw. vor dem ersten/nach dem letzten) wird die Lücke mit `classifyGap()` weiter aufgeschlüsselt: Es wird nach dem frühesten der folgenden Muster gesucht (zur erkannten Maschine passende Muster werden bevorzugt, alle übrigen dienen als Fallback); nicht zugeordnete Bereiche werden als feste Abschnitte („Kopfbereich", „Fester Zwischenbereich", „Fester Fußbereich") übernommen.

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

Dieses Muster wird nur innerhalb einer tatsächlichen **Lücke** zwischen zwei Arbeitsgängen (oder vor dem ersten/nach dem letzten) gesucht. Da zwischen zwei KRC-Arbeitsgängen (siehe 4.2) praktisch nie eine solche Lücke entsteht, wird ein `M00` zwischen zwei Arbeitsgängen dort in der Praxis Teil des vorangehenden Arbeitsgangs statt als eigener „Zwischenstop KRC" markiert — das KRC-Zwischenstopp-Muster bleibt trotzdem wirksam für KRC-Dateien ganz ohne erkanntes Arbeitsgang-Schema (siehe 4.4).

### 4.4 Programm ohne erkanntes Schema

Wird **kein** Arbeitsgang-Kommentarstil gefunden, bleibt das Programm trotzdem gültig und wird als Ganzes simuliert (Zwischenstopp-/Programmende-Muster werden weiterhin gesucht). Für die 3D-Darstellung wird die Datei stattdessen an jeder erkannten **Nullpunktverschiebung** in Farbabschnitte geteilt: `ORIGIN_RESET_RE = /\bO\b|\bG92(?!\d)/i` (führendes `\b` verhindert Treffer in „OPTI"/„OPEN"; **kein** abschließendes `\b` nach `G92`, damit auch das durch eine Austauschregel entstandene zusammengeklebte `G92X100.00` noch erkannt wird; `(?!\d)` verhindert Fehltreffer wie `G920`). `splitAtOriginResets(lines)` liefert die einzelnen Farbabschnitte. Dieselbe Regex wird auch von `extractMotion()` in `app.js` verwendet, um Nullpunktverschiebungszeilen von der Bahn-Darstellung auszunehmen (siehe 9.1).

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

1. Ein CNC-Programm wird geladen (Datei, Drag & Drop, „CNC-Code einfügen") — der Code bleibt zunächst **unverändert** sichtbar, unabhängig davon, welcher Maschinentyp zuvor für ein anderes Programm aktiv war.
2. Eine Voraberkennung schlägt sofort einen Maschinentyp im Dropdown vor (siehe 5.4) — angewendet ist er **noch nicht**.
3. Erst ein Klick auf „Ok" wendet den im Dropdown gerade ausgewählten Maschinentyp tatsächlich an: SWITCHCASEMACHINE-Regeln (Suchtext→Ersatztext) und danach CALCMACHINE-Achsberechnung (siehe 5.3) werden auf die Anzeige/Simulation angewendet.
4. `rawProgramText` bleibt dabei unangetastet — die Transformation ist eine reine Anzeige-/Simulationsebene (siehe `displayCurrentProgram()`), „CNC-Programm bearbeiten"/„Speichern als…" zeigen bzw. speichern weiterhin den unveränderten Originaltext.
5. Der Sondername **„ISO"** steht immer als erste, feste Dropdown-Option für „Original, keine Umrechnung" — er ist nach jedem neuen Laden eines Programms voreingestellter, tatsächlich angewendeter Zustand (unabhängig vom Vorschlag in Punkt 2).

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
IJMODE=absolute
CORNERRADIUS=BR=
[/MACHINETYPE Name="Biesse"]
```

`CNCSwap.parseMachineTypes(configText)` extrahiert daraus eine Liste `{ name, switchRules, calcRules, rawSwitchText, rawCalcText, swivelAngleDeg, ijMode, cornerRadiusMarker }` je Block (`swivelAngleDeg` ist `null`, wenn der Block keine (gültige) `SWIVELANGLE=<Zahl>`-Zeile enthält — siehe 5.12; `ijMode` ist `'incremental'`, `'absolute'` oder `null`, wenn der Block keine `IJMODE=`-Zeile enthält — siehe 5.12a; `cornerRadiusMarker` ist der rohe, noch nicht zu einem RegExp verarbeitete Markertext oder `null`, wenn der Block keine `CORNERRADIUS=<Marker>`-Zeile enthält — siehe 5.12b):

- **Blockgrenzen:** Ein Block beginnt bei `[MACHINETYPE Name="…"]` und reicht bis zum **Beginn** des nächsten solchen Treffers bzw. bis zum Dateiende — der schließende Marker `[/MACHINETYPE Name="…"]` wird für die Blockgrenze nicht ausgewertet.
- **SWITCHCASEMACHINE:** Inhalt zwischen `[SWITCHCASEMACHINE]` und der nächsten Zeile, die (getrimmt) mit `[` beginnt — wird an `parseSwapRules()` (Abschnitt 5.5, Syntax `Suchtext->Ersatztext`/`§Suchtext§Ersatztext§`) übergeben.
- **CALCMACHINE:** Inhalt zwischen `[CALCMACHINE]` und der nächsten Zeile, die mit `[` beginnt — je Zeile eine Regel `Achse*Faktor` (z. B. `Y*-1`, `X*1`), geparst zu `{axis, factor}`. Zeilen außerhalb dieses einfachen Schemas werden stillschweigend ignoriert (siehe 5.10).
- **SWIVELANGLE:** eine optionale, einzelne `SWIVELANGLE=<Zahl>`-Zeile direkt innerhalb des Blocks — siehe 5.12.
- **IJMODE:** eine optionale, einzelne `IJMODE=incremental`- bzw. `IJMODE=absolute`-Zeile direkt innerhalb des Blocks — siehe 5.12a.
- **CORNERRADIUS:** eine optionale, einzelne `CORNERRADIUS=<Marker>`-Zeile direkt innerhalb des Blocks, die den Eckenradius-Erkennungsmarker dieses Maschinentyps festlegt — siehe 5.12b.

**Bewusst tippfehler-tolerant:** Für die Abschnittsgrenzen zählt nur, dass die **nächste** Zeile mit `[` beginnt — der genaue Inhalt der schließenden Markierung wird nicht geprüft (z. B. wird `[/SWITCHCASEMACHIN]` ohne „E" oder `[/CALCMACHINE ]` mit Leerzeichen genauso behandelt wie die korrekte Schreibweise). Das entspricht der sonstigen Philosophie dieses Projekts (tolerant gegenüber N-Nummern, Leerraum-Varianten etc.).

### 5.3 Achsberechnung (`applyAxisCalc`)

Anders als die reinen Suchtext→Ersatztext-Regeln muss CALCMACHINE tatsächlich mit dem Zahlenwert rechnen. `CNCSwap.applyAxisCalc(text, calcRules)` funktioniert für beliebige Achsbuchstaben und beliebige Faktoren:

- **Faktor 1:** No-op — die Achse wird gar nicht angefasst (auch keine Neuformatierung).
- **Faktor -1:** Rein textuelle Vorzeichenumkehr — Achsbuchstabe, Zwischenraum und die exakte Ziffernfolge/Nachkommastellen bleiben erhalten, kein Umweg über `parseFloat`/`toString` (verlustfrei z. B. für `100.500`). Der Wert `0` bleibt vorzeichenlos (`-0.000` → `0.000`).
- **Jeder andere Faktor** (z. B. `2`, `0.5`, `-2`, …): echte Neuberechnung (`parseFloat(num) * factor`), formatiert mit derselben Anzahl an Nachkommastellen wie im Original; auch hier wird `-0`/`-0.000…` zu vorzeichenloser Null normalisiert.

`negateYZ(text)` ist ein dünner, weiterhin getesteter Wrapper über `applyAxisCalc(text, [{axis:'Y',factor:-1},{axis:'Z',factor:-1}])`.

Wie `applySwapRules` (5.5) lässt auch `applyAxisCalc` einen Achswert unverändert, wenn er (auch nur teilweise) innerhalb eines Kommentarbereichs liegt (siehe 5.11); die Kommentar-Maske wird dabei je Achsregel neu aus dem aktuellen Zwischenstand berechnet, um Positions-Verschiebungen durch bereits vorgenommene Ersetzungen korrekt zu berücksichtigen.

**Automatische Mitspiegelung der I/J/K-Bogenparameter:** Für jede CALCMACHINE-Regel auf `X`, `Y` oder `Z` wendet `applyAxisCalc` denselben Faktor zusätzlich auf den zugehörigen G02/G03-Bogenparameter-Buchstaben an — `I` (parallel zu `X`), `J` (parallel zu `Y`) bzw. `K` (parallel zu `Z`, aktuell ungenutzt, siehe 9.1: nur die XY-Ebene wird unterstützt). Das geschieht automatisch und ist nicht als eigene CALCMACHINE-Zeile sichtbar — es sei denn, ein Maschinentyp-Block definiert bereits selbst eine eigene `I`-, `J`- oder `K`-Regel; dann hat diese explizite Regel Vorrang und die automatische Ableitung entfällt für diesen Buchstaben. Hintergrund: I/J sind per Konvention Koordinaten/Offsets parallel zu X/Y (siehe 5.12a/9.1) — wird eine Achse hier gespiegelt oder skaliert, muss der zugehörige Bogenparameter zwingend denselben Faktor erhalten, sonst bezieht sich der daraus berechnete Bogenmittelpunkt (ob per `IJMODE=absolute` oder `IJMODE=incremental`) noch auf das ungespiegelte Koordinatensystem — ein geometrisch falscher Mittelpunkt, der sich bei großen I/J-Werten (große Radien, kurze Sehnen) dramatisch als scheinbar „riesengroßer Kreis" statt als kurzes Bogenstück bemerkbar machen kann. Für alle Maschinentypen ohne Achsspiegelung (Faktor `1` auf X/Y/Z) ist diese Mitspiegelung ein reines No-op.

**Automatischer Drehsinn-Tausch bei Achsspiegelung (`computeArcChiralityFlip`):** Eine Achsspiegelung in CALCMACHINE kehrt geometrisch zwingend auch den Drehsinn (Uhrzeigersinn/Gegenuhrzeigersinn) jedes G02/G03-Bogens um — ein Bogenmittelpunkt kann noch so korrekt (dank I/J-Mitspiegelung oben) berechnet sein, mit dem falschen Drehsinn tessellieren `tessellateArc()`/`arcSweepAngle()` trotzdem den jeweils FALSCHEN der beiden möglichen Bögen zwischen denselben zwei Punkten um denselben Mittelpunkt — bei großem Radius und kurzer Sehne sichtbar als nahezu vollständig durchlaufener, riesiger Kreis statt des kurzen, sanften Bogenstücks (exakt dasselbe äußere Symptom wie ein falscher Mittelpunkt). `CNCSwap.computeArcChiralityFlip(calcRules)` ermittelt daher automatisch aus dem VORZEICHEN der CALCMACHINE-Faktoren auf `X` und `Y` (nur das Vorzeichen zählt, nicht die Größe; `Z` bleibt unberücksichtigt, da Bögen nur in der XY-Ebene unterstützt werden), ob ein Tausch nötig ist: Ist genau EINE der beiden Achsen gespiegelt (Faktor negativ), muss der Drehsinn getauscht werden (Ergebnis `true`); sind beide oder keine gespiegelt, bleibt der Drehsinn unverändert (`false`) — eine Spiegelung beider Achsen gleichzeitig entspricht geometrisch einer 180°-Drehung, keiner Spiegelung. Angewendet wird das Ergebnis in `extractMotion()` (9.1): Direkt nach `scanArcDirection(line)` wird `cw`/`ccw` bei Bedarf vertauscht, bevor `interpolateArc()` aufgerufen wird — `interpolateArc()` selbst bleibt dadurch weiterhin vollständig frei von jeder Maschinentyp-Kenntnis. `setScope()` löst den für das aktuell aktive Maschinentyp geltenden Wert einmalig auf (`activeChiralityFlip`, aus denselben `calcRules` wie `activeIJMode`) und reicht ihn an jeden `extractMotion(...)`-Aufruf weiter. Für alle Maschinentypen ohne Achsspiegelung ist auch dies ein reines No-op. Diese automatische Ableitung ersetzt eine früher nötige, von Hand in SWITCHCASEMACHINE gepflegte Textregel (bei Biesse z. B. `G2->G3`/`G3->G2`) — eine solche Regel ist jetzt überflüssig und müsste, wenn sie dennoch im selben Maschinentyp-Block vorhanden ist, den Drehsinn ein zweites Mal tauschen (wieder falsch!), da sich beide Mechanismen sonst gegenseitig aufheben.

### 5.4 Voraberkennung/Vorschlag

`suggestMachineType(rawText, configuredNames)` (in `app.js`, reine Vorschlagslogik ohne Anwendung) ermittelt beim Laden eines neuen Programms eine Empfehlung für die Dropdown-Vorauswahl:

1. `CNCParser.detectMachine()` liefert `'SCM'|'HOMAG'|'FORMAT4'|'MOROFF'|'KRC'|'CIX'|'UNKNOWN'`; wird auf einen passenden konfigurierten Namen gemappt, falls einer existiert (`CIX` → Programme mit Biesse-typischer `.CIX`-Kopfzeile).
2. Fallback, falls (1) nichts Passendes liefert: einfache, case-insensitive Teilstring-Suche jedes konfigurierten Maschinentyp-Namens in den ersten 100 Zeilen des Programmtexts.
3. Sonst „ISO" (kein Vorschlag).

Der Vorschlag setzt nur die Dropdown-Auswahl, **nicht** den tatsächlich angewendeten Zustand — dieser bleibt bis zum Klick auf „Ok" bei „ISO" (siehe 5.1).

### 5.5 Regel-Syntax (SWITCHCASEMACHINE)

Eine Regel je Zeile, in einem von zwei Schemata (auch gemischt):

1. **Altes Schema:** `§Suchtext§Ersatztext§`
2. **Neues Schema:** `Suchtext->Ersatztext` — steht nach `->` nichts oder nur `/`, wird der Suchtext gegen nichts getauscht (entfernt). Führende/nachgestellte Leerzeichen in Such- und Ersatztext werden **nicht** getrimmt (bedeutsam, z. B. `O ->G92`).

`parseSwapRules(text)` verwendet eine `Map` (Suchtext → Ersatztext): Kommt derselbe Suchtext mehrfach vor, gewinnt der **letzte** Eintrag im jeweiligen SWITCHCASEMACHINE-Abschnitt. `applySwapRules(text, rules)` wendet alle Regeln in **einem** Durchlauf von links nach rechts an — an jeder Position wird unter allen passenden Regeln die mit dem **längsten** Suchtext gewählt („Maximal-Munch", nicht die Datei-Reihenfolge); ersetzter Text wird nicht erneut gescannt. Die Regeln werden auf die **gesamte** Rohdatei angewendet, **bevor** der Arbeitsgang-Parser läuft (Ausnahme: die Struktur-Erkennung selbst läuft dank `detectionText`, siehe 4.5, auf dem ungetauschten Text). Ein an sich passender Treffer wird zusätzlich übersprungen, wenn er (auch nur teilweise) innerhalb eines Kommentarbereichs liegt (siehe 5.11).

### 5.6 Standard-Maschinentypdatei

Beim ersten Programmstart (bzw. wenn noch nichts in localStorage gespeichert ist, siehe 5.8) ist eine fest eingebettete Standard-Konfiguration aktiv (`DEFAULT_MACHINETYPES_TEXT`, alphabetisch nach Name sortiert). Aktueller Stand — zehn Maschinentyp-Blöcke:

| Maschinentyp | SWITCHCASEMACHINE | CALCMACHINE |
|---|---|---|
| Biesse | 12 Regeln (u. a. `C1=->C`, `B1=->A`, `CR=->R`, `OPTI->BLA`, MACRO-Block-Entfernung, `G42`↔`G41`-Tausch) | `X*1`, `Y*-1`, `Z*-1`, `C*-1` (zusätzlich `IJMODE=absolute` und `CORNERRADIUS=BR=` außerhalb von CALCMACHINE, siehe 5.12a/5.12b) |
| FORMAT4 | 2 Regeln (`G77H->M6 T`, `G59->G92`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `CORNERRADIUS=E` außerhalb von CALCMACHINE, siehe 5.12b) |
| HolzHer | 5 Regeln (`CALL _DINISO ( VAL CODE:='->/`, `')->/`, `G03->G3`, `G02->G2`, `R=->R`) | `X*1`, `Y*1`, `Z*1` |
| HOMAG | 16 Regeln (u. a. `CR=->R`, `SUPA/G153/…A->/`, `G03->G3`, `G02->G2`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `CORNERRADIUS=G302 I` außerhalb von CALCMACHINE, siehe 5.12b) |
| Houfek | 5 Regeln (u. a. `Z1=->Z`, `Z2=->Z`) | `X*1`, `Y*1`, `Z*1` |
| KRC | keine Regeln (leere SWITCHCASEMACHINE) | `X*1`, `Y*1`, `Z*1`, `C*-1` (zusätzlich `IJMODE=incremental` außerhalb von CALCMACHINE, siehe 5.12a) |
| MAKA | 4 Regeln (`G03->G3`, `G02->G2`, `B->A`, `A->C`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `CORNERRADIUS=P71:` außerhalb von CALCMACHINE, siehe 5.12b) |
| MKM | 2 Regeln (`G03->G3`, `G02->G2`) | `X*1`, `Y*1`, `Z*1` |
| Reichenbacher | 1 Regel (`TRANS->G92`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `SWIVELANGLE=45` außerhalb von CALCMACHINE, siehe 5.12) |
| SCM | 15 Regeln (u. a. `G13D->G1`, `G03D->G0`, `H->Z`, `N X->` [entfernt „N X"], `Q->C`, `R->A`, `C=2->G41`, `PB->;PB`, `XN X=->;X=`, `X=-->;X=`) | `X*1`, `Y*1`, `Z*1` (zusätzlich `IJMODE=absolute` und `CORNERRADIUS=GFIL r=` außerhalb von CALCMACHINE, siehe 5.12a/5.12b) |

**Biesse** und **KRC** sind aktuell die einzigen beiden Maschinentypen mit einer von 1 abweichenden CALCMACHINE-Berechnung — bei den übrigen acht dient CALCMACHINE (Stand der Standard-Datei) noch keiner tatsächlichen Umrechnung. Der Benutzer kann Regeln jederzeit über „Maschinentypen bearbeiten" erweitern.

**Der KRC-Block:** Leere SWITCHCASEMACHINE bedeutet kein Textaustausch, nur die Achsberechnung wirkt (`X*1`, `Y*1`, `Z*1`, `C*-1`). Der Name „KRC" ist bewusst unabhängig vom bereits bestehenden, gleichnamigen `detectMachine()`-Erkennungsstil in `parser.js` (KUKA-Robotersyntax, erkannt an der Kopfzeile „(KRC ...)", siehe 4) — beide heißen zufällig gleich, sind aber zwei komplett unabhängige Mechanismen: der eine erkennt automatisch die Arbeitsgang-Struktur beim Laden, der andere ist dieser hier manuell im Dropdown ausgewählte Austausch-/Achsberechnungs-Maschinentyp. Nach Auswahl + „Ok" wird ausschließlich das Vorzeichen der Achse C umgekehrt (X/Y/Z bleiben unverändert, da Faktor 1 laut `applyAxisCalc()` als No-Op behandelt wird, siehe 5.3), Zurückschalten auf „ISO" stellt den Originaltext wieder her.

**Der MAKA-Block:** Der SWITCHCASEMACHINE-Abschnitt besteht aus den vier Regeln `G03->G3`, `G02->G2`, `B->A` und `A->C`; CALCMACHINE bleibt `X*1`/`Y*1`/`Z*1`. Das MAKA-Dialekt-eigene Eckenverrundungs-Schlüsselwort „P71:" (siehe 9.1a) ist dabei bewusst **keine** eigene SWITCHCASEMACHINE-Regel — es wird direkt von `parseCornerRadius()`/`applyCornerFillets()` aus der Programmzeile selbst ausgewertet (bzw. über den konfigurierten `CORNERRADIUS=P71:`-Marker, siehe 5.12b). Eine textuelle Ersetzung wäre hier ungeeignet: SWITCHCASEMACHINE-Regeln laufen auf dem gesamten Rohtext, **bevor** überhaupt geparst wird (siehe 5.5) — eine Regel, die „P71:" gegen irgendeinen anderen Text tauscht, würde die literale Markierung bereits entfernt haben, bevor die Eckenradius-Erkennung sie zu Gesicht bekommt, und die Eckenverrundung dadurch lautlos wirkungslos machen.

Beim Biesse-Block bilden die beiden ersten Regeln `C1=->C`/`B1=->A` die Kippachse des zugehörigen Sägeaggregats rein textuell VOR dem eigentlichen Parsen ab — aus „B1=-90.01" wird „A-90.01", aus „C1=-12.49" wird „C-12.49" (klassische, direkt unterstützte Achsworte-Notation, siehe `AXIS_RE`/9.1). Laut dieser Maschinentyp-Definition entspricht der Kippkanal „B1" dieses Aggregats kinematisch der Kipp-Achse **A** (Rotation um X, siehe `toolAxisDirection()`) des Simulators, nicht der Kipp-Achse B — eine reine Konvention dieser konkreten Maschine, die der generischen Fallback-Interpretation ohne gewählten Maschinentyp (dort wird „B1=" mangels besseren Wissens direkt auf die gleichnamige Achse B abgebildet) widerspricht; sobald „Biesse" im Dropdown gewählt und mit „Ok" bestätigt wird, gewinnt diese textuelle Vorab-Übersetzung, da `applySwapRules()` vor dem eigentlichen Parsen läuft.

Der Biesse-Block enthält bewusst **keine** eigene SWITCHCASEMACHINE-Regel für den Drehsinn (`G2`↔`G3`) mehr: CALCMACHINE spiegelt dort die Y-Achse (`Y*-1`), und diese Spiegelung kehrt geometrisch zwingend sowohl den Drehsinn jedes G02/G03-Bogens als auch die Lage seines I/J-Mittelpunkts um — beides wird seit 5.3 vollautomatisch aus den CALCMACHINE-Faktoren abgeleitet (I/J-Mitspiegelung UND `computeArcChiralityFlip`), eine von Hand gepflegte Textregel ist dafür nicht mehr nötig und würde, wäre sie trotzdem vorhanden, den Drehsinn ein zweites Mal (und damit wieder falsch) tauschen. Frühere Versionen dieses Blocks enthielten testweise `G2->G3`/`G3->G2` als Textregel, bevor beide Mechanismen automatisiert wurden (siehe 9.1 zur geometrischen Verifikation gegen ein reales Beispielprogramm mit sehr großen I/J-Radien). Der Biesse-Block enthält ebenfalls bewusst **keine** eigene SWITCHCASEMACHINE-Regel für den Eckenradius-Marker „BR=" (z. B. „BR=->R") — eine solche Regel würde die Markierung bereits vor der Eckenradius-Erkennung zerstören, exakt aus demselben Grund wie beim MAKA-Block oben; die Erkennung läuft stattdessen über den konfigurierten `CORNERRADIUS=BR=`-Marker (siehe 5.12b, 9.1d).

Die HolzHer-Regel `R=->R` (Release 128, Sebastian: "HolzHer hinzufügen bei der Austauschdatei: R=->R") entfernt bei einem im HolzHer-Dialekt mit Gleichheitszeichen notierten Bogenradius (z. B. „R=25.000") genau dieses Zeichen, sodass „R25.000" entsteht — exakt dasselbe Prinzip wie die bereits bestehende `CR=->R`-Regel bei Biesse/SCM (siehe oben): Das R-Format von G02/G03 (`ARC_PARAM_RE = /R\s*(-?\d+(?:\.\d+)?)/g`, siehe 9.1) erlaubt zwar optionalen Zwischenraum zwischen „R" und der Zahl, aber kein „=" dazwischen — ohne diese Regel würde „R=25.000" von `parseArcParams()` nicht als Bogenradius erkannt und der Bogen fiele auf eine gerade Linie zurück.

Beim SCM-Block kommentieren die drei letzten Regeln `PB->;PB`, `XN X=->;X=` und `X=-->;X=` bestimmte Zeilen(-anfänge) gezielt aus, statt sie zu ersetzen — z. B. wird aus „XN X=0 Q=0 IF=FLD=14" ein „;X=0 C=0 IF=FLD=14". `XN X=->;X=` ist dabei bewusst **länger und spezifischer** als die bereits bestehende, kürzere Regel `N X->` (entfernt „N X"): Ohne die längere Regel würde bei „XN X=…" wegen „Maximal-Munch an der am weitesten links liegenden Position" (siehe 5.5) die kürzere `N X`-Regel zuerst zuschlagen und dabei das „X" „aufessen", das eine reine `X=->;X=`-Regel an dieser Stelle zum Feuern noch gebraucht hätte — ein Beispiel für die in 5.10 beschriebene Klasse von Regel-Kollisionen durch literale Teilstring-Überschneidung. Die `IJMODE=absolute`-Zeile des SCM-Blocks (siehe 5.12a) ist geometrisch gegen ein reales SCM-Beispielprogramm verifiziert: Der aus I/J gebildete Bogenmittelpunkt stimmt für jeden darin enthaltenen G02/G03-Bogen nur bei absoluter Interpretation mit dem Abstand zu Start- UND Zielpunkt überein (Abweichung < 0,001mm statt teils >50mm bei inkrementeller Interpretation) — ohne diese Zeile hätte der global geltende Standard `incremental` gegriffen und an jedem Bogen einen deutlich falschen Mittelpunkt erzeugt, sichtbar als Knick/„Zacken" in der sonst glatten, großradiusigen Kontur.

### 5.7 Dropdown-Inhalt, Sortierung, Duplikat-Warnung

Der Dropdown-Inhalt wird bei jedem Neuaufbau (`populateMachineTypeSelect()`) **dynamisch** aus den `Name="…"`-Werten der aktuellen Konfiguration gebildet — kein hartkodierter, fester Namenskatalog. „ISO" steht dabei immer fest an erster Stelle, alle übrigen Namen folgen automatisch alphabetisch sortiert (case-insensitiv), unabhängig von ihrer Reihenfolge in der Konfigurationsdatei. Das Dropdown samt „Ok"-Knopf steht als hervorgehobene `.machine-type-quick-group` (fett/großgeschrieben in Akzentfarbe) direkt im Menüband, siehe Abschnitt 10.

Beim Bearbeiten der Konfiguration (Panel „Maschinentypen bearbeiten") prüft `CNCSwap.findDuplicateMachineTypeNames()` auf mehrfach vergebene Namen (Vergleich case-insensitiv) und zeigt bei einem Treffer eine sichtbare, aber **nicht blockierende** Warnung im Panel an (`#machineTypeDupWarning`) — das Speichern selbst wird dadurch nicht verhindert.

**Alphabetische Sortierung des rohen Konfigurationstexts selbst** (zu unterscheiden von der oben beschriebenen Dropdown-Sortierung): `CNCSwap.sortMachineTypeBlocksAlphabetically(configText)` ordnet die kompletten Block-**Text-Ausschnitte** (vom öffnenden `[MACHINETYPE Name="…"]`-Marker bis zum nächsten Block bzw. Dateiende, jeweils inklusive ihres gesamten Inhalts byte-identisch) alphabetisch nach Name um (case-insensitiv) — reine Verschiebung ganzer Blöcke, kein Eingriff in deren Inhalt. Etwaiger Text vor dem allerersten Block bleibt unverändert oben stehen; bei weniger als zwei Blöcken bzw. leerem/fehlendem Text bleibt der Text unverändert. Angewendet wird diese Umsortierung beim Import einer Einstellungsdatei über „Einstellungen importieren" (5.9 unten) auf das darin enthaltene `machineTypesConfigText`. **Nicht** automatisch umsortiert wird dagegen der aus localStorage wiederhergestellte Text bei einem gewöhnlichen Seiten-Reload sowie der Text, während er im „Maschinentypen bearbeiten"-Panel von Hand editiert wird.

Die frühere Möglichkeit, die komplette Maschinentyp-Konfiguration über einen eigenen „Maschinentypen laden"-Dateidialog zu ersetzen, entfällt — dieselbe Funktion ist bereits vollständig über „Einstellungen importieren" (siehe unten) abgedeckt, dessen JSON-Datei `machineTypesConfigText` vollständig enthält. Das Untermenü „Maschinentypen" (Abschnitt 10) besteht deshalb nur noch aus dem einen Eintrag „Maschinentypen bearbeiten" — seit Release 134 enthält das dadurch geöffnete Panel selbst zusätzlich einen „Maschinentypen zurücksetzen"-Knopf (siehe 5.7b).

### 5.7a Öffnen von „Maschinentypen bearbeiten": Sprung zum aktiven Block, Fenstergröße

**Sprung zum aktiven Block** (Release 128, Sebastian: "Wenn bei Maschinentyp ein Maschinentyp gewählt worden ist (z.B. Holzher) und man ins Menü ... 'Maschinentyp bearbeiten' geht, soll das Menü direkt da hinspringen was als Maschinentyp ausgewählt UND mit Ok bestätigt worden ist."): Beim Öffnen des Panels wird Cursor und Scroll-Position direkt auf den Block des aktuell **aktiven** Maschinentyps gesetzt — `activeMachineTypeName`, also der zuletzt per „Ok" bestätigte Name (siehe 5.1), nicht nur eine Dropdown-Vorauswahl ohne Bestätigung. Gesucht wird case-insensitiv nach dem öffnenden `[MACHINETYPE Name="…"]`-Marker im aktuellen Konfigurationstext. Ist kein Maschinentyp aktiv („ISO") oder wird der Marker nicht gefunden (z. B. der Block wurde zwischenzeitlich von Hand umbenannt), bleibt der Cursor wie bisher am Anfang stehen — ein reiner Komfort-Fallback, kein Fehler.

**Fenstergröße** (Release 128, Sebastian: "Das Maschinentyp bearbeiten-Fenster soll auch ca. 30% größer sein in der Länge nach oben und unten (jeweils 15%)"): Das Textfeld `#machineTypeEditArea` ist mit `height: 286px` (220px · 1,3) ca. 30 % höher als die gemeinsame Basishöhe `.paste-card textarea` (220px, unverändert für die vier übrigen Editor-Panels — Code einfügen, Programm bearbeiten, Arbeitsgang-Namen, Austauschdatei). Da `.paste-panel` den Dialog über Flexbox zentriert (`align-items`/`justify-content: center`), wächst die höhere Karte beim Zentrieren automatisch symmetrisch nach oben und unten — die gewünschten „jeweils 15%" ergeben sich dadurch von selbst, ohne eigene Positionierungslogik.

### 5.7b „Maschinentypen zurücksetzen"-Knopf

Release 134. Hintergrund: Sebastian löschte die gecachten Maschinentyp-Einstellungen bislang manuell über den Umweg „exe schließen → HTML-Datei überschreiben → WebView2-Profilordner (`CNCSimX.exe.WebView2`) löschen", um nach einem HTML-Austausch sicher wieder bei den eingebauten Standard-Maschinentypen zu landen (siehe 5.8 zur localStorage-Pfadbindung). Auf die Frage, ob sich das auch per eingebautem Knopf nachbilden lässt, wurden drei Varianten angeboten (nur Maschinentypen zurücksetzen / ein allgemeiner „Cache leeren" für alle gespeicherten Einstellungen zusammen / beides als getrennte Knöpfe); Sebastian wählte die erste, engste Variante.

**Knopf** `#btnMachineTypeResetDefaults` sitzt direkt im Panel „Maschinentypen bearbeiten", links von „Abbrechen"/„Übernehmen" abgesetzt (per `margin-right: auto` innerhalb der sonst rechtsbündigen `.paste-actions`-Knopfzeile), damit ein hastiger Klick ihn nicht mit den beiden häufiger benötigten Knöpfen verwechselt. Klick öffnet zunächst denselben nativen `window.confirm()`-Sicherheitsdialog wie „Standardwerte wiederherstellen" (12.3): „Alle Maschinentypen werden auf die eingebauten Standardwerte zurückgesetzt, eigene Änderungen an den Maschinentypen gehen dabei verloren. Fortführen?" — bei „Abbrechen" (Nein) passiert nichts.

Bei „Ok" (Ja) führt `resetMachineTypesToDefaults()` genau das aus, was vorher nur über den WebView2-Profilordner-Umweg erreichbar war:

1. Entfernt `cncsim.machineTypesConfigText` UND `cncsim.activeMachineTypeName` aus localStorage (`window.localStorage.removeItem(...)`, nicht etwa ein Überschreiben mit dem Standardtext) — dieselbe „entfernen statt überschreiben"-Architektur wie bei `resetAllDisplaySettingsToDefaults()` (12.3), damit `loadPersistedMachineTypesConfig()`/`loadPersistedActiveMachineTypeName()` (5.8) beim nächsten Zugriff nachweislich keinen eigenen Eintrag mehr vorfinden und auf `DEFAULT_MACHINETYPES_TEXT`/`DEFAULT_ACTIVE_MACHINETYPE_NAME` zurückfallen.
2. Setzt `machineTypesConfigText`/`machineTypes` sofort auch im laufenden Programmzustand auf die eingebauten Standardwerte (`CNCSwap.parseMachineTypes(DEFAULT_MACHINETYPES_TEXT)`) — kein Neuladen der Seite nötig, um die Wirkung zu sehen.
3. Füllt das Textfeld `#machineTypeEditArea` mit dem wiederhergestellten Standardtext neu, prüft erneut auf doppelte Namen (`checkMachineTypeDuplicates()`) und baut das Dropdown (`populateMachineTypeSelect()`) neu auf `DEFAULT_ACTIVE_MACHINETYPE_NAME` ('ISO' im eingebauten Standard) auf.
4. Zeigt, falls bereits ein Programm geladen ist, dieses sofort erneut mit dem jetzt aktiven Standard-Maschinentyp an (`displayCurrentProgram()`) und schließt das Panel — derselbe Ablauf wie beim regulären „Übernehmen"-Klick, nur mit `DEFAULT_MACHINETYPES_TEXT`/`DEFAULT_ACTIVE_MACHINETYPE_NAME` statt dem Textarea-Inhalt als Quelle.

**Bewusst abgegrenzter Geltungsbereich:** Betrifft ausschließlich die Maschinentyp-Konfiguration. Die Arbeitsgang-Namensliste (Abschnitt 6), die Knowledge Base (10.1a) sowie sämtliche Anzeigeeinstellungen (Farben/Spaltenbreiten, 12.3) bleiben unangetastet — exakt umgekehrt zu „Standardwerte wiederherstellen" (12.3), das wiederum ausdrücklich NICHT die Maschinentypen betrifft. Beide Reset-Mechanismen zusammen decken damit weiterhin nicht „alles auf einmal" ab; wer wirklich den Zustand eines komplett frischen Browserprofils will, braucht nach wie vor den ursprünglichen WebView2-Profilordner-Umweg (oder künftig einen dritten, noch nicht gebauten „alles zurücksetzen"-Knopf).

**Bezug zur localStorage-Pfadbindung (5.8/5.13):** Da `localStorage` unter `file://` strikt an den Dateipfad gebunden ist, war das eigentliche Problem hinter Sebastians Ausgangsfrage nie ein echtes „Caching" der HTML-Datei selbst, sondern ausschließlich der unter demselben Pfad weiterhin gespeicherte alte `cncsim.machineTypesConfigText`-Eintrag, der eine neu eingespielte HTML-Version mit abweichenden eingebauten Standard-Maschinentypen beim Start überdeckt. Dieser Knopf löst genau dieses eine Teilproblem direkt in der Anwendung, ohne den exe-Neustart- bzw. Profilordner-Umweg.

### 5.8 localStorage-Persistenz

Gespeichert werden der Text der Maschinentyp-Konfiguration (`cncsim.machineTypesConfigText`), der Name des zuletzt per „Ok" bestätigten Maschinentyps (`cncsim.activeMachineTypeName`), die Arbeitsgang-Namensliste (`cncsim.opNameListText`, siehe Abschnitt 6) sowie die Knowledge Base (`cncsim.knowledgeBaseText`, siehe 10.1a) — **nicht** das geladene CNC-Programm selbst, das geht beim Neuladen der Seite weiterhin verloren. Alle Lese-/Schreibzugriffe sind per `try`/`catch` abgesichert und fallen bei fehlender Verfügbarkeit (z. B. mancher privater Browser-Modus) einfach auf Standardwerte zurück, ohne die Anwendung zu beeinträchtigen.

### 5.9 Einstellungen exportieren/importieren

„Einstellungen exportieren" (`#btnSaveSettings`) und „Einstellungen importieren" (`#btnLoadSettings`, beide im „Einstellungen"-Menü, siehe Abschnitt 10) ergänzen die localStorage-Persistenz um einen expliziten Datei-Export/-Import, z. B. um dieselben Einstellungen auf einen anderen Rechner/Browser zu übertragen.

„Einstellungen exportieren" lädt sofort (ohne Zwischendialog) eine JSON-Datei `cncsim_einstellungen.json` mit dem aktuellen Stand herunter — analog zu „Speichern als…" über einen unsichtbaren `<a download>`-Link:

```json
{
  "version": 1,
  "opNameListText": "…",
  "machineTypesConfigText": "…",
  "activeMachineTypeName": "Biesse",
  "knowledgeBaseText": "…",
  "colorProfileText": "…",
  "columnScaleCode": 100,
  "columnScaleOps": 1,
  "menuHighlightBg": "#ad1f2d",
  "menuHighlightInk": "#ffffff",
  "fontColors": { "code": "#00ff00" },
  "opColors": ["#ad1f2d", "#2b6674", "#1f8a52", "#8a4bb0", "#c9922a", "#1f9e9e", "#b5507a", "#5c6bc0", "#7a8a3d", "#a15c2e"]
}
```

Enthalten sind die Arbeitsgang-Namensliste (Abschnitt 6), die komplette Maschinentyp-Konfiguration (Abschnitt 5), der Name des zuletzt bestätigten Maschinentyps, die Knowledge Base (`knowledgeBaseText`, siehe 10.1a), das Farbprofil (12.1), die beiden „Spalteneinstellungen"-Werte (`columnScaleCode`/`columnScaleOps`, siehe 7.3a), die Menü-Hervorhebungsfarbe (`menuHighlightBg`/`menuHighlightInk`, siehe 12.2), die gesetzten Schriftfarben (`fontColors`, siehe 12.3) und die Arbeitsgangfarbe (`opColors`, zehn Einträge, siehe 12.4) — bewusst **nicht** das geladene CNC-Programm oder eine importierte DXF-Unterlage (dieselbe Abgrenzung wie bei der localStorage-Persistenz).

**`opColors`:** entweder `null` (keine eigene Übersteuerung aktiv, „automatisch"/Theme-Standard — dieselbe Konvention wie bei „Hintergrundfarbe", 12.1) oder ein Array aus genau zehn Hexcodes (1. bis 10. Arbeitsgang, in dieser Reihenfolge). Export schreibt stets den tatsächlich aktuell wirksamen Stand (`activeOpColors`), Import übernimmt ein vorhandenes Feld nur, wenn es entweder `null` oder ein Array aus genau `OP_COLOR_COUNT` (10) Einträgen ist — ein Array anderer Länge wird dabei bewusst ignoriert, OHNE `applyOpColors()` überhaupt erst aufzurufen, damit die aktuell aktive Arbeitsgangfarbe garantiert unangetastet bleibt statt versehentlich auf „automatisch" zurückzufallen — fehlt das Feld komplett (ältere/handgeschriebene Datei), gilt dieselbe Rückwärtskompatibilitäts-Regel wie bei den übrigen Einstellungsteilen.

„Einstellungen importieren" öffnet eine solche Datei wieder (Dateiauswahl über ein verstecktes `<input type="file">`) und wendet alle enthaltenen Teile über dieselben Pfade an, die auch bei manueller Änderung benutzt werden (`applyOpNameList()`, `applyKnowledgeBaseText()`, `CNCSwap.parseMachineTypes()`, `applyColorProfile()`, `normalizeColumnScale()`+`applyProgCollapseStage()`/`setOpsWidth()`, `applyMenuHighlightColor()`, `applyFontColors()`, `applyOpColors()` — jeweils + Persistieren) — die importierten Werte werden dadurch sofort wirksam und zusätzlich in localStorage übernommen. Fehlt ein Feld in der geladenen Datei, bleibt der jeweils aktuell aktive Wert unverändert, statt auf einen Standardwert zurückzufallen. Ist die Datei kein gültiges JSON, erscheint ein einfacher Warnhinweis, ohne etwas zu verändern.

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

`S` liefert `getReichenbacherSwivelAngleDeg()`: Sucht den aktuell **aktiven** Maschinentyp (`activeMachineTypeName`) in `machineTypes` und liest dessen `swivelAngleDeg` — ist der Typ nicht auffindbar oder `swivelAngleDeg` `null`/keine gültige Zahl, wird auf **45°** zurückgefallen. Eine geänderte `SWIVELANGLE`-Zeile wirkt sich unmittelbar nach „Übernehmen" in „Maschinentypen bearbeiten" auf die 3D-Darstellung aus, ohne dass das Programm neu geladen werden muss.

**Namenserkennung des Reichenbacher-Schalters — Teilstring, nicht Exaktvergleich:** `isReichenbacherTypeName(name)` (`app.js`) prüft `name.toLowerCase().includes('reichenbacher')` — case-insensitive Teilstring-Suche, egal ob „Reichenbacher" am Anfang, in der Mitte oder am Ende des Namens steht (z. B. „Reichenbacher 2026", „Neu Reichenbacher"). `getReichenbacherSwivelAngleDeg()` selbst sucht dagegen exakt nach dem konkreten, aktiven Namen, welcher Name das im Einzelfall auch ist.

### 5.12a Konfigurierbare I/J-Bogenkonvention (`IJMODE`)

Bei G02/G03 im **I/J-Format** (`X.. Y.. I<Wert> J<Wert>`, siehe 9.1) ist nicht einheitlich festgelegt, ob I/J den Bogenmittelpunkt als Versatz vom Startpunkt (**inkrementell**) oder bereits als fertige, absolute Koordinate (**absolut**) angeben — das ist reine Postprozessor-Konvention und unterscheidet sich zwischen Maschinentypen. Deshalb ist diese Konvention, genau wie `SWIVELANGLE` (5.12), als einfache Direktive je Maschinentyp-Block konfigurierbar, statt fest im Code verdrahtet zu sein.

**Syntax:** Eine einzelne, einfache `IJMODE=incremental`- bzw. `IJMODE=absolute`-Zeile (Groß-/Kleinschreibung sowohl des Schlüsselworts als auch des Werts egal, Leerzeichen um `=` toleriert) **direkt innerhalb** eines `[MACHINETYPE …]`-Blocks (an beliebiger Position). `parseIJMode(blockLines)` (`swap.js`, `IJ_MODE_RE = /^IJMODE\s*=\s*(incremental|absolute)$/i`) durchsucht alle Zeilen des Blocks nach dem ersten Treffer und liefert `'incremental'`/`'absolute'` (stets klein geschrieben), bzw. `null`, wenn keine Zeile passt. `parseMachineTypes()` hängt das Ergebnis als `ijMode` an jeden Block-Eintrag an (siehe 5.2).

**Konsumiert von `interpolateArc(sx, sy, ex, ey, params, direction, ijMode)`** (`parser.js`, siehe 9.1): Bei `params.mode === 'ij'` ergibt sich der Mittelpunkt bei `ijMode === 'absolute'` direkt aus `{cx: params.i, cy: params.j}` (I/J sind die fertige Mittelpunkt-Koordinate), sonst — einschließlich `null`/fehlender Direktive — aus `{cx: sx + params.i, cy: sy + params.j}` (I/J sind ein Versatz vom Startpunkt). Der Standard ist also **`incremental`**. Den für das aktuell angezeigte Programm geltenden `ijMode` löst `setScope()` (`app.js`) einmalig aus dem aktiven Maschinentyp auf (`activeMt.ijMode || 'incremental'`, ohne eigene Namenserkennung wie bei `getReichenbacherSwivelAngleDeg()`/5.12 — eine Teilstring-Sonderregel ist hier nicht nötig) und reicht ihn an jeden `extractMotion(...)`-Aufruf weiter, die ihn unverändert an `interpolateArc()` durchreicht.

**Standard-Konfiguration:** Die drei eingebauten Maschinentypen Biesse (`IJMODE=absolute`), KRC (`IJMODE=incremental`) und SCM (`IJMODE=absolute`) tragen je eine eigene, geometrisch gegen reale Beispielprogramme verifizierte `IJMODE`-Zeile (siehe 5.6); alle übrigen Standard-Maschinentypen sowie jeder vom Benutzer selbst angelegte Maschinentyp ohne eigene `IJMODE`-Zeile verwenden weiterhin den Standard `incremental`. Eine geänderte `IJMODE`-Zeile wirkt sich — wie bei `SWIVELANGLE` (5.12) — unmittelbar nach „Übernehmen" in „Maschinentypen bearbeiten" auf ein bereits geladenes Programm aus, ohne dass es neu geladen werden muss.

### 5.12b Konfigurierbarer Eckenradius-Marker (`CORNERRADIUS`)

Jeder der in 9.1a–9.1e beschriebenen Eckenverrundungs-Dialekte (MAKA „P71:", HOMAG „G302 I", SCM „GFIL r=", Biesse „BR=", FORMAT4 „E") erkennt seinen jeweiligen Marker über einen eigenen, fest einprogrammierten regulären Ausdruck, unabhängig vom aktuell gewählten Maschinentyp — historisch bedingt, weil jeder Dialekt einzeln nachgerüstet wurde. `CORNERRADIUS=<Marker>` macht diese Zuordnung zusätzlich **maschinentyp-abhängig konfigurierbar**, als direktes Pendant zu `SWIVELANGLE` (5.12) und `IJMODE` (5.12a): Statt dass alle fünf fest einprogrammierten Erkenner bei jedem Programm unabhängig vom gewählten Maschinentyp gleichzeitig nach ihrem jeweiligen Marker suchen, kann ein Maschinentyp-Block seinen eigenen, dort tatsächlich geltenden Marker benennen.

**Syntax:** Eine einzelne, einfache `CORNERRADIUS=<Marker>`-Zeile **direkt innerhalb** eines `[MACHINETYPE …]`-Blocks (an beliebiger Position, `CORNERRADIUS_RE = /^CORNERRADIUS\s*=\s*(.+)$/i`). Das Schlüsselwort selbst ist case-insensitiv, der Marker-**Wert** dagegen bewusst case-**sensitiv** (anders als bei `IJMODE`/`SWIVELANGLE`) und wird unverändert übernommen, da die realen Marker-Schreibweisen selbst case-sensitiv sind (z. B. GFILs kleines „r=" gegenüber HOMAGs großem „I"). `parseCornerRadiusMarker(blockLines)` (`swap.js`) durchsucht alle Zeilen des Blocks nach dem ersten Treffer und liefert den getrimmten Marker-Text, bzw. `null`, wenn keine Zeile passt. `parseMachineTypes()` hängt das Ergebnis als `cornerRadiusMarker` an jeden Block-Eintrag an (siehe 5.2).

`<Marker>` ist dabei genau der Text, der im Programmcode unmittelbar vor der Radius-Zahl steht:

| Maschinentyp | Marker-Wert | Reale Notation im Programm |
|---|---|---|
| MAKA | `P71:` | `P71:7.00` |
| Biesse | `BR=` | `BR=6.00` |
| FORMAT4 | `E` | `E5.0000` |
| HOMAG | `G302 I` | `G302 I5` |
| SCM | `GFIL r=` | `GFIL r=5.000` |

Bei HOMAG und SCM reicht die bloße Befehlsbezeichnung („G302"/„GFIL") nicht aus — die Direktive steht dort auf einer **eigenen** Zeile ohne eigene Bewegung, und zwischen Befehlsname und Zahl liegt noch ein eigener Parameterbuchstabe (siehe 9.1b/9.1c); der konfigurierte Marker muss deshalb den vollständigen Text bis einschließlich dieses Parameterbuchstabens umfassen. Ein neuer, bislang unbekannter Dialekt mit einer Notation wie „CLAUDERADIUS 5.53" bräuchte dafür nur die eine Konfigurationszeile `CORNERRADIUS=CLAUDERADIUS` — keine Code-Änderung.

**Regel-Aufbau (`buildCornerRadiusMarkerRegex(marker)`, `parser.js`):** Der konfigurierte Markertext wird auf Leerzeichen in Tokens zerlegt (z. B. „G302 I" → `["G302", "I"]`), jedes Token einzeln regex-escaped (damit Sonderzeichen wie „." oder „=" in Markern wie „BR=" oder „GFIL r=" nicht als Regex-Metazeichen wirken) und mit einer nicht-gierigen Lücke `[^;]*?` zwischen den Tokens wieder zusammengesetzt — dieselbe Toleranz für dazwischenliegenden Text, die die fest einprogrammierten `G302_CORNER_RADIUS_RE`/`GFIL_CORNER_RADIUS_RE` (9.1b/9.1c) bereits für ihre jeweils zweiteiligen Marker verwenden. Ein einteiliger Marker wie „P71:"/„BR="/„E" ergibt dabei ein einzelnes escaptes Token ohne Lücke, strukturell identisch zu den jeweils fest einprogrammierten Erkennern (9.1a/9.1d/9.1e). Ein negativer Lookbehind `(?<![A-Za-z0-9])` davor (Stil wie `AXIS_RE`, siehe 9.1) verhindert, dass der Marker mitten in einem anderen Wort anschlägt. Liefert `null`, wenn kein Marker konfiguriert ist (leerer/fehlender String). `parseConfiguredCornerRadius(line, markerRegex)` wendet das gebaute RegExp auf eine Zeile an und liefert den gefundenen Zahlenwert bzw. `null` bei `markerRegex === null` oder fehlendem Treffer.

**Vorrang vor den fest einprogrammierten Erkennern, mit automatischem Rückwärtskompatibilitäts-Fallback:** `setScope()` (`app.js`) löst `activeCornerRadiusMarkerRegex` einmalig aus dem aktiven Maschinentyp auf (`CNCParser.buildCornerRadiusMarkerRegex(activeMt.cornerRadiusMarker)`, exakt dasselbe Auflösungsmuster wie `activeIJMode`/`activeChiralityFlip`, siehe 5.12a/5.3) und reicht es an jeden `extractMotion(...)`-Aufruf weiter. Innerhalb von `extractMotion()` wird auf jeder in Frage kommenden Zeile zuerst `parseConfiguredCornerRadius()` geprüft; liefert es einen Wert, hat dieser Vorrang vor sämtlichen fest einprogrammierten Erkennern (P71:/BR=/E auf derselben Zeile, G302/GFIL auf einer eigenen Zeile) — die fest einprogrammierten Erkenner werden in diesem Fall für diese Zeile gar nicht erst ausgewertet. Liefert der konfigurierte Marker dagegen `null` (kein `CORNERRADIUS=`-Marker für den aktiven Maschinentyp konfiguriert, oder die Zeile trägt zufällig trotzdem keinen Treffer dafür), fällt `extractMotion()` unverändert auf die fest einprogrammierten Erkenner zurück — ein bereits vor dieser Funktion angelegter bzw. per localStorage gespeicherter Maschinentyp ohne eigene `CORNERRADIUS`-Zeile funktioniert dadurch unverändert genau wie zuvor weiter, ohne dass der Benutzer etwas nachpflegen müsste.

**Automatische Unterscheidung Same-Line- vs. Own-Line-Marker:** Welcher der beiden strukturellen Fälle vorliegt (Marker steht auf derselben Zeile wie die Zielbewegung, wie bei P71:/BR=/E, oder auf einer eigenen, bewegungslosen Folgezeile, wie bei G302/GFIL) wird nicht gesondert konfiguriert, sondern automatisch aus der jeweiligen Zeile selbst abgeleitet: Trägt die aktuelle Zeile bereits ein echtes Achswort (`found && !isOriginReset`, siehe 9.1), wird der konfigurierte Marker im SAME-LINE-Zweig geprüft und bei einem Treffer direkt dem auf dieser Zeile neu erzeugten Punkt zugewiesen; andernfalls wird er im OWN-LINE-Zweig geprüft und bei einem Treffer rückwirkend dem zuletzt bereits erzeugten Punkt zugewiesen (wie bei G302/GFIL, siehe 9.1b/9.1c). Dieselbe Zeile kann dadurch nie doppelt ausgewertet werden.

Standardmäßig tragen die fünf bereits verifizierten Maschinentypen MAKA, Biesse, FORMAT4, HOMAG und SCM je eine eigene `CORNERRADIUS`-Zeile (siehe 5.6); alle übrigen Standard-Maschinentypen sowie jeder vom Benutzer selbst angelegte Maschinentyp ohne eigene `CORNERRADIUS`-Zeile verwenden weiterhin ausschließlich die fest einprogrammierten Erkenner. Eine geänderte `CORNERRADIUS`-Zeile wirkt sich — wie bei `SWIVELANGLE`/`IJMODE` (5.12/5.12a) — unmittelbar nach „Übernehmen" in „Maschinentypen bearbeiten" auf ein bereits geladenes Programm aus, ohne dass es neu geladen werden muss.

### 5.13 „Simulation speichern" — komplette Seite mit eingebackenen Einstellungen

Anders als „Einstellungen exportieren"/„Einstellungen importieren" (5.9, muss vom Benutzer aktiv wieder importiert werden) erzeugt „Simulation speichern" (Menü „Datei", siehe Abschnitt 10) eine **vollständig eigenständige, sofort lauffähige Kopie der kompletten Simulations-Seite selbst** als `.html`-Datei (`CNC_Simulation.html`) — der Benutzer öffnet diese Datei beim nächsten Mal einfach direkt (z. B. Doppelklick), ganz ohne Datei-Import-Schritt, und landet dort bereits mit seinen aktuellen Maschinentyp-/Arbeitsgang-Namen-Einstellungen als neuen Standardwerten, statt wieder bei den eingebauten Werksvorgaben zu starten. Diese Funktion ist eine reine Zwischenlösung, bis eine „richtige" Programm-Installation mit echter Einstellungs-Persistenz existiert.

**Umsetzung:** `PRISTINE_PAGE_HTML` (app.js, ganz oben) ist ein Schnappschuss von `document.documentElement.outerHTML`, genommen als allererste Anweisung des Scripts — also bevor auch nur eine einzige DOM-Änderung stattfinden konnte. Dieser Schnappschuss entspricht dadurch exakt der ursprünglich ausgelieferten Seite, unabhängig vom aktuellen Laufzeitzustand. `buildSavedSimulationHtml()` ersetzt darin drei mit `SIM_SAVE:*:BEGIN`/`SIM_SAVE:*:END`-Kommentaren fest markierte Konstanten-Deklarationen durch die aktuell aktiven Werte:

- `DEFAULT_MACHINETYPES_TEXT` ← aktueller `machineTypesConfigText`
- `DEFAULT_OPNAME_TEXT` ← aktueller `opNameListText`
- `DEFAULT_ACTIVE_MACHINETYPE_NAME` ← aktueller `activeMachineTypeName` (der zuletzt per „Ok" bestätigte Maschinentyp — eine im Dropdown nur ausgewählte, aber noch nicht per „Ok" bestätigte Auswahl zählt nicht)
- `DEFAULT_KNOWLEDGE_BASE_TEXT` ← aktueller `knowledgeBaseText` (siehe 10.1a)

Jeder Wert wird über `JSON.stringify()` sicher als einzeiliges, syntaktisch gültiges JS-String-Literal eingebettet. Ein fehlender Marker bricht den Vorgang nicht ab, sondern lässt den betroffenen Abschnitt einfach unverändert. Die BEGIN/END-Markerzeilen selbst bleiben in der gespeicherten Kopie erhalten, sodass auch aus einer bereits gespeicherten Kopie heraus erneut „Simulation speichern" funktioniert (verkettbar).

**Enthält bewusst NICHT** das aktuell geladene CNC-Programm (`rawProgramText`, geparste Arbeitsgänge, 3D-Ansicht-Zustand usw.) — unabhängig davon, ob beim Klick gerade ein Programm geladen ist oder nicht, da `PRISTINE_PAGE_HTML` der Schnappschuss vor jeder Programmladung ist. Der Dateiname wird beim Herunterladen identisch zum Original gewählt (`CNC_Simulation.html`).

**Hinweis zur Wirkung über WebView2-Updates hinweg:** `localStorage` wird bei `file://`-Seiten in Chromium/WebView2 über den **Dateipfad** verankert, nicht über den Dateiinhalt (siehe auch 10.3). Ein bereits per „Übernehmen" in „Maschinentypen bearbeiten" gespeicherter `localStorage`-Stand hat deshalb bei jedem künftigen Programmstart an diesem Pfad **Vorrang** vor der in einer neu eingespielten HTML-Datei eingebackenen `DEFAULT_MACHINETYPES_TEXT` — ein bloßes Überschreiben der Datei am selben Ort ändert die bereits aktive Maschinentyp-Konfiguration dadurch nicht. Um neue, in einer aktualisierten HTML eingebackene Standardwerte tatsächlich wirksam werden zu lassen, muss entweder der betroffene `localStorage`-Eintrag (`cncsim.machineTypesConfigText`) gezielt gelöscht, der Text im „Maschinentypen bearbeiten"-Panel von Hand aktualisiert, oder — innerhalb der WebView2-Hülle — das komplette WebView2-Nutzerprofil (der `*.WebView2`-Ordner neben der exe) zurückgesetzt werden; Letzteres löscht dabei zwangsläufig auch alle übrigen in 5.8 gelisteten Einstellungen (Farben, Arbeitsgang-Namen, Knowledge Base usw.), nicht nur die Maschinentypen.

## 6. Arbeitsgang-Namensliste (NAMED-Stil)

Editierbare Liste (ein Name je Zeile) für Postprozessoren ohne Trennzeilen-Konvention (siehe `MACHINES.NAMED` in Abschnitt 4.2). `parseOpNameList(text)` entfernt Leerzeilen und exakte Duplikate (Reihenfolge des ersten Vorkommens bleibt erhalten); Groß-/Kleinschreibung und Umlaute bleiben im gespeicherten Text unverändert, der Abgleich selbst erfolgt case-insensitiv. Über „Arbeitsgang-Namen bearbeiten" jederzeit erweiterbar. Standardliste, 92 eindeutige Einträge (Auszug):

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

**Persistenz:** Eine Bearbeitung über „Arbeitsgang-Namen bearbeiten" → „Übernehmen" übersteht auch ein Schließen/Neuladen der Seite (`cncsim.opNameListText` im localStorage, genau wie die Maschinentyp-Konfiguration, siehe 5.8).

## 7. Programm-Ansicht

### 7.1 Code-Liste

Das gesamte geladene Programm wird links als **eine durchgehende**, nummerierte Zeilenliste angezeigt (keine einzeln aufklappbaren Abschnitts-Boxen je Arbeitsgang/Zwischenstopp). Jede Zeile ist eingefärbt (`computeLineColors()`): sind Arbeitsgänge erkannt, erhält jede Zeile die Farbe ihres Arbeitsgangs (`pathColor(index)`, siehe Abschnitt 12); ohne erkanntes Schema greift stattdessen die Farbaufteilung nach Nullpunktverschiebungen (`splitAtOriginResets`, siehe 4.4). Sämtliche Leerzeilen (nach Trimmen, nach Anwendung des aktiven Maschinentyps) werden vollständig ausgeblendet — unabhängig davon, ob sie bereits im Original leer waren oder erst durch die Transformation entstanden sind.

Die aktuell simulierte/gescrubbte Zeile wird zusätzlich hervorgehoben (`.active-line`: Akzentfarbe + `--accent-soft`-Hintergrund) — **ohne** einen eigenen Dreh-/Kippwinkel-Hinweis direkt an dieser Zeile: Ein solcher Hinweis direkt in der Programmcode-Zeile würde als normaler Textknoten im selben Element sitzen und dadurch beim Markieren/Kopieren einer Codezeile ungewollt mit ausgewählt. Die Dreh-/Kippwinkel-Anzeige gibt es deshalb ausschließlich im Info-Fenster des 3D-Bereichs (`#readout`/`AXIS_META`, siehe 9.6) — komplett unabhängig von der Programmcode-Spalte.

Aus Performance-Gründen trägt jede `.code-line` `content-visibility: auto` mit `contain-intrinsic-size: auto 18.5px` als Platzhalterhöhe UND eine strukturell feste Höhe `flex: 0 0 18.5px; overflow: hidden` (`#tree` ist als `display: flex; flex-direction: column` mit potenziell zehntausenden `.code-line`-Elementen als Flex-Kindern aufgebaut) — dadurch muss der Flex-Container die Größenverteilung der Geschwister-Zeilen nicht neu berechnen, egal welcher Inhalt sich innerhalb einer einzelnen Zeile ändert (z. B. die Hervorhebung selbst). Die aktive Zeile wird ausschließlich über Farbe, Hintergrund und linken Rahmen hervorgehoben, nicht über `font-weight` (eine Textbreiten-/Fettschrift-Änderung in dieser großen Flex-Liste würde ansonsten einen teuren Neuaufbau der gesamten Liste erzwingen).

Ein Zeilensprung (Klick in der Arbeitsgänge-Spalte oder „Sprung in Zeile") schaltet `#tree` beim ersten Sprung dauerhaft auf die CSS-Klasse `force-layout` um (`.tree.force-layout .code-line { content-visibility: visible }`) und erzwingt einen synchronen Reflow, bevor das native `scrollIntoView({block:'center'})` springt — nötig, da Chromium bei zuvor nie gerenderten `content-visibility:auto`-Zeilen die `.tree`-Scrollhöhe sonst nur unvollständig berechnet und der Sprung an der falschen Stelle landet. Die `content-visibility:auto`-Optimierung bleibt für den Normalfall (Laden, Scrubben/Abspielen ohne Zeilensprung) vollständig erhalten; erst der erste tatsächliche Sprung „bezahlt" einmalig den vollen Layout-Preis und bleibt danach dauerhaft exakt.

`updateLineHighlight()`/`updateActiveOpHighlight()` überspringen Klassenwechsel/`scrollIntoView()` vollständig, wenn sich die betroffene Zeile bzw. der betroffene Arbeitsgang seit dem letzten Wiedergabe-Tick nicht geändert hat (z. B. bei einem tessellierten Kreisbogen mit mehreren Punkten auf derselben Quelltextzeile) — sie prüfen zuerst `lineEl !== lastActiveLineEl` bzw. `opIndex !== lastActiveOpIndex`.

Rechts neben der Kopfzeilen-Überschrift „CNC-Programmcode" (bzw. dem Reiter, siehe 7.1b) sitzt ein Edit-Icon (`#btnEditProgramIcon`, „✎", 24×24px) — öffnet über `openProgramEditPanel()` denselben Editor wie der Menüpunkt „CNC-Programm bearbeiten" (siehe Abschnitt 11); beide Auslöser sind an denselben beiden Stellen (de)aktiviert (aktiv nach dem Laden eines Programms, deaktiviert nach „CNC-Programm löschen").

### 7.1b Reiter „CNC-Programmcode" / „DXF-Layer"

Die Kopfzeile der Programmcode-Spalte ist ein Reiter-Paar (`#sidebarTabCode`/`#sidebarTabDxf`, siehe 9.8a für die vollständige Umschalt-/Prioritätslogik) statt einer reinen Überschrift — sichtbar/anklickbar abhängig davon, ob ein CNC-Programm und/oder ein DXF geladen sind.

Der Arbeitsgang-Zähler steht in der Kopfzeile der Arbeitsgänge-Spalte selbst (`Arbeitsgänge (N)`, siehe 7.2) statt in der Programmcode-Überschrift; Letztere zeigt nur bei fehlendem Arbeitsgang-Schema einen Hinweistext („Kein Arbeitsgang-Schema erkannt · gesamtes Programm").

**Programmlisten-Dropdown:** Sobald mindestens zwei Programme in der Programmliste liegen (siehe 3.2), erscheint direkt **unterhalb** dieser Kopfzeile eine eigene, volle Zeile mit der Auswahl des in der 3D-Simulation aktiven Programms (`#programListBar`) — bewusst als eigene Zeile statt als weiteres Element innerhalb der bereits engen Kopfzeile selbst. Bei nur einem oder keinem geladenen Programm bleibt diese Zeile verborgen und die App verhält sich optisch exakt wie ohne Programmliste.

### 7.2 Arbeitsgänge-Spalte

Rechts neben der Code-Liste, ein-/ausklappbar (auf 32px eingeklappt, siehe 7.3) und listet jeden erkannten Arbeitsgang als eigene Zeile mit Farbpunkt, laufender Nummer, Titel, einem „Auge"-Knopf und einem „3D"-Knopf. Die Kopfzeile (`.ops-col-head`) ist zweizeilig: oben die Titelzeile mit „Arbeitsgänge (N)" (`#opsColTitle`, N = Anzahl erkannter Arbeitsgänge) und dem Ein-/Ausklappen-Knopf, darunter auf voller Breite ein Umschalt-Knopf für „alle ausblenden"/„alle wieder einblenden".

- Klick auf die Zeile springt im Code zur Startzeile des Arbeitsgangs **und** bewegt die Wiedergabeposition (Scrubber) dorthin (`jumpToCodeLine` → `scrubToLineIdx`), sodass ein anschließendes Abspielen tatsächlich ab dieser Stelle weiterläuft.
- Klick auf das „Auge"-Symbol (`toggleOpVisibility(opIndex)`) blendet die Bahn genau dieses einen Arbeitsgangs im 3D-Bereich aus bzw. wieder ein (diagonaler Strich über dem Auge-Glyph, Zeile zusätzlich gedimmt). Wirkt **ausschließlich** auf das Zeichnen (siehe 9.3/9.5) — Wiedergabe, Scrubber und Scope-Auswahl sind davon unberührt. Die ausgeblendeten Arbeitsgänge (`hiddenOpIndexes`, eine `Set` von 1-basierten Indizes) werden bei jedem neu geladenen bzw. gelöschten Programm zurückgesetzt.
- Der Umschalt-Knopf im Spalten-Kopf (`#btnShowAllOps`) richtet Label/Optik ausschließlich nach `hiddenOpIndexes.size`: leer → Label „Alle AG ausblenden", ein Klick blendet alle aus; nicht leer (ob einzeln oder vollständig ausgeblendet) → Label „Alle AG anzeigen", ein Klick setzt alles zurück.
- Klick auf „3D" setzt den 3D-Scope auf genau diesen Arbeitsgang (`setScope({type:'op', index})`, siehe 9.3) — echter Umschalter: Ist genau dieser Arbeitsgang bereits der aktive Scope, wechselt ein erneuter Klick stattdessen zu `{type:'all'}` („Gesamtes Programm"); ein Klick auf einen anderen Arbeitsgang wechselt direkt zu diesem.
- **Unabhängig davon** wird der Arbeitsgang, dessen Zeilenbereich die aktuell simulierte/gescrubbte Position enthält, laufend mit der eigenen Klasse `.current` hervorgehoben und automatisch in den sichtbaren Bereich gescrollt — auch in der Standardansicht „Gesamtes Programm". `.current` und `.active` (aktiver Scope) sind unabhängige Klassen und können gleichzeitig zutreffen.

### 7.3 Automatische Spaltenbreiten & Ein-/Ausklappen

Die natürliche Breite der Programmcode-Spalte wird per Canvas-2D `measureText()` aus dem tatsächlichen Inhalt berechnet (`computeTreeContentWidth`: längste Codezeile in Mono-Schrift plus Zeilennummer-Spalte/Ränder/Sicherheitsabstand) und setzt darüber die per Ziehgriff verstellbare `sidebarWidth` neu (Grenzen: `max(360, min(900, innerWidth-260))`). Ist (noch) kein CNC-Programm geladen, aber ein DXF, gilt stattdessen eine eigene, nach demselben Prinzip arbeitende automatische Breite anhand der Layer-Namen (`computeDxfLayerContentWidth()`/`applyDxfLayerColumnWidth()`, siehe 9.8a) — ein geladenes Programm hat dabei immer Vorrang.

Die Programm-Spalte kennt **fünf Einklapp-Stufen** (`progCollapseStage`, 0–4): `PROG_STAGE_PCT = [100, 75, 50, 25, 0]` — Stufe 0 = volle automatisch ermittelte Breite, Stufe 4 = vollständig eingeklappt (Mindestbreite in dem Fall 160px statt der sonst geltenden 220px). Jedes (Neu-)Laden eines Programms setzt die Stufe auf die in „Spalteneinstellungen" (7.3a) hinterlegte, dauerhaft persistierte Standard-Stufe zurück. Zwei feste, gerichtete Knöpfe im Spaltenkopf bewegen die Stufe jeweils nur in ihre eigene Richtung und bleiben am erreichten Ende stehen (kein zyklisches Umlaufen): **Zuklappen** (`#btnProgCollapse`, „«", physisch links, Stufe steigt 0→4, jeweils −25 Prozentpunkte, bei Stufe 4 deaktiviert) und **Aufklappen** (`#btnProgExpand`, „»", physisch rechts, Stufe sinkt 4→0, bei Stufe 0 deaktiviert). Ein deaktivierter Knopf zeigt zusätzlich zur Abdunklung (`opacity: .4`) eine diagonale Durchstreichung des Icons.

Diese Stufe wirkt **nur für die laufende Sitzung**: Sie überschreibt die aus „Spalteneinstellungen" geladene Standard-Stufe temporär, wird aber beim nächsten Laden/Neuladen eines Programms wieder auf die dort persistierte Standard-Stufe zurückgesetzt.

Die Arbeitsgänge-Spalte kennt dagegen **nur zwei Zustände** (`columnScalePreference.ops`, ein strikter Binärwert 0/1) statt Zwischenstufen — ausgeklappt bedeutet immer die volle, automatisch aus dem längsten Arbeitsgang-Eintrag ermittelte Breite (`computeOpsContentWidth()`: berücksichtigt sowohl den breitesten Listeneintrag inkl. Farbpunkt/Nummer/Titel/„Auge"-Knopf/„3D"-Knopf als auch die Kopfzeile selbst inkl. beider möglichen Umschalt-Knopf-Beschriftungen — die größere der beiden Breiten gewinnt), eingeklappt bedeutet 32px. Eine zentrale Funktion `applyOpsScaleToCurrent()` wendet diese Breite überall konsistent an (nach dem Laden, nach „Übernehmen" im Spalteneinstellungen-Dialog, nach JSON-Import, nach „Standardwerte wiederherstellen").

Der Ziehgriff links der Arbeitsgänge-Spalte (`#opsResizer`) verändert **nicht** deren eigene Breite, sondern verschiebt die gesamte Programm-Spalte (`setSidebarWidth((e.clientX - rect.left) + opsColSpacePx())` auf `#sidebarCol`) — da die Arbeitsgänge-Spalte als Flex-Kind rechts daneben sitzt und ihre feste, automatisch ermittelte Breite behält, „wandert" sie beim Breiterwerden der Programmcode-Spalte sichtbar mit nach rechts, statt selbst schmaler zu werden. Im eingeklappten Zustand ist der Ziehgriff inaktiv.

### 7.3a Spalteneinstellungen — persistierte Standard-Skalierung

Menüpunkt „Spalten" (`#btnColumnScale`, unter „Einstellungen" → „Erscheinungsbild", siehe Abschnitt 10) öffnet den Dialog `#columnScalePanel` mit zwei unabhängigen Auswahlgruppen: „CNC-Programmcode" (`columnScaleCode`, vier Werte 25/50/75/100 %) und „Arbeitsgänge" (`columnScaleOps`, zwei Radio-Buttons „Ausgeklappt" `value="1"`/„Eingeklappt" `value="0"`) — sowie „Abbrechen"/„Übernehmen".

**Bedeutung:** Die gewählten Werte legen die **Standard-Skalierung** fest, die bei jedem (Neu-)Laden eines Programms angewendet wird — für „CNC-Programmcode" als Startwert der Einklapp-Stufe (`PROG_STAGE_PCT.indexOf(columnScalePreference.code)`), für „Arbeitsgänge" als Ausgeklappt/Eingeklappt-Zustand. Ein Klick auf „Übernehmen" wendet die Werte zusätzlich sofort auf ein bereits geladenes Programm an; dabei wird zwingend zuerst `setOpsWidth()` mit dem neuen Arbeitsgänge-Zustand aufgerufen und **erst danach** `applyProgCollapseStage()`, da Letzteres den verfügbaren Platz der Programmcode-Spalte anhand der aktuellen `opsWidth` berechnet.

**Persistenz:** `columnScalePreference = { code, ops }` (Standard `{ code: 100, ops: 1 }`) wird über `localStorage` gespeichert (`persistColumnScale()`) und beim Programmstart geladen; ein ungültiger/veralteter (z. B. noch prozentbasierter, aus einer älteren Einstellungsdatei stammender) `ops`-Wert wird über `normalizeColumnScale()` auf `1` (ausgeklappt) abgebildet, exakte `0` bleibt `0`. Zusätzlich Teil des JSON-Exports/-Imports (5.9).

## 8. Wiedergabe-Steuerung

Transportleiste (volle Fensterbreite, direkt unter der Symbolleiste, siehe Abschnitt 10) mit Play/Pause (ein Knopf, Symbol wechselt zwischen ▶/❚❚), Stopp (springt zurück auf Position 0, im Gegensatz zu Pause), Geschwindigkeitswahl **1x/5x/10x/20x** sowie Scrubber-Schieberegler und Positionsanzeige („n / gesamt"). Wiedergabe-Takt: alle ~130ms ein Tick (`requestAnimationFrame`-basiert); die Geschwindigkeitsstufe multipliziert dabei die Anzahl der pro Tick übersprungenen Punkte (nicht die Taktdauer) — die Wiedergabe bleibt dadurch gleichmäßig flüssig statt ruckartiger bei höherer Geschwindigkeit.

**Markierungen je Arbeitsgang auf der Wiedergabe-Leiste:** Ein reines Anzeige-Overlay (`#scrubMarks`, `pointer-events: none`, in `.scrub-wrap` über dem nativen `#scrub` positioniert) zeigt eine kleine, 14px hohe und 4px breite senkrechte Markierung (`.scrub-mark`) an jeder Stelle, an der im fortlaufenden `points`-Array — demselben Array, das auch `scrubEl.max`/`.value` zugrunde liegt — ein neues, nicht gedimmtes Segment beginnt (`seg.rangeStart`, siehe 9.3 „Bahn wächst mit der Wiedergabe"), mit Ausnahme des ersten Starts bei Index 0 (liegt ohnehin am linken Rand der Leiste). `updateScrubMarks()` wird am Ende von `setScope()` bei jedem Szenen-/Scope-Wechsel neu aufgebaut, die Positionen sind reine Prozentwerte relativ zur vollen Leistenbreite (ein horizontaler `transform: translateX(-2px)`-Ausgleich hält `left` an der Mitte der 4px breiten Markierung fest). In der Praxis liegen mehrere Segmente nur beim Scope „Gesamtes Programm" gleichzeitig in `points` (dort tragen alle Arbeitsgänge gleichermaßen bei) — bei einem einzeln ausgewählten Arbeitsgang (`{type:'op', index}`) bleiben alle vorherigen als gedimmter Kontext außerhalb von `points`, es gibt dort nur ein einziges Segment und folglich keine Markierung, was inhaltlich korrekt ist (kein „neuer Arbeitsgang" innerhalb der sichtbaren Leiste). Die Positionierung gleicht die native Breite des Schiebereglers/Rundgriffs selbst nicht gesondert aus (reine Prozent-Näherung über die volle Leistenbreite) — für kleine Orientierungsmarkierungen bewusst ausreichend genau.

**Farbe je Arbeitsgang:** `updateScrubMarks()` setzt die Hintergrundfarbe jeder Markierung per Inline-Style (`mark.style.background`) auf `seg.color` — exakt dieselbe `--path-1…10`-Farbe (`pathColor(seg.colorCursor)`), mit der die Bahn dieses Arbeitsgangs auch in der 3D-Ansicht gezeichnet wird (siehe 9.1 „Arbeitsgangfarben") und mit der auch die Farbfelder in der Arbeitsgänge-Liste und die Zeilen-Randfarbe im Code eingefärbt sind. Da eine Markierung den Beginn eines neuen Arbeitsgangs anzeigt, trägt sie die Farbe genau dieses (dort neu startenden) Arbeitsgangs. `recolorOpSegmentsAndPoints()` — dieselbe Funktion, die bei einem Theme-Wechsel (Hell-/Dark-Mode) oder beim Ändern eigener Arbeitsgangfarben im Einstellungen-Panel alle Bahnen/Zeilen/Farbfelder neu einfärbt — ruft `updateScrubMarks()` zusätzlich mit auf, damit die Markierungsfarben dabei sofort mitziehen, ohne dass ein kompletter Szenen-Neuaufbau (`setScope()`) nötig wäre. Die CSS-Regel `background: var(--ink-muted)` bleibt als reiner Fallback bestehen, falls einer Markierung ausnahmsweise keine Farbe zugeordnet werden kann.

Zusätzliche Navigationswege, die alle auf denselben zentralen Positions-/Anzeige-Update-Pfad (`updateReadoutAndHighlight()` — aktualisiert Readout, aktive Code-Zeile **und** die aktuell hervorgehobene Arbeitsgang-Zeile in der Ops-Spalte, siehe 7.2) münden:
- Pfeiltasten (← / →) bewegen die Position um einen Punkt.
- „Sprung in Zeile:"-Eingabefeld (Transportleiste) springt auf den nächsten simulierten Punkt ab der eingegebenen Quelltextzeile (bewegt nur die Wiedergabeposition, scrollt die Code-Liste nicht automatisch mit). Direkt daneben sitzt der Knopf „Codesprung" (`#btnCodeJump`) — ein eigenständiger, unabhängiger Werkzeug-Umschalter für den Klick-auf-CNC-Linie-Sprung im 3D-Bereich (siehe 9.4a), nicht zu verwechseln mit diesem Eingabefeld: Das Eingabefeld springt anhand einer eingetippten Zeilennummer, „Codesprung" dagegen anhand eines Klicks direkt auf die Bahn im 3D-Viewer.
- Klick auf eine Zeile in der Arbeitsgänge-Spalte (siehe 7.2) — dies ist der einzige Navigationsweg, der zusätzlich auch die Code-Liste selbst scrollt (`jumpToCodeLine`, siehe 7.1).

**Leertaste als Play/Pause-Kurzbefehl:** Ein `keydown`-Listener auf `window` reagiert auf `e.code === 'Space'` bzw. `e.key === ' '` und ruft `btnPlay.click()` auf — dieselbe Aktion wie ein Mausklick auf den Play/Pause-Knopf. Zwei Schutzmaßnahmen: Der Kurzbefehl greift nur, wenn tatsächlich ein Programm geladen ist (`points.length`), und er greift NICHT, während der Eingabefokus auf einem Formularelement liegt (`document.activeElement.tagName` ∈ {INPUT, TEXTAREA, BUTTON, SELECT} oder `isContentEditable`) — sonst würde die Leertaste in einem fokussierten Textfeld eines der Editoren (Abschnitt 11) versehentlich die Wiedergabe umschalten statt ein Leerzeichen einzufügen.


## 9. 3D-Viewer

### 9.1 Bewegungs-Extraktion

Generischer ISO-Achsen-Parser, unabhängig vom erkannten Maschinentyp. Zwei unterstützte Achswort-Notationen, in dieser Reihenfolge geprüft:

```js
const AXIS_RE = /(?<![A-Za-z])(?:([XYZABC])\d{0,2}\s*=\s*(-?\d+(?:\.\d+)?)|([XYZABC])\s*(-?\d+(?:\.\d+)?))/g;
```

1. **„="-Notation mit optionaler Kanalnummer:** direkt (ohne Leerzeichen) 0–2 Ziffern (optionale Kanalnummer, wird verworfen) und danach ein „=", erst danach der eigentliche Zahlenwert — z. B. „B1=-90.01", „C1=-12.49", „X=780.3" (auch ganz ohne Kanalnummer).
2. **Klassische Notation** (Fallback, falls kein „=" gefunden): Achsbuchstabe, optional Leerraum, Zahl direkt danach — „X780.27", „B-90".

Das vorangestellte `(?<![A-Za-z])` (negativer Lookbehind) verhindert, dass ein Achsbuchstabe erkannt wird, dem unmittelbar ein weiterer Buchstabe vorausgeht — nötig, damit Parameterzeilen wie „TRSX=0 TRSY=0 TRSZ=0" oder „ROTX=0 ROTY=0 ROTZ=0" (Verschiebungs-/Rotationsversatz-Parameter, keine echten Achsbewegungen) nicht fälschlich als eigene Achsbewegung gelesen werden — ein echtes Achswort beginnt stets nach einem Nicht-Buchstaben.

**Kommentar-Erkennung auch bei der Bewegungs-Extraktion:** `extractMotion()` berechnet für jede Zeile einzeln dieselbe Kommentar-Maske wie SWITCHCASEMACHINE/CALCMACHINE (`CNCSwap.computeCommentMask()`/`isRangeUncommented()`, dieselben drei Kommentar-Arten `;`/`KM="…"`/`{…}`, siehe 5.11) und verwirft jeden AXIS_RE-Treffer, der (auch nur teilweise) in einem erkannten Kommentarbereich liegt — eine per SWITCHCASEMACHINE-Regel oder im Original bereits auskommentierte Zeile wie `;X=0 C=0 IF=FLD=14` erzeugt dadurch weder einen Bahnpunkt noch aktualisiert sie den modalen Bewegungszustand.

Werte werden **modal** (jede Achse behält ihren letzten Wert, bis ein neuer folgt) in den Bewegungszustand fortgeschrieben. Eilgang-/Vorschub-Erkennung (`scanRapid`, `G_RE = /G0*([0-9]{1,2})(?![0-9.])/g`): `G0` → Eilgang, `G1`/`G2`/`G3` → Vorschub; für KUKA-Robotersyntax (KRC) zusätzlich `PTP` → Eilgang, `LIN`/`CIRC` → Vorschub; ohne erkennbares Wort bleibt der zuletzt bekannte Zustand (`prevRapid`) erhalten.

**Nullpunktverschiebungen (`G92`/`O`/getauschtes `TRANS`, `ORIGIN_RESET_RE`, siehe 4.4):** Eine Zeile, die dieses Muster matcht, erzeugt **keinen eigenen Bahnpunkt** (`extractMotion()`: `if (found && !isOriginReset)`) — sie legt lediglich ein neues Koordinatensystem fest, der Fräser bewegt sich dabei nicht wirklich zu den angegebenen Werten. Der modale Zustand führt dafür einen eigenen Versatz je Achse (`state.offsetX/offsetY/offsetZ`, nur X/Y/Z, da eine Nullpunktverschiebung per Definition keine Rundachsen betrifft): Auf einer Nullpunktverschiebungszeile setzt ein gefundenes X-/Y-/Z-Achswort **nur** den jeweiligen Versatz neu (modal je Achse, fehlt eine Achse in der Zeile, bleibt ihr bisheriger Versatz unverändert) und lässt `state.x/y/z` unangetastet. Jede normale Bewegungszeile aktualisiert weiterhin `state.x/y/z` mit dem programmierten (absoluten) Wert; ein erzeugter Bahnpunkt trägt `x: state.x + state.offsetX` (entsprechend y/z). `scrubToLineIdx()` findet für eine übersprungene Nullpunktverschiebungszeile automatisch den nächsten **echten** Bahnpunkt danach.

`toRender(p) = {x: p.x, y: p.z, z: p.y}` — die Render-Y-Achse (Bildschirm-„hoch", vor Anwendung der Kamera-Basis) entspricht der CNC-Z-Achse (Werkstückhöhe), die Render-Z-Achse (Tiefe) der CNC-Y-Achse.

**Bogeninterpolation G02/G03 (R-Format und I/J-Format):** Trägt eine Zeile ein **explizites** G02 (im Uhrzeigersinn) oder G03 (gegen den Uhrzeigersinn) — ermittelt über `scanArcDirection(line)`, „letzter G-Code auf der Zeile gewinnt", aber bewusst **ohne modales Fortschreiben** auf Folgezeilen ohne eigenes G02/G03 (siehe unten) — UND einen erkennbaren Bogen-Parametersatz (`CNCParser.parseArcParams(line)`), erzeugt `extractMotion()` mehrere tessellierte Zwischenpunkte entlang des tatsächlichen Kreisbogens (`CNCParser.interpolateArc(...)`) anstelle eines einzelnen Zielpunkts. Zwei Parameter-Formate werden erkannt — kommen beide auf derselben Zeile vor, gewinnt das R-Format, da `parseArcParams()` zuerst danach sucht:

- **R-Format** (`G02/G03 X.. Y.. R<Radius>`): Positiver Radius wählt den „kurzen" Bogen (Schwenkwinkel ≤180°), negativer Radius den „langen" Bogen (>180°). Ein Vollkreis (Start=Ziel) ist im R-Format nicht darstellbar (mehrdeutig) — `interpolateArc()` liefert dann `null` und die Bahn fällt auf eine gerade Linie zurück.
- **I/J-Format** (`G02/G03 X.. Y.. I<Wert> J<Wert>`): I/J beschreiben den Bogenmittelpunkt — je nach Postprozessor entweder als Versatz vom Startpunkt (Mittelpunkt = Start + I/J) oder als bereits fertige, absolute Mittelpunkt-Koordinate. Welche der beiden Konventionen gilt, legt die Direktive `IJMODE` des aktuell aktiven Maschinentyps fest (**`incremental`** bzw. **`absolute`**, siehe 5.12a) — ohne aktiven Maschinentyp bzw. ohne eigene `IJMODE`-Zeile im Block gilt der Standard **`incremental`**. `interpolateArc(sx, sy, ex, ey, params, direction, ijMode)` erhält den für das aktuell angezeigte Programm aufgelösten Modus als letzten Parameter (ermittelt in `setScope()`/`extractMotion()` aus `activeMt.ijMode`) und wählt danach zwischen beiden Mittelpunkt-Berechnungen.

Die drei eingebauten Standard-Maschinentypen Biesse (`IJMODE=absolute`), KRC (`IJMODE=incremental`) und SCM (`IJMODE=absolute`) sind geometrisch gegen reale Beispielprogramme verifiziert (siehe 5.12a); alle übrigen Maschinentypen ohne eigene `IJMODE`-Zeile verwenden weiterhin den Standard `incremental`.

Bei Biesse müssen zur korrekten I/J-Bogendarstellung zusätzlich zur `IJMODE`-Direktive beide in 5.3 beschriebenen automatischen Ableitungen aus CALCMACHINE greifen (dort `Y*-1`): die Mitspiegelung des Bogenparameters `J` UND der automatische Drehsinn-Tausch (`computeArcChiralityFlip`) — ein falscher Mittelpunkt und ein falscher Drehsinn erzeugen äußerlich dasselbe Symptom (stark vergrößerte, teils nahezu vollständig durchlaufene Kreise statt kurzer, sanfter Bogenstücke), sind aber zwei unabhängige Fehlerquellen, die unabhängig voneinander behoben werden mussten. Geometrisch gegen das reale, vollständige Biesse-Beispielprogramm (drei wiederholte Konturabschnitte mit insgesamt 48 G02/G03-Bögen plus Endsequenz) verifiziert: Mit beiden automatischen Ableitungen bleibt die resultierende Bahn durchgehend innerhalb der tatsächlichen Bauteil-Abmessungen (Bounding-Box ca. X:[-1, 3983], Y:[-506, 396]); fehlt nur der Drehsinn-Tausch (z. B. durch eine im selben Maschinentyp-Block zusätzlich vorhandene, dann doppelt wirkende SWITCHCASEMACHINE-Regel), bläht sie sich auf bis zu X:[-6678, 12243], Y:[-512, 18415] auf — exakt das „riesengroße Kreise"-Muster, nur durch den Drehsinn statt durch den Mittelpunkt verursacht.

Bewusste Einschränkung: Es wird ausschließlich die XY-Ebene (G17) unterstützt — in der Holzbearbeitung der ganz überwiegende Regelfall. Ein eventuell vorhandenes „K" (Z-Mittelpunktversatz für XZ-/YZ-Ebenen-Bögen unter G18/G19) wird von `parseArcParams()` nicht ausgewertet. Z selbst wird linear zwischen Start- und Endpunkt über die Bogenpunkte interpoliert — das erlaubt auch helixförmige Bögen (Z ändert sich während des Bogens) ohne Mehraufwand.

Ein G02/G03 muss auf **jeder** Bogenzeile explizit erneut stehen, damit ein „R"-Wert auf einer späteren, eigentlich unbeteiligten Zeile (z. B. die Rückzugsebene `R..` eines Bohrzyklus wie `G81`) nicht fälschlich als fortgesetzter Bogen-Radius fehlinterpretiert wird.

Segmentanzahl der Tessellierung (`tessellateArc()`, gemeinsam mit dem DXF-Bogenparser über `computeSegmentCount()` verwendet, siehe 9.8) richtet sich nach der Bogenlänge (ca. 10mm je Segment), begrenzt auf 4–180 Segmente je Bogen. Bei geometrisch nicht auflösbaren/unplausiblen Eingaben (z. B. R-Format mit zu kleinem Radius für den Punktabstand, unbekannter Parametersatz, ungültige Richtung) liefert `interpolateArc()` `null` — `extractMotion()` fällt dann auf das bisherige Verhalten (gerade Linie) zurück.

Die Wiedergabe (`stepPlayback()`) rückt pro Tick um eine feste ANZAHL PUNKTE weiter (`scrubIdx += playSpeed`), nicht um eine feste Strecke — ein Geradenstück (G0/G1) besteht unabhängig von seiner Länge immer aus genau 2 Punkten (Start/Ziel) und wird dadurch praktisch instantan durchlaufen, während ein tessellierter Bogen viele kurze Segmente und entsprechend viele Punkte erzeugt.

### 9.1a Eckenverrundung „P71:" (MAKA-Dialekt)

Der MAKA-Dialekt gibt eine gewünschte Eckenverrundung (Fillet) direkt in der G-Code-Zeile an: `G1 G12 X.. Y.. P71:<Radius>;` markiert den **Zielpunkt** dieser Zeile als Ecke, die mit dem angegebenen Radius verrundet werden soll — die Verrundung wirkt an der Ecke zwischen dieser Bewegung und der **nächsten** Bewegung (klassisches Eckenfillet zwischen zwei Linienzügen, wie es z. B. auch CAD-/CAM-Software beim „Ecke abrunden" verwendet). `CNCParser.parseCornerRadius(line)` (`CORNER_RADIUS_RE = /P71\s*:\s*(-?\d+(?:\.\d+)?)/`) liest den Radius direkt aus der Zeile selbst — unabhängig vom aktuell gewählten Maschinentyp: Anders als `IJMODE` (siehe oben) ist „P71:" kein mehrdeutiger, pro Maschinentyp unterschiedlich zu interpretierender Wert, sondern eine konkrete, eindeutige Syntax. Das modale „G12" selbst (MAKA's Kennzeichnung „mit Eckenverrundung") wird dabei nicht ausgewertet/benötigt — das explizite „P71:<Radius>" auf derselben Zeile reicht als eindeutiges Signal.

**Berechnung (`computeFillet(a, b, c, r)`, b = die zu verrundende Ecke):** Klassische Tangentenkreis-Geometrie zwischen den Strecken A→B und B→C. Aus den Einheitsvektoren von B nach A und von B nach C ergibt sich über `acos(dot)` der Winkel bei B; ist er nahe 0° oder nahe 180°, liegt keine echte Ecke vor und die Funktion liefert `null`. Der Tangentenabstand `d = r / tan(Winkel/2)` wird entlang jeder der beiden Strecken von B aus abgetragen; würde das über 90% der kürzeren der beiden angrenzenden Streckenlängen hinausreichen, wird `d` konservativ darauf gekappt und der tatsächlich wirksame Radius aus dem gekappten `d` zurückgerechnet (dieselbe Rundungstoleranz-Philosophie wie bei `arcCenterFromRadius()`, siehe oben). Der Kreismittelpunkt liegt auf der Winkelhalbierenden im Abstand `r / sin(Winkel/2)` von B. Die Durchlaufrichtung (cw/ccw) des Fillet-Bogens von Tangentenpunkt 1 nach Tangentenpunkt 2 ergibt sich aus dem Vorzeichen des Kreuzprodukts der beiden Kantenvektoren. Ein Radius ≤0 oder praktisch deckungsgleiche Nachbarpunkte liefern ebenfalls `null`.

**Tessellierung (`tessellateFillet()`):** Tessellierung des Bogens von Tangentenpunkt 1 nach Tangentenpunkt 2, mit derselben `computeSegmentCount()`-Formel wie die G02/G03-Bogeninterpolation oben und der DXF-Bogen-/Bulge-Tessellierung (9.8) — Fillet- und echte Bogenpunkte fallen dadurch optisch gleich fein/grob aus.

**Einfügung in die Bahn (`applyCornerFillets(points)`):** Läuft als Nachbearbeitungsschritt über die von `extractMotion()` bereits fertig aufgebaute Punkteliste, da sowohl der Vor- als auch der Nachfolgepunkt einer Ecke gebraucht werden — beide sind erst bekannt, nachdem der eigentliche zeilenweise Durchlauf abgeschlossen ist, weshalb `extractMotion()` nicht `points` direkt zurückgibt, sondern `applyCornerFillets(points)`. Jeder Punkt, dessen Quellzeile ein `P71:<Radius>` trägt, wird dabei durch den ersten Tangentenpunkt gefolgt vom tessellierten Fillet-Bogen bis zum zweiten Tangentenpunkt ersetzt — berechnet aus dem jeweils unmittelbaren Vor-/Nachfolgepunkt in der ROHEN, noch unveränderten Punkteliste, sodass mehrere unmittelbar aufeinanderfolgende verrundete Ecken unabhängig voneinander berechnet werden, auch wenn derselbe Eckpunkt im Programm mehrfach vorkommt (z. B. der Konturschluss eines geschlossenen Vielecks — dort entstehen zwei unabhängig berechnete Fillets für die beiden getrennten Durchläufe desselben Koordinatenpunkts). Eine als Ecke markierte Zeile am allerersten oder allerletzten Punkt einer Zeilengruppe (kein Vor- bzw. Nachfolger vorhanden) sowie ein von `computeFillet()` mit `null` quittierter, geometrisch nicht sinnvoll verrundbarer Fall bleiben unverändert als scharfe Ecke stehen — derselbe automatische, lautlose Fallback wie bei einer geometrisch nicht auflösbaren G02/G03-Zeile oben, nie ein Absturz oder eine verfälschte Kontur.

Die Z-Koordinate wird dabei **nicht** über den eingefügten Fillet-Bogen interpoliert — alle eingefügten Punkte (beide Tangentenpunkte wie auch die tessellierten Zwischenpunkte) übernehmen unverändert die Z-Höhe des ursprünglichen, ersetzten Eckpunkts. Das unterscheidet sich von einem echten G02/G03-Bogen (siehe oben), bei dem Z linear über die Bogenpunkte interpoliert wird.

„P71:" ist dabei bewusst eines von inzwischen fünf parallelen, gleichberechtigten Eckenverrundungs-Erkennungsmustern (die anderen vier: G302/HOMAG, 9.1b; GFIL/SCM, 9.1c; BR=/Biesse, 9.1d; E/FORMAT4, 9.1e) — alle fünf laufen standardmäßig immer gleichzeitig, unabhängig vom gewählten Maschinentyp. Seit Release 133 kann zusätzlich je Maschinentyp ein eigener, konfigurierbarer `CORNERRADIUS=<Marker>`-Erkennungsmarker hinterlegt werden (siehe 5.12b), der für diesen Maschinentyp Vorrang vor allen fünf fest einprogrammierten Erkennern hat; ohne eine solche `CORNERRADIUS`-Zeile bleibt „P71:" für diesen Maschinentyp unverändert über den hier beschriebenen, fest einprogrammierten Mechanismus aktiv.

### 9.1b Eckenverrundung „G302 I<Radius>" (Beckhoff TwinCAT/HOMAG-Dialekt)

Release 128 (Sebastian: "G302 wird noch nicht richtig erkannt. G302 ist ein Befehl für eine Eckenrundung", mit Verweis auf die Beckhoff-TwinCAT-CNC-Dokumentation und folgendem Beispiel):

```
G1 X50 Y0 F1000   ; Fährt zum Startpunkt
G1 X50 Y50        ; Fährt zur Ecke
G302 I5           ; Fügt an der Ecke einen Radius von 5 mm ein
G1 X100 Y50       ; Fährt zum nächsten Punkt weiter
```

Anders als „P71:" (MAKA, siehe 9.1a), das direkt **auf** der Zielpunkt-Zeile der Ecke selbst steht, steht „G302 I<Radius>" auf einer **eigenen, der Ecke nachfolgenden** Zeile ohne eigene Bewegung (kein X/Y auf dieser Zeile) — sie bezieht sich auf den Punkt, der von der unmittelbar **vorherigen** Bewegungszeile erreicht wurde. `CNCParser.parseG302CornerRadius(line)` (`G302_CORNER_RADIUS_RE = /\bG302\b[^;]*?\bI\s*(-?\d+(?:\.\d+)?)/`) liest den Radius direkt aus der Zeile — wortgrenzen-geschützt (`\bG302\b`), damit weder ein längerer Code wie „G3025" noch ein als Wortteil vorkommendes „G302" (z. B. „XG302") versehentlich erkannt wird. „I" ist hier — anders als bei G02/G03 — kein Bogenmittelpunkt-Offset, sondern direkt der gewünschte Eckenradius; die Erkennung ist dialektunabhängig (genau wie bei „P71:"/R/I/J), unabhängig vom aktuell gewählten Maschinentyp.

**Einfügung in die Bahn:** Da „G302" von `G_RE` (nur 1–2-stellige, führende Nullen tolerierende G-Codes wie „G1"/„G02") nicht erfasst wird, bleiben `scanRapid()`/`scanArcDirection()` von einer solchen Zeile unberührt; da sie zudem kein X/Y/Z/A/B/C enthält, erzeugt sie in `extractMotion()` auch regulär keinen eigenen Bahnpunkt. Stattdessen prüft `extractMotion()` **unabhängig** vom sonstigen Bewegungs-Handling jede Zeile zusätzlich auf „G302 I<Radius>" und versieht bei einem Treffer den **zuletzt bereits erzeugten** Punkt in `points` (die Ecke) nachträglich mit `cornerRadius` — identischer Verrundungsmechanismus wie bei „P71:" (`applyCornerFillets()`, siehe 9.1a), nur mit anderer Herkunft des `cornerRadius`-Werts und anderem Zeitpunkt der Zuweisung (rückwirkend auf den vorherigen statt direkt auf den aktuellen Punkt). Steht „G302" ganz am Anfang eines Arbeitsgangs ohne vorherige Bewegung, gibt es keinen Punkt, dem der Radius zugeordnet werden könnte — die Direktive wird dann stillschweigend ignoriert, kein Fehler.

Wie „P71:" (9.1a) ist auch „G302 I<Radius>" ein fest einprogrammierter, immer aktiver Erkenner, der seit Release 133 für einen Maschinentyp mit eigener `CORNERRADIUS=<Marker>`-Zeile (siehe 5.12b) durch den dort konfigurierten Marker ersetzt wird — der Standard-HOMAG-Block trägt dafür `CORNERRADIUS=G302 I` (bewusst inklusive des Parameterbuchstabens „I", siehe 5.12b); ohne eigene `CORNERRADIUS`-Zeile bleibt der hier beschriebene Legacy-Mechanismus unverändert wirksam.


### 9.1c Eckenverrundung „GFIL r=<Radius>" (SCM/WinPPZ-Dialekt)

Release 130 (Sebastian meldete, dass ein reales SCM/WinPPZ-Beispielprogramm Eckenverrundungen über ein eigenes `GFIL`-Kommando trägt, das bis dahin nicht erkannt wurde). Syntax-Beispiel:

```
G1  X0.000 V6000
GFIL r=5.000  V6000
G1  Y50.000 V3500
G1 X50.000 V3500
```

Strukturell identisch zu „G302 I<Radius>" (HOMAG, 9.1b): `GFIL r=<Radius>` steht auf einer **eigenen, der Ecke nachfolgenden** Zeile ohne eigene X/Y-Bewegung und bezieht sich auf den zuletzt erreichten Punkt (hier: den direkt zuvor über `G1 X0.000` angefahrenen Startpunkt der Kontur). `CNCParser.parseGFILCornerRadius(line)` (`GFIL_CORNER_RADIUS_RE = /\bGFIL\b[^;]*?\br\s*=\s*(-?\d+(?:\.\d+)?)/i`) liest den Radius wortgrenzen-geschützt (`\bGFIL\b`) direkt aus der Zeile — dialektunabhängig und unabhängig vom aktuell gewählten Maschinentyp, genau wie „P71:"/„G302". Das „r=" ist hier (anders als bei G02/G03) kein Bogenparameter, sondern direkt der gewünschte Eckenradius, dem „GFIL" (vermutlich „Geometrie-FILlet"/„G-Code-FILlet") als eigenständiges Kommando vorausgeht.

**Einfügung in die Bahn:** `GFIL` wird — wie „G302" (9.1b) — von `G_RE` nicht erfasst (kein numerischer G-Code) und enthält kein eigenes X/Y/Z/A/B/C, erzeugt also regulär keinen eigenen Bahnpunkt. `extractMotion()` prüft jede Zeile zusätzlich auf `GFIL r=<Radius>` und versieht bei einem Treffer den zuletzt bereits erzeugten Punkt in `points` nachträglich mit `cornerRadius` — identischer Mechanismus, identische Code-Stelle wie bei „G302" (`g302CornerRadius`/`gfilCornerRadius` werden in `extractMotion()` parallel, unabhängig voneinander geprüft, siehe auch 9.1b). Steht „GFIL" ganz am Anfang eines Arbeitsgangs ohne vorherige Bewegung, wird die Direktive stillschweigend ignoriert, kein Fehler.

Wie „P71:"/„G302" ist auch „GFIL r=<Radius>" ein fest einprogrammierter, immer aktiver Erkenner, der seit Release 133 für einen Maschinentyp mit eigener `CORNERRADIUS=<Marker>`-Zeile (siehe 5.12b) durch den dort konfigurierten Marker ersetzt wird — der Standard-SCM-Block trägt dafür `CORNERRADIUS=GFIL r=` (bewusst inklusive des Parameterbuchstabens UND des Gleichheitszeichens „r=", siehe 5.12b); ohne eigene `CORNERRADIUS`-Zeile bleibt der hier beschriebene Legacy-Mechanismus unverändert wirksam.

### 9.1d Eckenverrundung „BR=<Radius>" (Biesse-Dialekt)

Release 131 (Sebastian meldete, dass bei einem realen Biesse-Beispielprogramm an einigen Ecken trotz vorhandener `BR=`-Markierung keine Verrundung erschien). Syntax-Beispiel:

```
G1 X0 Y0 F1000
G1 X50 Y50 BR=6.00
G1 X100 Y50
```

Anders als „G302"/„GFIL" (9.1b/9.1c, eigene, der Ecke nachfolgende Zeile) steht „BR=<Radius>" — wie „P71:" (MAKA, 9.1a) — direkt **auf** der Zielpunkt-Zeile der zu verrundenden Ecke selbst, angehängt an die eigentliche Bewegung. `CNCParser.parseBRCornerRadius(line)` (`BR_CORNER_RADIUS_RE = /\bBR\s*=\s*(-?\d+(?:\.\d+)?)/i`) liest den Radius direkt aus der Zeile, wortgrenzen-geschützt vor „BR", unabhängig vom aktuell gewählten Maschinentyp.

**Ursache der ursprünglich fehlenden Verrundung — eine vom Benutzer selbst gepflegte SWITCHCASEMACHINE-Regel:** Der eigentliche Grund, warum „BR=" zunächst nicht wirkte, lag nicht am fehlenden Parser-Mechanismus, sondern an einer bereits im Biesse-Maschinentyp-Block vorhandenen, von Sebastian selbst gepflegten SWITCHCASEMACHINE-Regel `CR=->R` (siehe 5.6, zur Umwandlung des Biesse-eigenen `CR=`-Bogenradius-Formats in das generische „R"-Format, analog zur SCM-Regel). Eine zunächst versuchte, analog gebildete Regel „BR=->R" hätte dieselbe Zeichenfolge „BR=" bereits **vor** dem eigentlichen Parsen in „R" umgewandelt (SWITCHCASEMACHINE-Regeln laufen immer zuerst auf dem gesamten Rohtext, siehe 5.5) — genau wie beim in 5.6 beschriebenen MAKA-Block-Beispiel wäre dadurch die literale Markierung „BR=", auf die `parseBRCornerRadius()` angewiesen ist, bereits spurlos verschwunden, bevor der Eckenverrundungs-Mechanismus sie zu Gesicht bekommen hätte — die Verrundung wäre dadurch lautlos wirkungslos geblieben, exakt das beobachtete Symptom. Die korrekte Lösung ist dieselbe wie beim MAKA-Block: **keine** eigene SWITCHCASEMACHINE-Textregel für „BR=", sondern direkte Auswertung der Zeile durch `parseBRCornerRadius()`/`applyCornerFillets()` — eine zusätzliche, vom Format4-R-Format benötigte `CR=->R`-Regel für echte Bogenradien bleibt davon unberührt, solange sie nicht versehentlich auch „BR=" mittrifft (`CR=->R` trifft nur auf „CR=", nicht auf „BR=", da die Regel exakt auf die Zeichenfolge „CR=" matcht und „B" davor nicht Teil des Suchtexts ist — beide Regeln können also gefahrlos nebeneinander bestehen, solange keine eigene „BR=->…"-Regel hinzugefügt wird).

**Einfügung in die Bahn:** Da „BR=<Radius>" auf derselben Zeile wie die eigentliche Bewegung steht, läuft die Erkennung im selben Zweig wie „P71:"/„E<Radius>" (9.1e) — `extractMotion()` prüft pro Zeile in fester Rangfolge zunächst einen konfigurierten `CORNERRADIUS`-Marker (siehe 9.1a/5.12b), dann „P71:", dann „BR=", zuletzt „E<Radius>"; der erste tatsächliche Treffer gewinnt, der zugehörige Wert wird dem in dieser Zeile neu erzeugten Punkt direkt als `cornerRadius` zugewiesen. Die weitere Verarbeitung (Tangentenpunkt-Berechnung, Tessellierung, Einfügung in `points`) läuft identisch über denselben, bereits in 9.1a beschriebenen `applyCornerFillets()`-Mechanismus — unabhängig davon, welcher der vier gleichwertigen Marker den Radius geliefert hat.

Wie „P71:" ist auch „BR=<Radius>" ein fest einprogrammierter, immer aktiver Erkenner, der seit Release 133 für einen Maschinentyp mit eigener `CORNERRADIUS=<Marker>`-Zeile (siehe 5.12b) durch den dort konfigurierten Marker ersetzt wird — der Standard-Biesse-Block trägt dafür `CORNERRADIUS=BR=`; ohne eigene `CORNERRADIUS`-Zeile bleibt der hier beschriebene Legacy-Mechanismus unverändert wirksam.

### 9.1e Eckenverrundung „E<Radius>" (FORMAT4-Dialekt)

Release 132 (analog zu BR=/Biesse: ein reales FORMAT4-Beispielprogramm markiert Eckenverrundungen über ein einzelnes, direkt an die Bewegungszeile angehängtes „E<Radius>"). Syntax-Beispiel:

```
G1 X0 Y0 F1000
G1 X50 Y50 E6.00
G1 X100 Y50
```

Wie „BR="/„P71:" steht „E<Radius>" direkt **auf** der Zielpunkt-Zeile der zu verrundenden Ecke. `CNCParser.parseFormat4CornerRadius(line)` liest den Radius direkt aus der Zeile.

**Falsch-Positiv-Schutz — kein Treffer ohne unmittelbar folgende Zahl:** Der einzelne Buchstabe „E" kommt in FORMAT4-Programmen auch als Teil von reinem Freitext vor — insbesondere in den programmende-/zwischenstopp-typischen Markierungen „# PAUSE"/„# END OF PROGRAM" (siehe 4.3) sowie allgemein in Kommentartext. Der Erkennungs-Regex verlangt deshalb zwingend eine unmittelbar (ggf. mit optionalem Leerzeichen) folgende Zahl nach dem „E" und ist wortgrenzen-geschützt davor (kein vorausgehender Buchstabe) — ein bloßes „E" ohne angehängten Zahlenwert (wie in „END OF PROGRAM" oder einem reinen Kommentartext) erzeugt dadurch keinen Treffer und lässt die betroffene Ecke unverändert scharf. Gegen genau diesen Fall ist ein eigener Regressionstest vorhanden (`run_full_regression.js`, siehe 14): Ein synthetisches Programm mit den Kommentarzeilen „# PAUSE"/„# END OF PROGRAM" zwischen echten Bewegungszeilen erzeugt weiterhin ausschließlich scharfe Ecken.

**Einfügung in die Bahn:** Gleicher Zweig wie „P71:"/„BR=" (siehe 9.1d) — `extractMotion()` prüft pro Zeile in fester Rangfolge (konfigurierter `CORNERRADIUS`-Marker → „P71:" → „BR=" → „E<Radius>") und weist den ersten tatsächlichen Treffer dem in dieser Zeile neu erzeugten Punkt als `cornerRadius` zu; die weitere Verarbeitung läuft identisch über `applyCornerFillets()` (9.1a).

Wie „P71:"/„BR=" ist auch „E<Radius>" ein fest einprogrammierter, immer aktiver Erkenner, der seit Release 133 für einen Maschinentyp mit eigener `CORNERRADIUS=<Marker>`-Zeile (siehe 5.12b) durch den dort konfigurierten Marker ersetzt wird — der Standard-FORMAT4-Block trägt dafür `CORNERRADIUS=E`; ohne eigene `CORNERRADIUS`-Zeile bleibt der hier beschriebene Legacy-Mechanismus unverändert wirksam.


### 9.2 Kamera

Sphärische Kamera um einen Zielpunkt (`cam = {theta, phi, dist, minEyeDist, maxDist, target, roll}`):

```
cameraPosition():
  eyeDist = max(dist, minEyeDist)
  x = target.x + eyeDist·cos(phi)·sin(theta)
  y = target.y + eyeDist·sin(phi)
  z = target.z + eyeDist·cos(phi)·cos(theta)
```

`dist` ist der vom Nutzer per Mausrad gesteuerte Zoom-Wert, geklemmt auf `[30, maxDist]`. `minEyeDist` ist die Mindest-Augdistanz, mit der die Kamera tatsächlich positioniert wird (`frameCamera()` setzt sie auf `max(120, halbeBoundingBoxDiagonale + 60)` des aktuell angezeigten Programms/Arbeitsgangs/DXF, ohne Geometrie 220) — sie verhindert, dass die Kamera beim Heranzoomen physisch näher an den Zielpunkt heranfährt, als der nächstgelegene Bahnpunkt selbst entfernt ist (was sonst zu fälschlich als „hinter der Kamera" ausgeblendeten, nahen Punkten führen würde). Die Blickrichtung (und damit `right`/`up`/`fwd`, siehe `buildBasis()`) hängt nur von `theta`/`phi` ab, nicht von der tatsächlich verwendeten Distanz.

**`maxDist` — dynamisch statt fest:** `frameCamera()` setzt `dist` proportional zur Bounding-Box-Diagonale des angezeigten Bauteils — bei einem großen Bauteil (z. B. ca. 10000mm) kann das deutlich über einen festen unteren Richtwert hinausgehen. `frameCamera()` berechnet deshalb `maxDist` dynamisch als `max(4000, dist·8)` (4000 bleibt Untergrenze, normal große Bauteile bleiben davon unberührt), und der Mausrad-Zoom (wheel-Handler) klemmt `dist` gegen dieses `cam.maxDist` statt gegen eine feste Obergrenze — dadurch lässt sich ein großes Bauteil nach dem Einrahmen (z. B. per „Home") jederzeit vollständig wieder herauszoomen. Der über „+Werkzeuge (T)" gesicherte/wiederhergestellte Kamerazustand (siehe Ende dieses Abschnitts) sichert `maxDist` mit, damit ein weiter herausgezoomter Zustand dabei nicht verlorengeht.

`buildBasis(camPos)` — eine einzige, durchgehende Formel ohne Sonderfall-Korrektur:

```
fwd   = normalize(target - camPos)
right = normalize(cross(fwd, {x:0,y:1,z:0}))   // Fallback {1,0,0}, falls entartet (|right| < 1e-6)
up    = cross(right, fwd)
cam.roll rotiert right/up anschließend um fwd (Rotation um die eigene Blickachse, siehe Gizmo unten)
```

Diese Formel ist über den gesamten erlaubten `phi`-Bereich (±1,45 rad ≈ ±83°, der exakte Pol bei 90° wird durch das Clamping nie erreicht) für jeden Drehwinkel `theta` stetig. „↑ Oben" zeigt die Kamera-Position, die von Haus aus „X rechts, Y oben" liefert; „↓ Unten" die andere, physisch oberhalb liegende Position, die „Y unten" liefern würde.

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

„Vorn"/„Hinten" liegen auf derselben horizontalen Kamera-Höhe wie „Links"/„Rechts" (`phi=0`), aber um die Y- statt die X-Achse geschwenkt: „hinten" ist die +Y-Seite, „vorne" die −Y-Seite (Achs-Konvention „X+ nach rechts, Y+ nach hinten, Z+ nach oben").

**Bedienung:** Linke Maustaste + Ziehen rotiert (`theta -= dx·0.006`, `phi` analog, geklemmt auf ±1.45), rechte Maustaste + Ziehen verschiebt (`panCamera`, natives Kontextmenü im 3D-Bereich unterdrückt), Mausrad zoomt **zum Mauszeiger** hin/weg. Neues Laden eines Programms setzt die Ansicht immer auf das Oben-Preset zurück (`cam.roll` ebenfalls auf 0); reiner Arbeitsgang-Wechsel (`setScope`) behält den Blickwinkel bei und rahmt nur Distanz/Ziel neu ein (`frameCamera()`: Zielpunkt = Bounding-Box-Mittelpunkt der sichtbaren Punkte, `dist = max(90, diag·1.5+60)`, `minEyeDist = max(120, diag/2+60)`, `maxDist = max(4000, dist·8)`, siehe oben).

**Zoom zum Mauszeiger:** Da die Projektion orthografisch ist, hängt die Bildschirmposition eines Weltpunkts nur von seinem Versatz zu `cam.target` in der right/up-Bildebene ab, skaliert mit `scale`. Beim Zoomen wird `cam.target` zusätzlich zur reinen Distanzänderung so verschoben, dass der Punkt unter dem Mauszeiger an derselben Bildschirmposition bleibt:

```
shift = (distAlt − distNeu) / ((h/2)·f)
tr = (mouseX − w/2) · shift   // entlang right
tu = (h/2 − mouseY) · shift   // entlang up
```

Bei Zoom exakt im Bildschirmmittelpunkt ergibt sich `tr = tu = 0` (keine Zielpunkt-Verschiebung); am Zoom-Anschlag (`dist` bereits bei 30 oder `maxDist`) findet ebenfalls keine Verschiebung statt.

**„Home"-Knopf** (frei im 3D-Bereich platziert, links neben dem Ansichts-Gizmo, siehe unten): setzt `theta`/`phi`/`roll` fest auf das Oben-Preset und ruft anschließend `frameCamera()` erneut auf — stellt damit die Ansicht von oben her UND setzt Zoom/Zielpunkt auf das aktuell angezeigte Programm bzw. den aktuell aktiven Arbeitsgang-Scope zurück, **ohne** den 3D-Scope selbst zu verändern (anders als „Gesamtes Programm", das zusätzlich den Scope auf `{type:'all'}` umschaltet).

**Pos1-Taste:** globaler `keydown`-Listener auf `window`, ruft bei `e.key === 'Home'` exakt dieselbe, unveränderte `goHomeView()`-Funktion auf wie ein Klick auf `#btnHomeView` selbst — beide Auslöser sind dadurch garantiert immer exakt gleichwertig. Da `#btnHomeView` kein `disabled`-Attribut besitzt (`goHomeView()`/`frameCamera()` sind auch ganz ohne geladenes Programm/DXF sicher aufrufbar, siehe oben) ist auch die Pos1-Taste ohne jede Vorbedingung nutzbar. Wie bei den bestehenden Tastenkürzeln (Leertaste, Pfeiltasten) wird die Taste ignoriert, solange ein echtes Texteingabefeld (Eingabe/Textarea/Auswahlliste/`contenteditable`) fokussiert ist, damit „Pos1" dort seine gewohnte Bedeutung (an den Zeilenanfang springen) behält.

### 9.2a Rhombenkuboktaeder-Ansichts-Gizmo

Ein anklickbarer 3D-Körper (Rhombenkuboktaeder) oben rechts im 3D-Bereich (`GIZMO_MARGIN = 18px` Abstand zum Rand, `GIZMO_RADIUS = 34px`) dient als Bedienelement für Kamera-Presets und freies Drehen. `drawViewGizmo()` zeichnet ihn nur, solange tatsächlich Bahnpunkte eines Programms ODER eine geladene und sichtbare DXF-Unterlage vorhanden sind.

**Geometrie:** Rein mathematisch erzeugt (`GIZMO_FACES`, `app.js`) — kein importiertes 3D-Modell. Die 24 Eckpunkte sind alle Permutationen von `(±1, ±1, ±(1+√2))`; daraus ergeben sich 26 Flächen (6 Quadrate an den Hauptachsen, 12 Rechtecke an den Kanten, 8 Dreiecke an den Ecken) und 48 Kanten. Nur die sechs Hauptflächen (an den Achsen ±X/±Y/±Z) tragen eine Beschriftung: Normalenvektor `(1,0,0)` → „Right" → `VIEW_PRESETS.right`, `(-1,0,0)` → „Left", `(0,1,0)` → „Bottom", `(0,-1,0)` → „Top", `(0,0,1)` → „Rear", `(0,0,-1)` → „Front".

**Rendering:** Der Körper ist konvex und um den Ursprung zentriert und wird rein orthografisch (dieselbe Projektionslogik wie die Hauptszene) betrachtet — ein einfacher Rückseiten-Test (`dot(normal, fwd) < -ε`) genügt zur Sichtbarkeitsbestimmung. Die verwendete Basis (`buildBasis(cameraPosition())`) hängt nur von `cam.theta`/`cam.phi` ab, nicht von Zoom oder Zielpunkt — das Gizmo zeigt unabhängig vom aktuellen Zoom/Ausschnitt der Hauptszene stets die korrekte, aktuelle Blickrichtung. Jede sichtbare Fläche wird nach Beleuchtungsstärke unterschiedlich stark eingefärbt (Hauptflächen in der Akzentfarbe, übrige Flächen neutral grau).

**Klickbarkeit — alle 26 Flächen:** Jede aktuell sichtbare Fläche (nicht nur die sechs beschrifteten Hauptseiten) landet bei jedem Frame mit `{points, view, normal}` in `gizmoHitFaces` — bei Kanten/Ecken ist `view: null`, stattdessen ihre eigene `normal`. `hitTestGizmo()` (Standard-Punkt-in-Polygon-Test) liefert die getroffene Fläche und dient sowohl der Cursor-Rückmeldung (Mauszeiger wird zu „pointer") als auch der Klick-Auswertung. Ein Klick auf eine Hauptseite (`hit.view` gesetzt) ruft `applyViewPreset(hit.view)` auf (setzt `cam.theta`/`cam.phi` auf das exakte, benannte Preset); ein Klick auf eine Kante oder Ecke berechnet die Zielperspektive dagegen direkt aus der Flächennormale: Da `cameraPosition()` die Kamera-Richtung als reine Funktion von `theta`/`phi` ausdrückt, lässt sich diese Formel für eine beliebige Normale umkehren — `phi = asin(ny)`, `theta = atan2(nx, nz)` (`viewAnglesFromNormal()`). Eine Kante (zwei betragsgleiche Normalen-Komponenten) landet dadurch bei `phi ≈ ±45°`, eine Ecke (drei betragsgleiche Komponenten) bei `phi ≈ ±35,26°`. Beide Wege münden in derselben gemeinsamen Kernfunktion `applyViewAngles(theta, phi)`.

**Ein Klick fokussiert zusätzlich neu:** `applyViewAngles()` ruft zusätzlich zum Setzen von `cam.theta`/`cam.phi`/`cam.roll` auch `frameCamera()` auf (dieselbe Funktion, die auch `goHomeView()`/„Home" für Zielpunkt/Zoom verwendet, siehe 9.2) — ein Klick auf eine beliebige Gizmo-Fläche (Hauptseite über `applyViewPreset()`, Kante/Ecke direkt, beide münden in `applyViewAngles()`) fokussiert die aktuell sichtbare Geometrie dadurch zuverlässig neu, exakt wie „Home", nur aus der jeweils gewählten Perspektive statt zwingend von oben — auch wenn die Ansicht zuvor per rechter Maustaste weit weggeschwenkt wurde (`panCamera()`).

**Greif- und drehbar überall, plus Rotation um die eigene Achse (Roll):** Ein auf dem Gizmo (an beliebiger Stelle, auch mitten auf einer Hauptfläche) begonnener Zug dreht die Kamera exakt nach derselben Formel wie ein Zug außerhalb des Gizmos (`cam.theta -= dx·0.006`, `cam.phi` analog, geklemmt auf ±1.45) — ein `gizmoPointerId`-Zustand dient dabei nur noch der „echter Klick vs. Zug"-Unterscheidung (≤6px Bewegung zwischen `pointerdown`/`pointerup` → zusätzlicher Sprung auf die getroffene Voreinstellung). Ein schmaler Ring knapp außerhalb des sichtbaren Würfels (`hitTestGizmoRing()`, Abstand zwischen `GIZMO_RADIUS − 4` und `GIZMO_RADIUS + 14`) erlaubt eine reine Rotation der Kamera um ihre eigene Blickachse (`cam.roll`, über die Winkeländerung zum Gizmo-Mittelpunkt bei jeder Mausbewegung akkumuliert) — ändert nicht, wohin die Kamera schaut, nur welche Bildschirmrichtung dabei „oben" ist. Ein Sprung auf eine feste Voreinstellung (Preset-Klick, „Home", Laden eines neuen Programms) setzt `cam.roll` jeweils wieder auf 0 zurück. `cam.roll` ist fester Bestandteil der Bake-Cache-Parameter (siehe 9.5), damit ein reines Rollen zuverlässig einen Voll-Rebake auslöst.

**Passt sich der DXF-Ausdehnung an, auch ohne Programm:** Bodenraster (`drawGrid()`) und Achsen-Gizmo (`drawAxisGizmo()`, die drei farbigen X/Y/Z-Achsen im Ursprung) beziehen ihre Größe aus derselben kombinierten Punktmenge wie `frameCamera()` selbst (`viewPoints` plus, sofern geladen/sichtbar, die DXF-Unterlage über `dxfPointsForBounds()`), nicht nur aus den CNC-Bahnpunkten. Ohne geladenes Programm, nur mit importierter DXF, sind Raster und Achsen dadurch exakt an die tatsächlich angezeigte DXF-Ausdehnung angepasst statt auf einen festen `-100..100`-Fallback zurückzufallen.

**`lastFramedBounds`-Cache statt Live-Neuberechnung:** `drawGrid()`/`drawAxisGizmo()` lesen ihre Ausdehnung nicht bei jedem Frame live neu aus `boundsOf(viewPoints.concat(dxfPointsForBounds()))`, sondern aus einer eigenen Zustandsvariable `lastFramedBounds`, die das Ergebnis der jeweils letzten ECHTEN `frameCamera()`-Berechnung festhält — also genau der Momente, die auch tatsächlich die Kamera selbst bewegen (Laden/Löschen von Programm/DXF, Scope-Wechsel, „Home", Spiegeln/Drehen/Skalieren, siehe 9.2/9.2a/9.8c). `frameCamera()` schreibt `lastFramedBounds` bei jedem Aufruf neu (auch `null`, wenn wirklich nichts geladen/sichtbar ist). Grund: `dxfPointsForBounds()` liefert bewusst ein leeres Array, sobald alle DXF-Layer ausgeblendet sind (siehe 9.8a), unabhängig vom gewählten Weg (Sammel-Knopf oder einzelne Checkbox) — würde die Ausdehnung dabei live neu berechnet, fielen Raster/Achsen-Gizmo pro Frame plötzlich auf den kleinen, ortsfesten Nichts-geladen-Standard zurück, obwohl die Kamera selbst (`cam.target`/`cam.dist`) unverändert auf der vollen Ausdehnung stehen bleibt (Ein-/Ausblenden von Layern bewegt die Kamera bewusst nicht, analog zum „Auge" der Arbeitsgänge-Spalte, siehe 9.3). Da `drawGrid()`/`drawAxisGizmo()` ausschließlich den Cache lesen, verändert ein reines Ein-/Ausblenden von DXF-Layern (über beide Wege) weder Kamera noch Raster/Gizmo — ein bewusster Re-Frame (z. B. „Home"-Klick danach) berücksichtigt weiterhin korrekt nur die aktuell sichtbare Geometrie.


### 9.3 Szenen-Aufbau je Scope

`setScope(mode)`:
- **`{type:'all'}`** (Standard, „Gesamtes Programm"): Sind Arbeitsgänge erkannt, wird jeder in seiner eigenen, vollfarbigen Bahnfarbe (`pathColor`) in Originalreihenfolge dargestellt. Ohne erkanntes Schema wird stattdessen nach Nullpunktverschiebungen eingefärbt (4.4).
- **`{type:'op', index}`**: Der gewählte Arbeitsgang wird aktiv/vollfarbig dargestellt, alle **vorherigen** Arbeitsgänge bleiben als gedimmter Kontext sichtbar (`--ink-faint`, „bereits gefertigtes Bauteil"), spätere werden ausgeblendet.

Aktive Punkte (Scrubber/Readout/Wiedergabe) sind stets nur die des fokussierten Bereichs (nicht der gedimmte Kontext); für Kamera-Framing und Raster zählen dagegen alle sichtbaren Punkte inklusive Kontext.

**Bahn wächst mit der Wiedergabe:** Jedes NICHT gedimmte Segment (`context: false`) merkt sich in `setScope()` seinen eigenen Start-Index (`seg.rangeStart`) innerhalb des gemeinsamen, fortlaufenden Wiedergabe-Arrays `points`. `drawPolyline()` zeichnet von einem solchen Segment nur die Punkte, die bis zur aktuellen Wiedergabeposition bereits erreicht sind: `revealCount = clamp(scrubIdx − seg.rangeStart + 1, 0, seg.points.length)`. Gedimmte Kontext-Segmente werden davon ausgenommen und immer vollständig gezeichnet. Kamera-Einrahmung und Raster basieren weiterhin auf der vollständigen Bounding Box aller sichtbaren Punkte (`viewPoints`), damit die Ansicht beim Zusehen nicht mitwandert/-zoomt.

Das Wachsen gilt erst **ab dem tatsächlichen Start der Wiedergabe**, nicht bereits ab dem bloßen Laden: Ein Zustand `hasStarted` (`false` direkt nach Laden/Scope-Wechsel bzw. nach „Stop") merkt sich, ob der Benutzer die Wiedergabe bereits aktiv bewegt hat (Play, Ziehen am Scrubber, Pfeiltasten, Zeilensprung/Klick auf einen Arbeitsgang). `drawPolyline()` zeichnet ein nicht gedimmtes Segment weiterhin sofort vollständig, solange `!hasStarted` gilt, und wechselt erst danach auf die an `scrubIdx` gekoppelte Teilanzeige. So sieht man direkt nach dem Laden (und nach jedem „Stop") sofort das komplette Programm im Überblick, und erst ein echter Start lässt die Bahn wieder schrittweise nachwachsen.

Ein per „Auge"-Knopf (siehe 7.2) ausgeblendeter Arbeitsgang (`hiddenOpIndexes`) wird beim Rendern komplett übersprungen (`render()`: `if (hiddenOpIndexes.has(seg.opIndex)) return;`) — unabhängig vom „wächst mit der Wiedergabe"-Verhalten und ohne Einfluss auf `points`/`scrubIdx`/Kamera-Framing.

### 9.4 Messwerkzeug

Zwei Modi, exklusiv zueinander (Aktivieren des einen schaltet den anderen automatisch ab, inkl. Zurücksetzen bereits gesetzter Punkte): **Längenmessung** (`#btnMeasure`, „📏") und **Winkelmessung** (`#btnMeasureAngle`, „∠"). Beide Auslöser existieren doppelt und gleichwertig — als reine Icon-Knöpfe in der Kopfzeile des Info-Fensters (`#infoPanelHead`, siehe 9.6) sowie als Icon-Knöpfe in der immer sichtbaren Symbolleiste (`#toolbarRow3`, direkt hinter „⛶ Gesamtes Programm", siehe Abschnitt 10) — eine gemeinsame Zustandslogik (`toggleMeasure()`/`toggleMeasureAngle()`/`syncMeasureButtons()`) hält beide Knopfpaare stets synchron `.active`. Die Platzierung in der immer sichtbaren Symbolleiste (statt nur in der nur bei geladenem Programm sichtbaren Transportleiste) stellt sicher, dass beide Werkzeuge auch beim reinen Betrachten einer DXF-Unterlage ohne geladenes CNC-Programm nutzbar sind.

**Kalibrierte Icon-Größen:** Das Zeichen „∠" besteht nur aus zwei dünnen Linien und wirkt dadurch bei gleicher Schriftgröße optisch kleiner als das benachbarte, flächig gezeichnete Lineal-Emoji „📏". Statt einer pauschal gleichen Schriftgröße ist die tatsächlich gerenderte Zeichenhöhe („Ink-Height") von „∠" auf die von „📏" **im jeweiligen Kontext** kalibriert, über zwei getrennte, ID-spezifische CSS-Regeln in `template_top.html`: Symbolleiste `#btnMeasureAngleTransport { font-size: 19px; }` (📏 hat dort bei 11.5px eine Ink-Height von 13px, „∠" bei 19px ebenfalls 13px) und Info-Fenster-Kopfzeile `#btnMeasureAngle { font-size: 27px; }` (📏 hat dort bei 18px eine Ink-Height von 19px, „∠" bei 27px ebenfalls 19px). „📏" und alle übrigen `.toolbar-btn`/`.info-panel-btn`-Knöpfe bleiben bei ihrer regulären Größe.

**Gleich große Knopf-Rahmen in der Symbolleiste:** Da `.toolbar-btn` von Haus aus keine feste Breite/Höhe hat (Rahmen wächst mit Innenabstand + Zeilenhöhe des Inhalts), würde die für „∠" nötige größere `font-size` (19px gegenüber 11.5px bei „📏") sonst auch dessen Knopf-Rahmen sichtbar größer werden lassen. Beide Symbolleisten-Knöpfe tragen deshalb eine feste, identische Klickfläche: `#btnMeasureTransport, #btnMeasureAngleTransport { width: 36px; height: 26px; padding: 0; display: flex; align-items: center; justify-content: center; line-height: 1; }` — Flexbox-Zentrierung statt variabler Innenabstände, beide Rahmen dadurch pixelgenau gleich groß (je 36×26px). Die Info-Fenster-Variante (`.info-panel-btn`) hat unabhängig davon ohnehin eine feste 34×34px-Klickfläche für beide Mess-Knöpfe.

**Fangpunkt-Erkennung (`hitTest3D()`):** Ein Klick im 3D-Bereich sucht in dieser Reihenfolge:
1. den nächstgelegenen sichtbaren Bahn- oder DXF-Punkt im Bildschirmabstand `MEASURE_PICK_PX = 14`;
2. liegt keiner nah genug, den nächstgelegenen echten **Schnittpunkt zweier Linien** (Kontur- und/oder DXF-Linien) — `screenSegmentIntersect()` (klassische Zwei-Geraden-Schnittpunktformel über die Parameter t/u) prüft dafür alle Linienpaare, deren Bildschirm-Lot-Fußpunkt zum Klick höchstens 14px entfernt liegt, und akzeptiert nur einen Schnittpunkt, der tatsächlich auf BEIDEN Strecken selbst liegt (nicht nur auf ihrer gedachten Verlängerung); die 3D-Koordinate wird als Mittelwert der beiden getrennt interpolierten Punkte berechnet (bei ebenen 2D-Konturen liegen beide ohnehin exakt aufeinander);
3. liegt auch keiner nah genug, den nächsten Punkt auf einer nahen Bahn-Linie (Lot-Fußpunkt-Berechnung).

Ein Klick zählt nur dann als Auswahl, wenn sich die Maus zwischen Drücken und Loslassen um höchstens 6px bewegt hat (sonst wurde stattdessen die Kamera bedient). Beide Suchstufen beziehen sowohl `viewPoints`/`segments` (CNC-Bahn) als auch die (um `dxfOffset` und Spiegelung/Drehung transformierte) DXF-Geometrie ein, sofern ein DXF geladen UND sichtbar ist (siehe 9.8) — eine Messung funktioniert dadurch auch ganz ohne geladenes CNC-Programm sowie gemischt zwischen einem DXF-Punkt und einem Punkt der Fräskontur.

**Hover-Fangpunkt-Hervorhebung:** Solange das Messwerkzeug aktiv ist und gerade nicht per Drag die Kamera bedient wird, läuft bei jeder Mausbewegung über dem 3D-Bereich derselbe `hitTest3D()` wie beim tatsächlichen Klick, aber ohne bereits einen Mess-Punkt zu setzen — das Ergebnis wird als hohler, halbtransparent gefüllter Ring (Radius 9px) mit kleinem Mittelpunkt gezeichnet, deutlich unterscheidbar von den soliden, gefüllten Kreisen (Radius 5px) für bereits gesetzte Start-/Endpunkte.

**Längenmessung:** Erster Klick setzt den Startpunkt, zweiter den Endpunkt; ein dritter Klick verwirft die bisherige Messung und beginnt mit diesem Klick als neuem ersten Punkt von vorn. Das Ergebnis (ΔX/ΔY/ΔZ, „Ebene (XY)", „Direkt (3D)") erscheint im Info-Fenster (siehe 9.6). Zusätzlich zur gestrichelten direkten Verbindungslinie (Luftlinie) werden bis zu drei weitere, ebenfalls gestrichelte „Treppenweg"-Teilstrecken gezeichnet, die den Weg von Start- zu Endpunkt achsenweise nachzeichnen: `start → (endX, startY, startZ)` in der Achsfarbe X, `(endX, startY, startZ) → (endX, endY, startZ)` in der Achsfarbe Y, `(endX, endY, startZ) → end` in der Achsfarbe Z (dieselben CSS-Variablen `--axis-x`/`--axis-y`/`--axis-z` wie beim Achsen-Gizmo) — jede Teilstrecke nur, wenn ihre Länge tatsächlich ungleich null ist.

**Winkelmessung:** Sammelt bis zu drei Fangpunkte; der beim zweiten Klick gesetzte Punkt ist der Scheitelpunkt, an dem der Winkel zwischen den beiden Schenkeln zum ersten und dritten Punkt gemessen wird (ein vierter Klick beginnt neu). Berechnung über das Skalarprodukt der beiden Schenkelvektoren, `Winkel = acos((v1·v2)/(|v1|·|v2|))` in Grad (echte räumliche 3D-Vektoren, nicht auf eine Ebene projiziert); „Gegenwinkel" wird als Explementärwinkel verstanden (`360° - Winkel`). Das Panel zeigt zusätzlich zu Winkel/Gegenwinkel je einmal pro Schenkel dieselben ΔX/ΔY/ΔZ/„Direkt (3D)"-Zeilen wie bei der Längenmessung, mit Präfix „Schenkel 1"/„Schenkel 2". Ein kleiner Bildschirm-Bogen am Scheitelpunkt dient als rein optische Orientierungshilfe.

**Sichtbarkeit des Info-Fensters:** `#infoPanel` (umschließt `#readout` und `#measurePanel`) ist sichtbar, sobald entweder Bahnpunkte eines geladenen CNC-Programms existieren **oder** eine der beiden Messfunktionen aktiv ist (`updateInfoPanelVisibility()`, aufgerufen aus `setScope()` sowie aus `toggleMeasure()`/`toggleMeasureAngle()`) — dadurch bleibt das Messergebnis auch beim reinen DXF-Betrachten ohne geladenes CNC-Programm sichtbar. `#readout` selbst bleibt weiterhin ausschließlich an vorhandene Bahnpunkte gekoppelt.

**Tastenkürzel Shift+L / Shift+W:** globaler `keydown`-Listener auf `window`, geprüft per `e.code` (`'KeyL'`/`'KeyW'`, unabhängig von Tastaturlayout/Feststelltaste) zusammen mit `e.shiftKey` (und ohne gleichzeitig gedrückte Strg-/Alt-/Cmd-Taste) — ruft dieselbe, unveränderte `toggleMeasure()`/`toggleMeasureAngle()`-Logik auf wie ein Klick auf `#btnMeasure`/`#btnMeasureAngle`, inklusive derselben gegenseitigen Exklusivität der beiden Werkzeuge. Da es sich um einen echten Umschalter handelt, schaltet ein zweites Drücken derselben Kombination das jeweilige Werkzeug wieder aus. Dieselbe Eingabefeld-Ausnahme wie bei der Pos1-Taste oben (kein Auslösen bei fokussiertem Eingabe-/Textfeld).

**Escape bricht eine aktive Messung ab:** Der globale Escape-Handler (siehe Abschnitt 10 — schließt Menüs/Kontextmenü und bricht offene Dialoge ab) ruft zusätzlich `toggleMeasure()` bzw. `toggleMeasureAngle()` auf, sofern das jeweilige Werkzeug gerade aktiv ist — exakt derselbe Effekt wie ein erneuter Klick auf `#btnMeasure`/`#btnMeasureAngle` oder ein zweites Drücken von Shift+L/Shift+W: Das Werkzeug wird vollständig deaktiviert (nicht nur die bereits gesetzten Fangpunkte verworfen), der Cursor kehrt zum Normalzustand zurück, und das Info-Fenster blendet sich aus, sofern kein CNC-Programm geladen ist (siehe „Sichtbarkeit des Info-Fensters" oben). Läuft unbedingt bei jedem Escape-Druck mit, genau wie das bereits bestehende Schließen von Menüs — ein Escape-Druck kann dadurch gleichzeitig ein offenes Menü schließen UND eine laufende Messung beenden. Ein Escape-Druck ohne aktives Mess-Werkzeug bleibt wirkungslos für diesen Teil des Handlers (kein versehentliches erneutes Aktivieren).

**Escape deaktiviert ebenso einen aktiven Codesprung:** Derselbe Escape-Handler ruft zusätzlich `toggleCodeJump()` auf, sofern `codeJumpActive` gerade `true` ist (siehe 9.4a) — exakt derselbe Effekt wie ein erneuter Klick auf `#btnCodeJump`/den Kontextmenü-Eintrag „Codesprung": Cursor kehrt vom Fadenkreuz zum Normalzustand zurück, die „[]"-Markierung (`selectedLineHighlight`) wird entfernt.

### 9.4a Codesprung: Klick auf CNC-Linie springt zur Programmzeile + „[]"-Markierung

Der Codesprung ist ein dritter, zu den beiden Messwerkzeugen gleichberechtigter Werkzeug-Umschalter — ausgelöst über den Knopf `#btnCodeJump` („Codesprung") in der Symbolleiste, direkt neben „Sprung in Zeile" (siehe 7.1/10.1), sowie gleichwertig über den Kontextmenü-Eintrag „Codesprung" (siehe 10.2). Standardmäßig ist der Codesprung **aus** — ein einfacher Klick/Ziehen im 3D-Bereich verhält sich dadurch rein als Kamera-Bedienung („Hand": drehen/schwenken), zusätzlich weiterhin nutzbar für Längen-/Winkelmessung, sofern eines dieser beiden Werkzeuge aktiv ist.

**Dreifache gegenseitige Exklusivität (`codeJumpActive`):** Alle drei Werkzeuge — Längenmessung, Winkelmessung, Codesprung — konsumieren dieselbe Geste (einfacher Linksklick im 3D-Bereich) und schließen sich deshalb paarweise aus: Aktivieren eines der drei deaktiviert automatisch die beiden anderen (inklusive Zurücksetzen von deren bereits gesetzten Punkten/Markierungen). `toggleCodeJump()` setzt zusätzlich den Cursor auf ein Fadenkreuz (`crosshair`), solange der Codesprung aktiv ist, und räumt beim Ausschalten die „[]"-Markierung (`selectedLineHighlight`) auf, damit kein Markierungsrest stehen bleibt.

**Eigener, bewusst schlankerer Treffertest (`hitTestCncLine()`) statt Wiederverwendung von `hitTest3D()`:** Anders als das Messwerkzeug (9.4, das Punkt-Fangen, Schnittpunkt-Snapping UND DXF-Geometrie einbezieht) zählt hier ausschließlich „auf welcher CNC-Bahnlinie wurde geklickt" — ein reiner Lot-Fußpunkt-Vergleich (`pointToSegmentDist()`) über alle Teilstrecken aller `segments`, mit derselben Fangdistanz `MEASURE_PICK_PX = 14`. Eine DXF-Unterlage wird bewusst **nicht** einbezogen (eine DXF-Linie hat keine zugehörige Programmzeile, ein Sprung wäre dort sinnlos); gedimmte Kontext-Segmente (bereits gefertigtes Bauteil beim Scope „einzelner Arbeitsgang") sind dagegen bewusst mit anklickbar, genau wie beim Messwerkzeug.

**Klick-vs-Kamera-Unterscheidung:** Derselbe Schwellwert wie beim Messwerkzeug — ein Klick zählt nur dann als Auswahl, wenn sich die Maus zwischen Drücken und Loslassen um höchstens 6px bewegt hat (sonst wurde stattdessen die Kamera bedient) —, zusätzlich läuft der Line-Jump-Klick nur, wenn der Codesprung per Button/Kontextmenü explizit eingeschaltet ist (`codeJumpActive`).

**Codesprung:** `handleLineJumpClick()` ruft mit der Zielpunkt-Zeile (`p1.lineIdx`, ersatzweise `p0.lineIdx`, falls der Zielpunkt keine eigene Zeile trägt) exakt dieselbe `jumpToCodeLine()`-Funktion auf, die auch beim Klick auf einen Eintrag der Arbeitsgänge-Spalte verwendet wird (7.2) — scrollt die Code-Liste zur betroffenen Zeile UND bewegt die Wiedergabeposition (Scrubber) an dieselbe Stelle, inklusive derselben kurzen Aufblink-Animation (`.jump-flash`).

**„[]"-Markierung (`selectedLineHighlight`/`drawLineSelection()`):** Start- und Endpunkt der angeklickten Teilstrecke werden gemerkt und **jeden Frame frisch** (nicht über die Bake-Canvas, siehe 9.5) als kleines „[]"-Textglyph an der jeweils aktuellen Bildschirmposition gezeichnet (Akzentfarbe, `600 16px "Open Sans"`, zentriert) — bleibt dadurch auch bei Kamera-Drehung/-Zoom korrekt an der Bahn „haften", bis entweder eine andere Linie angeklickt, der Codesprung ausgeschaltet oder der Anzeigebereich gewechselt wird (`setScope()` setzt `selectedLineHighlight` konsequent zurück, analog zu den bereits bestehenden Mess-Punkten `measureStart`/`measureEnd`/`anglePoints`).

### 9.5 Rendering

Canvas-2D, `devicePixelRatio`-bewusst skaliert (`fitCanvas()`), Zeichnung per `requestAnimationFrame`-Batching (`requestRender()`/`rafPending`). Jeder Frame zeichnet in dieser Reihenfolge: Hintergrund, Bodenraster, Achsen-Gizmo, DXF-Unterlage (siehe 9.8, damit sie optisch unter der Fräskontur liegt), alle Bahn-Segmente (Eilgang gestrichelt, Vorschub durchgezogen, sofern nicht per „Eilgang anzeigen"-Kontrollkästchen ausgeblendet), der aktuelle Positionsmarker, das Werkzeug samt Halter (siehe 9.7), zuletzt das Messwerkzeug-Overlay und das Ansichts-Gizmo.

**Performance-Architektur:** Drei zusammenwirkende Techniken halten das Rendering auch bei großen Programmen/Dateien flüssig:

1. **Gebündelte Zeichenaufrufe:** `drawPolyline()` fasst aufeinanderfolgende Punkte mit identischem Zeichenstil (gedimmter Kontext vs. aktive Bahn, Eilgang vs. Vorschub) in einem einzigen zusammenhängenden Pfad (`beginPath()`/mehrere `lineTo()`/ein `stroke()`) statt für jedes Punktepaar neu anzusetzen. Dieselbe Technik verwendet `strokeGroupedByColor()` für die DXF-Unterlage: Alle Strichzüge werden nach ihrer aufgelösten Anzeigefarbe gruppiert (siehe 9.8) und je Farbe in einem Pfad gezeichnet — bei einheitlicher Farbe (kein Programm geladen bzw. nur ein Layer) genügt dadurch ein einziger `stroke()`-Aufruf für die komplette DXF-Datei statt eines Aufrufs je Entität.
2. **Offscreen-„Bake-Canvas" mit inkrementellem Anhängen:** Zusätzlich zum sichtbaren `<canvas id="canvas3d">` gibt es eine unsichtbare, gleich große Offscreen-Canvas (`bakeCanvas`/`bakeCtx`, `ensureBakeCanvas()`), auf die Raster/Achsen/DXF/Bahn „gebacken" werden. Bei jedem `render()`-Aufruf prüft `computeBakeParams()`/`bakeParamsEqual()`, ob sich seit dem letzten Bake etwas **Strukturelles** geändert hat (Kamera-Winkel/-Distanz/-Ziel/-Roll, Canvasgröße, „Eilgang anzeigen", Wiedergabe gestartet/nicht gestartet, Sichtbarkeit einzelner Arbeitsgänge, Hintergrundfarbe, DXF-Geometrie/-Verschiebung/-Sichtbarkeit/-Spiegelung/-Drehung/-Bauteilstärke/-Layer-Sichtbarkeit, `segments`-Referenz nach Programmwechsel). Ist das der Fall (oder wurde rückwärts gescrubbt), wird einmal komplett neu gebacken; hat sich dagegen nur die Wiedergabeposition vorwärts bewegt, wird pro Bahn-Segment lediglich der neu hinzugekommene Punktebereich inkrementell angehängt. Die fertige Bake-Canvas wird per `drawImage()` auf die sichtbare Canvas kopiert; Positionsmarker, Werkzeug und Messwerkzeug werden dagegen **jeden Frame frisch** direkt auf die sichtbare Canvas gezeichnet, damit sie nie „einfrieren". Rückwärts-Scrubben wird erkannt (Aufdeckungslänge eines Segments unter den zuvor gebackenen Stand gefallen) und erzwingt einen vollen Rebake, da die Bake-Canvas rein additiv ist.
3. **Strukturell feste Zeilenhöhe im Code-Panel** (siehe 7.1) — verhindert teure Flex-Neuberechnungen der gesamten Code-Liste bei jeder Zeilen-Hervorhebung, unabhängig vom Canvas-Rendering selbst.

**Leerer Viewer:** Solange kein CNC-Programm geladen ist, zeigt der 3D-Bereich nur Raster/Achsen (und ggf. eine DXF-Unterlage) ohne jeden Hinweistext. Ein Hinweistext (`#emptyViewerMsg`, „Keine ISO-Bewegungsdaten (X/Y/Z) in diesem Bereich erkannt.") erscheint ausschließlich, wenn ein Programm zwar geladen ist, aber im aktuell gewählten Anzeigebereich (Scope, siehe 9.3) keine auswertbaren Bewegungsdaten liefert.

### 9.6 Readout & Info-Fenster

Zeigt zur aktuellen Position: Quelltext-Zeilennummer, X/Y/Z (2 Nachkommastellen, „mm"), die im Programm tatsächlich genutzten Dreh-/Kippachsen (`AXIS_META`: `C` = „Dreh C", `A` = „Kipp A", `B` = „Kipp B", jeweils mit Hinweistext „…winkel (typisch, maschinenabhängig)"), bei vorhandenen Werkzeugdaten zusätzlich Werkzeug-Durchmesser (`toolRadius*2`) und -Länge bzw. bei einem Sägeaggregat die Sägeblattstärke (siehe 9.7), sowie ein Eilgang/Vorschub-Tag.

**Gemeinsames, frei verschiebbares Info-Fenster:** `#readout` und `#measurePanel` (siehe 9.4) sitzen gemeinsam in einem Container `#infoPanel`, standardmäßig unten links im 3D-Bereich (oberhalb der Copyright-Zeile). Ein dünner Trennstrich erscheint automatisch nur, wenn beide gleichzeitig sichtbar sind. Die beiden Mess-Icon-Knöpfe (`#btnMeasure`/`#btnMeasureAngle`, siehe 9.4) sitzen als eigene Kopfzeile `#infoPanelHead` oben im Panel, mit einer 34×34px-Klickfläche und zentriertem Icon (`display: flex; align-items: center; justify-content: center`), zeigen nur ihr Icon ohne Textlabel und tragen zusätzlich ein Griff-Symbol „⠿", das die Ziehbarkeit signalisiert.

Ein Drag-Mechanismus (`initInfoPanelDrag()`, `pointerdown`/`pointermove`/`pointerup` mit `setPointerCapture()`) erlaubt es, das gesamte Panel per Ziehen an der Kopfzeile frei innerhalb des 3D-Bereichs zu verschieben (geklemmt auf den sichtbaren Bereich des Viewers) — ein Ziehvorgang beginnt bewusst nicht, wenn der Klick auf einem der beiden Mess-Icons selbst beginnt, damit diese weiterhin normal anklickbar bleiben. Die aktuelle Position ist reine Sitzungs-/Layout-Kosmetik und wird nicht persistiert. Da „Messen"/„Winkel" Teil desselben Containers sind, bewegen sie sich bei jedem Drag automatisch mit.


### 9.7 Werkzeug-3D-Darstellung

Erkannte Werkzeugdaten (Radius/Länge, ggf. Sägeblatt-Kennzeichnung) werden als Drahtgitter-Zylinder plus Werkzeughalter an der aktuellen Wiedergabeposition dargestellt, unter Berücksichtigung von Dreh-/Kippwinkeln und Radiuskorrektur.

#### 9.7.1 Werkzeugdaten-Erkennung im Programmtext (`extractToolData()`, `parser.js`)

Läuft je erkanntem Arbeitsgang über dessen eigenen Zeilenbereich, bewusst auf den **ungetauschten** Rohzeilen (`rawProgramText`, vor Anwendung des aktiven Maschinentyps) — analog zur Struktur-Erkennung (`detectionText`, 4.5): Eine SWITCHCASEMACHINE-Regel wie „R->A" würde sonst auch das Wort „RADIUS" selbst verstümmeln. Erkannt werden fünf unabhängige, je einmal pro Abschnitt gesuchte Marker (erster Treffer gewinnt):

- **Radius/Länge**, zwei Notationen: Reichenbacher/Sinumerik-Werkzeugparameter `$TC_DP6[110,1]=6.000` (Radius, Parameter 6) / `$TC_DP3[110,1]=133.000` (Länge, Parameter 3); oder ein reines Schlüsselwort `RADIUS`/`LENGTH` (bzw. rückwärtskompatibel `LAENGE`/`LÄNGE`) irgendwo in der Zeile, direkt gefolgt vom Zahlenwert, umgeben von einem beliebigen Kommentarzeichen — `{ RADIUS 9.695 }` (Moroff), `; RADIUS 9.695` (SCM), `( RADIUS 9.695 )` (KRC), `KM="RADIUS 9.695"` (HOMAG), `# RADIUS 9.695` (FORMAT4).
- **`BLADE`** (`TOOL_BLADE_RE = /BLADE\s+(-?\d+(?:[.,]\d+)?)/i`, Komma oder Punkt als Dezimaltrennzeichen): Bei Sägeaggregaten beschreibt `LENGTH` nicht die tatsächliche Werkzeuglänge, sondern eine davon unabhängige Kenngröße — die für die 3D-Darstellung relevante Sägeblattdicke steht stattdessen in einer separaten `BLADE`-Zeile. Ist eine solche Zeile vorhanden (und wurde für diesen Arbeitsgang noch nie ein Werkzeugtyp per Dialog bestätigt, siehe unten), gewinnt ihr Wert als `length`, unabhängig von einer eventuell vorhandenen `LENGTH`-Zeile, und `isBlade` wird `true`.
- **`TOOLMODE`** (`/TOOLMODE\s+(EINGESPANNT|AUSGESPANNT)/i`): reiner Anzeige-/Darstellungs-Schalter, beeinflusst nicht den Zahlenwert selbst (siehe 9.7.4).
- **`TOOLTYPE`** (`/TOOLTYPE\s+(SAEGE|FRAESER)/i`): manuelle Werkzeugtyp-Festlegung, siehe unten.

**Zusammenspiel `BLADE`/`TOOLTYPE`/`LENGTH`:** Ist für einen Abschnitt eine `TOOLTYPE`-Markierung vorhanden (der „+Werkzeuge (T)"-Dialog wurde für diesen Arbeitsgang also mindestens einmal bestätigt), bestimmt sie `isBlade` (`toolType === 'saege'`) mit Vorrang vor einer eventuell noch vorhandenen rohen `BLADE`-Zeile, UND eine vorhandene `LENGTH`-Zeile gewinnt in diesem Fall immer gegen einen `BLADE`-Wert (nur ohne eigene `LENGTH`-Zeile wird auf `BLADE` zurückgegriffen). Fehlt die `TOOLTYPE`-Markierung, gilt weiterhin: eine vorhandene `BLADE`-Zeile gewinnt unbedingt gegen `LENGTH`, und `isBlade` folgt allein aus ihrer Anwesenheit. Dadurch bleibt sowohl das reine Erkennen einer realen Sägeprogramm-Datei (nur `BLADE`, nie manuell bearbeitet) als auch das nachträgliche Umschalten Säge↔Fräser per Dialog (inklusive neu eingegebener Länge) konsistent funktionsfähig.

`extractToolData()` liefert `{radius, length, isBlade, toolMode, toolType}`. `app.js` (`displayCurrentProgram()`) ruft die Funktion pro erkanntem Arbeitsgang auf und hinterlegt das Ergebnis am jeweiligen `sec` (`sec.toolRadius`/`toolLength`/`toolIsBlade`/`toolMode`/`toolType`); ohne Treffer bleiben `toolRadius`/`toolLength` `null` und es wird nichts gezeichnet.

#### 9.7.2 Zylinder-Geometrie (`drawTool(proj)`, direkt nach `drawMarker()`)

Gezeichnet wird — zusätzlich zum Positionsmarker — ein Drahtgitter-Zylinder für den Punkt an der aktuellen Wiedergabeposition (`points[scrubIdx]`), sofern für dessen Arbeitsgang Radius UND Länge gefunden wurden. Darstellung: zwei 16-Segment-Kreise (Grund-/Deckfläche) plus jede 4. Mantellinie, in `--tool-color`.

**Normales Werkzeug (Fräser):**
- **Spitze/Tiefe:** Die Zylinder-Grundfläche liegt exakt auf dem programmierten Punkt (ggf. inkl. Radiuskorrektur-Versatz, siehe unten) — die Tiefe im Programm (Z) gibt immer die Spitze des Fräsers wieder. Der Zylinder erstreckt sich von dort um `toolLength` entlang der Werkzeugachse weg vom Werkstück.
- **Durchmesser:** `toolRadius*2`, in derselben Skala (mm) wie die Bahn selbst.
- **Werkzeugachse / Dreh- & Kippwinkel:** Ausgehend von der unverkippten Achse `(0,0,1)` werden die modal mitgeführten Winkel `state.a/b/c` als Rotationen angewendet, in der Reihenfolge erst Kipp A (um X), dann Kipp B (um Y), zuletzt Dreh C (um Z). Bei nur einem gleichzeitig aktiven Winkel (der praktisch häufigste Fall) ist die Reihenfolge irrelevant.
- **Radiuskorrektur G41/G42:** `state.comp` (`'left' | 'right' | 'none'`) wird unabhängig von der Eilgang-/Vorschub-Erkennung an eigenen Mustern (`/\bG41\b/`, `/\bG42\b/`, `/\bG40\b/`) festgemacht. Der gezeichnete/simulierte Bahnpunkt bleibt immer die tatsächlich gefräste Kontur; nur die Position des Werkzeug-**Körpers** wird zusätzlich senkrecht zur Bewegungsrichtung versetzt — bei G41 („links") nach links, bei G42 („rechts") nach rechts, ohne Radiuskorrektur (Mittelpunktsbahn) gar nicht. Die Bewegungsrichtung ermittelt `tangentAt(idx)` aus den Nachbarpunkten desselben Segments (Mittel aus ein- und auslaufender Richtung); „links" ist die 90°-Drehung gegen den Uhrzeigersinn.

**Reichenbacher-Kardanwinkelkorrektur:** Reichenbacher-Programme geben den Kippwinkel als „Kipp B" (`state.b`) an — physisch handelt es sich dabei um einen kardanischen Schwenkkopf, dessen Schwenkachse fest um einen konstruktionsbedingten Schwenkwinkel `S` (per `SWIVELANGLE`, siehe 5.12) zwischen der Y- und der Z-Achse geneigt ist, statt parallel zur Y-Achse zu liegen. `reichenbacherToolAxisDirection(p)` ersetzt für diesen Fall die generische Rotationsformel durch die in 5.12 angegebene Kardanformel; `toolAxisDirection()` ruft sie anstelle der generischen Berechnung auf, sobald `reichenbacherCorrectionActive()` `true` liefert.

**Schalter „Kardan-Winkelkorrektur":** Ein Kippschalter (`#reichenbacherSwitchWrap`/`#chkReichenbacherCorrection`) sitzt in der Kopfzeile direkt rechts neben dem Dateinamen (siehe Abschnitt 10) und ist nur sichtbar, solange der zuletzt per „Ok" bestätigte Maschinentyp den Teilstring „reichenbacher" enthält (`isReichenbacherTypeName()`, siehe 5.12). Wird er dabei neu sichtbar, springt er auf „an" zurück (Standardzustand); bleibt er durchgehend sichtbar, wird ein manuell gesetzter Zustand nicht überschrieben. Ausgeschaltet zeigt er zum Vergleich die unkorrigierte generische Berechnung.

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

**Schnittrichtung am Schnittstart (`sawCutDirectionAt()`, statt der generischen `tangentAt()`):** Bei einem typischen Sägeschnitt fährt ein Eilgang (G0) das Blatt seitlich der Kontur an, erst der darauffolgende Vorschub (G1) ist der eigentliche Schnitt in die tatsächliche Schnittrichtung. `tangentAt()` würde am Übergangspunkt die (oft ganz andere) Eilgang-Anfahrrichtung mit der nachfolgenden Schnittrichtung mitteln und dadurch ein diagonal „schräg stehendes" Sägeblatt ergeben. `sawCutDirectionAt(idx)` bevorzugt deshalb gezielt die tatsächliche Vorschub-Richtung:

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

**„T-Längen" — Eingespannte/Ausgespannte Länge:** `TOOLMODE` (siehe 9.7.1) steuert, wie weit die Halter-Baugruppe (HSK63-Halter, ggf. Adapterkegel) den ansonsten unveränderten Zylinder überlappt:

```js
const clampInset = (p.toolMode === 'eingespannt') ? HSK63_CLAMP_INSET /* = 24.98mm */ : 0;
const holderAttach = clampInset > 0
  ? { x: top.x - axis.x*clampInset, y: top.y - axis.y*clampInset, z: top.z - axis.z*clampInset }
  : top;
```

Bei „Ausgespannt" (Standard, `clampInset = 0`) sitzt die Halter-Baugruppe wie bisher direkt am oberen Zylinderende; bei „Eingespannt" rückt sie um 24,98mm (die Länge des schmalen Ansatzstücks des HSK63-Profils) über das letzte Stück des unverändert vollen Zylinders. Der Zylinder selbst hat in beiden Fällen exakt die volle eingegebene Länge.

**Unabhängige Ein-/Ausblendbarkeit:** `showTool` (Werkzeug-Zylinder, Knopf „👁 Werkzeug") und `showToolHolder` (komplette Spannfutter-Baugruppe — HSK63-Halter samt ggf. vorhandenem Adapterkegel, Knopf „👁 Spannfutter") sind zwei unabhängige Zustandsvariablen; `drawTool()` berechnet die gemeinsam benötigte Geometrie (Achse/Ansatzpunkte) unbedingt weiter, zeichnet aber Zylinder und Halter-Baugruppe in getrennten `if`-Blöcken — jeweils eines der beiden kann unabhängig vom anderen ausgeblendet bleiben.

#### 9.7.5 Bedienung — „+Werkzeuge (T)"-Dialog

Werkzeugleisten-Knopf „+Werkzeuge (T)" (`#btnInsertToolData`) ist nur aktiv, solange der 3D-Scope genau auf einen einzelnen Arbeitsgang gesetzt ist (über dessen „3D"-Knopf, siehe 7.2 — ein bloßer Klick auf die Arbeitsgang-Zeile selbst genügt nicht). Öffnet `#toolDataPanel` mit vier Feldern, in dieser Reihenfolge:

1. **Werkzeugtyp** (`#toolTypeSelect`, „Fräser"/„Säge") — vorbelegt mit „Säge", sobald `sec.toolIsBlade` für diesen Arbeitsgang bereits gesetzt ist, sonst „Fräser". Ein Umschalten ändert **sofort**, noch ohne „Ok", sowohl die Beschriftung des zweiten Felds als auch dessen Werte (`applyToolTypeUI()`, bei `change` UND beim Öffnen aufgerufen) — siehe Punkt 2.
2. **Radius (R)** / **Länge (L)** bzw. **Sägeblattstärke** (Feldbeschriftung abhängig vom Werkzeugtyp) — vorbelegt mit den für diesen Arbeitsgang bereits erkannten Werten, **sofern sie zum gerade gewählten Werkzeugtyp gehören** (siehe unten), sonst mit typspezifischen Standardwerten: „Fräser" → **8**/**200**, „Säge" → **150**/**3,5**. Komma als Dezimaltrennzeichen wird akzeptiert und beim Übernehmen zu einem Punkt normalisiert (`normalizeToolValue()`, verlustfreie Ziffernübernahme ohne Umweg über `parseFloat`/`toString`).
3. **T-Längen:** (`#toolModeSelect`, „Ausgespannte Länge"/„Eingespannte Länge") — vorbelegt mit `sec.toolMode`, Standard „Ausgespannte Länge".

**Einfügeposition:** Direkt unter dem gesamten erkannten Arbeitsgang-Kommentar-/Titelblock (`sec.headerEndIdx`, siehe 4.5), in `rawProgramText`, als bis zu vier eigene Kommentarzeilen `; RADIUS <Wert>` / `; LENGTH <Wert>` / `; TOOLMODE …` / `; TOOLTYPE …` — dieselbe Notation, die `extractToolData()` ohnehin erkennt. Stehen an der Einfügeposition bereits Zeilen desselben Musters (von einem vorherigen Klick), werden deren Werte aktualisiert statt eine weitere Kopie einzufügen. Der 3D-Scope bleibt nach dem Bestätigen auf demselben Arbeitsgang, und die Kamera (`cam.target`/`cam.dist`/`cam.minEyeDist`/`cam.maxDist`) wird explizit gesichert und nach dem Neuanzeigen unverändert zurückgeschrieben, damit ein zuvor manuell eingestellter Zoom nicht durch den ansonsten bei jedem Neuanzeigen ausgelösten `frameCamera()`-Aufruf verloren geht.

**Typspezifische Standardwerte, live beim Umschalten:** `applyToolTypeUI()` (siehe Punkt 1 oben) aktualisiert beim Öffnen des Dialogs sowie bei jedem späteren Umschalten des Dropdowns innerhalb des bereits geöffneten Dialogs sowohl die Beschriftung als auch die Radius-/Längenfelder selbst: Für den gerade gewählten Werkzeugtyp gelten die für DIESEN Arbeitsgang bereits erkannten realen Werkzeugdaten (`sec.toolRadius`/`sec.toolLength`) nur dann, wenn sie tatsächlich zum gerade gewählten Typ gehören (`sec.toolIsBlade` entspricht der Auswahl) — andernfalls (für diesen Typ liegen in diesem Arbeitsgang keine erkannten Daten vor) greifen die typspezifischen Standardwerte. Ein Arbeitsgang mit bereits erkannten echten Sägeblattdaten zeigt beim Umschalten auf „Fräser" deshalb die Fräser-Standardwerte 8/200 (für den Fräser sind ja keine echten Daten bekannt), schaltet man von dort wieder zurück auf „Säge", erscheinen erneut die echten, zuvor erkannten Sägeblattdaten statt der generischen 150/3,5.

**Schließen nur über „Ok"/„Abbrechen"/Enter/Escape:** Siehe den eigenen Absatz „Kein Schließen mehr durch Klick auf den Hintergrund" in Abschnitt 11 (Editoren) — betrifft alle `.paste-panel`-Dialoge der Anwendung gleichermaßen, nicht nur „+Werkzeuge (T)".


### 9.8 DXF-Import

Ein separater, vom CNC-Programm unabhängiger Import einer DXF-Zeichnung als Referenz-/Unterlage im 3D-Bereich (z. B. um die simulierte Fräskontur gegen eine vorgegebene Zeichnung zu prüfen, auch direkt anmessbar, siehe 9.4). Laden, Bearbeiten oder Löschen des einen (CNC-Programm bzw. DXF) wirkt sich nicht auf den Zustand des anderen aus — `dxfGeometry`/`dxfFilename`/`dxfOffset`/`dxfVisible` u. a. werden weder von `loadText()`/`clearProgram()`/`displayCurrentProgram()` noch umgekehrt `loadDxfText()`/`clearDxf()` angefasst.

**Ladewege:** Menüpunkt „DXF Importieren" (`#btnDxfImport`, im „Datei"-Menü) bzw. Symbolleisten-Icon „📂 DXF" (`#btnDxfImportIcon`) öffnen denselben nativen Dateiauswahl-Dialog; zusätzlich per Drag & Drop derselben fensterweiten Dropzone wie das CNC-Programm, unterschieden anhand der Dateiendung (`/\.dxf$/i`, case-insensitiv). Die gewählte Datei wird wie ein CNC-Programm über `readFileAsText()`/`decodeBuffer()` eingelesen (automatische Kodierungserkennung, siehe Abschnitt 3) und per `parseDXF()` geparst. Ein erneuter Import ersetzt eine zuvor geladene DXF-Unterlage vollständig (inkl. Zurücksetzen von Verschiebung, Spiegelung, Drehung, Bauteilstärke, Layer-Sichtbarkeit und Gesamt-Sichtbarkeit auf den Ausgangszustand).

**Unterstützte Entitätstypen (`parseDXF()`, `parser.js`):** Ein eigenständiger Parser für das ASCII-DXF-Gruppencode-Format, beschränkt auf den `ENTITIES`-Abschnitt (Geometrie in `BLOCKS` ohne `INSERT`-Platzierung wird ignoriert, Block-Referenzen `INSERT` selbst werden nicht aufgelöst):

- **LINE** — ein Strichzug mit den beiden unveränderten Endpunkten.
- **POINT** — ein Ein-Punkt-Strichzug (wird beim Zeichnen übersprungen, nichts zu verbinden).
- **CIRCLE** — als geschlossener Strichzug tesselliert; Radius ≤ 0 wird übersprungen.
- **ARC** — wie CIRCLE, nicht geschlossen; der „kurze" Weg von Start- zu Endwinkel wird genommen, auch über den 0°/360°-Wraparound hinweg.
- **LWPOLYLINE** / **Alt-Style POLYLINE/VERTEX/SEQEND** — Vertices geradlinig verbunden, außer ein Vertex trägt einen **Bulge**-Wert (42) — dann wird das Segment als Kreisbogen tesselliert (`bulgeSegmentPoints()`, DXF-Standardformel `bulge = tan(θ/4)`; positiver Bulge wölbt nach links der Bewegungsrichtung, negativer nach rechts). Bei geschlossenen Polylinien wird zusätzlich das Schlusssegment zurück zum ersten Vertex gezeichnet. Bei der Alt-Style-Variante wird ein fehlendes `SEQEND` trotzdem beim Dateiende finalisiert, eine `VERTEX` außerhalb einer offenen `POLYLINE` wird ignoriert und nur gezählt.
- **3DFACE** — eine ebene Fläche aus bis zu vier Eckpunkten (Gruppencodes 10/20/30, 11/21/31, 12/22/32, 13/23/33), direkt in Weltkoordinaten (keine OCS-/Extrusionsrichtungs-Umrechnung nötig). Fehlt der vierte Eckpunkt oder wiederholt er exakt den dritten, wird die Fläche korrekt als Dreieck (3 statt 4 Eckpunkte) erkannt. Praktisch alle CAD-Programme exportieren echte 3D-Flächen-/Volumenmodelle als tausende bis zehntausende einzelne 3DFACE-Entitäten — Unterstützung dieses Typs macht dadurch reale 3D-Modelle (z. B. Treppenhaus-/Bauteilmodelle mit zehntausenden Flächen) grundsätzlich importierbar.
- **Nicht unterstützt** (werden erkannt, gezählt — `skippedTypes`, Tooltip der Statusanzeige — aber nicht gezeichnet): u. a. TEXT/MTEXT, HATCH, SPLINE, ELLIPSE. Extrusionsrichtung/OCS (Gruppencode 210/220/230) wird bei CIRCLE/ARC/LWPOLYLINE/POLYLINE nicht berücksichtigt — eine gedrehte, nicht in der Standard-XY-Ebene liegende Entität dieser Typen kann falsch orientiert erscheinen (3DFACE ist davon nicht betroffen, siehe oben).

Segmentanzahl der Bogen-/Bulge-Tessellierung: dieselbe `computeSegmentCount()`-Formel wie bei der G02/G03-Bogeninterpolation (9.1, 4–180 Segmente, ca. 10mm pro Segment).

### 9.8a Layer-Verwaltung

`parseDXF()` liest zusätzlich zur reinen Geometrie die Layer-Zuordnung jeder Entität (Gruppencode 8, Standardlayer `"0"`, falls nicht angegeben) sowie — aus der `TABLES`/`LAYER`-Sektion — eine Layer-Farbtabelle (siehe 9.8b), und liefert `layers` (eine numerisch-alphabetisch sortierte Liste aller tatsächlich gezeichneten Layer-Namen, `localeCompare(..., {numeric:true})`, sodass „LIST2" vor „LIST10" einsortiert wird) sowie `layerColors`.

**Reiter „DXF-Layer":** Die Kopfzeile der Programmcode-Spalte (siehe 7.1b) zeigt je nach Ladezustand entweder nur „CNC-Programmcode", nur „DXF-Layer" oder beide als echtes, anklickbares Reiter-Paar (`updateSidebarHeader()`, Zustand `sidebarView`: `'code'` | `'dxfLayers'`):

- Weder Programm noch DXF geladen, oder nur ein Programm: ausschließlich „CNC-Programmcode", nicht anklickbar.
- Nur ein DXF geladen: ausschließlich „DXF-Layer", nicht anklickbar — `#tree` wird ausgeblendet, stattdessen erscheint `#dxfLayerPanel` mit der Layer-Liste.
- Beide gleichzeitig geladen: beide Reiter sichtbar und anklickbar.

**Priorität beim Laden hat immer das CNC-Programm:** Das Laden eines Programms setzt `sidebarView` immer auf `'code'`, unabhängig vom zuvor aktiven Reiter; das Laden eines DXF setzt `sidebarView` nur dann auf `'dxfLayers'`, wenn noch kein Programm geladen ist. „CNC-Programm löschen" bei weiterhin geladenem DXF schaltet auf „DXF-Layer" zurück.

**Automatische Spaltenbreite für die Layer-Liste:** `computeDxfLayerContentWidth(layers)` misst — nach demselben Canvas-2D-`measureText()`-Prinzip wie `computeTreeContentWidth()`/`computeOpsContentWidth()` (siehe 7.3) — den breitesten Layernamen zzgl. der tatsächlichen Zeilen-/Spalten-Chrome (Checkbox, Farbfeld, Innenabstände von `.dxf-layer-row`/`.dxf-layer-panel`) und berücksichtigt zusätzlich, analog zum „Alle AG ausblenden"-Knopf in `computeOpsContentWidth()`, den „Alle Layer ausblenden"/„Alle Layer anzeigen"-Knopf, damit die Spalte auch bei durchweg kurzen Layernamen nie schmaler als dieser Knopf wird. `applyDxfLayerColumnWidth()` wendet das Ergebnis über `setSidebarWidth()` an — aber **ausschließlich, solange kein CNC-Programm geladen ist** (`rawProgramText == null`): Ist eines geladen, hat dessen Breite (`computeTreeContentWidth()`/`applyProgCollapseStage()`, siehe 7.3) immer Vorrang, unabhängig davon, welcher der beiden Reiter gerade sichtbar ist. Zwei Aufrufer: `loadDxfText()` (jeder DXF-Import/-Re-Import, berechnet die Breite bei jeder neuen Layer-Liste frisch) und `clearProgram()` (ein geladenes Programm wird entfernt, während noch ein DXF geladen ist — die Spalte fällt in diesem Moment sofort auf die Layername-basierte Breite zurück, statt die jetzt gegenstandslose Programm-Breite stehen zu lassen). Ein reiner Reiter-Wechsel zwischen „CNC-Programmcode" und „DXF-Layer" (`sidebarTabCode`/`sidebarTabDxf`) fasst die Spaltenbreite an keiner Stelle an — genau das macht das Hin- und Herwechseln bei gleichzeitig geladenem Programm und DXF ruckelfrei.

**Einzeln aus-/einblendbar:** `#dxfLayerPanel` zeigt je Layer eine Checkbox-Zeile mit Farb-Schwatch (dieselbe Farbe, die `drawDxf()` für diesen Layer beim Zeichnen verwenden würde) und Name. Ein Zustand `dxfHiddenLayers` (Set der ausgeblendeten Layer-Namen, leer = alle sichtbar) wird beim Umschalten aktualisiert und an allen drei Stellen gefiltert, an denen die DXF-Geometrie sonst durchlaufen wird: Zeichnen, Kamera-Framing (`dxfPointsForBounds()`) und Mess-Fangpunkte (`hitTest3D()`) — ein ausgeblendeter Layer ist dadurch weder sichtbar noch anmessbar noch beeinflusst er die automatische Kamera-Rahmung. `dxfHiddenLayers` wird bei jedem neuen DXF-Import zurückgesetzt (alle Layer sichtbar), bleibt aber über einen Reiterwechsel oder ein zusätzlich geladenes CNC-Programm hinweg unverändert erhalten.

**„Alle Layer ausblenden/anzeigen"-Knopf:** Direkt oberhalb der Checkbox-Liste sitzt ein zusätzlicher, volle Breite einnehmender Umschalt-Knopf (`#btnShowAllDxfLayers`, `.ops-showall-btn`), nach exakt demselben Muster wie der „Alle AG ausblenden/anzeigen"-Knopf der Arbeitsgänge-Spalte (`#btnShowAllOps`, siehe 7.2) — dieselbe Zustandslogik (`toggleAllDxfLayers()`/`updateShowAllDxfLayersButton()`) arbeitet auf demselben `dxfHiddenLayers`-Set. Label/Optik richten sich ausschließlich danach, ob `dxfHiddenLayers` leer ist oder nicht:
- **Leer** (kein Layer ausgeblendet, auch ein zuvor gemischter Zustand zählt hier nicht hinein) → Label „Alle Layer ausblenden"; ein Klick blendet **alle** Layer auf einmal aus (`dxfHiddenLayers` wird mit sämtlichen Layer-Namen befüllt).
- **Nicht leer** (ob einzelne oder bereits alle Layer ausgeblendet sind) → Label „Alle Layer anzeigen"; ein Klick setzt `dxfHiddenLayers` vollständig zurück (leere Set) und blendet dadurch wieder **alle** Layer ein — auch aus einem zuvor nur teilweise/gemischt ausgeblendeten Zustand heraus.

Jeder Klick erhöht zusätzlich `dxfHiddenLayersVersion`, ruft `renderDxfLayerList()` (baut die komplette Checkbox-Liste inkl. neuem Anzeigezustand neu auf, siehe oben) und `requestRender()` (3D-Ansicht) auf — identisch zum bestehenden Verhalten eines einzelnen Checkbox-Klicks. Der Knopf bleibt vollständig synchron zu individuellen Checkbox-Klicks: `renderDxfLayerList()`s Checkbox-`change`-Listener ruft nach jeder Einzeländerung ebenfalls `updateShowAllDxfLayersButton()` auf, sodass das Label auch bei einer rein manuellen, gemischten Auswahl sofort korrekt auf „Alle Layer anzeigen" wechselt, ohne dass der Knopf selbst geklickt wurde. Der Knopf ist nur sichtbar, solange tatsächlich mindestens ein Layer vorhanden ist (`layers.length > 0`), und wird bei jedem neuen DXF-Import zusammen mit `dxfHiddenLayers` auf den Ausgangszustand (alle Layer sichtbar, Label „Alle Layer ausblenden") zurückgesetzt.

### 9.8b Darstellung: Farbe und Strichelung

**Strichelung** hängt ausschließlich davon ab, ob gerade ein CNC-Programm geladen ist — live geprüft bei jedem Zeichnen (`rawProgramText != null`), nicht nur einmalig beim DXF-Import festgelegt: Ist keines geladen, wird die DXF-Unterlage durchgezogen gezeichnet; ist eines geladen (unabhängig von der Reihenfolge, in der DXF und Programm geladen wurden), gestrichelt (`ctx.setLineDash([6, 4])`).

**Farbe** wird je Entität über `resolveEntityColor()` aufgelöst, in dieser Rangfolge: (1) eine direkte True-Color an der Entität (Gruppencode 420), (2) ein direkter ACI-Farbindex an der Entität (Gruppencode 62, Werte 1–255; die Sonderwerte 0/256/257 zählen nicht als eigene Farbe), (3) die Farbe des Layers, auf dem die Entität liegt (aus der `TABLES`/`LAYER`-Sektion, ebenfalls bevorzugt 420, sonst 62), (4) keine — Standardfarbe `--dxf-color`. Ein ACI-Index wird über eine intern hinterlegte 256-Farben-Tabelle (`ACI_RGB`) in RGB umgerechnet.

**Farbe wird nur angezeigt, wenn beide Bedingungen erfüllt sind** (`showLayerColors = multiLayer && !dashed`):
- **`multiLayer`** — die DXF-Datei nutzt tatsächlich **mindestens zwei** unterschiedliche Layer (`layerCount >= 2`, gezählt über die tatsächlich gezeichneten Strichzüge, nicht über alle in `TABLES`/`LAYER` definierten Einträge). Bei genau einem (oder keinem) tatsächlich genutzten Layer bleibt die DXF unabhängig von einer eventuell dennoch auflösbaren Farbe einheitlich in `--dxf-color` — eine einfache Einzel-Layer-DXF (der weit überwiegende Normalfall, da viele CAD-Programme einem neuen Layer automatisch eine Farbe zuweisen) sieht dadurch unverändert neutral aus statt unerwartet bunt zu werden.
- **`!dashed`** — es ist gerade **kein** CNC-Programm geladen. Sobald ein Programm geladen ist, zeigt die DXF-Unterlage unabhängig von der Layer-Anzahl ausschließlich die einheitliche `--dxf-color` (und gestrichelt) — Layerfarben erscheinen nur beim reinen DXF-Betrachten ohne geladenes Programm.

Die reine Layer-**Erkennung**/-Zählung (`layerCount`, `layers`, für die „DXF-Layer"-Liste, siehe 9.8a) ist von dieser Anzeige-Einschränkung unberührt — nur die tatsächliche farbliche Darstellung wird zusätzlich unterdrückt.

### 9.8c Verschieben, Spiegeln, Drehen, Skalieren, Bauteilstärke

Fünf Transformationen, jeweils über einen eigenen Werkzeugleisten-Knopf bedient (`#toolbarRow2`, siehe Abschnitt 10; alle deaktiviert ohne geladenes DXF), zentral in `dxfTransformPoint(p)` zusammengefasst und in dieser Reihenfolge angewendet: Skalierung → Spiegelung → Drehung → Anker-Korrektur → `dxfOffset`. Dieselbe Funktion wird von `drawDxf()`, `dxfPointsForBounds()` (Kamera-Framing) und `hitTest3D()` (Messfunktion) gleichermaßen verwendet, damit Zeichnung, Kamera-Framing und Fangpunkt-Ermittlung immer exakt dieselbe Geometrie sehen.

**Automatisches „Home" nach Spiegeln/Drehen/Skalieren:** `goHomeView()` (siehe 9.2/„Home"-Knopf) setzt Blickrichtung auf das Oben-Preset zurück und ruft `frameCamera()` erneut auf; sie wird zusätzlich automatisch aufgerufen, sobald Spiegeln, Drehen oder Skalieren (inkl. des „Originalgröße"-Knopfs) tatsächlich etwas ändern — unabhängig davon, wie die Kamera zuvor per Hand gedreht/gezoomt wurde, landet man danach sofort wieder bei einer passend eingerahmten Oben-Ansicht der neuen Kontur, statt sie erst manuell wiederfinden zu müssen (insbesondere nach einer starken Skalierung oder einer Drehung kann die Kontur sonst weit außerhalb des bisherigen Bildausschnitts liegen). **Bewusst ausgenommen bleiben Verschieben und Bauteilstärke** — siehe deren jeweilige Beschreibung unten für die Begründung.

- **Verschieben** (`#btnDxfMoveIcon`, dazu gleichwertig der Menüpunkt „DXF Verschieben" im „Bearbeiten"-Menü): Dialog `#dxfMovePanel` mit drei Feldern X/Y/Z, vorbelegt mit dem aktuellen `dxfOffset` (Ausgangswert 0/0/0). „Ok" übernimmt und zeichnet sofort neu, **ohne** die Kamera zu verändern (kein `goHomeView()`/`frameCamera()`-Aufruf) — bewusst weiterhin unverändert, damit sich die Verschiebung visuell gegen die feststehende Kontur beurteilen lässt (anders als bei Spiegeln/Drehen/Skalieren bleibt die Kontur bei einer Verschiebung typischerweise im selben Bildausschnitt sichtbar, ein Home-Sprung würde hier eher den direkten Vorher/Nachher-Vergleich stören als helfen).
- **Spiegeln** (`#btnDxfMirrorX`/`#btnDxfMirrorY`, „⬍ DXF"/„⬌ DXF") — reine Umschalter ohne Dialog, sofortige Wirkung, unabhängig voneinander gleichzeitig aktivierbar (beide zusammen ergeben eine Punktspiegelung am Ursprung). Wirken um den DXF-eigenen Koordinatenursprung (0,0) der Rohgeometrie, vor `dxfOffset`. Löst danach `goHomeView()` statt eines reinen `requestRender()` aus (siehe oben).
- **Drehen** (`#btnDxfRotate`, „↻ DXF") — Dialog mit einem Winkel-Feld (°), wirkt ebenfalls um den DXF-eigenen Ursprung. „Ok" löst danach `goHomeView()` aus (siehe oben).
- **Skalieren** (`#btnDxfScale`, „🔍 DXF", rechts neben „🧱 DXF"/Bauteilstärke) — Dialog `#dxfScalePanel` mit einem Multiplikator-Feld sowie einem informativen Hinweistext `#dxfScaleCurrentHint` und einem „Originalgröße"-Knopf. Intern hält `dxfScale` (Ausgangswert 1) den **Gesamtfaktor relativ zur Rohgeometrie** beim Import, angewendet auf **X, Y und Z** gleichermaßen (nicht nur auf die Ebene), da eine falsch exportierte Einheit typischerweise alle drei Achsen gleichermaßen betrifft (u. a. die Z-Werte von 3DFACE-Entitäten, siehe 9.8). Die separat vom Benutzer in mm eingegebene Bauteilstärke (siehe unten) bleibt davon unberührt — sie ist eine eigenständige, von der DXF-Einheit unabhängige physische Maßangabe.
  - **Eingabe ist relativ zur aktuell sichtbaren Größe:** Die Dialog-Eingabe ist ein reiner **Multiplikator auf die aktuell sichtbare Größe**: `confirmDxfScale()` multipliziert den bestehenden `dxfScale` mit dem eingegebenen Faktor (`dxfScale *= Eingabe`), statt ihn zu ersetzen — mehrfache Skalierungen verketten sich dadurch wie erwartet (0,1 gefolgt von 10 ergibt wieder Faktor 1, also die Originalgröße). `openDxfScalePanel()` befüllt das Eingabefeld deshalb bei jedem Öffnen neutral mit „1" statt mit dem aktuellen Gesamtfaktor, damit sich ein versehentlich erneut eingetippter Wert nicht ungewollt aufmultipliziert. Der seit dem Import angesammelte Gesamtfaktor wird stattdessen informativ im Hinweistext „Aktuelle Gesamtskalierung seit Import: ×…" angezeigt.
  - **„Originalgröße"-Knopf:** setzt `dxfScale` direkt und absolut auf 1 zurück, ohne dass der Kehrwert der bisher verketteten Faktoren von Hand ausgerechnet werden muss. Löst ebenfalls `goHomeView()` aus.
  - Eine ungültige oder nicht positive Eingabe (0, negativ, leer) wird beim Bestätigen wie ein Faktor 1 behandelt, verändert den Gesamtfaktor also **nicht** (No-op), da 0 die Geometrie auf einen Punkt kollabieren ließe und ein negativer Faktor eine versteckte zusätzliche Spiegelung wäre.
  - „Ok" übernimmt und zeichnet sofort neu und löst dabei `goHomeView()` aus (siehe oben).
- **Bauteilstärke** (`#btnDxfThickness`, „🧱 DXF") — Dialog mit einem Stärke-Feld (mm). Da die DXF-Unterlage die Aufstandsfläche/Oberseite des Werkstücks darstellt, wird eine Stärke > 0 nach unten (kleineres Z) extrudiert: `drawDxf()` zeichnet zusätzlich zur Kontur dieselbe Kontur nochmal um die Stärke tiefer (etwas transparenter, als „Unterseite" erkennbar) sowie dünne senkrechte Verbindungslinien an jedem Konturpunkt — ein reines Drahtgitter-Volumen. Ober- und Unterseite werden aus demselben, bereits transformierten Punkte-Array gebaut und folgen dadurch automatisch jeder Skalierung/Spiegelung/Drehung/Verschiebung (die Unterseite ist rechnerisch identisch zur Oberseite, nur um die Stärke in Z versetzt). `dxfPointsForBounds()` bezieht bei gesetzter Stärke auch die Unterseiten-Punkte mit ein. Bauteilstärke löst bewusst **kein** automatisches „Home" aus — eine reine Z-Extrusion verändert die von oben sichtbare Ausdehnung der Kontur ohnehin nicht.

**Untere linke Ecke bleibt bei (0,0):** Da Skalieren/Spiegeln/Drehen um den DXF-eigenen Ursprung wirken, könnte eine Kontur, deren untere linke Ecke ursprünglich nicht bei (0,0) liegt, danach an anderer Stelle (z. B. in negativen Koordinaten) landen. Ein Korrekturversatz `dxfAnchorCorrection = {x, y}` (`recomputeDxfAnchorCorrection()`) durchläuft dafür die komplette Rohgeometrie, wendet dieselbe Skalierungs-/Spiegel-/Dreh-Rechnung wie `dxfTransformPoint()` an und ermittelt daraus `minx`/`miny` der resultierenden (noch nicht per `dxfOffset` verschobenen) Punktwolke; `dxfAnchorCorrection = {-minx, -miny}` wird vor `dxfOffset` addiert. Die untere linke Ecke der transformierten Kontur liegt dadurch unabhängig von ihrer ursprünglichen Lage und unabhängig von der gewählten Kombination aus Skalieren/Spiegeln/Drehen immer exakt bei (0,0), genauso wie beim ersten, unveränderten Import — ein zusätzlich über „DXF verschieben" gesetzter `dxfOffset` wirkt weiterhin als reine zusätzliche Verschiebung ab dieser 0/0-Ankerposition. Da eine gleichmäßige Skalierung (derselbe Faktor für X und Y) mit Spiegelung/Rotation um denselben Ursprung mathematisch vertauschbar ist, spielt es für das Ergebnis keine Rolle, dass die Skalierung in `dxfTransformPoint()`/`recomputeDxfAnchorCorrection()` als erster Schritt vor Spiegeln/Drehen angewendet wird.

Alle fünf Transformationswerte gehören zur jeweils geladenen DXF-Datei und werden bei jedem (Neu-)Import sowie bei „DXF löschen" auf den Ausgangszustand zurückgesetzt (kein Spiegeln, 0°, Skalierung 1, 0mm, `dxfOffset` 0/0/0).

### 9.8d Rendering, Kamera-Framing, Performance

`drawDxf(proj)` wird in `render()` bewusst **vor** der Segment-Schleife der Fräskontur aufgerufen, sodass die DXF-Unterlage an jeder Überlappungsstelle optisch unter der Kontur liegt (der Canvas arbeitet ohne Tiefenpuffer nach dem Malprinzip). `frameCamera()` bezieht die transformierten DXF-Punkte über `dxfPointsForBounds()` in die Bounding-Box-Berechnung mit ein (zusammen mit den CNC-Bahnpunkten, sofern ein Programm geladen ist) — ein DXF-Import ganz ohne geladenes Programm zentriert/zoomt die Kamera also korrekt allein auf die DXF-Geometrie; `dxfPointsForBounds()` liefert ein leeres Array, wenn `dxfVisible === false` oder ein Layer ausgeblendet ist, damit unsichtbare Geometrie die automatische Rahmung nicht beeinflusst.

Für die Zeichenperformance großer DXF-Dateien (insbesondere reale 3D-Flächenmodelle mit zehntausenden 3DFACE-Entitäten) gilt dieselbe gebündelte Zeichenaufruf-Technik wie für die Fräsbahn — siehe 9.5, `strokeGroupedByColor()`.

### 9.8e Ein-/Ausblenden, Löschen, Persistenz

Werkzeugleisten-Knopf „👁 DXF" (`#btnToggleDxf`) schaltet `dxfVisible` um (deaktiviert ohne geladenes DXF); `drawDxf()` zeichnet bei `dxfVisible === false` nichts. „DXF löschen" (Menüpunkt `#btnDxfDeleteMenu` im „Bearbeiten"-Menü, gleichwertiges Symbolleisten-Icon „✕ DXF") entfernt die importierte DXF-Unterlage vollständig; ein evtl. geladenes CNC-Programm bleibt unberührt.

Wie das CNC-Programm selbst wird eine importierte DXF-Unterlage **nicht** über `localStorage`, „Einstellungen exportieren"/„Einstellungen importieren" oder „Simulation speichern" gespeichert — ein Reload der Seite entfernt ein geladenes DXF vollständig, es muss danach erneut importiert werden (siehe auch Abschnitt 13).

Statusanzeige neben dem Dateinamen (`#dxfStatusGroup`, in der Kopfzeile, siehe Abschnitt 10): nur sichtbar, solange ein DXF geladen ist, zeigt ausschließlich den Dateinamen (Tooltip zusätzlich mit Entitäts-/Punktanzahl sowie ggf. nicht unterstützten/übersprungenen Entitätstypen). Ein Trennstrich (`#dxfFilenameSep`) erscheint zwischen dem Programmnamen und dem DXF-Dateinamen, aber nur, wenn tatsächlich beide gleichzeitig geladen sind.



## 10. Menüband (Topbar)

Von links nach rechts:

1. **„Datei"-Menü** (`#menuFileTrigger`, `.menu-trigger`) — öffnet `#menuFileDropdown`:
   - **„CNC-Programm"** (`#submenuFileCncProgramTrigger`/`#submenuFileCncProgramDropdown`, Untermenü-Flyout) → **Programm laden** (`#btnLoadFile`, nativer Dateiauswahl-Dialog), **Programm speichern als…** (`#btnSaveAsProgram`, deaktiviert ohne geladenes Programm)
   - **DXF Importieren** (`#btnDxfImport`, immer aktiv; importiert eine DXF-Datei als eigenständige, vom CNC-Programm unabhängige Referenz-Unterlage im 3D-Bereich, siehe 9.8)
   - **Simulation speichern** (`#btnSaveSimulation`, immer aktiv; lädt eine komplette, eigenständig lauffähige Kopie dieser Seite herunter, mit den aktuellen Maschinentyp-/Arbeitsgang-Namen-Einstellungen als neuer Standard eingebacken, aber ohne ein eventuell geladenes CNC-Programm, siehe 5.13)

2. **„Bearbeiten"-Menü** (`#menuEditTrigger`) — öffnet `#menuEditDropdown` mit drei Untermenü-Flyouts:
   - **„CNC"** (`#submenuCncTrigger`/`#submenuCncDropdown`) → **CNC-Code einfügen** (`#btnPaste`), **CNC-Programm bearbeiten** (`#btnEditProgram`, deaktiviert ohne geladenes Programm), **CNC-Programm löschen** (`#btnClearProgram`, deaktiviert ohne geladenes Programm), **CNC-Programmliste löschen** (`#btnClearProgramList`, deaktiviert solange die Programmliste leer ist — siehe 3.2)
   - **„DXF"** (`#submenuDxfTrigger`/`#submenuDxfDropdown`) → **DXF Verschieben** (`#btnDxfMoveMenu`, deaktiviert ohne importiertes DXF; zweiter, gleichwertiger Auslöser: Toolbar-Icon „✥ DXF"), **DXF löschen** (`#btnDxfDeleteMenu`, deaktiviert ohne importiertes DXF; zweiter, gleichwertiger Auslöser: Toolbar-Icon „✕ DXF")
   - **„Arbeitsgänge"** (`#submenuOpsTrigger`/`#submenuOpsDropdown`) → **Arbeitsgang-Namen bearbeiten** (`#btnEditOpNames`, immer aktiv)

3. **„Einstellungen"-Menü** (`#menuViewTrigger`) — öffnet `#menuViewDropdown`:
   - **„Erscheinungsbild"** (`#submenuAppearanceTrigger`/`#submenuAppearanceDropdown`) → fasst zusammen:
     - **„Farben"** (`#submenuColorsTrigger`/`#submenuColorsDropdown`, zweite Verschachtelungsebene) → **Buttonfarbe** (`#btnMenuColor`, öffnet den Dialog aus 12.2), **Hintergrundfarbe** (`#btnEditColors`, öffnet das Farbprofil-Panel aus 12.1), **Schriftfarbe** (`#btnFontColor`, öffnet das Panel aus 12.3)
     - **Spalten** (`#btnColumnScale`, kein eigenes Flyout; öffnet den Dialog aus 7.3a)
     - **Standardwerte wiederherstellen** (`#btnResetDisplayDefaults`; setzt Hintergrundfarbe/Buttonfarbe/Schriftfarbe/Spaltenbreiten in einem Schritt zurück, mit Ja/Nein-Sicherheitsabfrage, siehe 12.3)
   - **„Maschinentypen"** (`#submenuMachineTypesTrigger`/`#submenuMachineTypesDropdown`) → **Maschinentypen bearbeiten** (`#btnEditMachineTypes`, siehe 5.7/11; das Panel enthält seit Release 134 zusätzlich den Knopf „Maschinentypen zurücksetzen", siehe 5.7b)
   - **Einstellungen importieren** (`#btnLoadSettings`) / **Einstellungen exportieren** (`#btnSaveSettings`) — flache, direkte Einträge ganz am Ende, in dieser Reihenfolge

4. **„Hilfe"-Menü** (`#menuInfosTrigger`) — öffnet `#menuInfosDropdown`, direkt rechts von „Einstellungen". Enthält genau einen Eintrag: **„Knowledge Base"** (`#btnOpenKnowledgeBase`, öffnet das Panel aus 10.1a). Die interne ID (`menuInfos`/`menuInfosTrigger`/`menuInfosDropdown`) unterscheidet sich von der sichtbaren Beschriftung „Hilfe" — dasselbe gilt für das „?"-Menü (siehe unten), dessen ID intern weiterhin `menuInfo` lautet.

5. **„?"-Menü** (`#menuInfoTrigger`) — öffnet `#menuInfoDropdown`, ganz rechts nach „Hilfe". Enthält zunächst zwei reine Anzeige-Zeilen (`.menu-info-row`, `cursor: default`, kein Hover-Zustand):
   - **Version** — Wert in `#infoVersionValue`: „Release " + `APP_RELEASE_NUMBER` (Konstante ganz oben in `app.js`).
   - **Letzte Releasezeit:** — Wert in `#infoReleaseTimeValue`: `APP_RELEASE_TIMESTAMP` (Konstante direkt daneben, deutsche Zeit).

   Beide Konstanten werden von Hand bei jeder ausgelieferten Version aktualisiert. `APP_RELEASE_NUMBER` speist zusätzlich den „rNNN"-Zusatz im Markennamen ganz rechts (siehe Punkt 10 unten) sowie den Vergleichswert für die Update-Prüfung (siehe 10.3).

   Durch eine Trennlinie (`margin-top`/`padding-top`/`border-top`) von den beiden Anzeige-Zeilen abgesetzt, folgen drei anklickbare Einträge:
   - **„Versionsupdate prüfen"** (`#btnCheckForUpdate`) — prüft auf eine neuere Version und bietet ggf. ein Update an, siehe 10.3. Als einziger Eintrag dieses Menüs zusätzlich per Textfarbe/-gewicht hervorgehoben (`color: var(--accent)` + `font-weight: 600`, statt der sonst überall im Menü einheitlichen, neutralen `.menu-item`-Schriftfarbe), damit er als primärer Aktions-Knopf sofort ins Auge fällt. Der `:disabled`-Zustand während einer laufenden Prüfung (Knopftext „Prüfe…", siehe 10.3) bleibt über die bestehende `.menu-item:disabled`-Regel (`opacity: .4`) weiterhin zuverlässig abgedimmt, unabhängig von der zusätzlichen Akzentfarbe.
   - **„Datenschutzerklärung"** (`#btnShowPrivacy`) — öffnet die Datenschutzerklärung, siehe 10.4. Trägt dieselbe Trennlinien-Optik (`margin-top`/`padding-top`/`border-top`) wie „Versionsupdate prüfen" direkt darüber, aber **ohne** dessen Akzent-Hervorhebung — optisch ein gewöhnlicher `.menu-item`-Eintrag.
   - **„Impressum"** (`#btnShowImprint`) — öffnet das Impressum, siehe 10.4. Schließt sich direkt und ohne eigene Trennlinie an „Datenschutzerklärung" an.

   **Beschriftung/Farbe des Auslösers:** Sichtbarer Text ist „?", `aria-label="Info"`/`title="Info"` bleiben zur Barrierefreiheit erhalten. Die Schriftgröße ist die gemeinsame `.menu-trigger`-Größe (13px), identisch zu allen anderen Menü-Auslösern. Die Farbe ist dagegen fest auf `#ff8c00` (Orange) gesetzt — per ID-Selektor `#menuInfoTrigger` (höhere Spezifität als `.menu-trigger`) zusätzlich mit `!important` abgesichert, damit weder Hover noch der geöffnete Zustand (`.menu.open .menu-trigger`, das sonst über `--menu-active-ink` die vom Benutzer gewählte Buttonfarbe einfärben würde) noch irgendein Farbprofil (Standard/Dark Mode/Benutzerfarben) diese Farbe verändern kann.

6. **„Maschinentyp"-Label** (fett, Akzentfarbe, Großbuchstaben, siehe 5.7) **+ Maschinentyp-Dropdown** (`<select>`, dynamisch befüllt, „ISO" immer zuerst) **+ Ok** (deaktiviert ohne geladenes Programm; wendet den gewählten Maschinentyp tatsächlich an, siehe 5.1) — zusammen `.machine-type-quick-group`, direkt neben dem „?"-Menü.

7. *(vertikaler Trennstrich, `.topbar-sep`)*

8. **Dateiname-Anzeige** (`.file-status-row`, immer sichtbar — reine Status-Anzeige, kein Bestandteil eines Menüs), direkt gefolgt vom **„Kardan-Winkelkorrektur"-Schalter** (`#reichenbacherSwitchWrap`, siehe 9.7) — nur sichtbar, solange der bestätigte Maschinentyp „Reichenbacher" ist — sowie, unabhängig davon, der **DXF-Statusgruppe** (`#dxfStatusGroup`, siehe 9.8) — nur sichtbar, solange ein DXF importiert ist, und zeigt ausschließlich den reinen Dateinamen, ohne jeden Knopf (die DXF-bezogenen Aktionen sitzen vollständig im „Bearbeiten"-Menü bzw. in der Werkzeugleiste, siehe unten). Unmittelbar vor der DXF-Statusgruppe steht zusätzlich ein eigener Trennstrich (`#dxfFilenameSep`), aber nur, wenn tatsächlich sowohl ein CNC-Programm als auch ein DXF gleichzeitig geladen sind.

9. *(Flex-Abstandshalter)*

10. **Markenname** — „CNC SimX" (`.brand-mark`), mit dem `r`+`APP_RELEASE_NUMBER`-Zusatz (`#brandReleaseTag`, z. B. „CNC SimX r107"), ganz rechts, mit einem kleinen Marken-Icon davor (`<img class="brand-icon">`, dasselbe 32px-Hexagon-Symbol wie das Favicon). `<title>` des Browser-Tabs sowie ggf. Fenstertitel/MessageBox-Titel einer WebView2-exe sind ebenfalls auf „CNC SimX" eingestellt.

11. **Favicon/Tab-Icon** — ein `<link rel="icon">` mit einem eingebetteten 32×32px-PNG (Base64-`data:`-URI direkt in `template_top.html`, keine externe Bilddatei nötig, passend zur einzigen, in sich geschlossenen HTML-Datei) erscheint sowohl als Browser-Tab-Icon als auch direkt neben dem Markennamen.

**Menü-Verhalten** (`.menu`/`.menu-trigger`/`.menu-dropdown` in `template_top.html`, Verdrahtung in `app.js`): Klick auf einen Menü-Trigger öffnet dessen Dropdown und schließt dabei automatisch das jeweils andere (höchstens ein Hauptmenü gleichzeitig offen, `closeAllMenus()`/`openMenu()`); ein erneuter Klick auf denselben, bereits offenen Trigger schließt ihn wieder (Umschalter). Ein Klick auf einen beliebigen Menüeintrag löst zusätzlich zu seiner eigenen Aktion das Schließen des Menüs aus. Ein Klick außerhalb eines offenen Menüs (`document`-weiter Klick-Listener, geprüft per `!e.target.closest('.menu')`) sowie die Escape-Taste (`window`-weiter `keydown`-Listener) schließen ebenfalls. `aria-expanded` am jeweiligen Trigger wird passend mitgeführt. Deaktivierte Einträge (`:disabled`, dieselbe CSS-Regel wie bei den 3D-Werkzeugleisten-Knöpfen, siehe 9.7) bleiben klar erkennbar (blasser, „verboten"-Cursor) statt wirkungslos anklickbar zu wirken.

`.menu-trigger` trägt einen festen, dauerhaft sichtbaren Rahmen (`border: 1px solid var(--border)`), beim Hover dunkler (`border-color: var(--ink-faint)`) — dasselbe Muster wie der „Ok"-Knopf neben „Maschinentyp" (`#btnMachineTypeOk`, `.btn.ghost`). Der geöffnete Zustand (`.menu.open .menu-trigger`) überschreibt `border-color` weiterhin auf `transparent`. Der vertikale Innenabstand des Menübands (`.topbar { padding-top/padding-bottom: 6px; padding-left/padding-right: 16px; }`) ist knapp bemessen; alle Elemente innerhalb des Menübands (Menü-Auslöser, Maschinentyp-Gruppe, Dateiname, Marke usw.) behalten dabei ihre eigene, unveränderte Höhe.

**Untermenü-Flyout-Mechanik:** Ein `.menu-submenu` (z. B. `#submenuCnc`, `#submenuColors`) ist ein normaler `.menu-item`-Auslöser (zusätzliche Klasse `.submenu-trigger`, mit angehängtem „▸"-Pfeilglyph) mit einem direkt daran hängenden, seitlich (`left: 100%`) ausklappenden zweiten Dropdown (`.submenu-dropdown`) — optisch dieselbe `.menu-dropdown`-Bauweise (Karte, Schatten, `.menu-item`-Einträge), nur seitlich statt unterhalb positioniert. Ein Klick auf einen Flyout-Auslöser öffnet/schließt nur sein eigenes Flyout (`openSubmenu()`/`closeSubmenu()`) und schließt dabei automatisch jedes Geschwister-Flyout im selben übergeordneten Dropdown — das übergeordnete Hauptmenü selbst bleibt dabei geöffnet, da `e.stopPropagation()` verhindert, dass der Klick beim „Klick auf Menüeintrag schließt alles"-Listener des Hauptmenüs ankommt. Ein Klick auf einen tatsächlichen Aktions-Eintrag innerhalb eines Flyouts (z. B. „CNC-Code einfügen") schließt dagegen weiterhin alles — Hauptmenü und alle offenen Flyouts. Escape sowie ein Klick außerhalb schließen ebenfalls alles.

Die gesamte Flyout-Mechanik ist rein rekursiv aufgebaut: `allSubmenus` sammelt alle `.menu-submenu`-Elemente im Dokument per `document.querySelectorAll('.menu-submenu')`, unabhängig von ihrer Verschachtelungstiefe, und jedes bekommt dieselbe, unabhängige `position: relative`-Verankerung für sein direkt angehängtes `.submenu-dropdown` (`position: absolute; left: 100%`). Die Geschwister-Schließlogik in `openSubmenu()` vergleicht dabei ausschließlich `wrapper.parentElement` (das jeweils unmittelbare Elternelement), nie die absolute Tiefe — dadurch funktioniert „ein Klick auf einen anderen Flyout-Auslöser im selben übergeordneten Dropdown schließt die Geschwister" automatisch korrekt auf jeder Ebene, auch bei der zwei Ebenen tiefen Verschachtelung „Erscheinungsbild" → „Farben".

Ein früherer Hinweistext direkt unterhalb des Menübands wurde entfernt; die einstige Maschinen-/Struktur-Badge-Anzeige (`#machineBadge`) ist per `hidden`-Attribut dauerhaft ausgeblendet — sichtbar bleibt ausschließlich der Dateiname (`#filenameLabel`).

### 10.1 Symbolleisten (drei Zeilen)

`#viewerToolbar` ist der äußere, senkrecht stapelnde Rahmen (`display:flex; flex-direction:column`) um drei `.toolbar-row`-Zeilen (`#toolbarRow1`/`#toolbarRow2`/`#toolbarRow3`), jede mit `display:flex; align-items:center; gap:8px; flex-wrap:wrap`. Beide Leisten (`#viewerToolbar`, `#transport`) sitzen als eigene Zeilen zwischen dem Menüband (`.topbar`) und dem Hauptbereich (`.main`, die Zwei-Spalten-Aufteilung Programmcode/3D) und spannen über die volle Fensterbreite.

- **Zeile 1** (`#toolbarRow1`): „📂 CNC" (`#btnLoadFileIcon`, klassische Einzeldatei-Auswahl, siehe 3.), „✎ CNC" (`#btnEditProgramToolbarIcon`, öffnet `openProgramEditPanel()` — dritter, gleichwertiger Auslöser neben dem Menüpunkt `#btnEditProgram` und dem Icon-Knopf neben der Überschrift „CNC-Programmcode", siehe 7.1), „✕ CNC" (`#btnClearProgramIcon`, ruft `clearProgram()` auf, dieselbe Funktion wie der Menüpunkt „CNC-Programm löschen"; nur das „✕"-Zeichen ist rot eingefärbt, `.dxf-delete-glyph`/`--danger`), „☰+ Liste" (`#btnNewProgramListIcon`, öffnet den Mehrfachauswahl-Dialog `#fileInputList` und ersetzt die Programmliste über `startNewProgramList()` — siehe 3.2; nie deaktiviert, auch ohne geladenes Programm nutzbar), „☰✕ Liste" (`#btnClearProgramListIcon`, ruft `clearProgramList()` auf, dieselbe Funktion wie der Menüpunkt „CNC-Programmliste löschen" — siehe 3.2; deaktiviert, solange die Programmliste leer ist) — die übrigen drei (Laden/Bearbeiten/Löschen des aktiven Programms) deaktiviert ohne geladenes Programm bzw. Löschen der Liste ohne Einträge in der Programmliste (außer dem Lade-Icon und „☰+ Liste" selbst). Alle CNC-Icons dieser Zeile bilden eine einzige, durchgehende Knopfgruppe ohne Trennstriche dazwischen — genau wie die DXF-Gruppe in Zeile 2.

- **Zeile 2** (`#toolbarRow2`): die komplette, zusammenhängende Gruppe der DXF-Symbolleisten-Icons: „📂 DXF" (`#btnDxfImportIcon`, Laden — ruft dieselbe Datei-Auswahl wie der Menüpunkt „DXF Importieren" auf, `loadDxfText()`; bewusst ohne eigenes `disabled`-Verhalten, ein erneuter Import ersetzt das vorherige DXF), „👁 DXF" (`#btnToggleDxf`, Sichtbarkeit), „✥ DXF" (`#btnDxfMoveIcon`, Verschieben), „⬍ DXF"/„⬌ DXF" (Spiegeln an X-/Y-Achse), „↻ DXF" (Drehen), „🧱 DXF" (Bauteilstärke), „🔍 DXF" (`#btnDxfScale`, Skalieren), „✕ DXF" (`#btnDxfDeleteIcon`, Löschen, ruft `clearDxf()` auf; nur das „✕"-Zeichen ist rot eingefärbt) — siehe 9.8 für die vollständige Funktionsbeschreibung jedes einzelnen Knopfs. Alle DXF-spezifischen Knöpfe außer dem Lade-Icon sind ohne importiertes DXF deaktiviert.

- **Zeile 3** (`#toolbarRow3`): „+Werkzeuge (T)" (`#btnInsertToolData`), „👁 Werkzeug" (`#btnToggleTool`, Werkzeug-Zylinder ein-/ausblenden), „👁 Spannfutter" (`#btnToggleToolHolder`, Werkzeughalter ein-/ausblenden, siehe 9.7.4), Trennstrich, „⛶ Gesamtes Programm" (`#btnWholeProgram`), Trennstrich, „📏 Messen" (`#btnMeasureTransport`) + „∠ Winkel" (`#btnMeasureAngleTransport`) — das Mess-Knopfpaar sitzt hier statt (nur) in der Transportleiste, damit die Messfunktion auch beim reinen DXF-Betrachten ohne geladenes CNC-Programm nutzbar ist (die Symbolleiste ist stets sichtbar, die Transportleiste nur bei geladenem Programm, siehe unten).

Die „Perspektive:"-Beschriftung samt Ansichts-Pfeil-Icons existiert nicht mehr als eigene Symbolleisten-Gruppe — diese Funktion wurde vollständig durch das Rhombenkuboktaeder-Ansichts-Gizmo direkt im 3D-Bereich abgedeckt (siehe 9.2a). Der „⌂ Home"-Knopf (`#btnHomeView`) sitzt als freies Overlay unmittelbar links neben dem Gizmo im 3D-Bereich selbst (`position: absolute; top: 18px; right: 112px`).

Direkt darunter folgt die volle-Fensterbreite-Transportleiste (`#transport`, Play/Stopp/Scrubber/„Eilgang anzeigen"-Kontrollkästchen/Geschwindigkeit/Sprung-in-Zeile). Die DOM-Reihenfolge direkt unter `#app` lautet `.topbar` → `#viewerToolbar` → `#transport` → `.main` → `.footer-credit`.

**Einheitliche Knopfgröße je Gruppe:** Da jeder Knopf ein anderes Unicode-Symbol vor demselben „CNC"-/„DXF"-Textsuffix trägt (z. B. „📂" gegen „✎" gegen „✕"), rendern die Symbole selbst bei gleicher Schriftgröße unterschiedlich breit. Um dennoch eine gleichmäßige Optik zu erreichen, tragen alle zwölf CNC- plus DXF-Symbolleisten-Icons (Zeile 1 + Zeile 2) eine gemeinsame, feste Breite von 68×26px, mit Flexbox-Zentrierung statt variabler Innenabstände. Die drei Werkzeuge-Icons in Zeile 3 (`#btnInsertToolData`/`#btnToggleTool`/`#btnToggleToolHolder`, mit der deutlich längeren Beschriftung „+Werkzeuge (T)") bilden eine eigene, ebenfalls einheitliche Gruppe mit 118×26px. Die Höhe ist über beide Gruppen hinweg einheitlich 26px, ebenso bei „⛶ Gesamtes Programm" (`#btnWholeProgram`, eigene `height: 26px`-Regel, Breite bewusst variabel — dieser Knopf ist nie Teil einer gleichbreiten Gruppe) und beim Mess-Knopfpaar (`#btnMeasureTransport`/`#btnMeasureAngleTransport`, siehe 9.4).

### 10.1a Knowledge Base

„Hilfe" → „Knowledge Base" (`#btnOpenKnowledgeBase`) öffnet `#knowledgeBasePanel`, technisch dasselbe `.paste-panel`-Grundmuster wie „Maschinentypen bearbeiten"/„Arbeitsgang-Namen bearbeiten" (siehe 11): eine große Textarea (`#knowledgeBaseEditArea`) mit dem kompletten rohen Text, „Abbrechen"/„Übernehmen" unten. Eine editierbare Liste von Infos, in der Größe skalierbar (siehe unten), die Punkte sortieren sich beim Übernehmen automatisch alphabetisch, und oben im Fenster gibt es eine A-Z-Sprungleiste sowie einen Knopf, um ganz nach oben zu scrollen (siehe unten).

**Eintragsformat:** Ein Eintrag ist ein mit `[` beginnender und `]` endender Block, dessen erste Zeile der Titel ist (z. B. `Fräserlängen:`), gefolgt von beliebig vielen weiteren Zeilen. Mehrere Einträge stehen durch eine Leerzeile getrennt im selben Textfeld. Geparst wird per nicht-gierigem Regex ohne verschachtelte eckige Klammern (`parseKnowledgeBaseEntries()`/`knowledgeBaseEntryOffsets()`, `app.js`) — leere/nur aus Leerraum bestehende Blöcke werden übersprungen.

**Standard-Einträge** (`DEFAULT_KNOWLEDGE_BASE_TEXT`), aktuell elf Einträge:

```
[DXF:
3D DXF können auch importiert werden.]

[Fräserlängen:
1. Zu tief gefräst = Fräser zu kurz -> Fräserlänge einmessen und länger auf Maschine eingeben
2. Fräsung nicht tief genug = Fräser zu lang -> Fräserlänge einmessen und kürzer auf Maschine eingeben]

[Fräserradien:
1. Zu viel weggefräst = Fräserradius zu klein -> Fräserradius messen und vergrößern
2. Zu wenig weggefräst = Fräserradius zu groß -> Fräserradius messen und verkleinern]

[Messen:
Messen kann in der DXF erfolgen, im CNC-Programm und zwischen beidem! Shift + W für Winkel, Shift + L für Länge, mit Esc abbrechen]

[Reichenbacher:
Aggregatdrehung zum "normalen" Winkel erfolgt nach dem Laden des Programms und "Ok".]

[Shortcuts:
Shift + L = Längenmessung, Shift + W = Winkelmessung, mit Esc zu verlassen. Space startet/pausiert Simulation.]

[Simulation:
Space startet/pausiert Simulation.]

[Werkzeug:
Übergabe über:
; RADIUS 
; LENGTH
bzw. 
; BLADE
BLADE (für die Blattstärke) überschreibt LENGTH
BLADE ist ein fixer Eintrag aus Select/UP]

[Werkzeug2:
Mit dem Auge aus-/einblenden.]

[Werkzeug3:
Mit "Spannfutter" kann Spannfutter ausgeblendet werden.]

[Werkzeug4:
Beim Einfügen über "+Werkzeug" kann ausgewählt werden wie Werkzeug eingemessen worden ist, also die aus- oder eingespannte Länge.]
```

**Hinweis zur Persistenz:** Diese elf Einträge sind nur der Rückfallwert (`DEFAULT_KNOWLEDGE_BASE_TEXT`), der beim allerersten Öffnen (kein vorheriger `localStorage`-Eintrag vorhanden) angezeigt wird. Wurde die Knowledge Base in einem Browser bereits vorher einmal über „Übernehmen" gespeichert, zeigt dieser Browser weiterhin den zuletzt gespeicherten Stand (`cncsim.knowledgeBaseText`) — geänderte Standardeinträge erscheinen dort erst nach einem manuellen Nachtragen oder nach Löschen des gespeicherten Standes.

**Warnung bei fehlerhaftem Eintrag statt stillem Verwerfen:** Da `parseKnowledgeBaseEntries()` (siehe „Eintragsformat" oben) ausschließlich das liest, was zwischen `[` und `]` liegt, würde ein Block mit fehlender/verrutschter Klammer beim Klick auf „Übernehmen" sonst kommentarlos verschwinden. `findMalformedKnowledgeBaseBlocks(text)` (`app.js`) prüft deshalb zusätzlich separat: Der Text wird an Leerzeilen (derselben Konvention, in der Einträge ohnehin getrennt werden) in Blöcke zerlegt, und jeder nach dem Trimmen nicht-leere Block muss sowohl mit `[` beginnen als auch mit `]` enden. `#btnKnowledgeBaseApply` ruft diese Prüfung vor jedem „Übernehmen" auf:

- Findet sie mindestens einen fehlerhaften Block, erscheint eine Warnung (`#knowledgeBaseWarning`, Warnfarbe) mit der jeweils ersten Zeile jedes betroffenen Blocks als Kennung (z. B. „Achtung: folgender Eintrag beginnt nicht mit „[" bzw. endet nicht mit „]" und würde sonst verworfen: Ohne öffnende Klammer:") — **und „Übernehmen" wird abgebrochen**, weder Sortieren noch Speichern noch Schließen des Panels finden statt. Anders als die (nicht blockierende) Duplikat-Namen-Warnung bei „Maschinentypen bearbeiten" (siehe 5.7) ist das bewusst eine echte Blockade: Ein still verworfener Eintrag wäre Datenverlust, kein bloßer Hinweis.
- Der Benutzer kann den fehlerhaften Block direkt im weiterhin geöffneten Textfeld korrigieren und erneut „Übernehmen" klicken.
- Die Warnung wird beim erneuten Öffnen des Panels sowie nach einem erfolgreichen „Übernehmen" automatisch wieder ausgeblendet.
- Wohlgeformte Einträge (auch mehrere gleichzeitig fehlerhafte Blöcke) bleiben von dieser Prüfung unberührt — sie werden wie gewohnt geparst/sortiert/gespeichert, sobald kein fehlerhafter Block mehr im Text steht.

**Automatisches alphabetisches Sortieren:** Beim Klick auf „Übernehmen" (`applyKnowledgeBaseText()` → `sortKnowledgeBaseEntries()`) werden alle Einträge neu geparst und nach ihrer jeweils ersten Zeile (Titel) sortiert — `localeCompare('de', { sensitivity: 'base', numeric: true })`, dieselbe Vergleichsart wie beim numerisch-alphabetischen DXF-Layer-Sortieren (siehe 9.8a), damit Umlaute (ä/ö/ü) korrekt einsortiert werden und Groß-/Kleinschreibung keine Rolle spielt. Der neu zusammengesetzte, sortierte Text wird anschließend persistiert und ersetzt den Inhalt der Textarea beim nächsten Öffnen.

**Größenveränderbar (seitlich wie nach unten):** Die Karte des Panels trägt zusätzlich zur Klasse `.paste-card` die Modifikator-Klasse `.paste-card.resizable` (`resize: both; overflow: auto`, mit `min-width`/`min-height`) — als einziges Bearbeiten-Panel der Anwendung kann der Benutzer sie am unteren rechten Eck per Maus sowohl breiter/schmaler als auch höher/niedriger ziehen; die Textarea selbst wächst dabei über `flex: 1 1 auto` automatisch mit.

**A-Z-Sprungleiste (`#knowledgeBaseAzBar`):** 26 Buchstaben-Knöpfe (A–Z), beim Laden einmalig per JS erzeugt. Ein Klick auf einen Buchstaben sucht im aktuellen Textarea-Inhalt (auch vor einem „Übernehmen") den ersten Eintrag, dessen Titel ab diesem Buchstaben beginnt (oder alphabetisch danach folgt, falls der Buchstabe selbst nicht vorkommt — dann den letzten Eintrag), setzt den Cursor an dessen Anfang und scrollt die Textarea dorthin (`jumpToKnowledgeBaseLetter()`: Zeilennummer aus der Anzahl vorangehender Zeilenumbrüche, multipliziert mit der aktuellen `line-height` der Textarea).

**„Nach oben"-Knopf (`#btnKnowledgeBaseScrollTop`):** setzt Cursor und Scroll-Position der Textarea auf den Anfang zurück.

**Persistenz:** Wie die Arbeitsgang-Namensliste (siehe 6) per `localStorage` (`cncsim.knowledgeBaseText`) über Seiten-Neuladen hinweg gespeichert, mit `DEFAULT_KNOWLEDGE_BASE_TEXT` als Rückfallwert. Fließt außerdem in „Einstellungen exportieren"/„Einstellungen importieren" (siehe 5.9) sowie in „Simulation speichern" (siehe 5.13) mit ein — exakt dieselbe Behandlung wie die Arbeitsgang-Namensliste und die Maschinentyp-Konfiguration.

### 10.2 Rechtsklick-Kontextmenü

**Markup & Optik (`#appContextMenu`, `template_top.html`):** Ein einzelnes, fest positioniertes (`position: fixed`) Panel ganz am Ende des Dokuments, technisch dieselbe `.menu-item`-Bauweise wie die vier Hauptmenüs (siehe 10, Hover-/Fokus-/`disabled`-Optik identisch), aber mit eigenem `z-index: 500` (deutlich über `.menu-dropdown`/`.submenu-dropdown`, damit es auch über bereits offenen Dialogen/Panels liegt). Dreizehn Einträge, per `.context-menu-divider` (dünne Trennlinie) in vier Gruppen geteilt:

1. Längenmessung (`#ctxMeasureLength`) / Winkelmessung (`#ctxMeasureAngle`)
2. Codesprung (`#ctxCodeJump`) / Gesamtes Programm (`#ctxWholeProgram`)
3. Programm laden (`#ctxLoadProgram`) / Programm löschen (`#ctxClearProgram`) / Programmliste erstellen (`#ctxNewProgramList`) / Programmliste löschen (`#ctxClearProgramList`) / Alle Arbeitsgänge ausblenden (`#ctxToggleAllOps`)
4. DXF laden (`#ctxLoadDxf`) / DXF löschen (`#ctxClearDxf`) / DXF-Layer ausblenden (`#ctxToggleDxfLayers`) / DXF verschieben (`#ctxMoveDxf`)

Jeder Eintrag ruft in `app.js` dieselbe Funktion bzw. denselben Datei-Dialog-Auslöser wie sein bereits bestehendes Gegenstück auf: `toggleMeasure()`/`toggleMeasureAngle()` (siehe 9.4), `toggleCodeJump()` (siehe 9.4a, identisch zu einem Klick auf `#btnCodeJump`) und `() => setScope({ type: 'all' })` (identisch zu `#btnWholeProgram`, siehe 9.3/10.1), `#fileInput`/`#dxfFileInput`-Klick (wie „Programm laden"/„DXF Importieren" im Menüband), `#fileInputList`-Klick (wie „Neue CNC-Programmliste" im „Datei"-Menü bzw. „☰+ Liste" in der Symbolleiste, öffnet den Mehrfachauswahl-Dialog und ruft nach Auswahl `startNewProgramList()` auf, siehe 3.2), `clearProgram()`/`clearDxf()` (wie „CNC-Programm löschen"/„DXF löschen"), `clearProgramList()` (wie „CNC-Programmliste löschen"/„☰✕ Liste", siehe 3.2/10.1), `toggleAllOps()`/`toggleAllDxfLayers()` (wie die gleichnamigen Knöpfe im Arbeitsgänge-/DXF-Layer-Fenster, siehe 7.2/9.8a) und `openDxfMovePanel()` (wie „DXF Verschieben").

„Codesprung"/„Gesamtes Programm" sind — wie „Längenmessung"/„Winkelmessung"/„Programm laden"/„Programmliste erstellen"/„DXF laden" — ohne Vorbedingung immer aktiv (kein dynamisches `disabled`/Label-Umschalten in `updateContextMenuState()`, siehe unten): Diese Funktionen sind jederzeit sinnvoll auslösbar, unabhängig vom aktuellen Programm-/DXF-Zustand, genau wie ihre Symbolleisten-Gegenstücke `#btnCodeJump`/`#btnWholeProgram`/`#btnNewProgramListIcon` nie deaktiviert sind. „Programmliste löschen" ist dagegen — wie „Programm löschen"/„DXF löschen" — an einen dynamischen Zustand geknüpft (siehe unten).

**Öffnen/Position (`openContextMenu(x, y)`):** Blendet das Panel zunächst an der Klickposition ein, liest danach `offsetWidth`/`offsetHeight` aus und rückt es bei Bedarf so weit nach links/oben, dass es nie über den sichtbaren Fensterrand hinausragt (4px Mindestabstand zum Rand). Ruft vor dem Einblenden `closeAllMenus()` (schließt ein eventuell offenes Hauptmenü/Flyout) sowie `updateContextMenuState()` auf.

**Zustand bei jedem Öffnen neu ermittelt (`updateContextMenuState()`), nicht laufend mitgeführt** — spiegelt exakt denselben Zustand wie die jeweils bestehenden Knöpfe/Menüpunkte an ihrer gewohnten Stelle:
- „Programm löschen" deaktiviert ⟺ `#btnClearProgram` deaktiviert (kein Programm geladen).
- „Programmliste löschen" deaktiviert ⟺ `#btnClearProgramListIcon` deaktiviert (Programmliste leer, `programList.length === 0`).
- „Alle Arbeitsgänge ausblenden"/„…anzeigen" deaktiviert, solange keine Arbeitsgänge existieren; Label wechselt dynamisch auf „…anzeigen", sobald `hiddenOpIndexes` nicht leer ist — exakt dieselbe Abfrage wie `updateShowAllOpsButton()` (siehe 7.2), nur mit dem vollen Wort „Arbeitsgänge" statt der Abkürzung „AG" des kompakten Knopfs.
- „DXF löschen"/„DXF verschieben" deaktiviert ⟺ `#btnDxfDeleteMenu`/`#btnDxfMoveMenu` deaktiviert (kein DXF importiert).
- „DXF-Layer ausblenden"/„…anzeigen" deaktiviert, solange die aktuelle DXF-Datei keine Layer hat; Label wechselt dynamisch nach derselben `dxfHiddenLayers.size`-Logik wie `updateShowAllDxfLayersButton()` (siehe 9.8a).
- „Längenmessung"/„Winkelmessung"/„Codesprung"/„Gesamtes Programm"/„Programm laden"/„Programmliste erstellen"/„DXF laden" sind ohne Vorbedingung immer aktiv.

Ein deaktivierter Eintrag (`:disabled`, `pointer-events: none`) löst schon aus sich heraus kein `click`-Ereignis aus — die Klick-Handler selbst brauchen deshalb keinen zusätzlichen Enable-Guard.

**Auslöser — überall in der App:**

- Ein globaler `document`-weiter `'contextmenu'`-Listener öffnet das Menü an der Klickposition, außer das Ziel ist ein echtes Texteingabefeld (`INPUT`/`TEXTAREA`/`SELECT`/`contenteditable`) — dort wird bewusst **kein** `preventDefault()` aufgerufen, damit das native Browser-Kontextmenü (Markieren/Kopieren/Einfügen) dort erhalten bleibt.
- Der 3D-Bereich (`#canvas3d`) hat einen eigenen `'contextmenu'`-Handler mit `e.stopPropagation()` (der globale Listener bekommt Rechtsklicks auf dem Canvas dadurch nie zu Gesicht) UND einer eigenen Klick-vs-Zieh-Unterscheidung, damit das bestehende Rechtsklick-Ziehen zum Kamera-Schwenken (`panCamera()`, siehe 9.2) weiterhin funktioniert, ohne dass danach ungewollt das Menü aufspringt.

**Klick-vs-Zieh-Erkennung im 3D-Bereich — zweistufig, wegen Chromiums Ereignisreihenfolge:** Chromium feuert bei der rechten Maustaste `'contextmenu'` bereits **unmittelbar nach `'pointerdown'`** — also noch bevor `'pointerup'` überhaupt stattfindet — und zwar immer mit den Koordinaten der Drück-Position, unabhängig von einer eventuell folgenden Zieh-Bewegung. Im `'contextmenu'`-Handler selbst lässt sich eine Zieh-Bewegung deshalb noch gar nicht erkennen. Die eigentliche Entscheidung fällt daher zweistufig: `'contextmenu'` merkt nur die Position (`pendingContextMenuAt = {x, y}`) und ruft `preventDefault()`/`stopPropagation()` auf; erst im nachfolgenden `'pointerup'`-Handler, wenn die tatsächliche Loslass-Position feststeht, wird geprüft, ob sich die Maus seit `pointerdown` um höchstens 6px bewegt hat (dasselbe Kriterium wie beim Gizmo-/Messwerkzeug-Klick, siehe 9.2a/9.4) — nur dann öffnet `openContextMenu()` tatsächlich das Menü; bei größerer Bewegung (= Kamera-Schwenken per `panCamera()`) wird die gemerkte Position kommentarlos verworfen. `pendingContextMenuAt` wird zusätzlich bei `'pointercancel'`/`'pointerleave'` zurückgesetzt, damit kein verwaister Zustand einen späteren, unzusammenhängenden Rechtsklick fälschlich beeinflusst.

**Schließen:** Klick auf einen Menüeintrag selbst (schließt zusätzlich zu seiner eigenen Aktion), Klick irgendwo außerhalb des Menüs (`document`-weiter Klick-Listener, `!e.target.closest('#appContextMenu')`) sowie die Escape-Taste (dieselbe bestehende globale Escape-Behandlung wie bei den Hauptmenüs, siehe 10, um `closeContextMenu()` ergänzt).

### 10.3 Versionsupdate prüfen

„Versionsupdate prüfen" vergleicht die eigene Version gegen ein bei GitHub liegendes JSON-Manifest und bietet bei einer neueren Version ein automatisches Update an — innerhalb der WebView2-Host-Anwendung vollautomatisch, im gewöhnlichen Browser-Tab als Download zum manuellen Ersetzen.

**Auslöser (`#btnCheckForUpdate`, `app.js`):** Einzelner Knopf im „?"-Menü (siehe 10, Punkt 5), unterhalb der beiden Anzeige-Zeilen per Trennlinie optisch abgesetzt und per Textfarbe/-gewicht hervorgehoben, technisch derselbe `.menu-item`-Knopf wie jeder andere Menüeintrag.

**Ablauf (`checkForUpdate()`):**

1. Der Knopf wird während der Prüfung deaktiviert und sein Text auf „Prüfe…" umgestellt (verhindert Doppelklicks während eines laufenden Netzwerk-Aufrufs), am Ende (Erfolg **und** Fehler, per `.finally()`) wird beides zurückgesetzt.
2. `fetch(UPDATE_MANIFEST_URL, { cache: 'no-store' })` lädt ein kleines JSON-Manifest von einem öffentlichen GitHub-Repository — die Konstante `UPDATE_MANIFEST_URL` (direkt neben `APP_RELEASE_NUMBER`/`APP_RELEASE_TIMESTAMP` ganz oben in `app.js`) zeigt fest auf `https://raw.githubusercontent.com/Sebastian-CNCSimX/CNC_SimX/main/latest.json`. `cache: 'no-store'` stellt sicher, dass wirklich jedes Mal der aktuelle Stand geprüft wird, nicht eine zwischengespeicherte alte Antwort.
3. Das Manifest hat die Form `{"release": <Zahl>, "timestamp": "<deutsche Zeit>", "htmlUrl": "<Adresse der aktuellen CNC_Simulation.html>"}`. `release` wird direkt mit `APP_RELEASE_NUMBER` verglichen (`Number.isFinite`-Prüfung gegen einen kaputten/fehlenden Wert im Manifest).
4. Ist `release` nicht größer als die eigene `APP_RELEASE_NUMBER`, erscheint lediglich ein `alert()` „Sie verwenden bereits die aktuellste Version (Release N)" — kein weiterer Netzwerkzugriff.
5. Ist eine neuere Version verfügbar, fragt ein natives `confirm()` „Eine neuere Version ist verfügbar: Release N (Zeitstempel). Jetzt aktualisieren?" — bei „Nein" endet der Vorgang hier, ohne dass die eigentliche HTML-Datei überhaupt abgerufen wird.
6. Bei „Ja" wird `manifest.htmlUrl` per `fetch()` abgerufen (derselbe „raw.githubusercontent.com"-Host, keine zusätzliche Adresse einzurichten) und der komplette HTML-Text an `installUpdate()` übergeben.

**Installation (`installUpdate(htmlText, latestRelease)`) — zwei Wege, automatisch anhand der Laufzeitumgebung gewählt:**

- **`isRunningInHostWebView()`** prüft `window.chrome && window.chrome.webview && typeof window.chrome.webview.postMessage === 'function'` — genau dieses Objekt existiert ausschließlich innerhalb einer WebView2-Hülle (siehe „Bauanleitung_EXE_C-Sharp_WebView2.md"), niemals in einem gewöhnlichen Chrome-/Edge-Browser-Tab. Dieselbe Erkennung wie beim bereits bestehenden `window.__hostLoadProgramFromHost`-Hook (siehe 3.1), nur in umgekehrter Richtung.
- **Innerhalb der exe (WebView2 erkannt):** `window.chrome.webview.postMessage({ type: 'cncsim-install-update', html: htmlText })` — die neue HTML wird direkt an den C#-Host übergeben, der sie an derselben Stelle abspeichert, an der die exe die Datei ohnehin navigiert, und die Seite danach neu lädt (vollautomatisch, kein Benutzereingriff mehr nötig). Der dazugehörige `WebMessageReceived`-Handler auf der C#-Seite ist in „Bauanleitung_EXE_C-Sharp_WebView2.md", Abschnitt „Schritt 3a", dokumentiert. Ein kurzer `alert()` informiert währenddessen, dass die Installation läuft.
- **Im gewöhnlichen Browser-Tab (kein WebView2):** Fallback auf dasselbe Blob+`<a download>`-Muster wie bei „Simulation speichern" (siehe 5.13) — die heruntergeladene Datei heißt bewusst exakt `CNC_Simulation.html`, damit ein Speichern am selben Ort wie bisher die alte Datei ersetzt. Ein abschließender `alert()` bittet darum, die Datei „händisch an derselben Stelle wie bisher" abzulegen (vorhandene Datei ersetzen) und weist darauf hin, dass die Einstellungen dabei automatisch erhalten bleiben.

**Fehlerbehandlung:** Netzwerkfehler, ein nicht erreichbares Manifest (z. B. HTTP 404, noch keine Datei hochgeladen), kaputtes JSON im Manifest sowie ein fehlendes `htmlUrl`-Feld führen alle einheitlich zu einer verständlichen Fehlermeldung („Versionsprüfung nicht möglich (keine Internetverbindung oder das Update-Verzeichnis ist gerade nicht erreichbar)" + technische Detailmeldung) statt zu einem Absturz — der Knopf wird in jedem Fall wieder aktiviert.

**Einstellungen bleiben erhalten:** `localStorage` wird bei `file://`-Seiten in Chromium/WebView2 über den **Dateipfad** verankert, nicht über den Dateiinhalt. Da sowohl der WebView2-Auto-Install-Pfad als auch ein manuell ersetzter Browser-Download die Datei exakt an ihrem bisherigen Pfad/Namen belassen (nie umbenennen oder verschieben), bleiben alle in 5.9 gelisteten `localStorage`-Schlüssel (Maschinentypdatei, Farbeinstellungen, Arbeitsgang-Namen, Knowledge Base usw. — exakt der Umfang von „Einstellungen exportieren") nach einem Update automatisch erhalten, ganz ohne eigene Migrations-Logik.

**Manuelle Pflege bei jeder neuen Version:** Im GitHub-Repository müssen bei jeder neuen Version zwei Dateien aktualisiert werden — die neue `CNC_Simulation.html` sowie eine `latest.json` mit passender Releasenummer/Zeitstempel/`htmlUrl`. Ein automatisches Hochladen ins Repository durch das Tool selbst ist nicht vorgesehen.

### 10.4 Datenschutzerklärung / Impressum

Zwei Einträge im „?"-Menü (siehe 10, Punkt 5): „Datenschutzerklärung" (`#btnShowPrivacy`) und „Impressum" (`#btnShowImprint`). Ein Klick verhält sich abhängig von der Laufzeitumgebung unterschiedlich:

- **Innerhalb der dedizierten Windows-Host-Anwendung (WebView2, erkannt über `isRunningInHostWebView()`, siehe 10.3):** Der Klick sendet eine kleine Nachricht an den C#-Host — `window.chrome.webview.postMessage({type: 'cncsim-show-privacy'})` bzw. `{type: 'cncsim-show-imprint'}` — der daraufhin ein natives, rein lesbares (nicht editierbares) Fenster innerhalb der Host-Anwendung mit dem jeweiligen Text öffnet.
- **Im gewöhnlichen Browser-Tab (kein WebView2-Host verfügbar):** Da dort kein natives Fenster zur Verfügung steht, öffnet derselbe Klick stattdessen einen neuen Browser-Tab mit demselben reinen Text, über eine Blob-URL.

Der eigentliche Text (Datenschutzerklärung bzw. Impressum) liegt in einem JavaScript-Objekt `LEGAL_TEXTS` in `app.js` — der genaue Inhalt ist nicht Gegenstand dieser Spezifikation, die sich auf Verhalten/Architektur des Werkzeugs beschränkt.



## 11. Editoren

**Enter bestätigt, Escape bricht ab — global, für alle Eingabe-Dialoge:** Praktisch alle Eingabe-Dialoge der Anwendung — die sechs Editoren dieses Abschnitts, „Werkzeugdaten einfügen" und „DXF verschieben", die übrigen DXF-Dialoge „Drehen"/„Bauteilstärke"/„Skalieren" (9.8c) sowie „Farben einstellen"/„Schriftfarbe einstellen"/„Spalteneinstellungen"/„Buttonfarbe einstellen" (12) — teilen sich dieselbe `.paste-panel`-Struktur (sichtbar via CSS-Klasse `.open`) mit genau einem primären Bestätigen-Knopf (`.paste-actions .btn.primary`) und einem „Abbrechen"-Knopf, dessen ID stets auf „Cancel" endet (`.paste-actions button[id$="Cancel"]`). Ein einziger globaler `window`-`keydown`-Listener (direkt neben der bestehenden Escape-Behandlung für die Menüs, siehe Abschnitt 10) deckt dadurch automatisch **alle** diese Dialoge ab, auch künftig neu hinzukommende, ohne pro Dialog eigens verdrahtet werden zu müssen:

- **Escape** klickt (sofern ein `.paste-panel.open` existiert) dessen Cancel-Knopf — exakt dieselbe Wirkung wie ein Klick auf „Abbrechen": der Dialog schließt sich, ohne die Eingabe zu übernehmen. Funktioniert auch mit Fokus innerhalb eines mehrzeiligen Textfelds (z. B. „Programm bearbeiten").
- **Enter** klickt den primären Bestätigen-Knopf des aktuell offenen Panels — mit zwei Ausnahmen: Innerhalb eines `<textarea>` (die vier mehrzeiligen Editoren „NC-Code einfügen", „Programm bearbeiten", „Maschinentypen bearbeiten", „AG-Namen bearbeiten") fügt Enter weiterhin ganz normal einen Zeilenumbruch ein, statt den Dialog zu bestätigen. Und Felder, die bereits einen eigenen Enter-Handler besitzen (die „Zeile:"-Sprungfelder, sowie die einzelnen DXF-/Werkzeug-/Dateiname-Eingabefelder, die direkt ihre jeweilige `confirm…()`-Funktion aufrufen) lösen dabei bereits selbst `e.preventDefault()` aus — der globale Listener erkennt das über `e.defaultPrevented` und tut in diesem Fall nichts zusätzlich (z. B. bestätigt „zur Zeile springen" dadurch NICHT zugleich den ganzen Dialog).
- Bewusst NICHT über `.btn.ghost` (statt `button[id$="Cancel"]`) angesteuert: Der Skalieren-Dialog hat mit „Originalgröße" einen zweiten ghost-Knopf, der von Escape nicht gemeint ist.

**Kein Schließen mehr durch Klick auf den Hintergrund:** Jedes `.paste-panel` schließt sich ausschließlich über den primären Bestätigen-Knopf, den „Abbrechen"-Knopf, Enter (bestätigt) oder Escape (bricht ab) — siehe die beiden Absätze direkt oberhalb. Ein Klick irgendwo außerhalb der Karte selbst (auf den abgedunkelten Hintergrund) hat keinerlei Wirkung: Eine Textmarkierung per Maus INNERHALB eines Eingabe-/Textfelds (z. B. im Radius-Feld von „+Werkzeuge (T)", siehe 9.7.5), die versehentlich über den Feldrand hinaus bis auf den Hintergrund gezogen und dort losgelassen wird, zählt im Browser sonst als ganz normaler Klick mit dem Hintergrund als Ziel — ein Schließen-bei-Hintergrundklick-Verhalten würde den Dialog dadurch ungewollt schließen, im schlimmsten Fall mit Verlust noch nicht übernommener Eingaben. Dieses Verhalten ist deshalb bei allen 16 `.paste-panel`-Dialogen der Anwendung bewusst deaktiviert (Maschinentypen bearbeiten, Hintergrundfarbe, Schriftfarbe, Arbeitsgangfarbe, Spalteneinstellungen, Buttonfarbe, AG-Namen bearbeiten, Knowledge Base, Programm bearbeiten, Speichern als…, Werkzeugdaten einfügen, NC-Code einfügen, DXF Verschieben/Drehen/Bauteilstärke/Skalieren).

Sechs identisch aufgebaute Overlay-Panels („Paste-Panels"), jeweils mit „Abschicken" (primär) und „Abbrechen" (ghost):

| Panel | Zweck |
|---|---|
| CNC-Code einfügen | Programmtext direkt einfügen statt Datei zu laden |
| Maschinentypen bearbeiten | Komplette Maschinentyp-Konfiguration (alle Blöcke, Abschnitt 5) direkt als Text bearbeiten, inkl. nicht-blockierender Duplikat-Namen-Warnung |
| Arbeitsgang-Namen bearbeiten | Aktive Namensliste (Abschnitt 6) direkt bearbeiten |
| Knowledge Base | Editierbare `[…]`-Eintragsliste (Abschnitt 10.1a) direkt bearbeiten — einziges Panel mit zusätzlicher A-Z-Sprungleiste, „Nach oben"-Knopf und per `.paste-card.resizable` größenveränderbarer Karte (seitlich wie nach unten) |
| CNC-Programm bearbeiten | Geladenen Programmtext direkt bearbeiten (Änderungen fließen sofort in Anzeige/Simulation ein) |
| Speichern als… | Dateiname-Eingabe zum Herunterladen des aktuellen (ggf. bearbeiteten) Programmtextes |

(Die Panels selbst tragen ihre eigenen Überschriften „NC-Code einfügen" bzw. „Programm bearbeiten", unabhängig von den Menüpunkt-Beschriftungen aus Abschnitt 10.)

Zusätzlich zwei weitere `.paste-panel`-Dialoge nach demselben Grundmuster: „Werkzeugdaten einfügen" (`#toolDataPanel`, siehe 9.7) sowie „DXF verschieben" (`#dxfMovePanel`, drei Felder X/Y/Z statt zwei, siehe 9.8) — beide eher Werkzeug-/Eingabedialoge als reine Text-Editoren, folgen aber derselben Öffnen/Abbrechen/Übernehmen/Enter-bestätigt/Escape-bricht-ab-Technik (siehe Kasten oben).

„CNC-Code einfügen" und „CNC-Programm bearbeiten" besitzen zusätzlich eine Zeilennummer-Spalte (Gutter, synchron zum Scrollen des Textfelds) sowie ein eigenes „Zeile:"-Sprungfeld (`jumpToTextareaLine`) — das bewegt Cursor/Auswahl/Scrollposition innerhalb des Textfelds und ist unabhängig von der „Sprung in Zeile:"-Funktion der Transportleiste (die stattdessen die Wiedergabeposition im 3D-Viewer bewegt).

**Arbeitsgang-Sprung in „CNC-Programm bearbeiten":** Neben dem Zeilensprung-Feld sitzt (durch eine dünne Trennlinie abgesetzt) ein zusätzliches Dropdown „Arbeitsgang:" mit jedem per `current.sections` bereits erkannten Arbeitsgang (`index`. `title`, dieselbe Liste wie in der Arbeitsgänge-Spalte, siehe 7.2/9.3) — `populateProgramJumpOps()` füllt es beim Öffnen des Editors (`btnEditProgram`-Klick) frisch aus dem zu diesem Zeitpunkt aktuellen `current`. Eine Auswahl springt sofort per `jumpToTextareaLine()` zur Startzeile des gewählten Arbeitsgangs (`sec.startIdx + 1`, da 1-indexiert erwartet) — die gesamte Startzeile wird markiert und mittig in den sichtbaren Bereich gescrollt. Das Dropdown springt nach jeder Auswahl selbst sofort auf den (deaktivierten) Platzhalter „Springen zu…" zurück, damit sich derselbe Arbeitsgang erneut auswählen lässt (ein `<select>` feuert sonst bei zweimaliger Wahl desselben Werts kein `change`-Ereignis). Dropdown und Trennlinie bleiben verborgen, wenn das aktuell geladene Programm kein erkanntes Arbeitsgang-Schema enthält — dieselbe Fallback-Logik wie bei der Arbeitsgänge-Spalte selbst (7.2/7.3). Im Editor selbst vorgenommene Einfügungen/Löschungen verschieben die ursprünglich ermittelten Startzeilen — dasselbe Verhalten wie beim reinen Zeilensprung.

„Speichern als…" lädt den aktuellen `rawProgramText` als `Blob` (`type: 'text/plain;charset=utf-8'`) über einen unsichtbaren `<a download>`-Link herunter, ohne systemeigenen Speichern-Dialog.

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
--path-7:#b5507a  --path-8:#5c6bc0  --path-9:#7a8a3d  --path-10:#a15c2e
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
--path-7:#e78ab0  --path-8:#9aa8f0  --path-9:#b8cc6a  --path-10:#e0935a
--tool-color:#b7c6d2  --dxf-color:#9fb3c2
```

`--path-1` … `--path-10` sind die zyklisch verwendeten Bahn-/Zeilenfarben je Arbeitsgang (`pathColor(i)`, `OP_COLOR_COUNT = 10`, siehe 12.4). `--border`/`--border-soft` im hellen `:root`-Theme liefern flächendeckend jede Trennlinie zwischen Programmcode-/Arbeitsgänge-/3D-Bereich, jeden Panel-/Dialog-Rahmen, den Menüband-Trennstrich und Ähnliches; der System-Dark-Mode sowie der explizite Dark-Mode-Toggle verwenden dafür eigene, unabhängige Werte (siehe „Dunkel" oben). Die „Benutzerfarben"-Option (12.1) überschreibt ausschließlich die drei Bereichs-Hintergründe, nicht `--border`/`--border-soft`.

**Schriftarten:** „IBM Plex Mono" bleibt ausschließlich für tatsächlich angezeigten/bearbeiteten CNC-Programmcode reserviert — die Code-Anzeige selbst (`.code-line`, `.tree`), ihre Zeilennummern-Gutter (`.editor-gutter`) sowie die beiden Textfelder, die rohen Programmtext bearbeiten: „Code einfügen" (`#pasteArea`) und „Programm bearbeiten" (`#programEditArea`). Alle übrigen Schriftstellen der gesamten Oberfläche — Arbeitsgang-Index/-„3D"-Knopf, „Alle AG ausblenden", Readout-/Mess-Panel, Transport-/Sprung-Eingaben, Dateiname-Anzeige, Werkzeugleisten-Knöpfe sowie die Textfelder „Maschinentypen bearbeiten"/„Arbeitsgang-Namen bearbeiten"/„CNC Sim"-Dateiname — verwenden „Open Sans" (`:root`s Basis-`font-family`). Die per Canvas-2D `measureText()` gemessenen Hilfskonstanten für die automatische Spaltenbreiten-Berechnung (`OPIDX_FONT`, `OPTITLE_FONT`, `OP3D_FONT`, `OPS_HEAD_TITLE_FONT`, `OPS_SHOWALL_FONT` in `app.js`) sowie die Achsenbeschriftung „X"/„Y"/„Z" im 3D-Canvas selbst sind entsprechend gesetzt, damit Layout-Messung und tatsächliche Darstellung übereinstimmen; `TREE_FONT` (echter Programmcode) bleibt „IBM Plex Mono". Der Google-Fonts-`<link>` lädt „Open Sans" (Gewichte 400/500/600/700) und „IBM Plex Mono".

### 12.1 Farbprofile

Menüpfad: „Einstellungen" → „Erscheinungsbild" → „Farben" → „Hintergrundfarbe" (`#btnEditColors`), öffnet das Panel `#colorProfilePanel`.

**Drei Modi** (Radio-Auswahl im Panel, `applyColorProfile()` in `app.js`):
- **Standard** — erzwingt per `data-theme="light"` die eingebaute helle Palette, unabhängig von der Hell-/Dunkel-Einstellung des Browsers/Betriebssystems.
- **Dark Mode** — erzwingt per `data-theme="dark"` die eingebaute dunkle Palette.
- **Benutzerfarben** — lässt die übrige Oberfläche (Menüband, Knöpfe, Editoren, Buttons …) bewusst unangetastet weiter der Browser-/System-Einstellung folgen (kein `data-theme` gesetzt) und überschreibt ausschließlich drei Hintergründe per Hexcode: „Hintergrund CNC-Programmcode" (Hintergrund der Programmcode-Spalte `#sidebarCol`), „Hintergrund Arbeitsgänge" (`#opsCol`) und „Hintergrund 3D Simulation" (der 3D-Canvas selbst — malt seinen Hintergrund pro Frame per `ctx.fillRect()` aus der globalen CSS-Variable `--viewer-bg`, siehe `render()`). Jede Zeile hat sowohl ein Text- als auch ein natives `<input type="color">`-Feld, die sich gegenseitig synchron halten; ein ungültiger Hexcode (weder 3- noch 6-stellig) zeigt eine Warnung und verhindert „Übernehmen", statt kommentarlos zu übernehmen oder abzustürzen.
- Solange „Hintergrundfarbe" noch nie geöffnet/bestätigt wurde (weder in dieser Sitzung noch per `localStorage`), bleibt die Seite beim automatischen Verhalten — kein `data-theme` gesetzt, keine Hintergründe überschrieben.

**Automatische Lesbarkeit:** `deriveReadableInk(bgHex)` berechnet aus der relativen Luminanz (WCAG-Formel) der gewählten Hintergrundfarbe, ob eine helle oder dunkle Haupt-Schriftfarbe nötig ist (Schwelle 0.5), und blendet daraus zwei abgestufte Varianten (35 %/60 % Richtung Hintergrundfarbe) für „gedämpfte"/„blasse" Schrift — dieselbe optische Hierarchie wie bei den beiden eingebauten Paletten, nur dynamisch statt fest kodiert. Für Programmcode- und Arbeitsgänge-Spalte werden Hintergrund und die drei Schriftfarben-Variablen (`--ink`/`--ink-muted`/`--ink-faint`) direkt per Inline-Style auf genau dem jeweiligen Spalten-Element gesetzt (bewusst nicht als selbstreferenzierende CSS-Regel im Stylesheet — eine Formel wie `--ink: var(--code-ink, var(--ink))` im Stylesheet selbst würde einen CSS-Abhängigkeitszyklus erzeugen und `--ink` komplett ungültig machen; per Inline-Style gesetzte konkrete Werte umgehen dieses Risiko vollständig) — dank normaler CSS-Vererbung wirkt das automatisch auf jeden Text innerhalb der jeweiligen Spalte. Für den 3D-Bereich wird zusätzlich die Gitterlinienfarbe (`--grid-line`) passend zur neuen Kontrastfarbe eingefärbt (gleiche leichte Transparenz wie bei den eingebauten Paletten); ebenso folgen die Umrandung des Mess-Punkts und das „Loch" des Positionsmarkers `--viewer-bg` statt der fest verwendeten `--panel`, damit beide optisch zum tatsächlichen 3D-Hintergrund passen.

**Persistenz:** Das gewählte Profil (Modus + bei Benutzerfarben die drei Hexcodes) wird in `localStorage` gespeichert und beim nächsten Seitenaufruf automatisch wieder angewendet — analog zur Maschinentyp-Konfiguration (Abschnitt 5).

**„Einstellungen exportieren"/„Einstellungen importieren":** Die JSON-Einstellungsdatei (Abschnitt 5.9) enthält ein Textfeld `colorProfileText` in folgender Klammer-Syntax mit den drei tatsächlich aktiven (aufgelösten) Hintergrundfarben — unabhängig davon, ob gerade „Standard", „Dark Mode" oder „Benutzerfarben" aktiv ist:
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
Da nur konkrete Hexcodes gespeichert werden (kein separates Modus-Feld), wendet „Einstellungen importieren" diese drei Werte beim Wiedereinlesen immer als „Benutzerfarben" an — das stellt das exakte sichtbare Ergebnis wieder her, auch wenn die ursprüngliche Auswahl „Standard" oder „Dark Mode" war. Fehlt das Feld (ältere/handgeschriebene Datei), bleibt das aktuell aktive Farbprofil unangetastet, dieselbe Regel wie bei den übrigen Einstellungsteilen.

### 12.2 Buttonfarbe — Menü-Hervorhebungsfarbe

Menüpunkt „Buttonfarbe" (`#btnMenuColor`, „Einstellungen" → „Erscheinungsbild" → „Farben") öffnet den Dialog `#menuColorPanel` mit zwei Hexcode-Feldern (je mit synchronisiertem `<input type="color">`-Farbwähler, dieselbe Text/Picker-Kopplungstechnik wie bei den Farbprofil-Feldern, 12.1): „Hintergrundfarbe" (`#menuColorBgHex`/`#menuColorBgPicker`) und „Schriftfarbe" (`#menuColorInkHex`/`#menuColorInkPicker`), sowie „Abbrechen"/„Übernehmen".

**Wirkung:** Die gewählte Farbe färbt ausschließlich den Zustand „Menü gerade geöffnet" der Menü-Trigger „Datei"/„Bearbeiten"/„Einstellungen"/„Hilfe" (`.menu.open .menu-trigger`) — eine Farbe für alle vier gemeinsam. Der Standardzustand (Menü geschlossen) bleibt unberührt. **Ausnahme:** Das „?"-Menü (siehe 10) ist davon bewusst ausgenommen — seine Schriftfarbe ist per `#menuInfoTrigger`-Regel fest auf Orange gesetzt und bleibt das auch bei geöffnetem Menü, unabhängig von der hier gewählten Buttonfarbe (siehe Abschnitt 10, Punkt 5).

**Eigene CSS-Variablen statt Wiederverwendung von `--accent`:** Die Farbe wird über zwei eigens dafür angelegte CSS-Variablen `--menu-active-bg`/`--menu-active-ink` gesetzt, nicht über die bereits bestehenden `--accent`/`--accent-ink`, die an vielen weiteren Stellen der Oberfläche wiederverwendet werden (Fokusringe, Kippschalter, aktive Arbeitsgang-Hervorhebung, Badges u. a.) — eine direkte Umfärbung von `--accent` hätte all diese Stellen ungewollt mit ausgeweitet. `.menu.open .menu-trigger { background: var(--menu-active-bg, var(--accent)); color: var(--menu-active-ink, var(--accent-ink)); border-color: transparent; }` fällt über die CSS-`var()`-Fallback-Syntax automatisch auf die bisherige Optik (`--accent`/`--accent-ink`, Rot) zurück, solange keine eigene Menüfarbe gesetzt ist.

**Standard:** `DEFAULT_MENU_COLOR = { bg: '#ad1f2d', ink: '#ffffff' }` — dieselbe Rot-/Weiß-Kombination, die zuvor implizit über `--accent`/`--accent-ink` galt.

**Validierung „Schrift- und Hintergrundfarbe dürfen nicht identisch sein":** `applyMenuHighlightColor(color)` prüft Gültigkeit beider Hexcodes und dass sie sich voneinander unterscheiden (case-insensitiver String-Vergleich der normalisierten Hexcodes); ist eine der beiden Bedingungen verletzt, bleibt die zuvor aktive Farbe vollständig unverändert, der Dialog zeigt stattdessen den Hinweistext `#menuColorWarning` („Schrift- und Hintergrundfarbe dürfen nicht identisch sein.") und bleibt geöffnet.

**Persistenz:** `activeMenuColor = { bg, ink }` wird über `localStorage` (`LS_KEY_MENU_COLOR`) dauerhaft gespeichert (`persistMenuColor()`, try/catch-abgesichert wie alle übrigen Persistenz-Mechanismen, siehe 5.8) und beim Programmstart mit `loadPersistedMenuColor()` geladen und sofort angewendet. Zusätzlich Teil des JSON-Exports/-Imports „Einstellungen exportieren"/„Einstellungen importieren" (`menuHighlightBg`/`menuHighlightInk`, siehe 5.9).

### 12.3 Schriftfarbe einzeln festlegen + „Standardwerte wiederherstellen"

Menüpunkt „Schriftfarbe" (`#btnFontColor`) im „Farben"-Untermenü des „Einstellungen"-Menüs (`#submenuColorsDropdown`), direkt nach „Hintergrundfarbe" (12.1). Öffnet das Panel `#fontColorPanel` mit fünf unabhängigen Zeilen — für jeden Bereich eine eigene Zeile mit Häkchen + Hexcode-Textfeld + `<input type="color">`-Farbwähler (dieselbe Text/Picker-Kopplungstechnik wie bei „Hintergrundfarbe", 12.1, und „Buttonfarbe", 12.2):

- **CNC-Programmcode Zeilennummerierung** — die Zeilennummern-Spalte der Code-Anzeige (`.code-line .ln`)
- **CNC-Programmcode Code** — der eigentliche Programmcode-Text daneben (`.code-line .tx`)
- **Arbeitsgänge** — Index und Titel jedes Eintrags in der Arbeitsgänge-Spalte (`.op-item .op-idx`/`.op-title`)
- **Geschwindigkeit** — Beschriftung und Wert der vier Geschwindigkeitsstufen-Knöpfe im Transportbereich (`.speed-caption`, `.speed-btn`)
- **Sprung in Zeile** — Beschriftung und Eingabefeld des „Sprung in Zeile"-Reglers (`.jump-group label`, `.jump-group input[type="number"]`)

**Jede Zeile unabhängig ein-/ausschaltbar:** Ein aktiviertes Häkchen setzt für genau diesen Bereich die daneben gewählte Farbe fest; bleibt es deaktiviert, verwendet dieser Bereich unverändert automatisch die zum aktiven Hintergrund passende Farbe (siehe „Hintergrundfarbe"/`deriveReadableInk()`, 12.1) — die Funktion ist eine reine, optionale Ergänzung „obendrauf", keine Ablösung der automatischen Lesbarkeits-Logik. Ein ungültiger Hexcode bei einer aktivierten Zeile zeigt eine Warnung (`#fontColorHexWarning`) und verhindert „Übernehmen".

**Technische Umsetzung — zweiargumentiger `var()`-Fallback statt neuer `:root`-Variablen:** Eine neue Variable wie `--font-code-text: var(--ink);` direkt am `:root` zu deklarieren würde die „Benutzerfarben"-Funktion (12.1) brechen: Dort werden `--ink`/`--ink-muted`/`--ink-faint` bewusst nicht global am `:root`, sondern per Inline-Style direkt auf den einzelnen Spalten-Elementen (`#sidebarCol`/`#opsCol`) gesetzt, damit jede Spalte ihre eigene, zu ihrem eigenen Hintergrund passende Schriftfarbe erhält — eine globale `:root`-Variable, die sich beim Deklarieren bereits auf den globalen `--ink`-Wert bezieht, würde diese Spalten-genaue Auflösung umgehen. Stattdessen wird der Fallback direkt an jeder betroffenen CSS-Verwendungsstelle selbst eingebaut, z. B.:
```css
.code-line .ln { color: var(--font-code-linenum, var(--ink-faint)); }
.code-line .tx { color: var(--font-code-text, var(--ink)); }
```
Solange `--font-code-text` nirgends gesetzt ist, löst der Browser den inneren Fallback `var(--ink)` genau dort im Baum auf, wo die Regel tatsächlich greift — innerhalb von `#sidebarCol` bekommt er also weiterhin den dortigen Inline-Style-Wert von „Benutzerfarben" zu sehen. Erst wenn `applyFontColors()` (`app.js`) die jeweilige `--font-*`-Variable per `root.style.setProperty()` global am `:root` setzt, gewinnt sie überall — unabhängig vom lokalen `--ink`. Dieselbe Technik wird an allen fünf betroffenen Stellen angewendet, für „Arbeitsgänge" und „Sprung in Zeile" jeweils mit zwei eigenen Variablen (`--font-ops-idx`/`--font-ops-title` bzw. `--font-jump-label`/`--font-jump-input`), da Index/Titel bzw. Beschriftung/Eingabefeld unterschiedliche Standard-Fallbacks haben (`--ink-faint` bzw. `--ink-muted` statt `--ink`), beim Setzen einer eigenen Farbe aber beide gemeinsam dieselbe gewählte Farbe erhalten.

**Bewusste Ausnahme — „aktueller Arbeitsgang" behält seine Akzentfarbe:** Der jeweils aktuell aktive Arbeitsgang wird unverändert in der Akzentfarbe hervorgehoben (`.op-item.current .op-idx, .op-item.current .op-title { color: var(--accent); font-weight: 600; }`) — diese Regel ist spezifischer als die `.op-item .op-idx`-Regel und gewinnt daher weiterhin, auch wenn für „Arbeitsgänge" eine eigene Schriftfarbe gesetzt ist. Das ist beabsichtigt: Die Akzent-Hervorhebung ist ein Status-Hinweis, keine normale Textfarbe, und soll unabhängig von der gewählten Schriftfarbe erkennbar bleiben.

**„Standardwerte wiederherstellen"** — flacher Menüpunkt `#btnResetDisplayDefaults` direkt im „Einstellungen"-Menü (nicht im „Farben"-Untermenü), unmittelbar nach „Spalteneinstellungen" (`#btnColumnScale`). Ein Klick fragt zuerst per nativem `window.confirm()`-Dialog nach („Standardwerte werden wiederhergestellt, alle vorherigen Einstellungen zurückgesetzt. Fortführen?"); nur bei „OK" (Ja) ruft `resetAllDisplaySettingsToDefaults()` (`app.js`) tatsächlich alle fünf Anzeige-Einstellungen in einem Schritt zurück, bei „Abbrechen" (Nein) bricht die Funktion sofort ab, ohne dass irgendeine der fünf Einstellungen angerührt wird:
- **Hintergrundfarbe** — entfernt den persistierten Farbprofil-Eintrag, wodurch das automatische Standard-/Dark-Mode-Verhalten (kein `data-theme` erzwungen) wieder greift.
- **Buttonfarbe** — setzt die Menü-Hervorhebungsfarbe zurück auf `DEFAULT_MENU_COLOR` (Rot/Weiß, siehe 12.2).
- **Schriftfarbe** — entfernt alle fünf `--font-*`-Variablenpaare wieder vollständig vom `:root` (identisch zum Zustand „noch nie geöffnet") und löscht den zugehörigen `localStorage`-Eintrag.
- **Arbeitsgangfarbe** — entfernt alle zehn `--path-1…10`-Overrides wieder vollständig vom `:root` (das automatische Standard-/Dark-Mode-Verhalten der zehn eingebauten Bahn-/Zeilenfarben greift wieder), löscht den zugehörigen `localStorage`-Eintrag und färbt eine bereits geladene Szene sofort entsprechend neu ein (siehe 12.4).
- **Spaltenbreiten** — setzt `columnScalePreference` zurück auf die eingebauten Standardwerte (je 100 %, siehe 7.3a) und wendet diese sofort auf ein gerade geladenes Programm an, falls eines aktiv ist.

Alle fünf zugehörigen Bedienpanels (sofern gerade geöffnet oder als Nächstes geöffnet) zeigen unmittelbar danach wieder ihren jeweiligen Ausgangszustand, ohne dass ein erneutes Neuladen der Seite nötig ist. Bewusst der native `window.confirm()` statt eines eigenen `.paste-panel`-Dialogs: Der native Dialog blockiert zuverlässig jede weitere Interaktion bis zur Entscheidung, benötigt kein eigenes Markup/CSS und bildet die geforderte Ja/Nein-Logik direkt 1:1 ab.

**Abgrenzung zu „Maschinentypen zurücksetzen" (Release 134, siehe 5.7b):** Dieser Knopf hier betrifft ausdrücklich NICHT die Maschinentyp-Konfiguration — umgekehrt betrifft der engere, erst seit Release 134 existierende „Maschinentypen zurücksetzen"-Knopf im Panel „Maschinentypen bearbeiten" (5.7b) ausschließlich die Maschinentypen und keine der fünf hier aufgeführten Anzeige-Einstellungen. Beide Mechanismen sind bewusst getrennt gehalten (je eigener `window.confirm()`-Dialog, je eigene Funktion) statt zu einem gemeinsamen „alles zurücksetzen"-Knopf zusammengefasst, da sie an unterschiedlichen Stellen im Menü/den Panels sitzen und typischerweise aus unterschiedlichem Anlass ausgelöst werden (Anzeige-Vorlieben vs. gecachte Maschinentyp-Konfiguration nach einem Dateiaustausch).

**JSON-Export/-Import („Einstellungen exportieren"/„Einstellungen importieren", Abschnitt 5.9):** Die exportierte Einstellungsdatei enthält ein Feld `fontColors` mit genau den aktuell aktivierten Schriftfarben-Zeilen (z. B. `{"code":"#00ff00","speed":"#fedcba"}` — nur tatsächlich gesetzte Zeilen, keine leeren/deaktivierten). Beim Wiedereinlesen wendet „Einstellungen importieren" diese Werte über dieselbe `applyFontColors()`-Funktion wie beim regulären „Übernehmen" an und persistiert sie ebenso in `localStorage`; fehlt das Feld (ältere/handgeschriebene Datei), bleiben die aktuell aktiven Schriftfarben unangetastet — dieselbe Rückwärtskompatibilitäts-Regel wie bei den bereits bestehenden Einstellungsteilen (Farbprofil, Spaltenbreite, Menü-Hervorhebungsfarbe).

### 12.4 Arbeitsgangfarbe

Menüpunkt „Arbeitsgangfarbe" (`#btnOpColor`, „Einstellungen" → „Erscheinungsbild" → „Farben", direkt nach „Schriftfarbe", 12.3) erlaubt es, für jeden Arbeitsgang gezielt festzulegen, welche Farbe der 1. Arbeitsgang bekommt, welche der 2., der 3. usw., statt sich auf die automatische Durchnummerierung zu verlassen.

**Zyklus auf zehn Farben:** Einzige zentrale Stellschraube ist die Konstante `OP_COLOR_COUNT = 10` (direkt vor `pathColor()`), von der alle relevanten Stellen (`pathColor()`s Namens-Array, die Längenprüfung in `loadPersistedOpColors()`, Längenprüfung + Schleifengrenze in `applyOpColors()`, die Zeilen-Erzeugung `opColorFields`) die Zykluslänge ausschließlich ableiten.

**Hintergrund — die zehn zyklischen Bahn-/Zeilenfarben:** Jeder Arbeitsgang erhält seine Farbe über `pathColor(i)`, das zyklisch durch zehn feste CSS-Variablen `--path-1` … `--path-10` rotiert (Arbeitsgang 11 bekommt wieder die Farbe von `--path-1`, usw. — dieses Durchnummerieren ist bewusst so). Der Bediener kann diese Basisfarben selbst festlegen, statt nur die eingebaute Standard-/Dark-Mode-Palette zu bekommen. Die vier Farben `--path-7` … `--path-10` sind bewusst deutlich unterscheidbar von den übrigen sechs sowie voneinander gewählt (hell: `#b5507a` Rosé, `#5c6bc0` Indigo, `#7a8a3d` Oliv, `#a15c2e` Rost; passende aufgehellte/entsättigte Varianten im Dark-Mode) und in allen drei Theme-Blöcken (`:root`, `@media (prefers-color-scheme: dark)`, `:root[data-theme="dark"]`) parallel hinterlegt.

**Panel `#opColorPanel`:** zehn Zeilen „1. Arbeitsgang" … „10. Arbeitsgang", jede mit Hexcode-Textfeld + synchronisiertem `<input type="color">`-Farbwähler (dieselbe Text/Picker-Kopplungstechnik wie bei „Hintergrundfarbe"/„Buttonfarbe"/„Schriftfarbe", 12.1–12.3), Warnhinweis `#opColorHexWarning` bei ungültigem Hexcode (verhindert „Übernehmen"), sowie „Abbrechen"/„Übernehmen". Ohne eigene Übersteuerung zeigt das Panel beim Öffnen die aktuell wirksamen (Standard- bzw. Dark-Mode-)Hexcodes als sinnvollen Ausgangspunkt vor. Anders als bei „Buttonfarbe" (12.2, dort müssen die beiden Farben unterscheidbar sein) gibt es hier **keine** Unterscheidbarkeits-Prüfung — der Bediener darf ausdrücklich auch alle zehn Zeilen auf denselben Hexcode setzen; nur die automatische Standardpalette selbst ist bewusst zehnfarbig unterschiedlich.

**Technische Umsetzung — dieselbe Übersteuerungstechnik wie „Hintergrundfarbe" (12.1):** `applyOpColors(colors)` setzt bei gültigen zehn Hexcodes `--path-1` … `--path-10` per `root.style.setProperty()` direkt am `:root`, unabhängig von Hell-/Dark-Mode; ohne aktive Übersteuerung (`colors = null`) werden alle zehn Variablen wieder entfernt, wodurch die eingebauten Hell-/Dark-Mode-Paletten (siehe `:root`/`@media (prefers-color-scheme: dark)`/`:root[data-theme="dark"]`) automatisch wieder greifen — exakt dasselbe „automatisch statt erzwungener Standard"-Verhalten wie bei „Hintergrundfarbe".

**Neueinfärben einer bereits geladenen Szene, ohne Kamera/Scrubber zurückzusetzen:** `pathColor(i)` bäckt seine aufgelöste Farbe bereits bei der Konstruktion in `segments[].color`/`points[].color` ein (kein Live-Nachschlagen der CSS-Variable beim Zeichnen) — eine Farbänderung nach dem Laden eines Programms würde sich dadurch ohne weiteres Zutun gar nicht auswirken, bis ohnehin ein struktureller Neuaufbau fällig wird. Ein erneutes `setScope()` hätte das zwar erzwungen, dabei aber unerwünscht auch Kamera-Framing, Scrubber-Position und Wiedergabezustand zurückgesetzt. Stattdessen trägt jedes Segment/jeder Punkt sein eigenes `colorCursor` (0–9, oder `null` für gedimmte Kontext-Bahnen wie „bereits gefertigtes Bauteil" beim Scope „einzelner Arbeitsgang" — diese bleiben bewusst in `--ink-faint` und sind von Arbeitsgangfarben nicht betroffen). `recolorOpSegmentsAndPoints()` läuft nach jeder Anwendung einer neuen Arbeitsgangfarbe (Übernehmen, Reset, Import) einmal über alle Segmente/Punkte und setzt `seg.color`/`p.color` per `pathColor(seg.colorCursor)` neu, aktualisiert zusätzlich direkt im DOM die bereits gezeichnete Farbmarkierung der Code-Zeilen (`.code-line`-Randfarbe) und der Arbeitsgänge-Swatches (`.op-swatch`-Hintergrund), ohne `renderTree()`/`renderOpsColumn()` komplett neu aufzubauen (das hätte Scrollposition/DOM-Referenzen unnötig verworfen). Da die Bake-Canvas-Dirty-Prüfung (`computeBakeParams()`/`bakeParamsEqual()`, siehe 9.5) `segments` nur per Referenz vergleicht, würde ein reines In-Place-Umfärben dort sonst unbemerkt bleiben — `recolorOpSegmentsAndPoints()` erzwingt deshalb zusätzlich `bakeParams = null`, damit der nächste Frame vollständig neu backt.

**Persistenz:** `activeOpColors` (Array aus zehn Hexcodes, oder `null`) wird über `localStorage` (`LS_KEY_OP_COLORS = 'cncsim.opColors'`) dauerhaft gespeichert (`persistOpColors()`, try/catch-abgesichert wie alle übrigen Persistenz-Mechanismen, siehe 5.8) und beim Programmstart mit `loadPersistedOpColors()` geladen und sofort angewendet — `loadPersistedOpColors()` verlangt dabei exakt `OP_COLOR_COUNT` Einträge, ein gespeichertes Array anderer Länge wird verworfen und es gilt wieder die automatische Standardpalette. Zusätzlich Teil von „Standardwerte wiederherstellen" (12.3) sowie des JSON-Exports/-Imports „Einstellungen exportieren"/„Einstellungen importieren" (`opColors`, siehe 5.9) — ein beim Import vorgefundenes Array mit abweichender Länge wird dort ebenfalls ignoriert, ohne die aktuell aktive Arbeitsgangfarbe zu verändern, statt fälschlich nur einen Teil der zehn Farben zu übernehmen.


## 13. Bekannte Einschränkungen

1. Kein automatisches Laden aus einem lokalen Dateipfad (z. B. `C:\Temp`) — aus Browser-Sicherheitsgründen nicht möglich; Dateien müssen aktiv ausgewählt, per Drag & Drop gezogen oder eingefügt werden.
2. Die Dreh-/Kippwinkel-Zuordnung (A/B/C → „Kipp A"/„Kipp B"/„Dreh C") ist eine verbreitete, aber maschinenabhängige Konvention — keine maschinenspezifische Achsbelegung ist hinterlegt.
3. Der Bahn-Viewer nutzt einen generischen ISO-Achsen-Parser; für Dialekte ohne erkennbare G0/G1-Wörter wird die Bahn standardmäßig als Eilgang (gestrichelt) dargestellt, sofern kein anderslautendes Muster (z. B. KUKA PTP/LIN/CIRC) erkannt wird.
4. SWITCHCASEMACHINE-Regeln (Maschinentyp-Auswahl, Abschnitt 5) matchen rein literal auf Zeichenketten (kein G-Code-Tokenizer) und wirken auf die gesamte Rohdatei, bevor der Arbeitsgang-Parser läuft — kurze/generische Suchtexte (insbesondere einzelne Buchstaben) können theoretisch auch unbeteiligten Text treffen. Der `detectionText`-Mechanismus (4.5) schützt gezielt die strukturellen Erkennungsmarken; für sonstigen Freitext gilt ein reduziertes Restrisiko (nur noch außerhalb der drei erkannten Kommentar-Arten `;`/`KM="…"`/`{…}` sowie innerhalb runder Klammern, siehe 5.11). CALCMACHINE (5.3) unterstützt zudem nur das einfache Schema `Achse*Faktor`, keine komplexeren Formeln.
5. Gemischte Kommentarstile im selben Programm werden zwar unterstützt (jeder Arbeitsgang merkt sich seinen eigenen erkannten Stil), ein sehr kurzes/generisches Muster eines Stils könnte in einer Datei eines anderen Stils theoretisch zufällig zutreffen. Der Biesse/CIX-Stil ist davon bewusst ausgenommen (nur bei erkannter `.CIX`-Kopfzeile aktiv). Der listenbasierte `NAMED`-Stil trägt ein analoges Restrisiko: je mehr generische/kurze Einträge die editierbare Liste enthält, desto eher könnte eine eigentlich unbeteiligte Zeile fälschlich als Arbeitsgang erkannt werden.
6. Eine Nullpunktverschiebung (`G92`/`O`/getauschtes `TRANS`, siehe 9.1) wird ausschließlich literal an `ORIGIN_RESET_RE` erkannt — sie muss als eigenes Achswort/eigene Zeile im (ggf. bereits getauschten) Programmtext vorkommen. Enthält ein und dieselbe Zeile sowohl eine Nullpunktverschiebung als auch eine tatsächliche Werkzeugbewegung (in der Praxis unüblich), wird für diese Zeile ebenfalls kein Bahnpunkt gezeichnet.
7. „↑ Oben" zeigt bewusst die Kamera-Position, die „X rechts, Y oben" liefert (siehe 9.2) — bei einem Programm, dessen Z-Achse invertiert vorliegt (Z+ nach unten statt oben, in der Praxis selten), würde dieselbe Ansicht dann fälschlich als von unten wirken. Es ist keine maschinenspezifische Achsrichtungs-Erkennung hinterlegt; die Zuordnung geht von der gängigen Konvention „X rechts, Y nach hinten, Z nach oben" aus.
8. Die Werkzeugachsen-Rotationsreihenfolge in `toolAxisDirection()` (siehe 9.7) — erst Kipp A, dann Kipp B, zuletzt Dreh C — ist eine begründete, aber nicht verifizierte Annahme über die tatsächliche Schwenkkopf-Kinematik. Bei nur einem gleichzeitig aktiven Winkel spielt sie keine Rolle; bei mehreren gleichzeitig aktiven Winkeln könnte die dargestellte Werkzeugneigung von der realen Maschine abweichen.
9. Die Voraberkennung/Vorschlag im Maschinentyp-Dropdown (5.4) ist nur eine unverbindliche Empfehlung: Sie erkennt weder eine völlig neue, noch nicht konfigurierte Maschine, noch garantiert sie bei mehrdeutigem Kopftext/Freitext den „richtigen" Vorschlag — sie ersetzt daher nicht die Prüfung durch den Benutzer vor dem Klick auf „Ok".
10. Die Reichenbacher-Dreh-/Kippwinkel-Umrechnung (siehe 9.7, „Reichenbacher-Kardanwinkelkorrektur", sowie die Projektnotiz „Notiz_Reichenbacher_Aggregat_Winkelkorrektur" für die vollständige Herleitung) ist ein eigener, abschaltbarer Schalter neben dem Dateinamen, Standardzustand „an". Die Formel ist an drei Kippwinkeln (30°/45°/90°) aus realen Beispielpaaren exakt verifiziert; ein noch feineres/weiteres Beispiel (insbesondere ein „krummer" Winkel abseits von 30°/45°/90°) wäre trotzdem hilfreich, um sie endgültig abzusichern. Der eine darin frei wählbare, skalare Parameter — der Schwenkwinkel des Aggregats, baubedingt 45° — ist über den Parameter `SWIVELANGLE=<Zahl>` im jeweiligen Maschinentyp-Block konfigurierbar (Fallback 45°, siehe 5.12); die eigentliche trigonometrische Formel selbst bleibt fest in `app.js` verdrahtet, da CALCMACHINE (5.3) dafür strukturell nicht ausreicht (siehe Punkt 4). Der Reichenbacher-Maschinentyp-Block (SWITCHCASEMACHINE/CALCMACHINE, Abschnitt 5) bleibt davon ansonsten unberührt — CALCMACHINE enthält nur Platzhalter-Werte (`X*1`, `Y*1`, `Z*1`, keine Ersetzungsregeln), die Winkelkorrektur läuft komplett unabhängig davon in `toolAxisDirection()`.
11. Die G02/G03-Bogeninterpolation (9.1) unterstützt ausschließlich die XY-Ebene (G17, in der Holzbearbeitung der Regelfall) — ein Bogen in der XZ-/YZ-Ebene (G18/G19) wird als gerade Linie dargestellt. Ein G02/G03 muss zudem auf jeder Bogenzeile explizit erneut stehen (kein modales Fortschreiben, siehe 9.1) — ein Programm, das G02/G03 mehrzeilig-modal ohne Wiederholung nutzt, bekommt für die entsprechenden Folgezeilen eine gerade statt einer gebogenen Linie. Bei Bögen mit leicht unstimmigem Start-/Zielabstand vom Mittelpunkt (Rundungsungenauigkeit realer Postprozessoren) wird der Radius ausschließlich aus dem Startpunkt berechnet; der letzte tessellierte Punkt landet trotzdem immer exakt auf dem programmierten Zielpunkt, wodurch das allerletzte Bogensegment in diesem seltenen Fall minimal von einem exakten Kreisbogen abweichen kann. Sowohl das R-Format als auch das I/J-Format werden unterstützt (siehe 9.1); beim I/J-Format entscheidet die `IJMODE`-Direktive des aktiven Maschinentyps (`incremental` bezogen auf den Startpunkt bzw. `absolute` bezogen auf den Programm-Nullpunkt, siehe 5.12a), wie I/J interpretiert werden. Ohne gewählten Maschinentyp bzw. ohne eine `IJMODE`-Zeile im aktiven Block gilt der Standardwert `incremental` — bei einem Programm, dessen tatsächlicher Dialekt I/J absolut meint (z. B. Biesse/CIX unter „ISO", also ohne bestätigten Maschinentyp), kann der dargestellte Bogen dadurch geometrisch falsch ausfallen, bis der passende Maschinentyp mit korrektem `IJMODE` gewählt und bestätigt wird.
12. **DXF-Import (9.8):** Block-Referenzen (`INSERT`) werden nicht aufgelöst — Geometrie, die nur innerhalb eines `BLOCK`s definiert, aber nie per `INSERT` platziert wird, erscheint dadurch gar nicht. Nicht unterstützte Entitätstypen (u. a. TEXT/MTEXT, HATCH, SPLINE, ELLIPSE) werden erkannt/gezählt, aber nicht gezeichnet (siehe Tooltip der Statusanzeige) — 3DFACE ist davon ausgenommen und wird unterstützt (siehe 9.8). Extrusionsrichtung/OCS (Gruppencode 210/220/230) wird bei CIRCLE/ARC/LWPOLYLINE/POLYLINE nicht berücksichtigt — eine gedrehte, nicht in der Standard-XY-Ebene liegende Kreis-/Bogen-/Polylinien-Entität kann dadurch falsch orientiert erscheinen, reine 2D-Zeichnungen in der Standard-XY-Ebene sind davon nicht betroffen. 3DFACE-Entitäten sind von dieser Einschränkung nicht betroffen — ihre vier Eckpunkte liegen laut DXF-Format immer direkt in Weltkoordinaten vor, ohne OCS-Umrechnung. Die Werkzeugachsen-Rotationsreihenfolge/Kinematik-Annahmen (Punkt 8) betreffen die DXF-Unterlage nicht, da sie rein statisch (ohne Rotation) dargestellt wird. Wie das CNC-Programm selbst wird eine importierte DXF-Unterlage nicht über `localStorage`, „Einstellungen exportieren"/„Einstellungen importieren" oder „Simulation speichern" persistiert — sie muss nach einem Seiten-Reload erneut importiert werden; dasselbe gilt für die einzeln aus-/eingeblendete Layer-Auswahl (9.8a).
13. Die inzwischen fünf parallelen Eckenverrundungs-Erkennungsmuster — „P71:" (9.1a, MAKA), „G302 I<Radius>" (9.1b, HOMAG), „GFIL r=<Radius>" (9.1c, SCM), „BR=<Radius>" (9.1d, Biesse) und „E<Radius>" (9.1e, FORMAT4), sowie der seit Release 133 je Maschinentyp konfigurierbare `CORNERRADIUS=<Marker>` (5.12b), der für den jeweiligen Maschinentyp an die Stelle eines der fünf fest einprogrammierten Muster tritt — interpolieren allesamt nur X/Y entlang des eingefügten Kreisbogens über denselben gemeinsamen `applyCornerFillets()`-Mechanismus (9.1a): Z bleibt für alle eingefügten Verrundungspunkte unverändert auf dem Wert der ursprünglichen Eckkoordinate stehen, anders als bei einer echten G02/G03-Bogenzeile (dort wird Z linear mitinterpoliert, siehe 9.1); bei einer Ecke mit gleichzeitiger Z-Änderung (z. B. einer echten 3D-Bewegung) kann die Verrundung dadurch geometrisch unsauber wirken. Da `applyCornerFillets()` nur innerhalb des bereits aufgebauten `points`-Arrays eines einzelnen `extractMotion()`-Durchlaufs arbeitet und für eine Verrundung beide Nachbarpunkte der betroffenen Ecke kennen muss, bleiben der allererste und der allerletzte Punkt einer solchen Punktfolge grundsätzlich unverrundet, selbst wenn dort ein Radius-Wert steht. Ein Radius ≤ 0 oder eine geometrisch entartete Ecke (z. B. ein Radius größer als die zulässige, auf 90 % der kürzeren Schenkellänge begrenzte Tangentendistanz) lässt die Ecke stillschweigend scharf — in keinem dieser Fälle erscheint ein Hinweis für den Benutzer. Ein fehlerhaft konfigurierter `CORNERRADIUS`-Marker (5.12b), der auf keine Zeile des Programms passt, führt ebenfalls zu keiner Fehlermeldung — die betroffenen Ecken bleiben in diesem Fall einfach durchgehend scharf, exakt wie bei einem Maschinentyp ganz ohne eigene `CORNERRADIUS`-Zeile.


## 14. Automatisierte Tests

`test.js` (Node, `node test.js`) bindet `parser.js` und `swap.js` direkt per `require()` ein und prüft ausschließlich deren reine Logik (kein Browser/DOM nötig). Abgedeckt sind unter anderem: Maschinenerkennung für alle unterstützten Dialekte; alle Arbeitsgang-Kommentarstile (SCM, HOMAG, FORMAT4, MOROFF, KRC_BLOCK, CIX/Biesse, NAMED); Zwischenstopp-/Programmende-Muster; Nullpunktverschiebungs-Erkennung (`splitAtOriginResets`/`ORIGIN_RESET_RE`); SWITCHCASEMACHINE-/CALCMACHINE-Regelanwendung sowie `parseMachineTypes()`/`findDuplicateMachineTypeNames()`/`sortMachineTypeBlocksAlphabetically()`; `extractToolData()` (RADIUS/LENGTH/BLADE-Erkennung über alle unterstützten Kommentarstile); `negateYZ()`/`applyAxisCalc()`; `computeCommentMask()`/`isRangeUncommented()` (Kommentar-Maskierung für `;`, `KM="…"`, `{…}`, bewusste Nicht-Maskierung runder Klammern); `parseArcParams()`/`interpolateArc()` (G02/G03-Bogeninterpolation, ausschließlich R-Format); `parseSwivelAngle()`/`swivelAngleDeg` (Reichenbacher-Schwenkwinkel); sowie ein umfangreicher Testblock für `parseDXF()` (LINE, POINT, CIRCLE, ARC, LWPOLYLINE mit/ohne Bulge, klassisches POLYLINE/VERTEX/SEQEND, 3DFACE, nicht unterstützte Entitätstypen, leere/fehlerhafte Eingabe, SECTION/ENTITIES-Scoping, CRLF-Zeilenenden). Aktueller Stand: **281 grüne Assertionen**, Ausgabe endet mit „Alle Tests erfolgreich." bei Erfolg.

Reine `app.js`/DOM-/Canvas-Logik ohne Bezug zu `parser.js`/`swap.js` — Menüband und Symbolleisten, Farbprofile/Schriftfarben, das Messwerkzeug, die 3D-Kamera samt Gizmo, die Werkzeug-3D-Darstellung, der DXF-Import als UI (Menüpunkte, Statusgruppe, Verschieben-Dialog, Ein-/Ausblenden, Kamera-Framing), „Simulation speichern" sowie der Host-Hook für eine optionale native Windows-Hülle — wird nicht über `test.js`, sondern end-to-end per Playwright/Chromium verifiziert (u. a. reale und synthetische Testprogramme, Pixel-/`getComputedStyle()`-Vergleiche vor/nach einer Aktion, sowie algebraische Node-Kontrollrechnungen für Kamera-/Zoom-Formeln).

**Neuere, config-/dialektgetriebene Mechanismen (GFIL/SCM, BR=/Biesse, E/FORMAT4, `CORNERRADIUS=<Marker>`, siehe 9.1c–9.1e/5.12b):** Diese wurden außerhalb von `test.js` über synthetische Kunstprogramme sowie — soweit verfügbar — reale, von Sebastian gelieferte Beispielprogramme verifiziert (u. a. Vergleich der resultierenden Bahnpunkte gegen zuvor bestätigte Referenzläufe, Falsch-Positiv-Checks für „E" ohne folgende Zahl, Vorrang-/Isolationstests zwischen mehreren gleichzeitig konfigurierten Markern), jedoch ohne eigene, in `test.js` fest hinterlegte Assertionen — die oben genannte Zahl von 281 grünen Assertionen bezieht sich ausschließlich auf den unveränderten Bestand von `test.js` und wächst mit diesen neueren Mechanismen nicht automatisch mit.

**Build-Integritätsprüfung vor jeder Auslieferung:** Das gebaute `CNC_Simulation.html` (Konkatenation aus `template_top.html` + den vier in `<script>`-Tags gewickelten Quelldateien `parser.js`/`swap.js`/`encoding.js`/`app.js`, siehe Abschnitt 2) enthält exakt vier `<script>`-Tags und null Debug-Marker; zusätzlich läuft `node --check` einzeln über jede der vier Quelldateien, um Syntaxfehler vor der Auslieferung auszuschließen. Zusätzlich geprüft: `<meta charset="UTF-8">` steht exakt an Byte-Position 0 der ausgelieferten Datei (siehe Abschnitt 2).
