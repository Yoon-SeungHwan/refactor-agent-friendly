---
name: refactor-agent-friendly
description: Analyze and refactor a codebase for AI agent efficiency — rules, agents, pipeline, benchmark
argument-hint: [target app or directory path]
---

Refactor the target codebase for AI agent efficiency. Reduce context cost,
improve convention compliance, and set up autonomous development infrastructure.

Target: $ARGUMENTS

## Phase 1: Measure Baseline

Before making changes, measure the current state.

### 1A. Context Cost

```bash
APP_DIR="${ARGUMENTS:-apps/ooc-app}"
echo "=== CLAUDE.md files ==="
find . -name "CLAUDE.md" -not -path "*/node_modules/*" -exec wc -l {} +
echo "=== .claude/rules/ ==="
find "$APP_DIR/.claude/rules" -name '*.md' -exec wc -l {} + 2>/dev/null || echo "0 rules"
echo "=== .claude/agents/ ==="
find .claude/agents -name '*.md' -exec wc -l {} + 2>/dev/null || echo "0 agents"
```

### 1B. Code Modularity

```bash
echo "=== Files >300 lines ==="
find "$APP_DIR/src" \( -name '*.ts' -o -name '*.tsx' \) -exec wc -l {} + 2>/dev/null | awk '$1 > 300 && !/total$/ {count++; print $1, $2} END {print count " files total"}'
echo "=== export * violations ==="
grep -r 'export \*' "$APP_DIR/src"/**/index.ts 2>/dev/null | wc -l
```

### 1C. Convention Violations

```bash
echo "=== Inline styles ==="
grep -rn 'style={{' "$APP_DIR/src" --include='*.tsx' 2>/dev/null | wc -l
echo "=== Manual memoization ==="
grep -rn 'useCallback\|useMemo' "$APP_DIR/src" --include='*.ts' --include='*.tsx' 2>/dev/null | wc -l
echo "=== Manual query keys ==="
grep -rn "queryKey: \['" "$APP_DIR/src" 2>/dev/null | wc -l
```

Save all numbers. These are the baseline for comparison.

## Phase 2: Analyze Gaps

Read the existing CLAUDE.md files and .claude/ configuration. Identify:

### Context Efficiency Gaps
1. **CLAUDE.md duplication** — Do root and app CLAUDE.md overlap? Measure shared lines.
2. **Missing .claude/rules/** — Are there domain-specific patterns that load every session instead of on-demand?
3. **Missing agents** — Is there only quick-search? No role-based agents?
4. **Missing pipeline** — No orchestrated architect→builder→reviewer workflow?

### Code Structure Gaps
1. **Large files (>300 lines)** — Which files are too large for efficient AI context?
2. **Barrel export quality** — Any `export *`? Missing index.ts files?
3. **Convention violations** — Inline styles, manual memoization, manual query keys?

### Key Principles (from research)
- **CLAUDE.md under 300 lines** — "Would removing this line cause Claude to make mistakes? If not, cut it."
- **Rules over CLAUDE.md** — Domain-specific patterns should load on-demand via .claude/rules/
- **Subagent memory:project** — Architect and reviewer accumulate knowledge across sessions
- **Structured handoffs** — Each agent returns typed, token-budgeted output (<500/<300/<200 tokens)
- **Subagents can't spawn subagents** — Pipeline orchestration must happen at main thread

Present findings to user via AskUserQuestion before proceeding.

## Phase 3: Create Rules

Create `.claude/rules/` files for each major domain. General template:

```markdown
---
globs:
  - "src/path/to/domain/**/*"
description: One-line description of when this rule applies
---

# Domain Rules

## Key patterns
- Pattern 1 with example
- Pattern 2 with example

## Do / Don't
- Do X
- Don't Y

See also: other-rule for related patterns.
```

### Rules to consider (adapt per project):
- **feature-development.md** — globs: src/features/**/* — naming, barrel exports, imports
- **screen-development.md** — globs: src/screens/**/* — screen checklist, structure, error boundaries
- **navigation.md** — globs: src/app/navigation/**/* — registration, deep links, types
- **api-patterns.md** — globs: src/**/api/**/* — API client patterns, generated hooks
- **auth-flow.md** — globs: src/**/auth/**/* — auth lifecycle, token management
- **state-management.md** — globs: src/**/store/**/* — Zustand patterns, selectors

### Rules guidelines:
- Keep each under 50 lines (rules compete with code for context)
- Use cross-references ("See also: X rule") to avoid duplication
- Glob patterns should be specific enough to avoid false matches
- Include do/don't examples with code snippets

## Phase 4: Optimize CLAUDE.md

### Move domain content to rules
Any CLAUDE.md section that applies only to specific files should become a rule:
- API/Query patterns → api-patterns rule
- Auth flow → auth-flow rule  
- Socket/realtime → socket-patterns rule
- Analytics → analytics rule

### Keep in CLAUDE.md (always-loaded)
- Project structure overview
- Quick reference table (most common operations)
- Import/naming conventions (apply everywhere)
- AI Development Checklist
- Compaction Rules

### Add to CLAUDE.md
```markdown
## Quick Reference
| Task | Key files | Rule |
|------|----------|------|
| Add screen | types.ts → constants.ts → registry.ts | screen-development |
| Add feature | features/{domain}/ + index.ts | feature-development |
...

