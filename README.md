# Scientific Coding Skill

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that teaches Claude to write correct scientific code.

Scientific code serves truth, not users. This plugin encodes the principles that follow from that distinction: fail loudly instead of silently, require explicit parameters instead of dangerous defaults, prioritize correctness over convenience.

## Install

```bash
claude /plugin install --dir /path/to/Scientific-Coding-Skill
```

Or test locally during development:

```bash
claude --plugin-dir /path/to/Scientific-Coding-Skill
```

## What It Does

The skill activates automatically when Claude writes code for scientific computing, data analysis, simulations, numerical methods, or statistical analysis. 12 principles:

1. **Understand the experiment** before deciding what to parameterize
2. **Ask the researcher** when uncertain about methodology
3. **Fail loudly** -- a crash beats a silently wrong result
4. **No defaults for experimental parameters** -- every default is a hidden assumption
5. **Be explicit** about units, shapes, assumptions, conversions
6. **Correctness over performance** -- numerical stability first
7. **Raw data is immutable** -- never modify, always trace
8. **Reproducibility is non-negotiable** -- explicit seeds, pinned versions
9. **Validate against reality** -- known solutions, conservation laws, convergence
10. **Statistical honesty** -- no p-hacking, report effect sizes
11. **Clarity over abstraction** -- readable science beats clever patterns
12. **Do not refactor validated code** -- working science code is sacred

See [SKILL.md](SKILL.md) for the full set of principles.

## Structure

```
Scientific-Coding-Skill/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   └── scientific-coding/
│       ├── SKILL.md          # Core principles (loaded by Claude)
│       └── examples/
│           ├── common-mistakes.md  # Wrong vs right code examples
│           └── full-analysis.md    # Complete worked example
├── SKILL.md                  # Also at root for standalone use
└── README.md
```

The SKILL.md exists both at the root (for standalone `.claude/skills/` use) and inside `skills/scientific-coding/` (for plugin use).

## Standalone Use (Without Plugin)

Copy or symlink SKILL.md into your project or personal skills:

```bash
# Project-level
mkdir -p .claude/skills/scientific-coding
cp /path/to/Scientific-Coding-Skill/SKILL.md .claude/skills/scientific-coding/

# Personal (all projects)
mkdir -p ~/.claude/skills/scientific-coding
cp /path/to/Scientific-Coding-Skill/SKILL.md ~/.claude/skills/scientific-coding/
```

## License

MIT
