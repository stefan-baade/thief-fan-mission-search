# Retrieval Evaluation

Fan Mission Search finds one of roughly 1,400 community-made releases ("fan
missions") for *Thief: The Dark Project* / *Thief Gold* and *Thief II: The Metal
Age*, from a player's half-remembered description ([README](README.md)).

This document is how that search was measured: one baseline, six strategies rejected
on evidence, one defect diagnosed and fixed, one query-side step measured against
that baseline and integrated, and the part that is still open. Every number below
comes from a run against a written-down gold set, not from a vendor benchmark and
not from reading result lists and liking them.

The short version: **the winning change came out of diagnosing a specific bug in
how scores were combined, not out of tuning parameters.** Five of the six rejected
strategies were parameter or weighting variants. All five lost. The later
query-side change had to clear a rule written down before its re-samples.

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
reranking and fusion — with one exception, described in §6: the query-keyword
step sends the query text, and nothing else, to a hosted language model. No
corpus text is sent to a third party.

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
| Date query | 1 | One of my own searches that also states the release window; I found the answer afterwards by other means |

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
| Best 1 (kept) | **22** | **27** |
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

**"The corpus does not hold what people remember."** Refuted by measurement for the
misses in this split. For one miss, the gold package contains the remembered object
term in **34** chunks and two further remembered terms in 20 each — and appears in
no track's top 200. The content is there; the ranking does not reach it.

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

Two probes (before any binding was built) asked whether those campaigns can be bound:

| Route | Packages | What it needs |
|---|---:|---|
| One guide file per level | 27 | Reading the file layout; no parser |
| Ordinal or titled headings | 47 | `Mission 2:`, `Part Three:`, `Act 1 Scene 3:` |
| No walkthrough at all | 21 | Nothing to bind — a coverage problem |
| Partial structure | 3 | Manual reading |

Level titles are already indexed and bound at 100 %. Matching a walkthrough line
to one of them is where the difficulty turned out to be. **Three traps found in
the samples before a single line of splitter was written:** a loot line that
reads as a heading; a cross-reference that points the
reader to the next page; and one campaign that labels two *different* levels with
the same ordinal in two files. The rule that was tried is **title match first,
ordinal only as fallback**, and ignore lines shaped like table rows. The chunker
still does not apply it. The search index is unchanged.

**Recounted on 2026-09-24.** The table above is 98 packages two probes could sort
by hand (27 + 47 + 21 + 3). It is not every multi-mission package. The 130 named
above is a third set again: version groups whose indexed chunks already carry more
than one level id. A full pass over the catalog, every package with more than one
playable mission, comes to **157**.

| Layout | Packages |
|---|---:|
| Exactly one walkthrough file | 94 |
| As many walkthrough files as levels | 11 |
| Several files, not one per level | 10 |
| No walkthrough | 42 |

"As many files as levels" is not the hand count of 27, and equal counts are still
not one file per level. A package can hold two copies of the whole campaign, or
one file for a single mission beside a file that covers two. Of the 11, seven
have each walkthrough file on a level. Two of those seven were bound in this pass,
because each file really is one level. Of the other four, one holds two copies of
the whole campaign and stays unbound. The remaining three share two files. The
single-mission file now carries that level's id, after its objectives were read.
The file that covers both missions stays unbound. Those ids sit in the extracted
mission file a rebuild would read. No rebuild was run.

Heading lines, on walkthroughs that still have no level id, under the repaired
title rule (the line equals the title, or a heading prefix, a number, a colon or
dash, and the rest equals the title):

| Group | Packages |
|---|---:|
| A title line for every level | 22 |
| At least one level still needs an ordinal | 48 |
| More than one heading, not every level | 16 |
| A single heading line | 8 |

That is 94 packages where the matcher hit at least one line. The same 157
also split by whether anything is bound yet:

| Where the 157 sit | Packages |
|---|---:|
| A heading the rule accepts | 94 |
| Walkthrough text, no accepted line | 11 |
| No walkthrough | 42 |
| Every walkthrough file already on a level | 10 |

These are not the layout rows. The 94 here are every package with at least one
accepted heading, not the 94 that have exactly one walkthrough file. The 11
with no accepted line are not the 11 with as many files as levels. The 10
already on a level are not the 10 with several files. Only the 42 with no
walkthrough are the same packages in both tables. The old 47 was a hand sort
of heading-shaped files. Most campaigns have a single walkthrough file, which
is why the heading bucket is the large one.

**Eye checks, seeds fixed before each draw.** Packages first, then one cut, so
one campaign cannot fill several slots. A cut is wrong when the section it
starts is not that level. An unclear cut counts as wrong. The bar is about one
wrong cut in ten, so a draw of five allows none.

| Group | Drawn | Bar | Seed 20260924, proposer | Seed 20260925, reader who had not written the title rule |
|---|---:|---|---|---|
| Ordinal still needed | 10 | at most 1 | 0 wrong | not a new test; the ordinal rule had not changed |
| Title covers every level | 5 | none | 2 wrong | 1 wrong |
| Partial | 5 | none | 1 wrong | 0 wrong |
| Single heading line | 5 | none | 2 wrong | 2 wrong |

The first draw used a title rule that matched a title's words inside a sentence.
I did not review the cuts myself. The model that proposed the cuts
then scored all 25. That score does not adopt a group. Three of the ordinal
cuts, the ones whose catalog titles do not name the level, were checked against
each level's objectives by a model that had not written the rules: 3 of 3
correct. That closed the ordinal sample for the rule as it stood.

The title rule was then repaired: the whole line has to equal the title. The
second draw is that rule. Its ordinal ten were a fresh sample of a rule that
had not changed, so they add no new claim. The three groups the repair could
change were read by a model that had not written it, each section against every
level's objectives. Partial cleared its bar. Title and single-line did not.

**Patterns the samples actually produced.**

- A level title inside a sentence: a numbered step, an author's note, a preface.
  The first title rule matched these. The repair rejects them.
- A document title that equals a level title. The first line of the file is the
  level's name, and what follows is credits, or another level's walkthrough.
  The repair, which fixed the sentence, now matches this line. One file has the
  same shape and the section really is that level, so a rule that rejected every
  such line would have dropped a right cut. That rule was not adopted. Changing
  the title rule again would have needed a new seed.
- A pointer to the next page. The filter knows "next page". It did not know
  "following page", so "Mission Two on following page" became a cut a few
  lines early.
- A sentence fragment that starts with Part or Mission and a number ("part 1 of
  the mission"). The ordinal reader accepts the prefix. The title reader no
  longer accepts a title's words inside a sentence. The same shape with the next
  number would attach the end of one level to the next.

One right cut in the partial group is worth keeping in view. The heading says
episode 3, and that level is fifth in play order, because a briefing and an
earlier part sit in the list. The title matched. An ordinal reading of "3"
would have been the wrong level. Title before ordinal stays necessary.

**Stopped here.** The use is the mission in the top 10 while skimming. For a
campaign, the package in that list is enough: the level is recognizable inside
it. Level-found is the column binding would move, and the frozen set has seven
mission-scope rows, so a gain is barely measurable. The bind applies to 157 of
1,430 packages. Without a cut, the named-level path (out of scope here) sends
the whole campaign text, and a current model can find the named level in it.

What is kept: the two one-file-per-level packages, the single-mission file, and
every title cut in the partial group. Of the five cuts drawn from that group,
three were title cuts and two were ordinal. The keep is every title cut in the
group, not only those three lines. The ordinal cuts stay out, because the
ordinal holes are not confined to one group. The title cuts are not applied,
and they are not a reason to rebuild. A later rebuild for some other reason may apply them. It does not
apply ordinal cuts.

Left unbound: ordinal cuts in every group, the title-complete group, and the
single-line group. No further rule, no further draw, no corpus count of the
remaining patterns, no rebuild and no measurement for this remainder. Precision
before coverage, and then a stop, because the rest barely moves the list that
matters.

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

**Integrated, and measurably inert on this eval set: 21/41 and 25/41, unchanged.** That
is the correct outcome, not a failed fix. The corrections merge two version families
(3,300 chunks that were split across four groups), which removes duplicate rows from
result lists — but the two queries still missing are mission-scope, and merging
groups adds chunks without adding a single *bound* one. Binding the level
would be §5.3, and that work stopped there.

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

## 6. After the fusion fix: the query side

### 6.1 Why the query side, and why a model only now

The search was built and measured without any language model first, on purpose: a
baseline of how far retrieval alone gets. Only then a model was added where it
could be measured against that baseline, under a data boundary: the player's query
may go to a hosted model; corpus text does not.

The ten real forum questions pointed at the query side: vocabulary gaps (§5.2) and
facts the player states that the index never reads (a release date). The query side
also needed no rebuild. Level binding is why a mission-scope row can miss when
the package is found (§5.3). The rest of that bind was looked at and left.

Steps that would need corpus text — a model reordering the top 30 using the chunks
that made them rank; short per-level descriptions in player vocabulary generated at
index time — are only possible with a local model and are not built. A German query
path needs German eval queries written by hand, not by a model; that path is not
measured yet.

### 6.2 A stated date the index never read

One of my own searches described a mission and also said it was released within
the last two years. Those date words are noise to BM25, and the search did not find
it. I found it afterwards without the search: I sorted the loader's list by date,
scanned the releases from about two years back and recognised the title. That also
gave the gold for this row.

Through the product pipeline, pool 200: gold absent from the top 200. A post-filter
of that 200 to recent releases: still absent — a post-filter cannot rescue what the
pool never held. A pre-filter — restrict BM25 and vector search to the ~57 indexed
packages released in the window, then run the same pipeline inside it — put gold at
rank **11** (original wording) and **10** (date words removed).

Rank 10–11 is the edge of a top-10 list. What found the mission was seeing titles
inside a date window, not ranking. The catalog has a release date on most rows;
undated packages are kept under a date filter.

### 6.3 A model reading the date: measured, not integrated

**Hypothesis.** One hosted model call could read a release window from the query —
direction, years, and a verbatim quote that the code checks against the query — and
apply it as a pre-filter.

**Measured** on two versions of the prompt, one run each. A query saying "at least
15+ years ago" was first read in the wrong direction (within 15 years). After a
prompt revision, "It's old" became "older than 1 year", "played a few years ago" was read as a release
date ("within 3 years", which would have filtered out that row's gold), and the
date query itself came back with no reading. The model runs at the provider's
default sampling setting, in the eval and in the application alike, so one run
cannot separate a prompt effect from sampling. The
date rules were written after reading the date cues in the forum rows, so these
rows only check rule-following, not generalisation.

**Why rejected.** The reading went wrong in a different way on each run, and one
wrong reading would have filtered out the gold. Filters are therefore set by the
player in explicit fields; what the player sets is what the code applies. The model
reading stays in the eval code, not in the application.

### 6.4 Query keywords as a fourth track

One call to a hosted model (`claude-haiku-4-5`) with a fixed system prompt and the
player's query — nothing from the corpus. It returns up to 12 keywords in
walkthrough and game-world vocabulary. The same reply also carries a written
passage and a date reading; the product uses neither. Only query text reached the model: the eval
questions, including the ten forum questions, which are queries by nature. No
corpus text did.

Locally, before retrieval: any keyword containing a catalog title of two or more
words is removed (the prompt also forbids naming missions). A check for the model
naming the gold title fired on 0 rows. The reranker always scores against the
player's original query, so added words can bring candidates in but do not
re-weight the reranker.

The prompt's keyword instructions were frozen before the first run and never edited
in response to results; its example words appear in none of the 52 queries. Its
date section was revised once (§6.3). The eval model received only the system
prompt and the query, the same call the product makes.

Arms, 52 rows, one index, fresh base in the same run, `hit@10` / `hit@30` / gold in
the candidate pool:

| Arm | What the model output does | hit@10 | hit@30 | in pool |
|---|---|---:|---:|---:|
| Base | — | 25 | 31 | 45 |
| Widen | keywords + passage only widen the candidate pool | 26 | 31 | 49 |
| Tracks | keywords as an extra BM25 track + passage as an extra vector track | 23 | 32 | 49 |
| **Keywords track** | keywords as an extra BM25 track | **29** | **33** | 46 |
| Passage track (HyDE: a model-written sample answer used as the search text) | passage as an extra vector track | 16 | 30 | 48 |

HyDE hurt most on table-only questions: 8 → 1 hit@10 on that 9-row split; the
Tracks arm inherits that. Widen brings gold into the pool more often without
moving ranks. Base 25/52 is consistent with 21/41 in §4 plus 4/10 forum questions
(the forum split reads 4/10 in this run; §5.1 measured 5/10 on an earlier index —
see the caveat in §1 and §5.5).

**Decision rule**, written down before the run: hit@10 at least base + 3, no split
loses more than one hit@10, at most two rows fall out of the top 10. Only the
keywords track cleared it: +4, no split lost (frozen 6→7, table-only 8→8,
table-vs-prose 6→7, forum questions 4→6, diagnostic 1→1, date query 0→0), no row
left the top 10. Four rows entered: ranks 12→8, 11→10, 11→4, 23→6 — three from
just outside the cut; two of the four are real forum questions.

**Stability rule**, written down before re-sampling (the call runs at the
provider's default sampling setting, so another draw gives different keywords;
temperature 0 would reduce that variation but does not guarantee identical
output): integrate only if ≥ +3 hit@10 over base in all three samples and no sample loses a split. Result: three samples, each 25 →
**29** hit@10 (hit@30 33, 34, 34), the same four rows entering, none leaving,
splits identical. The draws were not copies: identical keyword lists on only
17–24 of 52 rows between any two samples, mean keyword overlap 0.81–0.86. The
rule was met, and the track was integrated into Find.

In the application: on by default, switchable off, 20 s timeout and no retries; if
the call fails or is off, Find runs the plain local search and says so; the
keywords used are shown under the result list.

**Latency and cost, measured afterwards.** Same 52 queries, the call exactly as
Find makes it, GPU warm. The reply carries the keywords first, then a passage and a
date reading that Find does not use — 58 % of the output characters in the cached
replies. A stop sequence right after the keyword list keeps the prompt
byte-identical: generation runs left to right, so everything up to the stop is
sampled as before. Cutting the 156 cached replies at that point gave the identical
keyword list every time.

| Call | Median / p95 | Output tokens | USD per 1,000 searches | Added per search, median / p95 |
|---|---:|---:|---:|---:|
| Full reply | 2.16 / 3.43 s | 180 | 1.61 | 2.56 / 4.35 s |
| Stop after keywords | 1.30 / 2.38 s | 78 | 1.10 | 1.59 / 3.17 s |

Prices assumed at USD 1 / 5 per million input / output tokens; input is 709 tokens
either way. "Added per search" is the call plus the extra search time of the fourth
track. Decision rule, written down before the runs: keep the stop only if three new
draws each hold ≥ +3 hit@10 over base with no split lost, and the median call is
faster. Result: 25 → 29 in each draw, no split lost, 0 parse failures. Find now
stops after the keyword list.

The rest of a search is local and dominated by the reranker. At pool 200 that
was about 2.0 s median without keywords and 2.4 s with them, roughly 95 % of it
reranking, because the keyword track enlarges the pool it scores.

**Pool depth, measured afterwards.** One run, pools 50, 100 and 200, the same
52 rows, keywords taken from the existing cache so the run is deterministic.
The run recorded the fused rank only, not candidate-pool membership.
"Absent" means the gold has no rank in the list returned (capped at 200).

| Pool | hit@10 | hit@30 | Absent from the list | Reranker median |
|---|---:|---:|---:|---:|
| 50 | 27 | 33 | 18 | 0.85 s |
| 100 | 30 | 33 | 14 | 1.44 s |
| 200 | 29 | 33 | 11 | 2.57 s |

Decision rule, written down before the run: adopt a smaller pool only if hit@10
is at most 1 below pool 200 and no split loses more than one. Pool 100 clears
it. hit@30 does not move. Three more rows are absent from the list than at 200,
and those rows were already outside the top 30. The reranker median falls from
2.57 s to 1.44 s. Pool 50 loses two hit@10 and is not adopted. Find uses pool 100.

**Not measured:** the combination with player-set filters. **Limits:** 52 rows, gains at the edge of the top 10, one model.

### 6.5 Filters the player sets

Integrated, not yet measured as retrieval. Fields: release year from/to (inclusive, no
hidden margin), author (case-insensitive substring over catalog spellings;
matched spellings shown), game, campaign yes/no, category tags (each either
"any of" or "must have"). Fields combine with AND.

The filter becomes a package allow-list applied **before** BM25 and vector search,
not after the pool — the mechanism from the date probe in §6.2. No re-embed, no
index rebuild: dates and tags live in side tables and are read at query time.
Unknown stays in and is marked (an undated package under a year filter, an empty
author, a package with no tags at all). A package that has tags but not a must-have tag is dropped, because the
list records main aspects.

The category tags come from a curated list: 54 categories in four groups (factions, locations, mission type, special features). The tags
were matched to packages by title, with a manual review of unclear matches; 1,251
of 1,430 packages carry at least one. The list records main aspects only, so a
missing tag is not evidence of absence. Tags restrict the candidate set; they are
never indexed as searchable text.

Nothing about the filters has been measured as retrieval yet; an honest
measurement needs filter values set from the query text alone, before any result
is seen.

### 6.6 Rules for every model step

- The model sees only what the product sends at runtime — in the eval too.
- A model step may add candidates or reorder them; it never removes them from the
  ranked list.
- What the player reads and what the code applies are one object. Player-set
  filters satisfy this by construction; the keywords used are shown.
- Save the full model reply in every eval report.
- Corpus text never goes to a hosted model.

---

## 7. What this evaluation is worth, and what it is not

**A lower bound on a small sample.** 22 + 18 + 1 diagnostic + 10 forum questions
plus one date query.
`hit@10` over 41 mixed rows folds two different gold conditions into one number;
package-found and mission-found belong in **separate columns**, and that change is
scheduled ahead of further ranking work. Without it, a drop cannot be read as a
retrieval failure or a level-resolution failure.

**Overfitting risk, named.** Every parameter touched here — the two BM25 constants,
the RRF constant, pool depth — was tuned against the same 22 queries. That is why
the one ranking change that was adopted is a defect fix with a mechanical explanation rather
than a value that happened to score well, and why the remaining untried weighting
idea is documented as set aside rather than tried. The query-side change had to
clear rules written down before the run and before the re-samples.

**Ranking work that is deliberately *not* next:** any further fusion-key or
weighting variant. Six were measured; five lost and one was adopted. The yield from that
direction is exhausted. Level binding is why a mission-scope row can miss when
the package is found, and §5.3 is where that work stops: further heading rules
would barely move the top-10 list. The query side came first because it needed
no rebuild.

---

## 8. What I would take to the next retrieval project

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
- **A model is better at vocabulary than at reading facts.** Query keywords held a
  stable +4 hit@10 across three draws although the words differed; date reading was
  wrong in different ways per run, so facts moved to fields the player sets.
- **Write the decision rule down before re-sampling.** A sampled model gives different
  output per call; a rule fixed after seeing the draws only confirms them.
- **Stop when the next improvement does not move the use.** Level binding was the
  measured bottleneck, and the samples showed that heading rules in this corpus
  have many special cases. The list a player skims is packages. For a campaign,
  that package is enough to recognise the mission. The rest of the bind was left
  on purpose.
- **Measure what a model adds against a baseline built without one.** Without the
  baseline, +4 has nothing to stand against; with it, the model-written passage
  losing nine rows was visible in the first run.