## AI Development Checklist
### Adding a New Feature
1. Create directory structure
2. Implement following naming conventions
3. Create barrel export
4. Register in navigation (if screen)
5. Run lint + typecheck
6. Verify with reviewer agent

## Compaction Rules
When compacting, always preserve:
- List of modified files with paths
- Current implementation plan (if in pipeline)
- Lint/typecheck results
- Reviewer feedback (if in review loop)
```

### Target: combined always-loaded context under 300 lines

## Phase 5: Create Agents

Create 3 agents in `.claude/agents/`:

### architect.md
```yaml
---
name: architect
description: Analyze requirements and design implementation plans (read-only)
tools: Read, Glob, Grep, Bash
disallowedTools: Write, Edit
model: opus
effort: high
memory: project
maxTurns: 30
---
```
- Read-only analysis
- Output structured plan (<500 tokens)
- memory:project accumulates codebase knowledge

### builder.md
```yaml
---
name: builder
description: Implement features following plans. Writes code, runs lint and typecheck.
tools: Read, Edit, Write, Bash, Glob, Grep
model: sonnet
effort: medium
maxTurns: 50
---
```
- Follows architect plan exactly
- Runs verification after changes
- Output completion report (<300 tokens)

### reviewer.md
```yaml
---
name: reviewer
description: Review implementation for quality and pattern compliance (read-only)
tools: Read, Glob, Grep, Bash
disallowedTools: Write, Edit
model: opus
effort: high
memory: project
maxTurns: 20
---
```
- Checks diff against CLAUDE.md + rules
- Returns APPROVE or REQUEST_CHANGES (<200 tokens)
- memory:project accumulates common mistake patterns

## Phase 6: Create Pipeline Skill

Create `.claude/skills/dev-pipeline/SKILL.md` that orchestrates:
1. Spawn architect → wait for plan
2. Present plan to user
3. Record pre-build SHA
4. Spawn builder with plan → wait for completion
5. Spawn reviewer with explicit diff range → wait for verdict
6. If APPROVE → done
7. If REQUEST_CHANGES → fresh builder with feedback → re-review (max 3 loops)

Key context management:
- Each agent runs in its own context window
- Structured handoff outputs only (plan/completion/verdict)
- Fresh builder per review loop (avoid accumulated confusion)
- maxTurns prevents runaway exploration

## Phase 7: Split Large Files

For each file >300 lines, identify extractable concerns:
- **Hooks >300L** → Extract sub-hooks (e.g., emit actions, event handlers)
- **Components >300L** → Extract sub-components or custom hooks for logic
- **Screens >300L** → Extract header, content, footer sections

Skip files that are 300-330 lines (near boundary, not worth the churn).
Skip test files and story files (they're naturally long).

## Phase 8: Benchmark

### Run automated benchmark
Create `run-benchmark.sh` that:
1. Runs `claude -p` with a standardized task on each branch
2. Auto-scores convention compliance (8-10 checks)
3. Captures: duration, cost, turns, token usage
4. Outputs JSON for comparison

### Key metrics to compare

| Metric | What it measures | How |
|--------|-----------------|-----|
| Duration | Speed | Seconds to complete task |
| Cost | Token efficiency | $ from claude -p output |
| Turns | Exploration needed | Number of agent turns |
| Convention score | Accuracy | Auto-checked patterns |
| Always-loaded lines | Context cost | wc -l CLAUDE.md files |
| On-demand rule lines | Smart context | wc -l .claude/rules/ |

### Expected improvements
- 30-40% faster (less exploration)
- 30% cheaper (fewer tokens)
- 40% fewer turns (rules provide answers directly)
- Same or better accuracy

## Phase 9: Report

Produce a final benchmark report comparing:
- main branch (baseline) vs refactored branch (after)
- Context cost breakdown (always-loaded vs on-demand)
- Convention compliance scores
- Speed/cost/turns comparison
- What's delivered (rules, agents, pipeline, file splits)
- What's deferred (with rationale)

## When NOT to Use
- Project has <5 source files (too small for rules/agents)
- No recurring AI development tasks (one-off projects)
- Project already has comprehensive .claude/ setup
