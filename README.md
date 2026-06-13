# SentinelRx — OTC Safety Operator for Elderly Patients

**[→ Open the full guide (with examples and setup instructions)](README.html)**

---

> ## ⚠️ UNITED STATES ONLY
>
> SentinelRx is built exclusively on **US regulations** — the FDA Beers Criteria (2023) and US drug classifications. Drug names, available formulations, approved ingredients, and interaction risks **differ significantly between countries**.
>
> **Do not use SentinelRx for patients outside the United States.** It will not produce reliable decisions for any other country's medications or regulatory context.

---

## What Is SentinelRx?

SentinelRx checks whether an over-the-counter product is safe for your specific patient before they take it.

You show it a product label. It tells you what to do — in plain language, every time.

---

## What You Get

Every scan returns one of four answers:

| Answer | What it means |
|---|---|
| ✅ **SAFE TO TAKE** | No known problem with current medications. OK to give. |
| ⚠️ **TAKE WITH CAUTION** | There is a concern, but it can be managed. Specific instruction given. |
| 🚫 **DO NOT TAKE** | Clear risk. Do not give. A safer alternative is suggested. |
| 📞 **CALL PHARMACIST FIRST** | Too complex to decide without a professional. Call before giving. |

---

## How to Use It

**When you're at the pharmacy or at home with a product:**

1. Look at the back of the box or bottle — find the section that says **Active Ingredients**
2. Tell SentinelRx what the product is and read out the active ingredients (or share a photo of the label)
3. Read the decision — it will tell you exactly what to do

That's it. No medical knowledge needed. No guessing.

---

## After Every Scan — Automatic Record Keeping

SentinelRx automatically writes a log entry to `audit/scan-log.md` after every scan. No copy-pasting required. This builds a human-readable history of every check — what was given, when, and why it was safe or not.

This record can be shared with a doctor or pharmacist if ever needed.

---

## Keep the Patient Profile Current

SentinelRx is only as accurate as the patient profile stored in `patients/`. **Update it immediately after:**

- Any prescription is added, changed, or stopped
- Any doctor or pharmacist visit
- Any hospital stay or emergency visit

If the profile is outdated, SentinelRx will warn you — but the safest habit is to update it the same day anything changes.

---

## Limitations

- **United States only.** See disclaimer above.
- **One patient at a time.** SentinelRx is set up for a single patient. Managing multiple patients is not supported in this version.
- **OTC products only.** It does not assess prescription medications, injections, or medical devices.
- **Not a pharmacist.** SentinelRx applies established FDA safety rules. For anything unusual or complex, it will tell you to call a pharmacist — do so.
- **Label must be readable.** Blurry, cut-off, or missing ingredient lists will always return "Call Pharmacist First."

---

## 3 Products That Catch Most Caregivers Off Guard

| Product | Why it's dangerous | SentinelRx decision |
|---|---|---|
| **Tylenol PM / Advil PM** | Contains diphenhydramine — FDA-flagged as unsafe for anyone over 65 | 🚫 DO NOT TAKE |
| **Benadryl / ZzzQuil** | Same ingredient, same risk — "allergy" or "sleep" label doesn't change it | 🚫 DO NOT TAKE |
| **Aleve (naproxen)** | Dangerous with blood thinners and blood pressure medication | 🚫 DO NOT TAKE |

---

## Folder Contents

```
SentinelRx/
├── CLAUDE.md                         ← Starts the agent
├── identity.md                       ← Agent persona
├── rules.md                          ← Decision logic
├── examples.md                       ← Edge case examples
├── templates/
│   └── patient-profile-template.md  ← Fill this in for your patient
├── patients/
│   └── [your-patient].md            ← Active patient profile
├── reference/
│   ├── beers-criteria-summary.md    ← FDA Beers Criteria 2023
│   ├── risk-reference.md            ← OTC × medication interaction rules
│   └── brand-ingredient-map.md      ← Ingredient lookup for common brands
├── audit/
│   └── scan-log.md                  ← Scan history — written automatically after every scan
├── assets/
│   └── XVI_logo.png                 ← Place logo here
├── README.md                        ← This file
└── README.html                      ← Full guide with examples
```

---

*SentinelRx · XVI Health Intelligence · 2026*
*US only · Decision logic grounded in: American Geriatrics Society Beers Criteria® 2023*
