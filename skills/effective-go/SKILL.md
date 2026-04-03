---
name: effective-go
description: Go design thinking and idiomatic architecture. Use this skill whenever writing Go code, designing Go packages, reviewing Go architecture, or helping with any Go project. Triggers on any Go code generation, Go project setup, package design, concurrency design, error handling, or interface design -- even when the user doesn't ask for idiomatic code. This skill teaches the THINKING PROCESS, not just patterns.
---

# Effective Go -- Design-First Development

Write the code first, then make it good. Don't overthink -- ship working Go, but ship it with taste.

The patterns below should be second nature, not a pre-flight checklist. Apply them as you write, not as a separate planning phase. If the request is simple, write simple code. If it's complex, the design emerges as you go.

## Design Instincts

These should be automatic when writing Go. Not a checklist to consult -- habits that shape every line.

### 1. Type Design -- Start With Zero Values

For every struct you're about to define, ask: what happens if someone writes `var x MyType` and starts calling methods? If the answer is "panic" or "nothing works," you have a design problem.

**The goal:** make the zero value useful whenever possible.

- Maps inside structs: lazy-initialize on first write, not in a constructor
- Slices: nil slice is fine, it behaves like empty
- Pointers to dependencies: nil means "use default" (like `http.Client` does)
- Mutexes: zero value is unlocked, ready to use

```go
// Design question: Does *Board work at zero value?
// Answer: No -- tasks is nil map, subscribers is nil map.
// Fix: lazy-init in methods, not in a mandatory constructor.

type Board struct {
    mu          sync.RWMutex
    tasks       map[string]*Task
    subscribers map[string]chan Event
}

func (b *Board) Add(t *Task) {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.tasks == nil {
        b.tasks = make(map[string]*Task)
    }
    b.tasks[t.ID] = t
    b.notify(Event{Type: "added", Task: t})
}
```

When a constructor IS required (external resource handles, validated config), name it `New` if the package has one primary type, or `NewThing` if multiple. Never `NewPkgThing` -- the package name is already in the import.

### 2. Interface Discovery -- Define at the Consumer

This is the most common Go design mistake: defining big interfaces at the provider, Java-style. In Go, interfaces are discovered at the point of use.

**The process:**
1. Write your function signatures first (even as mental pseudocode)
2. Look at what methods each function actually calls on its arguments
3. That's your interface -- define it locally at the consumer
4. The concrete type satisfies it implicitly

```go
// Wrong thinking: "I need a Storage interface for my database"
type Storage interface {  // 6 methods, defined in the storage package
    Get(id string) (*Item, error)
    Put(item *Item) error
    Delete(id string) error
    List() ([]*Item, error)
    Search(q string) ([]*Item, error)
    Close() error
}

// Right thinking: "This function needs to read items. What does it call?"
// In the handler package, where it's consumed:
type ItemGetter interface {
    Get(id string) (*Item, error)
}

func HandleGetItem(store ItemGetter, id string) (*Item, error) {
    return store.Get(id)
}
// Any concrete type with a Get method works. Testing is trivial.
```

**One-method interfaces** get the `-er` suffix: `Reader`, `Writer`, `Closer`, `Stringer`. Multi-method interfaces describe a role: `Handler`, `Conn`.

**Accept interfaces, return structs.** Functions should take the narrowest interface they need and return concrete types so callers get the full API.

### 3. Package Boundary Design -- Think Like the Caller

Before creating any file, imagine the import statement and every `pkg.Name` the caller will type.

**The test:** say the full qualified name out loud. Does it read well?

- `http.Client` -- yes
- `http.HTTPClient` -- no, stutter
- `ring.New()` -- yes, clear
- `ring.NewRing()` -- no, redundant
- `bufio.Reader` -- yes
- `bufio.BufReader` -- no, package already said "buf"

**Package naming rules:**
- Lowercase, single word, no underscores, no mixedCaps
- The package name IS the namespace -- use it
- `util`, `common`, `helpers` are design smells. Name for the domain: `auth`, `billing`, `render`

**Getter naming:** if the field is `owner`, the getter is `Owner()` (not `GetOwner()`). The setter is `SetOwner()`.

### 4. Data Flow Architecture -- Channels vs Mutexes

Before writing any concurrent code, map out the data flow:

**Who produces data? Who consumes it? Does ownership transfer?**

| Situation | Use |
|-----------|-----|
| Data ownership transfers between goroutines | Channel |
| Protecting shared state (counter, map, cache) | `sync.Mutex` or `sync.RWMutex` |
| Signaling completion or cancellation | Channel, `context.Context`, or `sync.WaitGroup` |
| One-time initialization | `sync.Once` |
| Bounding concurrency (max N goroutines active) | Buffered channel as semaphore |
| Fixed worker pool processing a queue | N goroutines reading from one channel |

**The mantra:** "Do not communicate by sharing memory; share memory by communicating." But don't dogmatize it -- mutexes are right for simple shared state.

```go
// Worker pool: fixed goroutines, bounded concurrency, clean shutdown
func process(ctx context.Context, jobs <-chan Job, results chan<- Result, n int) {
    var wg sync.WaitGroup
    for range n {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    results <- handle(job)
                }
            }
        }()
    }
    wg.Wait()
    close(results)
}
```

Never spawn unbounded goroutines. Gate with a semaphore or use a fixed pool.

### 5. Error Flow Design -- The Shape of the Function

Before implementing a function, think about its error shape:

