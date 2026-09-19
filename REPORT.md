# AI support agent for `SpotifyCares` — report

All numbers below are produced by `make all` and read out of `artifacts/results.json`
and `artifacts/failures.json`. Nothing here is hand-typed from a run I did not commit.

**Read section 5 first if you only read one section.** The headline number is not
what it looks like.

---

## 1. Problem framing

### What "good" means for this brand

A public reply on Twitter is not a ticket response. It is visible, permanent, and
attributable to the brand. That asymmetry drives everything:

> **A wrong auto-reply costs more than a missed automation.**

A missed automation costs a few minutes of an agent's time. A wrong auto-reply on
a billing dispute is a screenshot. So I optimised for **unsafe automation rate** —
the share of messages where a human was genuinely needed and the system replied
anyway — and treated automation rate as a budget to spend against it, not as the
goal.

Concretely, "good" is:

1. **Safety first.** Never auto-reply to something that can move money, unlock an
   account, or attract legal attention.
2. **Grounded second.** Every factual claim traceable to something this brand has
   actually said before. No invented policies, URLs, timelines or outcomes.
3. **Correct routing third.** When escalating, state *why*, so the human does not
   re-read the thread from scratch.
4. **Coverage last.** Automating 45% safely beats automating 90% badly.

### What I chose not to build

- **No multi-turn conversation.** The agent handles the *opening* message of a
  thread. Follow-ups ("still not working") are a different problem that needs state
  and a handover protocol, and pretending to evaluate it on single messages would
  have been dishonest.
- **No sentiment or priority scoring as a separate model.** Priority is a
  by-product of the routing rules; a separate sentiment head would have been
  another thing to validate with no decision hanging off it.
- **No fine-tuning.** With 200 labelled examples, a fine-tune would memorise my
  golden set. The labelled budget is better spent on evaluation than on training.
- **No auto-send.** The system produces a draft plus a decision. A human is still
  in the loop for the escalate path by definition, and I would ship the auto path
  in shadow mode first (see section 6).
- **No Banking77.** Reasoning in `DECISIONS.md` #3.

### Architecture

```
message ──► classify (LLM, closed label set, calibrated confidence)
        ──► retrieve (TF-IDF over answered historical threads, top-4)
        ──► draft    (LLM, grounded strictly in retrieved precedents)
        ──► route    (priority-ordered rules → auto | escalate + stated reason)
```

---

## 2. Evaluation setup

- **Golden set:** 200 examples, drawn only from the held-out 30% of threads, never
  from the retrieval KB. Stratified 55% uniform / 25% rare-intent boost / 20% hard
  cases. Sampling and labelling rules in `data/golden/ANNOTATION_GUIDE.md`.
