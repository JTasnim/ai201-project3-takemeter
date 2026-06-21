# TakeMeter — Planning

## Community

**Chosen community:** r/television

r/television is a good fit for this task because it's a general-purpose TV discussion subreddit with very high post volume and genuinely mixed discourse quality — unlike a single-show subreddit, posts here range across dozens of different shows, which means the *quality* of a post (whether it argues, asserts, or just emotes) varies independently of its topic. That separation matters for a classifier: I want the model learning the structural difference between an argument and a reaction, not learning to associate certain shows with certain labels. The subreddit also has enough volume that collecting 200+ public posts/comments without running out of material is not a concern.

## Labels

Three labels, modeled on the same analysis/hot_take/reaction distinction used as the project's reference example, adapted to TV discourse:

**`analysis`** — The post makes a structured argument about the show backed by specific, verifiable evidence: plot details, character arcs, season/episode comparisons, or craft elements (writing, pacing, directing, editing). The evidence would still support the claim even with the opinion framing removed.
- Example 1: "The pacing problems start exactly when they split the writers' room in S4 — episode runtimes drop from ~58 min to ~42 min average, and three major character arcs get compressed into the back half as a result."
- Example 2: "Compare how the show handles grief in the S2 finale (the funeral scene, the long static shots, no dialogue for almost two minutes) versus the S3 premiere (a quick voiceover and a time skip) — that shift in restraint is why a lot of people feel the show lost its nerve."

**`hot_take`** — A bold, confident judgment about the show stated with little or no supporting evidence. The post asserts rather than argues.
- Example 1: "This is the worst final season of any show ever made. Not close."
- Example 2: "The lead actor cannot carry a scene by himself. Never could."

**`reaction`** — An immediate emotional response to something that just happened in an episode. Little to no argument — the post is expressing a feeling in the moment, tied to a specific scene or event.
- Example 1: "I am NOT okay. They really killed him in the cold open??"
- Example 2: "Sobbing at my desk. Did not expect that twist at all."

## Hard edge cases

Two boundaries need explicit decision rules, since they're the places where two annotators are most likely to disagree. Both rules below have now been validated against real r/television comments collected during annotation (not just constructed hypotheticals).

**Boundary 1: `analysis` vs. `hot_take`**

The ambiguous case: a comment that *gestures* at a comparison without actually arguing from specific evidence.

Decision rule: if the comment names specific, checkable evidence (a scene, a line, an episode, a named comparison point) that would still support the claim with the opinion framing removed, label it `analysis`. If the evidence is vague, decorative, or merely gestured at, label it `hot_take`.

**Real validating example** (found during annotation): *"Lincoln Lawyer would have been better if they'd kept it as gritty as Bosch. Instead, it felt really lightweight, like an old-school network drama."* This is a near-exact match to the boundary I anticipated before collecting data. It makes a comparative claim (Lincoln Lawyer vs. Bosch) but supports it only with vibe-words ("gritty," "lightweight") rather than a specific scene or line — so it's `hot_take` under the rule, not `analysis`. The rule held up cleanly on contact with real text.

**Second real hard case** (a genuine new discovery, not anticipated in advance): *"It was fascinating watching both that and Bosch for how different they were in cinematography? Other stuff? If I knew more about tv production I could say something intelligent about it."* This comment explicitly admits it can't articulate the distinction it's gesturing at — the commenter names a real point of comparison (cinematography) but concedes they can't argue it. This extends the original rule: a comment that flags a real point of comparison but openly disclaims the ability to argue it is still `hot_take`, since an unargued claim is unargued regardless of whether the author admits it.

**Third real hard case** (found in a "what are you watching" weekly thread): *"The Americans - on season 4. Turning out to be a S-tier show that I'm prepared to put in the same tier as Sopranos, Breaking Bad, Mad Men etc if it sticks the landing. Incredible writing and acting, the storylines are complex and intelligent, and it's tense as fuck."* This one is trickier than the first two because name-dropping other prestige shows *looks* like the comparative-evidence move that earns `analysis` elsewhere in this dataset (see the Top Gear/Grand Tour budget-and-censorship comparison, which is `analysis`). The difference: that comparison names a specific, checkable mechanism (smaller budget, BBC censorship workarounds) as the reason for the judgment. This comment never says *why* The Americans belongs in the Sopranos/Breaking Bad tier — "incredible writing," "complex storylines," and "tense as fuck" are asserted, not demonstrated with a scene, line, or plot example. Decision: `hot_take`. This sharpens the rule further — naming other shows as a reference point is not itself evidence; the comparison still needs a stated, checkable reason to count as `analysis`.

