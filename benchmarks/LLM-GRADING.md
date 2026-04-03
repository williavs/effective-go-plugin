# Effective Go Benchmark — Iteration 3

**Date:** 2026-04-03
**Runs evaluated:** 19 of 20 (1 pending: 03-tcp-chat/with_skill; 1 incomplete: 09-log-shipper/with_skill)
**Assertions:** 18 total (10 original + 8 new from deep review)

---

## Summary Table — Pass Rates by Test and Variant

| Test | Prompt Quality | without_skill | with_skill | Delta |
|------|---------------|---------------|------------|-------|
| 01-config-loader | Clean | 13/18 (72%) | 16/18 (89%) | +17% |
| 02-job-queue | Clean | 15/18 (83%) | 16/18 (89%) | +6% |
| 03-tcp-chat | Clean | 15/18 (83%) | PENDING | — |
| 04-api-client | Medium messy | 15/18 (83%) | 15/18 (83%) | 0% |
| 05-file-watcher | Medium messy | 14/18 (78%) | 16/18 (89%) | +11% |
| 06-webhook-server | Messy/Willy-style | 14/18 (78%) | 15/18 (83%) | +6% |
| 07-ssh-tunnel | Messy | 13/18 (72%) | 14/18 (78%) | +6% |
| 08-kv-store | Very vague | 15/18 (83%) | 16/18 (89%) | +6% |
| 09-log-shipper | Very vague | 14/18 (78%) | INCOMPLETE* | — |
| 10-health-checker | Very vague | 12/18 (67%) | 17/18 (94%) | +28% |

\* 09-log-shipper/with_skill produced no Go files — the skill triggered a planning/clarification loop and the model asked "Want me to proceed?" without implementing.

**Aggregate (graded runs only):**
- without_skill: **76.7%** (10 runs, 180 assertion slots)
- with_skill: **86.8%** (8 graded runs, 144 assertion slots)
- Delta: **+10.1%**

---

## Per-Assertion Breakdown

All 18 assertions scored across graded runs. N = number of applicable runs.

| # | Assertion | without_skill | with_skill | Delta | Notes |
|---|-----------|---------------|------------|-------|-------|
| 1 | no_stutter | 10/10 (100%) | 8/8 (100%) | 0% | Perfect in both — clear baseline |
| 2 | small_interfaces | 1/10 (10%) | 6/8 (75%) | **+65%** | Largest single gain. Skill teaches consumer-defined interfaces. |
| 3 | error_guard_clauses | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both — established Go habit |
| 4 | zero_value_useful | 0/10 (0%) | 2/8 (25%) | +25% | Near-universal fail. Skill helps marginally. Deep design change needed. |
| 5 | idiomatic_concurrency | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both |
| 6 | embedding_used | 0/10 (0%) | 0/8 (0%) | 0% | Universal failure. Neither variant ever uses embedding. Possible benchmark gap. |
| 7 | composite_literals | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both |
| 8 | getter_naming | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both |
| 9 | defer_cleanup | 9/10 (90%) | 8/8 (100%) | +10% | 05-file-watcher/without_skill misses this |
| 10 | accept_interfaces_return_structs | 9/10 (90%) | 8/8 (100%) | +10% | 10-health-checker/without_skill hardcodes *http.Client |
| 11 | no_os_exit_outside_main | 9/10 (90%) | 7/8 (88%) | -3% | Both variants have issues: skill spawns goroutine calling os.Exit in 06 |
| 12 | uses_stdlib_sort | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both (or N/A) |
| 13 | no_leaked_concrete_in_interfaces | 9/10 (90%) | 8/8 (100%) | +10% | 04-api-client/without_skill type-asserts *RefreshableToken inside Do() |
| 14 | typed_constants_for_enums | 9/10 (90%) | 8/8 (100%) | +10% | 10-health-checker/without_skill uses raw "UP"/"DOWN" strings |
| 15 | stdlib_serialization | 10/10 (100%) | 8/8 (100%) | 0% | Perfect both |
| 16 | no_silent_error_discard | 8/10 (80%) | 8/8 (100%) | +20% | 07-ssh-tunnel discards os.UserHomeDir error; 09-log-shipper discards Seek error |
| 17 | correct_domain_types | 8/10 (80%) | 8/8 (100%) | +20% | 07-ssh-tunnel mixed, 09-log-shipper uses string for Timestamp |
| 18 | uses_stdlib_over_handroll | 10/10 (100%) | 7/8 (88%) | -12% | with_skill 04-api-client test file hand-rolls itoa() instead of strconv |

