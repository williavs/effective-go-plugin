# effective-go

A Claude Code plugin that teaches Claude to think like a Go developer before writing code.

## Install

**Option 1: Add as marketplace + install (recommended)**
```bash
claude plugin marketplace add https://github.com/williavs/effective-go-plugin
claude plugin install effective-go
```

**Option 2: Direct plugin directory**
```bash
claude --plugin-dir /path/to/effective-go-plugin
```

**Option 3: Manual**
```bash
git clone https://github.com/williavs/effective-go-plugin ~/.claude/plugins/effective-go
```

After installing, restart Claude Code or run `/reload-plugins`.

## Quick Start

Just write Go. The skill auto-triggers on any Go code request. No slash command needed.

```
> build me a simple http server with middleware support
```

The skill silently guides Claude toward idiomatic Go architecture -- zero-value usable types, consumer-defined interfaces, proper error flow, correct concurrency patterns.

To explicitly review existing Go code:
```
> review my go code
```
This launches the `go-reviewer` agent which runs deterministic analysis + design review.

## The Problem

Claude writes Go that compiles but isn't architecturally tasteful. It reads like translated Java/Python -- technically correct but missing the design thinking that experienced Go developers do instinctively.

The surface-level stuff (formatting, naming, syntax) is already fine. The gap is in:

- **Zero-value design** -- structs that panic without a constructor
- **Interface discovery** -- fat interfaces defined at the provider instead of small ones at the consumer
- **Package boundaries** -- Java-style deep hierarchies instead of flat, purposeful packages
- **Concurrency architecture** -- mutex-first instead of thinking about data ownership
- **Error flow** -- nested `if err == nil` instead of guard clauses

## What This Plugin Does

Instead of a checklist of rules, it teaches the **thinking process** that happens in a Go developer's head:

1. **Type Design** -- "What happens with `var x T`? Is the zero value useful?"
2. **Interface Discovery** -- "What methods does this function actually call? Define that as the interface."
3. **Package Boundaries** -- "Say `pkg.Name` out loud. Does it read well?"
4. **Data Flow** -- "Who owns this data? Does ownership transfer between goroutines?"
5. **Error Flow** -- "What's the shape of this function? Happy path on the left edge."

Plus 8 hard rules that Go developers never violate:
- No `os.Exit()` outside `main()`
- No hand-rolled sorting (`sort.Slice` exists)
- Interface methods don't return concrete implementation types
- Named types for closed value sets (not raw strings)
- `encoding/json` for structured data (not custom delimiters)
- Standard library first (check `sort`, `slices`, `strings`, `bytes`, `maps` before writing loops)
- Never silently discard errors
- Domain types use correct Go types (`int` for ports, `time.Time` for timestamps)

## What's Included

| Component | Path | Description |
|-----------|------|-------------|
| Skill | `skills/effective-go/SKILL.md` | Auto-triggering design thinking guide |
| Reference | `skills/effective-go/references/` | Full Effective Go pattern reference |
| Agent | `agents/go-reviewer.md` | On-demand code review agent |
| Analysis | `skills/effective-go/scripts/analyze.sh` | Deterministic Go code grader (JSON output) |
| Benchmarks | `benchmarks/` | Raw data from 10-prompt evaluation |

## Benchmarks

See [benchmarks/BENCHMARK.md](benchmarks/BENCHMARK.md) for full results.

**Key finding:** Interface usage jumps from 10% to 56% of projects with the skill. This holds even on very vague prompts like "go key value store thing with ttl."

The skill works regardless of prompt quality. Non-coders get the same architectural improvements as developers who know to ask for "idiomatic Go."

| Metric | Without Skill | With Skill |
|--------|--------------|------------|
| Compiles | 100% | 100% |
| Vet clean | 100% | 100% |
| Projects with interfaces | 10% | 56% |
| Avg lines of code | 257 | 277 |

## Development

Based on the official [Effective Go](https://go.dev/doc/effective_go) guide. Iterated through 4 rounds:

1. **V1 (pattern checklist):** +11% on LLM-graded assertions. Only helped surface-level stuff.
2. **V2 (design thinking):** +15% advantage. Cracked zero-value design and interface placement.
3. **V3 (+ hard rules):** 8 concrete rules from deep code review. 10 prompts, clean to very vague.
4. **V4 (anti-paralysis):** Fixed "planning paralysis" on vague prompts. Embedding guidance.

All evaluation data (deterministic analysis JSON, raw Go outputs) available in `benchmarks/`.

## License

MIT