**Boundary 2: `reaction` vs. `hot_take`**

The ambiguous case: comments that open as a pure reaction but tack on a general judgment, e.g. "I'm SOBBING. Worst writing decision they've ever made."

Decision rule: if the comment is anchored to a specific just-happened event and the emotional content dominates, label it `reaction` even if it contains a short judgment — the judgment is incidental to the reaction, not the point. If the comment is a standing judgment about the show as a whole, untethered to one scene, label it `hot_take` even if it opens emotionally.

This boundary hasn't produced a genuinely ambiguous real example yet — the first 20 annotated comments split cleanly. Will continue watching for this pattern in later batches and update this section if a real hard case surfaces.

**Boundary 3: does `analysis` require the argument to be about the show's craft specifically, or any well-evidenced claim discussed in the community?**

This came up during annotation of a real comment: *"Hammond was always a picky eater and seemingly kept on top of his health in terms of diet and exercising. Yet he's also been the one person in the trio who's been the closest to death with the amount of crazy accidents he's been in."* This is a structured, evidence-based claim — but about a cast member's personal history, not the show's writing/pacing/craft.

Decision rule: `analysis` is about argument *structure*, not subject matter. The taxonomy measures discourse quality (does the post argue from evidence, or just assert?), not topic. A well-evidenced claim about a host's accident history is structurally the same as a well-evidenced claim about an episode's pacing — same reasoning pattern, different object. Restricting `analysis` to "show craft only" would add a topic filter the taxonomy was never designed to need, and risks shrinking the label to the point where it captures too few examples. The comment above is labeled `analysis`.

## Data collection plan — CLOSED OUT (Milestone 3 complete)

**Final dataset: 209 examples** (exceeds the 200 minimum), collected from real public r/television posts and comments across ~17 threads spanning multiple thread types: cancellation/news announcements, weekly "what are you watching" rec threads, actor-death tribute threads, season reviews, and retrospective discussion threads.

**Final label distribution:**
- `analysis`: 99 (47.4%)
- `hot_take`: 67 (32.1%)
- `reaction`: 43 (20.6%)

No label exceeds the 70% imbalance threshold the project warns against, so the dataset is usable as collected. That said, the distribution is not perfectly even, and the gap is worth documenting honestly rather than papering over.

**Why `reaction` ended up smallest:** Early batches (cancellation-news threads) skewed toward `analysis`/`hot_take` because people were debating cancellation patterns and quality in the abstract, not reacting to anything that "just happened" on screen. I deliberately tried to correct this twice — first by pulling threads with more in-the-moment content (an actor-tribute thread, a "what are you watching this week" thread), which raised `reaction` from 15% to 27% at one point, and second by trying to source genuine current-episode "Episode Discussion" threads specifically to push it further.

The second attempt mostly failed: r/television's highest-upvoted, most-discussed threads turned out to consistently be **retrospective** content — "season 1 felt like a different show," "10 years later, was this episode actually good?", pre-release critic reviews — rather than live, same-day reactions to a freshly-aired episode. I skipped two such retrospective threads outright (logged as a deliberate decision, not an oversight) because processing them would have pushed `analysis` even higher without helping the `reaction` gap at all. A later large batch of House of the Dragon S3 pre-release coverage threads, while useful for `analysis`, reinforced the same pattern and brought `reaction`'s share back down slightly from its peak.

**Takeaway for the final report:** this is a genuine, documented finding about the community rather than a collection failure — r/television's most-discussed content is structurally biased toward analytical and critical discourse over pure in-the-moment reaction, likely because retrospective "was this good?" framing generates more discussion (and therefore more visibility) than a simple emotional response does. A model trained on this distribution will have proportionally less exposure to `reaction` examples than to the other two classes; this is flagged here so it can be referenced directly in the evaluation report if `reaction` shows weaker per-class performance.

