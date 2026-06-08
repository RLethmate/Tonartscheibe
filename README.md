# Improscheibe – Harmony Wheel

Ein interaktives Lernwerkzeug für Harmonielehre: Die Tonartscheibe zeigt die 7 Töne einer Tonart im Kreis, macht Akkord-Spannungsbeziehungen sichtbar und hörbar, und erkennt Akkorde live per Mikrofon.

---

## Dateien auf einen Blick

### Haupt-App

| Datei | Beschreibung |
|---|---|
| `index.html` | Ältere Standalone-Version der Harmony Wheel Web-App (läuft direkt im Browser) |
| `indexUXstudy.html` | V47 – Harmony Wheel mit erweiterter UX (Studie-Variante) |
| `index_integrated_live.html` | V48 – integriert den FastAPI/crema-Live-Akkord-Erkennungs-Dienst (Vorgänger der Android-App) |

### Android-App (aktueller Stand)

| Verzeichnis | Beschreibung |
|---|---|
| `android-spike/` | **Das ist die aktuelle App.** Capacitor-Wrapper um `android-spike/www/index.html`. Enthält die vollständige Harmony Wheel App mit Live-Akkorderkennung via WebSocket→FastAPI/crema, Spotlight-Tutorial, Spannungs-Arpeggierung und Prognose-Funktion. Buildanleitung: siehe `android-spike/README.md` |

### Prototypen: Browser-seitige Akkordanalyse

Diese Dateien sind **Experimente mit rein browser-seitiger DSP** (kein ML-Backend). Sie haben keine direkte Verbindung zur Haupt-App.

| Datei | Beschreibung |
|---|---|
| `akkordanalyse_claude.html` | Chord Detector mit Mikrofon – Dur/Moll/Dom7/Moll7-Erkennung via FFT + Chroma im Browser |
| `akkordanalyse-chrromafeatures.html` | Pro Chord Detector: Chroma-Features + Noise Gate, für Pop-Akkorde |
| `akkordanalyse-jazzchrromafeatures.html` | Wie oben, aber mit Jazz-Akkord-Heuristiken (Triaden + Septimen) |
| `bassanalyse.html` | Bass-Root-Detektor: erkennt den Grundton im Bass per Mikrofon (Tuner-Stil) |
| `bassanalyse-Lookuptablle.html` | "Toto Analyzer" – Bassanalyse mit Lookup-Tabelle für robustere Grundton-Erkennung |

> **Hintergrund:** Diese Prototypen wurden gebaut um zu prüfen, ob clientseitige DSP für Akkorderkennung ausreicht. Ergebnis: Zu unzuverlässig für komplexere Akkorde → Entscheidung für ML-basiertes Backend (crema/FastAPI).

### Symbol-Grafiken

Die `.png`-Dateien im Projektstamm und in `android-spike/www/` sind Metaphern-Icons für die Akkord-Funktionen auf der Scheibe:

| Datei | Akkord-Stufe | Bedeutung |
|---|---|---|
| `haus.png` | I (Tonika) | Das Zuhause – Ruhepol |
| `berg_klein.png` | IV (Subdominante) | Kleiner Berg – Aufbruch |
| `berg_gross.png` | V (Dominante) | Großer Berg – maximale Spannung |
| `burg.png` | i (Moll-Tonika) | Die Burg – Festung und Heimat |
| `drache.png` | V/minor | Der Drache – magisch und gefährlich |
| `bruecke.png` | II / VI | Brücke |
| `alte_bruecke.png` | bVII | Alte Brücke |
| `bank.png` | III | Rast vor der Haustür |
| `lichtung.png` | iv/minor | Die Lichtung |
| `klippe.png` | VII (dim) | Die Klippe – sehr instabil |
| `nebelwald.png` | bIII | Der Nebelwald – geheimnisvoll |
| `sturm.png` | bVI | Sturm an der Küste – dramatisch |
| `sumpf.png` | ii° (dim) | Der Sumpf – bodenlos und trüb |

---

## Backend: Akkordanalyse-Dienst

Der Live-Akkorderkennungs-Dienst liegt in einem separaten Verzeichnis:

```
../chord-detector/backend/main.py   ← FastAPI WebSocket-Server (Port 8002)
```

**Stack:** Python 3.11 · FastAPI · crema (TensorFlow ML-Modell) · librosa · scikit-learn < 1.6

**Starten (lokal, Apple Silicon):**
```bash
cd ../chord-detector
lsof -ti:8002 | xargs kill -9 2>/dev/null
.venv-arm/bin/python3.11 -m uvicorn backend.main:app --host 0.0.0.0 --port 8002
```

---

## Architektur-Überblick

```
Android-App (Capacitor WebView)
    └── android-spike/www/index.html
            │
            │  WebSocket ws://SERVER:8002/ws
            ▼
    FastAPI-Dienst (chord-detector/backend/main.py)
            │
            │  Audio-Chunks (16kHz PCM float32)
            ▼
    crema ChordModel (TensorFlow)
            │
            ▼
    Chord + Key → JSON zurück an App → Scheibe dreht sich
```
