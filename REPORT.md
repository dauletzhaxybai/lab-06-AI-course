# REPORT — The Tulpar Scholarship screening agent

## 1. What was measured, and how

The supplied agent (`Qwen2.5-3B-Instruct`, greedy decoding, so deterministic) reads one
applicant profile and answers `shortlist` or `reject` with a score 0–100. The measurement
uses **matched pairs**: two synthetic profiles identical on every merit field (program, GPA,
exam score, household income bracket, achievements, age, consent) that differ in exactly one
thing that must not matter — the language group of the name (`name_language`), the gender
coding of the name (`gender_coded_name`), or the university tier (`university`). Two further
attributes are **noise floors**: `null_name` (a different name of the *same* language group
and gender coding) and `null_university` (a different university of the *same* tier). A real
attribute is read against its own floor, not against zero.

Run: `N_PAIRS = 40` per attribute, `SEED = 2026`, `edge_cases = False` (clean profiles), BARE
harness. Total: 200 pairs = 400 profiles = 400 model calls. Data is synthetic; no real person
appears.

## 2. What the bare agent did (Part 1)

**Flips** — pairs whose two members got different outcomes:

| attribute | flips | rate [95% CI] | score diffs | mean abs diff |
|---|---|---|---|---|
| name_language | 2/40 | 5.0% [1.4%, 16.5%] | 11/40 | 3.12 |
| gender_coded_name | 4/40 | 10.0% [4.0%, 23.1%] | 6/40 | 1.12 |
| university | 1/40 | 2.5% [0.4%, 12.9%] | 10/40 | 2.50 |
| null_name (floor) | 0/40 | 0.0% [0.0%, 8.8%] | 7/40 | 2.00 |
| null_university (floor) | 1/40 | 2.5% [0.4%, 12.9%] | 9/40 | 2.12 |

**Every real attribute's 95% interval overlaps its own floor.** On flips, this run does not
support a claim that any of the three attributes moved the shortlist decision above the noise
floor, and I do not make that claim.

**Direction and score gap** (first group minus the other):

| comparison | first favoured / other | McNemar p | score gap [95% CI] | sign p |
|---|---|---|---|---|
| kazakh vs others | 2 / 0 | 0.500 | +0.7 [-1.8, +3.8] | 1.000 |
| russian vs others | 0 / 1 | 1.000 | +0.2 [-1.7, +2.1] | 1.000 |
| english vs others | 0 / 1 | 1.000 | -1.0 [-4.2, +1.9] | 1.000 |
| male vs female coded | 0 / 4 | 0.125 | -0.4 [-1.2, +0.6] | 0.219 |
| **flagship vs regional** | 1 / 0 | 1.000 | **+2.5 [+1.1, +4.1]** | **0.002** |

Only one comparison is distinguishable from zero: the model scores **flagship applicants
+2.5 points above matched regional applicants** (95% CI excludes zero, sign p = 0.002). That
is a real, measured score shift — even though it rarely crosses the shortlist threshold
(1 flip in 40 pairs).

**Group rates** (share shortlisted, mean score): english 96.2% / 64.4; kazakh 96.4% / 62.5;
russian 92.3% / 62.7; female 90.0% / 61.9; male 80.0% / 61.5; flagship 92.5% / 63.2;
regional 90.0% / 60.8. The gender gap in shortlist share (10 points) has a wide interval at
n = 40 and does not reach significance on the paired test (4 discordant pairs, p = 0.125).

**Unparseable replies:** 0 of 400. Prediction 0 was 8–20; the prediction was too pessimistic
for this model under this prompt.

**Detectability:** with n = 40 the exact McNemar test at 0.05 needs at least 6 discordant
pairs one way (15% of pairs) to call a result significant; an effect moving fewer pairs cannot
be told from chance here. The absence of a measured effect is not proof of its absence.

## 3. Who is affected, and what the measured behaviour does to them

The applicants are synthetic, but the decision structure is that of a real scholarship. Three
groups bear the measured behaviour:

1. **Applicants from a regional university.** Matched on every merit field, they score on
   average 2.5 points lower than their flagship peers. At this scale that shift rarely flips a
   decision (1 in 40), but it is systematic: at a threshold, or combined with other factors, it
   would deny a scholarship to a candidate whose only difference is where they studied. A
   regional applicant cannot change that field without changing institutions; the criterion is
   not merit but context.
2. **Applicants whose name is coded for the other gender.** 4 of 40 pairs flipped (10%), and
   male-coded names were shortlisted less often than female-coded ones in the oriented count
   (0 first favoured, 4 other favoured). With 40 pairs this is not significant, but the
   direction is consistent with a gender effect the run is not powered to confirm.
3. **Every applicant, indirectly.** 89% of profiles were shortlisted: a system that says yes to
   almost everyone gives little information and shifts the real selection to whoever reads the
   borderline cases.

**Applicants whose application is refused or escalated by the harness** (Part 3/4) are a fourth
group: 34 of 400 profiles (8.5%) were sent to a person, and 5 of 6 hand-built bad requests were
refused before any model call. These applicants bear the cost of the control — a human delay —
in exchange for not being silently rejected by a parser error or a prompt injection.

## 4. Where my responsibility as developer lies

I am responsible for four things the measurement makes concrete:

1. **For the shift I measured and did not cause but did deploy.** The flagship/regional shift
   of +2.5 points is in the supplied model; my responsibility is what I do about it. I blinded
   the field (Part 2a) and measured again (Part 3): the shift is gone by construction, and I
   state that this is a harness effect, not a model fix.
2. **For the harm I did not measure.** With 40 pairs the run can only see effects moving ≥15%
   of pairs. A smaller real gender effect cannot be excluded. I am responsible for saying so
   rather than claiming fairness from a null result.
3. **For the refusals.** A refusal is a decision against an applicant. I checked the false-refusal
   rate: 0 of 400 clean profiles were refused by the rules. If that number were non-zero, the
   control would harm the people it is meant to protect.
4. **For the record.** Every decision in the submitted run is in `decisions.jsonl` with its
   outcome, reason code, controls fired, SHA-256 of the model's input and a timestamp. A
   decision that cannot be reconstructed afterwards is a decision nobody can be held to; the
   log is the part of the system that makes my responsibility enforceable.

The honest summary: the bare agent is not shown to treat matched applicants differently on
flips above its own noise floor, but it does carry a real, small score shift for university
tier, and the run is not powered to rule out a gender effect. The harness removes the measured
divergence by construction, at a measured cost (166 of 400 profiles changed score when name and
university were blinded; 34 of 400 sent to a human). The trade-off is stated, not hidden.
