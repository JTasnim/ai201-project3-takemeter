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

Two boundaries need explicit decision rules, since they're the places where two annotators are most likely to disagree.

**Boundary 1: `analysis` vs. `hot_take`**

The ambiguous case: a post that *gestures* at evidence without actually arguing from it. Example: "This show has gone completely downhill since the original showrunner left — just compare the dialogue quality in season 1 vs now."

Decision rule: if the post names specific, checkable evidence (a scene, a line, an episode, a stat) that would still support the claim with the opinion stripped out, label it `analysis`. If the evidence is vague, decorative, or merely gestured at — "compare the dialogue" without naming a single line or scene — label it `hot_take`. The example above is `hot_take`: it tells the reader to go verify the claim themselves rather than presenting the evidence.

**Note:** this rule is designed, not yet validated against real posts. Before annotating 200 examples, I plan to find at least one real post like this in r/television and confirm the rule actually produces a clean call — and revise the rule if it doesn't.

**Boundary 2: `reaction` vs. `hot_take`**

The ambiguous case: posts that open as a pure reaction but tack on a general judgment, e.g. "I'm SOBBING. Worst writing decision they've ever made."

Decision rule: if the post is anchored to a specific just-happened event/scene and the emotional content dominates, label it `reaction` even if it contains a short judgment — the judgment is incidental to the reaction, not the point of the post. If the post is a standing judgment about the show as a whole, untethered to one scene, label it `hot_take` even if it opens emotionally. I will read several real examples of this pattern during the 30-40 post read-through before finalizing.

## Data collection plan

- **Source:** Public posts and comments from r/television (top/hot posts plus a mix of "Discussion" megathreads, to capture both original posts and reply-level takes).
- **Volume target:** At least 200 examples total, aiming for roughly 65-70 per label to avoid the >70%-imbalance flag the project warns about. If one label (most likely `analysis`, since structured arguments are less common than reactions/hot takes on a general subreddit) is underrepresented after an initial pass, I will deliberately seek out more discussion-thread and episode-reaction-thread content, where analysis-style comments are more common, rather than padding with borderline examples.
- **If a label is underrepresented after 200:** collect additional targeted examples from threads more likely to contain that label type (e.g., season-finale discussion threads tend to generate more `analysis`-style comments than reaction threads) rather than loosening the label definition to capture more examples artificially.
- **Format:** Manual collection into a CSV (`text`, `label`, and a notes column for difficult cases), per the project's recommended workflow — at 200 examples, manual collection is faster than building a scraper and keeps me close to the actual text during collection.

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

## AI Tool Plan

**Label stress-testing:** Before finalizing labels, I will give an AI tool my label definitions and the two edge-case decision rules above, and ask it to generate 5-10 synthetic r/television-style posts designed to sit at each boundary. If any of the generated posts can't be cleanly classified under my current rules, I'll tighten the rules before annotating 200 real examples.

**Annotation assistance:** I plan to use an LLM (Groq, `llama-3.3-70b-versatile`) to pre-label a batch of collected posts before reviewing every label myself. I will track which examples were pre-labeled (a column in my working spreadsheet, not the final CSV) so this can be disclosed accurately in the AI usage section of my README. I will not commit any pre-labeled example without reading the original post myself and confirming or correcting the label.

**Failure analysis:** After fine-tuning, I will paste the model's misclassified examples into an AI tool and ask it to identify patterns (e.g., post length, sarcasm, label pairs that are confused most often). I will independently re-read the flagged examples myself to verify any pattern the AI tool identifies before writing it into the evaluation report — the AI's pattern-spotting is a starting hypothesis, not a conclusion.