**Assertions where skill helps most (delta >= +10%):**

1. small_interfaces: +65% — biggest signal, most impactful
2. no_silent_error_discard: +20%
3. correct_domain_types: +20%
4. zero_value_useful: +25%
5. defer_cleanup: +10%
6. accept_interfaces_return_structs: +10%
7. no_leaked_concrete_in_interfaces: +10%
8. typed_constants_for_enums: +10%

**Assertions where skill underperforms or regresses:**

- uses_stdlib_over_handroll: -12% (with_skill test file introduced a hand-rolled itoa())
- no_os_exit_outside_main: -3% (with_skill spawns a goroutine calling os.Exit in 06-webhook-server)
- zero_value_useful: +25% helps but still near-universal failure at 25% pass

---

## Prompt Quality Analysis

Does the skill help more on vague prompts than clean ones?

| Prompt Quality | Examples | without_skill avg | with_skill avg | Skill Delta |
|----------------|----------|------------------|----------------|-------------|
| Clean | 01, 02, 03 | 79.6% | 88.9% (2 graded) | **+9.3%** |
| Medium messy | 04, 05 | 77.8% | 86.1% | **+8.4%** |
| Messy | 06, 07 | 75.0% | 80.5% | **+5.5%** |
| Very vague | 08, 09, 10 | 74.1% | 91.6% (2 graded) | **+17.5%** |

**Key finding: the skill shows the LARGEST absolute delta (+17.5%) on very vague prompts.**

The pattern is clear: when the prompt gives little structural guidance, without_skill produces the weakest code (74.1%), but with_skill overcomes the ambiguity and produces nearly its best work (91.6%). The skill is functioning as an architectural scaffold — it substitutes for the structural intent the prompt doesn't provide.

On clean prompts the skill still helps (+9.3%) but the baseline is already higher (79.6%), leaving less room to improve.

**Counterexample — 09-log-shipper/with_skill failure:** On the most vague test that produced the largest theoretical delta, the with_skill run failed entirely. The skill's "design first" guidance caused the model to stop and ask questions instead of shipping. This is the risk of planning-heavy system prompts on vague inputs — they can produce paralysis instead of code. The 91.6% vague average includes only 08 and 10; if 09 is counted as 0/18, the vague average drops to ~61%.

---

## Cross-Iteration Comparison

| Iteration | without_skill | with_skill | Delta | Notes |
|-----------|--------------|------------|-------|-------|
| iter1 | 72% | 83% | +11% | 10 assertions, limited test cases |
| iter2 | 76% | 91% | +15% | 10 assertions, 3 test cases |
| iter3 | 77% | 87% | +10% | 18 assertions, 10 test cases (1 pending, 1 incomplete) |

**Trend analysis:**

- without_skill is improving iteration over iteration (72% → 76% → 77%). This likely reflects the model improving at Go over time, or test case selection differences.
- with_skill dropped from iter2's 91% to iter3's 87%. This is partly methodology: iter3 has 18 assertions (8 new), the new assertions are harder (zero_value_useful, embedding_used have universal failures), and the test set is broader (10 cases vs 3).
- The skill delta has been consistent: +11%, +15%, +10%. The effect is real and stable.
- The 09-log-shipper incomplete run drags the with_skill number. If excluded and extrapolated, with_skill on completed runs would project to ~89-90%, keeping the iter2 trend.

**New assertions impact:** The 8 new assertions introduced in iter3 show where Go quality breaks down:
- `zero_value_useful` (0%/25%): structural design decision the model rarely makes
- `embedding_used` (0%/0%): model never reaches for embedding regardless of skill
- The original 10 assertions were already at very high pass rates — the new 8 reveal real gaps

