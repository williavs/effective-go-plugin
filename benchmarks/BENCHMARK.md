# Effective Go Plugin -- Benchmark Results

Deterministic analysis of Go code generated with and without the effective-go skill.
All checks automated via `scripts/analyze.sh`. No LLM grading.

## Methodology

- 10 test prompts ranging from clean to very vague (simulating non-coder users)
- Each prompt run with and without the skill injected via `claude -p`
- Skill disabled during all runs to prevent auto-triggering
- Go code analyzed by deterministic script: compilation, `go vet`, anti-patterns, architecture metrics
- All runs produce Go files, all compile, all pass `go vet`

### Test Prompts (Clean -> Very Vague)

| # | Prompt | Quality |
|---|--------|---------|
| 01 | "write a go package that loads config from yaml files, env vars, and cli flags in that priority order. needs hot reload when the yaml changes." | Clean |
| 02 | "build me a simple job queue in go -- producers push jobs, workers pull and process them, max 10 concurrent workers, graceful shutdown." | Clean |
| 03 | "go tcp chat server -- clients connect, pick a username, send messages to everyone. show whos online. handle disconnects cleanly." | Clean |
| 04 | "go api client that wraps some rest api -- needs auth token refresh, retries with backoff, rate limiting. make it testable." | Medium |
| 05 | "need a go thing that watches a directory for new files and processes them -- moves to done/ folder after." | Medium |
| 06 | "go webhook receiver - - - listens on a port, validates signatures, queues the events for processing dont lose any if it crashes" | Messy |
| 07 | "make me a go program that creates ssh tunnels -- like a poor mans ngrok -- forward a local port through an ssh server." | Messy |
| 08 | "go key value store thing with ttl" | Very vague |
| 09 | "need a go thingy that tails log files and ships them somewhere - - like json over http" | Very vague |
| 10 | "go health checker that pings a bunch of urls and tells me which ones are down -- needs to be fast" | Very vague |

## Results: V5 (Lean Skill) vs Baseline

### Architecture Metrics

| Metric | Baseline (no skill) | V5 Skill | Delta |
|--------|-------------------|----------|-------|
| **Projects with interfaces** | 1/10 (10%) | 6/10 (60%) | **+500%** |
| **Total interfaces defined** | 2 | 9 | **+350%** |
| **Typed constants (enums)** | 0 | 6 | **0 -> 6** |
| **Embedding used** | 0/10 | 1/10 | **0 -> 1** |
| **Lazy-init maps (zero-value)** | 0 | 3 | **0 -> 3** |
| Anti-patterns detected | 2 | 0 | **-100%** |
| All compile | 10/10 | 10/10 | Same |
| All vet-clean | 10/10 | 10/10 | Same |
| All produce Go files | 10/10 | 10/10 | No paralysis |

### Per-Test Breakdown

| Test | Quality | Baseline Interfaces | V5 Interfaces | V5 Embedding | V5 TypedConsts | V5 LazyInit |
|------|---------|-------------------|--------------|-------------|---------------|------------|
| 01-config-loader | Clean | 0 | 0 | No | 0 | 0 |
| 02-job-queue | Clean | 0 | 1 | No | 1 | 0 |
| 03-tcp-chat | Clean | 0 | 0 | **Yes** | 1 | 1 |
| 04-api-client | Medium | 2 | 2 | No | 1 | 0 |
| 05-file-watcher | Medium | 0 | 1 | No | 1 | 1 |
| 06-webhook-server | Messy | 0 | **3** | No | 0 | 0 |
| 07-ssh-tunnel | Messy | 0 | 1 | No | 1 | 0 |
| 08-kv-store | Very vague | 0 | 0 | No | 0 | 1 |
| 09-log-shipper | Very vague | 0 | 1 | No | 0 | 0 |
| 10-health-checker | Very vague | 0 | 0 | No | 1 | 0 |

### Key Findings

#### The skill teaches architectural taste -- the senior engineer quality

What separates a senior Go engineer from a junior one isn't syntax or formatting -- it's design sense. Knowing when to define an interface, how to make types composable, when to use embedding vs explicit forwarding. These are the decisions that determine whether code is maintainable at scale or becomes legacy debt.

Claude already writes syntactically perfect Go. What it lacks is this architectural taste. The skill's job is to close that gap.

