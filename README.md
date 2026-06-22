# ai201-project3-takemeter

TakeMeter: Intent Classification for an Endometriosis Support Community

A fine-tuned text classifier that labels posts and comments from r/endometriosis by their communicative intent, built for the "Show What You Know" project.


1. Overview

Most communities in this project split along a quality axis (hot take vs. analysis). A chronic-illness support community does not, and forcing a quality label onto someone's pain post would be both incoherent and inappropriate. So instead of measuring quality, TakeMeter classifies each post by what the author is primarily trying to do:


seeking_help — the post's main goal is to get something from the community: a question, a request for advice or recommendations, or an "is this normal?" check. Test: remove the question and the post loses its reason to exist.
sharing_experience — the post narrates the author's own situation (a vent, update, or story) with no specific request. Test: they'd post it even if no one replied.
providing_support — the post offers something outward: information, a resource, a tip that worked, or encouragement aimed at others. Test: the value is directed at other people, not the author.


These distinctions matter because they reflect how members and moderators actually read the sub: someone asking "is this normal?" needs a different response than someone venting after a hard day.


2. Dataset


Source: public posts and top-level comments from r/endometriosis.
Size: [FILL IN final row count] labeled examples, split 70/15/15 into train/validation/test (stratified by label). Test set = 33 examples.
Collection: posts were collected manually by reading the feed; comments were pulled from the threads of popular posts, because the two non-majority classes (providing_support, sharing_experience) live mostly in comments rather than top-level posts.
Label distribution: seeking_help [FILL IN] (~65%), sharing_experience [FILL IN] (~17%), providing_support [FILL IN] (~18%).


Labeling process and hard decisions

Each example was labeled by its primary communicative function. Decision rules used to break ties (full reasoning in planning.md):


Vent-plus-question: if removing the question leaves a complete, self-contained post → sharing_experience; if the question is load-bearing → seeking_help.
Share-vs-provide: a personal result with no actionable recommendation is sharing_experience; it becomes providing_support only when it tells the reader what to do or ask for.


Three genuinely difficult examples:


[FILL IN — e.g. the depo-shot post: a positive update that also ends with a real "can I say I have it?" question. Decided sharing_experience because removing the question leaves a complete success-story share.]
[FILL IN]
[FILL IN]


Ethics note

Posts in this community contain sensitive health disclosures. Usernames were stripped and identifying details lightly paraphrased. Self-identified minors and one detailed account of partner abuse were excluded. Several posts contained crisis or self-harm language; [describe how you handled these — scrubbed / excluded / flagged].


3. Model and Training


Base model: distilbert-base-uncased (HuggingFace).
Approach: fine-tuned a 3-class sequence classification head on the labeled training set.
Hyperparameters: 3 epochs, learning rate 2e-5, train batch size 16, max sequence length 256.
Hyperparameter decision: kept 3 epochs because with only ~150 training examples, more epochs risk overfitting, and validation accuracy had already plateaued by epoch 2 (0.636 at both epoch 2 and 3), confirming additional epochs would not help. Training loss kept falling (1.10 → 0.90) while validation accuracy stalled — mild fitting without generalization.



4. Evaluation Report

Overall accuracy

ModelTest accuracyZero-shot baseline (Groq llama-3.3-70b-versatile)0.849Fine-tuned DistilBERT0.667Difference (fine-tuned − baseline)−0.182

The fine-tuned model performed worse than the zero-shot baseline by 18 points. This is the headline result, and it is explained entirely by the fine-tuned model's behavior below.

The fine-tuned accuracy of 0.667 is also misleading on its own: it equals the proportion of the test set that is seeking_help (22 of 33). A classifier that predicts seeking_help for every input scores exactly this — which is precisely what the fine-tuned model does.

Per-class metrics — fine-tuned model

LabelPrecisionRecallF1Supportseeking_help0.671.000.8022sharing_experience0.000.000.005providing_support0.000.000.006macro avg0.220.330.2733weighted avg0.440.670.5333

Per-class metrics — zero-shot baseline


[FILL IN from your Section 5 output — the classification_report printed for the baseline. The JSON only stored overall accuracy (0.849), so copy the baseline per-class precision/recall/F1 here. The point to highlight: unlike the fine-tuned model, the baseline predicted all three classes and so has non-zero F1 for sharing_experience and providing_support.]



LabelPrecisionRecallF1Supportseeking_help[FILL IN][FILL IN][FILL IN]22sharing_experience[FILL IN][FILL IN][FILL IN]5providing_support[FILL IN][FILL IN][FILL IN]6

Confusion matrix — fine-tuned model (test set)

Rows = true label, columns = predicted label. (See also confusion_matrix.png.)

True \ Predictedseeking_helpsharing_experienceproviding_supportseeking_help2200sharing_experience500providing_support600

The entire predicted mass falls in the seeking_help column. The model never once predicted sharing_experience or providing_support.

Error analysis (3 representative failures)

All 11 errors share one direction: every misclassification is → seeking_help, with confidence in a narrow 0.39–0.44 band (barely above the 0.33 three-class chance floor). The model isn't confidently wrong — it's uniformly uncertain and defaulting to the majority label.

Error 1 — providing_support → seeking_help (conf. 0.41)


"Cheap supplies! As someone with endo I use a lot of pads and ibuprofen. Best place to buy is Dollar General..."



A textbook providing_support post — an outward, money-saving tip for others. The model ignored the outward-advice signal entirely. The boundary it failed on is "is the value directed at the author or at others?", which it never learned.

