# Benutzer-Handbuch (User Manual)

Willkommen im **TORUS TERMINAL**.
Dieses System ist ein hybrides Interface für Interaktionen mit der Komplex-Kybernetik.

## Verfügbare Befehle (`/help`)

Alle Nutzer haben Zugriff auf folgende Standard-Funktionen:

| Befehl    | Funktion      | Beschreibung                                                       |
| :-------- | :------------ | :----------------------------------------------------------------- |
| `/help`   | **Hilfe**     | Zeigt diese Liste. (Für Admins: Zeigt zusätzlich Admin-Tools).     |
| `/wallet` | **Guthaben**  | Zeigt deine aktuellen **Core Cycles (CC)** und Transaktionen.      |
| `/id`     | **Identität** | Zeigt deine interne System-ID (Soul ID) und Plattform-ID.          |
| `/delete` | **Löschen**   | **(Vorsicht)** Löscht deinen Account und Identität unwiderruflich. |

## Phase I: Initiation (Identity Manifestation)

Bevor du das System nutzen kannst, musst du deine Identität manifestieren:
1.  Gehe auf `/initiation`.
2.  Wähle zwei sichere Passwort-Komponenten (A und B).
3.  Das System generiert lokal (in deinem Browser) deinen **Sovereign Key** (Nostr Pubkey).
4.  Standardmäßig wird dein Username aus deinem Pubkey abgeleitet, kann aber später im **Profil** geändert werden.

> [!IMPORTANT]
> **Sicherheit:** Deine Passwörter verlassen niemals dein Gerät. Wir speichern nur deinen öffentlichen Schlüssel. Wenn du deine Passwörter verlierst, kann der Zugriff auf deine "Seele" nicht wiederhergestellt werden.

## Das Web-Terminal (The Nexus)

Im Terminal kannst du:
- Deinen aktuellen Status und dein Guthaben einsehen.
- Die Sprache des Systems ändern (8 Sprachen unterstützt).
- Dein **Profil** verwalten:
    - Öffentlichen Nutzernamen festlegen oder ändern.
    - Deinen **Telegram-Account verknüpfen** (siehe unten).
- Deine **Sicherheit** verwalten:
    - Anzeige deiner aktuellen Private/Public Keys (nach Re-Authentifizierung).
    - **Identitäts-Rotation** (Passwort-Änderung) zur Erhöhung der Entropie.
- Bei Bedarf deine Zustimmung zu aktualisierten Rechtstexten (AGB/Privacy) erneuern (**The Purgatory**).

## Telegram-Integration

Du kannst deine systemübergreifende Identität mit Telegram verknüpfen, um die KI direkt im Messenger zu nutzen:
1.  Öffne das **Profil** im Nexus.
2.  Klicke auf **"Telegram-Account verknüpfen"**.
3.  Du wirst zum Telegram-Bot weitergeleitet. Klicke dort auf **"Start"**.
4.  Das System erkennt den Token und verknüpft deine Telegram-ID sicher mit deinem Sovereign Key.

*Hinweis: Du kannst die Verknüpfung jederzeit im Profil wieder trennen (Unlink).*

## Die Ökonomie: Core Cycles (CC)

Jede Interaktion mit der KI verbraucht CC. 
- **Startguthaben**: 100 CC bei Initiation.
- **Täglicher Refill**: Dein Speicher füllt sich regelmäßig automatisch auf.
