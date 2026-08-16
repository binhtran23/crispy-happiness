## Lecture Learning

Lecture material lives under `LEC_DAYXX/` folders (course COMP2010) — the slide deck PDF(s) for that lecture, plus that folder's `knowledge/` subfolder.

### Roles & models

- **`lecture-tutor` (Opus)** — runs the actual teaching session: orientation, Socratic teaching, Examiner checks, recap, and maintains `LEC_DAYXX/knowledge/NOTES.md`. Opus is used here because the adversarial/Socratic reasoning benefits from it.
- **`lecture-extractor` subagent (Sonnet)** — pure source-grounded extraction only, used to produce `LEC_DAYXX/knowledge/core-knowledge.md` per `EXTRACTION.md`. No teaching, no interaction — a cheaper model is a good fit for offloading page-heavy reading.

### Extraction delegation policy

Before starting a session, `lecture-tutor` checks the lecture folder:

1. If `LEC_DAYXX/knowledge/core-knowledge.md` already exists → use it as the primary source; don't re-read the PDF(s).
2. If it doesn't exist and the deck is **≤ 40 pages** → `lecture-tutor` (Opus) reads the PDF(s) directly **using vision** (page images), same as before extraction existed — this is intentional for short decks, since seeing diagrams/figures directly helps teaching. This is different from the subagent's reading method (text extraction via the `pdf`/`pptx` skill, see `EXTRACTION.md`), and that's by design, not an inconsistency to "fix".
3. If it doesn't exist and the deck is **> 40 pages** → delegate: launch the `lecture-extractor` subagent (Task tool, `model: sonnet`) with the PDF path(s) and `LEC_DAYXX/knowledge/core-knowledge.md` as the target output, following `EXTRACTION.md`. Wait for it to finish, then use the resulting file as the primary source — don't pull the raw PDF into the tutoring session's own context in this case.

The 40-page threshold is a starting point, not fixed — adjust it here (single source of truth) if it proves too aggressive or too lax; don't hardcode a different number inside the `lecture-tutor` skill itself.

### Notes location

`NOTES.md` lives at `LEC_DAYXX/knowledge/NOTES.md`, alongside `core-knowledge.md`.

Claude orchestrates by invoking `lecture-tutor` (and, when needed, the `lecture-extractor` subagent) — it does not duplicate either one's internal instructions here.