# Hackathon-First Support Triage Agent Architecture

This repo is not asking for a generic enterprise support platform. It is asking for a terminal-based agent that produces strong `output.csv` rows against the shipped corpus and sample labels. The architecture should therefore optimize for grounded answers, practical routing, and low hallucination risk without drifting into unnecessary production complexity.

The core design principle is simple: reply when the corpus clearly supports an answer, escalate when it does not.

---

## 1. System Design

### Core Architecture: Evidence-Routing Pipeline

```
INPUT
  |
  v
STAGE 0: INPUT NORMALIZATION
- Clean subject + issue text
- Normalize whitespace and obvious noise
- Extract obvious signals (URLs, account words, legal language, card terms)

  |
  v
STAGE 1: HARD SAFETY SCREEN
- Detect truly unsafe or non-answerable cases
- Only narrow hard rules bypass retrieval

  |
  v
STAGE 2: CANDIDATE RETRIEVAL
- Use company as a hint, not a source of truth
- Run hybrid retrieval over the most likely corpus
- If company is weak, allow a cross-corpus probe

  |
  v
STAGE 3: EVIDENCE-BASED ROUTING
- Infer final company from ticket + evidence
- Infer product_area from top evidence cluster
- Decide replied vs escalated from answerability + safety
- Classify request_type

  |
  v
STAGE 4: RESPONSE GENERATION
- Answer only from retrieved evidence
- Prefer extractive or lightly abstractive responses
- Acknowledge secondary intent when relevant

  |
  v
STAGE 5: EVIDENCE VALIDATION
- Check every material claim against retrieved chunks
- If support is weak or contradictory, escalate

  |
  v
OUTPUT
status, product_area, response, justification, request_type
```

### Why this architecture fits the repo

- The evaluator scores row outputs, not architectural ceremony.
- The sample data suggests broad, corpus-shaped labels rather than rigid internal taxonomies.
- The biggest scoring risk is not under-engineering. It is hallucinating, over-escalating, or forcing the wrong label structure.

---

## 2. Routing Philosophy

The architecture should not treat every risky-looking phrase as an automatic escalation. It should separate three categories:

### 1. Hard escalate

These bypass retrieval because the agent should not attempt to resolve them directly.

Examples:
- Explicit legal threats
- PII exposure handling
- Severe emotional distress or safety concerns
- Requests requiring account-specific backend action not supported by docs
- Clearly malicious or manipulative prompts that are not support issues

### 2. Sensitive but answerable

These can still receive a grounded reply if the corpus contains procedural guidance.

Examples:
- Stolen traveler's cheques
- General fraud support routing
- Account help that has documented steps
- Billing or support-contact questions with clear published guidance

### 3. Unsupported or out-of-scope

These should receive a safe refusal or escalation depending on the row.

Examples:
- Trivia
- Requests unrelated to HackerRank, Claude, or Visa support
- Prompts with no retrievable grounding in the corpus

The decision standard is:

```python
if hard_escalate:
    status = "escalated"
elif evidence_supports_grounded_answer:
    status = "replied"
else:
    status = "escalated"
```

This is intentionally narrower than a production compliance gate. For this hackathon, broad escalation hurts usefulness and score.

---

## 3. Company and Product Area Inference

### Company is a hint, not a fixed truth

The input `company` field can be correct, missing, noisy, or misleading. The architecture should treat it as prior evidence, not as a hard router.

Recommended approach:

1. Build a lightweight keyword prior from the ticket text.
2. If the prior is weak or the input says `None`, run a small cross-corpus retrieval probe.
3. Choose the final company using both ticket signals and retrieval evidence.

This avoids a common failure mode:

```
wrong company -> wrong corpus -> wrong retrieval -> confident bad answer
```

### Product area should be evidence-derived

Do not make `product_area` a rigid taxonomy-first problem.

The challenge requires the "most relevant support category / domain area", and the sample outputs suggest broad labels like:

- `screen`
- `community`
- `privacy`
- `travel_support`

That means the system should infer `product_area` from the best supporting documents, not force fine-grained synthetic enums.

Recommended metadata per chunk:

