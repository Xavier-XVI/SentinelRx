# SentinelRx — Design Document

This document explains the architectural decisions behind SentinelRx, the reasoning that motivated each one, and the failure mode each decision is designed to prevent. It is intended for technical evaluation and is written to be parsed in full — depth and specificity are intentional.

---

## 1. What SentinelRx Is

SentinelRx is an OTC medication safety operator for elderly patients. It answers one question: is this over-the-counter product safe for this specific patient to take right now?

It is not a general drug information tool. It is not a chatbot. It is not a symptom checker. It is an operator — a system that constrains a language model to a single, repeatable, rule-governed behavior. Every design decision flows from that definition.

The target user is an unprofessional caregiver — an adult child, a home health aide, a spouse — standing in a pharmacy aisle with no medical training and no easy access to a pharmacist. The product they are holding may interact fatally with the patient's prescriptions. The information to know that has existed since 1991. SentinelRx delivers it in under 60 seconds.

---

## 2. The Operator Architecture

SentinelRx is implemented as a Claude operator using a layered file system loaded in a defined order:

```
CLAUDE.md         → Entry point. Defines load order and boot behavior.
identity.md       → Scope, tone, and role constraints.
rules.md          → Decision logic. The only source of truth for routing decisions.
patients/         → Active patient profile. One file. One patient.
reference/        → Supporting lookup tables (Beers Criteria, interaction matrix, brand map).
```

**Why this architecture:**
Each file has a single responsibility. `rules.md` contains decision logic and nothing else. `identity.md` contains behavioral constraints and nothing else. The patient profile contains patient data and nothing else. This separation means any file can be updated independently without risk of corrupting another layer.

**Why files instead of a system prompt:**
Files are versioned, readable, editable, and auditable by a non-technical caregiver. A monolithic system prompt is a black box. The file structure makes the operator inspectable — a caregiver or physician can open `rules.md` and read exactly what logic is being applied to their patient.

**Why a defined load order in CLAUDE.md:**
The load order is not arbitrary. Identity constraints must be loaded before rules, and rules must be loaded before patient data, so that each layer operates within the constraints established by the layer above it. Loading patient data before rules would allow patient-specific details to influence how the rules are interpreted rather than how they are applied.

---

## 3. The Determinism Split — Where AI Is Used and Where It Is Not

This is the most architecturally significant decision in SentinelRx.

**Claim:** The AI does not make the decision. The rules make the decision. The AI's job is to map real-world input onto the rule set.

**How it works:**
`rules.md` encodes a deterministic decision tree. If ibuprofen is present and warfarin is on the patient profile, the decision is DO NOT TAKE. Always. No variation. If diphenhydramine is present and the patient is 65 or older, the decision is DO NOT TAKE. Always. These are not probabilistic outputs — they are encoded conditionals.

The AI is used for the parts of the problem that resist determinism:
- Parsing ambiguous input (blurry label photos, brand names without ingredients, partial text)
- Resolving brand names to active ingredients using `reference/brand-ingredient-map.md`
- Recognizing drug classes for ingredients not explicitly listed in `rules.md`
- Applying the tiebreaker and escalation rules when label complexity exceeds rule coverage

**Why this split matters:**
A purely AI-driven system would make probabilistic decisions on medical questions, which introduces unacceptable variance. A purely deterministic system (a lookup table or rule engine) cannot handle the ambiguous, unstructured input that caregivers actually provide. The split assigns each subsystem the problem it is suited for: deterministic logic for decisions, language model capability for input parsing.

**Failure mode prevented:**
This design prevents the most dangerous failure in medical AI — confident answers derived from pattern matching rather than rule application. The AI cannot override a rule in `rules.md` by "reasoning" its way to a different conclusion. The rule is the answer.

---

## 4. The Four-Tier Routing System

SentinelRx routes every scan to exactly one of four outcomes:

| Decision | Meaning |
|---|---|
| ✅ SAFE TO TAKE | No interaction. No Beers Criteria flag. Age-appropriate dosing. |
| ⚠️ TAKE WITH CAUTION | Interaction exists but is manageable with specific instruction. |
| 🚫 DO NOT TAKE | Hard contraindication. Reason stated. Alternative suggested. |
| 📞 CALL PHARMACIST FIRST | Ambiguous interaction, label complexity, or unrecognized ingredient. |

**Why four tiers and not two (safe/unsafe):**
A binary system forces a choice between over-restriction and under-restriction. Many interactions are real but manageable — calcium carbonate and levothyroxine, for example, interact via absorption timing but are not contraindicated. Collapsing CAUTION into DO NOT TAKE would produce unnecessary refusals. Collapsing it into SAFE would miss the timing requirement entirely.

