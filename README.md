# TakeMeter

A fine-tuned text classifier that evaluates discourse quality on r/television, sorting posts and comments into three categories: `analysis`, `hot_take`, and `reaction`.

## Community Choice and Reasoning

**Community:** r/television

r/television is a general-purpose TV discussion subreddit with very high post volume and genuinely mixed discourse quality. Unlike a single-show subreddit, posts here range across dozens of different shows, which means the *quality* of a post — whether it argues, asserts, or just emotes — varies independently of its topic. That separation matters for a classifier: the goal is for the model to learn the structural difference between an argument, an assertion, and a reaction, not to learn shortcuts tied to specific shows. The subreddit's volume also made it straightforward to collect 200+ public examples without running out of material or needing to scrape.

## Label Taxonomy

Three labels, each defined by argument *structure* rather than topic, sentiment, or length:

| Label | Definition |
|---|---|
| `analysis` | Makes a structured argument backed by specific, verifiable evidence — plot details, character arcs, season/episode comparisons, or craft elements (writing, pacing, directing, editing). The evidence would still support the claim even with the opinion framing removed. |
| `hot_take` | A bold, confident judgment stated with little or no supporting evidence. The post asserts rather than argues. |
| `reaction` | An immediate emotional response to something that just happened in an episode. Little to no argument — the post expresses a feeling in the moment, tied to a specific scene or event. |

**Examples per label:**

- `analysis`: *"The pacing problems start exactly when they split the writers' room in S4 — episode runtimes drop from ~58 min to ~42 min average, and three major character arcs get compressed into the back half as a result."*
- `hot_take`: *"This is the worst final season of any show ever made. Not close."*
- `reaction`: *"I am NOT okay. They really killed him in the cold open??"*

Full definitions, additional examples, and the reasoning behind the taxonomy design are in [`planning.md`](./planning.md).

## Hard Edge Cases

Three boundaries required explicit decision rules, each validated against real text encountered during annotation (not just designed hypotheticals):

1. **`analysis` vs. `hot_take`** — a comment that *gestures* at comparative evidence without actually arguing from specifics (e.g., "it would've been better if they'd kept it as gritty as Bosch") is `hot_take`, not `analysis`. Naming a comparison point isn't itself evidence; the comparison needs a stated, checkable reason.
2. **`reaction` vs. `hot_take`** — a comment anchored to a specific just-happened event, where emotional content dominates, is `reaction` even if it contains a brief judgment. A standing judgment untethered to one scene is `hot_take` even if it opens emotionally.
3. **Does `analysis` require the argument to be about the show's craft specifically?** — Resolved: no. `analysis` is about argument *structure*, not subject matter. A well-evidenced claim about a cast member's personal history is structurally identical to a well-evidenced claim about an episode's pacing.

See [`planning.md`](./planning.md) for the full reasoning behind each rule, including the specific real comments that prompted and validated each one.

## Data Collection

- **Source:** Public posts and comments from r/television — cancellation/news threads, weekly "what are you watching" rec threads, actor-tribute threads, season reviews, and retrospective discussion threads.
- **Volume:** 209 labeled examples (exceeds the 200 minimum).
- **Label distribution:**

| Label | Count | Percentage |
|---|---|---|
| `analysis` | 99 | 47.4% |
| `hot_take` | 67 | 32.1% |
| `reaction` | 43 | 20.6% |

No label exceeds the project's 70% imbalance threshold, but the distribution isn't perfectly even. `reaction` ended up smallest because r/television's highest-upvoted, most-discussed threads consistently skew toward retrospective and critical analysis ("was this season actually good?", "season 1 felt like a different show") rather than live, same-day reactions to a freshly-aired episode. I made two deliberate attempts to correct this — targeting an actor-tribute thread and a weekly "what are you watching" thread, which raised `reaction`'s share from 15% to 27% at one point — but a later wave of House of the Dragon season-review threads pulled it back down. I judged this to be a genuine, documented finding about the community rather than a collection failure, and chose not to keep collecting indefinitely chasing a perfectly even split. Full reasoning is in `planning.md`.