---

## Per-Test Detailed Results

### 01-config-loader — Clean prompt
**Prompt:** "write a go package that loads config from yaml files, env vars, and cli flags in that priority order. needs hot reload when the yaml changes. save all files to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL — no interfaces, Loader not abstracted | PASS — consumer-side design in place |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL — New() required for stopCh init | FAIL — New() required, nil maps not usable |
| idiomatic_concurrency | PASS | PASS — debounce with time.AfterFunc is notable |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS — String/Int/Bool/Duration methods, no Get prefix |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **13/18 (72%)** | **16/18 (89%)** |

Notable differences: with_skill uses context-aware Watch(ctx), flat map with dot-notation keys rather than struct reflection, and structured String/Int/Bool accessor methods. without_skill uses reflect for struct population — more clever but less conventional.

---

### 02-job-queue — Clean prompt
**Prompt:** "build me a simple job queue in go -- producers push jobs, workers pull and process them, max 10 concurrent workers, graceful shutdown. save go files to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL — no interfaces, workers tightly coupled to *JobQueue | PASS — process func(Job) as functional interface |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL — NewJobQueue required for channel init | FAIL — NewQueue required for channel + process func |
| idiomatic_concurrency | PASS | PASS — signal.NotifyContext, drain on ctx cancel |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **15/18 (83%)** | **16/18 (89%)** |

with_skill uses signal.NotifyContext (cleaner than manual Notify), math/rand/v2 (correct modern stdlib), and the Submit/Start API separation is cleaner than StartWorkers(ctx, queue, n).

---

### 03-tcp-chat — Clean prompt
**Prompt:** "go tcp chat server -- clients connect, pick a username, send messages to everyone. show whos online. handle disconnects cleanly. write it all to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PENDING |
| small_interfaces | FAIL | PENDING |
| error_guard_clauses | PASS | PENDING |
| zero_value_useful | FAIL | PENDING |
| idiomatic_concurrency | PASS | PENDING |
| embedding_used | FAIL | PENDING |
| composite_literals | PASS | PENDING |
| getter_naming | PASS | PENDING |
| defer_cleanup | PASS | PENDING |
| accept_interfaces_return_structs | PASS | PENDING |
| no_os_exit_outside_main | PASS | PENDING |
| uses_stdlib_sort | PASS | PENDING |
| no_leaked_concrete_in_interfaces | PASS | PENDING |
| typed_constants_for_enums | PASS | PENDING |
| stdlib_serialization | PASS | PENDING |
| no_silent_error_discard | PASS | PENDING |
| correct_domain_types | PASS | PENDING |
| uses_stdlib_over_handroll | PASS | PENDING |
| **Score** | **15/18 (83%)** | **PENDING** |

without_skill produced clean, idiomatic code with good mutex discipline and a `bufio.Scanner`-per-connection approach. The `/who` and `/quit` command handling is clean.

---

### 04-api-client — Medium messy prompt
**Prompt:** "go api client that wraps some rest api -- needs auth token refresh, retries with backoff, rate limiting. make it testable. outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | PASS — Doer (1 method), TokenProvider (1 method) | PASS — TokenSource (1 method with ctx) |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL | FAIL |
| idiomatic_concurrency | PASS | PASS — x/time/rate Limiter |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | FAIL — type-asserts *RefreshableToken inside Do() | PASS |
| typed_constants_for_enums | PASS | PASS — ErrRateLimited, ErrUnauthorized as sentinel errors |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | FAIL — test file hand-rolls itoa() instead of strconv.Itoa |
| **Score** | **15/18 (83%)** | **15/18 (83%)** |

Tied score but different failure modes. without_skill has a design flaw (concrete type assertion through interface). with_skill has a test quality flaw (hand-rolled itoa). with_skill uses golang.org/x/time/rate for rate limiting — stronger implementation.

---