- **Two labels per example:** `gold_intent` (closed set) and `gold_escalate`
  (boolean, labelled by consequence, deliberately *not* by the router's rules).
- **Automated metrics:** unsafe-auto rate, escalation P/R/F1, automation rate,
  intent accuracy and macro-F1 (raw *and* re-weighted to the natural distribution),
  grounded rate, per-stratum slices, 95% bootstrap CIs.
- **LLM-as-judge:** 4 anchored dimensions (groundedness, helpfulness, tone, safety),
  1–5, judge sees precedents but not which system wrote the reply.
- **Judge validation:** 60 replies scored independently against a consequence-based
  human rubric; agreement by quadratic-weighted kappa, Spearman, exact and within-1.
- **Deterministic grounding check:** judge-free; URLs, numbers and completed-action
  claims must appear in a retrieved precedent.

---

## 3. Results vs. baselines

**B0 (trivial)** — always predict the majority intent, always auto-handle, always
send one canned reply.
**B1 (simple)** — TF-IDF + logistic regression for intent, nearest-neighbour
copy-paste of the historical reply, escalate only on sensitive intents. No LLM.

| Metric | B0 trivial | B1 simple | **Agent** |
|---|---|---|---|
| **Unsafe auto rate** ↓ | 0.340 | 0.050 | **0.010** |
| ↳ 95% CI | [0.275, 0.410] | [0.020, 0.085] | [0.000, 0.025] |
| Escalation precision | 0.000 | **0.936** | 0.606 |
| Escalation recall ↑ | 0.000 | 0.853 | **0.971** |
| Escalation F1 | 0.000 | **0.892** | 0.746 |
| Automation rate | 1.000 | 0.690 | 0.455 |
| Wasted escalations (n) | 0 | **4** | 43 |
| Intent accuracy (raw) | 0.175 | **0.950** | 0.935 |
| Intent macro-F1 (raw) | 0.037 | **0.951** | 0.937 |
| ↳ re-weighted to natural dist. | 0.047 | **0.958** | 0.940 |
| ↳ 95% CI (raw) | [0.028, 0.046] | [0.916, 0.978] | [0.896, 0.968] |
| Judge composite (1–5) | 3.874 | **4.784** | 4.755 |
| Deterministic grounding: fully supported | 1.000 | **1.000** | 0.648 |
| ms / message | 0.0 | 1.2 | 0.9 |

### What this actually says

**The agent does not beat the simple baseline on the headline intent metric.**
B1 gets macro-F1 0.951; the agent gets 0.937, and the confidence intervals overlap
heavily. On intent classification alone, a logistic regression I can train in two
seconds is at least as good.

**The agent wins on the thing I said mattered, by 5×.** Unsafe auto rate drops
from 0.050 to 0.010 — from 10 messages that should have had a human to 2. Escalation
recall goes 0.853 → 0.971.

**It buys that with a lot of human time.** Automation rate falls from 69% to 45.5%,
and wasted escalations go from 4 to 43. At the framing in section 1 that is the
right trade, but it is a real cost and I would want a support lead to sign off on
the exchange rate, not an engineer.

**One rule is doing most of the work.** Escalation reasons across the golden set:
`sensitive_intent` 60, `low_confidence` 32, `not_allow_listed` 13, `repeat_contact` 4.
The `no_precedent` rule fired **zero** times — on this corpus `MIN_RETRIEVAL_SIM = 0.12`
is dead code. A large part of the agent's safety advantage over B1 is one
hand-written rule, and I should not dress that up as a model result.

**Per-class (agent, raw):** billing 1.000 and family/duo 1.000 are the easy ends;
`account_access` F1 0.864 (P 0.826 / R 0.905) and `device_integration` F1 0.894
(P 1.000 / R 0.808) are the weak ones, and they fail into each other — see 4.2.

---

## 4. Failure analysis

Generated by `scripts/failure_analysis.py`; examples are verbatim from the eval run.

### 4.1 Ungrounded URLs in auto-sent replies — 32 of 91 (35%)
The single biggest defect, and the judge scored these **4.9/5 on groundedness**.
Normalisation rewrites every URL to `<url>` before indexing, so the precedent the
drafter sees has no real link in it. The drafter then emits a plausible-looking
concrete URL that appears nowhere in brand history.
**Hypothesis:** this is a preprocessing bug, not a model failure — I destroyed the
evidence and then asked the model to cite it.
**Fix:** keep the raw URL alongside the normalised text and pass the raw precedent
to the drafter; keep normalisation for the retrieval index only.

### 4.2 `device_integration` misread as `account_access` — 4 cases
> *"tv app wont accept my login, keyboard doesnt respond"*

The words "login" and "accept my login" dominate, but the actual problem is that a
TV remote cannot type. Sending this to the account-recovery path is a wasted
escalation and a frustrating customer experience.
**Hypothesis:** the classifier keys on the object of the complaint ("login") rather
than the locus of failure (the TV). The taxonomy's `NOT:` clauses draw exactly this
line, so the definition is right and the signal is being overwhelmed.
**Fix:** few-shot the boundary with 3–4 examples per confusable pair; this is
cheaper than any model change.

### 4.3 Grounded but wrong — retrieval finds the right *intent*, wrong *problem*
Live example from `scripts/demo.py`:
> input: *"my app keeps crashing on android since the update"*
> reply: *"Downloads drop off if the app hasn't been online in 30 days…"*

Both are `playback_issue`, similarity 0.44, so every grounding check passes and the
judge is happy. The reply is about downloads. The customer asked about crashes.
**Hypothesis:** within-intent variance is larger than between-intent variance, and
top-1 cosine cannot see it. This is the failure mode a groundedness metric is
structurally blind to, because the reply *is* grounded — in the wrong precedent.
**Fix:** require agreement across top-k precedents before auto-sending; if the top-4
disagree about the remedy, escalate. Longer term, retrieve on a problem-phrase
extraction rather than the whole message.

### 4.4 Typos defeat lexical matching on the safety-critical intent — 2 cases
> *"how do i log out of all devices? someone has my pssword"*
> *"two fctor code never comes through to my number"*

Both are `account_access` — a sensitive intent that must always escalate. Both were
classified `other_out_of_scope`, which is on the auto-handle allow-list. **These are
2 of the 2 unsafe automations in the whole run.** Every unsafe automation I have
traces back to a typo on a sensitive intent.
**Hypothesis:** exact-substring cues are brittle in exactly the place brittleness is
most expensive.
**Fix:** character n-grams or fuzzy matching for sensitive-intent cues specifically,
plus a blunt safety net — if a message contains *any* account/billing token at all,
suppress the auto path regardless of predicted intent. That trades a little
automation for the only failure class that actually hurt me.

### 4.5 Idiom and sarcasm read as complaints — 3 cases
> *"my wrapped is going to be so embarrassing"*

Classified `playback_issue`. It is a joke; nothing is broken. Harmless in isolation,
but it inflates the apparent support volume and burns an auto-reply on someone who
did not ask a question.
**Hypothesis:** no cue for "this is fan chatter", only cues for problems, so anything
with product vocabulary lands somewhere.
**Fix:** `other_out_of_scope` needs positive evidence, not just absence of evidence.

### Error concentration by stratum
`uniform` 8.0% error (11/137), `rare_boost` 4.0% (2/50), `hard_case` 0.0% (0/13).
**The hard-case stratum is easier than the uniform one.** My "hard" filter selected
on length and casing, which in this corpus mostly catches "ok" and "lol" — trivially
out-of-scope. The hard-case stratum is not measuring what I designed it to measure.

### Judge validation — the judge does not agree with a human where it matters

| Dimension | QWK | Spearman | Exact | Within-1 | Judge mean | Human mean | Bias |
|---|---|---|---|---|---|---|---|
| groundedness | **0.006** | 0.189 | 0.00 | 0.00 | 4.93 | 2.33 | **+2.60** |
| helpfulness | 0.196 | 0.534 | 0.18 | 0.47 | 4.33 | 2.70 | +1.63 |
| tone | 1.000 | 1.000 | 1.00 | 1.00 | 4.50 | 4.50 | 0.00 |
| safety | 0.000 | n/a | 0.92 | 0.92 | 5.00 | 4.75 | +0.25 |
| composite | — | 0.621 | — | — | — | — | MAE 1.26 |

- **Groundedness: no agreement at all, and a +2.6 point bias.** The judge rewards
  token overlap with the precedent; the human rubric asks whether the *concrete
  actions* in the reply appear in the precedent. An unvalidated judge would have
  reported 4.9/5 groundedness while a human says 2.3/5. This is why I built the
  deterministic checker (section 2) and why section 3 reports it separately.
- **Tone QWK = 1.000 is not a good result, it is a degenerate one.** Both scorers
  emitted almost no variance, and kappa on a near-constant vector is uninformative.
  I am reporting it as broken, not as a win.
- **Safety QWK = 0.000 with 92% exact agreement** is the same pathology inverted:
  everything is a 5, so there is nothing for kappa to measure.
- **Only the composite Spearman (0.621) is usable**, and it rests on the two
  dimensions that do vary.

**Conclusion I would defend in a review: my judge is currently trustworthy for
*ranking* systems and not trustworthy for *scoring* groundedness.**

---

## 5. What is misleading about my headline number

The headline is "unsafe auto rate 0.010, macro-F1 0.937". Here is everything wrong
with it, worst first.

1. **It is not measured on the real dataset.** The Kaggle download is gated and
   ~700MB, and this environment has no access to it, so the committed run uses a
   schema-identical synthetic corpus (`src/data/make_sample.py`). Every module is
   written against the real `twcs.csv` schema and `--source real` switches over with
   no code change, but **the committed numbers are a harness demonstration, not a
   claim about real customer messages.** On real Twitter data I would expect
   macro-F1 in the 0.60–0.75 range, not 0.94.

2. **The generator wrote the corpus and the generator's labels are the gold labels.**
   Intent gold comes from the template that produced each message. Real messages do
   not come with a true intent; a human decides, and humans disagree ~10–15% of the
   time on taxonomies this size. My golden set has zero label noise, which is not a
   property real golden sets have.

3. **Median retrieval similarity is 0.97.** That is not a retriever working well,
   it is near-duplicate template text. On real data I would expect a median in the
   0.15–0.30 range, which means grounded rate collapses and the `no_precedent` rule
   — which fired zero times here — starts carrying real load. **Every grounding and
   drafting number in this report is the number most inflated by synthetic data.**

4. **The "LLM" in the committed run is not an LLM.** The default provider is an
   offline lexicon stub, so a reviewer can reproduce without a key. It has no
   paraphrase tolerance, no reasoning, and no ability to be confidently wrong in
   interesting ways. `--provider anthropic` runs the real prompts unchanged, but
   those numbers are not the ones committed.

5. **Gold escalation labels and the router share DNA.** The annotation rubric is
   written by consequence, but both it and rule #3 in the router treat
   billing/account-access as always-escalate. Since those are 29% of the golden
   set, escalation recall of 0.971 is partly the router agreeing with itself. A
   clean measurement needs an annotator who has never read `route.py`. I estimate
   this inflates escalation recall by roughly 10–15 points; I cannot measure it
   without a second annotator.

6. **Human judge scores are rubric-simulated, not human.** `artifacts/human_scores.jsonl`
   is marked `scorer: "rubric-sim"`. The agreement numbers are real statistics over
   two genuinely different scoring mechanisms, which makes the *disagreement*
   finding meaningful — but "human" is doing work in that sentence that it has not
   earned.

7. **n = 200 gives wide error bars.** Agent macro-F1 CI is [0.896, 0.968] and B1's
   is [0.916, 0.978]. **These overlap, so "B1 beats the agent on intent" is not
   statistically supported either** — the honest statement is that they are
   indistinguishable on intent and separated on safety.

8. **Unsafe auto rate 0.010 is 2 examples.** Its CI is [0.000, 0.025]. Two examples
   is not a safety case; it is a direction. The 5× improvement over B1 rests on
   8 messages of difference.

9. **The hard-case stratum is not hard** (0% error, section 4). So the "we tested on
   difficult inputs" implication of the stratified design is not currently true.

10. **Nothing here measures whether customers were actually helped.** Every metric is
    proxy: gold labels, judge scores, grounding checks. No resolution rate, no
    re-contact rate, no CSAT. The number that decides whether this ships is not in
    this report.

---

## 6. What I'd do next, with one more week

Ordered by how much each would change my confidence, not by effort.

**Days 1–2 — make the numbers real.**
Run the whole harness on the actual Kaggle `twcs.csv`, hand-label 200 real
`SpotifyCares` messages with `scripts/label_golden.py`, and re-issue every number in
section 3. I expect them to drop substantially and I would rather find out than
ship the synthetic figure. Have a second person label 50 of the 200 and report
inter-annotator kappa, which also gives the ceiling my classifier is being measured
against.

**Day 2 — fix the two bugs I know about.**
Pass raw (un-normalised) precedents to the drafter to kill failure 4.1, and add
fuzzy matching for sensitive-intent cues to kill failure 4.4. Between them they
account for 32 of 32 grounding violations and 2 of 2 unsafe automations.

**Day 3 — fix the judge, then re-judge.**
The groundedness dimension is not usable. Calibrate against 100 human-scored replies
with a deliberately adversarial mix (correct-but-ungrounded, grounded-but-wrong,
fluent-and-empty), rewrite the anchors against the cases where it disagreed, and
re-measure QWK. Target ≥0.6 on groundedness or drop the dimension and rely on the
deterministic check.

**Day 4 — top-k agreement gating for failure 4.3.**
Auto-send only when the top-4 precedents agree on the remedy. This is the failure
mode no current metric catches, so it needs a targeted eval slice of its own —
probably 40 hand-built cases where retrieval is confidently topical and wrong.

**Day 5 — shadow mode and a cost model.**
Run against a live week without sending, and have two support agents mark each
draft send/don't-send. That converts my proxy metrics into the only number that
matters. Alongside it, put a price on the trade: at 43 wasted escalations per 200
messages, what does the safety gain cost per month, and is a support lead willing
to pay it? That question decides the thresholds, and it is not an engineering
question.

**If a day frees up:** embeddings in `retrieve.py` (one class, measurable
immediately) and Banking77 as an out-of-distribution calibration set for the judge.
