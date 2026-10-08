---
title: "From a sentence to my first fine-tuned model"
description: "An engineer’s first experiment with Vela: defining a classification task, preparing data, fine-tuning on a rented GPU, and publishing on Hugging Face."
pubDatetime: 2026-10-08T00:00:00+05:30
tags:
  - blog
  - machine-learning
  - fine-tuning
  - semantic-router
  - hugging-face
  - engineering
---

*How I adapted Vela to recognize expressed decisions—and what the small experiment actually taught me.*

I had been contributing to vLLM Semantic Router and experimenting with multi-turn conversations. Working on a router makes you think about the small judgments behind a request: what is this person asking for, and what should happen next?

One question had been sitting in the back of my mind for a while.

If someone writes, “I’m going to the office, and I’ll update the document today,” could a model recognize that they have expressed a firm plan? And could it distinguish that from “I like mountains”?

I could explain the difference to another person. I wanted to understand how to teach it to a model.

I also had a personal goal: publish my first model on Hugging Face. I wanted to know what lay between an idea and a checkpoint that another engineer could download and run.

This is the story of that experiment. It starts with two labels, uses a small dataset and one rented GPU, and ends with a working public model. There are encouraging results, but there is also unfinished evaluation. Both belong in the story.

## Before the model, I had to define “decision”

My original description was “decision taken or not.” Writing examples revealed that this was broader than it sounded.

“We chose PostgreSQL” records a choice. “I’ll update the document today” expresses a commitment, although it does not say the work has happened. “We decided not to migrate” expresses a choice to do nothing.

For this pilot, I defined the positive class as an **expressed decision, firm intention, or commitment**. That gave me two labels:

| Statement | Label | Reason |
| --- | --- | --- |
| “We have decided to update the document today.” | `DECISION_EXPRESSED` | An explicit chosen action. |
| “I’ll reply to the customer before the end of the day.” | `DECISION_EXPRESSED` | A firm commitment. |
| “We decided not to migrate this quarter.” | `DECISION_EXPRESSED` | Choosing inaction still counts. |
| “I like mountains.” | `NO_DECISION_EXPRESSED` | A preference. |
| “Could someone update the document?” | `NO_DECISION_EXPRESSED` | A request, rather than an affirmed commitment. |
| “We might migrate next quarter.” | `NO_DECISION_EXPRESSED` | A possibility. |

This boundary matters. The model does not tell me whether a promise was fulfilled, whether the speaker has authority, or whether a past decision still applies. It also does not decide which statements deserve an official entry in a decision log.

The narrow technical task is **single-label binary text classification**: give it a short statement, and choose one of two labels. “Single-label” means one output class per statement; “binary” means there are two possible classes.

My first lesson came before training: if I cannot consistently label the examples, changing the model will not resolve the disagreement.

## I chose an encoder for a small classification problem