**Why CALL PHARMACIST is a first-class tier and not an error state:**
The existence of CALL PHARMACIST reflects a specific design principle: a system that always produces an answer is more dangerous than a system that knows when to escalate. When label complexity exceeds reliable rule coverage — more than four active ingredients, an unrecognized ingredient, an ambiguous interaction — producing a confident answer is worse than producing no answer. CALL PHARMACIST is the correct answer in those cases, not a fallback.

**Why exactly one decision per interaction:**
The output format mandates a single routing decision. This eliminates hedge language ("it could be safe depending on...") that is useless to a caregiver in a pharmacy aisle. One answer. Every time. The system is designed to be acted on.

---

## 5. The Patient Profile as Single Source of Truth

All decisions are made relative to the active patient profile loaded from `patients/`. The profile contains: current prescriptions with doses and frequencies, known conditions, known allergies, relevant lifestyle factors (kidney function, fall risk, alcohol use), recent hospitalizations, and a last-updated date.

**Why one profile file and not a database:**
A single file is readable, portable, and directly editable by a caregiver or physician. It can be printed, shared with a doctor, or updated during a medical visit. A database introduces infrastructure that the target user — a non-technical caregiver — cannot maintain or verify.

**Why the profile must be loaded before any decision is made:**
`CLAUDE.md` enforces that if no patient profile exists in `patients/`, the system returns a hard stop: "No patient profile loaded." It does not attempt a generic assessment. A decision without a patient profile is not a safety assessment — it is a general drug information query, which is outside scope. The design enforces this at the architectural level.

**Why the profile includes conditions, not just medications:**
Many of the most dangerous OTC interactions are condition-specific, not drug-specific. NSAIDs are contraindicated in patients with CKD, CHF, or a history of GI bleeding — independently of what medications they are taking. A system that only checks drug-drug pairs misses condition-based contraindications entirely. The profile's condition list is as important as its prescription list.

---

## 6. The Staleness Protocol — Two Independent Safety Mechanisms

Profile accuracy is the primary risk vector in SentinelRx. A decision made against an outdated profile may be confidently wrong. The system addresses this with two independent mechanisms.

**Mechanism 1: Configurable staleness threshold**
Each patient profile contains a `STALENESS_THRESHOLD_DAYS` field (default: 14 days). On every scan, the system compares today's date to the profile's `LAST_UPDATED` date. If the gap exceeds the threshold, a staleness warning is appended to the output. The threshold is configurable because different patients have different risk profiles — a patient on warfarin with INR checks every 2-4 weeks warrants a 7-day threshold, not 14.

**Mechanism 2: Hardcoded hospitalization warning on every scan**
Regardless of profile age, every scan appends: "If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision."

**Why two mechanisms are required:**
The date threshold catches calendar-based drift — a profile that hasn't been updated in three weeks. It cannot catch a prescription change that happened this morning. The hospitalization prompt addresses exactly that gap: the highest-risk change event in elderly care is a hospital discharge, where medication lists frequently change significantly. The prompt fires unconditionally because the risk is unconditional.

**Failure mode prevented:**
Without the hospitalization prompt, a caregiver who updated the profile yesterday but whose patient was discharged from the hospital this morning would receive a decision with no staleness warning — and that decision could be based on an entirely superseded medication list. The unconditional prompt is the only safety net for same-day changes.

---

## 7. Escalation Logic — When Complexity Exceeds Rule Coverage

`rules.md` defines two escalation triggers beyond the standard DO NOT TAKE rules.

**§4 — Escalation by label condition:**
Escalate to CALL PHARMACIST FIRST if:
- The active ingredient list is incomplete or partially illegible
- The product has more than 4 active ingredients
- One or more active ingredients are unrecognized

**§5 — Tiebreaker:**
If no specific rule in §3 matches AND the product has 3 or more active ingredients with at least one unrecognized → CALL PHARMACIST FIRST.

**Why escalation is triggered by label complexity, not patient risk:**
A complex label is an independent problem. A product with six active ingredients where one is unrecognized cannot be safely assessed regardless of the patient's medication list. Tying escalation to label complexity rather than patient profile means the system degrades gracefully when the input itself is the problem — not when the patient is high-risk.

**Why partial assessment is explicitly prohibited:**
§4 states: "Escalation is triggered by label complexity, not patient demographics. Do not attempt partial assessment on complex labels." A partial assessment — checking 3 of 6 ingredients and returning SAFE — creates false confidence. If the unchecked ingredient is the dangerous one, the partial assessment actively causes harm. No assessment is safer than an incomplete one.