**Format:** Single CSV (`dataset.csv`) with `text`, `label`, and `notes` columns. Not pre-split — the notebook handles the 70/15/15 train/val/test split automatically. The `notes` column documents hard/ambiguous cases as they were encountered during annotation (see Hard Edge Cases above for the most significant ones).

**Cleaning process used throughout collection:** every batch of raw Reddit text required stripping UI chrome (upvote/downvote counts, "Reply," "Award," "Share," timestamps, avatar tags), removing promoted/ad content embedded in threads, and excluding comment fragments that only make sense as a reply to a parent comment (these aren't self-contained "takes" and would be mislabeled if read standalone). Off-topic tangents within otherwise on-topic threads (e.g., a TV-news thread that devolves into general political debate, or an actor-death thread that drifts into general health/medical discussion) were also excluded as out of scope for a taxonomy that measures *TV discourse* specifically.

## Evaluation metrics

- **Overall accuracy** — baseline sanity check, but not sufficient alone since with 3 roughly-balanced classes a trivial baseline is ~33%, and accuracy alone hides which specific boundary the model is failing at.
- **Per-class precision, recall, and F1** — necessary because the two boundaries identified above (`analysis`/`hot_take` and `reaction`/`hot_take`) are exactly the places I expect confusion to concentrate. Per-class F1 will tell me which boundary is actually hard for the model, not just that the model is imperfect overall.
- **Confusion matrix** — needed to see the *direction* of errors (e.g., is `hot_take` being predicted for true `analysis` posts, or the reverse?), which matters for diagnosing whether the issue is a labeling inconsistency or a genuine model limitation.
- **Baseline comparison (zero-shot Groq vs. fine-tuned)** — the only way to know whether fine-tuning added real value over a general-purpose LLM with no task-specific training, on the same test set.

## Definition of success

For this project, I will consider the classifier genuinely useful if:
- The fine-tuned model achieves **per-class F1 ≥ 0.70** on all three labels (per the project's own benchmark for "model is learning all distinctions well").
- The fine-tuned model **meaningfully outperforms the zero-shot Groq baseline** (not just matches it) — since the entire point of fine-tuning is to capture community-specific signal a general-purpose model wouldn't have.
- The confusion matrix shows errors concentrated at the two boundaries I already anticipated (`analysis`/`hot_take`, `reaction`/`hot_take`) rather than scattered unpredictably — concentrated, explainable errors mean the model learned something coherent even if imperfect; scattered errors would suggest the labels themselves are inconsistent.

I would accept "good enough for a real community tool" as: per-class F1 ≥ 0.70 across all labels, with any remaining errors traceable to one of the two known-hard boundaries rather than to random noise.

**Update after data collection:** the final dataset is not perfectly balanced (`analysis` 47.4%, `hot_take` 32.1%, `reaction` 20.6% — see Data Collection Plan for why). `reaction` has roughly half the examples of `analysis`. I'm not lowering the F1 ≥ 0.70 bar for `reaction` because of this, but I will pay close attention to whether `reaction`'s F1 specifically comes in lower than the other two classes, since that would be a plausible and expected consequence of having less training data for it — not necessarily a sign the label itself is poorly defined.

## AI Tool Plan

**Label stress-testing:** Before finalizing labels, I will give an AI tool my label definitions and the two edge-case decision rules above, and ask it to generate 5-10 synthetic r/television-style posts designed to sit at each boundary. If any of the generated posts can't be cleanly classified under my current rules, I'll tighten the rules before annotating 200 real examples.

**Annotation assistance:** I plan to use an LLM (Groq, `llama-3.3-70b-versatile`) to pre-label a batch of collected posts before reviewing every label myself. I will track which examples were pre-labeled (a column in my working spreadsheet, not the final CSV) so this can be disclosed accurately in the AI usage section of my README. I will not commit any pre-labeled example without reading the original post myself and confirming or correcting the label.

**Failure analysis:** After fine-tuning, I will paste the model's misclassified examples into an AI tool and ask it to identify patterns (e.g., post length, sarcasm, label pairs that are confused most often). I will independently re-read the flagged examples myself to verify any pattern the AI tool identifies before writing it into the evaluation report — the AI's pattern-spotting is a starting hypothesis, not a conclusion.
