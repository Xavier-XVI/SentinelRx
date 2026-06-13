# SentinelRx — Examples

Reference patient for all examples: Margaret Chen, 86F. See `patients/sample-patient.md`.
Active medications include: warfarin, lisinopril, levothyroxine, metoprolol, furosemide, atorvastatin, metformin, digoxin.

---

## Example 1 — The PM Suffix Trap

**Input:** Photo of Tylenol PM box. Caregiver asks: "Can she take this for nighttime pain?"

**Active ingredients read from label:** Acetaminophen 500mg, Diphenhydramine HCl 25mg

**Rules triggered:** §3 Diphenhydramine — patient is 65+

```
DECISION: 🚫 DO NOT TAKE

PRODUCT: Tylenol PM
ACTIVE INGREDIENTS: Acetaminophen 500mg, Diphenhydramine HCl 25mg

REASON: Diphenhydramine (the "PM" sleep component) is a Beers Criteria Class I risk for all patients 65+. It causes confusion, fall risk, and urinary retention. No dose is safe for this patient's age group.

ACTION: The pain component (acetaminophen 500mg) is safe on its own. Use regular Tylenol instead. For sleep, consult her physician — low-dose melatonin (0.5–1mg) may be appropriate if no sedative is already prescribed.

If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision.
---
Decision based on active patient profile and FDA Beers Criteria 2023. When in doubt, call your pharmacist.
```

---

## Example 2 — The "Natural" Supplement Trap

**Input:** Caregiver considers giving high-dose omega-3 fish oil capsules (1200mg) purchased at a warehouse store.

**Active ingredients:** Omega-3 fatty acids 1200mg (EPA + DHA)

**Rules triggered:** §3 Supplements — patient is on warfarin

```
DECISION: 🚫 DO NOT TAKE

PRODUCT: Fish Oil 1200mg (Omega-3)
ACTIVE INGREDIENTS: Omega-3 fatty acids 1200mg (EPA + DHA)

REASON: Fish oil at this dose has anticoagulant properties. Margaret is on warfarin. Combining them increases bleeding risk and can elevate INR unpredictably, which warfarin's monitoring is calibrated to avoid.

ACTION: Do not administer without physician or pharmacist sign-off. If fish oil is being used for cardiovascular benefit, her cardiologist should weigh in — a lower dose with INR monitoring adjustment may be possible.

If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision.
---
Decision based on active patient profile and FDA Beers Criteria 2023. When in doubt, call your pharmacist.
```

---

## Example 3 — Timing Matters (Not All Cautions Are Absolute)

**Input:** Caregiver asks about Tums (calcium carbonate) for heartburn.

**Active ingredients:** Calcium carbonate 750mg

**Rules triggered:** §3 Antacids — patient is on levothyroxine

```
DECISION: ⚠️ TAKE WITH CAUTION

PRODUCT: Tums Extra Strength
ACTIVE INGREDIENTS: Calcium carbonate 750mg

REASON: Margaret takes levothyroxine every morning. Calcium carbonate binds to thyroid medication in the GI tract and significantly impairs absorption. This is a timing interaction, not a hard contraindication.

ACTION: Do not take Tums within 4 hours of her morning levothyroxine dose. Afternoon or evening administration is safe. A standard 1–2 tablet dose is appropriate for her age.

Standard adult dosing only. For patients over 80, consider starting at half dose unless physician has specified otherwise.

If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision.
---
Decision based on active patient profile and FDA Beers Criteria 2023. When in doubt, call your pharmacist.
```

---

## Example 4 — Hidden Acetaminophen Accumulation

**Input:** Margaret already takes Excedrin Extra Strength for headaches (prescribed PRN). Caregiver wants to add regular Tylenol for back pain the same afternoon.

**Active ingredients (Excedrin Extra Strength, from brand-ingredient-map.md):** Acetaminophen 250mg, Aspirin 250mg, Caffeine 65mg
**Active ingredients (Tylenol Regular Strength):** Acetaminophen 325mg

**Rules triggered:** §3 Acetaminophen total dose rule; §3 NSAIDs (aspirin)

```
DECISION: 🚫 DO NOT TAKE

PRODUCT: Tylenol Regular Strength (added to existing Excedrin)
ACTIVE INGREDIENTS: Acetaminophen 325mg

REASON: Excedrin Extra Strength already contains 250mg acetaminophen per tablet. If Margaret has taken 2 Excedrin tablets today (500mg acetaminophen), adding one Tylenol Regular Strength (325mg) brings the total to 825mg from this combination alone. Across a full day with multiple doses, this risks exceeding the 3,000mg/day safe limit for patients over 65. Additionally, Excedrin contains aspirin — a NSAID — which carries independent risk for patients on warfarin (see §3 NSAIDs).

ACTION: Do not add Tylenol to an Excedrin dose. For ongoing pain, ask her physician to specify a single analgesic with a defined daily limit — not a combination product.

If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision.
---
Decision based on active patient profile and FDA Beers Criteria 2023. When in doubt, call your pharmacist.
```

---

## Example 5 — Stale Profile After Hospitalization (Highest-Risk Scenario)

**Context:** Margaret was hospitalized 3 days ago for a cardiac event. Her physician added apixaban, increased furosemide, and discontinued warfarin. The caregiver has not yet updated `patients/sample-patient.md`. The profile still lists warfarin as active. Last updated date on file: 22 days ago.

**Input:** Caregiver photographs Advil (ibuprofen 200mg).

**What the operator does with the stale profile:**
- Profile shows warfarin → operator correctly routes to 🚫 DO NOT TAKE (NSAIDs + anticoagulant)
- The reason is still correct (apixaban has the same NSAID interaction), but the listed drug is wrong
- Profile age = 22 days → exceeds 14-day threshold → staleness warning fires

```
DECISION: 🚫 DO NOT TAKE

PRODUCT: Advil
ACTIVE INGREDIENTS: Ibuprofen 200mg

REASON: Margaret is on warfarin (per stored profile). NSAIDs combined with anticoagulants carry a major bleeding risk and can destabilize INR.

ACTION: Use acetaminophen (regular Tylenol) instead for pain. Do not administer any NSAID while on anticoagulant therapy.

⚠️ PROFILE STATUS: Last updated 22 days ago — exceeds the 14-day check interval. Verify that all medications on file are still current.

If the patient had a physician visit, ER visit, or hospitalization since this profile was last updated, verify all prescriptions are current before acting on this decision.
---
Decision based on active patient profile and FDA Beers Criteria 2023. When in doubt, call your pharmacist.
```

**Why this is the highest-risk scenario:** The decision (DO NOT TAKE) happens to be correct, but for the wrong drug. If the hospitalization had *added* a new medication that creates a new interaction — not replaced one — the operator would miss it entirely. The staleness warning is the only defense. This is why updating the profile immediately after any hospitalization is non-negotiable.
