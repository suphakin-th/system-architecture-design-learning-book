# Architecture 35 — GhostExec: OS-Native Serverless (Original Design)

---

## At a Glance

| | |
|---|---|
| **Type** | Self-hosted serverless runtime built entirely on OS primitives — no container runtime, no cloud SDK |
| **Complexity** | High (the payoff is zero external dependencies, at the cost of hand-managing what a platform normally hides) |
| **Best for** | Short-lived jobs where you want serverless *ergonomics* (ephemeral, isolated, self-cleaning) without a serverless *platform* |
| **Avoid when** | You already have Kubernetes/Lambda in place and the migration cost isn't worth "no dependency" as a goal in itself |
| **Status** | Original architecture design — full spec written, not yet implemented |

---

## What Is It?

Every cloud provider's "serverless" offering is really a bundle of five guarantees: a job runs in an isolated sandbox, gets private scratch storage, moves data in and out safely, saves its result somewhere durable, and cleans up completely when it's done. GhostExec asks: **what if you built those five guarantees directly out of what the operating system already gives you, instead of out of a managed platform?**

The answer turns out to be "yes, and the primitives to do it — `tmpfs`, atomic `rename()`, namespaces, `cgroups`, `epoll` — have existed on every Unix-like system for decades, largely unused for this purpose because reaching for a container platform is the path of least resistance." GhostExec is what falls out of taking that "no" seriously and working through the consequences properly: not a toy demo, but a design that survives the kind of scrutiny a real distributed system needs — concurrent writers, crash recovery, memory forensics, fan-out safety.

**This isn't serverless-without-a-cloud-bill. It's "what is a cloud platform actually made of, underneath the marketing," rebuilt from first principles.**

---

## The Five Guarantees, and What Backs Each One

A cloud FaaS platform (Lambda, Cloud Functions) gives you these implicitly, as part of what you're paying for. GhostExec's design makes each one explicit and traces it to a specific OS-level mechanism:

| Guarantee | Cloud platform's version | GhostExec's version |
|---|---|---|
| **Isolated execution** | A managed micro-VM or gVisor sandbox per invocation | Chosen per-job from three levels — lightweight namespace isolation, a real QEMU microVM, or a managed background service — selected by a small, deterministic rule set, not hardcoded |
| **Private scratch storage** | Ephemeral `/tmp` inside the sandbox, wiped on teardown | A private, RAM-backed filesystem mount per job, named by a cryptographically random ID, that exists for exactly the job's lifetime |
| **Safe data movement** | The platform's internal queue/storage layer, opaque to you | Every handoff — a payload arriving, a job's state changing, a result being claimed — happens as one atomic filesystem operation, so no reader can ever observe a half-finished state |
| **Durable result** | Write to DynamoDB/S3 as part of the function's own code | An optional sink stage that claims the result exactly once and confirms it's saved *before* the workspace is allowed to be destroyed |
| **Complete cleanup** | The platform reclaims the sandbox; you never see it happen | A destruction sequence wired to the process lifecycle itself — not to "the code path that runs on success" — so it fires on crash and timeout exactly as reliably as on a clean exit |

The interesting design work isn't in the list above — it's in what breaks if you implement each row naively, and what it takes to make each guarantee actually hold under concurrency.

---

## Where the Real Design Work Is: Failure Modes, Not Happy Paths

A first-draft version of every layer above is almost easy to sketch. What makes this a genuine systems-design exercise is the list of ways a naive version fails silently under load — and the fact that each failure has the same underlying shape, which is the part worth internalizing:

**A background helper process outlives its purpose.** A microVM's virtual filesystem server is launched in the background to bridge host storage into the guest. If nothing records its process ID, nothing can kill it when the job ends — it becomes a slow, silent process leak, one per virtualized job, invisible until someone notices unexplained memory drift weeks later. The fix isn't complicated (capture the PID at launch, kill it explicitly, in the right order relative to tearing down the VM) — but you only find the bug by asking "who is responsible for ending this process's life?" for every process the system starts, not just the obvious ones.

**Cleanup order matters more than cleanup happening.** Reclaiming a RAM-backed filesystem mount and then deleting its now-empty directory tree is safe. Deleting the directory tree *first*, while something might still hold an open handle into it, then unmounting — is not: a stuck handle can leave memory pages un-reclaimed indefinitely, silently defeating the entire point of "ephemeral by default." Two operations that look interchangeable in a code review are not interchangeable at runtime.

**Randomness has to be cryptographically real, not merely "looks random."** When many workers can spawn child jobs concurrently, each generates its own private workspace ID. If that ID is seeded from anything with low real entropy — wall-clock time and process ID are the classic mistake — then workers all spawned in the same instant (exactly what happens during a fan-out burst) have far less actual randomness between them than the ID's bit-length suggests. Two jobs landing on the same ID means two jobs silently sharing one workspace, corrupting both. The fix is drawing from the kernel's cryptographic random source specifically, plus treating a collision as a retry-worthy event rather than an assumed impossibility.

