# Changelog

Alle wesentlichen Änderungen dieses Projekts werden hier dokumentiert. Die
Versionierung folgt [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-09-01

Erster öffentlicher Release.

### Enthalten

- HTTPMOD-Konfiguration für Login und Einzelgeräte-Polling
- explizites Mapping der MEATER-Cloud-Readings
- abgeleitete Cook-, Zeit- und Cloud-Zustände
- mehrzeiliges `stateFormat` mit aktiver Stale-Warnung
- neutraler Offline-Zustand ohne veraltete Temperaturanzeige
- einmalige Cook-Ereignisse über DOIF und Cook-ID
- vollständiger DbLog-Ausschluss

### Live verifiziert

- Start eines Cooks trotz API-State `Configured`
- aktive Verbindung, Verbindungsabbruch und Wiederverbindung
- genau eine Stale- und eine Abschlussmeldung pro Test-Cook
- normaler inaktiver Zustand ohne Fehlalarm

### Noch offen

- Live-Beobachtung der Ereignisse bei 5 °C Restdifferenz,
  `Ready For Resting` und `OVERCOOK!`

[0.1.0]: https://github.com/TeeVau/fhem-MeaterCloudAPI/releases/tag/v0.1.0
