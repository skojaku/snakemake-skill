# Snakemake Style Guide & Utilities

## Installation

### As an AI coding assistant skill

**Claude Code:**
```bash
npx claude-code skills add skojaku/snakemake-skill
```

**OpenAI Codex / Cursor / other agents:** Copy `skills/snakemake/SKILL.md` into your system prompt, project instructions, or `AGENTS.md`.

### As a standalone utility

Copy `skills/snakemake/utils.smk` to your project root:

```python
include: "./utils.smk"
```

## What is Snakemake?

[Snakemake](https://snakemake.readthedocs.io/) is a workflow engine for data pipelines. You define rules — each rule takes input files, runs a script, and produces output files. Snakemake figures out the dependency graph automatically.

Change a parameter or fix a script? Snakemake detects exactly which downstream files are stale and reruns only what's needed. You don't have to remember which scripts depend on what. It handles that for you.

It scales from a laptop to a cluster. Same workflow file, no code changes.

## Why this repo?

I've used Snakemake since 2020 for all my research projects and fell in love with it. My routine: kick off a pipeline in the morning, go for a run, come back to results. Change one parameter, and Snakemake traces every downstream dependency and reruns exactly what's needed — I don't have to keep any of that in my head. It gives me reproducibility, scalability, and free time.

But Snakemake is flexible enough to become messy. Over the years I've settled on conventions that keep projects clean as they grow: how to name files, structure rules, handle parameters, write scripts that work both inside and outside Snakemake. This repo captures those patterns so I (and my AI coding assistants) follow them consistently.

## What's included

- **`skills/snakemake/SKILL.md`** — Style guide: project layout, naming, rule patterns, script conventions, recipes
- **`skills/snakemake/utils.smk`** — Drop-in utilities:
  - `to_paramspace(dict)` — Cartesian product of parameters -> Snakemake `Paramspace` with `wildcard_pattern` for file paths
  - `expand(file, *param_dicts, **kw)` — Like built-in `expand()` but unfilled wildcards stay as `{name}` instead of erroring
  - `partial_format(template, **kw)` — Fill some placeholders, leave the rest

## Quick example

```python
from os.path import join as j
include: "./utils.smk"

params_model = {"dim": [64, 128], "lr": [0.001]}
model_ps = to_paramspace(params_model)

MODEL_FILE = j("data", "{data}", f"model_{model_ps.wildcard_pattern}.pt")
# -> data/{data}/model_dim~{dim}_lr~{lr}.pt

rule all:
    input: expand(MODEL_FILE, params_model, data=["physics", "biology"])
    # generates all combinations: 2 dims x 1 lr x 2 datasets = 4 files
```

## Key conventions

- **Paths** in `UPPER_CASE`, built with `j()`. Parameterized via `to_paramspace().wildcard_pattern`.
- **Param dicts** — values always lists, even single values: `{"dim": [64]}`
- **Rules** — named I/O, lambda for wildcards, aggregation rule per `.smk` file
- **Scripts** — dual-mode (`if "snakemake" in sys.modules:` / `else:`) so they run both in pipeline and interactively
- **Reuse** — `use rule X as Y with:` instead of copying scripts

See `skills/snakemake/SKILL.md` for the full guide.

## Learn Snakemake

New to Snakemake? See [TUTORIAL.md](TUTORIAL.md) for an introduction covering directives, wildcards, paramspace, recommended config, and links to real project examples.

## License

MIT
