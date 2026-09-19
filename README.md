# AI support agent — `SpotifyCares` (Hiver SDE Intern take-home)

Classifies inbound customer messages, drafts a reply grounded in how the brand has
actually resolved similar issues, and decides auto-handle vs. escalate with a stated
reason — plus the evaluation harness that argues about whether it can be trusted.

**Start with [`REPORT.md` §5, "What is misleading about my headline number"](REPORT.md#5-what-is-misleading-about-my-headline-number).**
The short version: the committed run uses a synthetic stand-in corpus, so the
headline figures demonstrate the harness rather than make a claim about real
customer messages. Section 5 lists all ten reasons.

---

## Reproduce the headline results (~2 minutes, no API key, no downloads)

```bash
git clone <repo> && cd hiver-support-agent
pip install -r requirements.txt
make all
```

That runs: build corpus → build golden set → evaluate all three systems → judge →
judge-vs-human agreement → failure analysis. Output lands in `artifacts/`.

Expected final lines:

```
B0_trivial   macroF1=0.037 (rw 0.047)  unsafe_auto=0.340  auto_rate=1.000  judge=3.874
B1_simple    macroF1=0.951 (rw 0.958)  unsafe_auto=0.050  auto_rate=0.690  judge=4.784
agent        macroF1=0.937 (rw 0.940)  unsafe_auto=0.010  auto_rate=0.455  judge=4.755
```

Other entry points:

```bash
make test        # 12 invariant tests, ~1s
make demo        # run the agent on one message, see the full decision
python scripts/demo.py "i got charged twice this month"
```

## Run it on the real Kaggle data

```bash
# download thoughtvector/customer-support-on-twitter, unzip twcs.csv into data/raw/
python scripts/build_golden_set.py --source real --n 200   # stratified sample
python scripts/label_golden.py                             # hand-label (resumable)
python -m src.eval.run_eval --source real --provider anthropic
```

No code changes needed — the loader is written against the real `twcs.csv` schema
and the sample corpus is schema-identical. `--provider anthropic` needs
`ANTHROPIC_API_KEY`; `--provider stub` runs offline.

---

## What's here

| Path | What it is |
|---|---|
| `src/taxonomy.py` | The 8 intents, with explicit `NOT:` boundaries |
| `src/data/load.py` | Thread reconstruction, two-pass brand recovery, normalisation |
| `src/data/make_sample.py` | Synthetic stand-in corpus (same schema as `twcs.csv`) |
| `src/agent/classify.py` | Intent classification, closed label set, calibrated confidence |
| `src/agent/retrieve.py` | TF-IDF resolution index over answered historical threads |
| `src/agent/draft.py` | Grounded reply drafting |
| `src/agent/route.py` | **Escalation policy — priority-ordered rules, readable in 60s** |
| `src/baselines.py` | B0 trivial, B1 TF-IDF+LR (no LLM) |
| `src/eval/metrics.py` | Safety / routing / intent metrics, re-weighting, bootstrap CIs |
| `src/eval/judge.py` | LLM-as-judge, 4 anchored dimensions |
| `src/eval/agreement.py` | Quadratic-weighted kappa, Spearman, exact / within-1 |
| `src/eval/grounding_check.py` | Judge-free deterministic hallucination check |
| `data/golden/ANNOTATION_GUIDE.md` | Labelling rubric + sampling plan |
| `REPORT.md` | Framing, results, failure analysis, caveats, next steps |
| `DECISIONS.md` | 17 non-obvious decisions and why |

## Design in one paragraph

A public reply is permanent and attributable, so **a wrong auto-reply costs more
than a missed automation**. The system therefore optimises unsafe-automation rate
and spends automation rate to buy it. Intent classification and retrieval are kept
separate so a retrieval miss and a classification miss stay distinguishable in the
failure analysis. Escalation is deterministic rules over model output rather than a
second LLM call, because the safety valve needs to be inspectable, free, identical
every run, and editable by a support lead. Groundedness is measured twice — once by
the judge and once deterministically — because the judge turned out not to agree
with a human on that dimension at all.

## Headline finding

The agent **does not beat the simple TF-IDF+LR baseline on intent macro-F1**
(0.937 vs 0.951, overlapping CIs). It cuts unsafe automation 5× (0.050 → 0.010) and
buys that with automation rate (69% → 45.5%) and 43 wasted escalations per 200
messages. Under the framing above that is the right trade, but the exchange rate is
a support-lead decision, not an engineering one.

## Provider swap

`src/llm.py` has one interface method, `complete(system, user, json_mode)`. Two
providers ship: `anthropic` (real API) and `stub` (offline, deterministic — a
lexicon approximation used so this repo reproduces without a key; **stub numbers
are not LLM numbers**). An open model is a ~10-line addition.

## Borrowed / cited

- Dataset: [Customer Support on Twitter](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) (Kaggle, `thoughtvector`).
- Banking77 (`PolyAI/banking77`) was offered and **deliberately not used** — reasoning in `DECISIONS.md` #3.
- scikit-learn for TF-IDF and logistic regression; SciPy for Spearman.
- Quadratic-weighted kappa implemented from the standard definition (Cohen 1968) in `src/eval/agreement.py`; not copied from a library.
- Prompts, routing rules, taxonomy, metrics, annotation guide and failure analysis are my own.
- AI coding assistance was used for scaffolding and boilerplate, as the brief permits; every design decision in `DECISIONS.md` is mine and I can modify any of it live.

## Known limitations

Listed in full in `REPORT.md` §5. The three that matter most: the committed run is
on synthetic data, the judge's groundedness dimension is not trustworthy (QWK 0.006),
and `unsafe_auto=0.010` is 2 examples with a CI of [0.000, 0.025].
