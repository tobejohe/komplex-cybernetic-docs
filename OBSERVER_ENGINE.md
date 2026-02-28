# The Observer Engine

**Version**: 1.0 (Psychometric Expansion)
**Role**: Asynchronous Psychometric Profiling & Vectorization

## 1. Overview

The Observer Engine is a "Ghost Worker" that operates exclusively in the background. Its purpose is to distill raw, sensitive chat conversations into abstract, multi-dimensional psychometric vectors. This process enables long-term memory and precise user-matching without persisting personal identifiable information (PII) indefinitely.

## 2. The 6 Psychometric Dimensions

The Engine extracts 6 specific vectors from every interaction batch:

| Dimension | Key | Description |
| :--- | :--- | :--- |
| **Elitism** | `v_elitism` | Status striving, exclusivity vs. egalitarianism. |
| **Bandwidth** | `v_bandwidth` | Cognitive density, over-intellectualization vs. emotional simplicity. |
| **Hedonism** | `v_hedonism` | Sensory intensity, excess vs. asceticism/restraint. |
| **Entropy** | `v_entropy` | Desire for chaos/boundary dissolution vs. order/predictability. |
| **Friction** | `v_friction` | Conflict readiness, hardness (Masochism/Sadism) vs. harmony. |
| **Vector** | `v_vector` | Fundamental drive, compensatory behavior, or spiritual yearning. |

## 3. The Distillation Process (Lifecycle)

1. **Trigger**: The HTTP Producer sends a `{ session_id }` event to the `DISTILLATION_QUEUE`.
2. **Retrieval**: The Observer fetches up to 5 undistilled messages for that session from D1.
3. **Analytic Transformation**:
   - The LLM (Gemini 2.5 Flash) receives the raw text and the `system_observer.md` prompt.
   - It outputs a structured JSON object containing a descriptive sentence for each of the 6 dimensions.
4. **Vectorization**:
   - Each of the 6 descriptive sentences is sent to `text-embedding-004`.
   - Google returns a 768-dimensional float array for each sentence.
5. **Persistence**:
   - The 6 vectors are stored in **Cloudflare Vectorize**.
   - Metadata includes `session_id`, `type` (e.g., "hedonism"), and `timestamp`.
6. **Amnesia Protocol**:
   - The source messages in D1 are marked as `is_distilled = 1`.
   - The scheduled Garbage Collector will delete these raw messages after the retention period (8 weeks).

## 4. Usage Patterns

### Semantic Matching
To match a user to an event or another user, the system calculates the **Cosine Similarity** between their stored vectors in Vectorize. Because the descriptions are clinical and abstracted, matching works across domains (e.g., matching a psychological desire to a musical Vibe).

### Active Recall (Future)
By retrieving the most recent vectors for a session, a Chat Agent (like Sybil) can understand the user's current psychological "location" without needing to read their deleted chat history.
