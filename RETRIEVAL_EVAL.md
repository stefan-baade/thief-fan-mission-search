# Retrieval Evaluation

Fan Mission Search finds one of roughly 1,400 community-made releases ("fan
missions") for *Thief: The Dark Project* / *Thief Gold* and *Thief II: The Metal
Age*, from a player's half-remembered description ([README](README.md)).

This document is how that search was measured: one baseline, six strategies rejected
on evidence, one defect diagnosed and fixed, and the part that is still open. Every
number below comes from a run against a written-down gold set, not from a vendor
benchmark and not from reading result lists and liking them.

The short version: **the winning change came out of diagnosing a specific bug in
how scores were combined, not out of tuning parameters.** Five of the six rejected
strategies were parameter or weighting variants. All five lost.

> **Reading this document.** Individual queries are described by shape, or carry an
> arbitrary label where a table needs rows (`R…` for the real forum questions, `M…`
> for the level-reach measurement). Those labels are stable within this document and
> carry no published mapping to content. Corpus material, package names and file
> paths are deliberately absent — see the note at the end of the [README](README.md).

---

## 1. Setup

**Corpus.** 1,430 packages, 1,913 playable levels, **84,090 chunks** in two
indexes. One package is one release, and holds either a single level or a whole
campaign. Chunks come from prose walkthroughs, loot tables and spreadsheets,
package readmes, in-game readable text, objectives and level titles. Images inside
those guides are deliberately *not* indexed: an image cannot be chunked, ranked or
quoted back as evidence.

Everything in this document ran on one machine — lexical index, embeddings,
reranking and fusion. No step described here sends content to a third party.

**Three retrieval tracks, fused by Reciprocal Rank Fusion (k=60):**

| Track | Model / method | What it is supposed to carry |
|---|---|---|
| Lexical | BM25 (SQLite FTS5) | Proper nouns, place names, item names |
| Dense | BGE-M3, FP16 on a 16 GB consumer GPU | Paraphrase, and German questions against an English corpus |
| Reranker | BGE-reranker-v2-m3, self-hosted | Query-and-chunk scored together, over the union of the two raw pools |

The reranker scores the chunk body **without** the level-title prefix that each
chunk carries for readability; including the prefix in the score was measured and
was worse (4/22 vs. 6/22).

**Gold sets.**

| Set | n | Nature |
|---|---:|---|
| Frozen | 22 + 2 controls | English queries written before any run; the two controls must stay empty |
| Second split | 18 | Loot-table-shaped questions, written later, deliberately not merged into the frozen set |
| Diagnostic | 1 | One package used to isolate the fusion defect |
| Real forum questions | 10 | Verbatim "help me find this mission" questions with the answer the thread confirmed |

**Gold scope is written on every pair**, and it is two different conditions:

- **package scope** — a hit if any row from the right version group is in the top *k*,
  including a package-level row.
- **mission scope** — a hit only if a row carrying the right level id appears.

Seven of the frozen 22 are mission-scope. They are therefore judged under a
strictly harder condition than the rest, which is a known weakness of folding both
into one `hit@10`.

**Honesty constraints I held to, because they cost score:**

- The frozen set was written first and never rewritten to match results.
- The second split includes packages *with* a prose walkthrough and *without* one.
  Measuring only the guided half would have produced a flattering and useless number.
- The two honest-miss controls stay in every run; a change that fills them is a
  regression, not an improvement.
- Every arm runs through the same code the application uses, not through a
  convenience script that skips a stage. Where an arm did diverge, it is marked.

**One caveat on comparing numbers across this document.** The index was rebuilt
several times during these runs (the chunk count above is the current build; the
first baseline was measured on an earlier one of 78,153 chunks), and some arms ran
in separate scripts with slightly different pools. Absolute scores are therefore
comparable **within** a section, not across sections. Where an arm's own baseline
differs, it is stated. §5.5 explains why that is a hard rule here and not pedantry.

---

## 2. Baseline

Three tracks, RRF fused **at chunk level**, then collapsed to one row per level:

**6 / 22 `hit@10`, 0 / 22 `hit@1`**, both controls correctly empty.

Adding loot-table and spreadsheet chunks to the index later moved the frozen set to
**5 / 22** — a measured *cost*, accepted on purpose: the frozen 22 contain no
question whose answer lives only in a loot table, so they can measure the cost of
that content and not its benefit. The benefit was measured separately on the
18-query split (**12 / 18** `hit@10`). Reverting on the 5/22 alone would have been
optimising the metric instead of the product.

### Diagnosis came before tuning

For three clear single-track misses I went into the per-track scores instead of
reaching for a knob. The result was uncomfortable and useful: **the losing tracks
were not broken.** The corpus simply holds closer competitors. In one case the gold
walkthrough line loses to another package's item table on *both* the reranker
(0.91 vs. 0.24) and the dense track (the exact-phrase chunk ranks 751 of 20,000).

That reframed everything after it. These were not tuning slack; they were real weak
spots, and a weight that "fixed" them would be fitting 22 rows.

---

## 3. Six strategies, measured and rejected

### 3.1 Rare-term reweighting in the lexical query

**Hypothesis.** BM25 scores an OR of all query tokens, so frequent words dilute the
one distinguishing rare word. Measured document frequencies over the real index (via
the full-text vocabulary table, not a substring scan): a common object noun appears
in **11 %** of all chunks, a domain-generic noun in **10 %**, while the rare noun
that actually identifies the target level sits at **0.06 %** — a factor of roughly
180, treated by the scorer as broadly comparable evidence.

**What I built.** Two query-rewriting variants, in a throwaway script only — the
product's query builder was never touched:

- **S1** — the two rarest content tokens become mandatory (AND); the rest stay OR.
- **S3** — drop unigrams above 5 % corpus document frequency *unless* they sit
  inside a literal, stopword-unbroken 2- or 3-gram. This protects a real
  three-word phrase without accidentally promoting two unrelated words that happen
  to sit next to each other.

**Measured** at BM25 alone *and* after the full three-track fusion, against the
frozen 22 plus two hand probes (one long query, one single-word rare-noun query
with known gold).

| Variant | Holds previously-hit queries | Lifts the target case into top 10 | Score over 7 relevant cases |
|---|---|---|---:|
| Baseline | 6 / 6 | no (rank 18) | **6 / 7** |
| S3 alone | 5 / 6 | no (rank 18 → 12) | 5 / 7 |
| S1 alone | 4 / 6 | no | 4 / 7 |
| S1 + S3 | 4 / 6 | **yes (rank 3)** | 5 / 7 |

**Why rejected.** The AND-gate assumes the second-rarest query word is
load-bearing. Often it is a verb or a gerund that the gold document simply phrases
differently — and then the gate deletes the right document outright. The one clean
win needs the harshest variant, which also breaks the most. No variant beats the
baseline; the best ties one below it.

**Explicitly left open, not claimed impossible:** an *additive-only* phrase bonus,
where a locked n-gram adds one extra OR clause and no unigram is ever removed. It
carries the same 22-query overfitting risk as every parameter already tuned, so it
was set aside rather than tried.

### 3.2 A sparse-lexical track from the dense model's pretrained head

**Hypothesis.** The dense model in the pipeline also produces *learned* per-token
weights — roughly IDF's job, but trained end-to-end rather than derived from this
corpus's frequencies. Adding those as a fourth fusion track would supply word
importance without the corpus-frequency blindness of 3.1, at almost no cost: the
model is already loaded.

**What I built.** A cheap pre-test *before* any whole-corpus encode: the 22 eval
queries, the two hand probes, and a handful of hand-picked gold-vs-competitor chunk
pairs, scored with the model's own lexical matching function.

**The first pass was invalid, and catching that mattered more than the result.**
The local model snapshot was missing the two head weight files; the library silently
random-initialised them and produced plausible-looking numbers. Fixed by fetching
the original heads (~3.5 KB and ~2.1 MB, no corpus encode needed) and re-running.

**Measured, with real pretrained weights.** Rare-token pieces now carry real weight
(~0.15–0.25) — but that is the *same range* as the common nouns from 3.1 (0.27 and
0.29). The learned importance is calibrated to the model's own pretraining corpus,
not to this one.

| Gold-vs-competitor pair | Gold score | Competitor | Verdict |
|---|---:|---:|---|
| Rare-term case | 0.118 | **0.221** | wrong document |
| Hand probe 1 | 0.110 | **0.142** | wrong document |
| Hand probe 2 | 0.083 | **0.110** | wrong document |
| Fourth case | **0.104** | 0.083 | gold |
| Two further cases | — | — | at the noise floor |

**Why rejected — and this is the structural finding of the whole exercise.** A
sparse dot-product score sums products over matching tokens, exactly as BM25 does.
A document with many medium-weight overlaps can therefore still outscore the one
document carrying the single decisive rare term — **independent of where the
per-token weights come from.** Corpus IDF or a pretrained head makes no difference
to that shape.

### 3.3 Training a corpus-specific model

**Considered, not pursued** — and rejected by the finding above rather than by
effort. Classic TF-IDF / vector-space models and unsupervised embeddings trained on
this corpus (Word2Vec, GloVe, LSA) would learn corpus-native word statistics but
still combine evidence as a sum over overlapping terms, so the diagnosed failure
mode survives the retraining.

The one architecture that targets the failure mode directly is end-to-end supervised
learning-to-rank on labelled query→document pairs. That needs hundreds to thousands
of pairs. With 22 queries, such a model would memorise the eval set, and the
resulting number would mean nothing.

Writing down *why* an appealing option is not attempted is part of the record. Not
attempting it without a reason would have been the mistake.

### 3.4 Fusing on the (version group, level) key

**Hypothesis.** Resolve results at level granularity, so a campaign's individual
levels compete as separate rows.

**Measured** on the combined 41 rows: `hit@10` 22, `hit@30` 26 — *and* the
diagnostic case collapses from rank 2 to **36**, another query from 46 to 105.

**Why rejected.** Package-wide chunks — readme, loot table, FAQ prose — carry no
level id by design, because they genuinely describe the whole package. Under this
key they land in a different bucket and stop contributing to the row they support.
The key choice silently disenfranchised the evidence that was doing the most work.

### 3.5 Taking the best 2 or 3 ranks per track instead of the best 1

**Hypothesis.** A package with several relevant chunks is more likely to be the
answer than a package with one, so let more than one chunk per track contribute.

**Measured:**

| Variant | `hit@10` (41 rows) | `hit@30` |
|---|---:|---:|
| Best 1 (shipped) | **22** | **27** |
| Best 2 | 22 | 24 |
| Best 3 | 20 | 24 |

**Why rejected.** Length bias, and it is severe. Campaigns and long walkthroughs
gain (one query 7 → 1, another 20 → 8) while single-chunk answers collapse: 9 → 44,
11 → 33, 20 → 41. The hypothesis was really "more text means more relevant", which
is a property of the corpus, not of the query.

### 3.6 Expanding packages into per-level rows

**Hypothesis.** One row per level surfaces the specific level a mission-scope
question asks for.

**Measured.** With a cap of 2 rows per package: recovers exactly **one** row (to
rank 11 — still outside the top 10) and costs **five** (3 → 5, 7 → 9, 8 → 10,
10 → 12, 12 → 14). `hit@10` over the 41 rows drops by one, 20 → 19 — this arm ran in
a separate script whose own baseline reads 20 rather than the 22 above, which is why
only the delta inside it is quoted (see the caveat in §1). The count of *distinct
packages* inside the top 30 falls from 30.0 to 24.1 — the list stops being a list
of candidates and becomes a list of one package's levels. Uncapped: `hit@10` **16**,
clearly worse.

**Why rejected.** It trades breadth for a precision it does not deliver, because the
levels it promotes are not the ones the query wanted — the wanted level's own text
usually never reaches the pool at all (§5.2, §5.3).

---

## 4. What worked: fusing on the package key

Not a parameter. A defect.

### The defect

RRF was keyed on the **chunk**. When the three tracks each found a *different* chunk
of the same package, their contributions never added up — while a rival whose single
chunk sat at rank 10 in two tracks collected nearly double. Isolated on one
diagnostic package:

| Track | Best raw chunk rank | Which chunk | RRF term |
|---|---:|---|---:|
| BM25 | 10 | package-level loot table | 0.0143 |
| Dense | 12 | that level's objectives | 0.0139 |
| Reranker | 94 | package-level FAQ prose | 0.0065 |

Three tracks, three hits, three different chunks, **no sum.** Fused rank: **39**.

This is the failure mode RRF exists to prevent, and it was invisible in aggregate
scores. It only became visible by printing one package's per-track ranks side by
side.

### The fix

Per track, build a rank map *version group → best rank*, then sum `1/(k + rank)`
across tracks with `k = 60`. The chunk-keyed function stays in place, untouched, so
every rejected run above remains reproducible.

Three details that were not obvious from that one-sentence description:

- The fused row must still be a real hit, so it is **represented by the chunk with
  the lowest raw rank across all tracks** — the chunk carrying the largest single
  `1/(k+rank)` term. Ties go to the track passed first.
- **Keys are tuples, never joined strings.** The version group key already contains
  a separator byte internally (title, author, game). An early evaluation arm joined
  them into a string and scored `None` on all 41 rows — a total failure that looked
  like a catastrophic regression rather than a key bug.
- The old version-collapse step became a **no-op**: measured, 0 of 41 queries had a
  row left for it to remove. It was deliberately left in place for this run so the
  measurement covered the scoring change alone, and scheduled for removal as its
  own step.

### The result

Script arm, 41 rows (frozen 22 + 18 second split + 1 diagnostic):

| Split | n | `hit@10` chunk key | `hit@10` package key | `hit@30` chunk | `hit@30` package |
|---|---:|---:|---:|---:|---:|
| Frozen | 22 | 6 | **8** | 10 | 9 |
| Second split, table-only | 9 | 7 | 7 | 9 | 9 |
| Second split, table vs. prose | 9 | 6 | 6 | 9 | 8 |
| Diagnostic | 1 | 0 | **1** | 0 | 1 |
| **All** | **41** | **19** | **22** | **28** | 27 |

Read the last row as **22 of 41 rows** at `hit@10`, up from 19. The denominator is
41; the frozen set happens to have the same size as the new figure, which invites a
misreading. On the frozen set alone the change reads **6 → 8 of 22**.

The diagnostic case moved **39 → 2**. Individual queries: `None` → 4, 29 → 3,
13 → 2, 4 → 1.

**The price, stated because it is real:** table-only queries tighten (4 → 7, 4 → 8,
5 → 9, 7 → 11). There the answer sits in *one* chunk, so it never had a multi-chunk
lift to gain and now competes against packages that do. `hit@30` overall falls by one.

**Through the product pipeline** (the same call the application makes, including the
reranker and the collapse step) the same change reads **21 / 41 at 10** and
**25 / 41 at 30** — one lower than the script arm at `hit@10`, two lower at
`hit@30`.

**That lower number is the honest one.** The pipeline's gold check rejects a row
whose level id is not the gold level when the pair is mission-scope; the script arm
compared the package key only. A package row inherits the level id of its
representative chunk, so for a multi-level package that level is picked *by
accident*. One query in the script arm matched only by luck.

---

## 5. What is still open

### 5.1 Ten real questions taught more than 41 frozen rows

Ten verbatim "help me find this mission" questions from community forum threads,
each with the answer the thread confirmed, collected by hand and resolved against
the catalog by an exact-match script: **10 of 10 resolved exactly**. Kept as its own
split, never merged into the frozen set.

**`hit@10` 5 / 10 · `hit@30` 6 / 10 · `hit@1` 0 / 10.**

| Case | Rank | Chunks in gold package |
|---|---:|---:|
| R1 | 57 | 33 |
| R2 | none | 127 |
| R3 | 7 | 270 |
| R4 | 10 | 90 |
| R5 | none | 205 |
| R6 | 3 | 188 |
| R7 | 10 | 33 |
| R8 | 23 | 176 |
| R9 | 8 | 61 |
| R10 | 145 | 114 |

**The selection bias stays attached to the number:** these are the questions whose
asker could *not* work it out alone. They are the hard tail, not a cross-section,
and this split must never be quoted as a headline score.

Two observations worth more than the score. **Zero misses come from thin coverage** —
every miss has a well-covered package, so this is a ranking problem, not a data
problem. And two hits survived a *wrong premise*: one asker misremembered a number,
another had the wrong game, and the pipeline found the level anyway. In one case the
human answerer in the thread also guessed wrong; the pipeline's top three were the
right class of level and the wrong level — the same error a knowledgeable human made.

### 5.2 Three readings of that result were wrong first

Recorded because the corrections are the actual work.

**"The corpus does not hold what people remember."** Refuted by measurement. For one
miss, the gold package contains the remembered object term in **34** chunks and two
further remembered terms in 20 each — and appears in no track's top 200. The content
is there; the ranking does not reach it.

**"It is a vocabulary gap."** Partly true, and still the best reading for this split.
The asker remembers a generic word where the corpus uses the level's own proper noun;
*ramp* against *stairs*. What makes it a finding rather than an excuse: **the dense
track returns the gold nowhere near the top for three of these questions** — which is
precisely what that track exists to prevent. Plausible and unmeasured cause: an
800–1000 character block of walkthrough steps has no single centre of meaning for a
"what did the room look like" query. And since the reranker only scores the union of
the two raw pools, whatever both miss is invisible to it by construction.

**"The mission-scope golds are absent from the pool entirely."** **Wrong, and it was
my own measurement error.** The diagnostic script matched the *package*, not the
level, and reported ranks of 40, 67, 38 and 18 — those were package ranks. The
conclusion drawn from them ("a retrieval gap no key choice addresses") was stated
confidently in my own notes and had to be retracted.

What actually holds, measured correctly:

| Case | Chunks in that level | Package rank | Level rank | Package's best-ranked chunk |
|---|---:|---:|---:|---|
| M1 | 6 | 2 | 2 | readme (unbound) |
| M2 | 3 | 40 | 3189 | readable (unbound) |
| M3 | 51 | 67 | 607 | readable, bound to a *different* level |
| M4 | 32 | 38 | 238 | walkthrough, bound to a *different* level |
| M5 | 70 | 7 | 7 | readable, bound to the right level |
| M6 | 7 | 18 | 399 | readable (unbound) |
| M7 | 2 | 337 | 1305 | readable (unbound) |

The **package** reaches the pool; the **level** does not. It rides in on package-wide
text while the level's own text is either tiny (2, 3, 6, 7 chunks) or ranks far below
the cut. One case sits at 238 — just outside a pool of 200.

### 5.3 The measured bottleneck: level binding

Plain SQL over the whole index: **76,097 of 84,090 chunks (90.5 %) carry no level id.**
The five largest sources:

| Source | Unbound | Bound | Unbound share |
|---|---:|---:|---:|
| In-game readable text | 53,538 | 2,166 | 96 % |
| Walkthrough | 18,394 | 849 | 96 % |
| Readme | 3,971 | 0 | 100 % |
| Objective | 85 | 3,143 | 3 % |
| Level title | 0 | 1,835 | 0 % |

**This matches the specification exactly**, which is why it is unbuilt scope and not
breakage: the binding rule was written for archive-internal text with a level
segment in its path. Walkthroughs and readmes were never in that rule; readable text
binds only when the archive happens to use a per-level folder.

**And it matters far less than 90 % suggests.** A single-level package loses nothing
by having unbound text — there is only one level to bind to. The damage sits in
multi-level campaigns: **130** exist, and **95** have at least half their text
unbound. For those, an entire walkthrough sits in the index as one package-level
blob, and no question can aim at one level inside it. The one campaign that *is*
properly split is also the only mission-scope frozen row that lands well.

Two probes (nothing built yet) asked whether those campaigns can be bound:

| Route | Packages | What it needs |
|---|---:|---|
| One guide file per level | 27 | Reading the file layout; no parser |
| Ordinal or titled headings | 47 | `Mission 2:`, `Part Three:`, `Act 1 Scene 3:` |
| No walkthrough at all | 21 | Nothing to bind — a coverage problem |
| Partial structure | 3 | Manual reading |

Level titles are already indexed and bound at 100 %, so nothing has to be guessed —
only matched. **Three traps found in the samples before a single line of splitter was
written:** a loot line that reads as a heading; a cross-reference that points the
reader to the next page; and one campaign that labels two *different* levels with
the same ordinal in two files. Hence the rule for when it is built: **title match
first, ordinal only as fallback**, and ignore lines shaped like table rows. Twenty
cuts get read by eye before that logic goes near the chunker.

### 5.4 A grouping key that splits families it should hold together

Chasing "why does this package have no guide when its sibling has a full one" led
somewhere better than a missing feature. The version group key is title + author +
game, and both fields vary by *spelling* in the catalog: the same author appears with
and without a real name in parentheses, with a nickname in a different position, and
one row is outright truncated with an empty author field. Version families therefore
split across groups, and guides that exist one row over never reach the package that
needs them.

Two probe rounds were wrong in an instructive way before that became visible. Round
one matched every title containing a single letter against a one-letter package title
— a broken catalog row that produced roughly thirty false donor pairs. Round two
ranked candidates by "same author string", which is exactly the signal that does not
work here. What decides it properly is **level titles**, which are bound at 100 %:
comparing those gives clean verdicts (13/13 identical titles, all named in the
donor's guide → inherit; 0 shared levels → reject; identical single level but
different authors → hold, because a same-named level proves nothing).

The fix is not a normalisation rule. A rule that fuses two spellings of one author
will eventually fuse two real authors. It is a **hand-curated correction list**, the
same pattern this project already trusts elsewhere (758 reviewed rows in the guide
map) — curated, checked, never overwritten automatically.

**Shipped, and measurably inert on this eval set: 21/41 and 25/41, unchanged.** That
is the correct outcome, not a failed fix. The corrections merge two version families
(3,300 chunks that were split across four groups), which removes duplicate rows from
result lists — but the two queries still missing are mission-scope, and merging
groups adds chunks without adding a single *bound* one. Their fix is §5.3, not this.

A related catalog-health scan over all 1,388 distinct catalog title rows found
exactly **five** untrustworthy titles (two mis-decoded from another codepage,
three that took a single-letter filename as the title) — 0.4 %. Worth recording
for two reasons: those five rows wrecked the sibling probe above and will wreck
any future fuzzy title matching the same way; and the scan's *first* pass flagged
29 by calling every short title broken, which was simply wrong — several real
levels have three- and four-letter names. A screen full of false positives is how
a real two-row problem gets ignored.

### 5.5 Comparisons hold only within one index

Two consecutive pipeline runs on an unchanged index are **identical row for row**;
inference is deterministic. But across an index rebuild, rows moved with no change to
their content at all: 1 → 3, 3 → 5, 2 → 3, 20 → 22, 3 → 2. Same text, same vectors,
different neighbour graph — the vector store rebuilds its approximate index, and
approximate search returns slightly different neighbours afterwards.

Two consequences, and they matter more than any single score:

- **Take a fresh baseline after every rebuild.** A before/after that straddles a
  rebuild measures the graph as much as the change.
- **Do not explain small differences away as noise.** Earlier notes of mine waved off
  ±1 rank differences as FP16 inference noise. There is no such noise within an
  index. The moves were real — they just were not caused by what was being tested.

A related isolate, run for the same reason: the dense encoder in FP32 rather than
FP16 scored **4/22** against FP16's 5/22, and the query that had been suspected of
being a precision casualty returned the *same* ten rows in both. The wall-clock gap
(16 vs. 40 minutes) was not a clean comparison either, because another GPU workload
was running during one of them — so it is reported as unusable rather than as a
speedup.

---

## 6. What this evaluation is worth, and what it is not

**A lower bound on a small sample.** 22 + 18 + 10 queries. `hit@10` over 41 mixed
rows folds two different gold conditions into one number; package-found and
mission-found belong in **separate columns**, and that change is scheduled ahead of
further ranking work. Without it, a drop cannot be read as a retrieval failure or a
level-resolution failure.

**Overfitting risk, named.** Every parameter touched here — the two BM25 constants,
the RRF constant, pool depth — was tuned against the same 22 queries. That is why
the one change that shipped is a defect fix with a mechanical explanation rather
than a value that happened to score well, and why the remaining untried weighting
idea is documented as set aside rather than tried.

**Ranking work that is deliberately *not* next:** any further fusion-key or
weighting variant. Six were measured; five lost and one shipped. The yield from that
direction is exhausted, and the measured bottleneck is level binding.

---

## 7. What I would take to the next retrieval project

- **Diagnose before tuning.** The one change that worked came from printing a single
  package's per-track ranks side by side. No aggregate score would have shown it —
  and five weighting variants measured around it all lost.
- **A failure mode can be structural.** Both rare-term attempts failed for the same
  reason: a score that sums over matching terms lets many medium matches beat one
  decisive rare match. Knowing that ruled out a third idea (train a corpus-specific
  model) before it cost a week.
- **Keys are architecture.** Three of six rejected strategies failed on the grouping
  key alone — chunk vs. package vs. level, and which evidence each key silently
  excludes.
- **Write the gold set first and never rewrite it.** Including the honest-miss
  controls, including the half of the corpus with weaker source material, including
  the cases that make the number look worse.
- **Retract your own conclusions in writing.** Three readings of one result were
  wrong, one of them because my diagnostic script measured the wrong key. Each is
  recorded next to the correction, because a note that only contains the final answer
  cannot be checked.
- **Format rules on broken input produce clean-looking errors.** Confirmed three
  separate times in the extraction layer. The lever is extraction, not a stricter
  instruction to whatever consumes the text.
- **Real questions measure the product; frozen sets measure regressions.** Ten
  verbatim forum questions exposed more in one run than 41 frozen rows did all day.
