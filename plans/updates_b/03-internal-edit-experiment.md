# Internal-edit experiment: protocol, results and decision

## Status and reading guide

**Feature 4 is complete. The decision is reject default adoption.** Lezgi accuracy
did not improve; Tsez grounded unseen-word accuracy fell by 0.619 percentage points.
The application retains its existing learner. Experimental code/reports were
published and merged into main for reproducibility, which does not enable the
overlay in normal application commands.

Use [results and decision](#results-and-decision) for outcomes,
[frozen protocol](#frozen-protocol) for the preregistered comparison,
[commands and recovery](#commands-and-recovery) for reproduction, and
[main integration](#main-integration) for branch/publication history.

## Scope and completed checklist

The user initially deferred Feature 4, then explicitly approved it after A/B
product integration. The separate `experiment/gloss-internal-edits` branch started
from `bc0a5df`. A's published main `eaab130` was already included. The starting
suite passed 129 tests, including the optional neural tests.

- [x] Fetch/check A's main changes and create the separate experiment branch.
- [x] Read the existing proposal, current learner/reader and evaluation interfaces.
- [x] Freeze the model, data, budget and decision criteria below before corpus runs.
- [x] Verify the full starting regression suite and define the targeted fixtures.
- [x] Implement an explicitly prepared, version-checked inference overlay in a new
      module, preserving the base model and its schema-1 checkpoints.
- [x] Test replacement/insertion/deletion, competing tags, Unicode, grounding,
      duplicates/revisions, stale-overlay rejection, read-only prediction,
      checkpoint reconstruction, and exact baseline equivalence without internal evidence.
- [x] Build a Windows-compatible isolated comparison runner with a hard 120-second
      child-process deadline and memory measurement with explicit units.
- [x] Run full pinned Lezgi and Tsez training/development comparisons with the same
      examples and 512 total rule slots; record candidates and private scoring separately.
- [x] Reconstruct scores and changed-word/grounded-error diagnostics from artifacts;
      report runtime, memory, regressions and incomplete runs honestly.
- [x] Apply the frozen adopt/reject criteria, run combined regressions, and document
      results, commands and limitations in this folder. Do not promote the default model.

## Frozen protocol

Settings and gates were committed at `9afb8b8` before corpus outcomes. The original
protocol remains available in Git at
`9afb8b8:plans/updates_b/feature-4/01-plan-and-frozen-protocol.md`. The comparison
runner deliberately reads that historical commit/path, not the current notes.
Its constant and recorded hashes are unchanged by this consolidation; no stub
copy of the old file is needed. The following retains the frozen conditions.

### Frozen model condition

The base is the current `GlossLearner(mode='rules', max_rules=512,
paradigm_window=32, contextual=False)`. Existing prefix/suffix hypotheses and
new internal hypotheses compete for **512 total slots**, not 512 each. All
authoritative training annotations and their original order are identical.

Within each existing bounded lexical paradigm, infer one changed middle block
with at least one unchanged Unicode codepoint on each side. Reuse the existing
prefix/suffix gloss transformations and lexical-key grounding rule. Include both
directions. Rank the union by descending distinct-lexeme support, ascending edit
complexity, then the existing six-field rule tuple (internal uses the source-side
name `internal`). No minimum-support tuning or extra unreported rule pool.

When several edit positions recover the same anchor under one rule, count that
anchor once. Distinct hypotheses retain separate baseline-style votes. Known-form
candidates remain exactly the base model's candidates. The overlay is explicitly
prepared after teaching and rejects an outdated base version/configuration;
prediction never prepares a deferred cache. Checkpoints save only the unchanged
base model; the overlay is deterministically reconstructed after load.

Internal spelling decompositions are surface hypotheses, not gold morpheme
segmentations. Repeated characters may permit more than one edit location. Inputs
are NFC-normalized by the existing reader; spans are codepoints, not phonemes.
Scores remain uncalibrated. No contextual model, neural training or new root meanings.

### Frozen data and reporting

Use the checksum-pinned full Lezgi and Tsez training/development files. Keep the
offline baseline's duplicate-record weighting; disclose within-split duplicates
and source conflicts rather than silently changing historical membership. Exclude
exact training-source overlaps from development. Do not inspect Gitksan or any
corpus test labels. Settings are fixed before outcomes; no development-result
tuning of edit bounds, ranking or budgets.

Run each language in a fresh subprocess with a hard **120-second** wall deadline.
Report parsing, base fitting, overlay preparation, prediction and verification
times separately. Peak RSS/working-set bytes describe the process containing
both base and overlay and its artifacts, not isolated per-component allocations.

Report word/OOV/grounded-OOV/nonpunctuation/sentence accuracy, coverage and top-five
results; raw correct/incorrect transitions; known-form output equality; hypothesis
counts/budget displacement; and grounded failures classified by absent root
recovery, missing grammatical output or ranking. Gold is private scoring data only.
Save full artifacts under ignored runs/ and compact reports in a new experiment
directory; historical reports and production evaluation/session code stay fixed.

### Predeclared decision criteria

Recommend an adoption review only if both full runs finish and all correctness
tests pass, and all of the following hold:

- Overall word accuracy drops by no more than **0.5 percentage points** on either
  language; known-form candidate equality is exact.
- Mean grounded-OOV accuracy gain is at least **1 percentage point**, with neither
  language losing more than **0.5 points** on that metric.
- Each overlay prediction pass is at most **2× baseline time or adds at most
  0.25 seconds** (the absolute allowance avoids unstable tiny-denominator ratios).
- Measured process peak is at most **512 MiB** and the 120-second deadline holds.

Otherwise reject default adoption and retain the frozen baseline. These are
declared engineering/research screening criteria, not statistical significance
tests. Passing would still require a separate approval before any default-model
change. A negative result is a complete, useful Feature 4 outcome.

### Starting verification and concrete fixtures

All 129 starting tests pass, including the optional neural tests. Fixtures are
defined before the overlay: `man/men` transferring to grounded `tal/tel`; reverse
transfer from an observed plural; middle insertion/deletion; repeated deletion
positions recovering one anchor; competing plural/dual tags; accented Unicode;
unanchored roots; pure edge morphology with full candidate equivalence; and
duplicate/revised annotations followed by explicit preparation and save/load.


## Implementation and verification

| File | Purpose |
|---|---|
| `decipher/gloss_internal.py` | Explicitly prepared internal-edit overlay, stale-state guard and checkpoint reconstruction |
| `tests/test_gloss_internal.py` | Eleven linguistic/state/grounding/equivalence fixtures |
| `tests/test_internal_experiment.py` | Six timeout, memory, scoring and decision-gate fixtures |
| `tools/evaluate_internal_edits.py` | Frozen full-data comparison, persisted regression record and adoption decision |
| `tools/verify_internal_edits.py` | Independent saved-artifact scoring, membership and full-candidate replay |
| `reports/gloss/internal-edits/` | Compact versioned results, audits, regression record and decision |
| `runs/internal-edits-v1/{lezgi,tsez}/` | Ignored full manifests, candidate logs, private scoring, rule tables and base checkpoints |

The original learner, product CLI/UI, persistence and online evaluator files have
no experimental changes. Existing schema-1 base checkpoints reconstruct the
overlay; no experimental schema migration is needed. Candidate scores remain
uncalibrated, so the existing product confidence calibration is not transferable
to this experiment.

### What was built

`decipher/gloss_internal.py` adds `InternalGlossOverlay`, an experimental layer
around a fitted, authoritative `GlossLearner`. Nothing routes the product into it
automatically. Worker A's workspace/UI and the existing teaching/review CLI keep
their established behavior.

The layer learns spelling changes inside a word, such as `man` to `men`, from
forms with teacher-provided meanings. It can then transfer that pattern from an
observed `tal` meaning `tree` to unfamiliar `tel`, suggesting `tree-PL`. It cannot
invent a root meaning: reversing a rule must recover an observed source word,
and changing its gloss must preserve that word's learned lexical key.

Middle insertion and deletion work the same way. Both directions are learned.
An internal change must leave at least one source character on each side. The
existing prefix/suffix and new internal hypotheses share one 512-rule budget,
ranked by the same support, edit cost and deterministic tie order. Different
positions that recover the same anchor under one rule vote only once.

### State and restart behavior

- Construction and `prepare()` explicitly prepare the overlay. Prediction and
  recovery inspection only read prepared indexes; they do not train or rank rules.
- A changed base version or configuration makes the old overlay reject prediction.
  The caller must finish teaching and explicitly prepare it again.
- Duplicate feedback does nothing. Revised feedback removes the old annotation's
  evidence through the authoritative learner, then preparation rebuilds the layer.
- Known words return the base model's complete candidate records unchanged.
- Saving uses the existing schema-1 base checkpoint. `from_checkpoint()` loads
  that base and reconstructs the overlay, avoiding a second authoritative format.
- This experiment only supports rules mode with contextual prediction disabled.
  Its scores are ranking weights, not calibrated probabilities.

### Tests before corpus outcomes

The 129 existing tests passed before changes. The 17 new tests pass:

- Eleven model tests cover replacement, insertion, deletion, competing tags,
  Unicode, unsupported roots, known-word equality, duplicate/revised feedback,
  stale state, read-only prediction/RNG, schema-1 reconstruction, and full
  baseline-candidate equality when there is no internal evidence.
- An exhaustive reference checks indexed anchor recovery and the shared budget.
- Six runner tests check actual subprocess timeout and exit status, memory units,
  private-score reconstruction, every adoption gate, and tiny timing allowances.

Focused command: `.venv/Scripts/python.exe -X utf8 -S -m unittest
tests.test_gloss_internal tests.test_internal_experiment -v`.

### How the corpus runner works

`tools/evaluate_internal_edits.py compare` starts a fresh, standard-library-only
child for one language. The parent imposes a 120-second execution timeout and
records the last completed phase if it expires. Process creation and cleanup can
add platform overhead to the parent's elapsed time. This follows Python's
[subprocess timeout behavior](https://docs.python.org/3/library/subprocess.html).

Before fitting, the child checks the frozen base learner source and all pinned
files for that language, including its license. Both systems receive identical
full training records and development words. Exact whole-source train/dev
overlaps are excluded; within-split duplicates and conflicting annotations retain
the offline baseline's weighting and are counted explicitly.

Each phase is timed separately. On Windows, memory uses the documented
[PeakWorkingSetSize byte count](https://learn.microsoft.com/en-us/windows/win32/api/psapi/ns-psapi-process_memory_counters).
It measures the lifetime process peak, including base, overlay, prediction arrays
and checkpoint reconstruction, rather than pretending to isolate each component.

Raw local artifacts include membership, selected rules, source-only predictions,
private gold and changed-word diagnostics, scoring context, and a base checkpoint.
Compact versioned reports include hashes, exact metric counts, runtime/memory,
known-word equality, gains/losses and grounded-error categories. Every full
candidate record and selected rule is checked again after checkpoint loading.

`tests` writes a regression record tied to source fingerprints. `decide` applies
the criteria committed in the protocol before any corpus outcomes. Even a
passing result only recommends a separate adoption review; it changes no default.

### Completed verification

At implementation commit `4eb8632`, all **146 combined tests passed**, with zero
failures, errors or skips. This includes all 129 previous tests and the 17 new
tests. Both full corpus runs use that same fingerprinted implementation. The
record is [regressions.json](../../reports/gloss/internal-edits/regressions.json).

After collecting results, `tools/verify_internal_edits.py` independently reads
only the saved artifacts. It verifies their checksums and membership/order,
reconstructs every metric, changed-word record, sample and grounded-error count,
then rebuilds the checkpoint and checks all full candidates and selected rules.
Both language audits passed. This audit helper does not change the model or
comparison implementation tested at `4eb8632`.

Five additional negative sanity checks on a temporary copy also passed: corrupt
candidates, incorrect reported scores, incomplete status, a different source
fingerprint and an existing output path are all rejected. The original artifacts
were preserved. Evidence is in
[audit-negative-checks.json](../../reports/gloss/internal-edits/audit-negative-checks.json).

The five new Python files also passed Python 3.10 syntax parsing; execution was
on installed Python 3.13. Report line endings are fixed to LF to preserve byte
hashes across Windows Git checkout.

## Results and decision

### Decision

**Reject default adoption of this internal-edit overlay.** The experiment is
complete, including its negative result. Keep the existing learner for the
application. No defaults or production checkpoint formats were changed.

The settings and decision gates were committed at `9afb8b8`, before either corpus
run. The tested and evaluated implementation is `4eb8632`. No settings, data
membership or acceptance thresholds were adjusted after seeing the outcomes.

The [machine-readable decision](../../reports/gloss/internal-edits/decision.json)
fails two preregistered gates:

- Mean grounded unseen-word gain is **-0.309 percentage points**, below the
  required +1 point.
- Tsez grounded unseen-word accuracy drops **0.619 points**, beyond the allowed
  0.5-point loss.

The overall accuracy, technical correctness, time and memory gates pass. Passing
those gates alone does not justify adopting a model that misses its transfer goal.

### Data used

Both systems use source words only and identical full pinned annotations. There
were no alignment exclusions or exact train/development source overlaps in these
runs. Within-split repeated sentences retain the historical offline weighting.

| Language | Training sentences / word labels | Development sentences / words | Unseen words | Grounded unseen words |
|---|---:|---:|---:|---:|
| Lezgi | 634 / 6,250 | 88 / 992 | 222 | 119 |
| Tsez | 3,547 / 37,396 | 445 / 4,761 | 928 | 808 |

Training has 107 repeated whole sources in Lezgi and 14 in Tsez, with one
conflicting annotation in each. Development has three repeated sources in Lezgi
and none in Tsez. Both systems retain them equally. The full-record development
denominators therefore differ from the earlier unique-source online experiments.
All selected files and licenses match `data/glossing/SOURCES.json`. No reserved
Gitksan annotations or corpus test labels were inspected or evaluated.

"Unseen" means the exact source word was absent from training. "Grounded unseen"
additionally means its private gold lexical key already occurs in the trained
lexical paradigms. That label is used only for scoring, never as model input.

### Accuracy and coverage

Each cell shows baseline -> experiment. These are percentages, not relative gains.

| Metric | Lezgi | Tsez |
|---|---:|---:|
| Word exact | 72.177 -> 72.177 | 75.089 -> 75.005 |
| Unseen word exact | 15.766 -> 15.766 | 35.237 -> 34.806 |
| Grounded unseen exact | 27.731 -> 27.731 | 38.614 -> 37.995 |
| Nonpunctuation word exact | 61.268 -> 61.268 | 75.037 -> 74.953 |
| Top five exact | 77.016 -> 77.016 | 83.953 -> 83.995 |
| Coverage | 85.181 -> 85.282 | 90.401 -> 90.338 |
| Sentence exact | 11.364 -> 11.364 | 11.011 -> 11.011 |

Coverage means the model returned a gloss, even if it was wrong. Top five means
the correct gloss was among the returned candidates, even if it ranked below the
first choice. Neither should be described as first-choice accuracy.

Lezgi remains **716/992** correct, including **33/119** grounded unseen words.
Thirty word candidate records change, but only three first-choice glosses change;
all three were wrong before and after. Two words gain a prediction and one loses
it. The extra covered word does not improve accuracy.

Tsez changes from **3,575/4,761** to **3,571/4,761** correct. Three words improve
and seven regress, for a net loss of four. Grounded unseen accuracy changes from
**312/808** to **307/808**: two gains and seven losses in that subset. The third
overall improvement is a grammatical label outside that lexical subset. There
are 115 changed candidate records, 14 changed first choices, three newly covered
words and six words that lose coverage. Known-form full candidates are identical
for all **770 Lezgi** and **3,833 Tsez** known-word occurrences.

### What the changes tell us

The same 512 slots now contain 96 internal rules in Lezgi and 40 in Tsez,
displacing exactly that many selected baseline rules. Before selection, Lezgi has
3,048 baseline and 656 internal hypotheses; Tsez has 77,448 and 12,474.

Five of Tsez's seven regressions have a baseline winning-evidence rule that is no
longer selected. Four previously correct words now have no candidate. This is
consistent with competition for the fixed budget. It is not a controlled
attribution of every loss: additional votes can also change ranking, and the
recorded evidence rule is only one contributor to an aggregated score.

Representative private development examples:

- `ik’ixanu,` improves from unknown to `go-PRS-NEG`, using observed `ik’ix,`.
- `boysƛin”,` improves from unknown to `III-do-PST.WIT-QUOT`, using `boys,`.
- `ƛexunanu.` regresses from `remain-PFV.CVB-NEG` to `remain-PST.UNW-NEG`.
- `č’edoł` regresses from `earth-CONT.ESS` to unknown after its supporting edge
  hypothesis loses its selected slot.

All three improvements contain punctuation. An insertion immediately before a
comma or period qualifies as an internal character edit because the punctuation
is an unchanged final character. These gains are therefore not evidence that
the system learned a productive linguistic stem alternation. The frozen rule
definition intentionally stayed unchanged after discovering this limitation.

### Grounded unseen-word failures

The following categories describe the returned top-five candidates. "Lexical key
absent" means no returned candidate has the expected lexical key; it does not
prove the root is absent from all possible anchors or lower-ranked hypotheses.
"Grammar missing" means that key is represented, but the exact target gloss is
not in the returned five. "Ranking" means the target is returned below first.

| Failure category | Lezgi baseline -> experiment | Tsez baseline -> experiment |
|---|---:|---:|
| Expected lexical key absent from returned candidates | 59 -> 60 | 409 -> 410 |
| Exact grammatical output missing from returned candidates | 17 -> 16 | 41 -> 39 |
| Correct gloss returned below first | 10 -> 10 | 46 -> 52 |

Among experimental grounded failures, a selected internal inverse edit reaches
an observed anchor with the expected lexical key for only **6/86 Lezgi** and
**19/501 Tsez** cases. The remaining 80 and 482 cases lack such an internal
recovery. Merely knowing the lexical meaning elsewhere in training is not enough
for this bounded surface-change method to recover the unfamiliar form.

Known-form errors also remain: 89 in Lezgi and 585 in Tsez. This experiment keeps
the base known-form behavior, so it cannot resolve those ambiguities. Worker A's
separate contextual feature is outside this frozen comparison.

### Time, memory and reproducibility

| Measurement | Lezgi | Tsez |
|---|---:|---:|
| Full isolated child run, including reconstruction | 0.964 s | 3.738 s |
| Baseline fit | 0.047 s | 0.872 s |
| Overlay preparation | 0.008 s | 0.162 s |
| Baseline prediction | 0.0109 s | 0.0465 s |
| Overlay prediction | 0.0156 s | 0.1327 s |
| Checkpoint load plus overlay preparation | 0.073 s | 1.060 s |
| Process lifetime peak working set | 49.96 MiB | 177.12 MiB |

Both runs exit normally within their 120-second deadline and 512-MiB memory
allowance. Tsez prediction takes 2.86 times baseline, but adds only 0.086 seconds,
so it passes the predeclared alternative allowance of at most 0.25 added seconds.
These are single-process timing measurements on this Windows machine, not a
stable cross-hardware speed claim. Memory includes base plus overlay, prediction
arrays and restart verification; it is not an isolated overlay memory estimate.

All **146 tests** pass with zero skips. Both runs preserve the base model identity,
known-word candidate records, selected rules and every candidate after save/load.
Independent artifact replay also reconstructs every score, transition, sample and
grounded failure, and checks training/development membership and checksums.

Compact evidence is in [the experiment reports](../../reports/gloss/internal-edits).
Full local artifacts remain under `runs/internal-edits-v1/`, which Git ignores.
Historical reports were preserved. Reproduction commands and file roles are in
[the handoff](#commands-and-recovery).

This is a bounded development experiment, not a significance test, untouched-test
result, general translation system or demonstration that internal morphology is
unhelpful in general. Its supported conclusion is that **this frozen overlay
should not replace the current default**.


## Commands and recovery

From the repository root in PowerShell, using the existing `.venv`. The following
uses **v2 paths** so the delivered v1 evidence remains intact. Choose another
unused suffix if those paths already exist. Corpus files must already be present
and match the pinned checksums; the experiment performs no downloads.

```powershell
$env:PYTHONUTF8='1'
.venv/Scripts/python.exe tools/evaluate_internal_edits.py tests --output reports/gloss/internal-edits/regressions-v2.json
.venv/Scripts/python.exe -X utf8 -S tools/evaluate_internal_edits.py compare --language lezgi --run-dir runs/internal-edits-v2/lezgi --report-dir reports/gloss/internal-edits/lezgi-v2 --timeout 120
.venv/Scripts/python.exe -X utf8 -S tools/evaluate_internal_edits.py compare --language tsez --run-dir runs/internal-edits-v2/tsez --report-dir reports/gloss/internal-edits/tsez-v2 --timeout 120
.venv/Scripts/python.exe -X utf8 -S tools/verify_internal_edits.py --run-dir runs/internal-edits-v2/lezgi --output reports/gloss/internal-edits/lezgi-v2/verification.json
.venv/Scripts/python.exe -X utf8 -S tools/verify_internal_edits.py --run-dir runs/internal-edits-v2/tsez --output reports/gloss/internal-edits/tsez-v2/verification.json
.venv/Scripts/python.exe -X utf8 -S tools/evaluate_internal_edits.py decide --reports reports/gloss/internal-edits/lezgi-v2/results.json reports/gloss/internal-edits/tsez-v2/results.json --tests reports/gloss/internal-edits/regressions-v2.json --output reports/gloss/internal-edits/decision-v2.json
```

Run each command only after the preceding command succeeds. The regression
command intentionally enables installed packages so all seven optional PyTorch
tests run; `-S` on the corpus commands shows that the gloss experiment itself
uses only the standard library. Without PyTorch, neural tests skip and the strict
adoption screen records an incomplete regression requirement.

Only `compare` has the frozen 120-second child deadline; tests and independent
artifact audits are separate checks. A timed-out/failed comparison remains marked
incomplete, preserves its last phase and cannot qualify for adoption. The tools
refuse to replace nonempty experiment directories or existing audit/decision/test
outputs. A rejected adoption decision still exits successfully because applying
the decision is a completed operation.

To re-audit delivered v1 artifacts, use the verifier with the v1 run directory and
a fresh output path, for example `reports/gloss/internal-edits/lezgi-v1/recheck.json`.
No original corpus files or new model corrections are needed for that audit.

## Main integration

| Checkpoint | Meaning |
|---|---|
| `bc0a5df` | Integrated teaching/review base, including A's `eaab130` |
| `9afb8b8` | Frozen pre-outcome protocol |
| `4eb8632` | Tested and evaluated experimental implementation |
| `32bb612` | Results, independent artifact auditor and handoff |
| `b46e649` | Publication note; both completed branches published and main synchronized |

The user explicitly authorized publishing `feat/worker-b-teaching-review` and
`experiment/gloss-internal-edits` and merging both into main. After pulling
`origin/main`, both histories were included through fast-forward integration.
An atomic push published both branches and main to
`https://github.com/CoolestToDoIt/Unknown-Language-Translation.git`. Remote checks
confirmed product branch `bc0a5df`, experiment/main `b46e649`, a clean tree and
zero local/remote-main divergence at completion. No branches were deleted and no
history was force-pushed. Main has since received further project work.

An automatic approval review initially blocked publication because the destination
was unverified. The user answered the exact-destination question with the requested
pull/merge/push sequence, after which the push succeeded. No workaround was used.

The regression reflects fixed-budget competition and changed votes: five of seven
Tsez losses have displaced baseline winning-evidence rules. Some apparent internal
changes occur before punctuation. The user requested no further investigation;
no tuning or extra model experiment followed. Publication preserved the negative
result, not a default-model promotion.
