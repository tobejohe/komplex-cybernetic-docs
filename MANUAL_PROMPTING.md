# Prompt Engineering Manual ("GitOps Workflow")

Dieses Dokument beschreibt den Prozess zum Erstellen, Bearbeiten und Aktivieren von KI-Prompts im Komplex-System.
Wir verwenden einen **"Code-First" Ansatz** (GitOps), bei dem die Wahrheit im Dateisystem liegt und in die Datenbank synchronisiert wird.

---

## 🏗️ 1. Wo erstelle und finde ich Prompts?

Die Prompts sind **keine** mystischen Datenbank-Einträge mehr, sondern einfache Markdown-Dateien.

*   **Pfad:** `src/prompts/`
*   **Format:** `.md` (Markdown)

### Bestehende Prompts
Öffne `src/prompts/` in deinem Editor (VS Code). Du siehst Dateien wie:
*   `core_identity.md` (Die Basis-Persönlichkeit)
*   `mode_cortex.md` (Die Anweisungen für den Cortex-Modus)

### Neuen Prompt erstellen
1.  Erstelle eine neue Datei in `src/prompts/`.
2.  Benenne sie logisch, z.B. `skill_coding.md` oder `persona_pirate.md`.
3.  Der Dateiname (ohne `.md`) ist später der **"Handle"** (Unique ID) in der Datenbank.

**Beispiel:**
Datei: `src/prompts/fun_mode.md`
Handle: `fun_mode`

---

## 🔄 2. Wie bringe ich Änderungen in die Datenbank?

Wenn du eine Datei lokal bearbeitet hast, weiß die KI noch nichts davon. Du musst den **Sync** auslösen.

Führe folgenden Befehl im Terminal aus:

```bash
npm run prompts:push
```

### 2b. Andersrum: Datenbank zu lokal (Pull)
Falls du (oder ein anderer Admin) direkt in der Datenbank Änderungen gemacht hast und diese lokal haben willst:

```bash
node scripts/pull_prompts.js
```
Dies überschreibt deine lokalen Dateien mit dem Stand aus der Datenbank.

**Was passiert im Hintergrund?** (`scripts/sync_prompts.js`)
1.  Das Skript scannt alle `.md` Dateien in `src/prompts/`.
2.  Es vergleicht sie mit der Datenbank (`torus-db`).
3.  Es führt ein **Upsert** durch:
    *   **Neu:** Legt den Eintrag in `prompt_library` an (Version 1).
    *   **Geändert:** Aktualisiert den Inhalt, erhöht die **Version** (+1) und aktualisiert den Timestamp.
    *   **Archiv:** Speichert die *alte* Version automatisch in der Tabelle `prompt_versions` (Backup).

✅ **Ergebnis:** Die KI nutzt ab sofort die neue Text-Version.

---

## 🔗 3. Wie aktiviere ich einen Prompt für einen Bot?

Ein Prompt in der Datenbank (`prompt_library`) ist wie ein Buch in der Bibliothek – es wird erst genutzt, wenn jemand es "ausleiht".
Ein **Agent** (z.B. der Bot im "Cortex" Modus) besitzt einen **Stack** von Prompts.

### Schritt 1: Stack prüfen
Der Stack ist in der Tabelle `agents` in der Spalte `prompt_stack` definiert (als JSON Array).
Beispiel für den Agent `cortex`:
```json
["core_identity", "mode_cortex"]
```
Das bedeutet: "Lade erst die Identität, dann die Modus-Anweisungen."

### Schritt 2: Stack bearbeiten
Um deinen neuen Prompt (z.B. `fun_mode`) zu aktivieren, musst du ihn dem Stack hinzufügen.

**Variante A: Via SQL (aktuell)**
Führe einen Befehl aus, um den Stack zu updaten:

```bash
npx wrangler d1 execute torus-db --command "UPDATE agents SET prompt_stack = '[\"core_identity\", \"mode_cortex\", \"fun_mode\"]' WHERE slug = 'cortex'" --remote
```

**Variante B: Via Admin-Tool (Zukunft)**
Später bauen wir dafür ein UI oder einen Slash-Command (z.B. `/admin set_stack cortex core_identity,fun_mode`).

---

## 🛡️ Best Practices & Regeln

1.  **Modularität**: Mache keine riesigen Monolithen.
    *   Gut: `core_identity.md` + `skill_translate.md` + `tone_friendly.md`
    *   Schlecht: `ein_riesiger_prompt_fuer_alles.md`
2.  **Markdown nutzen**: Nutze **Fettgedrucktes**, Listen und Code-Blöcke. Die KI liest Struktur besser als Fließtext.
3.  **Keine Secrets**: Schreibe NIEMALS API-Keys oder Passwörter in die Prompts.
4.  **Service Consistency (Anti-Hallucination)**:
    - **Verbot:** Erfinde NIEMALS Ressourcen (Bücher, Events, Orte).
    - **Gebot:** Nutze IMMER ein Tool (`fetch_database_item`), um Daten zu holen.
    - **Fallback:** Wenn die DB leer ist, nutze prozedurale Generierung (z.B. `shadow_events`) statt Lügen.
5.  **Context Awareness**:
    - Beachte injizierte Variablen wie `USER_VIBE_MODE`.
    - Wenn `USER_VIBE_MODE == 'CALIBRATION_NEEDED'`, hat die Datenerhebung Vorrang vor dem Chat.

---

## 🆘 Troubleshooting

**Frage:** "Ich habe gepusht, aber der Bot ändert sein Verhalten nicht."
**Lösung:**
1.  Hast du `npm run prompts:push` ausgeführt? (Check auf grüne Haken ✅)
2.  Ist der Prompt im `prompt_stack` des Agents? (Siehe Punkt 3)
3.  Gibt es Syntax-Fehler im JSON des Stacks?

**Frage:** "Ich habe meinen Prompt kaputt gemacht."
**Lösung:**

---

## 🧠 4. Architectural Patterns ("Kernel vs. Persona")

Wir unterscheiden zwei Arten von Prompts:

### A. Kernel Modules (Das "Betriebssystem")
Diese Dateien fangen meist mit `kernel_` an. Sie definieren grundlegende Verhaltensweisen, die für **alle** Bots gelten.
*   **`kernel_security`**: Auth & Identity Checks.
*   **`kernel_process`** (Thinking Protocol): Zwingt die KI zum "Nachdenken" (`<thinking>...</thinking>`), bevor sie antwortet.
*   **`kernel_tooling`**: Definition der verfügbaren Tools.

### B. Persona Modules (Die "Rolle")
Definieren den Charakter und die spezifische Aufgabe.
*   `persona_sybil_identity`: "Ich bin Sybil, die Seherin..."
*   `mode_cortex`: "Ich bin ein hilfreicher Assistent..."

### Implementation des Thinking Protocols
Um das Thinking Protocol zu aktivieren, muss `kernel_process` in den `prompt_stack` des Agents eingefügt werden – idealerweise **nach** der Security und **vor** der Persona.

**Stack-Beispiel:**
1.  `kernel_security` (Blockt Hacker)
2.  `kernel_process` (Startet Denkprozess) 🧠
3.  `persona_sybil_identity` (Gibt Charakter)

So wird sichergestellt, dass die KI erst sicher ist, dann nachdenkt, und dann erst "schauspielert".

