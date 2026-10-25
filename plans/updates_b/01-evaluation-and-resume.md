# Evaluation and resumable gloss experiments

## Status and reading guide

This completed milestone delivered reproducible source-only evaluation, accurate
feedback budgets and restartable gloss sessions. Both final development matrices
completed: **36 runs total**. The suite at this milestone passed **63 tests with
no skips**, including all seven optional neural tests. Later milestones expanded
that suite; these are historical verification counts, not a new test run.

Use [development results](#development-results) for the measured outcomes,
[commands and recovery](#commands-and-recovery) to reproduce a run, and
[verification](#verification) for the evidence. The [index](README.md) shows how
this foundation connects to the later product and model work.

## Scope and completed checklist

The objective is transfer from corrected examples to unfamiliar forms whose
meanings have teacher-provided anchors. Evaluation uses word-aligned source-only
glossing. It is not unrestricted sentence translation. Worker B owned experiment
selection, manifests, online events, session checkpoints, CLI wiring and reports;
Worker A supplied the learner and equivalent bulk-fit interface.

- [x] Read the plan, engine handoff and existing code; verify the starting suite.
- [x] Restore checksum-pinned Lezgi/Tsez inputs, keeping Gitksan reserved.
- [x] Freeze source-disjoint selections, provenance, metrics and annotation budgets.
- [x] Implement atomic validated checkpoints and interruption/resume commands.
- [x] Test delays, feedback policies, revisions, invalid inputs and exact restart.
- [x] Complete both bounded 18-condition development matrices.
- [x] Reconstruct saved metrics/models and record commands, results and limits.

## Implementation and provenance

The branch is `feat/gloss-online-evalulation` (original spelling). The shared
baseline was `524163c64150cf06773c0afcc1a28ea4d797774c`; starting HEAD `24a5a386`
already contained A's `83e70a0` bulk-fit API.

| Commit | Meaning |
|---|---|
| `ab72592` | Manifests, metrics, resume/checkpoint infrastructure and initial fixtures |
| `293ef56` | Bounded Windows lock retry and normalized implementation fingerprints |
| `ded160c752e2fe87c70fbed5fa358c1f68bfc304` | Immutable archives and atomic manifests; measured implementation |
| `c5c81e0` | Completed report/handoff checkpoint |

## Files delivered

| File | Purpose |
|---|---|
| `decipher/gloss_evaluate.py` | Frozen selections and provenance, duplicate accounting, real candidate metrics, sentence/word label budgets, separate first/probe/retention/recall scores, AUC ranges, latency, and experiment orchestration |
| `decipher/gloss_checkpoint.py` | Standard-library adapter, immutable complete archives, atomic/checksummed manifests, and restore validation |
| `decipher/gloss_app.py` | Bulk training and the `stream`, `resume`, and `report` options |
| `decipher/online.py` | Request/arrival events, model-call timing, optional source/feedback validation, and full UTF-8 event export; existing neural save/load retained |
| `tests/test_gloss_stream.py` | 18 integration tests, including all 18 model/delay/policy combinations and exact restart comparisons |
| `reports/gloss/online/` | Both complete matrices, compact manifest/verification indexes, and the real restart check |
| `plans/updates_b/` | Ordered checklist and plain-language explanations of each development step |

Worker A's engine and tests, shared README/CURRENT, corpus manifests, existing
baselines, synthetic/neural algorithms, dependencies, and English/Turkish reports
were not edited. Gitksan labels were neither fetched nor evaluated.

## Protocol and checkpoint design

Predictions precede their answers; older due answers arrive before the next
prediction. Support, stream and probes are disjoint by normalized whole source.
Online selection keeps the first annotation for a repeated source and records
later exclusions/conflicts. Shared individual words are allowed.

Original full candidates, ordering, model versions and answer arrival events
remain immutable through restart. Actual top-five scoring uses the saved
alternatives, rather than scoring only the best guess under a top-five label.
Teacher decisions, requested, arrived, revised, superseded and pending labels
are counted separately in sentences and words. Pending answers are not drained.

Each `checkpoint.json` points to a uniquely named complete ZIP in its sibling
`checkpoints/` folder and records its checksum. Keep both together. The archive
contains the existing schema-1 gloss model, session/event history, remaining stream
and configuration, private scoring data and fixed probe predictions. Component
checksums, source/order/configuration checks, and a private history replay are
validated before continuation. Historical scores always use the saved predictions;
validation replay does not replace them.

Reports separately expose support and stream sentence/word budgets, teacher
decisions, requested/arrived/revised/superseded/pending answers, and processed
inputs. Probe curves are sampled at inputs 0/25/50/75/100. Sentence and word AUC
each declare their own arrived-answer range. Training recall and fixed support
retention are not counted as unseen accuracy. Full word candidates are retained.

## Windows checkpoint reliability

The first Lezgi `v1` attempt completed five conditions before Windows/OneDrive
refused a checkpoint ZIP replacement (`WinError 5`). A permission-error-only
retry allows six attempts and at most 1.55 seconds of waiting; unrelated errors
still fail immediately. The `v2` attempt still stopped after six memory conditions.
Both attempts remain incomplete diagnostics, not successful matrix results.

The final design writes a complete ZIP under a fresh name in `checkpoints/`, then
atomically replaces only the small `checkpoint.json` pointer. The ZIP includes
the model, original events/predictions, pending answers, private scoring and run
configuration. Old generations remain recovery points. Failed pointer writes
leave the previous pointer usable; an unreferenced new ZIP is retained harmlessly.

Restore validates the pointer's relative local path/checksum, all archive component
checksums, schemas, event order, policy/delay and model consistency against private
feedback replay. A standalone session ZIP can also be inspected. Replaying history
does not replace original guesses. LF/CRLF-normalized source fingerprints allow
the same code revision across Git line-ending conversion, while real code changes
still reject continuation. Periodic/final saves avoid duplicate generations at
the same input boundary. Only final `v3` runs completed both languages.

## Verification

- Full suite: **63 passing tests** = 45 starting tests + 18 new Worker B tests.
  All seven optional neural tests executed successfully.
- Every fixture model/delay/policy run resumed through two interruptions and
  matched uninterrupted predictions, candidate scores/order, model versions,
  answer order, pending state, fixed probes, and final learned state.
- Both real matrices: **18 complete runs each, 36 total**. All saved logs, probe
  snapshots, exported models and reconstructed metrics agree with their checkpoints.
- A real Lezgi CLI run stopped at 37 stream inputs, with 17 stream labels arrived
  and 20 pending. After resume it reached 100 inputs, with 80 labels arrived,
  20 pending, and model version 90. All 158 original events stayed exactly intact.
  Its history, probes and final model match the uninterrupted delay-20 rule run,
  excluding only timestamps and measured durations.
- The CLI stream/resume/report check also ran with `-S`, which disables third-party
  package loading. Glossing needs only the standard library.
- Final Lezgi/Tsez offline checksum verification and `git diff --check` passed.

Validation evidence:

- [Lezgi manifest index](../../reports/gloss/online/lezgi-prefix-v3/manifest-index.json)
  and [18-run verification](../../reports/gloss/online/lezgi-prefix-v3/verification.json).
- [Tsez manifest index](../../reports/gloss/online/tsez-prefix-v3/manifest-index.json)
  and [18-run verification](../../reports/gloss/online/tsez-prefix-v3/verification.json).
- [Real-corpus restart verification](../../reports/gloss/online/resume-verification.json).
- [Readable results and interpretation](#development-results).

The 18 added integration tests also cover invalid feedback/source inputs without
clock advancement, revisions replacing pending answers, Unicode source duplicates,
true second-candidate top-five scoring, read-only probes, report reconstruction
with prediction/correction disabled, truncated/corrupt archives, escaping archive
paths, incompatible manifests, transient locks and permanent failed saves. Failed
saves preserve the previous complete checkpoint. Synthetic fixture correctness
is kept separate from real-language accuracy claims.

## Development results

Both languages used memory, analogy and rules, delays 0/5/20, and all-label or
correction-only feedback. Each run began with 10 support sentences and the next
100 unique training sources in upstream order. These are bounded prefixes, not
full-training replacements. The learner and earlier English/Turkish baselines
were unchanged.

### Data actually used

| Language | Support sentences / word labels | Stream sentences / word labels | Fixed probe sentences / words |
|---|---:|---:|---:|
| Lezgi | 10 / 106 | 100 / 1,009 | 85 / 935 |
| Tsez | 10 / 90 | 100 / 1,085 | 445 / 4,761 |

The reader found 107 repeated whole training sources in Lezgi and 14 in Tsez,
including one conflicting annotation in each language. Lezgi also has three
repeated probe sources; Tsez has none. The online selection keeps the first
annotation and excludes later copies before taking the prefix. All excluded IDs
and conflict counts are saved. There were no train/dev whole-source overlaps or
misaligned examples in these inputs. Shared individual words across the sets are
allowed: learning how familiar words combine or inflect is part of the task.

### Results with immediate, all-label feedback

“First guesses” means the original stream predictions before their answers.
“Final probes” means unseen development sources after the stream. OOV means a
source word spelling not yet observed in arrived annotations. AUC summarizes the
probe learning curve over 0–100 arrived stream sentence labels; support labels
are additional and are not included on that x-axis.

| Language | Model | First word accuracy | Final probe word accuracy | Final probe OOV | Sentence-label AUC |
|---|---|---:|---:|---:|---:|
| Lezgi | Memory | 42.91% | 57.01% | 0.00% | 49.06% |
| Lezgi | Analogy | 44.20% | 58.50% | 4.19% | 50.95% |
| Lezgi | Rules | 45.09% | 59.57% | 7.19% | 50.43% |
| Tsez | Memory | 21.84% | 30.69% | 0.00% | 21.16% |
| Tsez | Analogy | 27.93% | 35.35% | 7.38% | 25.45% |
| Tsez | Rules | 27.83% | 41.59% | 17.25% | 28.15% |

Rules have the highest final word score in these two prefixes. Analogy has a
slightly higher learning-curve AUC on Lezgi, so the final endpoint alone does not
identify the best method at every annotation budget. These are single ordered
development prefixes, without language-level uncertainty estimates.

Unseen-word performance remains weak. Exact full-sentence probe accuracy is only
1.18% for all three Lezgi models; in Tsez it is 0% for memory/rules and 0.67% for
analogy. Lezgi's rule model scores 44.81% on nonpunctuation words, below its 59.57%
overall word score. These are word-gloss results, not sentence translation scores.

### Delays, feedback exposure, and retention

All-label runs requested 100 stream answers. Delay 0/5/20 left respectively
0/5/20 pending answers and 100/95/80 arrived answers. Pending answers were not
drained. Every stream sentence had at least one mistaken word in these prefixes,
so correction-only runs also requested all 100 answers and happened to share the
same schedules. The code supports different exposure when models make completely
correct sentence guesses; the fixture tests exercise that case.

For the rule model, final probe word accuracy under delays 0/5/20 was respectively
59.57% / 59.57% / 57.97% on Lezgi and 41.59% / 40.64% / 38.16% on Tsez. Fewer
arrived labels accompany the longer delays. Each run's raw points and actual
sentence/word AUC budget range are in its JSON report.

Fixed support-set word retention fell by 5.66 percentage points from its best
snapshot in Lezgi and 4.44 points in Tsez. This is measured separately from unseen
accuracy. The word-level model can change a familiar form's majority gloss when
later examples provide a competing annotation; retaining all corrections does not
guarantee perfect recall of every originally labeled sentence.

### Measured model-call latency

For immediate all-label runs, median / p95 milliseconds were:

| Language | Model | Prediction | Feedback application |
|---|---|---:|---:|
| Lezgi | Memory | 0.146 / 0.309 | 0.294 / 0.763 |
| Lezgi | Analogy | 2.039 / 4.742 | 0.349 / 0.693 |
| Lezgi | Rules | 0.268 / 0.758 | 0.276 / 0.782 |
| Tsez | Memory | 0.152 / 0.315 | 0.783 / 1.683 |
| Tsez | Analogy | 4.371 / 11.454 | 0.786 / 1.529 |
| Tsez | Rules | 0.726 / 1.817 | 0.807 / 1.749 |

These time only model calls over support and stream inputs. Probe scoring,
checkpoint compression, reporting, startup, and other file I/O are excluded, so
they are not end-to-end CLI runtime estimates. Environment details are recorded
in every manifest. The corpus matrices ran sequentially.

## Commands and recovery

Run commands from the repository root; common Windows/test setup is in the [index](README.md#verification-and-environment).

### Exact completed matrix commands

These commands produced the final `v3` runs. For a rerun, change `v3` in both
output paths to a fresh suffix: existing experiments/reports are protected from
accidental overwrite. The default model list is memory, analogy and rules.

```powershell
.venv/Scripts/python.exe -X utf8 -m decipher.app gloss stream --train data/glossing/lezgi/lez-train-track1-uncovered --dev data/glossing/lezgi/lez-dev-track1-uncovered --run-dir runs/gloss-online-worker-b-20261024-lezgi-prefix-v3 --report-dir reports/gloss/online/lezgi-prefix-v3 --support 10 --stream 100 --delays 0 5 20 --policies all corrections --probe-every 25 --checkpoint-every 25

.venv/Scripts/python.exe -X utf8 -m decipher.app gloss stream --train data/glossing/tsez/ddo-train-track1-uncovered --dev data/glossing/tsez/ddo-dev-track1-uncovered --run-dir runs/gloss-online-worker-b-20261024-tsez-prefix-v3 --report-dir reports/gloss/online/tsez-prefix-v3 --support 10 --stream 100 --delays 0 5 20 --policies all corrections --probe-every 25 --checkpoint-every 25
```

### Try interruption and resume

The following uses a new example directory. `--max-inputs` is an additional input
budget for each invocation; stopping does not drain queued answers.

```powershell
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss stream --train data/glossing/lezgi/lez-train-track1-uncovered --dev data/glossing/lezgi/lez-dev-track1-uncovered --run-dir runs/gloss-resume-demo --support 10 --stream 100 --modes rules --delays 20 --policies all --probe-every 25 --checkpoint-every 25 --max-inputs 37

.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss resume --checkpoint runs/gloss-resume-demo/rules-delay-20-all/checkpoint.json

.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss report --checkpoint runs/gloss-resume-demo/rules-delay-20-all/checkpoint.json --output runs/gloss-resume-demo/reconstructed.json
```

The actual verified restart used the same options under
`runs/gloss-online-worker-b-20261024-resume-smoke`. Its final reports are retained
locally. The completed matrix artifacts remain in their separate `v3` directories.

## Limits and operational notes

- The experiment is a deterministic upstream-order prefix, not the full training
  corpus, a replicated language study, or unrestricted translation. Unseen-word
  accuracy and exact sentence accuracy remain substantial limitations.
- Correction-only exposure can depend on the model. In these real prefixes every
  sentence required a correction, so the two policies had identical exposure.
- Continuation intentionally rejects changed core code or changed original input
  files. Original dataset paths recorded in the manifest must remain accessible;
  cross-machine path remapping is not implemented.
- Resume operates on one run. Its `metrics.json` and a fresh `gloss report` reflect
  the resumed state. A previously written matrix-level summary remains a snapshot
  of its original invocation; it is not automatically rebuilt by single-run resume.
- Checkpoint validation replays history and can take longer than an individual
  model call. Reported model latency excludes this work and file I/O. Old immutable
  archive generations are kept locally, including harmless unreferenced generations
  after a failed manifest replacement.
- Lezgi `v1` and `v2` are incomplete file-lock diagnostics. Only `v3` is the final
  matrix. The final format completed both languages without replacement failures.
- This milestone was subsequently integrated. There is no remaining Worker B
  implementation blocker for its evaluation/resume scope.