I started from [Vela-1.0-Encoder-307M](https://huggingface.co/vllm-sr/Vela-1.0-Encoder-307M), using the exact foundation revision `720ab37904ce15054068d429381bfae7549f00f6`.

Vela uses a ModernBERT encoder architecture and identifies mmBERT as its upstream foundation. An encoder turns tokens into contextual numerical representations. A classification head uses those representations to produce class scores. The [Transformers ModernBERT documentation](https://huggingface.co/docs/transformers/v4.57.1/en/model_doc/modernbert) describes its sequence-classification support.

That fit my task: read a statement and return a label. I did not need the model to generate a paragraph explaining its answer.

The vocabulary can be confusing at first. **Pretraining** gives a model its initial learned representations. **Fine-tuning** updates an existing model using examples for a particular task. Fine-tuning is one form of post-training; post-training is a wider term that also covers other adaptation methods.

Here, I used supervised fine-tuning: statements were paired with target labels, and the model learned from those pairs. I reused the pretrained encoder and initialized a fresh task head for my two classes.

The experiment followed this sequence: define the two labels, write contrastive examples, split by example group, measure CPU baselines, load the pinned Vela encoder, train the final two layers and a fresh head, select a checkpoint using development results, reload and verify it, then publish the model and dataset. The next step is independent evaluation.

## The dataset was small on purpose

I began with 100 synthetic examples arranged in 50 contrastive groups. A group might contain a commitment and a closely related request:

> “I’ll reply to the customer before the end of the day.”
>
> “Could someone reply to the customer before the end of the day?”

The topic is almost identical. The relationship to action changes. That is the distinction I wanted the classifier to learn.

I kept each group together when splitting the data. Otherwise, the model could see one wording during training and encounter its near-twin during evaluation, making the evaluation easier than it appears.

I later added 48 synthetic examples inspired by patterns in a private context log. They were newly authored and anonymized; the original log was not published. All additional examples went into training. The existing development and reserved test files stayed unchanged.

The final split was:

| Split | Examples | Purpose |
| --- | --- | --- |
| Training | 118: 59 per class | Update the model’s parameters. |
| Development | 14: 7 per class | Inspect errors and select a checkpoint. |
| Reserved test | 16 | Left unevaluated and unpublished. |

These examples were assistant-authored and accepted by me for the pilot. That is different from independent expert annotation. The public dataset retains its provenance and review-status fields rather than pretending otherwise.

There is another limitation: I authored extra training examples after seeing earlier development errors. Keeping the development file unchanged did not make the overall experiment independent of it. I had already learned from it.

## I measured the simple approaches first

Before renting a GPU, I tried a fixed keyword rule, TF-IDF with logistic regression, and an existing Decision Kai model with a fixed yes/no question.

TF-IDF represents a sentence using weighted word features. Logistic regression learns how those features relate to the two labels. It is a useful first comparison because it can reveal whether a small classification task already responds well to inexpensive methods.

On the final pilot dataset, the development results were:

| Approach | Correct out of 14 | Macro F1 |
| --- | --- | --- |
| Fixed keyword rule | 12 | 0.8542 |
| TF-IDF + logistic regression | 13 | 0.9282 |
| Decision 2.0 Kai with a fixed question | 11 | 0.7754 |
| Selected fine-tuned Vela checkpoint | 14 | 1.0000 |

Macro F1 averages the F1 score for each class. F1 combines precision—how often a predicted class is correct—with recall—how many examples of that class the model finds.

The TF-IDF result was a useful reality check. It was already close on this tiny set. The neural experiment helped me learn fine-tuning, but these numbers do not establish that Vela is the better production choice.

## What I actually changed in Vela

The loaded model had **307,531,778 parameters**. I updated **10,622,210**, roughly 3.5% of them:

- The final two encoder layers, indices 20 and 21.
- A newly initialized task head, including its final two-class projection.

The embeddings and earlier encoder layers stayed frozen. Their parameter values were checked after training to confirm they had not changed.

This was partial fine-tuning of the existing model weights. I did not use LoRA for this run. Although the training tool calls the mode `full`, the explicit trainable-layer setting restricted which parameters could receive updates. The saved artifact is a complete checkpoint, so readers can load it without assembling an adapter.

Why give the head a fresh start? The existing task head’s output meaning was not my new label contract. The pretrained encoder provided the starting representations; the new head learned how to map them to expressed decisions and non-decisions.

```text
short statement
  → tokenizer
  → frozen embeddings and earlier encoder layers
  → final two encoder layers (updated)
  → mean pooling of token representations
  → fresh task head (updated)
  → two class scores
  → cross-entropy against the target label
       └─ gradients update the final encoder layers and task head
```

In this checkpoint, pooling uses the mean of the non-padding token representations. The head then produces two **logits**: raw numerical scores, one for each class. Softmax turns them into scores that sum to one.

During inference, the classifier returns the class with the larger score. During training, the target label tells us which score should be larger.

## What happens during a training step?

Consider a training example with the label `DECISION_EXPRESSED`.

The tokenizer converts its text into token IDs. The encoder processes them, pooling produces a statement representation, and the head produces two logits. Cross-entropy measures how poorly those logits support the correct class.

For one example, the intuition is:

```text
loss = -log(score assigned to the correct class)
```

If the model assigns little probability to the target, the loss is large. Assigning more probability reduces it. In code, cross-entropy takes the logits directly rather than requiring us to calculate softmax first.

Backpropagation calculates how changes to trainable parameters would affect that loss. The optimizer then uses those gradients to update the parameters. Repeating this across labeled examples is how the classifier adapts.

For my run, each microbatch contained two examples. I accumulated gradients over two microbatches before an optimizer update, giving four sampled examples per update. Each microbatch’s mean loss was divided by two so the accumulated gradients reflected the intended average.

Here is the shape of the loop. This is explanatory pseudocode; the [published trainer](https://huggingface.co/TwilightTechie/vela-commitment-classifier-307m/tree/main/reproducibility) includes the actual batching, checks, scheduling and saving:

```python
optimizer.zero_grad()

for batch in two_microbatches:
    with torch.autocast("cuda", dtype=torch.bfloat16):
        logits = model(**batch.inputs).logits
        loss = cross_entropy(logits.float(), batch.labels) / 2
    loss.backward()

clip_grad_norm_(trainable_parameters, max_norm=1.0)
update_learning_rates_for_this_step()
optimizer.step()
```

The optimizer was AdamW with weight decay 0.01. The encoder’s configured learning rate was `2e-5`; the fresh head’s was `1e-4`. A smaller rate for the reused encoder limited how aggressively it changed, while the new head had more room to adapt. Both rates followed six warmup steps and cosine decay across the 100-step run.

A step here means an optimizer update, not an epoch through the dataset. The
100-step run used 400 sampled example draws, with replacement; it was not 100
passes over all 118 training rows.

The forward pass used BF16 autocast. Parameters stayed in FP32, cross-entropy used FP32 logits, and development evaluation ran in FP32. Mixed precision was a computation choice; it did not make every stored weight BF16.

## I prepared locally, then rented one GPU

I did not have a GPU available at the beginning. That did not stop me from defining labels, preparing splits, running CPU baselines or checking the inference path.

For the actual neural training run, I used one NVIDIA L4 through Modal. Its [GPU interface](https://modal.com/docs/guide/gpu) lets a function request a GPU by type. I first ran a bounded 10-step smoke test, then the 100-step pilot.

| Setting | Value |
| --- | --- |
| Training examples | 118 |
| Maximum input length | 256 tokens, including special tokens |
| Microbatch size | 2 |
| Gradient accumulation | 2 |
| Optimizer updates | 100 |
| Development evaluation | Before training, then every 25 steps |
| Random seed | 42 |
| GPU | One NVIDIA L4 |
| Training environment | Python 3.12, Torch 2.8.0+cu128, Transformers 4.57.6 |

The GPU function took about 104 seconds, including checkpoint handling and reload verification. That excludes the broader setup, foundation download and publishing process. Peak allocated training memory was about 1.37 GiB; this is a measurement of this configuration, not a general hardware requirement or the GPU’s total memory use.

I also learned a less glamorous lesson about remote execution: paths available in the local checkout were not automatically available inside the container. An initial import failed because the wrapper assumed a local directory layout. Explicit source mounts and separate local/remote paths fixed it before the successful training runs.

Preparing the data and understanding the execution environment took more attention than the final 100 optimizer updates.

## Step 75 became the published checkpoint

I evaluated the model on the 14 development examples at steps 0, 25, 50, 75 and 100. The numbers correct were 7, 13, 12, 14 and 14.

The selection rule used development macro F1 and retained the first checkpoint to reach the highest score. That was **step 75**. Running 100 updates did not mean the last checkpoint had to be the one I published.

Seeing 14/14 was encouraging. It also made it tempting to say something much larger than the experiment supported.

The model had been selected using those examples. The set was tiny, synthetic, and already involved in my development process. Even one additional mistake would change the reported fraction noticeably. This was a successful development run, not evidence of general accuracy or enterprise readiness.

The next quality experiment needs fresh, realistic examples labeled independently of the predictions. I plan to freeze the checkpoint and label contract, collect an initial 200 statements, resolve annotation disagreements, and inspect precision, recall, F1 and false positives. Related statements from the same conversation should stay together.

If I use that evaluation to tune the model, I will need another untouched set for the next quality claim. The existing 16-example synthetic test set is also too small and too closely related to this authored pilot to settle the broader question.

## A saved model still needs to survive a reload

A training process that returns good predictions is only part of the work. I needed the saved artifact to return them too.

I reloaded the selected checkpoint on the GPU and on my local CPU. Development predictions agreed. The largest probability difference was about `0.00000185`, consistent with small numerical differences between those executions.

I then published the complete checkpoint, tokenizer, configuration, label contract and model card. The release also includes exact trainer-source hashes, the environment and a reproduction recipe. The synthetic train/validation dataset lives in a separate repository.

After upload, I verified the remote files against the local release and loaded the published model through its Hugging Face ID. The two sample predictions and their scores matched the locally packaged checkpoint.

That verifies artifact integrity and usability. It does not replace independent evaluation.

The weights are shared under MIT with Vela/mmBERT attribution recorded. The reproduction code and synthetic dataset are under Apache-2.0. The release notes document the upstream license declarations and absence of separate upstream notice files; they do not invent a copyright holder. The [model card](https://huggingface.co/TwilightTechie/vela-commitment-classifier-307m) records these details alongside the task and limitations.

## Try the model on your CPU

You can run the published checkpoint without a GPU or a hosted endpoint:

```bash
python -m pip install torch transformers==4.57.6
```

```python
from transformers import pipeline

classifier = pipeline(
    "text-classification",
    model="TwilightTechie/vela-commitment-classifier-307m",
    revision="f0e7192426d053573d09143ad1cc08e557eb15de",
    device=-1,
)

for text in [
    "We have decided to update the document today.",
    "I like mountains.",
]:
    if len(classifier.tokenizer(text)["input_ids"]) > 256:
        raise ValueError("This pilot supports inputs up to 256 tokens")
    print(text, classifier(text))
```

The first run downloads approximately 1.1 GiB of weights plus the tokenizer files. The examples return `DECISION_EXPRESSED` and `NO_DECISION_EXPRESSED`, respectively.

Keep the scores in perspective. A score close to one is a softmax output, not a verified probability that the model is correct in your workflow. Calibration remains unmeasured. English short statements are the evaluated scope; long conversations and other languages need their own evaluation.

If you want to repeat the training, the [reproduction README](https://huggingface.co/TwilightTechie/vela-commitment-classifier-307m/blob/f0e7192426d053573d09143ad1cc08e557eb15de/reproducibility/README.md) provides the foundation download and exact command. Use its recorded Linux GPU environment and check the data hashes before running. Matching the recipe does not guarantee identical results across hardware and software environments.

## Where this could lead

For an enterprise workflow, I can imagine this classifier suggesting statements for a human-reviewed decision log. It could also become a candidate signal in a routing workflow after the necessary integration and evaluation.

Neither integration has been completed in this experiment. A useful system would still need conversation context, attribution, handling of canceled or superseded decisions, privacy controls, and a measured tolerance for false positives. “We agreed to deploy” is not permission to deploy.

What changed for me is that fine-tuning now feels concrete. I can trace the path from a sentence to tokens, representations, logits, loss, gradients and updated weights. I can explain which parameters changed and why a particular checkpoint was selected.

My first model is small in ambition and early in its evaluation. But it is downloadable, its training choices are recorded, and I know what experiment comes next.

If you have a narrow classification problem in mind, write a few contrasting examples first. Notice where you disagree with your own labels. Measure a simple baseline. Then, if adapting a pretrained encoder still makes sense, you have a task you can actually train toward.

The question that started this was ordinary: did someone express a decision? Following it all the way to a published checkpoint gave me a much better understanding of what it means to teach a model a new task.

---

**Experiment artifacts:** [model and model card](https://huggingface.co/TwilightTechie/vela-commitment-classifier-307m), [synthetic dataset](https://huggingface.co/datasets/TwilightTechie/commitment-statements-pilot), and [pinned training recipe](https://huggingface.co/TwilightTechie/vela-commitment-classifier-307m/blob/f0e7192426d053573d09143ad1cc08e557eb15de/reproducibility/README.md).

*This is a personal learning experiment derived from Vela, not an official vLLM Semantic Router release or an endorsed production model.*