Error 2 — providing_support → seeking_help (conf. 0.39)


"I had both tubes removed in order to have kids — I now have two perfect boys via IVF."



A reassurance comment offering hope to the original poster. The model has no representation of encouragement-as-support and defaulted to the majority class.

Error 3 — sharing_experience → seeking_help (conf. 0.40)


"Why I think ceasing exlaps is a bad thing. They took away the one tool to properly diagnose..."



An opinion/vent with no request. The model could not detect the absence of an ask. This is the seeking-vs-sharing boundary I flagged during annotation as the hardest to label — the same boundary the model failed to learn.

Which labels are confused, and why that boundary is hard: the confusion is entirely one-directional (everything → seeking_help), so this is not really a confusion between two specific classes — it is the collapse of two classes into the majority one. The boundary is hard for this model because the defining signals of the minority classes (outward-directed value; absence of a request) are subtle and under-represented: with only ~150 training rows and 65% in one class, there were too few providing_support / sharing_experience examples for the model to learn those signals.

Is this a labeling problem or a data problem? I labeled these examples consistently against my decision rules (verified by re-reading them), so the failure is not annotation inconsistency. It is a training-data-distribution problem: the loss is minimized by always predicting the majority class, so the minority boundaries were never learned. The fact that the zero-shot baseline — which never saw my training data — handled all three classes far better confirms the task itself is learnable; the problem is my small, imbalanced training set.

Patterns I tested and discarded: using an AI tool, I surfaced candidate patterns and then verified each by hand. I checked whether short posts or sarcastic posts failed more often. Neither held — the failing posts range from very short to very long and none are notably sarcastic. I discarded both hypotheses. The only real pattern is total majority-class collapse.

What would fix it: rebalance the training data (subsample seeking_help to ~50, or apply class weights in the loss) so the model is forced to learn the minority classes, and collect more providing_support / sharing_experience examples to raise their floor.

Sample Classifications (fine-tuned model)

Post (truncated)PredictedConfidence[FILL IN a correctly-predicted seeking_help post]seeking_help[FILL IN]"Cheap supplies! ... Best place to buy is Dollar General"seeking_help0.41"I had both tubes removed ... two perfect boys via IVF."seeking_help0.39"Why I think ceasing exlaps is a bad thing..."seeking_help0.40

Why one correct prediction is reasonable: [FILL IN — e.g. "the seeking_help post above contains an explicit question and request for advice; predicting seeking_help is reasonable because that is the surface signal the class is defined by. It is also the class the model defaults to, so this is a case where the model's prior and the true label happen to coincide — the prediction is right for a shallow reason, not because the model distinguished it from the other classes."]


5. Reflection: Intended vs. Learned Behavior

I intended a three-way intent classifier that separates asking, narrating, and helping. What the model actually learned was a single decision rule: predict seeking_help always. It overfit to the base rate of the dominant class rather than to any linguistic feature of intent, and it captured none of the outward-directed-value signal that defines providing_support or the no-request signal that defines sharing_experience.

The gap is instructive. My label definitions are clean — a human, and (per the baseline) a large zero-shot LLM, can apply them and distinguish all three classes. But a small model trained on a heavily imbalanced sample does not learn the definition; it learns the distribution. The single most telling result in this project is that fine-tuning made the model worse than doing nothing task-specific at all (0.667 vs. 0.849): the zero-shot model already understood intent, and my training data actively taught DistilBERT to discard two of the three categories. The boundary the model most conspicuously missed — seeking vs. sharing — is the exact boundary I identified as hardest during label design, which suggests the difficulty is real and inherent to the task, not an artifact of sloppy annotation.


6. Spec Reflection

One way the spec helped: [FILL IN — e.g. "the requirement to design a strong, mutually-exclusive taxonomy forced me to abandon a vague 'good take / bad take' framing early and adopt an intent-based taxonomy a second reader could apply consistently."]

One way my implementation diverged and why: [FILL IN — e.g. "the spec frames collection around top-level posts, but the three classes are not recoverable from posts alone — providing_support lives almost entirely in comments — so I diverged to a posts-plus-comments collection frame to meet the per-class balance requirement."]


7. AI Usage

Instance 1 — Annotation assistance (pre-labeling): I directed an AI assistant to pre-label batches of posts and comments I had collected, against my own label definitions. It produced a tentative label and one-line rationale per row; I reviewed and corrected every row myself, overriding it on the borderline vent-plus-question and share-vs-provide cases. The final label on every row is my own judgment; pre-labeled rows are marked in a prelabeled column.

Instance 2 — Error pattern analysis: I pasted my 11 misclassified test examples into an AI tool and asked it to identify common themes. It surfaced the dominant → seeking_help collapse and the tight low-confidence band, which I verified by re-reading the examples. It also proposed "short posts fail more" and "sarcasm," which I checked and discarded because my errors do not support them.

[Optional Instance 3 — disclose if true, e.g. "I used an AI assistant to debug the Colab notebook (import errors, the Groq prompt that was returning placeholder tokens) and to draft the structure of this README, which I then verified and edited."]


8. How to Run

[FILL IN — how to open the Colab notebook, upload takemeter_dataset.csv, set the Groq key via Colab Secrets, set runtime to T4 GPU, and run each section in order. Committed artifacts: takemeter_dataset.csv, evaluation_results.json, confusion_matrix.png.]