**The rule:** errors peel off to the right. The happy path runs straight down the left edge. If your function's main logic is indented, refactor.

```go
func process(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("open %s: %w", name, err)
    }
    defer f.Close()

    data, err := io.ReadAll(f)
    if err != nil {
        return fmt.Errorf("read %s: %w", name, err)
    }

    return save(data)
}
```

**Error design decisions:**
- Can the caller recover? Use a sentinel (`var ErrNotFound = errors.New(...)`) or custom type
- Is the error just informational? Wrap with `fmt.Errorf("context: %w", err)`
- Error strings: lowercase, no punctuation, prefix with origin (`"image: unknown format"`)
- Never discard errors silently. If you can't return it, log it

---

## Hard Rules -- Things Go Developers Never Do

These are non-negotiable. Violating any of these is an instant tell that the code was not written by a Go developer.

1. **Never `os.Exit()` outside of `main()`.** Return errors up the call stack. Only `main()` decides to exit. Helper functions that call `os.Exit` are untestable and prevent defer cleanup. In cobra apps, `RunE` returns errors -- let cobra handle the exit code.

2. **Never hand-roll sorting.** Use `sort.Slice` or `slices.SortFunc`. Writing a manual insertion sort or bubble sort loop signals "translated from another language." If `sort.Slice` exists, use it.

3. **Interface methods must not return concrete implementation types.** If an interface method returns `*ssh.Session`, it's not really an interface -- it's permanently coupled to one implementation. Return `io.Reader`, `io.ReadCloser`, or another interface. The caller should not need to know what's underneath.

4. **Use named types for closed sets of values.** Event types, states, categories -- any finite set of known values gets a named type with `iota` or string constants. Never use raw strings like `"added"`, `"removed"` as event types.

    ```go
    type EventType string
    const (
        EventAdded   EventType = "added"
        EventToggled EventType = "toggled"
    )
    ```

5. **Use `encoding/json` for structured data.** Never invent custom delimited formats (pipe-separated, comma-separated). Go's standard library has `encoding/json`, `encoding/csv`, `encoding/gob`. Use them.

6. **Use the standard library reflexively.** Before writing a loop that does something that sounds like it should exist, check `sort`, `slices`, `strings`, `bytes`, `maps`, `sync`, `io`. Go developers reach for stdlib instinctively. If you're reimplementing something, you're probably doing it wrong.

7. **Never silently discard errors with `_`.** If the error truly cannot happen, add a comment explaining why. If you're discarding it for convenience, you're hiding bugs. Same for comma-ok returns -- if the `bool` carries semantic meaning, use it.

8. **Domain types use correct Go types.** Ports are `int`, timestamps are `time.Time`, durations are `time.Duration`. Parse strings to typed values at the boundary (when reading config, env vars, APIs). Never pass strings through the domain layer when a more specific type exists.

---

## Supporting Patterns

These patterns support the design decisions above. Read `references/effective-go-patterns.md` for full examples and edge cases on any of these.

### Embedding -- When to Reach for It

Embed a type when you want ALL its methods promoted to the outer type. This is Go's composition mechanism -- not inheritance, but delegation with automatic forwarding.

**Reach for embedding when:**
- Wrapping an `http.ServeMux` or `http.Server` with extra behavior
- Adding methods to a logger (`*log.Logger` embedded in a service type)
- Composing interfaces (`ReadWriter` embeds `Reader` + `Writer`)
- Building test mocks that satisfy a large interface but only override 1-2 methods
- A struct "is a" something with extra state (e.g., `TimedMutex` embeds `sync.Mutex`)

```go
// Embed when you want all methods promoted
type Server struct {
    *http.ServeMux           // promotes Handle, HandleFunc, ServeHTTP
    timeout time.Duration
}

// Embed for test mocks -- satisfy the interface, override what you need
type mockStore struct {
    *RealStore              // satisfies all methods
    getFn func(string) Item // override just Get
}
func (m *mockStore) Get(id string) Item { return m.getFn(id) }

// Embed interfaces to compose them
type ReadWriteCloser interface {
    io.Reader
    io.Writer
    io.Closer
}
```

**Don't embed** when you only need 1-2 methods -- write explicit forwarding instead. Embedding promotes EVERYTHING, which can expose methods you didn't intend.

### Composite Literals -- Named Fields Always

```go
cfg := Config{
    Host:    "localhost",
    Port:    8080,
    Timeout: 10 * time.Second,
}
```

Positional fields break silently when struct fields are reordered. Named fields are self-documenting and safe.

### new vs make

- `new(T)` -- zeroed memory for any type, returns `*T`
- `make(T, args)` -- initialized slices, maps, channels only, returns `T`

### Defer -- Immediately After Acquisition

```go
mu.Lock()
defer mu.Unlock()

f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()
```

Never defer in a loop -- defer runs at function return, not loop iteration. Extract the loop body to a separate function.

### Context Propagation

- Always first parameter, always named `ctx`
- Never store in a struct
- Pass at call time, not construction time

---

## Quick-Check

Before finalizing Go code, verify:

- [ ] Zero values: would `var x T` panic? If so, fix or document
- [ ] Interfaces: defined at consumer, not provider? As small as possible?
- [ ] Package names: does `pkg.ExportedName` read well with no stutter?
- [ ] Error flow: happy path on the left edge? Errors wrap with context?
- [ ] Concurrency: bounded goroutines? Clear ownership model?
- [ ] Functions: accept narrowest interface, return concrete type?

For full pattern reference with edge cases: `references/effective-go-patterns.md`