```python
{
    "company": "HackerRank | Claude | Visa",
    "top_level_area": "screen | community | privacy | ...",
    "path": "data/.../file.md",
    "title": "doc title"
}
```

Preferred policy:

- Use the top evidence cluster to infer `product_area`
- Prefer broad corpus-aligned labels
- Allow blank or `unknown` on weak escalations

---

## 4. Retrieval Strategy

### Hybrid retrieval is still the right default

Use both lexical and semantic search:

- BM25 or keyword search for exact phrases, product names, error text, and procedural wording
- Vector search for paraphrased natural-language issues
- Merge results with simple rank fusion

### Retrieval should be answerability-driven

Do not rely on fixed universal similarity thresholds. A score like `0.72` is not portable across embedding models, chunking choices, or corpora.

Instead, use a composite view of evidence quality:

- Does the top result match the likely company?
- Do top chunks agree on the same area?
- Is there strong title/path overlap with the issue?
- Is one area clearly dominating the top results?
- Do retrieved chunks contradict each other?

Good answerability signal:

- Top evidence is concentrated in one company and one area
- Retrieved chunks contain procedural or factual support for the user's ask

Bad answerability signal:

- Evidence is split across companies
- Chunks are loosely related but not responsive
- Top results disagree materially
- Only generic support pages appear

### Retrieval flow

```python
def retrieve_candidates(ticket):
    prior_company = infer_company_prior(ticket)
    corpora = select_corpora(prior_company)

    bm25_results = bm25_search(ticket.search_query, corpora)
    vector_results = vector_search(ticket.search_query, corpora)
    fused_results = fuse(bm25_results, vector_results)

    return rerank_with_metadata(fused_results, ticket)
```

---

## 5. Classification Strategy

### Keep classification lightweight

Not every field needs a heavy LLM classification pass.

The only strict output enum in the challenge is:

- `product_issue`
- `feature_request`
- `bug`
- `invalid`

That makes `request_type` a good structured classification task. The rest of the routing should be evidence-led.

### Primary intent drives the row

Rows can contain multiple requests. Since the output CSV only has one `status`, one `product_area`, and one `request_type`, define a clear policy:

- Choose one `primary_intent` for the row
- Track `secondary_intent` separately
- Mention secondary intent in `response` or `justification`
- If the secondary intent is unsafe or unsupported, escalate the whole row

This keeps the pipeline deterministic and explainable.

### Avoid chain-of-thought as an output requirement

Do not require freeform chain-of-thought in model output. It is unnecessary here and makes parsing brittle.

Prefer concise structured rationale:

```python
{
    "request_type": "product_issue",
    "primary_intent": "user wants to know how long HackerRank tests remain active",
    "secondary_intent": None,
    "routing_reason": "retrieved HackerRank screen docs directly answer test expiration behavior"
}
```

---

## 6. Response Generation

### Prefer extractive or lightly abstractive answers

Most support answers in this dataset are procedural. The safest response style is:

- copy or normalize steps from the retrieved docs
- preserve exact contact details, URLs, time windows, and warnings only when they appear in the corpus
- avoid broad synthesis unless multiple chunks clearly agree

### Response guidelines

1. Use only retrieved evidence
2. Answer the primary intent directly
3. Acknowledge the secondary intent if needed
4. Do not invent policies, timelines, contact channels, or feature behavior
5. Keep the wording concise, but do not over-compress step-by-step answers

The previous instinct to globally force "3-5 sentences maximum" is too rigid for procedural support. Some rows need a short answer; some need ordered steps.

### Suggested response prompt shape

```text
You are a support agent answering from provided documentation only.

Rules:
1. Use only the supplied evidence
2. If the evidence is partial, say so and avoid unsupported claims
3. Do not invent steps, URLs, contacts, dates, or policies
4. Answer the primary intent first
5. Keep the response concise but complete
```

---

## 7. Validation Layer

### Default to lightweight evidence validation

Do not require an LLM judge on every answered row. It adds cost, latency, and false negatives without guaranteeing better outputs.

Default validation should check:

- Every material claim appears in at least one retrieved chunk
- URLs, phone numbers, and instructions are present in evidence
- The response does not mix contradictory chunks into one answer

If validation fails:

