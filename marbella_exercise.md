# Dental Benefits Verification Agent

**1. The job.** Insurance verification at a 4-dentist, ~12-person general dental clinic. Each day someone checks tomorrow's insured patients: is coverage active, annual maximum remaining, deductible met, frequency limits, coverage % for the planned procedure. They then type the result into the practice-management system (PMS). Today the office manager or a part-time coordinator does it via payer portals and phone holds: ~75 patients/week × ~12 min ≈ 15 hrs/week. At ~$24/hr + 25% burden = $30/hr, that is $450/week ≈ **$1,950/month**. (Assumptions to confirm on discovery calls.)

**2. The agent.**
Trigger: nightly run, plus any appointment booked within 48 hours.

1. Pull the next 5 days of appointments, insurance and planned procedure (CDT) codes from the PMS (Open Dental API first; Dentrix/Eaglesoft via export).
2. Query eligibility via a clearinghouse API (270/271, e.g. Stedi or Availity); if a payer has none, a browser agent uses its portal; last resort, an AI voice call.
3. An LLM structures the response: status, max remaining, deductible, frequency limits, waiting periods, coverage % per CDT code.
4. Compute the patient's estimate and write it into the PMS in the clinic's note format.
5. Exceptions go to a queue; the office manager gets a morning digest ("41 verified, 3 need a human").

**What happens when it's wrong.** Every written field cites its source (payer response + timestamp). Deterministic checks run first (DOB, member ID, plan dates). Low confidence, a payer response that contradicts the prior record, or an unreachable payer goes to a human. It never writes silently. Weekly, I compare estimates with what the payer actually paid (835 remittance) and review every miss. Early on, staff spot-check 5 random verifications weekly. Guarantee: a denial caused by our error makes that month free.

```
 PMS schedule ──► [Nightly trigger] ──► Clearinghouse API ──┐
 (tomorrow's pts)                       Portal bot ─────────┼─► LLM parse ─► Rule checks
                                        Voice call (last) ──┘        │            │
                                                              pass ◄─┴─► fail/low confidence
                                                                │            │
                                                     Write to PMS      Human exception queue
                                                                │            │
                                                     Morning digest ◄────────┘
                                                                │
                                          Weekly: compare to 835 claim result ─► fix errors
```

**3. Pricing.** **$600/month flat** per clinic (~150 verifications).

| | Per month |
|---|---|
| Human today | $1,950 |
| Agent $600 + ~3 hrs/week of human exception handling (~$390) | ~$990 |
| **Saved** | **~$960 (~50%)** |

Nobody gets fired: the coordinator moves to collections and treatment-plan follow-up, which earn money. My cost is roughly $0.30–0.60 per verification (clearinghouse + LLM), ~$60/month, so ~90% gross margin. The price sits between the median agent ($140) and AiSDR (~$1,000) because it replaces a defined job at under a third of its cost.

**4. First 10 customers.**
- **Buy signal:** independent practices (2–6 dentists) with an open "insurance coordinator" or "front office" posting on Indeed. They're paying for this job right now. Found via Google Maps + Apollo, starting in the Bay Area.
- **Communities:** Open Dental forums (easiest first integration), office-manager Facebook groups, local dental societies.
- **Offer to the first 3:** a free 30-day *shadow-mode* pilot. The agent verifies the same patients as their staff, and I hand over an accuracy report with every mismatch. After that, $300/month locked for 6 months in exchange for a case study and a weekly 15-minute feedback call.

**5. The hard part.** Payer data is messy (downgrades, shared maximums, waiting periods), and a confidently wrong estimate is worse than none. The hard part is *knowing when not to answer*, more than parsing.

**Test before charging:** (a) Under a BAA, backtest 200 past verifications from a friendly clinic against actual claim outcomes. Target ≥98% accuracy on the ~10 fields that drive the patient estimate, with zero silent errors (every miss must have been flagged). (b) Run two weeks of shadow mode at 3 clinics. Success: exception rate under 15% and staff stop re-checking. If either fails, I don't charge. HIPAA compliance (BAA, encryption, audit logs) is a gate, not a feature.
