# Provenance Guard

A backend system for AI content attribution on creative writing platforms. Accepts a piece of text, runs it through a two-signal detection pipeline, returns a confidence score and a transparency label, and handles appeals from creators who think they've been misclassified.

---

## Architecture Overview

When a piece of text is submitted, here's the full path it takes:

1. **POST /submit** receives `text` and `creator_id`. Flask-Limiter checks the rate quota first, if the IP has hit its limit, the request is rejected with 429 before anything else runs.
2. The text is handed to both detection signals simultaneously:
   - **Signal 1 - LLM Classifier (Groq)**: sends the text to `llama-3.3-70b-versatile` with a structured prompt that asks for an `ai_probability` score (0–1). Captures semantic and stylistic coherence holistically.
   - **Signal 2 - Stylometric Analyzer (pure Python)**: computes sentence length variance, type-token ratio (TTR), and punctuation density, averaged into a single `stylo_score` (0–1). Captures structural statistical fingerprints.
3. Both scores are combined into a single `confidence` value: `0.6 × llm_score + 0.4 × stylo_score`. For short texts (< 50 words) the stylometric weight drops to 0.1 because variance metrics are unreliable on small samples.
4. The confidence score is mapped to one of three transparency label variants.
5. A structured entry is written to the SQLite audit log.
6. The response returns `content_id`, `attribution`, `confidence`, both signal scores, and the label text.

For **appeals**: `POST /appeal` accepts a `content_id` and `creator_reasoning`. The system looks up the original record, flips its status to `under_review`, appends the reasoning and a timestamp to the log entry, and returns a confirmation. No automated re-classification happens, a human reviewer would triage from the log.

---

## Detection Signals

### Signal 1: LLM Classifier (Groq / llama-3.3-70b-versatile)

**What it measures**: Semantic and stylistic coherence holistically. The model has internalized what AI-generated text feels like, balanced hedging language, generic phrasing, unnaturally smooth transitions, and a kind of confident completeness that human writing rarely has.

**Why I chose it**: It's the only signal that can catch semantic patterns. A human writer has personality, inconsistency, emotional register shifts, tangents, things that don't emerge from next-token prediction. Stylometric heuristics alone would miss a polished AI paragraph that happens to have irregular sentence lengths.

**What it misses**: It will flag human writing that deliberately mimics AI style, corporate communications, academic boilerplate, policy writing. It also can't catch AI text that's been lightly edited to introduce natural-sounding irregularities.

**Output**: a float 0.0–1.0 extracted from a structured JSON response (`{"ai_probability": 0.83, "reasoning": "..."}`). The prompt explicitly tells the model that 0.5 means genuinely uncertain, not "leaning slightly."

---

### Signal 2: Stylometric Heuristics (pure Python)

**What it measures**: Statistical structure of the text at the sentence and vocabulary level. Three sub-metrics:

- **Sentence length variance**: AI text clusters sentences around a "comfortable" length. Low standard deviation → high AI score. Normalized so stddev of 0 → 1.0, stddev of 15+ words → 0.0.
- **Type-token ratio (TTR)**: unique words / total words. AI text clusters in the 0.55–0.75 range. Very low TTR (repetitive poetry) or very high TTR (stream-of-consciousness) is more likely human. Scored by distance from the AI-typical midpoint.
- **Punctuation density**: punctuation marks / total characters. AI text is conservative and predictable. Higher density → more human → lower AI score.

The three sub-scores are averaged into a single `stylo_score`.

**Why I chose it**: It's genuinely independent from the LLM signal, one is semantic, one is structural. Texts that fool the LLM (unusual topics written in AI's characteristic style) still have AI's statistical smoothness, which stylometrics catches.

**What it misses**: Short texts (< 50 words) produce unreliable statistics. Highly edited human writing, literary fiction, journalism, also scores AI-like because editors flatten exactly the irregularities these metrics look for.

---

## Confidence Scoring

**How it works**: `confidence = 0.6 × llm_score + 0.4 × stylo_score`

I weighted LLM heavier because it captures semantic meaning, which is harder to fake structurally. Stylometrics is a useful independent check but it's noisier, especially on short texts.

**Thresholds**:

| Confidence Range | Attribution | Label Shown |
|---|---|---|
| 0.70 – 1.00 | `likely_ai` | High-confidence AI |
| 0.40 – 0.69 | `uncertain` | Uncertain |
| 0.00 – 0.39 | `likely_human` | High-confidence human |

