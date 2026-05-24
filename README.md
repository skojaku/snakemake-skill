# Snakemake Style Guide & Utilities

A style guide and utility functions for building clean, reproducible [Snakemake](https://snakemake.readthedocs.io/) workflows.

## What's included

- **`skills/snakemake/SKILL.md`** — A comprehensive style guide covering project organization, naming conventions, rule patterns, script conventions, and recipes for common tasks
- **`skills/snakemake/utils.smk`** — Drop-in utility functions for your Snakemake projects:
  - `to_paramspace()` — Create `Paramspace` objects from parameter dicts (Cartesian product)
  - `expand()` — Custom expand with partial wildcard substitution (unfilled wildcards stay as `{name}` instead of raising errors)
  - `partial_format()` — Fill some placeholders in a template, leave the rest untouched
  - `make_filename()` — Build parameterized filename templates

## Usage

### Using `utils.smk` in your project

Copy `skills/snakemake/utils.smk` to your project root and include it at the top of your Snakefile:

```python
include: "./utils.smk"
```

Then use it:

```python
# Define parameters (values are always lists)
params_model = {"dim": [64, 128], "lr": [0.001]}

# Create paramspace
model_paramspace = to_paramspace(params_model)

# Use in file paths
MODEL_FILE = j(DATA_DIR, "{data}", f"model_{model_paramspace.wildcard_pattern}.pt")
# -> data/{data}/model_dim~{dim}_lr~{lr}.pt

# Expand with partial wildcards (unfilled ones stay as templates)
expand(MODEL_FILE, params_model, data=["dataset_a", "dataset_b"])
```

### Using as an AI coding assistant skill

This repo can be used as a skill/prompt for AI coding assistants so they follow consistent Snakemake conventions.

#### Claude Code

```bash
claude skills add skojaku/snakemake-skill
```

#### OpenAI Codex / Other agents

Copy the contents of `skills/snakemake/SKILL.md` into your system prompt, project instructions, or `AGENTS.md` file.

## Key conventions from the style guide

- **Project layout**: `Snakefile` + `utils.smk` at root, rules in `workflow/rules/`, scripts in `workflow/<category>/`
- **Naming**: `UPPER_CASE` for file path constants, `snake_case` for rules, parameter dict values always wrapped in lists
- **Parameterized paths**: Always use `to_paramspace().wildcard_pattern` — never hand-build `key~value` strings
- **Named I/O**: Rule inputs and outputs are always named, not positional
- **Dual-mode scripts**: All scripts work both via Snakemake and interactively using `if "snakemake" in sys.modules:`
- **Rule reuse**: `use rule ... as ... with:` instead of duplicating scripts
- **Aggregation rules**: Each `.smk` file has a master target rule; the main `rule all:` references them

## License

MIT
