# Teaching, review and workspace integration

## Status and reading guide

**Features 2 and 6 are complete and integrated with Worker A's real workspace.**
The teaching CLI uses `.dproj` projects by default. Review, search, immutable first
predictions, revision history, correction conflicts and separately validated
uncertainty estimates are implemented. The product milestone passed **129 tests,
none skipped**; this is the recorded milestone result, not a new test run.

Use [commands and recovery](#commands-and-recovery) to start a project,
[development results](#development-results) for the measured calibration/review
outcomes, and [project contract](#project-contract-and-persistence) for API details.

## Scope and completed checklist

Work began from main `a9281397919d9589315c8e43277ad725d81c0755` on
`feat/worker-b-teaching-review`. A owns persistence, the web UI, contextual modeling
and packaging. B first used an explicit fixture adapter, then integrated A's
published `eaab130` workspace. Feature 4 was initially deferred by the user,
then separately approved and completed in the [experiment milestone](03-internal-edit-experiment.md).

- [x] Read requirements, create the feature branch and verify the 83-test baseline.
- [x] Define the project/store contract and establish user-flow/safety fixtures.
- [x] Implement submit, predict/evidence, confirm, correct, save/reopen and export.
- [x] Implement deterministic review, Unicode search, history and conflict checks.
- [x] Fit/validate uncertainty separately from learner training; reject stale estimates.
- [x] Bind the CLI to A's `.dproj` workspace and protect CLI/browser writes.
- [x] Verify the real teaching/backup/restore flow and all existing regressions.
- [x] Complete bounded Lezgi/Tsez calibration/review checks and reconstruct artifacts.
- [x] Record budgets, results, reproduction commands and scope limits.

## Implementation and source checkpoints

| File | Responsibility |
|---|---|
| `decipher/teaching.py` | Typed project/store contract, immutable predictions and correction semantics |
| `decipher/teaching_app.py` | One-shot commands and interactive teaching shell |
| `decipher/workspace.py` | Adapter to A's `LanguageProject` and real `.dproj` files |
| `decipher/review.py` | Read-only priority, source conflicts, Unicode search and history lookup |
| `decipher/review_calibration.py` | Frozen-model calibration, validation and model identity checks |
| `decipher/project.py`, `decipher/gloss_web.py` | Shared atomic revision/write guards and browser conflict responses |
| `tools/evaluate_review_calibration.py` | Pinned bounded review/calibration measurements |
| `tests/test_teaching.py`, `test_review.py`, `test_review_calibration.py`, `test_teaching_workspace.py` | Teaching, review, calibration and production integration fixtures |

| Commit | Meaning |
|---|---|
| `8261882` | Teaching contract/fixture checkpoint on `feat/real-text-teaching-cli` |
| `b152ac7` | Review/search/conflicts, calibration and measurement driver |
| `80feecb` | Merge A's `eaab130`, retaining both `gloss web` and `gloss teach` |
| `e1fad21` | Real workspace integration and measured corpus implementation |
| `dd54772` | Literal apostrophes, revision reasons and native paths in shell input |
| `bc0a5df` | Completed product report/handoff; also the `feat/efficient-review` checkpoint |

The complete product branch is `feat/worker-b-teaching-review`; the individual
teaching branch is an earlier dependency checkpoint. Both completed product and
experiment histories were subsequently published and merged into main at
`b46e649`; see [integration history](03-internal-edit-experiment.md#main-integration).

## Project contract and persistence

`ProjectStore.create(path, name, language, mode)` and `ProjectStore.open(path)`
return a project adapter. The adapter provides:

| Method/property | Meaning |
|---|---|
| `project_id`, `name`, `language`, `model_version` | Stable identity and current model revision |
| `submit(TextExample)` / `examples()` | Register/list public source sentences, with no gold attached |
| `predict(TextExample)` | Read-only word predictions with full candidates/evidence |
| `remember_prediction(record)` / `prediction(id)` | Store/retrieve immutable before-answer predictions |
| `annotation(id)` / `history(id)` | Latest verified annotation and all actual annotation revisions |
| `correct(...)` | Atomically check expected annotation revision and operation ID, then apply the correction |
| `model_identity()` / `training_sources()` | Bind calibration to the exact model and exclude its training sources |
| `save()` / `export(path)` | A-owned persistence boundary |

Correction requests include a prediction ID, one label per word, the annotation
revision reviewed, an idempotency operation ID, a reason and an explicit replacement
choice. Repeating an identical operation does not teach twice; reusing its ID for
a different request is an error. Replacing a different existing annotation requires
an explicit replacement choice and reason. Confirmation also rejects a prediction
from an older model version.

The production adapter stores immutable prediction records, idempotency receipts
and human revision reasons in an optional `teaching` field inside A's schema-1
`.dproj` file. Existing UI/imported projects need no migration. There is no second
production project format or sidecar; the same metadata travels through the
existing project export/restore ZIP. The fixture backend is opt-in test/demo
infrastructure, selected with `--backend fixture`.

Inspection showed that A's first API saved corrections immediately and did not
detect stale clients. B needed to save the correction and its audit information
together and prevent a CLI/browser client from overwriting newer work.

- `LanguageProject.submit` registers public manual source sentences.
- `correct(..., expected_revision=..., persist=False)` checks the reviewed
  annotation revision and allows one final save after the audit metadata is added.
  Existing callers retain their normal save behavior.
- `save()` uses an exclusive writer lock and checks the file identity from open
  before replacing it. Failed writes clean up temporary files and retain the last
  complete project. A stale writer must reopen instead of silently overwriting.
- The web form sends its displayed revision. Stale annotation/file writes return
  HTTP 409 so the user can refresh. The page layout and project format are retained.
- Calibration identity includes A's contextual setting as well as model evidence
  and settings, so artifacts cannot be reused across different prediction behavior.

These are targeted storage/API safety additions required by the teaching/review
flow. A's contextual model extension is preserved.

## Teaching and review workflow

Public prediction inputs contain only an ID and NFC-normalized source words.
First predictions are saved before answers and never rewritten. Teaching requires
one nonempty gloss per source word. Unknown/incomplete predictions cannot be
confirmed; an existing source ID cannot silently acquire different text.

The reviewed annotation revision, operation ID and explicit replacement reason
protect corrections. Duplicate operations do not teach twice; reused IDs with
different requests are errors. Superseded annotations are removed from evidence.
Confirmation rejects a prediction from an older model. Two clients cannot silently
overwrite each other's revisions or newer saved files. A stale client must reload.
Failed writes retain the last complete project; abandoned writer locks are
reported rather than stolen automatically.

Review shows conflicting annotations for the same whole source first, followed
by unsupported/unseen and ambiguous forms. Reviewed sentences are normally skipped
unless a source conflict remains. Ties are stable. Search handles Unicode/case
and can include past annotations/reasons on request. Source aliases are diagnosed,
not automatically merged: some ambiguity needs human review or contextual modeling.

## Calibrated uncertainty

The model's raw scores are not probabilities. A separate calibration artifact
estimates how often the top word gloss is correct for observed groups of candidate
kind, known/unseen form and relative candidate margin. Small groups back off to a
broader supported group; unsupported groups get no estimate. Unknown output has
zero exact-gloss correctness against a nonempty target.

Calibration uses a frozen model, a complete labeled fitting set and a disjoint
validation set. Neither set can contain a whole source used to train the learner.
Correction-only logs are rejected as a calibration policy because selected errors
do not represent all predictions. Calibration and validation labels are additional
supervision and are counted separately; they never update the gloss learner.

The default validation gate requires at least 100 fitting words and 100 scored
validation words, groups of at least 20 words, at least 90% estimate coverage,
ECE at most 0.10, and Brier score no more than 0.005 worse than a constant estimate
from fitting-set accuracy. These are declared development criteria, not a universal
guarantee. Rejected/insufficient artifacts supply no confidence estimates.

ECE measures the gap between estimated and observed correctness in reliability
bins; Brier score measures squared probability error. The calibration background
is [Guo et al., 2017](https://proceedings.mlr.press/v70/guo17a.html). This implementation
uses empirical groups, not that paper's temperature scaling, and reports both
metrics instead of relying on ECE alone.

An artifact is bound to the exact model evidence, ordering, settings, revision and
engine identity. Teaching changes invalidate it immediately. Review then falls
back to explicitly uncalibrated ordering. It does not reuse a stale percentage.
The estimates concern word correctness in the supplied development population,
not whole-sentence probability or language-level statistical significance.

## Development results

### What was measured

Both bounded checks completed using real `.dproj` workspaces. Each starts from
the existing rule model with **110 unique training sentences**, then fits
uncertainty calibration on the next **100 unique training-file sources** and
validates on unique development sources. Whole-source duplicates are removed
before selection; the first annotation is kept and exclusions/conflicts are logged.

Context was explicitly disabled for this controlled measurement so the frozen
word-gloss baseline condition stays comparable with the earlier prefix results.
Normal user projects retain A's configured context support. Calibration never
changed the learner, and full before/after candidate records match exactly.

### Supervision budgets

| Language | Learner training | Calibration fitting | Calibration validation |
|---|---:|---:|---:|
| Lezgi | 110 sentences / 1,115 word labels | 100 / 916 | 85 / 935 |
| Tsez | 110 sentences / 1,175 word labels | 100 / 1,088 | 445 / 4,761 |

Calibration and validation labels are additional supervision. They are not
counted as labels used to teach vocabulary or morphology, and do not enter the
learner's annotations. The three selections are disjoint by whole NFC source.
Every record ID/order, source hash, original file checksum and configuration is
saved. Gitksan was not accessed.

### Calibration outcome

| Language | Frozen word-gloss accuracy | Estimate coverage | Held-out ECE | Brier score | Constant-prior Brier | Gate |
|---|---:|---:|---:|---:|---:|---|
| Lezgi | 59.57% | 100% | 0.025968 | 0.063041 | 0.242034 | Validated |
| Tsez | 41.59% | 100% | 0.019659 | 0.085437 | 0.243017 | Validated |

Lower ECE and Brier are better. Both artifacts passed the thresholds declared
before these runs. This validates correctness estimates for the supplied frozen
development samples. It does not improve the underlying gloss predictions or
establish calibration for arbitrary languages or future model revisions.

The unchanged word-gloss accuracy is the same as the earlier immediate-feedback
110-sentence prefix condition. A major contributor to good correctness calibration
is recognizing unsupported outputs: an unknown gloss cannot exactly match a
nonempty target. A low calibration error is not evidence that unknown meanings
have been learned.

### Which items were presented first

The following compares the first **10 sentence reviews** in each order. A sentence
review requires a label for every word, so the word annotation cost is also shown.

| Language | Order | Word labels reviewed | Incorrect words found | Errors per reviewed word |
|---|---|---:|---:|---:|
| Lezgi | Upstream source order | 140 | 49 | 35.00% |
| Lezgi | Unknown/ambiguity priority | 61 | 42 | 68.85% |
| Lezgi | Calibrated priority | 61 | 42 | 68.85% |
| Tsez | Upstream source order | 111 | 59 | 53.15% |
| Tsez | Unknown/ambiguity priority | 87 | 81 | 93.10% |
| Tsez | Calibrated priority | 87 | 81 | 93.10% |

Priority concentrated mistakes into fewer word annotations. On Lezgi it found
fewer total errors per ten sentences, but used substantially fewer word labels.
On Tsez it found more errors with fewer word labels. These are different cost
comparisons; the report does not hide them behind one “efficiency” percentage.

The heuristic and calibrated order selected the same first ten items in both
languages. These runs therefore do **not** demonstrate extra ranking benefit
from calibration at that budget. They establish supported correctness estimates
and a useful unknown-first review order. No corrections were applied while
measuring the review queues, so this is error concentration, not a measured
active-learning gain.

### Reproducibility and integrity

Measurements record implementation commit `e1fad21`. A later shell text-parsing
fix does not change the learner or calibrator. Verification against the final
code reconstructed:

- original full validation candidates from reopened workspaces;
- word metrics from saved predictions and separate private scoring records;
- calibration Brier/ECE and reliability bins;
- review order and word-annotation costs;
- source memberships, checksums, model identities and the absence of validation
  labels from authoritative learner annotations.

The production CLI smoke run taught anchors and a plural form, correctly predicted
`mirum` as `bird-PL` before confirmation, retained that original prediction, kept
an unreviewed sentence containing an apostrophe literal, and reopened/restored its
backup with identical model and teaching metadata. It ran with third-party package
loading disabled.

The final **129-test suite** passes, including all seven neural tests and A's
workspace/web tests. New and modified runtime files also parse with Python 3.10
grammar; execution was on the local recorded Python 3.13 runtime, so this is not
a claim of a separate Python 3.10 runtime test.

### Reports and full local artifacts

- [Lezgi summary](../../reports/product/review-calibration/lezgi-v1/summary.md)
  and [detailed results](../../reports/product/review-calibration/lezgi-v1/results.json).
- [Tsez summary](../../reports/product/review-calibration/tsez-v1/summary.md)
  and [detailed results](../../reports/product/review-calibration/tsez-v1/results.json).
- [Reconstruction and real CLI verification](../../reports/product/review-calibration/verification.json).

Raw workspaces, calibration artifacts, prediction logs and private scoring files
remain under ignored `runs/product-review-{lezgi,tsez}-v1`. The synthetic real-CLI
smoke workspace and backup are under `runs/product-teaching-smoke`.

Historical reports are unchanged. These measurements do not establish language-level
significance, unrestricted translation or an external user trial.

## Verification

**129 tests passed, none skipped.** This includes the original 83 tests, A's six
new project/web tests, and B's 40 tests: 14 teaching, seven review, nine calibration,
and ten production workspace integration tests. All seven optional neural tests ran.

Checks cover prediction-before-answer behavior, immutable history, malformed input,
source-ID reuse, Unicode, apostrophes, stale revisions, retries, explicit replacements,
read-only review/calibration, CLI/web stale writers in both directions, failed
atomic saves, reopen, export/restore and malformed audit metadata. CLI tests and
the real smoke flow also run with `-S`, without third-party site packages.

Both bounded Lezgi/Tsez checks passed calibration gates and reconstructed saved
scores/candidates/costs exactly. See [results and limits](#development-results).
Final offline corpus checksum verification and `git diff --check` passed.

## Commands and recovery

### Start a real project

From the repository root, choose a fresh path:

```powershell
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach init --project runs/my-language.dproj --name "My language" --language Demo
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach shell --project runs/my-language.dproj
```

Inside the shell:

```text
submit anchors tal mir
predict anchors
correct tree bird
submit plural talum
predict plural
correct tree-PL
submit probe mirum
predict probe
confirm
review 10
search bird
history probe
export runs/my-language-backup.zip
quit
```

Reopen with the same shell command. Each successful change is saved. Source and
gloss words are whitespace segmented; source text and apostrophes are literal.
Use simple sentence IDs without spaces for interactive `submit` commands.
`replace REASON :: WORD-GLOSSES` replaces a different verified answer after a
new prediction. `reload` reopens newer saved work after a stale-client conflict.
`gloss teach --help` lists all one-shot commands and `COMMAND --help` shows options.

A project imported/created through A's UI can be opened with the same CLI:

```powershell
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach review --project PATH_TO_PROJECT.dproj --limit 10
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach search --project PATH_TO_PROJECT.dproj --query bird --history
```

The actual smoke project is available locally at
`runs/product-teaching-smoke/demo.dproj`, with its backup and restored copy beside it.
The fixture backend is optional test infrastructure; normal commands do not need it.

### Fit and use calibration

Supply separate complete labeled source/gloss files. Training-source overlaps,
duplicates and alignment exclusions are reported, and fitting/validation sets must
remain nonempty and disjoint. These labels are additional supervision.

```powershell
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach calibrate --project PATH_TO_PROJECT.dproj --fit PATH_TO_CALIBRATION_TIERS --validation PATH_TO_VALIDATION_TIERS --output runs/my-calibration.json
.venv/Scripts/python.exe -X utf8 -S -m decipher.app gloss teach review --project PATH_TO_PROJECT.dproj --calibration runs/my-calibration.json --limit 10
```

Passing the declared gate enables empirical estimates of **top word-gloss
correctness** for that exact model and development population. It does not make
raw rule scores probabilities. Changed model evidence, context settings or engine
identity make the artifact stale. Insufficient/failed artifacts produce no estimates;
review uses clearly uncalibrated priorities instead. Calibration artifacts are
explicit separate JSON files; preserve them separately if needed with a backup.

### Reproduce the development checks

The following are the exact completed commands. Change both output paths to a new
suffix when rerunning; existing artifacts are protected from overwrite.

```powershell
.venv/Scripts/python.exe -X utf8 -S tools/evaluate_review_calibration.py --language lezgi --run-dir runs/product-review-lezgi-v1 --report-dir reports/product/review-calibration/lezgi-v1
.venv/Scripts/python.exe -X utf8 -S tools/evaluate_review_calibration.py --language tsez --run-dir runs/product-review-tsez-v1 --report-dir reports/product/review-calibration/tsez-v1
```

Defaults are 110 learner-training sentences, 100 calibration-fitting sentences,
the unique development set, and a comparison of the first ten sentence reviews.
The tool verifies pinned corpus checksums and uses real workspaces with context
explicitly off to preserve the frozen baseline condition. It never accesses Gitksan.

Run the test suite on this Windows environment with UTF-8 enabled:

```powershell
$env:PYTHONUTF8='1'
.venv/Scripts/python.exe -m unittest discover -s tests -v
.venv/Scripts/python.exe tools/fetch_corpora.py --collection glossing --corpus lezgi --corpus tsez --verify-only
```


## Remaining boundaries

Review/calibration modules can be reused by future UI widgets. This milestone
exposes review/search/history through the CLI and adds the necessary browser
revision guard. It does not redesign A's UI, complete an external user trial or
claim an active-learning gain. Calibration files remain explicit separate JSON
artifacts and should be preserved separately if needed with a backup. No reserved
language evaluation, new dependency or default stem-model promotion was introduced.
There is no remaining B implementation blocker for Features 2 and 6.
