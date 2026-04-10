# refactor-agent-friendly

A Claude Code plugin that analyzes and refactors any codebase for AI agent efficiency.

Reduce context cost, improve convention compliance, and set up autonomous development infrastructure — measured with before/after benchmarks.

## What It Does

9-phase workflow that transforms a codebase from "AI can edit it" to "AI develops autonomously with high accuracy":

1. **Measure** — Baseline metrics (context cost, file sizes, convention violations)
2. **Analyze** — Identify gaps (missing rules, duplicated CLAUDE.md, no agents)
3. **Rules** — Create `.claude/rules/` with path-scoped, on-demand context
4. **CLAUDE.md** — Optimize always-loaded context (target: under 300 lines)
5. **Agents** — Create architect/builder/reviewer with structured handoffs
6. **Pipeline** — Orchestrate agents sequentially with review loops
7. **Split** — Break large files (>300 lines) for better AI context efficiency
8. **Benchmark** — Run `claude -p` on standardized tasks, compare before/after
9. **Report** — Full comparison: speed, cost, turns, accuracy

## Results (Real Benchmark)

Tested on a React Native monorepo (80K+ lines, 32 features):

```
+====================================================================+
|  Metric                | main           | refactored     | delta    |
|------------------------|----------------|----------------|----------|
|  Convention Score      | 8/8 (100%)     | 8/8 (100%)     | =        |
|  Duration              | 380s           | 261s           | -31%     |
|  Total Cost            | $1.91          | $1.33          | -30%     |
|  Turns                 | 58             | 33             | -43%     |
|  Always-loaded context | 253 lines      | 312 lines      | +23%     |
|  On-demand rules       | 0              | 311 lines      | +311     |
|  Agent definitions     | 1              | 4              | +3       |
+====================================================================+
```

**31% faster, 30% cheaper, 43% fewer turns** — same accuracy.

## Install

### As a Claude Code plugin (recommended)

```bash
claude plugin add hwani6736/refactor-agent-friendly
```

This installs everything: the main skill, dev-pipeline skill, and 3 agents (architect, builder, reviewer).

### Manual install

```bash
git clone https://github.com/hwani6736/refactor-agent-friendly.git /tmp/raf

# Copy skills
cp -r /tmp/raf/skills/refactor-agent-friendly .claude/skills/
cp -r /tmp/raf/skills/dev-pipeline .claude/skills/

# Copy agents
cp /tmp/raf/agents/*.md .claude/agents/
```

## Usage

```
/refactor-agent-friendly apps/my-app
/refactor-agent-friendly src
/refactor-agent-friendly .
```

The skill walks you through each phase interactively. It measures before making changes so you can see the exact improvement.

## What's Included

**Plugin provides (installed automatically):**
```
agents/
├── architect.md    # Read-only, opus, memory:project — designs plans
├── builder.md      # Write-enabled, sonnet — implements code
└── reviewer.md     # Read-only, opus, memory:project — reviews quality

skills/
├── refactor-agent-friendly/SKILL.md   # The main 9-phase workflow
└── dev-pipeline/SKILL.md              # architect → builder → reviewer orchestration
```

**The skill creates in your project:**
```
your-project/
├── .claude/
│   ├── rules/                    # Path-scoped convention rules (on-demand)
│   │   ├── feature-development.md
│   │   ├── screen-development.md
│   │   ├── api-patterns.md
│   │   └── ...
│   └── benchmark-report.md       # Before/after comparison
└── CLAUDE.md                     # Optimized (under 300 lines)
```

## Key Concepts

### Rules > CLAUDE.md

CLAUDE.md loads **every session**. Rules load **only when editing matching files**.

```
BEFORE: 400 lines loaded every session (wasteful)
AFTER:  312 lines always + 311 lines on-demand (efficient)
```

A 50-line screen rule loads 0 tokens when editing features. Full context when editing screens.

### Structured Handoffs

Each agent returns typed, token-budgeted output:

| Agent | Output | Budget |
|-------|--------|--------|
| Architect | Implementation plan (files, changes, patterns) | <500 tokens |
| Builder | Completion report (files changed, lint results) | <300 tokens |
| Reviewer | Verdict (APPROVE/REQUEST_CHANGES + file:line) | <200 tokens |

### Agent Memory

Architect and reviewer use `memory: project` — they accumulate codebase knowledge across sessions. The architect remembers past patterns. The reviewer remembers common mistakes. They get smarter over time.

## Supported Project Types

Works with any project, but optimized for:
- React / React Native apps
- TypeScript monorepos
- Feature-based architecture
- Projects using generated API clients (Orval, OpenAPI, etc.)

The skill adapts its rules and agent prompts to your project's conventions.

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- Git repository
- Existing CLAUDE.md (or the skill creates one)

## Research Behind This

Based on:
- [Official Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) — CLAUDE.md under 300 lines, path-scoped rules
- [Create Custom Subagents](https://code.claude.com/docs/en/sub-agents) — memory, effort, structured outputs
- [Context Engineering for Coding Agents (Martin Fowler)](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) — signature-only retrieval, tree-sitter indexing
- [Awesome Claude Code Subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) — architect/reviewer patterns

## License

MIT