**A classifier that can answer two different ways for the same input isn't a function — and if a job-routing engine wants to route jobs at all reliably, that has to be a hard rule.** A set of routing rules, expressed declaratively, is elegant — until two rules can both match the same job and there's no defined priority ordering. Without a strict "first match wins, nothing else can be considered" discipline, the same job description can end up routed two different ways depending on how the rules happen to get evaluated, which is a correctness bug wearing a design-elegance costume.

**Self-replication needs a stopping condition designed on purpose.** Letting jobs spawn other jobs (for fan-out/batch work) is a real, useful feature — and, left unbounded, it's a fork bomb with a job scheduler bolted onto it. A system that supports self-spawning hasn't actually finished designing that feature until it has *also* designed how deep a chain can recurse and how wide a single burst can fan out, both enforced *before* any real resources are committed, not after.

**"Deleted" and "forensically unrecoverable" are different claims, and conflating them is dangerous specifically when the data is sensitive.** RAM-backed storage sounds inherently safe — until you remember the OS can swap those pages to disk under memory pressure, same as any other memory. Destroying the mount doesn't touch that swap copy. For a system that might ever handle credentials or untrusted payloads, closing this gap (excluding the workspace from swap entirely, a one-time mount-level setting) is not optional polish, it's the difference between the security claim being true and merely sounding true.

**Result persistence and self-destruction are two independent processes racing to touch the same file, and nobody designed who wins.** If a job's result needs to land in a database, and cleanup fires immediately and unconditionally on job completion (which it must, to guarantee cleanup happens even on crash), then an asynchronous database writer and the cleanup process are both reaching for the same result file with no coordination between them. Whichever one is slower loses — and if cleanup wins, the result vanishes with no record it ever existed. This needed the exact same "claim it atomically first" discipline used everywhere else in the design, applied to one more handoff.

---

## The One Idea That Generalizes Across All of the Above

Every fix above has the identical shape, once you strip away the specific mechanism it's applied to:

> **The visible name of a piece of data changes via exactly one atomic operation. A reader only ever sees the fully-old state or the fully-new state — never something in between, and never two readers who both believe they arrived first.**

A file becomes visible only by being renamed into place after it's completely written, never by being written-in-place at its final name. A state transition is a rename between two fixed names, guarded so only one caller can win it. A workspace ID collision is treated as "try again," never as "assume this can't happen." A result gets *claimed* — atomically, exactly once — before anyone is allowed to consider it consumed.

This single discipline, applied consistently everywhere two independent parts of the system could otherwise race on the same piece of data, is doing most of the actual correctness work in the whole design. Almost everything else — which isolation backend to pick, which language to implement the daemon in, how jobs get routed — is a much easier decision once this one is settled, because it defines the contract every other piece has to respect.

---

## Design Decisions Worth Naming Explicitly

- **A declarative, rule-based router for deciding how to run each job** — rather than a cascading if/else — makes the routing logic auditable as a flat list of "when this is true, route here" statements, provided (see above) that at most one rule is ever allowed to match.
- **Not every execution path gets equally strong isolation, and that asymmetry has to be a stated decision, not an accident.** The cheapest, most common execution path shares a kernel with the host; only the heavier path gets a real hardware-virtualized boundary. If the system's default path is also its weakest one, and jobs of genuinely unknown trust can land on the default, that's backwards — the fix is either flipping the default or hardening the cheap path as far as it can go (dropping process capabilities, restricting which system calls it's even allowed to make) so "less isolated" doesn't mean "unrestricted."
- **Reimplementing the daemon core in a memory-safe systems language earns its place specifically because the bugs found in this design were bookkeeping failures, not logic errors** — a process handle nobody tracked, an operation ordering nobody enforced, two copies of the same cleanup logic quietly drifting apart. Those are exactly the class of mistake that a language with compiler-enforced ownership catches structurally, before the code ships, rather than under production load months later. The raw system calls underneath don't change; what changes is what's holding the bookkeeping together.

---

## Why This Exercise Was Worth Doing

The value of working through a design like this isn't "now I have a Docker alternative" — a mature, battle-tested container/orchestration ecosystem already exists and beats a fresh design on almost every practical axis. The value is what the exercise forces: naming every implicit guarantee a platform normally hides from you, then proving — not assuming — that each one actually holds when two independent processes are racing to touch the same piece of data at the same moment. That skill transfers directly to any distributed system, not just this one.

---

## Source & License

This page is an original-prose summary written for this learning library. The full technical specification — syscall-level detail, complete code sketches, and the race-condition analysis this writeup summarizes — lives in a separate repository, **[suphakin-th/ghostexec](https://github.com/suphakin-th/ghostexec)**, under an all-rights-reserved license: reading it for evaluation is welcome, but any use, copying, or reproduction of its architecture, documentation, or code requires prior written authorization from the author (see that repo's `LICENSE` for the exact terms).

This writeup itself does not reproduce that document's text or code — it's a fresh explanation of the same ideas — but the underlying design, the GhostExec name, and the source specification remain the property of the author under that license. Treat this page as a summary and pointer, not as a substitute for, or a grant of rights to, the source material.
