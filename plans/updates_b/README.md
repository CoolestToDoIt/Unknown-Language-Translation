# Worker B updates

## Start here

These are three consolidated milestone documents, ordered from infrastructure to
product features to the model experiment. Each brings together scope/checklist,
implementation decisions, verification, measured results, commands and limitations.
Test counts describe the recorded milestone, not fresh tests of every later change.

| Document | What to use it for | Recorded outcome |
|---|---|---|
| [01 — Evaluation and resume](01-evaluation-and-resume.md) | Experiment manifests, annotation budgets, checkpoint recovery, online matrix results and reproduction | 63 tests; 36 complete development runs; restart verified |
| [02 — Teaching and review](02-teaching-and-review.md) | Real `.dproj` teaching commands, safe revisions, review/search/history, calibration and workspace integration | Features 2/6 complete; 129 tests; both calibration checks validated |
| [03 — Internal-edit experiment](03-internal-edit-experiment.md) | Frozen Feature 4 protocol, implementation, error analysis, adoption decision and publication history | 146 tests; both corpus/audit runs complete; reject default adoption |

The completed product and experimental branches were pushed and integrated into
main at `b46e649`. Including experimental code/reports does not enable that model
in the application. Feature 4's earlier deferral is historical; the approved
experiment is complete and the default learner remains unchanged.

For practical use, start with [teaching commands](02-teaching-and-review.md#commands-and-recovery).
For the latest model conclusion, start with [the decision](03-internal-edit-experiment.md#results-and-decision).

## Verification and environment

Run from the repository root. Windows UTF-8 mode handles the language fixtures and
is inherited by subprocesses. The installed Windows Store Python environment
requires execution access; no new dependencies were needed for these milestones.

```powershell
$env:PYTHONUTF8='1'
.venv/Scripts/python.exe -m unittest discover -s tests -v
.venv/Scripts/python.exe tools/fetch_corpora.py --collection glossing --corpus lezgi --corpus tsez --verify-only
```

The gloss tools use the standard library. Several documented smoke checks use
`-S` to disable third-party packages; the full suite runs without it so installed
PyTorch tests execute. Historical runs emitted a missing-NumPy warning but their
neural checks passed. The original 45-test baseline's Windows encoding errors
were resolved by `PYTHONUTF8=1`, without changing the learner.

Use fresh run/report paths when reproducing experiments. Full local workspaces,
candidate logs, private scoring and checkpoints live under ignored `runs/`;
compact versioned evidence lives under `reports/`. Gitksan remains reserved.

## Organization and maintenance

The consolidation replaces **20 files with four**: this index and one substantive
document per milestone. Repeated status updates and obsolete pending claims were
removed; important decisions, results, commands, provenance and limits were kept.
The original dated notes and `product-features/` and `feature-4/` note sequences
remain available in Git history. Repository links now target the consolidated
documents and their relevant sections. The original frozen Feature 4 protocol is
still read from its historical commit, as explained in that milestone document.

- [x] Review the old notes and repository references.
- [x] Group the material into three completed milestones and one navigation page.
- [x] Preserve the frozen conditions, numerical results, commands and source history.
- [x] Replace old links and remove superseded files.
- [x] Validate local links/anchors, historical protocol lookup and documentation-only scope.

Consolidation checks passed for 77 local links/section anchors across ten
changed/current documents. All four frozen protocol sections match the original
verbatim, and the historical protocol hash still matches both recorded language
reports. Runtime code, tests and machine-readable results were unchanged, so
verification focused on documentation rather than rerunning the model suite.

For future updates, extend the matching milestone's substantive sections. Create
a new document only for a new major deliverable or decision, rather than one file
per implementation step. Keep code, raw evidence and machine-readable reports in
their existing locations, linking to them instead of copying them into more notes.
