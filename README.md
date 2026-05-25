# Snakemake Style Guide & Utilities

[Snakemake](https://snakemake.readthedocs.io/) is, in short, is a Python-based workflow manager that defines & manages analysis steps.

I've used Snakemake since 2020 for all my research projects. Snakemake keeps my projects reproducible as I go. For example:

1. We sometimes want to change a part of scripts in an upperstream process of data. Snakemake identifies the dependencies of data and scripts, and rerun only the scripts that needed to update the results.
2. We sometimes want to scale up the experiment, e.g., testing the same analysis with new data and new parameters. With Snakemake, I can simply add parameter values and paths to some lists. Snakemake then runs the analysis for *every* combination of parameters and data.

Snakemake lets me free from managing complex workflow and stay focus on what matters.

But Snakemake is flexible enough to become messy. Over the years I've settled on conventions that keep projects clean as they grow, things like how to name files, structure rules, handle parameters, write scripts that work both inside and outside Snakemake. This repo captures those patterns so I (and my AI coding assistants) follow them consistently.

## Installation

### Using `npx skills` (works across agents)

[`npx skills`](https://github.com/vercel-labs/skills) is a package manager for AI agent skills. It installs to Claude Code, Codex, Cursor, and others.

```bash
# Install to all detected agents
npx skills add skojaku/snakemake-skill --all

# Or target a specific agent
npx skills add skojaku/snakemake-skill -a claude-code
npx skills add skojaku/snakemake-skill -a codex
```

### Manual install

**Claude Code:** Copy `skills/snakemake/SKILL.md` to `~/.claude/skills/snakemake/SKILL.md`

**Codex / Cursor / other agents:** Copy the contents of `skills/snakemake/SKILL.md` into your `AGENTS.md` or system prompt.

### As a standalone utility (no AI agent needed)

Copy `skills/snakemake/utils.smk` to your project root:

```python
include: "./utils.smk"
```

## My conventions

All file path constants are `UPPER_CASE` and built with `j()` (alias for `os.path.join`). When a path depends on parameters, use `to_paramspace().wildcard_pattern` to generate the wildcard portion of the filename. This keeps parameterized paths consistent and parseable, e.g., `model_dim~64_lr~0.001.pt`, without ever hand-building `key~value` strings.

```python
from os.path import join as j
include: "./utils.smk"

DATA_DIR = config["data_dir"]
INPUT_NET = j(DATA_DIR, "{data}", "preprocessed", "net.npz")
```

Parameters are defined as plain Python dicts where every value is a list, even single values like `{"dim": [64]}`. `to_paramspace()` takes the Cartesian product, so adding a new parameter or value to the dict automatically generates all combinations.

```python
params_model = {"dim": [64, 128], "lr": [0.001]}
model_ps = to_paramspace(params_model)

MODEL_FILE = j(DATA_DIR, "{data}", "derived", f"model_{model_ps.wildcard_pattern}.pt")
# -> data/{data}/derived/model_dim~{dim}_lr~{lr}.pt
```

Rules always use named inputs and outputs (not positional), so scripts access them by `snakemake.input["net_file"]`. Wildcard values are forwarded to scripts via `params:` with lambdas. Each `.smk` file defines an aggregation rule that collects all its outputs, and the main `rule all:` references these via `rules.stage_all.input`.

```python
# workflow/rules/fitting.smk
rule fit_model:
    input:
        net_file = INPUT_NET,
    output:
        output_file = MODEL_FILE
    params:
        dim = lambda wildcards: wildcards.dim,
        lr = lambda wildcards: wildcards.lr,
    script:
        "../fitting/fit-model.py"

rule fitting_all:
    input: expand(MODEL_FILE, params_model, data=DATA_LIST)
    # 2 dims x 1 lr x 2 datasets = 4 files
```

```python
# Snakefile
include: "workflow/rules/fitting.smk"
include: "workflow/rules/plot.smk"

rule all:
    input:
        rules.fitting_all.input,
        rules.plot_all.input,
```

Scripts are written in dual-mode: they read from `snakemake.input` / `snakemake.params` when run in the pipeline, and fall back to hardcoded paths in an `else` block for interactive development and testing. This way you can iterate on a script in a notebook or terminal without running the full pipeline. Note that params arrive as strings, so convert types explicitly.

```python
# workflow/fitting/fit-model.py
import sys

if "snakemake" in sys.modules:
    net_file = snakemake.input["net_file"]
    output_file = snakemake.output["output_file"]
    dim = int(snakemake.params["dim"])         # str -> int
    lr = float(snakemake.params["lr"])         # str -> float
else:
    net_file = "data/physics/preprocessed/net.npz"
    output_file = "data/physics/derived/model.pt"
    dim = 64
    lr = 0.001

# Main logic (runs the same way in both modes)
import numpy as np
# ... training code ...
```

When multiple rules share the same logic but differ in inputs or parameters (e.g., empirical vs. simulated data), use `use rule X as Y with:` to inherit the script and override only what changes. This avoids duplicating scripts and keeps the logic in one place.

```python
use rule fit_model as fit_model_baseline with:
    input: net_file = BASELINE_NET,
    output: output_file = BASELINE_MODEL
    params: dim = lambda wildcards: wildcards.dim,
```

See `skills/snakemake/SKILL.md` for the full guide.

## What is Snakemake?

[Snakemake](https://snakemake.readthedocs.io/) is a language that defines workflow. Once you set up a workflow with Snakemake, you can run the whole pipeline from raw data all the way down to figures in your paper by pressing a button. Snakemake is fast and powerful, as it automatically parallelizes the workflow and distributes computation.

A workflow consists of *rules*. Each rule takes a file and generates another file. A rule can take files generated by another rule as input, creating links between rules that form a directed network, a *computation graph*, from first input to final output. Snakemake finds an optimal schedule to execute each rule and does not execute a rule if its output already exists. This is useful because we often change parts of a workflow and want to redo only the calculations affected by the change. See [TUTORIAL.md](TUTORIAL.md) for an introduction covering directives, wildcards, paramspace, recommended config, and links to real project examples.

## Open question: conventions in the AI era

These conventions were designed when humans wrote every line of the Snakefile. That's no longer the case. AI agents write most of the workflow code now, and that changes the calculus.

Some conventions mattered because they made *writing* easier for humans. Short aliases like `j()`, terse syntax, compact patterns. Those matter less now because AI writes the code. We switched to `pathlib.Path` not because AI needs it, but because it's more readable when humans *review* AI-generated code.

A harder question is metadata in filenames. We encode parameters directly in filenames (`model_dim~64_lr~0.001.pt`) so you can `ls`, `grep`, and `rm` by pattern without any tooling. The alternative is short hashes (`4f8c2e.pt`) with a registry. AI can look up hashes instantly, so the human-readability argument weakens. But filenames are the one thing that's always visible: when you share files, debug on a remote machine, or just want to see what happened. No tooling, no lookup, no AI needed.

More broadly: should workflow conventions optimize for AI writing + human reviewing? Or should we let AI manage everything, including file organization, and just trust the agent? The more AI takes over, the less these conventions matter. But the less they matter, the harder it gets to understand what's going on when you need to.

We don't have a settled answer. See [issue #1](https://github.com/skojaku/snakemake-skill/issues/1) for the ongoing discussion.

## License

MIT
