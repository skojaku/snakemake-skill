# Snakemake Skill for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that provides a comprehensive Snakemake style guide and utility functions for building reproducible data pipelines.

## What's included

- **SKILL.md** — Style guide covering project organization, naming conventions, rule patterns, script conventions, and recipes for common tasks
- **utils.smk** — Utility functions (`to_paramspace`, `expand`, `partial_format`, etc.) to include in your Snakemake projects

## Installation

Add this skill to your Claude Code configuration:

```bash
claude skills add skojaku/snakemake-skill
```

Or add it manually in your `.claude/settings.json`:

```json
{
  "skills": ["skojaku/snakemake-skill"]
}
```

## Usage

Once installed, Claude Code will automatically follow the Snakemake conventions defined in this skill when creating or modifying Snakemake workflows. You can also invoke it explicitly with `/snakemake`.

### Key conventions

- Use `utils.smk` for parameterized file paths via `to_paramspace()` and custom `expand()`
- File path constants in `UPPER_CASE`, parameter dicts with list values
- Scripts support both Snakemake and interactive execution via `if "snakemake" in sys.modules:`
- Rules use named inputs/outputs and lambda wildcards for conditional params
- Rule reuse with `use rule ... as ... with:` instead of duplicating scripts

## License

MIT
