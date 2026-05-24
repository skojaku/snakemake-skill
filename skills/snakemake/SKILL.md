# Snakemake Style Guide

Copy `utils.smk` from this skill to project root. Include at top of every Snakefile.

## Layout

```
project/
├── Snakefile          # Config, globals, params, includes
├── utils.smk          # Utilities (from this skill)
├── workflow/
│   ├── config.template.yaml
│   ├── rules/         # One .smk per stage
│   ├── <category>/    # Scripts by purpose (stats/, plot/, fitting/)
│   └── preprocessing/{dataset}/
└── data/{dataset}/    # Optional tiers: preprocessed/, derived/, plot_data/
```

## Snakefile

```python
from os.path import join as j
include: "./utils.smk"
configfile: "workflow/config.yaml"

DATA_DIR = config["data_dir"]
DATA_LIST = ["dataset_a", "dataset_b"]

params_model = {"dim": [64], "lr": [0.001]}  # values ALWAYS lists
model_ps = to_paramspace(params_model)

# Paths: UPPER_CASE, j(), wildcard_pattern in f-strings, {{x}} for literal braces
INPUT_NET = j(DATA_DIR, "{data}", "preprocessed", "net.npz")
MODEL_FILE = j(DATA_DIR, "{data}", "derived", f"model_{model_ps.wildcard_pattern}.pt")

include: "workflow/rules/stage_a.smk"
rule all:
    input: rules.stage_a_all.input,
```

Naming: paths `UPPER_CASE`, param dicts `lowercase`, rules `snake_case`.

## utils.smk

- `to_paramspace(dict)` — Cartesian product -> Paramspace. Use `.wildcard_pattern` for paths.
- `expand(file, *param_dicts, **kw)` — Replaces built-in. Unfilled wildcards stay as `{name}`.
- `partial_format(template, **kw)` — Fill some placeholders, leave rest.

## Rules

Named I/O. Lambda for wildcards. Each .smk has aggregation rule. Scripts relative to .smk file.

```python
rule calc_something:
    input: net_file = INPUT_NET,
    output: output_file = OUTPUT_FILE
    params:
        dim = lambda wildcards: wildcards.dim,
        label = lambda wildcards: "A" if wildcards.data == "x" else "B",
    script: "../stats/calc-something.py"

rule stage_a_all:
    input: expand(OUTPUT_FILE, data=DATA_LIST),
```

Reuse: `use rule X as Y with:` — inherits script, override I/O/params.

## Scripts

Dual-mode (Snakemake + interactive). Params arrive as strings — convert types.

```python
import sys
if "snakemake" in sys.modules:
    input_file = snakemake.input["input_file"]
    dim = int(snakemake.params["dim"])         # str->int
    flag = snakemake.params["flag"] == "True"  # str->bool
else:
    input_file = "../data/sample.npz"
    dim = 64
    flag = True
```

## Recipes

**New computation:** dual-mode script in `workflow/<category>/` -> `OUTPUT = j(...)` -> rule -> add `expand(OUTPUT, ...)` to aggregation rule.

**New parameter:** add to param dict as list -> paths auto-update via `wildcard_pattern` -> `params: x = lambda w: w.x` -> `int(snakemake.params["x"])` in script.

**New dataset:** create `workflow/preprocessing/{name}/Snakefile` -> include from main -> add to `DATA_LIST`.

## Checklist

- UPPER_CASE paths, list-valued params, `j()` everywhere
- `to_paramspace()` + `.wildcard_pattern`, never hand-build `key~value`
- Custom `expand()`, not built-in
- Named I/O, dual-mode scripts, type conversion
- Aggregation rule per .smk, `use rule` over duplication