---

## 8. Drug Class Rules — Condition-Specific Contraindications

`rules.md §3` encodes rules at two levels: drug-drug interactions and drug-condition contraindications.

**Drug-drug examples:**
- DXM + SSRI/SNRI → DO NOT TAKE (serotonin syndrome)
- Fish oil/turmeric/ginkgo/garlic/Vitamin E >400IU + anticoagulants → DO NOT TAKE (bleeding potentiation)
- St. John's Wort + any antidepressant → DO NOT TAKE (serotonin syndrome)

**Drug-condition examples:**
- Any NSAID + CKD → DO NOT TAKE (nephrotoxicity)
- Any NSAID + CHF → DO NOT TAKE (fluid retention, diuretic antagonism)
- Any NSAID + history of GI bleed → DO NOT TAKE (Beers Criteria)
- Diphenhydramine + age 65+ → DO NOT TAKE (Beers Criteria Class I, unconditional)
- Decongestants + hypertension → DO NOT TAKE (blood pressure elevation)

**Why condition-specific rules are encoded separately from drug-drug rules:**
These represent different classes of risk. A drug-drug interaction requires both drugs to be present. A drug-condition contraindication exists independently of every other medication — an NSAID is dangerous in a patient with CKD whether or not they are on any other drug. Encoding them separately makes the rule set auditable: a reviewer can confirm that all relevant condition classes are covered without parsing drug pair combinations.

**Why the Beers Criteria is the primary source:**
The American Geriatrics Society Beers Criteria is the most widely accepted clinical standard for potentially inappropriate medication use in older adults. It is US-specific, updated approximately every 3 years, and grounded in systematic literature review. Using it as the primary source anchors the operator to an authoritative, versioned standard rather than ad hoc clinical judgment.

---

## 9. Acetaminophen — Cumulative Dose Tracking

The acetaminophen rule in §3 does not assess only the product being scanned. It requires summing total daily acetaminophen across all products the patient takes.

**Why:**
Acetaminophen is present in hundreds of combination OTC products — NyQuil, Excedrin, Theraflu, DayQuil, many "PM" formulations — under brand names that do not prominently disclose it. A caregiver who gives Excedrin Extra Strength (250mg acetaminophen per tablet) and then adds regular Tylenol (325-500mg) for additional pain relief may not realize they have combined acetaminophen sources. The safe daily limit for patients 65+ is 3,000mg/day — lower than the standard adult limit of 4,000mg/day due to reduced hepatic reserve.

**How `reference/brand-ingredient-map.md` supports this:**
The brand-ingredient map provides active ingredient lookups for common OTC brands. When a product name is provided without an ingredient list, the operator uses this file to identify hidden acetaminophen before making a decision. The map is a reference layer, not a decision layer — the decision is always made by `rules.md`.

**Failure mode prevented:**
Without cumulative tracking, every product scanned in isolation would appear safe. The overdose risk does not live in any single product — it lives in the combination. This rule exists because the most common acetaminophen overdose pattern in elderly patients is inadvertent polypharmacy across multiple products, not a single excess dose.

---

## 10. Scope Restriction as a Reliability Mechanism

`identity.md` defines what SentinelRx will not do:
- Answer general drug information questions
- Assess prescription medications
- Advise on dosing changes
- Operate without a loaded patient profile

**Why scope restriction is a safety feature, not a limitation:**
A system that answers anything can be confidently wrong about anything. A system with a defined and enforced scope can be trusted within that scope precisely because it refuses to operate outside it. When SentinelRx declines a question, it is not failing — it is correctly identifying that the question is outside the domain where its rules are reliable.

**The alternative:**
A general medical assistant that attempts to answer OTC safety questions without a patient profile, without the Beers Criteria ruleset, and without the structured output format produces responses that are plausible but not trustworthy. Plausible-but-untrustworthy is the most dangerous failure mode in medical decision support. Scope restriction prevents it.

---

## 11. Tone Constraints — Why Hedging Is Prohibited

`identity.md` specifies: "Never hedge. No 'you might want to consider' or 'it could potentially.'"

**Why:**
Hedge language transfers the decision burden back to the caregiver. A caregiver without medical training who receives "this medication could potentially interact with warfarin in some cases" has received no actionable information. They cannot evaluate "could potentially" or "in some cases." They are in exactly the same position they were in before asking.

The operator is designed to absorb that decision burden. When a rule applies, the output states it directly. When a rule does not apply cleanly, the output escalates to a pharmacist. There is no middle state where the caregiver is left to weigh probabilistic language against their own judgment.