- **Difficult examples:** 5 examples in the dataset are flagged with annotation notes documenting genuinely hard labeling decisions (see the `notes` column in `dataset.csv`, and the Hard Edge Cases section above for the three most significant ones).
- **Cleaning process:** raw Reddit text required stripping UI chrome (upvote counts, "Reply," "Award," timestamps), removing promoted/ad content embedded in threads, and excluding comment fragments that only make sense as a reply to a parent comment. Off-topic tangents within otherwise on-topic threads (e.g., a TV-news thread that devolves into general political debate, or an actor-death thread that drifts into medical discussion) were excluded as out of scope for a taxonomy measuring TV discourse specifically. Two entire threads were discarded outright for containing content not appropriate to extract into a training dataset regardless of nominal topic fit.

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased`
- **Training setup:** 70/15/15 stratified train/val/test split (146/31/32 examples), tokenized with the DistilBERT tokenizer (max length 256).
- **Hyperparameters:** 3 epochs, learning rate 2e-5, batch size 16 — the notebook's suggested defaults for a small dataset. I did not adjust these for this submission (see Reflection below for why, and what I'd try instead).
- **Key hyperparameter decision:** I used the defaults rather than tuning them, in order to honestly evaluate what the suggested starting configuration produces on a dataset this size, rather than search for a configuration that would produce a more flattering result before ever seeing a baseline number to compare against.

## Baseline Description

- **Model:** Groq's `llama-3.3-70b-versatile`, zero-shot (no task-specific training).
- **Prompt:** Names the community (r/television) and task, defines each label in plain language with one example per label (drawn directly from the same definitions used for human annotation), and instructs the model to output only the label name. See cell 20 of the notebook for the exact prompt.
- **Parsing:** Model output is matched against the three label strings; responses that don't match any label are excluded from baseline accuracy and flagged separately.

## Evaluation Report

### Overall Results

| Model | Accuracy | Test Set Size |
|---|---|---|
| Zero-shot baseline (Groq, `llama-3.3-70b-versatile`) | **71.9%** | 32 |
| Fine-tuned (`distilbert-base-uncased`) | **46.9%** | 32 |

Fine-tuning did not improve on the baseline — it performed **25 points worse**. This was not the result I expected, but it's a real, fully diagnosed one.

### Per-Class Metrics (Fine-Tuned Model)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.47 | 1.00 | 0.64 | 15 |
| `hot_take` | 0.00 | 0.00 | 0.00 | 10 |
| `reaction` | 0.00 | 0.00 | 0.00 | 7 |

### Confusion Matrix (Fine-Tuned Model)

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 15 | 0 | 0 |
| **True: hot_take** | 10 | 0 | 0 |
| **True: reaction** | 7 | 0 | 0 |

Every one of the 32 test examples was predicted as `analysis`. This isn't a model that's confidently wrong in one direction — it's a model that predicted the same label regardless of input. (Confusion matrix image also committed as `confusion_matrix.png`.)

### Diagnosis: Why the Fine-Tuned Model Failed

My first hypothesis was that 32 test examples is too small a sample to draw conclusions from, and that's worth keeping in mind generally — but it doesn't explain this specific result, because the model isn't making varied mistakes that a small sample might exaggerate. It's making the *same* prediction every time. Three pieces of evidence pin down what actually happened:

**1. Validation accuracy was identical across all three training epochs.**

| Epoch | Training Loss | Validation Loss | Validation Accuracy |
|---|---|---|---|
| 1 | 1.0849 | 1.0728 | 0.4839 |
| 2 | 1.0694 | 1.0551 | 0.4839 |
| 3 | 1.0207 | 1.0311 | 0.4839 |

Training loss decreased slightly each epoch, meaning the model's weights were changing — but validation accuracy never moved, across three full epochs. The model wasn't learning to distinguish the classes; it was adjusting weights within the basin of "always predict the majority class."

**2. Confidence scores on wrong predictions were barely above chance.**

Every misclassified test example had predicted-class confidence between 0.36 and 0.38. With three classes, a uniformly random guess scores ~0.33 confidence on each. A model that had genuinely (if wrongly) learned to associate certain language with `analysis` would show much higher confidence on its mistakes. This near-uniform confidence means the model's output distribution was close to flat across all three classes for every input — `analysis` won the argmax because it was the majority class, not because the model identified anything in the text.

**3. The training setup gave the model very little signal.**

146 training examples across 3 classes, at batch size 16, produced only 30 total training steps across all 3 epochs. That's a small number of gradient updates for a transformer fine-tuned from a generic pretrained checkpoint with no prior exposure to this task. Combined with a conservative learning rate (2e-5), the model likely needed more data, more epochs, or a higher learning rate to escape predicting the majority class before training ended.

### Three Specific Wrong Predictions

**1. "Sundays are back"**
True label: `reaction` · Predicted: `analysis` (confidence: 0.37)

About as pure a `reaction` example as exists in the dataset — short, enthusiastic, zero argument content. The near-chance confidence here shows the model wasn't engaging with the text at all: even a crude heuristic like "short, exclamatory posts are usually reactions" would have gotten this right.

**2. "Maximum Pleasure Guaranteed, man what a bunch of cringe horseshit. The acting is awful."**
True label: `hot_take` · Predicted: `analysis` (confidence: 0.37)

A clean `hot_take` by the project's own annotation rule: a blunt judgment with no specific scene or line cited as evidence. A model that had learned the analysis/hot_take boundary should have picked up on the total absence of named evidence.

**3. "It was fascinating watching both that and Bosch for how different they were in cinematography? ... If I knew more about tv production I could say something intelligent about it."**
True label: `hot_take` · Predicted: `analysis` (confidence: 0.37)

One of the genuine hard cases documented in `planning.md` — a comment that gestures at real evidence (cinematography) but explicitly disclaims its own ability to argue it. I expected this kind of nuanced boundary case to be exactly where a fine-tuned model might add value over a more rigid baseline. Instead it was predicted at the same near-chance confidence as the unambiguous examples, confirming the model wasn't making fine-grained distinctions of any kind — easy and hard cases were treated identically because there was no real learned signal behind any prediction.

### What the Model Captured vs. What I Intended

I intended for the model to learn the structural distinction between an argued claim, an asserted claim, and an emotional reaction — the same distinction a human annotator applies by checking for specific, verifiable evidence. Instead, the model learned a single fact: `analysis` is the most common label in the training set. It did not learn anything about the content of the text — the near-uniform confidence on every input, regardless of content, confirms predictions were essentially independent of what the model was reading.

This is a different and more useful finding than "the model is bad at the task" or "the labels are inconsistent." It's a training-configuration problem: 146 examples and 30 gradient steps were not enough for a model with no prior task exposure to learn a real signal, especially when the safest strategy (always predict the majority class) achieves nearly 50% accuracy with no learning required.

### Why the Baseline Outperformed the Fine-Tuned Model

Groq's 71.9% baseline reflects a large, heavily pretrained model bringing broad language understanding to the task with no task-specific training — it can recognize argumentative structure, hedging, and emotional register from pretraining alone, then map that recognition onto the label definitions given in the prompt. DistilBERT started from a similarly capable pretrained checkpoint but was given too little task-specific signal to build on that foundation before evaluation. The comparison ended up testing "a capable pretrained model with a good prompt" against "a capable pretrained model given an insufficient nudge in a new direction" — and the nudge didn't take.

### What I'd Try Differently

- **More training steps** — increasing epochs (e.g., 3 → 8-10) to see if the model could break out of the majority-class plateau given more gradient updates.
- **Higher learning rate** — 2e-5 is conservative; a higher rate (e.g., 5e-5) might move the model out of the flat region faster on a dataset this small.
- **More training data** — the structural fix. 146 examples is genuinely small for fine-tuning a transformer from a generic checkpoint, and `hot_take` (47 train examples) and `reaction` (30 train examples) had especially little to learn from.
- **Class-weighted loss** — weighting the loss to penalize errors on minority classes more heavily might force the model to engage with `hot_take` and `reaction` examples rather than defaulting to the safe answer.

I did not re-run training with these changes for this submission. The goal was to honestly diagnose and report what happened with the spec's suggested default hyperparameters, rather than iterate until finding a configuration that produced a more flattering result.

### Sample Classifications

| Post | Predicted Label | Confidence | Notes |
|---|---|---|---|
| "Sundays are back" | `analysis` | 0.37 | Incorrect — see wrong-prediction analysis above. |
| "Maximum Pleasure Guaranteed, man what a bunch of cringe horseshit. The acting is awful." | `analysis` | 0.37 | Incorrect — see wrong-prediction analysis above. |
| "There's a moment in the second season where the Tully lord orders Daemon to execute the Blackwood where it feels like something actually happened... but it doesn't quite match up" | `analysis` | — | Correctly predicted — true label is `analysis`, and the comment does name a specific scene as evidence for its claim, which is consistent with the model's (mostly-uninformative) tendency to predict this class regardless of content. |

*(Given that the fine-tuned model predicted `analysis` for all 32 test examples, every correct prediction in this run happens to be an `analysis` example — this isn't meaningful evidence of correct classification, just confirmation of the majority-class collapse described above.)*

## Spec Reflection

**One way the spec helped guide my implementation:** the spec's hint to "read 30-40 posts before committing to labels" turned out to be essential. My first edge-case rule (for the analysis/hot_take boundary) was designed before reading any real text, and it held up — but a second hard case (the cinematography comment that disclaims its own argument) only surfaced once I was actually annotating real comments, and it meaningfully sharpened the rule in a way I wouldn't have anticipated from design alone.

**One way my implementation diverged from the spec, and why:** the spec's suggested workflow implies running the baseline, then fine-tuning, then comparing — with an expectation that fine-tuning will generally help. My result diverged from that expectation, and rather than treating this as something to fix before submitting, I followed the spec's repeated emphasis on "evaluate honestly" and "wrong predictions are your most valuable data" by documenting the failure as the actual finding, including a full diagnosis of why it happened, rather than tuning hyperparameters until I got a result that matched the expected narrative.

## AI Usage

I used Claude (Anthropic) throughout this project as a planning, annotation, and analysis collaborator. Two specific instances:

1. **Annotation assistance and consistency-checking during data collection.** I pasted raw Reddit threads to Claude, which cleaned the text (removing UI chrome, ads, and out-of-scope content) and proposed a label for each comment against my `planning.md` definitions. I reviewed every proposed label myself before it was added to `dataset.csv` — I did not commit any label without confirming it matched my own reading of the comment. Claude also flagged genuinely ambiguous comments as candidate hard cases, which I then decided how to resolve and recorded in `planning.md`.

2. **Failure analysis on the fine-tuned model's wrong predictions.** After getting the `evaluation_results.json` output, I asked Claude to help diagnose why fine-tuning underperformed the baseline. Claude identified the pattern across the per-epoch training table (flat validation accuracy) and the wrong-prediction confidence scores (near-chance, ~0.37) as evidence of majority-class collapse rather than a more typical learning failure. I verified this independently by checking that the confusion matrix showed literally 100% of predictions going to one class, and that the per-epoch table I pulled from Colab showed truly identical accuracy values across all three epochs — both of which confirmed the hypothesis rather than just taking it on faith.

I did not use AI-assisted pre-labeling of unlabeled batches before review (i.e., I did not have an LLM bulk-label data that I then only spot-checked); every label in `dataset.csv` reflects a label I read and confirmed against the original text myself.
