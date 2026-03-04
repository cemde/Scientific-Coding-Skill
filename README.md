# Scientific Coding Skill

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that teaches Claude to write correct scientific code.

Scientific code serves truth, not users. This plugin encodes the principles that follow from that distinction: fail loudly instead of silently, require explicit parameters instead of dangerous defaults, prioritize correctness over convenience.

## Install

```bash
claude /plugin install --dir /path/to/Scientific-Coding-Skill
```

Or test locally:

```bash
claude --plugin-dir /path/to/Scientific-Coding-Skill
```

## What It Does

The skill activates automatically when Claude writes code for scientific computing, data analysis, simulations, numerical methods, or statistical analysis. 12 principles:

1. **Understand the experiment** before deciding what to parameterize
2. **You are a co-researcher** -- ask questions, verify, read papers
3. **Fail loudly** -- a crash beats a silently wrong result
4. **No defaults in code** -- defaults belong in config files
5. **Be explicit** about units, shapes, assumptions, conversions
6. **Correctness over performance** -- numerical stability first
7. **Raw data is immutable** -- never modify, always trace
8. **Reproducibility is non-negotiable** -- explicit seeds, pinned versions
9. **Validate against reality** -- known solutions, conservation laws, convergence
10. **Statistical honesty** -- no p-hacking, report effect sizes
11. **Do not reinvent** -- use established tools, skip custom infrastructure
12. **Do not refactor validated code** -- working science code is sacred

See [skills/scientific-coding/SKILL.md](skills/scientific-coding/SKILL.md) for the full set of principles.

## Structure

```
Scientific-Coding-Skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── scientific-coding/
│       ├── SKILL.md             # Core principles
│       └── examples/
│           ├── common-mistakes.md
│           └── full-analysis.md
└── README.md
```

## Standalone Use

Copy the skill directory into your project or personal skills:

```bash
# Project-level
cp -r /path/to/Scientific-Coding-Skill/skills/scientific-coding .claude/skills/

# Personal (all projects)
cp -r /path/to/Scientific-Coding-Skill/skills/scientific-coding ~/.claude/skills/
```

## License

MIT