**What this requires of the rule set:**
Prohibiting hedge language only works if the rule set is comprehensive enough to produce a definitive answer for the vast majority of cases, and if the escalation logic is robust enough to catch the cases where a definitive answer is not safe. Both conditions are satisfied by the combination of §3 (drug class rules), §4 (escalation conditions), and §5 (tiebreaker).

---

## 12. Known Failure Modes — Explicit Documentation

`rules.md §8` documents four known failure modes that the operator cannot fully prevent:

1. **Stale profile after hospitalization** — The date threshold cannot detect same-day changes. Addressed by the unconditional hospitalization prompt, but not fully solved.
2. **Combination product overload** — Products with more than 4 active ingredients may contain unrecognized ingredients. Addressed by §4 escalation.
3. **Supplement label gaps** — Active ingredient amounts on supplement labels are frequently understated or absent. Addressed by escalation when amounts are missing, but undetectable if the label simply omits the ingredient.
4. **Brand name without ingredient list** — A brand name alone is insufficient to make a decision. Addressed by requiring ingredient-level information before proceeding.

**Why documenting failure modes is a design decision:**
A system that claims to handle all cases is either lying or doesn't know its own limits. Documenting failure modes explicitly serves three purposes: it forces the design to account for them, it communicates them to the user so they can compensate (update the profile, read the full label), and it establishes the system's epistemic honesty — it knows what it doesn't know.

---

## 13. The Automatic Audit Trail

Every scan appends a structured log entry to `audit/scan-log.md` automatically, without user action. The entry records: date and time, patient name, product name, active ingredients, routing decision, one-line reason, and the profile's last-updated date at the time of the scan.

**Why automatic and not manual:**
A manual log depends on the caregiver completing an additional step after receiving the decision. Under real-world conditions — a caregiver in an aisle, in a hurry, with a patient waiting — that step will be skipped. Automating the log means the record is complete regardless of caregiver behavior.

**Why the profile date is recorded in each log entry:**
The profile date in the log entry captures what the system knew at the time of the decision. If the profile is later updated (because a prescription changed), the historical log entries still reflect the information state at the time each decision was made. This makes the audit log a verifiable record — not just what was decided, but what was known when it was decided.

**Why this matters beyond convenience:**
The audit log is a longitudinal care record. A physician reviewing the patient's care can see every OTC check ever performed, what the system recommended, and whether the profile was current at the time. This turns a point-in-time safety tool into a continuous documentation system — one that can surface patterns (repeated scanning of the same product class, decisions made against a stale profile) that no single scan would reveal.

---

## 14. The §6 Age Adjustment

When a product label shows standard adult dosing only (no guidance for patients 65+), the output appends: "Standard adult dosing only. For patients over 80, consider starting at half dose unless physician has specified otherwise."

**Why this rule exists:**
Standard adult dosing in clinical trials is derived from populations that skew younger and typically exclude patients over 75. Pharmacokinetics change significantly with age — reduced renal clearance, reduced hepatic metabolism, lower body weight, altered drug distribution. A standard adult dose may be a high dose for an 85-year-old. The rule flags this gap explicitly rather than assuming the label's adult dosing is appropriate for the patient.

**Why the threshold is 80 and not 65:**
The Beers Criteria applies broadly to adults 65+. The age adjustment rule targets a sub-population within that group — patients over 80 — where physiological changes are more pronounced and evidence for standard adult dosing is thinner. It is a layer on top of the existing Beers-based rules, not a replacement for them.

---

## 15. Summary of Design Principles

The following principles are embedded across the entire system and should be understood as a coherent whole:

**Determinism over flexibility.** Rules decide. AI parses. The two roles do not overlap.

**Explicit escalation over partial confidence.** When the system cannot be reliably confident, it says so and routes to a professional. It does not produce a best-guess answer.

**Two mechanisms where one can fail.** The staleness protocol uses both a date threshold and an unconditional hospitalization prompt because neither alone is sufficient.

**Scope restriction as trust-building.** The operator's reliability within its domain depends on its refusal to operate outside it.

**Failure modes acknowledged, not hidden.** The system documents what it cannot catch. This is the basis for informed use, not a concession of weakness.

**Every output is actionable.** The tone constraints, the four-tier routing, and the alternative-suggestion requirement all exist to ensure the caregiver can act on the decision immediately, without additional interpretation.

---

*SentinelRx · XVI Health Intelligence · 2026*
*Decision logic grounded in: American Geriatrics Society Beers Criteria® 2023*
*US only. Not a substitute for pharmacist or physician consultation.*
