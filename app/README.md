# Improscheibe – Android App (Capacitor)

Die Android-App ist ein Capacitor-Wrapper um `www/index.html`. Sie enthält die vollständige Harmony Wheel App mit Live-Akkorderkennung, Spotlight-Tutorial und Spannungs-Arpeggierung.

## Voraussetzungen

- Node.js + npm
- Android Studio (mit installiertem Android SDK)
- Android-Gerät mit aktiviertem USB-Debugging **oder** AVD-Emulator
- Laufender Akkordanalyse-Dienst (siehe `../chord-detector/backend/`)

## Build & Deploy

Im Ordner `app/`:

```bash
# Web-Inhalt zum nativen Projekt synchronisieren
npx cap copy android

# Projekt in Android Studio öffnen
npx cap open android
# → in Android Studio: Run ▶ auf das verbundene Gerät
```

## Struktur

```
app/
├── www/
│   ├── index.html        ← gesamte App-Logik (HTML/CSS/JS)
│   └── *.png             ← Metaphern-Icons für Akkord-Stufen
├── android/              ← nativer Android-Wrapper (Capacitor-generiert)
└── capacitor.config.json ← WebSocket-URL, App-ID etc.
```

## Backend-Verbindung

Die App verbindet sich per WebSocket mit dem FastAPI-Dienst. Die URL steht in `www/index.html`:

```js
const WS_URL = 'ws://192.168.2.100:8002/ws';  // lokale Entwicklung
```

Für Cloud-Deployment diesen Wert auf die Server-URL ändern (dann auch `wss://` verwenden).
