## Evaluation Report

### Overall Results

| Model | Accuracy | Test Set Size |
|---|---|---|
| Zero-shot baseline (Groq, `llama-3.3-70b-versatile`) | **71.9%** | 32 |
| Fine-tuned (`distilbert-base-uncased`) | **46.9%** | 32 |

Fine-tuning did not improve on the baseline — it performed **25 points worse**. This is not the result I expected going in, but it's a real and fully diagnosed one, and it taught me more about fine-tuning than a clean win would have.

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

Every single one of the 32 test examples was predicted as `analysis`. This isn't a model that's confidently wrong in one direction — it's a model that predicted the same label regardless of input.

### Diagnosis: Why the Fine-Tuned Model Failed

My first hypothesis was that 32 test examples is too small a sample to draw conclusions from, and that's worth keeping in mind generally — but it doesn't explain *this* result, because the model isn't making varied mistakes that a small sample might exaggerate. It's making the *same* prediction every time.

Three pieces of evidence pin down what actually happened:

**1. Validation accuracy was bit-for-bit identical across all three training epochs.**

| Epoch | Training Loss | Validation Loss | Validation Accuracy |
|---|---|---|---|
| 1 | 1.0849 | 1.0728 | 0.4839 |
| 2 | 1.0694 | 1.0551 | 0.4839 |
| 3 | 1.0207 | 1.0311 | 0.4839 |

Training loss did decrease slightly each epoch, which means the model's weights were changing — but validation accuracy never moved even fractionally, across three full epochs. The model wasn't learning to distinguish the classes; it was just adjusting weights within the basin of "always predict the majority class."

**2. The confidence scores on wrong predictions were barely above chance.**

Every misclassified test example had a predicted-class confidence between 0.36 and 0.38. With three classes, a uniformly random guess would score ~0.33 confidence on each. A model that had genuinely (if wrongly) learned to associate certain language with `analysis` would show much higher confidence on its mistakes. 0.36–0.38 confidence on every single error means the model's output distribution was close to uniform across all three classes for every input — `analysis` won the argmax not because the model identified it, but because it was the path of least resistance during training (it's the majority class at 47% of the training set).

**3. The training setup gave the model very little signal to work with.**

146 training examples split across 3 classes, at batch size 16, produced only 30 total training steps across all 3 epochs (visible in the Colab progress bar: `[30/30]`). That's an extremely small number of gradient updates for a transformer model being fine-tuned from a generic pretrained checkpoint with no prior exposure to this classification task. Combined with a conservative learning rate (2e-5, the spec's suggested default), the model likely needed either more data, more epochs, or a higher learning rate to escape the safe, low-risk strategy of predicting the majority class before training time ran out.

### What the Model Learned vs. What I Intended

I intended for the model to learn the structural distinction between an argued claim, an asserted claim, and an emotional reaction — the same distinction a human annotator applies by checking for specific, verifiable evidence (see `planning.md`'s edge-case rules). Instead, the model learned a single fact: `analysis` is the most common label in the training set. It did not learn anything about the content of the text at all — the near-uniform confidence scores on every input (regardless of the input's actual content) confirm that the model's predictions were essentially independent of what it was reading.

This is a meaningfully different finding than "the model is bad at the task" or "the labels are inconsistent." It's a training-configuration problem: 146 examples and 30 gradient steps were not enough for a model with no prior exposure to this task to learn a real signal, especially when the safest strategy (predict the majority class) achieves nearly 50% accuracy by default and requires no learning at all.

### Why the Baseline Outperformed the Fine-Tuned Model

The zero-shot Groq baseline's 71.9% accuracy reflects a large, heavily pretrained model bringing broad language understanding to the task with no task-specific training at all — it can recognize argumentative structure, hedge language, and emotional register from its pretraining alone, then map that recognition onto the label definitions given in the prompt. The fine-tuned DistilBERT, by contrast, started from a similarly capable pretrained checkpoint but was given too little task-specific signal to build on that foundation before evaluation. In effect, the comparison ended up testing "a capable pretrained model with a good prompt" against "a capable pretrained model given an insufficient nudge in a new direction" — and the nudge didn't take.

### What I'd Try Differently

- **More training steps**: increasing epochs (e.g., from 3 to 8-10) on this same small dataset, to see if the model could eventually break out of the majority-class plateau given more gradient updates.
- **Higher learning rate**: 2e-5 is a conservative default; a higher rate (e.g., 5e-5) might move the model out of the flat region faster on a dataset this small.
- **More training data**: the structural fix. 146 examples is genuinely small for fine-tuning a transformer from a generic checkpoint, and `hot_take` (47 examples) and `reaction` (30 examples) had especially little to learn from.
- **Class-weighted loss**: since `analysis` is the majority class, weighting the loss function to penalize errors on minority classes more heavily might have forced the model to engage with `hot_take` and `reaction` examples rather than defaulting to the safe answer.

I did not re-run training with these changes for this submission, since the goal here was to honestly diagnose and report what happened with the spec's suggested default hyperparameters, rather than iterate until I found a configuration that produced a more flattering result.
