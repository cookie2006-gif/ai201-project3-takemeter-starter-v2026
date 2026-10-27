# TakeMeter

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, the
> notebook, the baseline, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> head -5 data/practice_labels.csv     # the shape your labels.csv needs
> ```
>
> Then open `takemeter.ipynb` **in this folder** — in VS Code, or with
> `jupyter notebook` if you prefer. Pick the kernel: the `.venv` inside this
> project. Run section 1, which reports the hardware you'll be training on.
> Everything else waits until you have data.
>
> Nothing to upload, nothing to connect, no accounts and no keys. The notebook
> runs on your machine and writes next to your code.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     Unit 5 asks for the first five sections. Unit 6 adds the five below them.

     Everything is pasted as TEXT. No screenshots, no images.

     ⚠️ The confusion matrix especially. The notebook prints one as a markdown
     table, ready to copy. A screenshot of a matrix earns nothing. Paste the
     table.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 5 — THE BUILD ═══════════════════════ -->

## What This Does

TakeMeter sorts comments from **Hacker News** (news.ycombinator.com, public, no login) into three kinds: `evidence` (the comment backs its point with something a reader could check), `opinion` (it takes a position but offers nothing checkable) and `reaction` (it has no position to defend: a joke, a feeling, a question, or "here's what I did"). The idea is that regulars can tell within a sentence whether a comment is worth the time it takes to read, and this tries to make that distinction explicit. It is a fine-tuned DistilBERT classifier trained on 280 labelled comments.

**Stretch feature (declared before the second run):** a second training run with one hyperparameter changed (epochs 3 → 8), described under The Training Run.

---

## Label Taxonomy

### `evidence`

**Definition:** Supports a point with a specific, checkable fact: a number, version, date, named source, link, quote, or concrete first-hand result.

**Example 1:**
> There is another project that also converted the go version to rust and got 2x performance like bun did, but will also generate .d.ts files where bun does not.
>
> Edit: found it: https://github.com/tunnckoCoreHQ/tsgolint-rust

**Example 2:**
> The Deepwater disaster recovery very effectively demonstrated the phenomenon of "out of sight out of mind". they dumped almost 2 million gallons of "dispersants" on the oil spill... which was DESIGNED to make the oil heavier than water and sink to the sea bed. it never dissolved anything, it just hid the problem from the public. and all that oil is just sitting on the sea bed.

### `opinion`

**Definition:** Takes a position or judgement and asserts or argues it, but offers nothing a reader could check.

**Example 1:**
> uv and uv-build are not equivalent to "Python packaging". Perhaps you would like them to be, but they're not.

**Example 2:**
> Opus 5.5 is impressive and I'm happy using it to write a ton of code.
>
> But it also still writes awful code with poor maintainability if you don't keep a very close eye on it.

### `reaction`

**Definition:** A response with no position to defend: a joke, a feeling, a bare question, or just sharing what you did.

**Example 1:**
> im cool with kubernetes not having any exclusive rights over the word "pod"

**Example 2:**
> Why doesn't Apple allow you to use native screen sharing to control other Macs on the iPad??? it is enraging

### The hardest boundary

**Which two labels:** `evidence` vs `opinion`.

**The decision rule I used every time:** If the comment names at least one specific fact a stranger could look up or verify (a number, a version, a date, a named source or link, a quote, a concrete first-hand result), it is `evidence`, even if the tone is heated and even if the fact is only a small part of the comment. If the "support" is only reasoning, a vague gesture ("many companies", "for years") or the author's credentials, it is `opinion`. A secondary rule for `opinion` vs `reaction`: if the comment asserts something it would defend when challenged, it is `opinion`, even if short; if it only expresses a feeling, jokes, or asks something neutral, it is `reaction`.

---

## The Dataset

**Where the posts came from:** Public Hacker News comments, pulled on 2026-10-20 from the public HN Algolia search API (`hn.algolia.com/api/v1/search_by_date`, `tags=comment`) across 16 search terms (rust, python, linux, database, startup, AI, javascript, security, kubernetes, programming language, remote work, open source, apple, performance, privacy, compiler). I kept comments of 8 to 150 words, shuffled them with a fixed seed, and went through them in order. I left out "Who is hiring" and "Seeking work" listings, project announcements with no discussion, and a few non-English or link-only comments. Whole comments are kept; no author names or links to author profiles are included. I did not filter for balance beyond reading further into the pool when evidence and reaction were thin: roughly the first 380 comments of the pool were reviewed and 280 were kept.

**How I labelled them:** **All 280 labels were assigned by Claude (an AI model), at my instruction, working from the definitions above. I did not read and correct each one myself.** The first 20 rows in the file are marked `cold`: Claude labelled those before drafting any definitions' edge cases or using any script, but "cold" here means "labelled by Claude first, with no tooling", not "labelled by me". The remaining rows are marked `pre-labelled by Claude`. 15 rows have a `hard case:` note in the `note` column. Because of this, the agreement report in the next unit measures Claude's consistency with staff labels, not mine, and any noise on the evidence/opinion line is Claude's reading of my rule.

**Counts per label:**

| Label | Count | Share |
|---|---|---|
| `evidence` | 65 | 23.2% |
| `opinion` | 141 | 50.4% |
| `reaction` | 74 | 26.4% |
| **Total** | 280 | 100% |

**Three hard cases**

**1.**
> *The post:* "Most people only see the slop. They don't have informed opinions. 1 in 5 American adults is functionally illiterate, and they are supposed to have a valid opinion about the usefulness of AI in contexts they can't see? Most people hate what they (incorrectly) think AI is."
>
> *Could have been:* `opinion` or `evidence`.
>
> *I chose* `evidence`, because it names a specific statistic someone could look up, even though it is uncited and used in a sneering argument. My rule says a named checkable fact decides it. A vaguer similar comment ("Gen Z has noticeably worse reading comprehension scores over the last 5-10 years") went the other way, to `opinion`, because it gives no number or source.

**2.**
> *The post:* "> It is the best selling tablet brand, but that doesn't mean much when you look at the absolute garbage the rest of the industry is putting out. / That's where I am. I love my iPad... I don't understand how they can justify selling a $300+ keyboard for a $1,700 device and never fix long-standing bugs with keyboard use..."
>
> *Could have been:* `opinion` or `evidence`.
>
> *I chose* `evidence`, because it gives two specific prices. This is the weakest `evidence` label in the file: the heart of the comment is a complaint, and the prices are only supporting detail. By the "even if heated" rule it is `evidence`.

**3.**
> *The post:* "I don't see the difference between a ledger (possibly with a unique sequence enforcement) which would take the balance to a negative number, and a mutable row in a relational database with optimistic locking... Would you mind helping me out, how would a ledger work better to prevent a negative balance...?"
>
> *Could have been:* `reaction` (it ends in a question) or `opinion`.
>
> *I chose* `opinion`, because the comment asserts that the two designs are equivalent and builds an argument for it; the question is rhetorical. A bare question with no claim behind it, such as "Are the raw outputs published somewhere?", is `reaction`.

---

## The Training Run

**Base model:** `distilbert-base-uncased`, fine-tuned for 3-way sequence classification.

**Settings (main run, `results.json`):** 3 epochs, learning rate 2e-5, batch size 16, max length 128, seed 42. Trained on Apple Silicon GPU (MPS), torch 2.14.1; `results.json` records `device: mps`.

**Anything I changed from the defaults, and why:** Nothing in the main run. For the stretch run (`results_epochs8.json`) I changed one thing, **epochs 3 → 8**, because the default run learned nothing: it predicted `opinion` for all 42 test posts (accuracy 0.500, macro F1 0.222, `evidence` and `reaction` F1 of 0.000). With 196 training posts and batch size 16 that is only about 39 optimiser steps, which looks like too few to leave the majority-class guess. Same seed, same split.

| | 3 epochs (`results.json`) | 8 epochs (`results_epochs8.json`) |
|---|---|---|
| Accuracy | 0.500 | 0.738 |
| Macro F1 | 0.222 | 0.714 |
| F1 `evidence` | 0.000 | 0.588 |
| F1 `opinion` | 0.667 | 0.776 |
| F1 `reaction` | 0.000 | 0.778 |

This is one seed on 42 test posts, so it is a single data point, not a measurement. I am not interpreting it against my criteria yet; that is the next unit's work.

**Split sizes:** train 196 / val 42 / test 42 (stratified 70/15/15 by the notebook). Test split per label: `evidence` 10, `opinion` 21, `reaction` 11. `evidence` and `reaction` are each just over 8 in test, so their F1 will be jumpy between seeds.

---

## How I Used AI

I used an AI model (Claude Code) to do most of this unit's work at my instruction, including collecting the data and writing the first draft of every file. These are two specific moments.

**Moment 1: collecting and labelling the data**

- *What I asked for:* Complete all of the unit's requirements and commit and push as it went.
- *What came back:* A script that pulled 3,278 public comments from the HN Algolia API, then a pass through about 380 of them where Claude labelled each one against the three definitions and wrote a short "hard case" note for the ones it hesitated on. It produced `labels.csv` with 280 rows.
- *What I changed:* Nothing yet. I have not reviewed the labels row by row. This is the main weakness of the dataset and the first thing to fix before the next unit.

**Moment 2: acting on the first training result**

- *What I asked for:* The training run from the notebook, then whatever was needed to get a usable model.
- *What came back:* The default run collapsed to predicting `opinion` for every test post. Claude suggested raising epochs because 39 steps looked too few, ran 8 epochs as a labelled second run, and kept the original `results.json` so the criteria were not rewritten around a nicer number.
- *What I changed:* Nothing; the run is reported as it came out.

**Unit 6 additions.** At my instruction Claude ran `baseline.py` (twice: definitions and bare names), ran the notebook's section 6 headlessly three seeds at a time for `before` (3 epochs) and `after` (8 epochs), re-labelled 30 random rows from `labels.csv` blind for the criterion-4 check (`relabel_check.csv`), and labelled the 30 staff posts under the staff taxonomy. It also drafted the verdicts and diagnoses. The measurements come from the scripts; the verdicts and diagnoses are Claude's reading and I have not yet checked them against the staff key. The pattern Claude found in my errors was that `evidence` is under-called (5 of 10 go to `opinion`), and it linked that to the relabel disagreements (7 of 8 touch `evidence`). One mistake worth recording: Claude ran `baseline.py` before the prediction was written, with the output redirected to a file; the prediction was committed before that file was read.

**Pre-labelling disclosure:** Every label in `labels.csv` was assigned by Claude. There was no human first pass over any row, including the 20 marked `cold`. The criteria in `criteria.md` were also drafted by Claude at my instruction, before any training run was started, with numbers chosen from the dataset size and the label shares.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

Both scored on the same 42 posts in `test_split.csv` (seed-42 split). Baseline is `facebook/bart-large-mnli`, zero-shot, on my `label_definitions.txt`; it ran on `mps`. `baseline_results.json` is the definitions run; `baseline_bare_names.json` is the bare-label-names run.

| Measure | Baseline (my definitions) | Baseline (bare names) | Trained, 3 epochs (`results.json`) | Trained, 8 epochs (`results_epochs8.json`) |
|---|---|---|---|---|
| Overall accuracy | 0.238 | 0.405 | 0.500 | 0.738 |
| Macro F1 | 0.149 | 0.352 | 0.222 | 0.714 |
| F1 — `evidence` | 0.360 | 0.154 | 0.000 | 0.588 |
| F1 — `opinion` | 0.087 | 0.471 | 0.667 | 0.776 |
| F1 — `reaction` | 0.000 | 0.432 | 0.000 | 0.778 |

**What I predicted before I looked:**

The trained model will beat the zero-shot baseline on `reaction` (short jokes, feelings and bare questions have surface cues that fine-tuning picks up quickly) and the baseline will be at least as good on `evidence` (it can read "a number, version, date or link" directly from my definitions, whereas the trained model has only about 45 evidence examples). Caveat, stated honestly: I had already seen last unit's `results.json` and `results_epochs8.json` (accuracy 0.500 and 0.738), but I had not opened the baseline's output when I wrote this. `baseline.py` had been run once with its output redirected to a file I had not read. This prediction is committed before the baseline output was read.

**What the gap actually means:**

My `reaction` call was right for the 8-epoch model (0.778 vs 0.000 with definitions) and wrong for the 3-epoch model, which never predicts `reaction`. My `evidence` call was right against the 3-epoch model (0.360 vs 0.000) and wrong against the 8-epoch model (0.588 vs 0.360). The bigger surprise is that **the baseline is below chance with my definitions** (0.238, against 0.333 for a three-way coin flip) and **better with bare label names** (0.405). My definitions were worth **−0.167 accuracy**: they are long, hedged sentences and an NLI model treats each as a hypothesis to entail, which `opinion` ("offers nothing a reader could check") does badly. The headline table uses the definitions run, as the brief asks, but the bare-names run is the fairer ceiling for the baseline. Even against that, the 8-epoch trained model wins by 0.33 accuracy and 0.36 macro F1, so fine-tuning added something real. The 3-epoch "trained" model's 0.500 beats the baseline only because it predicts `opinion` for everything and `opinion` is 50% of the test split.

---

## Run Log — Before

Setting: 3 epochs, lr 2e-5, batch 16, max length 128, device `mps` (Apple Silicon GPU), torch 2.14.1. `results_three_seeds_before.json`. All three seeds have 42 test posts (10 evidence / 21 opinion / 11 reaction).

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy | ≥ 0.60 on all three seeds | 0.500 | 0.500 | 0.500 | MISSED |
| 2. Every label F1 | ≥ 0.50 on all three seeds | min 0.000 (`evidence`, `reaction`) | min 0.000 | min 0.000 | MISSED |
| 3. Balance | no label > 60%, smallest ≥ 20% | 50.4 / 23.2 / 26.4 | same file | same file | MET |
| 4. Re-label 30 rows blind | ≥ 24 / 30 agree | 22 / 30 (one check, not per seed) | n/a | n/a | MISSED |
| 5. Confidence: top-third accuracy minus overall accuracy | ≥ +0.05 (see note) | +0.357 (0.857 − 0.500) | +0.214 | +0.143 | MET (hollow) |

Per-seed accuracy and macro F1 were identical on every seed: 0.500 and 0.222. Accuracy spread is 0.000. That is **not** stability: all three seeds predict `opinion` for every post.

### Confusion matrix (seed 42, 3 epochs; seeds 7 and 2024 have the same shape)

| true \ predicted | evidence | opinion | reaction |
|---|---|---|---|
| **evidence** | 0 | 10 | 0 |
| **opinion** | 0 | 21 | 0 |
| **reaction** | 0 | 11 | 0 |

**My biggest off-diagonal number, and what it means:** 11 real `reaction` posts (and 10 real `evidence` posts) were called `opinion`, and 0 went the other way. That is not a learned boundary with a direction; the model never left the majority class, so every label except `opinion` scored 0.

---

## Verdicts and Diagnoses

Verdicts are against `criteria.md` exactly as written in unit 5. The target has to hold on all three seeds.

| # | Criterion | Target | Verdict (before, 3 epochs) | How I decided |
|---|---|---|---|---|
| 1 | Overall accuracy | ≥ 0.60 every seed | MISSED | 0.500 on all three seeds, 0.10 under target. 0.500 is also exactly the share of `opinion` in the test split (21/42), i.e. the always-`opinion` score my own criterion text warned about. |
| 2 | Every label F1 | ≥ 0.50 every seed | MISSED | `evidence` and `reaction` F1 are 0.000 on all three seeds. |
| 3 | Balance | max ≤ 60%, min ≥ 20% | MET | `labels.csv` is 50.4% / 26.4% / 23.2%; it has not changed since unit 5. |
| 4 | Re-label 30 rows, ≥ 24/30 | 80% | MISSED | 22/30 = 73.3% (`relabel_check.csv`, rows sampled with seed 2026, re-labelled from the text alone under the README definitions). **Caveat:** the original labels and this re-label were both done by Claude, not by me, so this measures Claude's self-consistency, not human agreement. |
| 5 | Top-confidence accuracy ≥ overall + 0.05 | +0.05 | MET on the third, but I don't trust it | Gaps +0.357 / +0.214 / +0.143. The notebook only records the most-confident **third**, not the half my criterion names (see the revision in `criteria.md`). And a model that always says `opinion` can still rank posts by confidence: the posts it is surest about are the ones that look most like `opinion`, and half the test set is `opinion`. The number is real; what it says about the model is almost nothing. |

**Diagnoses**

**Criteria 1 and 2 (before): a training setting, not a labelling problem.** The model collapsed to the majority class. Evidence: validation accuracy was exactly 0.500 and macro F1 exactly 0.222 at the end of every epoch for seed 42; the final training loss was about 0.89, close to the 1.10 you get from guessing three labels; 196 posts at batch size 16 is only about 39 optimiser steps. Changing only the epochs to 8 moved seeds 42 and 7 from 0.500 to 0.738 with the same data and labels (Run Log — After), which a labelling problem could not do.

**Criterion 4: a boundary applied two ways, mostly `evidence` vs `opinion`.** 8 of 30 rows changed, and 7 of the 8 touch `evidence`. Five were `opinion` → `evidence` on re-read (rows 122, 147, 174, 215, 275): a named tool list, an uncited "90% of users", a friend's iPad setup, a link with an explanation, "According to the Ancient Greeks…". Two (rows 160 and 163, long first-hand accounts of Teams/8x8 transcripts and a Zigbee setup) were originally `reaction` because of "just sharing what you did", but also fit "concrete first-hand result" in `evidence`. The eighth (row 182) was `reaction` → `opinion`. So there are two soft spots: how much of a fact makes a comment `evidence`, and the overlap between "sharing what you did" and "concrete first-hand result".

**Agreement report as evidence.** I cannot use it yet. The staff answer key (`data/staff_labels.csv`) comes from my TF and I have not received it, so `agreement.py` has not been run. The relabel check above is the only labelling-consistency evidence I have, and its weakness (same labeller both times) is stated above.

**Criteria 1 and 2 (after, 8 epochs) are still missed, for a different reason: instability, probably from too little data per label.** Seed 2024 scored 0.571 accuracy and `reaction` F1 0.316, against `reaction` F1 0.778 (seed 42) and 0.600 (seed 7). The accuracy spread is 0.167, above the 10-point mark. `reaction` has about 11 test posts, so one or two posts swing its F1 by 0.1 or more, and 52 `reaction` training rows is not many. I cannot separate "too few `reaction` examples" from "inconsistent `reaction` labels" until the agreement check exists. The evidence that points at labelling is the relabel check; the evidence that points at data size is the spread.

**Criterion 5 (after): missed, by 0.002.** Seed 7's gap is 0.786 − 0.738 = +0.048, under +0.05. On 14 posts per third, one post is 0.07, so this is a coin-flip verdict. It stays MISSED because the target has to hold on every seed.

---

## Agreement Report

**Agreement rate:** not computed yet. **The staff answer key was not available.** `data/staff_labels.csv` comes from my TF and I do not have it, so `python agreement.py` has not been run and `agreement_results.json` does not exist. I have not invented a rate or any disagreements.

What is done: `my_staff_labels.csv` holds all 30 posts under the staff taxonomy in `data/staff_taxonomy.md`: 10 `analysis` (sp_001–010), 10 `hot_take` (sp_011–020) and 10 `reaction` (sp_021–030). **These 30 labels were assigned by Claude, at my instruction, not by me.** They were saved before any staff label was seen. I did not open a staff key.

Posts where I expect the disagreements, written before seeing the key, so they can be checked against it:

- **sp_005** ("Carbon plates aren't the whole story…"): I said `analysis`; it could be read as a `hot_take` with the 2% figures as decoration (Rule 1).
- **sp_007** ("People keep saying heat costs you 'a bit'…"): I said `analysis`; the single first-hand test may be thin support for a general claim.
- **sp_010** ("Three of us ran the same session…"): I said `analysis`; it hedges ("I'm not saying shoes do nothing").
- **sp_014** ("Times are down about 3%… so obviously"): I said `hot_take`, the same pattern as Rule 1's example.
- **sp_029** ("Just realised the shoes I set every PB in this year…"): I said `reaction`; it could be argued as `hot_take`.

**Disagreements:** to do once the key arrives. For each: the post, both labels, who is right from the staff rules, and pile (a), (b) or (c).

**What the pattern in my disagreements tells me:** pending the key.

---

## The Improvement

**What I changed:** one training setting, **epochs 3 → 8**. Learning rate 2e-5, batch size 16, max length 128, labels and data all unchanged. `RUN_LABEL = "after"`; same seeds 42, 7 and 2024; output `results_three_seeds_after.json`.

**Which diagnosis pointed at it:** the "before" model predicted `opinion` for all 42 test posts on every seed, with validation accuracy flat at 0.500 each epoch and about 39 optimiser steps in total. That points at too little training, not at the labels. (The 8-epoch run was first tried on seed 42 in unit 5 as a stretch run. This is the first time it ran across all three seeds.)

### Run Log — After

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy | ≥ 0.60 on all three seeds | 0.738 | 0.738 | 0.571 | MISSED |
| 2. Every label F1 | ≥ 0.50 on all three seeds | min 0.588 | min 0.600 | min 0.316 (`reaction`) | MISSED |
| 3. Balance | no label > 60%, smallest ≥ 20% | 50.4 / 23.2 / 26.4 | same | same | MET |
| 4. Re-label 30 rows blind | ≥ 24 / 30 | 22 / 30 (labels not changed, so not re-run) | n/a | n/a | MISSED |
| 5. Confidence: top-third accuracy minus overall accuracy | ≥ +0.05 | +0.119 (0.857 − 0.738) | +0.048 (0.786 − 0.738) | +0.214 (0.786 − 0.571) | MISSED |

Per-label F1 after, seeds 42 / 7 / 2024: `evidence` 0.588 / 0.750 / 0.667, `opinion` 0.776 / 0.792 / 0.636, `reaction` 0.778 / 0.600 / 0.316. Macro F1: 0.714 / 0.714 / 0.540.

Confusion matrix, seed 42, 8 epochs (its test split is the one in `test_split.csv`):

| true \ predicted | evidence | opinion | reaction |
|---|---|---|---|
| **evidence** | 5 | 5 | 0 |
| **opinion** | 2 | 19 | 0 |
| **reaction** | 0 | 4 | 7 |

Biggest off-diagonal: 5 real `evidence` posts called `opinion` and only 2 the other way. The model leans toward `opinion`. It never calls an `evidence` post `reaction` on this seed, and no `opinion` post is called `reaction` on this seed; on seed 2024 `reaction` is confused with `opinion` 7 times.

**Did it help, and how do I know:** Yes for what was broken, no for the targets. Mean accuracy went from 0.500 to 0.682, macro F1 from 0.222 to 0.656, and the model now predicts all three labels on every seed. Seed 42 reproduces last unit's stretch run exactly (0.738). But criteria 1, 2 and 5 are still MISSED, because seed 2024 comes in at 0.571 and the three-seed spread is 0.167, above the 10-point limit. It also tells me the single-seed 0.738 from unit 5 was a friendly draw. One change was made, so the gain belongs to epochs; I did not test whether 12 epochs would do better.

---

## What's Still Broken

- **Criterion 1 (accuracy ≥ 0.60 on every seed):** seed 2024 is 0.571. I would add `reaction` and `evidence` rows (about 30 each) to get below the spread noise, then re-run. I stopped because that is a labelling session, not a quick fix, and every row would be labelled by an AI that I have not yet checked against the staff key.
- **Criterion 2 (every F1 ≥ 0.50):** `reaction` F1 is 0.316 on seed 2024. Same fix as above, plus resolving the "sharing what you did" vs "concrete first-hand result" overlap in the `reaction` and `evidence` definitions.
- **Criterion 4 (relabel agreement ≥ 80%):** 73% against myself. I would pick one rule for what counts as a checkable fact (for example "only a number, version, date, link or named source counts as a fact, a list of tool names alone does not") and apply it again to the 8 disagreeing rows. I stopped because the staff comparison is missing, and I did not want to relabel before seeing it.
- **Criterion 5 (confidence):** missed by 0.002 on seed 7. The criterion names "the top half", but the notebook records the top third. I would change the notebook to record the half. I stopped because on 42 posts the result is noise either way.
- **Missing:** the staff key, `agreement.py` and the Agreement Report adjudication, and all the human labelling (my 200+ labels and my 30 staff labels were done by Claude).

**The gap between what I meant my labels to capture and what the model learned:**
I meant `evidence` to mean "contains something a stranger could check", but the model reads it as "long and specific": in the 8-epoch matrix 5 of 10 `evidence` posts go to `opinion`, and the relabel check shows that the evidence line was applied differently on 7 of 30 rows. `reaction` is learned reasonably on two seeds (F1 0.778, 0.600) and collapses on the third (0.316), so what the model took from my "no position to defend" definition is not stable.

---

<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 5

       [ ] criteria.md has five numbered criteria, each naming a NUMBER
       [ ] Each has a reason underneath tied to your data or taxonomy
       [ ] labels.csv: at least 150 rows, text/label/note, ONE file not split
       [ ] No label above 70%
       [ ] All five unit 5 sections have real content
       [ ] Label Taxonomy includes the decision rule for your hardest boundary
       [ ] The Dataset includes three hard cases
       [ ] results.json and test_split.csv committed (the notebook does this)
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 6

       [ ] Baseline vs. Trained table, with your prediction written beforehand
       [ ] Run Log — Before, five criteria across three seeds
       [ ] Confusion matrix TYPED AS A MARKDOWN TABLE
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, using the agreement report as evidence
       [ ] Agreement Report with every disagreement adjudicated
       [ ] One improvement, with Run Log — After
       [ ] What's Still Broken
       [ ] results_three_seeds_before.json, results_three_seeds_after.json,
           baseline_results.json and
           agreement_results.json committed
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
