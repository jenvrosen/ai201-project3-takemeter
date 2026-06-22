TakeMeter — Planning

A fine-tuned intent classifier for discourse in the r/endometriosis support community.


1. Community

Choice: r/endometriosis (https://www.reddit.com/r/endometriosis/), a peer support community for people managing endometriosis — a chronic, painful, and frequently under-diagnosed condition.

Why this community: Most of the example communities in this project split along a quality axis (hot take vs. analysis). A support community doesn't, and forcing a quality label onto it would be both incoherent and inappropriate — you never want a model that ranks someone's pain post as "low quality." So I'm measuring intent instead: what is the author primarily trying to do?

Why it's a good fit for classification: the discourse is genuinely varied in function even though it's narrow in topic. On any given day the front page mixes people asking urgent medical questions, people narrating a diagnosis or venting through a flare, and people offering hard-won tips and encouragement to others. Those three modes are distinct, they're frequent, and members and mods already read posts through this lens — which is exactly the kind of real, community-grounded distinction the project asks for. It also avoids the trap of a subjective "good/bad" taxonomy.


2. Labels

Each post is classified by its primary communicative function, not its surface form (a question mark does not automatically mean help-seeking).

seeking_help

The post's primary purpose is to get something back from the community — a medical or practical question, advice on treatment or doctors, or an "is this normal?" check.
Test: remove the question and the post loses its reason to exist.


Example A: [replace with a real post] — e.g. "Has anyone had a lap that actually reduced pain long term? Trying to decide if surgery is worth it."
Example B: [replace with a real post] — e.g. "Started a new hormonal med two weeks ago and the headaches are brutal — did this ease up for anyone, or should I call my doctor?"


sharing_experience

The post primarily narrates the author's own situation — a diagnosis update, symptom description, a vent, or a win — with no specific ask.
Test: they'd post it even if no one ever replied.


Example A: [replace with a real post] — e.g. "Finally got my excision surgery scheduled after three years of fighting for it. Feeling hopeful."
Example B: [replace with a real post] — e.g. "Today is a flare day and I just need to say it out loud: this is exhausting."


providing_support

The post primarily offers something outward — information, a resource, a tip that worked, or encouragement aimed at others.
Test: the value of the post is directed at other people, not the author.


Example A: [replace with a real post/comment] — e.g. "PSA: pelvic floor PT made a big difference for me. Here's what to ask for at your first appointment."
Example B: [replace with a real post/comment] — e.g. "For anyone newly diagnosed and scared — it does get more manageable once you find the right care team."


Requirements check: the labels are mutually exclusive (sorted by primary function, with the rule below to break ties), exhaustive enough that I don't expect a meaningful "other" bucket, and grounded in how the community actually operates.


The example posts above are illustrative. Before annotating, I will read 30–40 real posts and replace every placeholder with a real example.




3. Hard edge cases

The genuinely ambiguous post is the vent-plus-question:


"I'm so done being dismissed by doctors. How do I even find a specialist who actually listens?"



It sits between seeking_help and sharing_experience. A trickier variant is grammatically a question but functionally emotional:


"Is anyone else's partner this unsupportive?"



— so I can't shortcut with "has a question mark → seeking_help."

Decision rule (applied identically every time):


If removing the question leaves a complete, self-contained post → sharing_experience.
If the question is load-bearing (the post exists to get an answer) → seeking_help.


Applied: the first example is seeking_help (venting is framing, the specialist question is the point); the second is sharing_experience (rhetorical question, function is venting / seeking solidarity).

A second anticipated boundary is providing_support vs. sharing_experience in a single post that says "here's what happened to me, and here's what I'd tell others." Rule: if the outward advice is the takeaway the post is built around → providing_support; if the personal story dominates and the advice is a closing aside → sharing_experience.


4. Data collection plan


Source: public posts and top-level comments from r/endometriosis. Support-giving lives mostly in replies, so posts-only scraping would starve providing_support. Pulling comments too keeps all three labels healthy.
Targets: at least 200 examples total, split train / validation / test (roughly 70 / 15 / 15). Aim for ≥ 20% per label (the project's balance hint) — ideally close to a 40 / 40 / 20 split across seeking / sharing / providing, since support-providing is genuinely rarer.
If a label is underrepresented after 200 examples: rather than ship an 80/10/10 dataset that teaches the model to predict the majority class, I'll (1) targeted-collect more of the thin class — for providing_support, pull from comment threads on popular help-seeking posts where advice concentrates; and if that still falls short, (2) reconsider whether providing_support should fold into a 2-label seeking-vs-sharing taxonomy on posts alone, which is still valid under the 2–4 rule. I'll decide based on what the real distribution looks like, and document the decision.


Privacy / ethics (this data is sensitive health disclosure): strip usernames, lightly paraphrase identifying details, check Reddit's data terms, and decide on repo visibility (likely private, or scrubbed before any public commit) before committing the CSV — especially relevant if a second annotator sees it.


5. Evaluation metrics

Accuracy alone is the wrong headline here because the classes are imbalanced — if providing_support is ~20% of the data, a model that never predicts it can still score ~80% accuracy while being useless for that class. So:


Macro-F1 (headline metric): averages F1 across the three classes equally, so the rare class counts as much as the common ones. This is the number I'll optimize and report first.
Per-class precision, recall, and F1: because the cost of an error differs by class. In a real community tool, missing a seeking_help post (classifying it as sharing_experience) means someone asking for help doesn't get routed to answers — worse than the reverse. So I care specifically about recall on seeking_help.
Confusion matrix: to see which pairs get confused. My prior is that the seeking ↔ sharing boundary (the vent-plus-question) will be the main error cluster, and the confusion matrix is how I'll confirm or refute that.
Baseline-relative gain: every metric above is also computed for the zero-shot Groq llama-3.3-70b-versatile baseline on the same test set, so I can state how much fine-tuning actually helped rather than reporting an absolute number in a vacuum.



6. Definition of success

Concrete, checkable bars (set before seeing results so I can't move the goalposts):


Beats the baseline: the fine-tuned model's macro-F1 exceeds the zero-shot Groq baseline's macro-F1 on the test set. If it doesn't, fine-tuning didn't earn its keep and I report that honestly.
Macro-F1 ≥ 0.70 on the held-out test set. With three classes, chance is ~0.33, so this is a meaningful margin above guessing while being realistic for a subjective intent task on ~200 examples.
seeking_help recall ≥ 0.80, because that's the costliest class to miss for any real use.


"Good enough for deployment": I'm scoping deployment as a low-stakes assistive tool, e.g. flagging likely-unanswered help requests for mods, or auto-tagging posts — never anything that gates or hides a person's post. For that use, high seeking_help recall matters more than perfect overall accuracy: a few false flags are cheap, a missed help request is not. I'd accept criteria 1–3 above as the deployment bar, with a human always in the loop. I would not deploy anything that makes an irreversible decision about a post in a health support space.


7. AI Tool Plan

There's no implementation code in this project, so AI tools help in three specific places:

Label stress-testing (before annotating)

I'll give an LLM my three label definitions and the edge-case description and ask it to generate 5–10 posts that deliberately sit on the seeking ↔ sharing and sharing ↔ providing boundaries. If I can't classify its outputs cleanly with my decision rules, the definitions need tightening — and I'll do that before labeling 200 examples, not after. Verification: I apply my own written rules to each generated post and note any that resist classification.

Annotation assistance (optional pre-labeling)

Decision: I may use an LLM to pre-label a batch, then review every pre-labeled example myself rather than trusting it. If I do, I'll add a prelabeled boolean column to the CSV so it's tracked, and I'll report in my AI-usage section what fraction were pre-labeled and how often I overrode the model. The final label on every row is my own judgment, not the model's.

Failure analysis (after evaluation)

I'll hand my list of wrong test predictions to an AI tool and ask it to spot systematic patterns (e.g. "short posts get misread," "sarcasm trips it up," "seeking↔sharing is the dominant confusion") before I write the evaluation. I'll then verify each proposed pattern against the actual confusion matrix and specific examples — treating the AI's suggestions as hypotheses to check, not conclusions to copy.

Disclosure

I used an AI assistant (Claude) to help me think through the intent-based taxonomy and to draft and structure this planning document. The community choice, the final label boundaries, the success criteria, and all annotation decisions are mine. I'll keep this disclosure current in the project's AI-usage section as the work progresses.


Milestone checkpoints


 Community chosen and justified.
 2–4 labels defined with examples.
 Hardest edge case named with an explicit decision rule.
 Data collection, metrics, and success criteria specified.
 AI Tool Plan with disclosure.
 30–40 real posts read; placeholder examples replaced with real ones.
 Labels stress-tested against AI-generated boundary posts; definitions tightened if needed.



Outcome vs. plan (added after Milestone 5)

My original success criteria were: beat the zero-shot baseline, macro-F1 ≥ 0.70, and seeking_help recall ≥ 0.80. Recording how that held up against reality:


Beat the baseline: NOT met. The fine-tuned model (accuracy 0.667) performed worse than the zero-shot Groq baseline (0.849), a regression of 0.182.
Macro-F1 ≥ 0.70: NOT met. Actual macro-F1 was 0.27.
seeking_help recall ≥ 0.80: met (recall 1.00), but only because the model predicted seeking_help for every example — so this "success" is an artifact of majority-class collapse, not real skill.


I'm leaving the original criteria above unchanged rather than editing them down, because the gap between what I predicted and what happened is itself the central finding: with ~150 training examples and 65% of them in one class, the model learned to always predict the majority label and never learned the two minority classes. The seeking-vs-sharing boundary I flagged here as hardest to annotate is the exact boundary the model failed on. Full analysis is in the README evaluation report.