- remove the unsupported claim, or
- escalate the row

### Optional verifier for borderline rows

An LLM verifier can still be useful for:

- multi-document synthesis
- borderline answerability cases
- contradiction detection when top chunks disagree

But it should be optional, not mandatory for every reply.

---

## 8. Decision Logic

The decision tree should stay simple and inspectable:

```python
if hard_safety_screen(ticket):
    return escalated_row(...)

candidates = retrieve_candidates(ticket)
decision = route_from_evidence(ticket, candidates)

if decision.answerable is False:
    return escalated_row(...)

draft = generate_response(ticket, decision, candidates)
validated = validate_response(draft, candidates)

if not validated:
    return escalated_row(...)

return replied_row(...)
```

This is easier to debug than a mega-prompt and easier to defend in the judge interview than an over-layered agent graph.

---

## 9. Failure Modes and Tradeoffs

### 1. Over-escalation vs usefulness

If the system escalates too aggressively, it avoids hallucination but produces weak scores. This challenge rewards safe usefulness, not blanket refusal.

### 2. Taxonomy rigidity vs corpus realism

Rigid product-area taxonomies look clean in design docs but often mismatch the actual labels implied by the corpus and sample outputs.

### 3. Classification-led vs retrieval-led routing

Classification-led designs are easy to explain but brittle when labels are fuzzy. Retrieval-led routing is often stronger for support corpora because the corpus defines the available answer space.

### 4. Verifier rigor vs practical value

A verifier on every row may sound safer, but it also adds complexity. Lightweight evidence checks often deliver most of the value with fewer moving parts.

### 5. Single-row outputs vs multi-intent tickets

The data model only allows one row-level decision, so the architecture must formalize primary-intent handling instead of pretending every intent can be independently routed.

### 6. Corpus heterogeneity

HackerRank, Claude, and Visa docs are not shaped the same way. Chunking and retrieval metadata need to respect that instead of assuming one universal support-doc format.

---

## 10. Implementation Sketch

The implementation should revolve around a few explicit internal artifacts:

```python
NormalizedTicket = {
    "subject": str,
    "issue": str,
    "provided_company": str,
    "clean_text": str,
    "search_query": str,
}

SafetyScreenResult = {
    "hard_escalate": bool,
    "flags": list[str],
    "reason": str,
}

RetrievalCandidate = {
    "company": str,
    "top_level_area": str,
    "title": str,
    "path": str,
    "snippet": str,
    "scores": dict,
}

RoutingDecision = {
    "final_company": str,
    "product_area": str,
    "status": str,
    "request_type": str,
    "primary_intent": str,
    "secondary_intent": str | None,
    "decision_reason": str,
    "answerable": bool,
}

ValidatedAnswer = {
    "response": str,
    "justification": str,
    "evidence_paths": list[str],
    "validation_passed": bool,
}
```

This gives the implementation enough structure to stay deterministic without locking it into an overly complex agent framework.

---

## 11. Test Strategy

Use `sample_support_tickets.csv` as the calibration source for both routing behavior and label style.

The architecture should be tested against:

1. `company=None` rows where retrieval should infer the domain
2. Wrong or noisy company hints
3. Multi-intent tickets
4. Sensitive but answerable cases
5. Unsupported or out-of-scope prompts
6. Weak-retrieval rows that should escalate
7. Procedural answers that require exact steps from one document
8. Conflicting evidence cases that should escalate

Success looks like:

- low hallucination rate
- moderate escalation rate
- corpus-faithful `product_area`
- short, traceable justifications
- responses that read like grounded support answers rather than generic chatbot prose

---

## Final Checklist

Before implementation, verify:

- Hard escalation rules are narrow and explicit
- Company inference can use retrieval when the input field is weak
- `product_area` is evidence-derived rather than taxonomy-locked
- Responses are grounded in retrieved corpus content
- Unsupported or contradictory evidence causes escalation
- Multi-intent handling has a clear primary-intent policy
- Justifications are concise and traceable
- The architecture is optimized for this repo's CSV outputs, not generic production theater

The best submission here will not be the one with the most layers. It will be the one that routes cleanly, answers from evidence, and knows exactly when not to guess.
