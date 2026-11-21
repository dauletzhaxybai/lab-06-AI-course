# JUSTIFICATION — The harness design

## 1. What the system may do

The system may screen a scholarship application **for the purpose of scholarship screening**
(`purpose="scholarship_screening"`). It may read a profile in which the merit fields — program,
GPA, entrance exam score, household income bracket, achievements, age, and consent — are
present, and it may ask the model for a shortlist/reject decision with a score. It may return
that decision to the applicant.

It may **not** see the applicant's name or university: those fields are blinded (Part 2a),
because the measurement found a real score shift on university tier and non-zero flips on the
name attributes, and neither field is a merit criterion for a scholarship.

## 2. What the system must refuse

The harness refuses five classes of request, each with its own reason code, all before any
model call:

| reason code | what it refuses | where it comes from |
|---|---|---|
| `purpose_not_allowed` | a purpose outside `scholarship_screening` (e.g. rank by ethnicity) | built-in `refusal_reason` |
| `minor` | an applicant under 18 | built-in `refusal_reason` |
| `no_consent` | an applicant who did not consent to automated processing | built-in `refusal_reason` |
| `invalid_input` | out-of-range or wrong-type fields (GPA 9.9, exam score -5) | built-in `validate_profile` |
| `prompt_injection_suspected` | free text that tries to rewrite the model's instructions | my `my_extra_refusal` |

It must also **escalate** rather than decide silently when (a) the model's reply cannot be
parsed, (b) the score falls in the ambiguity band (45, 49) measured in Part 1, or (c) my
`my_extra_escalation` fires (score 40–44, or a decision with an unusably short reason). In
Part 3, 34 of 400 profiles (8.5%) were escalated; in Part 4, 5 of 6 hand-built bad requests
were refused, with 0 model calls each.

## 3. What record the system keeps

`decisions.jsonl` — one JSON line per call, written by `screen` when `log_path` is set. Each
record holds: `record_id`, `pid`, the SHA-256 of the text the model saw, the field names shown
and withheld, `model_id`, `prompt_version`, the harness configuration, the model's raw reply,
the parsed decision and score, the controls that fired, the outcome, the reason code, whether
it went to a human, and a timestamp. The log holds **field names, never field values**; the
model's raw reply may quote values the model was shown, and a blinded field is never shown.

The log is what makes a decision auditable afterwards: who was decided, why, by which control,
at what time. It is the accountability record the bare pipeline did not have.

## 4. Controls, principles and instruments

Every control is tied in code (Part 5 `CONTROL_REGISTER`) and in this document to a named
ethical principle and a named legal instrument with a provision. The main instrument is the
**Law of the Republic of Uzbekistan No. ZRU-547 of 2 July 2019 "On Personal Data"**, which
applies to this prototype: the profiles, although synthetic, are processed as personal data.
The principles of lawful purpose (Art. 5), accuracy (Art. 5), protection guarantees (Art. 27)
and confidentiality (Art. 28) bear directly on the controls.

The **Council of Europe Convention 108+ (2018)** and the **Council of Europe Framework
Convention on AI (2024)** are used as **benchmarks**: Uzbekistan is not a party to either, so
they do not bind this prototype, but their provisions on proportionality (108+ Art. 5(3)),
accountability (108+ Art. 10), human oversight (AI Convention Art. 14) and effective remedy
(AI Convention Art. 13) name the principles the controls implement. The **OECD Privacy
Guidelines (2013)** are likewise a benchmark (Data Quality para. 8, Accountability para. 14).
I read each instrument's text before citing it; the register records the provision, not a
summary.

## 5. Trade-offs accepted

1. **Blinding vs information.** Hiding name and university removes the channel for the measured
   divergence, but it also changes the prompt's form. In Part 3, **166 of 400 profiles changed
   score** (mean |delta| = 14.01 points) and 46 changed outcome. This is the price of blinding:
   the model's view of the applicant is no longer the one the supplied agent was tuned on. I
   accepted it because the removed fields are not merit criteria and the measured shift was
   real.
2. **Human workload vs silent decisions.** The band (45, 49) sent **34 of 400 profiles (8.5%)**
   to a person. Every escalated profile is work; every profile outside the band is a call nobody
   re-checks. I chose the band from the Part 1 score table (reject scores reach 45, shortlist
   scores start at 45), not a round number, and I state the actual share: 8.5%.
3. **Refusal strictness vs false refusals.** A refusal rule that is too broad harms the people
   it is meant to protect. I measured the false-refusal rate: **0 of 400 clean profiles** were
   refused by the rules. The rules fire only on malformed, non-consenting, minor,
   wrong-purpose or injection-bearing requests.
4. **Counterfactual check not run in the measured run.** `counterfactual_check` doubles the
   model calls per profile. It is left off in the 200-pair run and offered as an optional
   extension on 8 name-language pairs. The trade-off is explicit: the check catches name
   divergence at twice the cost, blinding catches it at the cost of a changed prompt. Both are
   documented; neither is hidden.
5. **Power of the measurement.** With 40 pairs the run detects only effects moving ≥15% of
   pairs one way. A smaller real gender effect cannot be excluded. I accepted this limit and
   said so, rather than claiming fairness from a null result.

## 6. AI-use declaration

I used an AI assistant (the chat model behind this session) for the following tasks: (a) to
draft and tighten the English wording of `REPORT.md`, `JUSTIFICATION.md`, the `CONTROL_REGISTER`
entries, and the `PREDICTION_*` and `OBSERVATION_*` texts; (b) to check that the structure of
the register matches the fields `check_register` expects; (c) to compare the numbers in the
Part 1, Part 3 and Part 4 outputs against the predictions and to suggest how to state the
comparison. I did **not** use the AI to produce the legal citations as authority: every citation
in the register and in this document is to an instrument I identified, and I read the provision
in the primary text before relying on it. The AI's suggestions about which article applies were
treated as pointers to check, not as citations. All measured numbers in this submission come
from the notebook's own runs (`OBSERVATION_1`, `OBSERVATION_3`, `OBSERVATION_4`) and from
`decisions.jsonl`, not from the AI. The code of the harness, the rules `my_extra_refusal` and
`my_extra_escalation`, and the choice of controls are mine.
