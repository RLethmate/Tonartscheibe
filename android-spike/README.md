# Mikrofon-Spike: getUserMedia in Android-WebView (Capacitor)

Testet, ob der Kern des Improscheibe-Live-Features – Mikrofonzugriff über
`navigator.mediaDevices.getUserMedia` + `AudioContext`/`AnalyserNode` – innerhalb
einer Android-WebView (Capacitor-Hülle) funktioniert. Das ist die zentrale
technische Voraussetzung für einen Wrapper-basierten Android-Port.

## Was schon vorbereitet ist

- `www/index.html` – eine fokussierte Diagnoseseite (kein Akkord-Algorithmus,
  nur Mikrofon-Pipeline): zeigt Permission-Status, AudioContext-Status,
  Sample-Rate, RMS-Pegel und die lauteste FFT-Frequenz live an, plus ein
  Log-Fenster für Fehlermeldungen.
- `android/` – natives Android-Projekt (von `npx cap add android` generiert).
- `AndroidManifest.xml` – ergänzt um `RECORD_AUDIO`, `MODIFY_AUDIO_SETTINGS`
  und `<uses-feature android:name="android.hardware.microphone">`.

**Wichtiger Befund beim Einrichten:** Capacitors `BridgeWebChromeClient`
implementiert `onPermissionRequest` bereits inkl. Bridging von
`AUDIO_CAPTURE` auf die Android-Laufzeit-Berechtigung `RECORD_AUDIO`
(`node_modules/@capacitor/android/.../BridgeWebChromeClient.java:101-124`).
Eine eigene `MainActivity`/`WebChromeClient`-Anpassung ist also **nicht**
nötig – die Permission-Bridging-Logik, die ich ursprünglich als größtes
Risiko vermutet hatte, bringt das Framework von Haus aus mit.

## Voraussetzungen

- Android Studio installiert (✓ erledigt)
- Ein Android-Gerät mit aktiviertem USB-Debugging, per USB verbunden
  (oder ein AVD-Emulator – Mikrofon-Tests sind auf einem echten Gerät
  aussagekräftiger, da Emulatoren das Host-Mikrofon nur teilweise/anders
  durchreichen)

## Build & Test

Im Ordner `android-spike/`:

```bash
# 1) Web-Inhalt zum nativen Projekt synchronisieren
npx cap sync android

# 2a) Projekt in Android Studio öffnen (Gradle-Sync läuft dort zuverlässiger
#     als über die CLI, da Android Studio JDK/Gradle mitbringt)
npx cap open android
# -> in Android Studio: Run ▶ auf das verbundene Gerät

# 2b) Alternativ direkt über die CLI bauen & installieren (Gerät verbunden vorausgesetzt)
npx cap run android
```

## Was zu beobachten ist

Auf dem Gerät:

1. App öffnen → "Mikrofon starten" antippen.
2. **Erwartet (Erfolg):** Ein System-Dialog "Diese App Zugriff auf das
   Mikrofon erlauben?" erscheint → Erlauben → die Anzeige zeigt
   `permState: granted`, eine reale `sampleRate` (z. B. 48000 Hz oder
   44100 Hz), `ctxState: running`, und RMS/Peak-Frequenz reagieren live
   auf Geräusche/Töne.
3. **Mögliche Fehlerbilder und ihre Bedeutung:**
   - `getUserMedia ist NICHT verfügbar` → WebView-Version zu alt/eingeschränkt;
     müsste über das System-WebView-Update (Play Store) behoben werden.
   - Permission-Dialog erscheint nicht / `getUserMedia` schlägt mit
     `NotAllowedError` fehl → Bridging-Problem; dann wäre doch eine eigene
     `WebChromeClient`-Anpassung nötig.
   - `permState: denied` nach Antippen von "Verweigern" → normal, einfach
     in den App-Einstellungen die Berechtigung manuell erteilen und erneut
     testen (testet den Re-Request-Pfad).
   - RMS bleibt bei `0.0000` trotz `granted`/`running` → Audio-Pfad bricht
     irgendwo ab (z. B. falsches Routing); im Log-Fenster nach Fehlern
     schauen.

## Nach erfolgreichem Test

Wenn der Spike grün ist, ist der Weg frei, eine der bestehenden
Akkordanalyse-Prototyp-Seiten (`akkordanalyse-*.html` aus dem Hauptprojekt)
probeweise als `www/`-Inhalt einzusetzen und auf dem Gerät zu prüfen, ob
Performance/Latenz der FFT-Analyse in der WebView ausreichen.
