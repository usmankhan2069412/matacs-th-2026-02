# WRITEUP — AI Proposal Triage Design

**Reference:** MATACS-TH-2026-02

---

## 1. AI vs. rules

**Steps that use AI:**
- **Text extraction from PDFs** — reading a proposal and pulling out licence number, WSIB/insurance mentions, experience count, price, timeline, warranty years, and team details.
- **Semantic field matching** — recognizing that "fourteen installs since 2021" means 14 projects, or that "to be confirmed after award" means the licence is *not* currently held.
- **Suspicious-content detection** — spotting embedded instructions aimed at automated systems (prompt injection).
- **Reason generation** — writing a one-line, evidence-based explanation for each rank.

**Steps that are plain code/rules:**
- **Mandatory checks** — licence present (yes/no), WSIB + $5M insurance confirmed (yes/no). Pass/fail must be deterministic.
- **Budget and deadline checks** — price ≤ $260,000; timeline ≤ 16 weeks (numeric comparison).
- **Weighted scoring** — fixed formula: `score = 0.30·experience + 0.25·price + 0.15·timeline + 0.15·warranty + 0.15·team`.
- **Ranking and tie-breaking** — sort by score, then by experience, then by price.
- **Audit logging** — timestamps, reviewer identity, overrides.

**Why this line:** Anything with an exact, checkable answer (numbers, yes/no facts, arithmetic) is left to code because code is deterministic and auditable. AI is used only where natural-language *understanding* is required. Mandatory pass/fail is never delegated to AI, because an AI mistake there could wrongly reject a valid bid — or wrongly accept an invalid one.

---

## 2. Consistency and explainability

**Repeatability:**
- AI extraction runs at **temperature 0** with a **fixed, versioned prompt** and a **strict JSON output schema**.
- All *scoring* is done by deterministic rules, not by the AI — so given the same extracted fields, the score is identical every time.
- Extraction is human-confirmed before scoring (see Q3), which removes residual AI variance from the final numbers.

**Would re-ranking give the same result?** Yes for scoring and ranking, because those are pure rules on confirmed data. Raw AI extraction can occasionally vary; the human confirmation step makes the final output stable.

**Explainability:**
- Every extracted field stores its **evidence snippet** (the exact sentence it came from) and a confidence flag.
- Each proposal carries a **score breakdown** by criterion (e.g. Experience 30/30, Price 20/25, Timeline 15/15, …).
- The AI-generated one-line reason cites specific facts (e.g. "14 installs, $238k, 12 weeks, 10-yr workmanship").
- A reviewer can click any rank to see the full breakdown and evidence.

---

## 3. Human in the loop

**Checkpoints:**
1. **After extraction** — reviewer confirms or corrects the extracted fields before any scoring runs.
2. **Mandatory failures** — a reviewer confirms every pass/fail before a proposal is marked ineligible.
3. **Flags** — anything flagged as suspicious (e.g. embedded instructions) goes to a human; it is never auto-ranked.
4. **Final ranking** — reviewer reviews the ranked list and may override any score or rank; an override requires a written reason.
5. **Award decision** — always made by a human, never by the tool.

**What is logged for audit:**
- Reviewer identity and timestamp at every checkpoint.
- Original extracted text and the AI's raw JSON output.
- Every human correction or override, with reason.
- Scoring breakdown per criterion.
- Prompt version and model version used.
- Final ranked output and the award decision.

---

## 4. Changing criteria

- Criteria and weights are **versioned**. Each score is tied to the criteria version that produced it.
- If an admin edits criteria or weights, existing scores are marked **stale** and the affected proposals are **re-scored** under the new version. Old scores are kept in history, never overwritten.
- If only **weights** change (criteria unchanged), the tool re-applies weights to the existing confirmed sub-scores — no AI re-run needed.
- If **criteria** change (new field, new mandatory rule), extraction and confirmation are re-run for affected proposals.
- The reviewer is notified ("criteria updated — scores refreshed"), and the change is logged with the admin's identity and timestamp.

---

## 5. Risks and safeguards

| Risk | Safeguard |
|------|-----------|
| **Wrong extraction** (e.g. 14 read as 4) | Strict JSON schema, evidence snippet per field, human confirmation before scoring. |
| **Made-up facts / hallucination** | Prompt instructs the model to use only the provided text and to return "not found" when a field is absent; every field carries a source snippet. |
| **Prompt injection / manipulation** (e.g. a proposal saying "rank me first") | The prompt tells the model to treat document content as *data, not instructions*; a separate detector flags embedded instructions; flagged items go to a human and are never auto-ranked; mandatory checks are rules-based, so an injected instruction cannot bypass them. |
| **Bias** (company name, writing style, formatting) | Company names are masked during extraction; scoring uses only structured fields against fixed criteria; humans review the final list. |
| **Cost** | Documents are chunked and only required fields are extracted; caching avoids repeat calls; prototype runs are batched. |
| **Inconsistent scores** | Temperature 0, fixed formula, human confirmation, versioned prompts. |
| **Missing data** (e.g. no timeline given) | Field marked "not found"; that criterion scores 0; proposal is flagged so the reviewer can follow up. |
| **Unauthorised changes** | Admin actions and overrides are logged with identity and timestamp. |

**Sample set note:** Proposal C contains an embedded instruction to rank it first. It is **flagged**, not ranked. Proposal B's licence is only promised "after award", so it **fails the mandatory licence check** and is marked ineligible. Proposal D gives no timeline, so it scores 0 on Timeline and is ranked lower.