I set the human threshold at 0.39 rather than 0.49 deliberately, a false positive (labeling a human's work as AI) is worse than a false negative on a writing platform. The system needs a stronger signal before making an accusation, not just a coin flip.

**How I validated the scores are meaningful**: Tested the four reference inputs from the spec:

| Input | LLM Score | Stylo Score | Combined | Attribution |
|---|---|---|---|---|
| Clearly AI ("Artificial intelligence represents a transformative...") | 0.80 | 0.59 | **0.78** | `likely_ai` |
| Clearly human ("ok so i finally tried that new ramen place downtown...") | 0.20 | 0.57 | **0.35** | `likely_human` |
| Borderline - formal human writing ("The relationship between monetary policy...") | 0.80 | 0.65 | **0.79** | `likely_ai` |
| Borderline - lightly edited AI output ("I've been thinking a lot about remote work lately...") | 0.40 | 0.54 | **0.41** | `uncertain` |

The two clearly different inputs span a 0.43 confidence range (0.35 vs 0.78). The lightly edited AI text moved from a pure-LLM score of 0.40 to a combined score of 0.41, the stylometric signal pushed it into the `uncertain` zone rather than letting it slip through as `likely_human`. That's the combination working as designed.

---

## Transparency Label

Three variants, written out exactly as the API returns them (with `{confidence_pct}` replaced by the actual percentage):

**High-confidence AI** (`confidence >= 0.70`):
> AI-Assisted Content - Our system found strong signals that this content may have been generated with AI assistance (confidence: 78%). This label reflects an automated assessment, not a definitive determination. The creator can contest this classification through the appeal process.

**Uncertain** (`0.40 <= confidence < 0.70`):
> Origin Unclear - Our system couldn't confidently determine whether this content was written by a human or generated with AI assistance (confidence: 42%). Treat this content as you would any unverified post.

**High-confidence human** (`confidence < 0.40`):
> Likely Human-Written - Our system found no strong signals of AI generation in this content (confidence of AI: 35%). This is an automated assessment and does not constitute a guarantee of human authorship.

A few design decisions behind this wording:
- All three variants show the raw confidence percentage so a non-technical reader can see how certain the system actually is.
- The AI label explicitly mentions the appeal path so creators see it immediately, this is intentional.
- The human label includes a hedge ("does not constitute a guarantee") because overclaiming in the human direction erodes trust just as much as false AI accusations.
- None of the labels say "we verified." The language is consistently probabilistic across all three variants.

---

## Rate Limiting

**Limits applied to `POST /submit`**: `10 per minute; 100 per day`

**Reasoning**:

- **10 per minute**: A real writer submitting their own work might test a few pieces in a session, but not 10 in 60 seconds, that pace is scripted. A legitimate burst of 10 is already generous. The Groq API itself has rate limits, so allowing unlimited volume would just produce downstream failures anyway.
- **100 per day**: A prolific writer publishing multiple pieces per day might submit 5–10 pieces. 100/day gives 10× headroom above that for any realistic single-IP usage pattern. An adversary trying to flood the system to probe the classifier or exhaust API quota would hit this ceiling quickly.

The `/appeal` and `/log` endpoints are not rate-limited. Appeals are low-volume by nature, and `/log` is read-only.

**Evidence** - running 12 rapid requests confirms the limiter fires after 10:

```
1: 200
2: 200
3: 200
4: 200
5: 200
6: 200
7: 200
8: 200
9: 200
10: 200
11: 429
12: 429
```

---

## Audit Log

Every submission and appeal is written to a SQLite database (`provenance.db`). The `GET /log` endpoint returns all entries as JSON.

**Sample entries** (from actual test runs):

```json
{
  "content_id": "3d6f4094-f4a7-40d2-ae52-a70fceb53437",
  "creator_id": "m5-label-ai",
  "timestamp": "2026-06-26T00:53:47.123456+00:00",
  "text_snippet": "Artificial intelligence represents a transformative paradigm shift...",
  "attribution": "likely_ai",
  "confidence": 0.7791,
  "llm_score": 0.8,
  "stylo_score": 0.5905,
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself for a university policy brief. My department requires formal academic register and hedged language — that is the style standard, not evidence of AI use.",
  "appeal_timestamp": "2026-06-26T00:54:29.759491+00:00",
  "label": "AI-Assisted Content — Our system found strong signals..."
}
```

```json
{
  "content_id": "127170cf-6603-4f47-9c6d-18debb31ae75",
  "creator_id": "m5-label-uncertain",
  "timestamp": "2026-06-26T00:53:51.234567+00:00",
  "text_snippet": "I have been thinking a lot about remote work lately...",
  "attribution": "uncertain",
  "confidence": 0.4156,
  "llm_score": 0.4,
  "stylo_score": 0.5386,
  "status": "classified",
  "appeal_reasoning": null,
  "appeal_timestamp": null,
  "label": "Origin Unclear — Our system couldn't confidently determine..."
}
```

```json
{
  "content_id": "59c735d1-c2e4-4335-8801-50809575a434",
  "creator_id": "m5-label-human",
  "timestamp": "2026-06-26T00:53:55.345678+00:00",
  "text_snippet": "ok so i finally tried that new ramen place downtown...",
  "attribution": "likely_human",
  "confidence": 0.3526,
  "llm_score": 0.2,
  "stylo_score": 0.5663,
  "status": "classified",
  "appeal_reasoning": null,
  "appeal_timestamp": null,
  "label": "Likely Human-Written — Our system found no strong signals..."
}
```

---

## Known Limitations

**Formal academic and professional writing by humans will frequently score as AI-generated.**

A graduate student writing a thesis, a lawyer drafting a brief, or a researcher writing an abstract produces text that triggers both signals simultaneously: hedged language and balanced assertions (LLM signal leans AI), combined with uniform sentence structure and low punctuation density (stylometrics agrees). In testing, a two-sentence monetary policy abstract scored 0.79, firmly in the `likely_ai` zone, despite being human-written.

This isn't a calibration bug I can fix by tweaking weights. It's a fundamental property of the signals: they're measuring surface features that correlate with AI generation but are also produced by formal writing conventions. The only mitigation is the appeal path, and this is the scenario it was designed for.

**Short texts (< 50 words) rely almost entirely on the LLM signal.**

The stylometric weight drops to 10% for texts under 50 words because variance statistics are unreliable on small samples. A haiku, a tweet-length poem, or a two-sentence blurb gets classified almost purely on the LLM's holistic read, with no structural check. That's a weaker classification and the audit log doesn't currently flag it explicitly.

---

## Spec Reflection

**One way the spec helped**: Writing the three label variants word-for-word in planning.md before touching the code meant I couldn't fudge it during implementation. When I wrote `generate_label()`, I was just transcribing text I'd already committed to, the asymmetric threshold (0.39 for human, 0.70 for AI) and the probabilistic language in each variant were already decided. The spec forced a design decision that would have stayed vague if I'd left it for implementation.

**One way the implementation diverged**: The spec described the stylometric weight as 40% flat. In practice, applying 40% stylometric weight to a 40-word text produced noisy, counterintuitive results, the variance of 3–4 sentence lengths is statistically meaningless. I added a short-text adaptation that drops the stylometric weight to 10% for texts under 50 words. This wasn't in the original spec because I didn't think through how the metric would behave at the tail until I saw actual scores. The spec said "flag as low-reliability" as an alternative, I went with the weight adjustment instead because it was cleaner and didn't require adding a separate field to the response.

---

## AI Usage

**Instance 1 - Flask app skeleton + LLM classifier function**

I gave Claude my Detection Signals spec section (the output format, prompt strategy, and function signature I wanted) plus the architecture diagram. I asked for: a Flask app skeleton with a `POST /submit` stub, a standalone `classify_with_llm(text)` function that returns a float, and a SQLite-backed log helper with a `GET /log` endpoint.

What it produced was mostly correct but had one issue: the initial prompt didn't explicitly tell the model that 0.5 means *genuinely uncertain*, it just asked for an AI probability. This caused the model to treat 0.5 as a mild lean rather than a neutral signal. I revised the system prompt to include: *"0.5 means genuinely uncertain (use this when you truly cannot tell, not as a default)"* which produced more calibrated outputs on borderline inputs.

**Instance 2 - Stylometric analyzer function**

I gave Claude the Signal 2 spec (the three sub-metrics with their normalization logic) and asked for a standalone `analyze_stylometrics(text)` function that returns each sub-score individually alongside the combined average.

The generated function computed TTR correctly but the sentence splitting used `text.split('.')` which broke on abbreviations and decimal numbers. I replaced the sentence splitter with `re.split(r'(?<=[.!?])\s+', text.strip())`, a lookbehind regex that only splits on sentence-ending punctuation followed by whitespace, which handles most cases correctly. I also added filtering for empty strings after the split, which the original didn't handle.

---

## API Reference

| Method | Endpoint | Request Body | Response |
|---|---|---|---|
| `POST` | `/submit` | `{ "text": "...", "creator_id": "..." }` | `{ content_id, attribution, confidence, llm_score, stylo_score, stylo_detail, label, timestamp }` |
| `POST` | `/appeal` | `{ "content_id": "...", "creator_reasoning": "..." }` | `{ status, content_id, message, appeal_timestamp }` |
| `GET` | `/log` | - | `{ count, entries: [...] }` |

**Run locally**:
```bash
pip install -r requirements.txt
# add GROQ_API_KEY to .env
python app.py
```

## Demo Video

The video covers: an overview of the two detection signals (LLM classifier via Groq and pure-Python stylometric analyzer) and how their scores combine into a confidence value with asymmetric thresholds; live end-to-end demos of the `POST /submit` endpoint across three cases, clearly AI-generated text, clearly human text, and a borderline uncertain case; a walkthrough of the appeal workflow showing a `POST /appeal` call flipping a record to `under_review`; the rate limiting behavior (10/min, 100/day) with a live loop showing 429s after request 10; the `GET /log` audit trail; and a discussion of two design decisions, the probabilistic label wording and why the appeal path matters more than detection accuracy.

**[Watch the demo (demo.mp4)](demo.mp4)**