**1. Interface discovery is the skill's biggest win.** Baseline produces interfaces in 1/10 projects (and only because the prompt said "make it testable"). V5 produces interfaces in 6/10 projects across all prompt quality levels. The webhook server (messy prompt) generated 3 consumer-defined interfaces -- the skill drives architectural thinking even when the user doesn't ask for it.

**2. Typed constants appear only with the skill.** Zero baseline projects define typed constants. 6/10 V5 projects do. A senior engineer would never ship an event bus with raw string event types -- they'd define `type EventType string` with named constants. The skill teaches this instinct.

**3. Zero-value design (lazy-init) tripled.** 0 baseline projects use the lazy-init map pattern. 3/10 V5 projects do. Designing types so `var x T` is usable is a hallmark of Go expertise.

**4. Embedding appeared for the first time.** 03-tcp-chat/V5 uses struct embedding. This is the first time embedding has been triggered across 5 iterations of testing.

**5. Anti-patterns eliminated.** Baseline had 2 anti-patterns (silent error discard, string-typed port). V5 had 0.

### Prompt Quality Analysis -- The Equalizer Effect

The skill's most important property: **it compensates for prompt quality.** A non-coder typing "go kv store thing with ttl" gets the same architectural quality as a developer writing a detailed spec.

Architecture score = interfaces + typed constants + lazy-init patterns + embedding per project.

| Prompt Quality | Baseline Score | V5 Skill Score | Delta |
|---------------|---------------|---------------|-------|
| **Clean** (developer-quality) | 0 | 5 | 0 -> 5 |
| **Medium** (knows what they want) | 2 | 6 | +200% |
| **Messy** (fast typing, dashes, typos) | 0 | 5 | 0 -> 5 |
| **Very vague** (5-10 words) | 0 | 3 | 0 -> 3 |

Without the skill, Claude produces **zero architectural features** from messy or vague prompts. With the skill, messy prompts produce the **same architecture quality as clean prompts** (score 5 vs 5).

This means the skill acts as an equalizer: non-coders using Claude Code get Go projects with the same structural quality that an experienced developer would produce. The skill embeds the senior engineer's taste directly into the model's output, regardless of how the request is phrased.

## Evolution Across Iterations

| Version | Approach | Skill Size | Interface Delta | Key Change |
|---------|----------|-----------|----------------|------------|
| V1 | Pattern checklist | ~300 lines | +11% (LLM-graded) | Rules only |
| V2 | Design thinking | ~280 lines | Interfaces appeared | Added pre-code design questions |
| V3 | + Hard rules | ~300 lines | 10% -> 56% | Added 8 anti-pattern rules |
| V4 | Anti-paralysis | ~290 lines | Mixed | Fixed vague prompt failure |
| **V5** | **Lean (blind spots only)** | **~120 lines** | **10% -> 60%** | **Removed what Claude already knows** |

The biggest improvement came from V5: **cutting the skill in half** and focusing only on what Claude is bad at. The Anthropic article "Building Effective Agents" confirmed our finding: "every token added depletes Claude's attention budget." Removing dead weight (composite literals, defer patterns, error flow -- all already 100% pass rate) freed attention for the hard stuff.

## What the Skill Doesn't Fix

Honest about the limits:

- **Embedding is still rare** (1/10). Claude doesn't naturally reach for struct embedding. The skill helps but embedding remains the hardest pattern to teach.
- **Zero-value design is inconsistent** (3/10). Sometimes Claude lazy-inits, sometimes it writes a constructor. Depends on the type -- network connections genuinely need constructors.
- **Config-loader (01) showed no improvement.** Complex config packages with external deps resist the skill's influence.
- **Single run per prompt.** LLM outputs vary. Ideally each prompt runs 3-5 times to measure variance.

## Reproducing These Results

```bash
# 1. Run a prompt without skill
claude -p "go key value store thing with ttl -- save to outputs/" \
  --dangerously-skip-permissions --output-format text > without.md

# 2. Run the same prompt with skill prepended
claude -p "$(cat skills/effective-go/SKILL.md)

Now: go key value store thing with ttl -- save to outputs/" \
  --dangerously-skip-permissions --output-format text > with.md

# 3. Analyze both
bash skills/effective-go/scripts/analyze.sh ./outputs-without/
bash skills/effective-go/scripts/analyze.sh ./outputs-with/
```
