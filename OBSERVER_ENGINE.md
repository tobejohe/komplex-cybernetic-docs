# Die Observer Engine

**Version**: 2.0 (Kuratorische & Hybride Semantische Erweiterung)
**Rolle**: Asynchrones psychometrisches Profiling, Faktenextraktion & Vektorisierung

## 1. Überblick

Die Observer Engine ist ein "Ghost Worker", der ausschließlich im Hintergrund agiert. Ihr Zweck ist es, rohe, sensible Chat-Konversationen in einen hybriden Zustand zu destillieren: deterministische Fakten (harte Grenzen) für die SQL-Speicherung und abstrakte psychometrische Vektoren (Core Identity) für den Ähnlichkeitsabgleich. Diese duale Architektur verhindert den "semantischen Hitzetod" und gewährleistet die absolute Nachverfolgung von Konsens (Consent), ohne persönlich identifizierbare Informationen (PII) auf unbestimmte Zeit zu speichern.

## 2. Die 6 psychometrischen Dimensionen (Core Identity)

Die Engine extrahiert 6 spezifische Vektoren aus konsolidierten Interaktionsstapeln (Batches):

| Dimension     | Schlüssel     | Beschreibung                                                              |
| :------------ | :------------ | :------------------------------------------------------------------------ |
| **Elitism**   | `v_elitism`   | Statusstreben, Exklusivität vs. Egalitarismus.                            |
| **Bandwidth** | `v_bandwidth` | Kognitive Dichte, Überintellektualisierung vs. emotionale Einfachheit.    |
| **Hedonism**  | `v_hedonism`  | Sensorische Intensität, Exzess vs. Askese/Zurückhaltung.                  |
| **Entropy**   | `v_entropy`   | Sehnsucht nach Chaos/Grenzauflösung vs. Ordnung/Vorhersehbarkeit.         |
| **Friction**  | `v_friction`  | Konfliktbereitschaft, Härte (Masochismus/Sadismus) vs. Harmonie.          |
| **Vector**    | `v_vector`    | Grundlegender Antrieb, Kompensationsverhalten oder spirituelle Sehnsucht. |

## 3. Der Destillationsprozess (Lebenszyklus)

1. **Auslöser (Trigger)**: Der HTTP-Producer sendet ein `{ session_id }` Event an die `DISTILLATION_QUEUE`. Dies wird in verzögerten Konsolidierungszyklen (z. B. nächtlich) ausgeführt, um Aktualitäts-Bias (Recency Bias) zu vermeiden.
2. **Abruf (Retrieval)**: Der Observer ruft den un-destillierten Interaktionsstapel für diese Sitzung aus D1 ab.
3. **Analytische Transformation (Hybrider Output)**:
   - Das LLM (z. B. Gemini 2.5 Flash) erhält den Rohtext zusammen mit der *existierenden* Core Identity.
   - Es gibt ein strukturiertes JSON-Objekt aus, das strikt trennt zwischen:
     - **Harten Grenzen (Hard Boundaries)**: SQL-taugliche Tags (strikte Consent-Regeln, Dealbreaker).
     - **Psychometrischem Zustand**: Aktualisierte beschreibende Sätze für die 6 Dimensionen, die neue Erkenntnisse mit dem alten Kern verschmelzen.
4. **Speicherung (Der Duale Schreibvorgang)**:
   - **Deterministisch (D1)**: Harte Grenzen werden in der SQL-Datenbank aktualisiert, um als absoluter "Türsteher" (Bouncer) zu fungieren.
   - **Probabilistisch (Vektorisierung)**: Jeder der 6 beschreibenden Sätze wird an das Embedding-Modell gesendet. Die neuen 768-dimensionalen Float-Arrays **überschreiben** die vorherigen Arrays via UPSERT (Token-Effizienz).
5. **Amnesie-Protokoll**:
   - Die Quellnachrichten in D1 werden als `is_distilled = 1` markiert.
   - Der geplante Garbage Collector löscht diese rohen Nachrichten nach der Aufbewahrungsfrist (8 Wochen).

## 4. Nutzungsmuster

### Kuratorisches Matchmaking (Das Türsteher-Muster)
Um einen Nutzer mit einem Event oder einem anderen Nutzer abzugleichen, verzichtet das System auf reines Vektor-Matching zugunsten einer hybriden Abfolge:
1. **Der Türsteher (SQL)**: Absolute Ablehnung von Entitäten, die gegen Harte Grenzen (Consent/Sicherheit) verstoßen.
2. **Semantische Sortierung (Vektorisierung)**: Der verbleibende gültige Pool wird evaluiert, indem die **Kosinus-Ähnlichkeit (Cosine Similarity)** der 6 psychometrischen Dimensionen berechnet wird.
3. **Chaos-Injektion**: Eine bewusste Serendipitätsmarge (z. B. das Einmischen zufälliger Elemente, die den SQL-Filter passiert haben) wird den Endergebnissen hinzugefügt, um subkulturelle Friktion zu induzieren und algorithmische Echokammern zu verhindern.

### Aktiver Abruf (Active Recall)
Durch den Abruf der aktuellsten Vektoren und harten Grenzen für eine Sitzung kann ein Chat-Agent (wie Sybil) den aktuellen psychologischen "Standort" des Nutzers verstehen, ohne dessen gelöschten Chat-Verlauf lesen zu müssen.