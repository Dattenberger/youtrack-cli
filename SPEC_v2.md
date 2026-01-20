# YouTrack CLI - Spezifikation

> **Package:** `@dattenberger/youtrack-cli`  
> **Version:** 1.0.0  
> **Node.js:** >= 24 LTS  
> **Framework:** [Oclif](https://oclif.io/)

---

## Inhaltsverzeichnis

1. [Übersicht](#1-übersicht)
2. [Installation & Setup](#2-installation--setup)
3. [Commands](#3-commands)
4. [Ordnerstruktur](#4-ordnerstruktur)
5. [Datei-Formate](#5-datei-formate)
6. [Sync-Logik](#6-sync-logik)
7. [Attachment-Handling](#7-attachment-handling)
8. [Content-Konvertierung](#8-content-konvertierung)
9. [Fehlerbehandlung](#9-fehlerbehandlung)
10. [YouTrack API](#10-youtrack-api)
11. [Technischer Stack](#11-technischer-stack)

---

## 1. Übersicht

Die YouTrack CLI ermöglicht das lokale Bearbeiten von YouTrack Knowledge Base Artikeln. Artikel werden als Markdown-Dateien heruntergeladen, können lokal editiert und wieder hochgeladen werden.

### Kernfunktionen

- **Authentifizierung** mit YouTrack Permanent Tokens (sicher gespeichert via Keytar)
- **Download** aller Artikel inkl. Attachments in eine strukturierte Ordnerhierarchie
- **Upload** von lokalen Änderungen zurück zu YouTrack
- **Status-Übersicht** über lokale/remote Änderungen und Konflikte
- **Bidirektionale Sync-Erkennung** mit Konflikt-Handling
- **Automatische Content-Konvertierung** für Inline-Bilder und Attachment-Referenzen

### Einschränkungen

- **Nur eine YouTrack-Instanz** wird unterstützt (keine Multi-Server-Konfiguration)
- **Comments werden nicht synchronisiert** - Fokus liegt auf Artikel-Content
- **Keine Echtzeit-Sync** - Manuelle Befehle erforderlich
- **Keine Merge-Funktion** - Bei Konflikten muss User entscheiden

---

## 2. Installation & Setup

### Installation

```bash
npm install -g @dattenberger/youtrack-cli
```

### Erste Schritte

```bash
# 1. Authentifizierung
youtrack auth login

# 2. Artikel herunterladen
youtrack download --output ./my-articles

# 3. Artikel bearbeiten (in beliebigem Editor)
# 4. Änderungen hochladen
youtrack upload
```

---

## 3. Commands

### 3.1 Auth Commands

#### `youtrack auth login`

Interaktive Authentifizierung mit YouTrack.

```bash
youtrack auth login
```

**Ablauf:**
1. Abfrage der YouTrack Base-URL (z.B. `https://example.youtrack.cloud`)
2. Abfrage des Permanent Tokens
3. Validierung durch API-Call (`/api/users/me`)
4. Speicherung in System-Keychain via [Keytar](https://www.npmjs.com/package/keytar)

**Flags:**
| Flag | Beschreibung |
|------|--------------|
| `--verbose` | Detaillierte Ausgabe |

---

#### `youtrack auth logout`

Entfernt gespeicherte Credentials.

```bash
youtrack auth logout
```

---

#### `youtrack auth status`

Zeigt aktuelle Verbindungsinformationen.

```bash
youtrack auth status
```

**Beispiel-Ausgabe:**
```
✓ Authenticated

  URL:   https://example.youtrack.cloud
  User:  john.doe
  Name:  John Doe
```

---

### 3.2 Sync Commands

#### `youtrack download`

Lädt alle Artikel herunter, auf die der User Zugriff hat.

```bash
youtrack download [--project <n>] [--output <path>] [--no-attachments] [--force]
```

**Flags:**
| Flag | Default | Beschreibung |
|------|---------|--------------|
| `--project <n>` | alle | Nur Artikel eines bestimmten Projekts (shortName) |
| `--output <path>` | `./youtrack-articles` | Zielverzeichnis |
| `--no-attachments` | `false` | Attachments nicht herunterladen |
| `--force` | `false` | Bestehende Dateien ohne Nachfrage überschreiben |
| `--verbose` | `false` | Detaillierte Ausgabe |

**Verhalten:**
- Erstellt Ordnerstruktur basierend auf Projekt → Artikel-Hierarchie
- Erstellt `.youtrack`-Marker-Datei im Root-Verzeichnis
- Generiert `.gitignore` im Root-Verzeichnis
- Paginiert automatisch durch alle Artikel (API-Limit: 42 pro Request)
- Erkennt bereits existierende Artikel und fragt vor Überschreiben (außer `--force`)
- **Gelöschte Remote-Artikel:** Lokale Ordner werden gelöscht
- **Umbenannte Remote-Artikel:** Lokale Ordner werden umbenannt
- **Content-Konvertierung:** Inline-Bild-Referenzen werden auf lokale Pfade angepasst

---

#### `youtrack upload`

Lädt lokale Änderungen zu YouTrack hoch.

```bash
youtrack upload [--dry-run] [--article <id>] [--include-new] [--force] [--mute-notifications]
```

**Flags:**
| Flag | Default | Beschreibung |
|------|---------|--------------|
| `--dry-run` | `false` | Zeigt was passieren würde, ohne Änderungen |
| `--article <id>` | alle | Nur einen bestimmten Artikel hochladen |
| `--include-new` | `false` | Auch neue lokale Artikel erstellen |
| `--force` | `false` | Konflikte überschreiben (remote gewinnt nicht) |
| `--mute-notifications` | `false` | Keine YouTrack-Benachrichtigungen auslösen |
| `--verbose` | `false` | Detaillierte Ausgabe |

**Verhalten bei Konflikten:**
- Ohne `--force`: Abbruch mit Fehlermeldung, User muss entscheiden
- Mit `--force`: Lokale Version überschreibt remote

**Nach erfolgreichem Upload:**
1. Artikel wird von API neu abgerufen (GET)
2. `.youtrack.json` wird mit aktuellen Remote-Daten aktualisiert
3. `sync.*` Felder werden auf aktuelle Werte gesetzt
4. Neue Attachments erhalten ihre IDs (Dateien werden umbenannt)

**Uploadbare Felder:**
- `summary` (Titel)
- `content` (Markdown-Inhalt)
- `hasStar`
- `visibility`
- `tags` (werden automatisch erstellt wenn nicht vorhanden)
- `parentArticle` (durch Ordnerverschiebung)
- Attachments (neue, gelöschte, umbenannte)

**Content-Konvertierung:** Lokale Attachment-Pfade werden auf YouTrack-Namen zurückkonvertiert.

---

#### `youtrack status`

Zeigt Übersicht über Änderungen.

```bash
youtrack status [--verbose]
```

**Beispiel-Ausgabe:**
```
YouTrack Articles Status
========================

Modified locally:
  M  NP/NP-A-1 Getting Started/article.md

Modified remotely (needs download):
  R  NP/NP-A-3 API Guide/

Conflicts (both modified):
  C  NP/NP-A-2 Setup/

New locally (not uploaded):
  +  NP/__new-1 Mein Draft/

Deleted remotely:
  D  NP/NP-A-8 Old Article/

Renamed:
  →  NP/NP-A-5 Old Name/ → NP/NP-A-5 New Name/

Moved:
  ↳  NP/NP-A-7 Child Article/ (new parent: NP-A-1)

Attachments:
  +A NP/NP-A-1 Getting Started/attachments/new-image.png
  -A NP/NP-A-1 Getting Started/attachments/237-3_old-image.png
  ~A NP/NP-A-2 Setup/attachments/237-4_new-name.png (was: old-name.png)
  #A NP/NP-A-2 Setup/attachments/237-5_modified.png (content changed)

Summary: 2 modified, 1 conflict, 1 new, 1 deleted, 1 renamed, 1 moved, 4 attachment changes
```

**Legende:**
- `M` = Modified locally
- `R` = Modified remotely
- `C` = Conflict (both modified)
- `+` = New local
- `D` = Deleted remotely
- `→` = Renamed
- `↳` = Moved (parent changed)
- `+A` = New attachment
- `-A` = Deleted attachment
- `~A` = Renamed attachment
- `#A` = Modified attachment (content changed)

---

### 3.3 Utility Commands

#### `youtrack open`

Öffnet einen Artikel im Standard-Browser.

```bash
youtrack open [article-id | .]
```

**Beispiele:**
```bash
youtrack open NP-A-1    # Öffnet spezifischen Artikel
youtrack open .         # Öffnet Artikel im aktuellen Ordner
youtrack open           # Alias für "youtrack open ."
```

**Technisch:**
- URL-Schema: `https://{baseUrl}/articles/{idReadable}`
- Verwendet [open](https://www.npmjs.com/package/open) Package für plattformübergreifende Browser-Öffnung

---

#### `youtrack create`

Erstellt einen neuen Artikel lokal (ohne Upload).

```bash
youtrack create <title> --project <shortName> [--parent <article-id>]
```

**Flags:**
| Flag | Required | Beschreibung |
|------|----------|--------------|
| `--project <shortName>` | Ja | Projekt-Kurzname (z.B. `NP`) |
| `--parent <article-id>` | Nein | Parent-Artikel für Unterartikel |

**Beispiele:**
```bash
youtrack create "Mein neuer Artikel" --project NP
youtrack create "Unterartikel" --project NP --parent NP-A-1
```

**Verhalten:**
- Erstellt Ordner mit temporärer ID: `NP/__new-1 Mein neuer Artikel/`
- Erstellt `article.md` mit leerem Content und Frontmatter
- Erstellt `.youtrack.json` mit `"id": null` (markiert als neu)
- Bei `youtrack upload --include-new` wird Artikel auf YouTrack erstellt

**Parent-Validierung (bei `--parent`):**
1. Sofortige API-Validierung: `GET /api/articles/{article-id}`
2. Bei 404: Fehler `"Parent article not found: {article-id}"`
3. Bei 403: Fehler `"No access to parent article: {article-id}"`
4. Nur bei Erfolg: Lokalen Ordner erstellen

---

#### `youtrack move`

Verschiebt einen Artikel zu einem anderen Parent.

```bash
youtrack move <article-id> --parent <article-id | null>
```

**Beispiele:**
```bash
youtrack move NP-A-5 --parent NP-A-1    # Macht NP-A-5 zum Child von NP-A-1
youtrack move NP-A-5 --parent null      # Macht NP-A-5 zum Root-Artikel
```

**Verhalten:**
- Verschiebt Ordner in der Dateisystem-Hierarchie
- Aktualisiert `.youtrack.json` mit neuem `parentArticle`
- Änderung wird bei `youtrack status` angezeigt
- Bei `youtrack upload` wird Parent auf YouTrack geändert

**Alternative:** Manuelles Verschieben des Ordners → wird automatisch erkannt.

---

## 4. Ordnerstruktur

### 4.1 Arbeitsverzeichnis-Erkennung

Die CLI erkennt das Arbeitsverzeichnis durch:

1. **Marker-Datei:** `.youtrack` im Root-Verzeichnis
2. **Aufwärtssuche:** Von aktuellem Ordner nach oben suchen bis `.youtrack` gefunden wird

**`.youtrack` Marker-Datei:**
```json
{
  "version": "1.0.0",
  "baseUrl": "https://example.youtrack.cloud",
  "createdAt": 1705620052796
}
```

### 4.2 Grundstruktur

```
./youtrack-articles/                          # Root (konfigurierbar via --output)
├── .youtrack                                 # Marker-Datei (identifiziert Root)
├── .gitignore                                # Auto-generiert
├── {PROJECT_SHORT_NAME}/                     # Pro Projekt ein Ordner
│   ├── {idReadable} {title}/                 # Artikel-Ordner
│   │   ├── article.md                        # Artikel-Content mit YAML-Frontmatter
│   │   ├── .youtrack.json                    # Metadaten & Sync-Informationen
│   │   ├── attachments/                      # Attachments (optional)
│   │   │   ├── {id}_{sanitized-name}.{ext}   # Attachment-Dateien mit ID-Prefix
│   │   │   └── ...
│   │   └── children/                         # Unterartikel (optional)
│   │       └── {idReadable} {title}/         # Rekursive Struktur
│   │           ├── article.md
│   │           ├── .youtrack.json
│   │           ├── attachments/
│   │           └── children/
│   └── ...
└── ...
```

### 4.3 Beispiel

```
./youtrack-articles/
├── .youtrack
├── .gitignore
├── NP/
│   ├── NP-A-1 Getting Started with YouTrack/
│   │   ├── article.md
│   │   ├── .youtrack.json
│   │   ├── attachments/
│   │   │   ├── 237-3_screenshot-dashboard.png
│   │   │   └── 237-4_workflow-diagramm.pdf
│   │   └── children/
│   │       ├── NP-A-2 Installation/
│   │       │   ├── article.md
│   │       │   ├── .youtrack.json
│   │       │   └── attachments/
│   │       └── NP-A-3 Erste Schritte/
│   │           ├── article.md
│   │           ├── .youtrack.json
│   │           └── children/
│   │               └── NP-A-4 Projekt erstellen/
│   │                   ├── article.md
│   │                   └── .youtrack.json
│   └── NP-A-5 API Dokumentation/
│       ├── article.md
│       └── .youtrack.json
└── DEMO/
    └── DEMO-A-1 Beispiel Artikel/
        ├── article.md
        └── .youtrack.json
```

### 4.4 Ordnernamen-Format

**Format:** `{idReadable} {sanitized-title}`

Der Ordnername besteht aus:
1. **idReadable** (z.B. `NP-A-1`)
2. **Leerzeichen** als Trenner
3. **Sanitized Title** - der Titel mit entfernten unerlaubten Zeichen

**Sanitization-Regeln für Titel:**

Folgende Zeichen werden entfernt (nicht erlaubt im Dateisystem):
- `<` `>` `:` `"` `/` `\` `|` `?` `*`
- Steuerzeichen (ASCII 0-31)
- Trailing/Leading Leerzeichen und Punkte

**Beispiele:**
| Summary | Ordnername |
|---------|------------|
| `Getting Started` | `NP-A-1 Getting Started` |
| `API: Übersicht & Doku` | `NP-A-2 API Übersicht & Doku` |
| `Test / Demo` | `NP-A-3 Test  Demo` |
| `Was ist "neu"?` | `NP-A-4 Was ist neu` |
| (leer oder null) | `NP-A-5 Untitled` |

**Beim Upload:** Wenn der lokale Titel unerlaubte Zeichen enthielt, werden diese auch im YouTrack-Summary entfernt.

### 4.5 Attachment-Dateinamen

Attachments werden mit ihrer **YouTrack-ID als Prefix** gespeichert.

**Format:** `{attachmentId}_{sanitized-name}.{ext}`

**Sanitization-Regeln:**
- Leerzeichen → `-` (Bindestrich)
- Ungültige Zeichen (`< > : " / \ | ? *`) → entfernen
- Mehrfache Bindestriche → einzelner Bindestrich
- Trailing/Leading Bindestriche → entfernen
- Extension bleibt erhalten

**Beispiele:**

| YouTrack Name | Lokaler Name |
|---------------|--------------|
| `Screenshot Test.png` | `237-3_Screenshot-Test.png` |
| `API: Übersicht (1).pdf` | `237-4_API-Uebersicht-(1).pdf` |
| `my  file   name.jpg` | `237-5_my-file-name.jpg` |

**Neue Attachments (lokal erstellt):**
Dateien ohne ID-Prefix werden als neue Attachments erkannt:
```
attachments/
├── 237-3_existing-image.png    # Existiert remote (hat ID)
└── new-screenshot.png          # Neu (keine ID) → wird hochgeladen
```

Nach Upload wird die Datei automatisch umbenannt:
```
new-screenshot.png → 238-1_new-screenshot.png
```

**YouTrack-Name wird automatisch abgeleitet:**
Der Name in YouTrack entspricht immer dem lokalen Namen ohne ID-Prefix:
```
Lokal:    237-3_Dashboard-Overview.png
YouTrack: Dashboard-Overview.png (automatisch synchronisiert)
```

### 4.6 Neue Artikel (lokal erstellt)

Lokal erstellte Artikel haben noch keine `idReadable` und verwenden temporäre IDs:

```
NP/
└── __new-1 Mein neuer Artikel/
    ├── article.md
    └── .youtrack.json     # "id": null
```

Nach `youtrack upload --include-new`:
- Artikel wird auf YouTrack erstellt
- Ordner wird umbenannt zu `NP-A-99 Mein neuer Artikel/`
- `.youtrack.json` erhält echte IDs

### 4.7 Rename-Erkennung

Wenn der `summary` eines Artikels geändert wird (lokal oder remote), ändert sich der erwartete Ordnername.

**Erkennung:** Die `idReadable` im Ordnernamen bleibt gleich, nur der Titel-Teil ändert sich.

**Lokale Änderung:**
1. User ändert `summary` in `article.md` Frontmatter
2. `youtrack status` zeigt: `→ NP/NP-A-1 Old Name/ → NP/NP-A-1 New Name/`
3. `youtrack upload` lädt Änderung hoch UND benennt Ordner um

**Remote Änderung:**
1. `youtrack status` zeigt: `→ NP/NP-A-1 Old Name/ → NP/NP-A-1 New Name/ (remote)`
2. `youtrack download` benennt Ordner um

**Manuelle Ordner-Umbenennung:** Wenn jemand nur den Ordner umbenennt (nicht den Summary im YAML):
- CLI erkennt Diskrepanz zwischen Ordnername und YAML-Summary
- `youtrack status` zeigt Warnung
- Beim Upload wird der YAML-Summary verwendet (Ordner wird korrigiert)

---

## 5. Datei-Formate

### 5.1 article.md

Enthält YAML-Frontmatter mit **änderbaren** Feldern und den Markdown-Content.

```markdown
---
summary: "Getting Started with YouTrack"
hasStar: true
visibility: null
tags:
  - name: "documentation"
  - name: "getting-started"
---

## Einleitung

Willkommen bei YouTrack! Dieser Artikel erklärt die ersten Schritte...

![Dashboard Screenshot](attachments/237-3_screenshot-dashboard.png)

## Installation

Lorem ipsum dolor sit amet...
```

**YAML-Felder (änderbar):**

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `summary` | `string` | Artikel-Titel |
| `hasStar` | `boolean` | Favorit-Markierung |
| `visibility` | `object \| null` | Sichtbarkeitseinstellungen |
| `tags` | `array` | Liste von Tags |

**Visibility-Objekt (optional):**
```yaml
visibility:
  permittedGroups:
    - id: "3-1"
      name: "Developers"
  permittedUsers:
    - id: "1-1"
      name: "john.doe"
```

**Tags:**
- Tags werden beim Upload automatisch erstellt, wenn sie nicht existieren
- Nur der `name` ist relevant für neue Tags

**Leerer Artikel:**
```markdown
---
summary: "Neuer Artikel"
hasStar: false
visibility: null
tags: []
---

```

### 5.2 .youtrack.json

Enthält **read-only** Metadaten und Sync-Informationen.

```json
{
  "id": "226-0",
  "idReadable": "NP-A-1",
  "project": {
    "id": "81-1",
    "shortName": "NP"
  },
  "reporter": {
    "id": "24-0",
    "name": "John Doe"
  },
  "created": 1593001508713,
  "updated": 1605620052796,
  "updatedBy": {
    "id": "24-0",
    "name": "John Doe"
  },
  "ordinal": 0,
  "hasChildren": true,
  "parentArticle": null,
  "attachments": [
    {
      "id": "237-3",
      "localName": "237-3_Screenshot-Test.png",
      "size": 64014,
      "mimeType": "image/png",
      "hash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
    }
  ],
  "sync": {
    "lastSyncedAt": 1705620052796,
    "contentHash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "summaryHash": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
    "remoteUpdated": 1605620052796
  }
}
```

**Felder:**

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `id` | `string \| null` | Datenbank-ID (`null` für neue Artikel) |
| `idReadable` | `string \| null` | Lesbare ID (z.B. `NP-A-1`) |
| `project` | `object` | Projekt-Informationen |
| `reporter` | `object` | Ersteller des Artikels |
| `created` | `number` | Erstellungszeitpunkt (Unix-Timestamp ms) |
| `updated` | `number` | Letzte Änderung (Unix-Timestamp ms) |
| `updatedBy` | `object` | Letzter Bearbeiter |
| `ordinal` | `number` | Position in der Hierarchie |
| `hasChildren` | `boolean` | Hat Unterartikel |
| `parentArticle` | `object \| null` | Parent-Artikel Referenz |
| `attachments` | `array` | Attachment-Metadaten (ID, localName, size, mimeType, hash) |
| `sync` | `object` | Sync-Metadaten |

**Sync-Objekt:**

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `lastSyncedAt` | `number` | Letzter Sync-Zeitpunkt |
| `contentHash` | `string` | SHA-256 Hash des Contents beim letzten Sync |
| `summaryHash` | `string` | SHA-256 Hash des Summary beim letzten Sync |
| `remoteUpdated` | `number` | Remote `updated` Timestamp beim letzten Sync |

### 5.3 .youtrack (Marker-Datei)

Identifiziert das Root-Verzeichnis des Arbeitsbereichs.

```json
{
  "version": "1.0.0",
  "baseUrl": "https://example.youtrack.cloud",
  "createdAt": 1705620052796
}
```

### 5.4 .gitignore (auto-generiert)

```gitignore
# YouTrack CLI
# Diese Datei wurde automatisch generiert

# Sync-Metadaten (optional auskommentieren wenn gewünscht)
# .youtrack.json

# Marker-Datei
# .youtrack

# Temporäre Dateien
*.tmp
.DS_Store
Thumbs.db
```

---

## 6. Sync-Logik

### 6.1 Änderungserkennung

**Lokale Änderung erkannt wenn:**
```
hash(aktueller content) ≠ sync.contentHash
ODER
hash(aktueller summary) ≠ sync.summaryHash
```

**Remote Änderung erkannt wenn:**
```
remote.updated > sync.remoteUpdated
```

**Konflikt erkannt wenn:**
```
lokale Änderung UND remote Änderung
```

### 6.2 Konflikt-Handling

Bei Konflikten (ohne `--force`):

```
Error: Conflict detected for NP-A-1

  Local changes:
    - summary changed
    - content modified

  Remote changes:
    - Updated by "Jane Doe" at 2024-01-15 14:30
  
  Options:
    1. Run "youtrack download --force" to discard local changes
    2. Run "youtrack upload --force" to overwrite remote
    3. Manually resolve and retry
```

**Struktur-Konflikte (Parent-Verschiebung):**
Werden identisch zu Content-Konflikten behandelt:
- Ohne `--force`: Abbruch mit Fehlermeldung
- Mit `--force` bei Upload: Lokale Struktur überschreibt remote
- Mit `--force` bei Download: Remote Struktur überschreibt lokal

### 6.3 Hash-Berechnung

**Artikel-Content:**
```
contentHash = SHA-256(content ohne Frontmatter, trimmed, normalized)
summaryHash = SHA-256(summary, trimmed)
```

**Whitespace-Normalisierung vor Hash:**
- Trailing Whitespace entfernen
- Konsistente Line-Endings (LF)
- Attachment-Pfade normalisieren (vor Hash-Berechnung auf YouTrack-Format konvertieren)

**Attachments:**
```
attachmentHash = SHA-256(file binary content)
```

Attachment-Hashes werden in `.youtrack.json` pro Attachment gespeichert.

### 6.4 Parent-Verschiebung erkennen

**Erkennung durch Ordnerstruktur:**

1. Beim `status` wird für jeden Artikel geprüft:
   - Aktueller Dateisystem-Pfad → erwarteter Parent
   - `.youtrack.json` → gespeicherter Parent
2. Wenn unterschiedlich → Verschiebung erkannt

**Beispiel:**
```
Vorher: NP/NP-A-1 Parent/children/NP-A-2 Child/
Nachher: NP/NP-A-2 Child/  (manuell verschoben)

→ parentArticle von NP-A-2 ändert sich von "NP-A-1" zu null
```

### 6.5 Gelöschte Remote-Artikel

**Erkennung:**
- Lokaler Artikel existiert mit gültiger `id` in `.youtrack.json`
- API-Abfrage für diese ID liefert 404

**Verhalten:**
- `youtrack status` zeigt: `D  NP/NP-A-8 Old Article/`
- `youtrack download` löscht den lokalen Ordner

---

## 7. Attachment-Handling

### 7.1 Attachment-Erkennung

Attachments werden über ihre **ID im Dateinamen** identifiziert.

**Format:** `{id}_{name}.{ext}`

**Neue Attachments erkennen:**
- Datei existiert in `attachments/`
- Dateiname hat **kein** ID-Prefix (kein `_` oder kein gültiges ID-Format vor `_`)
- → Neues Attachment

**Gelöschte Attachments erkennen:**
- Eintrag existiert in `.youtrack.json`
- Datei mit entsprechender ID existiert NICHT in `attachments/`
- → Gelöscht

**Umbenannte Attachments erkennen:**
- ID-Prefix im Dateinamen bleibt gleich
- Teil nach `_` hat sich geändert
- → Rename (wird automatisch bei Upload synchronisiert)

**Geänderte Attachments erkennen (Content):**
- ID-Prefix im Dateinamen gleich
- `SHA-256(Datei) ≠ hash` in `.youtrack.json`
- → Content geändert

### 7.2 Hash-Berechnung für Attachments

```
attachmentHash = SHA-256(file binary content)
```

Der Hash wird bei jedem Sync in `.youtrack.json` gespeichert und beim `status`-Command verglichen.

### 7.3 Upload-Verhalten

| Situation | Aktion |
|-----------|--------|
| Neue lokale Datei (kein ID-Prefix) | `POST /api/articles/{id}/attachments` (multipart), dann Datei lokal umbenennen mit neuer ID |
| Bestehende Datei, Hash gleich | Name-Update via `POST /api/articles/{id}/attachments/{attachmentId}` mit aktuellem Namen |
| Bestehende Datei, Hash unterschiedlich | `DELETE` + `POST` (neue ID), Datei lokal umbenennen |
| Datei in JSON aber nicht lokal | `DELETE /api/articles/{id}/attachments/{attachmentId}` |

**Automatische Namens-Synchronisation:**
Bei jedem Upload wird der YouTrack-Name automatisch auf den aktuellen lokalen Namen (ohne ID-Prefix) gesetzt:
```
Lokal:    237-3_Dashboard-Overview.png
          ↓ (automatisch)
YouTrack: Dashboard-Overview.png
```

### 7.4 Download-Verhalten

| Situation | Aktion |
|-----------|--------|
| Neues Remote-Attachment | Download, lokal speichern als `{id}_{sanitized-name}` |
| Gelöschtes Remote-Attachment | Lokale Datei löschen |
| Geändertes Remote-Attachment | Lokale Datei aktualisieren, Hash neu berechnen |

**Sanitization beim Download:**
```
YouTrack: "Screenshot Test.png" (id: 237-3)
          ↓ (Leerzeichen → Bindestrich)
Lokal:    237-3_Screenshot-Test.png
```

### 7.5 Upload-Flow mit automatischem Re-Download

Nach erfolgreichem Upload eines Artikels:
1. Artikel wird von API neu abgerufen (GET)
2. `.youtrack.json` wird mit aktuellen Remote-Daten überschrieben
3. Attachment-Hashes werden auf aktuelle lokale Werte gesetzt
4. `sync.lastSyncedAt` wird aktualisiert

→ Garantiert Konsistenz zwischen lokal und remote.

---

## 8. Content-Konvertierung

### 8.1 Inline-Bilder in YouTrack

YouTrack verwendet Markdown-Syntax für Inline-Bilder:
```markdown
![alt text](filename.png)
```

Die Referenz ist der **Dateiname**, nicht die Attachment-ID.

### 8.2 Download-Konvertierung

**YouTrack → Lokal:**
```markdown
# Vorher (YouTrack)
![Screenshot](Dashboard-Overview.png)

# Nachher (Lokal)
![Screenshot](attachments/237-3_Dashboard-Overview.png)
```

**Regex-Pattern:**
```
!\[([^\]]*)\]\(([^)]+)\)
```

**Konvertierungslogik:**
1. Finde alle Bild-Referenzen im Content
2. Prüfe ob Dateiname einem Attachment entspricht (via `localName` ohne ID-Prefix)
3. Wenn ja: Ersetze mit `attachments/{localName}` (mit ID-Prefix)
4. Externe URLs (http/https) nicht ändern

### 8.3 Upload-Konvertierung

**Lokal → YouTrack:**
```markdown
# Vorher (Lokal)
![Screenshot](attachments/237-3_Dashboard-Overview.png)

# Nachher (YouTrack)
![Screenshot](Dashboard-Overview.png)
```

**Konvertierungslogik:**
1. Finde alle Bild-Referenzen mit `attachments/` Prefix
2. Extrahiere ID aus Dateiname (Teil vor erstem `_`)
3. Entferne `attachments/` Prefix und ID-Prefix
4. Ergebnis: `{name}.{ext}` (Name nach dem `_`)

### 8.4 File-Links (nicht Bilder)

YouTrack unterstützt auch Links zu Attachments:
```markdown
# YouTrack
[Download PDF](document.pdf)

# Lokal
[Download PDF](attachments/237-6_document.pdf)
```

**Gleiche Konvertierungslogik wie bei Bildern.**

**Gleiche Konvertierungslogik wie bei Bildern.**

### 8.5 Edge Cases

**Externes Bild (URL):**
```markdown
![Logo](https://example.com/logo.png)
```
→ Keine Konvertierung

**Attachment nicht gefunden:**
```markdown
![Missing](attachments/nonexistent.png)
```
→ Warnung ausgeben, keine Konvertierung

**Neues Attachment ohne ID-Prefix:**
```markdown
![New](attachments/new-image.png)
```
→ Wird beim Upload hochgeladen, danach Referenz aktualisiert mit ID

**Relative Pfade außerhalb attachments/:**
```markdown
![Image](../other-folder/image.png)
```
→ Keine Konvertierung (nur `attachments/` Pfade werden verarbeitet)

---

## 9. Fehlerbehandlung

### 9.1 Retry-Logik

**Bei Netzwerkfehlern:**
- Automatisches Retry (3 Versuche)
- Exponential Backoff: 1s, 2s, 4s
- Nach 3 Fehlversuchen: Fehler ausgeben, fortfahren mit nächstem Artikel

**Kein Rate-Limiting:** API-Calls werden sequentiell ausgeführt.

### 9.2 Partielle Fehler

Bei partiellen Fehlern (z.B. 3 von 10 Artikeln fehlgeschlagen):

```
Download completed with errors:

✓ 7 articles downloaded successfully
✗ 3 articles failed:
  - NP-A-5: 403 Forbidden (no access)
  - NP-A-8: 500 Internal Server Error
  - NP-A-12: Network timeout

Run with --verbose for details.
```

**Verhalten:**
- Erfolgreiche Operationen werden gespeichert
- Fehlgeschlagene werden übersprungen
- Zusammenfassung am Ende
- Exit-Code ≠ 0 bei Fehlern

### 9.3 Validierungsfehler

**Ungültiges YAML:**
```
Error: Invalid YAML in NP/NP-A-1 Getting Started/article.md

  Line 3: Unexpected token
  
  Fix the YAML syntax and try again.
```

**Fehlende Pflichtfelder:**
```
Error: Missing required field 'summary' in NP/NP-A-1 Getting Started/article.md
```

---

## 10. YouTrack API

### 10.1 Base URL

```
https://{instance}.youtrack.cloud/api
```

Oder für Self-Hosted:
```
https://{domain}/youtrack/api
```

### 10.2 Authentifizierung

**Permanent Token** im Authorization Header:

```
Authorization: Bearer perm:am9obi5kb2U=.WW91VHJhY2s=.abc123...
```

### 10.3 Endpoints

**Hinweis zu Artikel-IDs:**
Bei allen Endpoints kann `{articleID}` entweder die lesbare ID (`NP-A-1`) oder die Datenbank-ID (`226-0`) sein. Beide Formate werden von der API akzeptiert.

#### Artikel abrufen

**Alle Artikel:**
```http
GET /api/articles?fields={fields}&$top=42&$skip=0
```

**Einzelner Artikel:**
```http
GET /api/articles/{articleID}?fields={fields}
```

**Fields-Parameter:**
```
id,idReadable,summary,content,created,updated,updatedBy(id,name),
hasStar,ordinal,hasChildren,parentArticle(id,idReadable),
project(id,shortName),reporter(id,name),visibility(permittedGroups(id,name),
permittedUsers(id,name)),tags(id,name),
attachments(id,name,size,mimeType,extension,url,created,updated)
```

#### Artikel aktualisieren

```http
POST /api/articles/{articleID}?fields={fields}&muteUpdateNotifications={boolean}

{
  "summary": "Neuer Titel",
  "content": "Neuer Inhalt...",
  "hasStar": true,
  "visibility": null,
  "tags": [{"name": "documentation"}]
}
```

#### Artikel erstellen

```http
POST /api/articles?fields={fields}&muteUpdateNotifications={boolean}

{
  "project": {"shortName": "NP"},
  "summary": "Neuer Artikel",
  "content": "Inhalt...",
  "parentArticle": {"id": "226-0"}
}
```

#### Artikel löschen

```http
DELETE /api/articles/{articleID}
```

#### Parent ändern

```http
POST /api/articles/{articleID}

{
  "parentArticle": {"id": "226-5"}
}
```

Oder für Root-Level:
```json
{
  "parentArticle": null
}
```

#### Attachments abrufen

```http
GET /api/articles/{articleID}/attachments?fields=id,name,size,mimeType,extension,url,created,updated
```

#### Attachment herunterladen

```http
GET /api/files/{attachmentID}?sign={signature}&updated={timestamp}
```

Die URL kommt aus dem `url`-Feld des Attachments.

#### Attachment hochladen

```http
POST /api/articles/{articleID}/attachments?fields={fields}&muteUpdateNotifications={boolean}
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="screenshot.png"
Content-Type: image/png

{binary data}
--boundary--
```

#### Attachment umbenennen

```http
POST /api/articles/{articleID}/attachments/{attachmentID}

{
  "name": "neuer-name.png"
}
```

#### Attachment löschen

```http
DELETE /api/articles/{articleID}/attachments/{attachmentID}
```

### 10.4 Pagination

Die API gibt maximal 42 Artikel pro Request zurück.

**Parameter:**
- `$top`: Anzahl (max 42)
- `$skip`: Offset

**Beispiel für vollständiges Laden:**
```
Request 1: GET /api/articles?$top=42&$skip=0
Request 2: GET /api/articles?$top=42&$skip=42
Request 3: GET /api/articles?$top=42&$skip=84
...
```

Wiederholen bis Response-Array weniger als 42 Einträge hat.

### 10.5 Fehler-Responses

```json
{
  "error": "Not Found",
  "error_description": "Entity with id XY-A-999 not found"
}
```

**HTTP Status Codes:**
- `200` - Erfolg
- `400` - Bad Request
- `401` - Unauthorized (Token ungültig)
- `403` - Forbidden (keine Berechtigung)
- `404` - Not Found
- `500` - Server Error

### 10.6 User-Info (für Auth-Validierung)

```http
GET /api/users/me?fields=id,login,name,email
```

---

## 11. Technischer Stack

### 11.1 Runtime & Framework

| Komponente | Technologie | Version |
|------------|-------------|---------|
| Runtime | Node.js | >= 24 LTS |
| CLI Framework | [Oclif](https://oclif.io/) | latest |
| Language | TypeScript | 5.x |

### 11.2 Dependencies (vorgeschlagen)

| Package | Zweck |
|---------|-------|
| `@oclif/core` | CLI Framework Core |
| `keytar` | Sichere Credential-Speicherung (OS Keychain) |
| `open` | Browser öffnen (plattformübergreifend) |
| `yaml` | YAML Parsing für Frontmatter |
| `gray-matter` | Markdown Frontmatter Parsing |
| `chalk` | Terminal-Farben |
| `ora` | Spinner für Ladeanimationen |
| `inquirer` | Interaktive Prompts |

### 11.3 Credential-Speicherung

**Keytar** speichert Credentials sicher im OS-spezifischen Keychain:

| OS | Speicherort |
|----|-------------|
| macOS | Keychain |
| Windows | Credential Vault |
| Linux | libsecret (GNOME Keyring / KWallet) |

**Service-Name:** `youtrack-cli`
**Account-Name:** `default` (nur eine Instanz unterstützt)

**Gespeicherte Daten:**
```json
{
  "baseUrl": "https://example.youtrack.cloud",
  "token": "perm:abc123..."
}
```

### 11.4 Config-Datei

Nicht-sensitive Konfiguration in `~/.config/youtrack-cli/config.json`:

```json
{
  "defaultOutput": "./youtrack-articles"
}
```

Token wird **nicht** in der Config gespeichert, sondern ausschließlich in Keytar.

---

## Anhang

### A. Berechtigungen

Die CLI benötigt folgende YouTrack-Berechtigungen:

| Berechtigung | Für |
|--------------|-----|
| Read Article | Download |
| Create Article | Neue Artikel erstellen |
| Update Article | Artikel aktualisieren |
| Add Attachment | Attachments hochladen |
| Delete Attachment | Attachments entfernen |
| Create Tag | Neue Tags erstellen |
| Apply Commands Silently | `--mute-notifications` Flag |

### B. Bekannte Einschränkungen

1. **Nur eine YouTrack-Instanz** - Keine Multi-Server-Unterstützung
2. **Comments werden nicht synchronisiert** - Fokus liegt auf Artikel-Content
3. **Keine Echtzeit-Sync** - Manuelle Befehle erforderlich
4. **Keine Merge-Funktion** - Bei Konflikten muss User entscheiden
5. **Ordinal nicht änderbar** - Reihenfolge wird von YouTrack verwaltet

### C. Glossar

| Begriff | Beschreibung |
|---------|--------------|
| `idReadable` | Menschenlesbare Artikel-ID (z.B. `NP-A-1`) |
| `id` | Datenbank-ID (z.B. `226-0`) |
| `shortName` | Projekt-Kurzname (z.B. `NP`) |
| `Permanent Token` | Langlebiger API-Token für Authentifizierung |
| `Marker-Datei` | `.youtrack` Datei die das Root-Verzeichnis identifiziert |
