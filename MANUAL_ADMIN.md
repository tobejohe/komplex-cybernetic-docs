# Admin Handbuch (Operator Manual)

Dieses Dokument beschreibt die privilegierten Funktionen für Operatoren mit entsprechenden Autorisationen (Bits).

## Zugriff
Der Zugriff erfolgt dynamisch. Wenn Ihre Rolle autorisiert ist, erweitert der Befehl `/help` automatisch seine Ausgabe um die untenstehenden Tools.

## Admin Befehle

| Befehl    | Parameter      | Beschreibung                                                                                                                                                    |
| :-------- | :------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/mode`   | `[agent_slug]` | Wechselt den aktiven Ki-Agenten (z.B. `/mode shiva`, `/mode architect`).                                                                                        |
| `/ping`   | `[target]`     | System-Diagnose. <br> - `/ping` (System Info) <br> - `/ping db` (D1 Datenbank Latenz) <br> - `/ping ai` (Gemini API Test) <br> - `/ping cache` (KV Store Check) |
| `/debug`  | `[on\|off]`    | Schaltet den Debug-Modus. Wenn `ON`, werden interne Logs an jede Nachricht angehängt.                                                                           |
| `/debug`  | `whoami`       | Zeigt detaillierte User-Record Daten (Roles, Consent Flags).                                                                                                    |
| `/debug`  | `probe`        | Führt einen Schreib-Lese-Test auf der `auth_tokens` Tabelle durch (DB Integrity Check).                                                                         |
| `/config` | *tbd*          | *In Entwicklung: Anpassung von Systemeinstellungen.*                                                                                                            |

## Admin Console (Web UI)
Die grafische Administration erfolgt über das **Basement** (`basement.komplex.cc`).

### 1. Rights Management (Rechteverwaltung)
- **Pfad**: Seitenzufluss `Rights`.
- **Funktion**: Zuweisung von Capabilities (Bits) an Nutzer-IDs.
- **Scope**: Festlegung des Wirkungsbereichs (`*` für Global).

### 2. Log & Error Explorer
- **Pfad**: Seitenzufluss `Logs`.
- **AI Telemetry**: Einsicht in KI-Entscheidungsprozesse und Traces.
- **Audit Logs**: Chronologische Liste aller administrativen Änderungen am System.
- **Security Faults**: Überwachung verdächtiger Navigationsmuster (404s) zur Betrugsprävention und IDS-Auditierung.
- **Debug Logs**: Echtzeit-Fehlermeldungen aus dem System-Kernel (KV-basiert).

### 3. Communication Channels (Bot-Management)
- **Pfad**: Seitenzufluss `Channels`.
- **Funktion**: Dynamisches Management von Bots (Telegram, Discord, etc.).
- **Security**: Hinterlegung von Tokens erfolgt vollverschlüsselt via AES-GCM.

### 4. Agent Governance (Ki-Management)
- **Pfad**: Seitenzufluss `Agents`.
- **Zentrale**: Echtzeit-Überwachung der neuralen Knoten.
- **Traffic Lights**:
    - 🟢 **Grün**: Konfiguration OK, Betrieb sauber.
    - 🟡 **Gelb**: Betriebsstörungen in den letzten 24h (Log-Check).
    - 🔴 **Rot**: Harter Fehler (z.B. fehlender API-Endpunkt).
- **Editor**: Direkte Anpassung der LLM-Parameter und Endpunkte mit integrierter Validierung.

## Troubleshooting
...

### "Unknown Command"?
- Senden Sie `/help admin`.
- Wenn Sie keine Admin-Liste sehen, haben Sie keine Rechte.
- Checken Sie Ihre Berechtigungen in der DB: `SELECT capability_mask FROM authorizations WHERE user_id = '...'`.

### Silent Failure (Keine Antwort)
- Prüfen Sie `wrangler tail` Logs.
- Oft ein Formatierungsfehler (Markdown) in der Antwort.
