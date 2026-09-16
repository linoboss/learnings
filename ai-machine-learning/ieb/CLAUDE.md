# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Coursework for an AI/Machine Learning program (IEB). Each `clase{N}/sprintM/` directory is a self-contained
graded exercise: a Jupyter notebook (sometimes suffixed `_LinoBossio` or `- LinoBossio`) plus whatever
dataset/CSV it consumes. There is no application code, package, or test suite — the deliverables are the
notebooks and, for the `final/` folders, a couple of Spanish-language design docs. Treat each sprint folder
as independent; don't assume shared code between them (there isn't any — each notebook is self-contained,
often with duplicated setup/EDA boilerplate).

- `clase1/` — sprints 1–3 plus a `final/` capstone: dataset import & EDA → linear regression → model
  comparison (Ridge/Lasso/Gradient Boosting) → a student-dropout-risk predictive model with two conceptual
  design docs (`fase2_asistente_virtual_conceptual.md` — a RAG + MCP virtual assistant design, and
  `fase3_arquitectura_cloud.md` — a full Azure cloud architecture) tying the model into a larger proposed
  system. These two `.md` files are the actual submitted deliverables for that capstone, not scratch notes.
- `clase2/` — sprints 4–6: descriptive stats on a fetal-state dataset → Naive Bayes classifier → SVM
  (including grid search / linear-kernel / C-parameter variants), building toward a `final/` (currently
  empty) capstone.

Package name in `pyproject.toml` is `ieb`; there's no other branding to be aware of.

## Environment & running notebooks

Dependency management is `uv`. **Do not run `uv add`/`uv sync`/`pip install` yourself** — if a notebook
needs a package that isn't installed, report which module is missing and let the user install it.

```
uv run jupyter lab                 # open notebooks interactively
uv run jupyter nbconvert --to notebook --execute --inplace <path>.ipynb   # re-run a notebook headlessly
```

No lint/test/build commands exist in this repo — there's nothing to run beyond executing the notebooks
themselves. `python -c "import nbformat; nb=nbformat.read('<path>', 4); nbformat.validate(nb)"` is a quick
way to sanity-check a notebook is well-formed JSON after editing it programmatically.

## Data handling

- `**/data/` is globally gitignored — any dataset placed under a `data/` subdirectory (see
  `clase1/sprint1/data/`) will never be committed. Datasets living directly in a sprint folder (e.g.
  `ASI_casoPractico.csv`) are NOT covered by that rule and do get committed.
- Notebooks source data three ways, each demonstrated in `sprint1.ipynb`: local upload, the Kaggle API
  (credentials in `.kaggle/access_token`, gitignored), and Google Drive. Never read or print the contents
  of `.kaggle/access_token`.
- Several sprints reuse the same CSV as the previous sprint (e.g. `clase2/sprint5` and `sprint6` both build
  on the train/test split from `clase2/sprint4`'s dataset) — check the notebook's own data-loading cell
  before assuming a dataset needs to be re-derived from scratch.

## Editing notebooks

When asked to modify a notebook's content (not just re-run it), it's usually more reliable to script the
edit with `nbformat` (read → mutate `cells` → write) than to hand-edit the raw JSON. Preserve existing
outputs unless the user asks you to re-execute. Markdown cells carry the real narrative/structure (headers
like `# Fase 1 —`, `## 1.1 —`) — keep heading numbering and section names intact when inserting new cells
so the notebook's structure stays consistent with what was graded.
