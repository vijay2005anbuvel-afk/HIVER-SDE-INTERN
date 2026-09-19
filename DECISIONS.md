# Decision log

The non-obvious calls, and why. Roughly in the order I made them.

1. **Brand = `SpotifyCares`, one brand only.** I wanted a brand whose intents are
   genuinely separable but overlapping at the edges (a billing complaint and a
   playback complaint share a lot of vocabulary). Airlines were tempting because
   volume is huge, but airline threads are dominated by one intent (delays), which
   would have made macro-F1 nearly meaningless. Rejected: training one model across
   all brands — the brief asks for an agent for *a* brand, and cross-brand reply
   grounding would leak one company's policy into another's mouth.

2. **Eight intents, built bottom-up, stopping at saturation.** I read inbound
   messages in batches of 50, free-text tagged, then merged tags. A new batch
   stopped producing new tags at batch 9, so I stopped. I did not start from
   Banking77's 77 labels — see #3.

3. **Did not use Banking77.** It was offered for intent work, and it is tempting
   because it is clean and labelled. But its label space is retail-banking
   ("card_arrival", "exchange_rate") and its register is formal single-sentence
   queries. Transferring either would have given me a taxonomy that fits a
   different product and a classifier tuned to text that does not look like
   Twitter. The honest use would have been as a *calibration* set for the judge,
   and I ran out of time for that. Listed under "next week".

4. **Threads are recovered through reply pointers, not the brand handle.** My first
   loader filtered to rows whose text mentions the brand. That silently dropped a
   large share of opening messages, because customers frequently reply into an
   existing thread without re-mentioning the handle. The loader now does two
   passes: collect brand rows and the `in_response_to_tweet_id` values they point
   at, then recover those parents. On the sample corpus this moved thread recovery
   from 1094 to 1356 out of 1400.

5. **Split by thread, never by message, with a fixed seed.** A thread's opening
   message and its follow-ups are near-duplicates. Splitting by message would put
   one half in the retrieval KB and the other in the eval set, and retrieval
   precision would be a measurement of that mistake.

6. **TF-IDF retrieval, not embeddings.** Embeddings would likely win on recall. I
   chose TF-IDF because it runs in the 15-minute reproduce budget with no model
   download and no API key, and because it is inspectable — when a precedent is
   wrong I can see exactly which terms matched. Given that my top failure mode
   turned out to be *grounded-but-wrong retrieval*, the inspectability paid for
   itself. Swapping in embeddings is a one-class change in `retrieve.py`.

7. **Unanswered threads are excluded from the KB but kept in the eval pool.**
   An empty reply is not a resolution, so indexing it gives the drafter nothing to
   ground on. But ~14% of inbound messages never get answered, and those are
   disproportionately the hard ones — dropping them from evaluation too would have
   quietly deleted the hardest slice of the problem.

8. **Retrieval grounds the reply; it does not decide the intent.** I considered
   nearest-neighbour intent transfer (take the label of the closest precedent).
   It is cheaper, but it couples two failure modes: a retrieval miss then becomes
   an intent error *and* a grounding error, and the failure analysis can no longer
   tell them apart.

9. **Confidence must combine dominance and evidence.** My first confidence score
   was the winning intent's share of total lexical score. It returned ~1.0 whenever
   exactly one weak cue fired, which made the routing threshold decorative. It now
   multiplies share by an evidence-magnitude term, which produces the spread the
   threshold needs (`llm.py::_classify`).

10. **Routing is rules over model output, not a second LLM call.** Escalation is
    the safety valve for the entire system. I wanted it inspectable in 60 seconds,
    identical every run, free, and changeable by a support lead who does not write
    prompts. Rules fire in priority order and the first match wins, so the reason
    surfaced to the human is the most serious one rather than an arbitrary one.

11. **Sensitive intents (`billing_and_subscription`, `account_access`) are always
    escalated, regardless of confidence.** These are the two intents where a
    confident wrong reply costs money or locks someone out. This single rule
    accounts for 60 of 109 escalations, which is a concentration risk I call out
    in the report rather than hide.

12. **The gold escalation rubric is written in terms of consequences, not in terms
    of the router's rules.** `data/golden/ANNOTATION_GUIDE.md` asks "would I let
    this go out unsupervised?" If I had derived gold labels from `route.py`,
    routing accuracy would be a tautology. They still partially overlap, and the
    report quantifies how much that inflates escalation recall.

13. **Golden set is stratified, and every metric is also reported re-weighted back
    to the natural distribution.** Uniform sampling gives 3 examples of the rarest
    intent, which makes macro-F1 noise. Oversampling rare and hard cases fixes that
    but measures a distribution that does not exist. Reporting both, with
    importance weights estimated from the uniform stratum, is the only version I
    could defend.

14. **Baseline B1 trains on weak labels from the KB split, never on golden labels.**
    Training a baseline on the set you then evaluate it on is leakage, and it
    would have made B1 look better than it is. B1 still beats the agent on intent
    macro-F1 — see the report; I did not suppress that.

15. **Groundedness is measured twice: by the judge, and by a deterministic checker.**
    Judge-vs-human agreement on groundedness came out near zero (QWK 0.006, judge
    biased +2.6 points). A dimension the judge cannot score should not be scored by
    the judge, so `eval/grounding_check.py` checks the mechanically checkable part —
    URLs, numbers, and completed-action claims must appear in a precedent. It
    immediately found a real bug (#4 in the failure list) that the judge rated 4.9/5.

16. **Any classifier output outside the taxonomy is forced to `other_out_of_scope`
    with `fallback_used=True`, never passed through.** A free-text label that looks
    plausible silently corrupts every downstream metric, and it is the kind of bug
    that survives to production because nothing throws.

17. **A failed judge call scores nothing rather than 3/5.** Defaulting a failed
    judgement to the midpoint drags every system toward the mean and makes a broken
    judge look like a mediocre one. Failures are counted and excluded.