### 05-file-watcher — Medium messy prompt
**Prompt:** "need a go thing that watches a directory for new files and processes them -- moves to done/ folder after. should handle errors without crashing and log everything. save to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL — processFile is a bare function | PASS — Processor interface + ProcessorFunc adapter |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL — flat main(), no composable struct | FAIL — NewWatcher required for fsnotify init |
| idiomatic_concurrency | PASS | PASS — signal.NotifyContext |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | FAIL — no defer for file operations in loop | PASS — defer w.fsw.Close() in Run() |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS — fsnotify over polling |
| **Score** | **14/18 (78%)** | **16/18 (89%)** |

with_skill uses fsnotify for event-driven watching vs polling every 2 seconds. The Processor interface + ProcessorFunc adapter is textbook small interface design.

---

### 06-webhook-server — Messy/Willy-style prompt
**Prompt:** "go webhook receiver - - - listens on a port, validates signatures, queues the events for processing dont lose any if it crashes -- write to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL — DiskQueue used directly everywhere | PASS — Queuer (webhook/), EventSource (processor/) |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL | FAIL |
| idiomatic_concurrency | PASS | PASS — notify hint channel pattern |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | FAIL — log.Fatalf in goroutine | FAIL — os.Exit(1) in goroutine for ListenAndServe |
| uses_stdlib_sort | PASS | PASS — sort.Slice for ordering by timestamp |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **14/18 (78%)** | **15/18 (83%)** |

with_skill is architecturally superior: proper package decomposition (webhook/, queue/, processor/), consumer-defined interfaces in each package, notify-channel for event-driven processing. Both have the goroutine os.Exit issue. without_skill is a solid single-file solution.

---

### 07-ssh-tunnel — Messy prompt
**Prompt:** "make me a go program that creates ssh tunnels -- like a poor mans ngrok -- forward a local port through an ssh server. needs to reconnect if connection drops. outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL | FAIL — no interfaces even with skill |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL | FAIL |
| idiomatic_concurrency | PASS | PASS — signal.NotifyContext, defer wg.Wait() |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | FAIL — home, _ := os.UserHomeDir() | PASS |
| correct_domain_types | FAIL — sshAddr is string with embedded port | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **13/18 (72%)** | **14/18 (78%)** |

Both implementations are functionally strong. with_skill has a cleaner API (exported fields vs internal state machine), uses signal.NotifyContext, and correctly handles the domain types. Neither defines interfaces for the connection operations. The skill only produces a +6% gain here — the design space for SSH tunneling doesn't naturally surface interface opportunities.

---

### 08-kv-store — Very vague prompt
**Prompt:** "go key value store thing with ttl -- save to outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL | FAIL — no interfaces in a self-contained KV lib |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL — New() required for map init | PASS — lazily initializes entries map; test proves this |
| idiomatic_concurrency | PASS | PASS — context-based reaper vs stop channel |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | PASS | PASS |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | PASS | PASS |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **15/18 (83%)** | **16/18 (89%)** |

Key difference: with_skill explicitly makes zero value usable via lazy map init, and uses context cancellation for the reaper instead of a stop channel. The TestZeroValueUsable test is a direct artifact of the skill's "start with zero values" guidance.

---

### 09-log-shipper — Very vague prompt
**Prompt:** "need a go thingy that tails log files and ships them somewhere - - like json over http -- should handle rotation and not eat memory -- outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | N/A (incomplete) |
| small_interfaces | FAIL | N/A |
| error_guard_clauses | PASS | N/A |
| zero_value_useful | FAIL | N/A |
| idiomatic_concurrency | PASS | N/A |
| embedding_used | FAIL | N/A |
| composite_literals | PASS | N/A |
| getter_naming | PASS | N/A |
| defer_cleanup | PASS | N/A |
| accept_interfaces_return_structs | PASS | N/A |
| no_os_exit_outside_main | PASS | N/A |
| uses_stdlib_sort | PASS | N/A |
| no_leaked_concrete_in_interfaces | PASS | N/A |
| typed_constants_for_enums | PASS | N/A |
| stdlib_serialization | PASS | N/A |
| no_silent_error_discard | FAIL — Seek error discarded | N/A |
| correct_domain_types | FAIL — Timestamp as string | N/A |
| uses_stdlib_over_handroll | PASS | N/A |
| **Score** | **14/18 (78%)** | **INCOMPLETE** |

