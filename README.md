# effective-go

A Claude Code plugin that teaches Claude to think like a Go developer before writing code.

## The Problem

Claude writes Go that compiles and runs but isn't architecturally tasteful. It reads like translated Java or Python -- technically correct but missing the design thinking that experienced Go developers do before typing.

The surface-level stuff (formatting, naming, syntax) is already fine. The gap is in:

- **Zero-value design** -- structs that panic without a constructor
- **Interface discovery** -- fat interfaces defined at the provider instead of small ones at the consumer
- **Package boundaries** -- Java-style deep hierarchies instead of flat, purposeful packages
- **Concurrency architecture** -- mutex-first instead of thinking about data ownership
- **Error flow** -- nested `if err == nil` instead of guard clauses

## What This Plugin Does

Instead of giving Claude a checklist of rules, it teaches the **thinking process** that happens in a Go developer's head before they write code:

1. **Type Design** -- "What happens with `var x T`? Is the zero value useful?"
2. **Interface Discovery** -- "What methods does this function actually call? Define that as the interface."
3. **Package Boundaries** -- "Say `pkg.Name` out loud. Does it read well?"
4. **Data Flow** -- "Who owns this data? Does ownership transfer between goroutines?"
5. **Error Flow** -- "What's the shape of this function? Happy path on the left edge."

Plus 8 hard rules that Go developers never violate (no `os.Exit` outside main, no hand-rolled sorts, no leaked concrete types in interfaces, etc.).

## Installation

```bash
# Clone into your Claude Code plugins directory
cd ~/.claude/plugins
git clone https://github.com/willyvansickle/effective-go-plugin.git effective-go
```

Or install via the Claude Code plugin system.

## What's Included

### Skill: `effective-go`

Auto-triggers whenever you're writing Go code. Teaches design-first patterns from the Effective Go guide.

- `skills/effective-go/SKILL.md` -- Core design thinking process (~200 lines)
- `skills/effective-go/references/effective-go-patterns.md` -- Full pattern reference with edge cases

### Agent: `go-reviewer`

Invoke with "review my go code" or trigger proactively after writing Go. Runs deterministic analysis + design review.

### Analysis Script

`skills/effective-go/scripts/analyze.sh` -- Deterministic Go code grader. Checks:

- Compilation (`go build`)
- `go vet` cleanliness
- Anti-pattern detection (GetFoo getters, os.Exit outside main, silent error discards, string-typed ports, raw string enums)
- Code metrics (lines, files, packages, interfaces)

Outputs JSON. No dependencies beyond bash and the Go toolchain.

## Benchmarks

See [benchmarks/BENCHMARK.md](benchmarks/BENCHMARK.md) for full results.

**Key finding:** The skill's biggest measurable impact is on interface usage. Without the skill, 10% of projects define interfaces. With the skill, 56% do -- even from very vague prompts like "go key value store thing with ttl."

The skill works regardless of prompt quality. Non-coders get the same architectural improvements as developers who know to ask for "idiomatic Go."

## Development

Based on the official [Effective Go](https://go.dev/doc/effective_go) guide, iterated through 3 rounds of testing:

1. **V1 (pattern checklist):** +11% on LLM-graded assertions. Only helped surface-level stuff.
2. **V2 (design thinking):** +15% advantage, cracked zero-value design and interface placement for the first time.
3. **V3 (+ hard rules):** Added 8 concrete rules from deep code review. Tested across 10 prompts with quality gradient (clean to very vague).

## License

MIT
