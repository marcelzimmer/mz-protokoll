# MZ-Protokoll – Entwicklerdokumentation

**Autor:** Marcel Zimmer<br>
**Web:** [www.marcelzimmer.de](https://www.marcelzimmer.de)<br>
**X:** [@marcelzimmer](https://x.com/marcelzimmer)<br>
**GitHub:** [@marcelzimmer](https://github.com/marcelzimmer)<br>
**Version:** 1.0.1<br>
**Sprache:** Rust<br>
**Lizenz:** MIT

---

## Inhaltsverzeichnis

1. [Überblick](#überblick)
2. [Abhängigkeiten](#abhängigkeiten)
3. [Projektstruktur](#projektstruktur)
4. [Datenmodell](#datenmodell)
5. [Architektur und Programmfluss](#architektur-und-programmfluss)
6. [UI-Schicht](#ui-schicht)
7. [Schriftarten-Laden](#schriftarten-laden)
8. [Markdown-Export und -Import](#markdown-export-und--import)
9. [PDF-Export](#pdf-export)
10. [Theme-System](#theme-system)
11. [Datei-Dialoge und Thread-Kommunikation](#datei-dialoge-und-thread-kommunikation)
12. [Tastenkombinationen](#tastenkombinationen)
13. [Erweiterungsmöglichkeiten](#erweiterungsmöglichkeiten)
14. [Build und Installation](#build-und-installation)
15. [Lizenz](#lizenz)

---

## Überblick

MZ-Protokoll ist eine plattformübergreifende Desktop-App zum Erstellen und Exportieren von Meeting-Protokollen (Markdown & PDF). Primäres Entwicklungs- und Zielsystem ist **Omarchy Linux** — eine Arch-Linux-Distribution — inklusive nativer Unterstützung des aktuellen Omarchy-Farbschemas über das eingebaute `Omarchy`-Theme. Die Anwendung läuft selbstverständlich ebenso unter **anderen Linux-Distributionen, macOS und Windows**. Die Oberfläche wird mit **egui/eframe** (Rust-GUI-Framework) gerendert; als Ausgabeformat stehen **Markdown** (maschinenlesbar, versionierbar) und **PDF** (druckfertig, mit Seitenzahlen und Linkverzeichnis) zur Verfügung.

Die gesamte Anwendungslogik befindet sich in einer einzigen Quelldatei: `src/main.rs`.

---

## Abhängigkeiten

| Crate    | Version | Verwendungszweck                                          |
|----------|---------|-----------------------------------------------------------|
| `eframe` | 0.31    | Anwendungsrahmen und Ereignisschleife (egui-Backend)      |
| `egui`   | -       | Immediate-Mode-GUI (Teil von eframe)                      |
| `chrono` | 0.4     | Aktuelles Datum, Wochentag, Zeitstempel                   |
| `rfd`    | 0.15    | Datei-Öffnen/Speichern-Dialoge (plattformnativ)           |
| `genpdf` | 0.2     | PDF-Dokument-Generierung                                  |

---

## Projektstruktur

```
mz-protokoll/
├── src/
│   └── main.rs           – gesamte Anwendungslogik (Datenmodell, UI, Export)
├── assets/
│   ├── icon.png          – App-Icon (Linux/generisch, aus icon.svg)
│   ├── icon.svg          – App-Icon (Quelle)
│   ├── icon.ico          – App-Icon für Windows-Binary
│   ├── icon.icns         – App-Icon für macOS-Bundle
│   ├── icon_macos.png    – gepaddetes Icon für das egui-Fenster auf macOS
│   └── Info.plist        – macOS-Bundle-Metadaten
├── .github/workflows/
│   └── release.yml       – CI/CD: Release-Builds für Linux, macOS, Windows
├── build.rs              – bettet icon.ico unter Windows in die .exe ein
├── Cargo.toml            – Paketdefinition und Abhängigkeiten
├── Cargo.lock            – reproduzierbare Builds
├── install.sh            – Installations-Skript (Linux, legt .desktop-Datei an)
├── PKGBUILD              – Arch-Linux-Paketdefinition
├── LICENSE               – MIT-Lizenz
└── README.md             – diese Datei
```

---

## Datenmodell

### `ProtokollApp` (Hauptzustand)

Zentrale Struct, die den vollständigen Anwendungszustand hält. Sie implementiert
`eframe::App` und wird von der egui-Ereignisschleife verwaltet.

**Protokoll-Kopfdaten:**

| Feld            | Typ                | Bedeutung                                        |
|-----------------|--------------------|--------------------------------------------------|
| `projekt`       | `String`           | Optionaler Projektname über dem Titel            |
| `titel`         | `String`           | Meeting-Titel (Hauptüberschrift)                 |
| `datum_text`    | `String`           | Datum als Freitext (z. B. „Montag, 05.02.2026") |
| `ort`           | `String`           | Veranstaltungsort                                |
| `protokollant`  | `Person`           | Protokollführer (Pflichtfeld)                    |
| `teilnehmer`    | `Vec<Person>`      | Liste der Meetingteilnehmer                      |
| `zur_kenntnis`  | `Vec<Person>`      | Personen, die das Protokoll erhalten             |
| `ueber_meeting` | `String`           | Freitext-Beschreibung des Meetings               |
| `ist_entwurf`   | `bool`             | Status: Entwurf                                  |
| `ist_freigegeben` | `bool`           | Status: Freigegeben                              |
| `sicherheit`    | `Sicherheit`       | Klassifizierungsstufe                            |
| `eintraege`     | `Vec<Eintrag>`     | Alle Tabelleneinträge                            |

### `Person`

```rust
struct Person {
    name: String,           // vollständiger Name
    kuerzel: String,        // Kürzel für TODO-Einträge (z. B. „MZ")
    kuerzel_manuell: bool,  // verhindert automatische Kürzel-Ableitung
}
```

Das Kürzel wird automatisch aus den Anfangsbuchstaben des Namens gebildet
(`Person::auto_kuerzel`), sofern `kuerzel_manuell = false`.

### `Eintrag`

```rust
struct Eintrag {
    punkt: String,     // Tagesordnungspunkt (leer bei Art::Todo)
    art: Art,          // Typ des Eintrags
    notiz: String,     // Freitext, Markdown-Links erlaubt
    kuemmerer: String, // Kürzel der verantwortlichen Person (nur Todo)
    bis: String,       // Fälligkeitsdatum TT.MM.JJJJ (nur Todo)
}
```

### `Art` (Eintragstyp)

| Variante      | Farbe      | Felder aktiv      |
|---------------|------------|-------------------|
| `Leer`        | Grau       | -                 |
| `Abgebrochen` | Rot        | Punkt, Notiz      |
| `Agenda`      | Lila       | Punkt, Notiz      |
| `Entscheidung`| Blau       | Punkt, Notiz      |
| `Fertig`      | Grün       | Punkt, Notiz      |
| `Idee`        | Gelb       | Punkt, Notiz      |
| `Info`        | Grau       | Punkt, Notiz      |
| `Todo`        | Orange     | Notiz, Kümmerer, Bis |

Bei `Art::Todo` wird der Punkt-Text automatisch geleert und die Felder
„Kümmerer" und „Bis" werden editierbar.

### `Sicherheit` (Klassifizierungsstufe)

`Oeffentlich` → `Intern` → `Vertraulich` → `StrengVertraulich`

### `DialogErgebnis`

Kommunikationstyp zwischen Datei-Dialog-Threads und dem Haupt-Thread:

```rust
enum DialogErgebnis {
    Laden(PathBuf, String),   // Pfad + Dateiinhalt
    Speichern(PathBuf),       // gewählter Speicherpfad
    PdfExport(PathBuf),       // gewählter PDF-Speicherpfad
}
```

---

## Architektur und Programmfluss

MZ-Protokoll folgt dem **Immediate-Mode-GUI-Muster** von egui:

```
┌────────────────────────────────────────┐
│  eframe-Ereignisschleife               │
│  (läuft ~60 Hz oder bei Ereignis)      │
│                                        │
│  ProtokollApp::update()                │
│  ┌──────────────────────────────────┐  │
│  │ 1. Tastenkombinationen prüfen    │  │
│  │ 2. Dialog-Ergebnisse verarbeiten │  │
│  │ 3. Theme anwenden                │  │
│  │ 4. UI rendern (deklarativ)       │  │
│  │ 5. Dialoge anzeigen              │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
         │ Nutzeraktion (Klick/Eingabe)
         ▼
   Zustandsänderung in ProtokollApp
         │
         ▼
   nächster Frame → neu rendern
```

### Datei-Dialoge (Thread-Kommunikation)

Da native Datei-Dialoge (`rfd`) den UI-Thread blockieren würden, werden sie in
separaten Threads ausgeführt. Ergebnisse werden über einen `mpsc`-Kanal
zurückgegeben und im nächsten `update()`-Aufruf ausgewertet:

```
Haupt-Thread               Dialog-Thread
     │                          │
     │── mpsc::channel() ──────►│
     │                          │
     │   (UI läuft weiter)      │── rfd::FileDialog::new()...
     │                          │        (blockiert hier)
     │◄── tx.send(Ergebnis) ────│
     │                          │
     │── dialog_rx.try_recv() ──┤
     │   Ergebnis verarbeiten   │
```

---

## UI-Schicht

### Aufbau der Oberfläche

```
┌─────────────────────────────────────────────────────┐
│ [Projektname]                            [☰]        │
│ [Titel des Meetings]                                │
│ [Datum]  |  [Ort]                                   │
│ ─────────────────────────────────────────           │
│ ScrollArea:                                         │
│   Protokollführer  [Name]              [Kürzel]     │
│   ─────────────────────────────────────────         │
│   Teilnehmer [+]   [Name]              [Kürzel] [×] │
│   ─────────────────────────────────────────         │
│   Zur Kenntnis [+] [Name]              [Kürzel] [×] │
│   ─────────────────────────────────────────         │
│   Über dieses Meeting  [Freitext...]                │
│   ─────────────────────────────────────────         │
│   Status    [✓] Entwurf  [ ] Freigegeben            │
│   ─────────────────────────────────────────         │
│   Klassifizierung [ ] Öff  [✓] Int  [ ] Vertr ...   │
│   ─────────────────────────────────────────         │
│   Eintrags-Tabelle:                                 │
│   ┌──────────┬────────┬────────┬────────┬─────┬──┐  │
│   │ Punkt    │ Art    │ Notiz  │Kümmerer│ Bis │  │  │
│   ├──────────┼────────┼────────┼────────┼─────┼──┤  │
│   │ ...      │ TODO ▼ │ ...    │ MZ  ▼  │...  │▲▼×│ │
│   └──────────┴────────┴────────┴────────┴─────┴──┘  │
│   [+ Eintrag hinzufügen]                            │
└─────────────────────────────────────────────────────┘
```

### UI-Hilfsfunktionen

| Funktion                              | Beschreibung                                                   |
|---------------------------------------|----------------------------------------------------------------|
| `fette_schrift(groesse)`              | Erstellt eine `egui::FontId` für die „Bold"-Familie            |
| `personen_zeile(ui, person, ...)`     | Rendert eine Name+Kürzel-Zeile, gibt (gelöscht, Enter) zurück  |
| `abschnitts_beschriftung(ui, ...)`    | Linksbündige fette Überschrift mit fixer Mindestbreite         |
| `abschnitts_beschriftung_mit_plus(…)` | Wie oben, aber mit „+"-Button; gibt `true` bei Klick zurück    |

### Eintrags-Tabelle

Die Tabelle wird als `egui::Grid` mit 6 Spalten gerendert:
`Punkt | Art | Notiz | Kümmerer | Bis | Aktionen`

Besonderheit: Bei `Art::Todo` werden Punkt-Feld (inaktiv) und Kümmerer/Bis-Felder (aktiv)
gerendert. Bei anderen Typen ist es umgekehrt.

**Cursor-Navigation zwischen Notizfeldern:** Pfeiltasten `↑`/`↓` springen aus dem
obersten/untersten Zeilende eines Notizfeldes ins vorherige/nächste Notizfeld.
Die Implementierung speichert jedes Frame in `notiz_had_focus` den letzten Fokus-Index
und die Cursor-Position, um im nächsten Frame die Navigation auswerten zu können.

---

## Schriftarten-Laden

egui benötigt für die Anzeige von fettem Text eine separate Font-Family „Bold".
Die Anwendung liest Systemschriften zur Laufzeit – es werden keine Schriften eingebettet.

**Windows** (in dieser Reihenfolge):
- Arial, Segoe UI, Calibri, Tahoma (`C:\Windows\Fonts\`)

**macOS** (in dieser Reihenfolge):
- Arial, Verdana, Georgia, Trebuchet MS (`/System/Library/Fonts/Supplemental/`)

**Linux** (in dieser Reihenfolge):
- Liberation Sans (Arch, Fedora, Debian, Ubuntu)
- Noto Sans
- DejaVu Sans (Fallback)

Wird keine Schrift gefunden, verwendet egui seine eingebettete Fallback-Schrift
(ohne fette Variante).

---

## Markdown-Export und -Import

### Dateiformat

Das MZ-Protokoll-Markdown-Format ist ein strukturiertes, abschnittsbasiertes Markdown:

```markdown
**Projekt:** Projektname

# Titel des Meetings

**Datum:** Montag, 05.02.2026 | **Ort:** Berlin

---

## Protokollführer

Marcel Zimmer [MZ]

## Teilnehmer

- Anna Beispiel [AB]
- Bob Muster [BM]

## Zur Kenntnis

- Carol Test [CT]

## Über dieses Meeting

Kurzbeschreibung des Meetings.

## Status

- [x] Entwurf
- [ ] Freigegeben

## Klassifizierung

- [ ] Öffentlich
- [x] Intern
- [ ] Vertraulich
- [ ] Streng vertraulich

---

## Einträge

| Punkt | Art | Notiz | Kümmerer | Bis |
|-------|-----|-------|----------|-----|
| Beispielpunkt | INFO | Notiz zum Punkt | | |
| | TODO | Aufgabe erledigen | MZ | 31.12.2026 |

---

**Erstellt:** 05.02.2026 10:00 von Marcel Zimmer

**Geändert:** 05.02.2026 14:30 von Marcel Zimmer

*Erstellt mit MZ-Protokoll...*
```

### Parser (`markdown_parsen`)

Der Parser ist ein zeilenbasierter Zustandsautomat mit dem internen Enum `Section`.
Beim Einlesen einer `## Überschrift` wechselt der Zustand:

```
Header → Protokollfuehrer → Teilnehmer → ZurKenntnis →
UeberMeeting → Status → Sicherheit → Eintraege
```

**Wichtig:** `|`-Zeichen in Zellen werden escaped (`\|`) gespeichert.
Die Funktion `tabellenzeile_aufteilen` verarbeitet dies beim Einlesen korrekt.

### Serialisierer (`markdown_erstellen`)

Baut den Markdown-String durch `String::push_str`-Aufrufe auf. Zeilenumbrüche in
Notizfeldern werden als ` <br> ` codiert, damit die Markdown-Tabelle einzeilig bleibt.

---

## PDF-Export

### Zweiphasen-Rendering

genpdf kennt die Gesamtseitenzahl erst nach dem Rendern. Um dennoch „Seite X von Y"
in die Fußzeile schreiben zu können, wird der Inhalt zweimal gerendert:

```
Durchlauf 1 (In-Memory-Puffer):
  → genpdf::SimplePageDecorator zählt Seiten mit
  → Ergebnis: gesamtseiten: usize

Durchlauf 2 (echte Datei):
  → FusszeileDekorator verwendet gesamtseiten + Druckzeitstempel
  → schreibt Fußzeile auf jede Seite:
     links "MZ-Protokoll" | Mitte Druckdatum | rechts "Seite X von Y"
```

### `FusszeileDekorator`

Implementiert `genpdf::PageDecorator`. Die Fußzeile wird auf dem rohen Seitenbereich
platziert, bevor die Seitenränder gesetzt werden, damit sie im Randbereich liegt.
Enthält eine simulierte Trennlinie (Unterstriche in Grau) sowie drei Textelemente:
links „MZ-Protokoll" (fett), Mitte Druckdatum, rechts „Seite X von Y".

### `ZellenHintergrund<E>`

Da genpdf keine echte Tabellenformatierung mit Hintergrundfarben bietet, werden
TODO-Zeilen grau hinterlegt, indem sehr dichte horizontale Linien (0,15 mm Abstand,
Graustufe 220) gezeichnet werden. Weiße Zeilen verwenden Graustufe 255, um grauen
Überlauf der Vorgängerzeile abzudecken.

### Markdown-Links im PDF

Da genpdf keine Hyperlinks unterstützt, werden `[Text](URL)`-Links durch
`Text [N]` ersetzt und am Ende des Dokuments als nummiertes Linkverzeichnis gedruckt
(Funktion `markdown_links_extrahieren`).

### Schriftarten für PDF

Für den PDF-Export (`schrift_laden`) sucht die App nach Systemschriften –
es werden keine Schriften eingebettet oder mitgeliefert.

**Windows:** Arial, Verdana, Calibri, Segoe UI (`C:\Windows\Fonts\`)

**macOS:** Arial, Verdana, Georgia, Trebuchet MS (`/System/Library/Fonts/Supplemental/`)

**Linux:** Liberation Sans, Noto Sans (über `genpdf::fonts::from_files`),
Fallback: DejaVu Sans.

Wird keine Schrift gefunden, erscheint ein Fehlerdialog mit Installationshinweis.

---

## Theme-System

### Varianten

| Theme     | Beschreibung                                          |
|-----------|-------------------------------------------------------|
| `Hell`    | Helles egui-Standard-Theme                            |
| `Dunkel`  | Dunkles Theme, Hintergrund reines Schwarz             |
| `Omarchy` | Liest Farben aus `~/.config/omarchy/current/theme/colors.toml` |

### Omarchy-Integration

Die Funktion `omarchy_farben_laden` liest TOML-Zeilen der Form `key = "#rrggbb"` ein.
Verwendete Schlüssel:

| TOML-Schlüssel         | Verwendung in der App                            |
|------------------------|--------------------------------------------------|
| `background`           | Fensterhintergrund                               |
| `foreground`           | Eingabetext in Textfeldern, Primärtext           |
| `accent`               | Buttons, Selektion, Hyperlinks, Fokus-Strich     |
| `selection_background` | Hintergrund für Textauswahl [Fallback: accent]   |
| `selection_foreground` | Textfarbe für Textauswahl                        |

Abschnittsbezeichnungen, Trennlinien und Hover-Flächen werden aus einem WCAG-konformen Blend zwischen `foreground` und `background` abgeleitet. Der Basis-Modus [hell/dunkel] folgt automatisch der Luminanz des Hintergrunds.

Damit sieht die App bei jedem aktuellen und zukünftigen Omarchy-Theme sauber aus, ohne dass pro Theme eine eigene Farbdatei nötig ist.

Das `Omarchy`-Theme wird nur im Cycle angeboten, wenn die Konfigurationsdatei
gefunden wurde [`has_omarchy = true`].

---

## Datei-Dialoge und Thread-Kommunikation

Da `rfd::FileDialog` den Haupt-Thread blockieren würde, laufen alle Dialoge in
eigenen Threads. Die Kommunikation erfolgt über `std::sync::mpsc`:

```rust
let (sender, empfaenger) = mpsc::channel::<DialogErgebnis>();
self.dialog_rx = Some(empfaenger);
std::thread::spawn(move || {
    if let Some(pfad) = rfd::FileDialog::new()...pick_file() {
        let _ = sender.send(DialogErgebnis::Laden(pfad, inhalt));
    }
});
// Im nächsten update()-Aufruf:
if let Ok(ergebnis) = self.dialog_rx.try_recv() { ... }
```

Es kann immer nur ein Dialog gleichzeitig geöffnet sein (`dialog_rx` ist `Option`).

---

## Tastenkombinationen

| Kombination | Aktion                              |
|-------------|-------------------------------------|
| `Strg+N`    | Neues Protokoll (aktuelle Daten verwerfen) |
| `Strg+O`    | Datei öffnen (Markdown laden)       |
| `Strg+S`    | Speichern (Markdown)                |
| `Strg+P`    | PDF erzeugen (PDF-Export)           |
| `Strg+T`    | Theme wechseln                      |
| `Strg+Q`    | Beenden (mit Bestätigungsdialog)    |
| `Strg+I`    | Über-Dialog öffnen                  |
| `Strg+H`    | Hilfe-Website öffnen                |
| `↑`/`↓`     | Cursor zwischen Notizfeldern bewegen |

---

## Erweiterungsmöglichkeiten

### Neue Eintragsart hinzufügen

1. In `enum Art` eine neue Variante ergänzen.
2. In `Art::label()` einen Anzeigetext definieren.
3. In `Art::color()` eine Farbe zuweisen.
4. In `Art::all()` die Variante eintragen.
5. In `art_parsen()` den Text-Mapping-Eintrag ergänzen.
6. Ggf. in `pdf_inhalt_hinzufuegen` und im UI-Rendering behandeln.

### Neues exportiertes Feld hinzufügen

1. Feld in `ProtokollApp` als `String` oder passendem Typ hinzufügen.
2. In `markdown_erstellen` ausgeben.
3. In `markdown_parsen` einlesen (neuer `Section`-Zustand oder Header-Parsing).
4. In `pdf_inhalt_hinzufuegen` in die info_table-Zeile aufnehmen.
5. UI-Widget in `update()` ergänzen.

### Neue Schriftart unterstützen

In `new()` (für egui) und in `schrift_laden()` (für genpdf) den entsprechenden
Pfad in den jeweiligen Suchpfad-Arrays ergänzen.

### Omarchy-Farben erweitern

In `update()` im `Theme::Omarchy`-Zweig weitere `colors.get("key")`-Abfragen
und die entsprechenden `visuals`-Zuweisungen ergänzen.

---

## Build und Installation

### Voraussetzungen

- Rust (stable, getestet mit Edition 2021)
- **Linux (Arch/Omarchy):** Alles außer Rust ist auf Omarchy Linux bereits vorhanden. Für Minimal-Arch-Installationen: `base-devel`, `pkg-config`, `gtk3`, `openssl`, `libxkbcommon`.
- **Linux (Debian/Ubuntu):** `pkg-config`, `libssl-dev`, `libgtk-3-dev`, `libxcb-render0-dev`, `libxcb-shape0-dev`, `libxcb-xfixes0-dev`, `libxkbcommon-dev`.
- Für alle Linux-Distributionen zusätzlich: eine Systemschrift (Liberation Sans, Noto Sans oder DejaVu Sans) für den PDF-Export.
- **Windows:** [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) mit der Komponente „Desktop development with C++" (enthält MSVC-Compiler und Windows SDK)
- **macOS:** macOS 26 oder neuer, Xcode Command Line Tools (`xcode-select --install`); Systemschriften (Arial, Verdana etc.) sind standardmäßig vorhanden

### Debug-Build

**Linux / macOS:**
```bash
cargo build
./target/debug/mz-protokoll   # Binary-Name bleibt mz-protokoll (Cargo.toml)
```

**Windows:**
```cmd
cargo build
target\debug\mz-protokoll.exe
```

### Release-Build

**Linux / macOS:**
```bash
cargo build --release
./target/release/mz-protokoll
```

**Windows:**
```cmd
cargo build --release
target\release\mz-protokoll.exe
```

### Linux-Installation (benutzerbezogen)

```bash
chmod +x install.sh
./install.sh
```

Das Skript kopiert die Binary nach `~/.local/bin/mz-protokoll`, das Icon
nach `~/.local/share/icons/hicolor/256x256/apps/` und die `.desktop`-Datei
nach `~/.local/share/applications/`. Stelle sicher, dass `~/.local/bin` in
deinem `PATH` enthalten ist.

### macOS – .app-Bundle erstellen

Das `.app`-Bundle wird automatisch vom CI-Workflow `.github/workflows/release.yml`
beim Tag-Release erzeugt (Ziel: `aarch64-apple-darwin`). Der Workflow erstellt
`MZ-Protokoll.app` mit Binary, `Info.plist` und `icon.icns` und hängt sie als
`mz-protokoll-macos-aarch64.zip` an das GitHub-Release an.

Lokaler Bundle-Aufbau (falls manuell benötigt):

```bash
cargo build --release --target aarch64-apple-darwin
mkdir -p MZ-Protokoll.app/Contents/{MacOS,Resources}
cp target/aarch64-apple-darwin/release/mz-protokoll MZ-Protokoll.app/Contents/MacOS/
cp assets/Info.plist MZ-Protokoll.app/Contents/
cp assets/icon.icns MZ-Protokoll.app/Contents/Resources/
open MZ-Protokoll.app
```

### macOS – Gatekeeper-Hinweis

Da die App nicht mit einem Apple-Entwicklerzertifikat signiert ist, blockiert macOS
beim ersten Start die Ausführung. So lässt sich die App trotzdem starten:

```bash
xattr -cr /pfad/zu/MZ-Protokoll.app
```

---

## Lizenz

### MIT-Lizenz – Nutzung, Rechte und Pflichten

MZ-Protokoll steht unter der **MIT-Lizenz**. Der vollständige Lizenztext befindet sich in der Datei `LICENSE`.

**Was die MIT-Lizenz erlaubt:**

- **Nutzung** – Die Software darf frei genutzt werden, auch **kommerziell**.
- **Modifikation** – Der Quellcode darf verändert und angepasst werden.
- **Weitergabe** – Die Software darf weitergegeben und weiterverteilt werden, auch in veränderter Form.

**Was ausdrücklich ausgeschlossen ist:**

- **Haftung ist ausgeschlossen.** Der Autor haftet nicht für Schäden, Datenverlust, Fehlfunktionen oder sonstige Folgen, die durch die Nutzung dieser Software entstehen – weder direkt noch indirekt.
- **Keine Gewährleistung.** Die Software wird **„wie sie ist"** bereitgestellt, ohne jegliche Garantie auf Funktionsfähigkeit, Eignung für einen bestimmten Zweck oder Fehlerfreiheit.
- **Keine Support-Pflicht.** Es besteht keinerlei Verpflichtung, Fehler zu beheben, Fragen zu beantworten, Updates bereitzustellen oder irgendeine Form von Wartung oder Support zu leisten.

**Pflichten bei der Nutzung:**

- **Eigenverantwortliche Code-Prüfung:** Wer diese Software in einem produktiven Umfeld einsetzt, ist **selbst dafür verantwortlich**, den Quellcode zu lesen, zu verstehen, zu prüfen und ggf. anzupassen. Eine Nutzung ohne vorherige Prüfung erfolgt auf eigenes Risiko.

---

*Diese README wurde am 17.04.2026 erstellt und zuletzt am 22.04.2026 aktualisiert [Version 1.0.1].*