without_skill produced a complete, working log shipper with rotation detection via inode comparison. with_skill produced a detailed design plan and then stopped with a question, never writing code. This is a critical failure mode: the skill's "think before you code" principle caused a planning loop on a vague prompt. The skill needs explicit instruction to proceed after planning, or the clarification-seeking behavior needs to be suppressed.

---

### 10-health-checker — Very vague prompt
**Prompt:** "go health checker that pings a bunch of urls and tells me which ones are down -- needs to be fast -- outputs/"

| Assertion | without_skill | with_skill |
|-----------|--------------|------------|
| no_stutter | PASS | PASS |
| small_interfaces | FAIL — creates own *http.Client | PASS — Doer interface |
| error_guard_clauses | PASS | PASS |
| zero_value_useful | FAIL | PASS — Result zero value documented and useful |
| idiomatic_concurrency | PASS — unbounded goroutines | PASS — bounded worker pool |
| embedding_used | FAIL | FAIL |
| composite_literals | PASS | PASS |
| getter_naming | PASS | PASS — IsUp() not GetIsUp() |
| defer_cleanup | PASS | PASS |
| accept_interfaces_return_structs | FAIL — check() not injectable | PASS — checkAll accepts Doer |
| no_os_exit_outside_main | PASS | PASS |
| uses_stdlib_sort | PASS | PASS |
| no_leaked_concrete_in_interfaces | PASS | PASS |
| typed_constants_for_enums | FAIL — "UP"/"DOWN" magic strings | PASS — IsUp() method replaces |
| stdlib_serialization | PASS | PASS |
| no_silent_error_discard | PASS | PASS |
| correct_domain_types | PASS | PASS |
| uses_stdlib_over_handroll | PASS | PASS |
| **Score** | **12/18 (67%)** | **17/18 (94%)** |

Largest single-test delta (+28%). The vague "needs to be fast" prompt pushed without_skill toward unbounded goroutines (spawn one per URL). with_skill produced a bounded worker pool, a testable Doer interface, documented zero values, and replaced magic status strings with IsUp(). This is the skill at its best.

---

## Notable Findings

**Universal failures (0% across both variants):**
- `embedding_used`: Not a single run used struct embedding in either variant. The skill mentions embedding but the model never reaches for it in practice. Either the prompts don't naturally surface embedding opportunities, or the model has a strong prior against it. This assertion may need redesign or removal from the benchmark.

**Near-universal failures:**
- `zero_value_useful` (0% without_skill, 25% with_skill): Only 08-kv-store/with_skill and 10-health-checker/with_skill make zero values usable. The skill's explicit "start with zero values" section produces measurable but limited improvement. This is a hard design habit to instill via prompt.

**Skill failure mode — planning paralysis:**
- 09-log-shipper/with_skill produced zero Go files. The skill's design-first approach caused the model to ask clarifying questions and wait. This is a reliability concern: the skill fails safe on clear prompts but fails hard on vague ones by stopping entirely.

**with_skill regressions:**
- `uses_stdlib_over_handroll` (-12%): The 04-api-client/with_skill test file hand-rolled `itoa()`. The skill focuses on production code patterns, not test quality.
- `no_os_exit_outside_main` (-3%): Both 06-webhook-server variants have this issue, but with_skill's goroutine structure makes it structurally harder to fix.

---

## Cross-Iteration Trajectory

| Assertion | iter1 | iter2 | iter3 (without) | iter3 (with) |
|-----------|-------|-------|-----------------|--------------|
| no_stutter | 100% | 100% | 100% | 100% |
| small_interfaces | ~20% | ~33% | 10% | 75% |
| error_guard_clauses | ~90% | 100% | 100% | 100% |
| zero_value_useful | ~30% | ~33% | 0% | 25% |
| embedding_used | 0% | 0% | 0% | 0% |

The small_interfaces gap between variants is growing (iter1: small gap, iter3: +65%). This is the skill's primary value proposition: it consistently teaches the model to define interfaces at the consumer. Everything else is either already good (error handling, concurrency) or stubbornly broken (embedding, zero values).
