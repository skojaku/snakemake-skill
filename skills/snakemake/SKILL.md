# Snakemake Style Guide

Copy `utils.smk` from this skill to project root. `include: "./utils.smk"` at top of every Snakefile.

## Layout

```
project/
├── Snakefile              # Config, globals, params, includes
├── utils.smk              # Utilities (from this skill)
├── workflow/
│   ├── config.template.yaml
│   ├── rules/             # One .smk per stage
│   ├── <category>/        # Scripts by purpose (stats/, plot/, fitting/)
│   └── preprocessing/{dataset}/
└── data/{dataset}/            # Optional tiered structure:
    ├── preprocessed/      #   Inputs
    ├── derived/           #   Computed
    └── plot_data/         #   For figures
```

## Snakefile

```python
from os.path import join as j
include: "./utils.smk"
configfile: "workflow/config.yaml"

DATA_DIR = config["data_dir"]
DATA_LIST = ["dataset_a", "dataset_b"]

# Param dicts — values ALWAYS lists
params_model = {"dim": [64], "lr": [0.001]}
model_ps = to_paramspace(params_model)

# Paths — UPPER_CASE, use j(), wildcard_pattern in f-strings
INPUT_NET = j(DATA_DIR, "{data}", "preprocessed", "net.npz")
MODEL_FILE = j(DATA_DIR, "{data}", "derived", f"model_{model_ps.wildcard_pattern}.pt")
# -> data/{data}/derived/model_dim~{dim}_lr~{lr}.pt
# Use {{wildcard}} for literal braces in f-strings

include: "workflow/rules/stage_a.smk"

rule all:
    input: rules.stage_a_all.input,
```

## Naming

File paths `UPPER_CASE`. Param dicts `lowercase`. Rules `snake_case`. Param values always lists.

## utils.smk

- `to_paramspace(dict)` — Cartesian product -> Paramspace. `.wildcard_pattern` for paths. Never hand-build `key~value`.
- `expand(file, *param_dicts, **kw)` — Use instead of built-in. Unfilled wildcards stay as `{name}`.
- `partial_format(template, **kw)` — Fill some placeholders, leave rest.

## Rules

Named I/O always. Lambda for wildcards/conditionals. Each .smk has aggregation rule.

```python
rule calc_something:
    input:
        net_file = INPUT_NET,
    output:
        output_file = OUTPUT_FILE
    params:
        dim = lambda wildcards: wildcards.dim,
        label = lambda wildcards: "A" if wildcards.data == "x" else "B",
    script:
        "../stats/calc-something.py"  # relative to .smk file

rule stage_a_all:  # aggregation rule
    input: expand(OUTPUT_FILE, data=DATA_LIST),
```

Master target: `rule all: input: rules.stage_a_all.input,`

### Rule Reuse

`use rule X as Y with:` — inherits `script:`, override I/O/params:

```python
use rule calc_stat as calc_stat_sim with:
    input: net_file = SIM_NET,
    output: output_file = SIM_STAT
    params: label = "Simulated",
```

## Scripts

Dual-mode: Snakemake + interactive. Params arrive as strings — convert types.

```python
import sys
if "snakemake" in sys.modules:
    input_file = snakemake.input["input_file"]
    output_file = snakemake.output["output_file"]
    dim = int(snakemake.params["dim"])         # str -> int
    flag = snakemake.params["flag"] == "True"  # str -> bool
else:
    input_file = "../data/sample.npz"
    output_file = "../data/output.csv"
    dim = 64
    flag = True
```

## Recipes

### New computation

1. Write script in `workflow/<category>/calc-something.py` using dual-mode pattern
2. Define path in Snakefile or .smk: `OUTPUT_FILE = j(DATA_DIR, "{data}", "derived", "something.csv")`
3. Write rule with named I/O pointing to that path
4. Add to aggregation: `expand(OUTPUT_FILE, data=DATA_LIST)` in the `rule stage_all:`

### New parameter

1. Add to param dict in Snakefile: `params_model = {"dim": [64], "new_param": [10, 20]}`
2. File paths using `wildcard_pattern` auto-include it — no path changes needed
3. Forward in rule: `params: new_param = lambda wildcards: wildcards.new_param,`
4. Convert in script: `new_param = int(snakemake.params["new_param"])`

### New dataset

1. Create `workflow/preprocessing/my_dataset/Snakefile` with ingestion rules
2. Include from main Snakefile: `include: "workflow/preprocessing/my_dataset/Snakefile"`
3. Add to list: `DATA_LIST = ["dataset_a", "dataset_b", "my_dataset"]`

## Checklist

- UPPER_CASE path constants, list-valued param dicts
- Custom `expand()` from utils.smk, not built-in
- `to_paramspace()` + `.wildcard_pattern` for parameterized paths
- Named rule I/O, `j()` for all paths
- Dual-mode scripts with type conversion
- Aggregation rule per .smk, `use rule` over duplication
- `{{wildcard}}` in f-strings
