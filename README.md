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

**Pre-labelling disclosure:** Every label in `labels.csv` was assigned by Claude. There was no human first pass over any row, including the 20 marked `cold`. The criteria in `criteria.md` were also drafted by Claude at my instruction, before any training run was started, with numbers chosen from the dataset size and the label shares.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

<!-- Both models on the same posts. `python baseline.py --trained results.json`
     prints this table for you. -->

| Measure | Baseline | Trained | Difference |
|---|---|---|---|
| Overall accuracy |  |  |  |
| Macro F1 |  |  |  |
| F1 — `label_one` |  |  |  |
| F1 — `label_two` |  |  |  |

**What I predicted before I looked:**
<!-- Milestone 1 asks you to write this BEFORE seeing the trained numbers. A
     prediction made afterwards isn't one. -->

**What the gap actually means:**
<!-- If the baseline matched your trained model, your fine-tuning added
     nothing — and that is a real finding, not a failure. Say it plainly. -->



---

## Run Log — Before

<!-- Five criteria across three seeds. The notebook's section 6 prints the
     spread table; the Target and Verdict columns are yours. -->

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

### Confusion matrix

<!-- ⚠️ TYPED AS A MARKDOWN TABLE. The notebook prints one ready to paste.
     An image of a matrix earns nothing. -->

| true \ predicted |  |  |  |
|---|---|---|---|
| **** |  |  |  |
| **** |  |  |  |
| **** |  |  |  |

**My biggest off-diagonal number, and what it means:**
<!-- Not "the model made mistakes" — WHICH boundary it didn't learn, and which
     direction. "7 real analysis posts were called hot_take and only 3 went the
     other way" is a direction, not just an error rate. -->



---

## Verdicts and Diagnoses

<!-- MET or MISSED against LAST UNIT's target. The target has to hold across
     all three seeds, not turn up sometimes. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**

<!-- For each miss: the cause, and how you know. The four common causes are:
     too few examples for a label, a boundary you applied inconsistently, a
     genuinely hard label pair, and a task the model can't reach from this
     much data.

     ⚠️ Use your agreement report as evidence. It is the only instrument you
     have that can tell a LABELLING problem from a MODEL problem, and this
     section is graded on whether you used it that way. -->



---

## Agreement Report

<!-- Your rate against the staff set, and every disagreement adjudicated.

     Remember you labelled these 30 under the STAFF taxonomy in
     data/staff_taxonomy.md, not your own — so every argument below is made
     from those definitions and those decision rules. -->

**Agreement rate:** ___ / 30 = ___%

<!-- Nobody grades this number. A 60% who argues every disagreement from the
     stated rules beats a 95% who wrote "staff was right" nine times. Several
     of the 30 were chosen because they're genuinely ambiguous — you should be
     winning some of these. -->

**Disagreements**

<!-- Three lines each: the post, both labels, and who you think is right and
     why — grounded in the staff definitions you were both applying.

     Then sort each into one of three piles:
       (a) the rule covered it and I applied it loosely → a consistency problem
       (b) the rule genuinely doesn't say               → a gap in the definitions
       (c) the rule is ambiguous here and my reading is defensible → argue it.
           This is a legitimate win.

     Pile (a) is the one that matters most for your diagnosis: if you applied a
     written rule two different ways on 30 posts, that is direct evidence about
     what you did across your own 200. -->

**1.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**2.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**What the pattern in my disagreements tells me:**



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Run Log — After

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it didn't, say so. Relabelling that didn't help is a genuinely
     interesting result and earns full credit. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped. -->



**The gap between what I meant my labels to capture and what the model
learned:**
<!-- Two sentences. Your confusion matrix is the evidence. -->



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
