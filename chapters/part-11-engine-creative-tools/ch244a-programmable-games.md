# Chapter 244a: Programmable Games and Competitive-Code Sandboxes

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics and engine developers who need to understand how a game can execute untrusted third-party code as its core mechanic, and the isolation architectures that make that survivable; browser and web platform engineers will recognise the V8-isolate and WebAssembly machinery in §2 and §8 as the same primitives that back site isolation and Wasm sandboxing on the web.
> **Status**: First draft — 2026-08-08

Most games take input from a human at interactive rates. A small but architecturally distinctive family of games instead takes *a program* from the player, loads it into the simulation, and runs it — turn after turn, tick after tick, often against programs submitted by other players. The player's skill expression is the source code. There is no direct input channel at all: the agent inside the world is driven entirely by code the player wrote and the server executes.

That inversion moves the engineering problem out of input handling and rendering and into **untrusted code execution**. A programmable game's central design decision is not what the agent can do in the world; it is where the trust boundary sits, what crosses it, and what happens when player code loops forever, allocates without bound, or reaches for the filesystem. This chapter reads five open-source programmable games as *sandbox architectures*, not as gameplay designs, and arranges them along a single axis: how much of the isolation burden the host actually carries.

The escalation runs one way. Screeps executes player JavaScript **in-process**, inside a dedicated V8 isolate per player, and therefore must engineer memory limits, CPU accounting, and capability stripping itself. Robocode Tank Royale pushes each bot out to a **separate OS process** speaking a schema-defined WebSocket protocol, so the operating system supplies the isolation and the schema supplies the validation. RoboCup Soccer Simulation does the same thing over **raw UDP/TCP sockets** with an S-expression protocol, with the agent process typically not even on the same machine. Battlesnake goes further still: the bot is an **HTTP server the player hosts themselves**, so the game engine never executes player code at all. And Halite III, at the far end, runs bots as **plain subprocesses** via `/bin/sh` with full ambient host privilege — a model that is language-agnostic precisely because it delegates all trust to whoever operates the machine. Each step trades isolation strength for language freedom and operational simplicity, and reading them in order makes the tradeoff legible.

---

## Table of Contents

- [1. Programmable Games as a Sandbox Problem](#1-programmable-games-as-a-sandbox-problem)
  - [1.1 The Trust Boundary Is the Game](#11-the-trust-boundary-is-the-game)
  - [1.2 Four Things Every Sandbox Must Bound](#12-four-things-every-sandbox-must-bound)
- [2. Screeps: World — Per-Player V8 Isolates](#2-screeps-world--per-player-v8-isolates)
  - [2.1 Four Repositories, Four Responsibilities](#21-four-repositories-four-responsibilities)
  - [2.2 `isolated-vm`: One V8 Isolate per Player](#22-isolated-vm-one-v8-isolate-per-player)
  - [2.3 Capability Injection and the Cleanup Script](#23-capability-injection-and-the-cleanup-script)
  - [2.4 CPU Accounting: Script Timeouts and Heap Metering](#24-cpu-accounting-script-timeouts-and-heap-metering)
  - [2.5 Intents: The Only Channel to the World](#25-intents-the-only-channel-to-the-world)
  - [2.6 The PixiJS Renderer and Its Metadata Pipeline](#26-the-pixijs-renderer-and-its-metadata-pipeline)
- [3. Robocode: From SecurityManager to Process Isolation](#3-robocode-from-securitymanager-to-process-isolation)
  - [3.1 The In-Process Sandbox and Its Collapse](#31-the-in-process-sandbox-and-its-collapse)
  - [3.2 Tank Royale: Bots as Processes Behind a WebSocket](#32-tank-royale-bots-as-processes-behind-a-websocket)
  - [3.3 What the Schema Enforces](#33-what-the-schema-enforces)
- [4. RoboCup Soccer Simulation: Network Clients and S-Expressions](#4-robocup-soccer-simulation-network-clients-and-s-expressions)
  - [4.1 The 2D League: `rcssserver` over UDP](#41-the-2d-league-rcssserver-over-udp)
  - [4.2 The 3D League: SimSpark, ODE, and the OpenGL Monitor](#42-the-3d-league-simspark-ode-and-the-opengl-monitor)
- [5. Battlesnake: The Bot as a Microservice](#5-battlesnake-the-bot-as-a-microservice)
- [6. Halite III: Subprocess stdio and the Limits of Host Trust](#6-halite-iii-subprocess-stdio-and-the-limits-of-host-trust)
- [7. Sandbox Architecture Compared](#7-sandbox-architecture-compared)
  - [7.1 The Comparison Table](#71-the-comparison-table)
  - [7.2 Reading the Spectrum](#72-reading-the-spectrum)
  - [7.3 What the Open Record Cannot Show: Closed Judges](#73-what-the-open-record-cannot-show-closed-judges)
- [8. WebAssembly as the Successor Sandbox](#8-webassembly-as-the-successor-sandbox)
- [Integrations](#integrations)
- [References](#references)
- [Roadmap](#roadmap)

---

## 1. Programmable Games as a Sandbox Problem

### 1.1 The Trust Boundary Is the Game

In a conventional game the untrusted input is a stream of button states and cursor positions, validated by range checks. In a programmable game the untrusted input is a Turing-complete program, and "validation" is no longer a meaningful concept — you cannot statically decide whether a submitted script will terminate. The only workable posture is *containment plus metering*: run the code somewhere it cannot reach anything valuable, give it a bounded budget of time and memory, and cut it off when the budget is exhausted.

This makes programmable games an unusually honest source of sandboxing engineering. A game server that runs thousands of players' scripts on shared hardware, continuously, with real money or ranking at stake, cannot use a sandbox that merely *discourages* misbehaviour. Failures are load-bearing and visible, and the open-source examples in this chapter therefore contain production-tested code for problems — per-tenant heap limits, deterministic CPU accounting, capability revocation — that most engines never have to solve.

The relevance to a graphics stack book is twofold. First, these games are also renderers: the visualisation layer is usually a separate, independently versioned component that consumes a serialised world state, and its architecture (WebGL scene graph, OpenGL monitor, SVG board) is dictated by the fact that it must be decoupled from the simulation. Second, the isolation primitives themselves — V8 isolates, WebAssembly linear memory — are exactly the primitives a browser uses to keep a web page away from the GPU driver, covered from the browser side in Chapter 98.

### 1.2 Four Things Every Sandbox Must Bound

Across all five systems examined here, the same four resources recur as the things that must be bounded, and the architectural differences reduce almost entirely to *which layer* does the bounding:

1. **Wall-clock or CPU time per turn** — otherwise one player's infinite loop stalls the simulation for everyone.
2. **Memory** — otherwise a leaking script exhausts the host.
3. **Ambient authority** — filesystem, network, process spawning, and (in a graphics context) any device node. A script that can `open("/dev/dri/card0")` is not sandboxed.
4. **The action channel** — the set of effects the program may have on the world, which must be a small, validated, server-interpreted vocabulary rather than direct mutation of simulation state.

The four bounds are not independent. Pushing player code into its own OS process makes (3) nearly free, because the process runs as whatever user the host chooses and the game server holds no references it could leak; but it makes (1) and (2) coarse, because the host can now only observe wall-clock latency and must resort to signals to enforce anything. Keeping player code in-process makes (1) and (2) precise — V8 can report exact heap usage and CPU nanoseconds — but makes (3) an ongoing engineering hazard, because every global reachable from the script is a potential escape.

---

## 2. Screeps: World — Per-Player V8 Isolates

Screeps: World is the deepest available case study because the server, the simulation engine, the sandbox driver, and the renderer are all published separately under the ISC licence: `screeps/screeps` (the private-server package), `screeps/engine` (simulation), `screeps/driver` (storage, sandbox, and native modules), and `screeps/renderer` (the graphics engine used by the official client). [Source](https://github.com/screeps/screeps) [Source](https://github.com/screeps/engine) [Source](https://github.com/screeps/driver) [Source](https://github.com/screeps/renderer)

The game's premise is that the player never issues a command directly. They write a JavaScript module; the server runs that module once per tick for every room the player occupies, and whatever the module does to the game objects it can see becomes a set of queued *intents* that the engine resolves.

### 2.1 Four Repositories, Four Responsibilities

```
┌────────────────────────────────────────────────────────────────┐
│ screeps/screeps  — private-server package, launcher, mod host  │
└───────────────┬────────────────────────────────────────────────┘
                │
     ┌──────────┴───────────┐
     ▼                      ▼
┌─────────────────┐   ┌──────────────────────────────────────────┐
│ screeps/engine  │   │ screeps/driver                           │
│ tick processing │◄──│  lib/runtime/user-vm.js  (isolate mgmt)  │
│ intent resolve  │   │  lib/runtime/runtime-driver.js (eval)    │
└─────────────────┘   │  lib/runtime/runtime.js  (in-isolate API)│
     │                │  native/  (node-gyp: pathfinder, etc.)   │
     │                └──────────────────────────────────────────┘
     ▼  serialised room state (JSON)
┌────────────────────────────────────────────────────────────────┐
│ screeps/renderer — PixiJS/WebGL 2D, metadata-driven processors  │
└────────────────────────────────────────────────────────────────┘
```

The split matters because it places the sandbox in its own package with its own native build step. `@screeps/driver` declares a `node-gyp rebuild` install hook and pins its sandbox dependency to an exact git commit rather than a semver range:

```json
{
  "name": "@screeps/driver",
  "version": "5.3.0",
  "license": "ISC",
  "scripts": {
    "install": "node-gyp rebuild -C native"
  },
  "dependencies": {
    "isolated-vm": "github:laverdet/isolated-vm#cb93efcc1881c826ee98ad34e66955c08713acb2"
  }
}
```

[Source](https://github.com/screeps/driver/blob/master/package.json)

Pinning a sandbox to a commit hash rather than a version range is a deliberate posture: the isolation layer is a security boundary, and a transitive minor-version bump inside it is a change to the threat model, not a routine upgrade.

### 2.2 `isolated-vm`: One V8 Isolate per Player

`isolated-vm` is a native Node module that exposes V8's isolate API directly to JavaScript: each `Isolate` is a fully independent V8 instance with its own heap and garbage collector, and objects cannot be shared across the boundary except by explicit copy or reference. [Source](https://github.com/laverdet/isolated-vm)

Screeps allocates one isolate per player, per runtime worker, with a memory limit computed from the world's static terrain size:

```javascript
// screeps/driver — lib/runtime/user-vm.js
let isolate = new ivm.Isolate({
    inspector,
    snapshot,
    memoryLimit: 256 + staticTerrainDataSize / 1024 / 1024,
});
let context = await isolate.createContext({inspector});
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

Two details in that constructor call carry most of the design.

The **`snapshot`** is a V8 startup snapshot loaded from a prebuilt binary artefact:

```javascript
snapshot = new ivm.ExternalCopy(
    fs.readFileSync(require.resolve('../../build/runtime.snapshot.bin')).buffer
);
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

Because a new isolate must otherwise parse and compile the entire game API before it can run a single line of player code, and because isolates are created and destroyed continuously as players' rooms are processed, the snapshot converts per-isolate API construction from a compile step into a memory-image restore. This is the same mechanism Chromium and Node use to shorten startup, applied per tenant rather than per process.

The **`memoryLimit`** is expressed in megabytes. It is important to read `isolated-vm`'s own characterisation of the option: it is documented as more of a guideline than a hard cap, since some allocations are made outside the isolate's accounted heap. [Source](https://github.com/laverdet/isolated-vm) The limit therefore functions as a *back-pressure* signal — a script that overruns it will hit V8 out-of-memory behaviour in its own isolate and be torn down — not as an absolute reservation of address space.

The `staticTerrainDataSize` term reflects the fact that terrain is shared, immutable, and large: the base 256 MB budget is topped up by however many megabytes of terrain the world requires, so that a bigger map does not silently shrink every player's usable heap.

Isolates are also **evicted when idle**. The driver installs a sweep that runs once a minute and clears any isolate last used more than three minutes ago:

```javascript
// screeps/driver — lib/runtime/user-vm.js (in exports.init)
setInterval(() => {
    for (let userId in vms) {
        if (vms[userId] && vms[userId].lastUsed < Date.now() - 3 * 60 * 1000) {
            exports.clear(userId);
        }
    }
}, 60 * 1000);
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

The retained-isolate model is what makes the design economically viable: a player's isolate persists across ticks, so their compiled code, their JIT state, and their in-memory caches survive, and only genuinely inactive players pay the cost of a cold start. The other path that discards an isolate is a code change — `exports.create` clears the existing VM when the submitted code's timestamp is newer than the one the isolate was built from, so a player editing their script gets a clean context rather than a context carrying stale globals. [Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js) This is the crucial performance advantage of in-process isolation over process-per-turn models, and it is the reason a game with tens of thousands of concurrently simulated scripts can run at all.

### 2.3 Capability Injection and the Cleanup Script

The hardest part of in-process sandboxing is not creating the isolate; it is arranging for privileged objects to be *available during setup* and *unreachable afterwards*. Screeps does this in two phases.

First, the driver injects the privileged handles into the fresh context's global object:

```javascript
// screeps/driver — lib/runtime/user-vm.js
context.global.setIgnored('global', context.global.derefInto());
context.global.setIgnored('_ivm', ivm);
context.global.setIgnored('_isolate', isolate);
context.global.setIgnored('_context', context);
context.global.setIgnored('_worldSize', index.getWorldSize());
context.global.setIgnored('_nativeMod', nativeModInstance.derefInto());
context.global.setIgnored('_constants', new ivm.ExternalCopy(index.constants).copyInto()),
context.global.setIgnored('_customObjectPrototypes', new ivm.ExternalCopy(index.customObjectPrototypes).copyInto()),
context.global.setIgnored('_customIntentTypes', new ivm.ExternalCopy(index.config.customIntentTypes).copyInto()),
context.global.setIgnored('_halt', new ivm.Reference(function() {
    vm.didHaltByUserRequest = true;
    isolate.dispose();
}));
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

These are genuine capabilities. `_ivm` is the module that can create isolates; `_nativeMod` is a handle to a compiled `.node` binary loaded via `ivm.NativeModule`; `_halt` is an `ivm.Reference` wrapping a host-side closure that disposes the isolate. If any of them remained reachable from player code, the sandbox would be over.

Second, immediately after the in-isolate bootstrap has captured what it needs into closures, a cleanup script deletes every injected global by name:

```javascript
// screeps/driver — lib/runtime/user-vm.js (cleanup pass)
isolate.compileScript('new ' + function () {
    delete global._ivm;
    delete global._isolate;
    delete global._context;
    delete global._init;
    delete global._evalFn;
    delete global._start;
    delete global._setStaticTerrainData;
    delete global._worldSize;
    delete global._nativeMod;
    delete global._constants;
    delete global._halt;
    delete global._customObjectPrototypes;
    delete global._customIntentTypes;
});
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

This is the classic **inject-then-revoke** pattern, and its correctness is entirely a matter of the delete list being exhaustive. Every name added on the injection side must appear on the deletion side; an omission is a capability leak. The pattern is fragile by construction, which is precisely the argument for the WebAssembly model in §8, where the module's imports are its *only* capabilities and there is no ambient global object to forget to clean.

The in-isolate runtime compounds this with defensive object construction. Intents are accumulated on prototype-less objects and incoming tick data has its prototype severed before player code can observe it:

```javascript
// screeps/driver — lib/runtime/runtime.js
global._start = function (data) {
    systemFunctions.Object.setPrototypeOf(data, null);
    // ...
};
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js)

Using `Object.create(null)` for intent containers and stripping prototypes from host-supplied data closes prototype-pollution paths: a script that assigns to `Object.prototype` cannot thereby inject a key into an intent object that the engine will later read, because the intent object has no prototype chain to poison. The runtime also captures pristine references to intrinsics (a `systemFunctions` namespace) before player code runs, so the sandbox's own logic does not call methods the player may have monkey-patched.

### 2.4 CPU Accounting: Script Timeouts and Heap Metering

Time is enforced at two levels. The coarse level is V8's script timeout, applied when the driver evaluates player code:

```javascript
// screeps/driver — lib/runtime/runtime-driver.js
exports.evalCode = function(module, globals, returnValue, timeout, scriptCachedData) {
    // ...
    options.timeout = timeout + 5;
    if (options.timeout < 30) {
        options.timeout = 30;
    }
    // ...
};
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime-driver.js)

The five-millisecond grace and 30 ms floor are pragmatic: V8's timeout is a wall-clock interrupt on script execution, and a floor prevents pathological configurations where the allotted budget is smaller than the interrupt granularity. When the timeout fires, V8 throws with a generic message, which the driver rewrites into a game-domain error so that the player sees a CPU-budget explanation rather than a V8 internal:

```javascript
// screeps/driver — lib/runtime/runtime-driver.js
if (e.message === 'Script execution timed out.') {
    e.message = 'Script execution timed out: CPU time limit reached';
}
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime-driver.js)

The driver also wraps the submitted module so that it executes as a function body rather than in global scope:

```javascript
// screeps/driver — lib/runtime/runtime-driver.js
'(function __module(module,exports){ ' + module.code + "\n})(__module, __module.exports)"
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime-driver.js)

Beyond CommonJS emulation, the wrapper prevents top-level `var` and function declarations from landing on the global object, keeping the isolate's global namespace under the sandbox's control across ticks. The driver additionally maintains a per-user, timestamp-keyed script cache and scrubs host paths out of stack traces before returning errors to the player — the latter being an information-disclosure control, not a cosmetic one.

The fine-grained level is CPU accounting inside the isolate, read from V8 itself:

```javascript
// screeps/driver — lib/runtime/runtime.js
function nowCpuTime() {
    return Number(isolate.cpuTime) / 1e6;
}
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js)

`isolate.cpuTime` is nanosecond-resolution CPU time attributed to that isolate, so dividing by 10⁶ yields milliseconds. Because the value is per-isolate rather than per-process, a player is charged only for their own execution even though many isolates share a worker — which is the property that makes a CPU-based game economy possible at all. The runtime exposes this to player code as the in-game CPU meter, and the `_halt` reference from §2.3 provides the hard stop: the in-isolate side wraps it as

```javascript
// screeps/driver — lib/runtime/runtime.js
const cpuHalt = halt ? function() {
    halt.applySync();
    throw new Error("No one should ever see this message.");
} : undefined;
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js)

The unreachable `throw` is a defensive marker: `halt.applySync()` calls into the host, which disposes the isolate, so execution should never resume past that line. If it ever does, the exception message makes the impossible state loud rather than silent.

Memory is metered through V8's own heap statistics, read per isolate. `exports.getMetrics` collects raw statistics for every live VM, guarding against disposed isolates:

```javascript
// screeps/driver — lib/runtime/user-vm.js
if (!vms[userId].isolate.isDisposed) {
    result.heap = vms[userId].isolate.getHeapStatisticsSync();
}
```

and an optional reporting interval turns those statistics into per-user figures:

```javascript
// screeps/driver — lib/runtime/user-vm.js (config.engine.reportMemoryUsageInterval)
let heap = require('v8').getHeapStatistics();
console.log(`# Main heap: ${heap.total_heap_size}`);
console.log(`# ExternalCopy.totalExternalSize: ${ivm.ExternalCopy.totalExternalSize}`);

exports.getMetrics().forEach(user => {
    console.log(`# User ${user.userId} heap: ${user.heap.total_heap_size + user.heap.externally_allocated_size}`);
});
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js)

Charging a user `total_heap_size + externally_allocated_size` rather than heap alone is the correct choice for a multi-tenant context: `ExternalCopy` allocations — the mechanism by which structured data crosses the isolate boundary — are real memory the host has committed on that player's behalf, and a metric that ignored them would be gameable. The separate `ExternalCopy.totalExternalSize` line gives the process-wide total, so an operator can see whether boundary-crossing copies are the dominant cost across all tenants.

### 2.5 Intents: The Only Channel to the World

Even with a leak-free isolate, the sandbox would be pointless if player code could mutate simulation state directly. It cannot. The in-isolate runtime builds an intent accumulator on a prototype-less object and charges CPU per intent:

```javascript
// screeps/driver — lib/runtime/runtime.js
let intentCpu = 0.2,
    freeMethods = systemFunctions.Object.create(null, {say: {value: true}, pull: {value: true}});

let intents = {
    list: systemFunctions.Object.create(null),
    cpu: 0,
    set(id, name, data) {
        this.list[id] = this.list[id] || systemFunctions.Object.create(null);
        if (!freeMethods[name] && !this.list[id][name]) {
            this.cpu += intentCpu;
        }
        this.list[id][name] = data;
    },
    // push, pushByName, remove ...
};
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js)

Three properties of this design are worth naming. First, every action a script takes is *recorded, not performed*: the isolate produces a list of requested intents and the engine — outside the sandbox — validates and resolves them against the authoritative world state. A script that requests an illegal move produces a rejected intent, not a corrupted simulation.

Second, intents are priced. The `intentCpu` charge of 0.2 means the action channel itself has a cost, so a script cannot spam intents for free even within its time budget. Two details refine this: `freeMethods` exempts `say` and `pull` from the charge, and `set` only charges when the intent is *new* for that object (`!this.list[id][name]`), so overwriting a pending intent for the same object in the same tick is free — the cost tracks distinct queued actions rather than the number of assignments a script makes. The `remove` method correspondingly refunds `intentCpu` when a queued intent is withdrawn.

Third, the persistent-state channel is explicitly capped. Screeps offers a raw string memory segment, and the runtime rejects oversized writes:

```javascript
// screeps/driver — lib/runtime/runtime.js (RawMemory setter)
if (value.length > 2 * 1024 * 1024) {
    throw new Error('Raw memory length exceeded 2 MB limit');
}
```

[Source](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js)

Because this data is serialised and persisted by the host between ticks, an unbounded write would be an unbounded storage cost — a resource outside the isolate's own heap accounting, and therefore one that needs its own explicit limit.

### 2.6 The PixiJS Renderer and Its Metadata Pipeline

Screeps' graphics engine was published as an open-source project distinct from the game itself, described at release as being based on PixiJS and containing the same renderer code and images used in the official client, with two intended uses: custom graphics for private servers, and third-party GUI utilities such as standalone room-history viewers. [Source](https://blog.screeps.com/2018/08/renderer/)

The published package confirms the technology choice:

```json
{
  "name": "@screeps/renderer",
  "version": "1.6.10",
  "license": "ISC",
  "main": "dist/renderer.js",
  "types": "./main.d.ts",
  "devDependencies": {
    "pixi.js": "^7.4.2",
    "webpack": "^5.99.9"
  }
}
```

[Source](https://github.com/screeps/renderer/blob/master/engine/package.json)

PixiJS 7 is a WebGL-backed 2D scene graph, so the entire visualisation runs through the browser's GPU path — the same `WebGLRenderingContext`/`WebGL2RenderingContext` plumbing that Chapter 98 traces down to the platform driver.

The architecturally interesting part is that the renderer is **metadata-driven** rather than hard-coded per object type. State enters through a single entry point and is dispatched by lookup:

> `GameRenderer.applyState` receives serialised game state, which populates a `World`; preprocessors (for example `setBadgeUrls`) run over it; each object becomes a `GameObject`; the object's type is looked up in a metadata table which supplies its `calculations`, `actions`, and `processors`. [Source](https://github.com/screeps/renderer/blob/master/README.md)

Processors fall into two families. One family creates PIXI display objects — `object`, `container`, `draw` (producing a `PIXI.Graphics`), `sprite`, `text`. The other implements special-purpose behaviour — `circle`, `creepActions`, `creepBuildBody`, `moveTo`. [Source](https://github.com/screeps/renderer/blob/master/README.md)

Externalising the object-type table into a separate `@screeps/renderer-metadata` package, with mod examples shipped by the launcher, is what makes third-party object types renderable without forking the renderer: a private server that adds a new structure type supplies metadata describing how to draw it, and the existing processors do the work. [Source](https://github.com/screeps/renderer/blob/master/README.md) [Source](https://github.com/screeps/launcher/tree/ptr/init_dist/example-mods/renderer)

This is the same decoupling the sandbox requires, applied to graphics: the renderer consumes a serialised snapshot and holds no reference to simulation internals, which is exactly what allows it to be reused in a replay viewer or an external tool with no server at all.

---

## 3. Robocode: From SecurityManager to Process Isolation

Robocode is the oldest widely known programmable game in this family: players write Java classes controlling a tank, and the battles run in a desktop application with a live 2D view. The project remains active — `robo-code/robocode` shipped v1.10.3 in March 2025, VER_1_11_0 in June 2025, and v1.11.1 in July 2025 — and is licensed under the Eclipse Public License v1.0. [Source](https://github.com/robo-code/robocode) [Source](https://github.com/robo-code/robocode/blob/master/LICENSE.txt)

Its interest here is historical and forced: Robocode's original sandbox was built on a JVM facility that the platform removed, and the response — a ground-up rewrite in which bots are no longer in-process at all — is the cleanest available illustration of the shift this chapter traces.

### 3.1 The In-Process Sandbox and Its Collapse

Classic Robocode loads player bots into the same JVM as the battle engine and constrained them with a custom security manager layered on Java's `SecurityManager`. That approach depended on a mechanism the JDK deprecated for removal in Java 17 via JEP 411, and then permanently disabled in Java 24 via JEP 486. [Source](https://openjdk.org/jeps/411) [Source](https://openjdk.org/jeps/486)

The project's own release notes record the consequences with unusual directness. The v1.9.5.5 entry (29 March 2025) states that Robocode has its own security manager built on top of Java's Security Manager, "which has now been removed with Java 24," and that fixing the issue "requires a big rewrite of large parts of the security mechanisms in Robocode." The v1.10.0 entry (4 June 2025) records that new security mechanisms were implemented that work with Java 24+ while maintaining compatibility with Java 8+, and that the code around `RobocodeSecurityManager` was refactored for compatibility with Java 24 and newer. [Source](https://github.com/robo-code/robocode/blob/master/versions.md)

**Note: needs verification.** The release notes assert that replacement mechanisms exist but do not describe what they are. The specific technique used by Robocode 1.10.0 to constrain in-process bot code after `SecurityManager` was disabled is not documented in the sources consulted here, and should not be inferred.

The general lesson is independent of the specific fix, and it is the reason §2's cleanup-script fragility matters: **a language-level in-process sandbox is only as durable as the language runtime's commitment to that facility.** JEP 411's own rationale is that the Security Manager was not an effective mechanism for sandboxing untrusted code and imposed a maintenance burden disproportionate to its value. [Source](https://openjdk.org/jeps/411) A project that built its entire trust boundary on it had no migration path that preserved the architecture.

### 3.2 Tank Royale: Bots as Processes Behind a WebSocket

`robocode-dev/tank-royale` is the modern rewrite, written primarily in Kotlin and licensed under Apache-2.0, with v1.0.0 released in May 2026. [Source](https://github.com/robocode-dev/tank-royale)

Its architectural answer is to stop hosting bots at all. A bot is an independent program — the official APIs cover the JVM, .NET, Python, and JavaScript — that connects to a server over WebSocket and exchanges JSON messages. Battles are observed through a web-based viewer, which is a separate WebSocket client of the same server.

A bot from the JVM tutorial is an ordinary Java class with an ordinary `main`:

```java
// MyFirstBot.java — Tank Royale JVM tutorial
import dev.robocode.tankroyale.botapi.*;
import dev.robocode.tankroyale.botapi.events.*;

public class MyFirstBot extends Bot {
    public static void main(String[] args) {
        new MyFirstBot().start();
    }

    @Override
    public void run() {
        while (isRunning()) {
            forward(100);
            turnGunRight(360);
            back(100);
            turnGunRight(360);
        }
    }

    @Override
    public void onScannedBot(ScannedBotEvent e) {
        fire(1);
    }

    @Override
    public void onHitByBullet(HitByBulletEvent e) {
        double bearing = calcBearing(e.getBullet().getDirection());
        turnLeft(90 - bearing);
    }
}
```

[Source](https://robocode.dev/tutorial/jvm/my-first-bot-for-jvm.html)

The deployment shape is what changed. A bot directory contains the source, a shell script that launches it, and a JSON descriptor whose filename must match the directory name:

```sh
#!/bin/sh
java -cp ../lib/* MyFirstBot.java
```

[Source](https://robocode.dev/tutorial/jvm/my-first-bot-for-jvm.html)

Because the launch script is an arbitrary shell command, the server does not need to know anything about the bot's language or toolchain — it needs only to start a process and accept a WebSocket connection from it. That is the same language-agnosticism property that Halite achieves in §6, but reached without also handing the bot a shared address space.

### 3.3 What the Schema Enforces

Tank Royale's protocol is specified as a set of JSON Schema (draft 2020-12) documents, one per message type, with a common base schema that all messages extend. [Source](https://github.com/robocode-dev/tank-royale/tree/master/schema/schemas) The schema is not documentation-after-the-fact; it is the contract that replaces the in-process type system as the validation boundary.

Connection begins with a handshake in each direction. The bot's handshake carries identity and capability metadata: `sessionId`, `name` (1–30 chars), `version` (1–20), `authors` (up to 20 entries), `description` (up to 250), `homepage`, `countryCodes` as ISO 3166-1 values, `gameTypes` from `classic`/`melee`/`1v1`, `platform`, `programmingLang`, optional `initialPosition`, team fields, an `isDroid` flag documented as a team bot with 120 energy but no scanner, a `secret` used for access control with the server, and a `debuggerAttached` flag. Only `sessionId`, `name`, `version`, and `authors` are required. [Source](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/bot-handshake.schema.yaml)

The server's handshake mirrors it and adds two fields that matter for competitive integrity: a `variant` identifying the product, a SemVer `version`, a `behaviorVersion` documented as a server-owned compatibility version for outcome-changing battle behaviour, and a `features` object advertising capabilities such as `debugMode`. [Source](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/server-handshake.schema.yaml)

A distinct, integer `behaviorVersion` separate from the release version is a notable piece of design. In a competitive game, a change that alters battle outcomes is categorically different from a bug fix: rankings computed under different physics are not comparable. Making that a first-class protocol field lets clients and tooling detect outcome-relevant divergence without parsing changelogs.

Per-turn control flows through `bot-intent`, which is explicitly **delta-encoded** — the schema states that a field only needs to be set if the value must be changed. [Source](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/bot-intent.schema.yaml) The message carries `turnRate`, `gunTurnRate`, `radarTurnRate`, `targetSpeed`, `firepower`, the `adjustGunForBodyTurn`-family flags, `rescan`, `fireAssist` (default true), seven colour fields, `teamMessages` (at most 4 items), `debugGraphics`, and — importantly — `stdOut` and `stdErr`.

Three observations follow from the intent schema. First, the bot never states a position or a state; it states *rates and targets*, which the server integrates. All actuator values are server-clamped: `firepower` is bounded to an effective `[0.1, 3.0]`, and a shot occurs only when gun heat is zero and the bot has more energy than the requested firepower. A bot that sends nonsense gets clamped values, not an inconsistent world.

Second, delta encoding is a bandwidth and clarity decision that also constrains the bot: because unset fields mean "unchanged," there is no way to express an atomic multi-field state assignment that the server did not model.

Third, routing the bot's `stdOut` and `stdErr` **through the protocol as data fields** rather than reading the process's pipes is a small but telling inversion. The bot's console output becomes part of the turn message, attributable to a specific turn and displayable in the viewer, and the server never has to manage the bot's file descriptors. Compare §6, where Halite reads the subprocess's stderr directly and must therefore impose its own length cap.

The tick message flowing the other way is minimal and fully required: `roundNumber` (1-based), `botState`, `bulletStates`, and `events` — all four mandatory. [Source](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/tick-event-for-bot.schema.yaml) There is no partial world state and no optional field to misinterpret; each turn the bot receives a complete, schema-validated view of what it can perceive.

---

## 4. RoboCup Soccer Simulation: Network Clients and S-Expressions

RoboCup's simulation leagues predate most of this chapter's other examples and arrive at the same conclusion from a different direction. The goal is research reproducibility rather than commercial sandboxing, but the architecture is identical in shape: a simulation server owns the world, agents are separate processes, and everything crosses a socket.

### 4.1 The 2D League: `rcssserver` over UDP

`rcsoccersim/rcssserver` is the 2D league's simulator, written in C++ and licensed LGPL-3.0. [Source](https://github.com/rcsoccersim/rcssserver) It builds two binaries, `rcssserver` and a reference `rcssclient`, and deliberately does not include a visualiser — `rcssmonitor` is a separate program that connects as another network client.

The server's default ports are declared in source rather than convention:

```cpp
// rcssserver — src/serverparam.cpp
const int ServerParam::DEFAULT_PORT_NUMBER = 6000;
const int ServerParam::COACH_PORT_NUMBER   = 6001;
const int ServerParam::OLCOACH_PORT_NUMBER = 6002;
```

[Source](https://github.com/rcsoccersim/rcssserver/blob/master/src/serverparam.cpp)

Three ports for three privilege classes — players on 6000, the trainer/coach on 6001, and the online coach on 6002 — is a capability separation expressed as network topology. A player client physically cannot issue coach commands, because the socket it holds does not accept them.

Agents speak an ASCII S-expression protocol over UDP. A client announces itself and receives its assigned identity:

```
(init TeamName [(version VerNum)] [(goalie)])
→ (init Side Unum PlayMode)
```

where `Side` is `l` or `r` and `Unum` is a uniform number from 1 to 11. Actions are single S-expressions: `(dash Power [Direction])`, `(turn Moment)` with `Moment` in −180…180, `(kick Power Direction)`, `(catch Direction)`, `(move X Y)`, and `(tackle PowerOrAngle [Foul])`. Perception arrives as `(see Time ObjInfo+)`, `(hear Time Sender "Message")`, and a `(sense_body Time (view_mode …) (stamina …) (speed …) (head_angle …) …)` bundle. [Source](https://rcsoccersim.readthedocs.io/en/latest/)

Two properties are worth extracting for the comparison in §7. The protocol is **UDP and lossy by design**: an agent that fails to respond within a simulation cycle simply does not act that cycle, so timeout enforcement is inherent to the transport rather than bolted on. And perception is **egocentric and partial** — `(see …)` reports relative polar observations of what the agent can currently see, not the world state. That is a sandbox property as much as a research one: an agent cannot cheat by reading global state, because the socket never carries it.

### 4.2 The 3D League: SimSpark, ODE, and the OpenGL Monitor

The 3D league runs on SimSpark, hosted at `gitlab.com/robocup-sim/SimSpark` and described as a generic physical multiagent simulator system for agents in three-dimensional environments, built on the Spark application framework and used as the official RoboCup 3D simulation server; `rcssserver3d` is the soccer-specific application on top of it. [Source](https://gitlab.com/robocup-sim/SimSpark) [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/About-SimSpark)

Physics is **pluggable rather than hard-wired**. SimSpark's abstract physics layer is responsible for physics simulation and collision detection; the physics module originally used the Open Dynamics Engine directly, and the abstraction layer was introduced to make the engine swappable, with ODE becoming a plugin behind that interface. The wiki also records in-progress work on a Bullet implementation motivated specifically by the prospect of GPU acceleration. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Abstract-Physical-Layer) [Source](https://www.ode.org/)

Agents connect over **TCP port 3100** by default. Messages in both directions are S-expressions in the default ASCII character set, and each message is prefixed with the payload length as a 32-bit unsigned integer in network order (big endian). On connecting, an agent must first send a create-effector message, followed by an init-effector message. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Network-Protocol)

The length prefix is the necessary consequence of moving from UDP to TCP: datagrams are self-delimiting, streams are not. The choice of S-expressions is justified in the protocol documentation on grounds that read as forward-compatibility engineering rather than aesthetics — the syntax is compact and human-readable for debugging, and new sensors can be added because a client-side parser can simply ignore parts it does not recognise. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Network-Protocol)

The effector vocabulary is where the 3D league's ambition shows. A humanoid model has 22 controllable joints, and control is per-joint:

```
(scene rsg/agent/nao/nao.rsg)          ; load the robot model into the world
(init (unum 1)(teamname FHO))          ; claim a uniform number and team
(beam 10.0 -10.0 0.0)                  ; position self before kickoff: x, y, facing°
(lae3 5.3)                             ; drive one hinge joint at 5.3 deg/cycle
(say helloworld)                       ; broadcast to nearby agents
(syn)                                  ; synchronisation signal
```

[Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Effectors)

The `(scene …)` command is notable as a sandbox boundary: the agent does not construct its body, it *names a scene-graph file the server already trusts*. The server instantiates the model from `rsg/agent/nao/nao.rsg`, so an agent cannot define a robot with advantageous physics — it can only select from server-side assets.

Perception is a stream of sensor S-expressions, again egocentric:

```
(HJ (n laj3) (ax -1.02))                                   ; hinge joint angle
(GYR (n torso) (rt 0.01 0.07 0.46))                        ; gyro rates
(ACC (n torso) (a 0.00 0.00 9.81))                         ; accelerometer
(FRP (n lf) (c -0.14 0.08 -0.05) (f 1.12 -0.26 13.07))     ; force resistance: point + force
(TCH n bumper val 1)                                       ; touch
(See (G2R (pol 17.55 -3.33 4.31)))                         ; vision: polar (dist, φ, θ)
(GS (sl 1) (sr 2) (t 0.00) (pm BeforeKickOff))             ; game state
(AgentState (temp 48) (battery 75))                        ; simulated robot health
(hear MyTeam 12.3 self helloyourself)                      ; team, time, source, message
```

[Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Perceptors)

The force-resistance perceptor `(FRP …)`, reporting both a contact point and a force vector, is a direct read-out of the physics layer's contact solver, and the `(AgentState (temp …) (battery …))` perceptor simulates thermal and power limits — constraints on the *agent's* resources expressed inside the game world rather than enforced by the host. That is a third category of metering, distinct from the CPU/memory budgets of §2 and the timeouts of §5 and §6: the simulation itself charges the agent for aggressive actuation.

**The update loop does not guarantee reproducibility.** It is tempting to assume that a server-owned physics simulation is deterministic, and SimSpark's documentation explicitly says otherwise. The simulator implements a simple internal event model that immediately executes every action received from an agent, and makes no attempt to compensate for network latency or for differing computing resources across connected agents. The consequence is stated directly: SimSpark does not guarantee that events are reproducible, and repeated simulations may have different outcomes depending on network delays or load variations on the machines hosting the agents and the server. The stated benefit of the simple structure is speed, which makes it attractive for machine-learning workloads where large numbers of agent and simulation configurations are tested repeatedly. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Simulation-Update-Loop)

The main loop is built from plugins called simcontrol nodes, which respond to control events: an `init` event at startup, a `done` event at shutdown, and a repeating cycle of `start cycle`, `sense agent`, `act agent`, `end cycle`. Each cycle is 20 ms — 50 Hz. If the simulation runs faster than real time it waits; if it runs slowly it executes several physics updates in a row without interacting with agents, and if it falls far enough behind it gives up catching up and emits a warning. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Simulation-Update-Loop)

That degradation behaviour is the mechanism behind the non-reproducibility, and it is a real architectural cost of the network-client model. Screeps, by contrast, can be strict because it *owns* the execution of player code and can charge CPU deterministically per isolate; SimSpark cannot, because the agent's thinking happens on a machine it does not control and the loop must keep moving regardless. Trading reproducibility for throughput is defensible for a research platform used to train agents, and would be much harder to defend for a ranked competitive ladder.

**The monitor is a separate rendering client.** Visualisation follows the same decoupling as the 2D league. The SimSpark monitor is responsible for rendering the current simulation; it connects to a running server instance and continuously receives a stream of updates describing simulation state either as full snapshots or as incremental updates, in a customisable language called the Monitor Format. The monitor renders only the pure scene and defers game-specific state — play mode, scores — to plugins that draw it as an overlay, and it can also play back recorded log files. A server-internal monitor implementation also exists, enabled in the `simspark.rb` setup script by turning on the rendering and input plugins. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Monitor)

The rendering path is OpenGL. The GUI's monitor frame renders the scene graph and custom render nodes in an OpenGL widget (`SparkGLWidget`), packaged as an attachable-frame plugin. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/GUI-Monitor-Frame)

Two things about this design generalise. Streaming *either* full snapshots *or* incremental updates lets a monitor join a match in progress — it takes a snapshot to establish state and deltas thereafter — which is the same problem a spectator client or a replay scrubber has to solve. And separating pure-scene rendering from game-state overlays is precisely the split Screeps achieves with its metadata table in §2.6: the renderer knows about geometry, and something pluggable knows about the game.

---

## 5. Battlesnake: The Bot as a Microservice

Battlesnake takes the isolation argument to its logical endpoint: the game engine does not run player code at all. `BattlesnakeOfficial/rules` is the Go implementation of the game rules, licensed AGPL-3.0. It is the reference engine and the one the local CLI below embeds, but it is no longer what runs ranked play: the production platform at arena.battlesnake.com is `BattlesnakeOfficial/arena`, an Axum-based Rust rewrite of the older `play.battlesnake.com` with its own PostgreSQL-backed web service and a Rust rules crate that simulates Standard games directly — calling each snake's endpoints in parallel — rather than shelling out to the Go engine. [Source](https://github.com/BattlesnakeOfficial/rules) [Source](https://github.com/BattlesnakeOfficial/arena) The wire contract to a snake is unaffected by which engine is driving it, since `GET /`, `POST /start`, `/move`, and `/end` are defined by the Battlesnake API rather than by either implementation.

A snake is an HTTP server that the player deploys and operates. The engine is a pure HTTP *client*, calling four endpoints:

```
GET  /       → {"apiversion":"1","author":…,"color":"#888888",
                "head":"default","tail":"default","version":…}
POST /start  → response ignored (turn is always 0)
POST /move   → {"move":"up","shout":"Moving up!"}
POST /end    → response ignored
```

`move` must be one of `up`, `down`, `left`, `right`; `shout` is capped at 256 characters. [Source](https://docs.battlesnake.com/api/webhooks)

Local development runs the same engine as a CLI against a URL the developer supplies:

```bash
battlesnake play -W 11 -H 11 --name MySnake --url http://localhost:8000 -g solo -v
```

with game modes `solo`, `standard`, `wrapped`, `royale`, and `constrictor`. [Source](https://github.com/BattlesnakeOfficial/rules)

The CLI's request timeout defaults to 500 ms:

```go
// BattlesnakeOfficial/rules — cli/commands/play.go
playCmd.Flags().IntVarP(&gameState.Timeout, "timeout", "t", 500, "Request Timeout")
// ...
if gameState.Timeout == 0 {
    gameState.Timeout = 500
}
// ... http.Client{ Timeout: time.Duration(gameState.Timeout) * time.Millisecond }
```

[Source](https://github.com/BattlesnakeOfficial/rules/blob/main/cli/commands/play.go)

**Note: needs verification.** The 500 ms figure above is the default of the open-source Go CLI, read from source, and describes local development via `battlesnake play`. The production ranked platform runs the separate Rust engine in `BattlesnakeOfficial/arena` (below), whose request timeout is not published in its public source, so the production per-request latency budget should not be assumed equal to the CLI default.

Architecturally, Battlesnake is not a sandbox — it is the *absence* of one, and that is the point. All four of §1.2's bounds are handled by construction rather than by enforcement:

- **Time** is bounded by an HTTP client timeout. A slow snake simply does not get to move.
- **Memory** is not the engine's problem; the snake runs on hardware the player pays for.
- **Ambient authority** is irrelevant, because the snake has full authority over its own machine and none whatsoever over the engine's.
- **The action channel** is a single JSON field with four legal values.

What this model buys is total language and runtime freedom — any stack that can serve HTTP qualifies — at the cost of two things a hosted sandbox provides. Determinism is weakened: a snake's behaviour can depend on network conditions, and a competitor with better-provisioned infrastructure has a real advantage within the latency budget. And replayability is weakened: a recorded game can be replayed, but a *bot* cannot be re-run later unless its author keeps the service alive.

The visualisation follows the same decoupling. The board renderer is a separate AGPL-3.0 repository, `BattlesnakeOfficial/board`, written in Svelte and described as the game board with playback control. [Source](https://github.com/BattlesnakeOfficial/board) A DOM/SVG board is a defensible choice at Battlesnake's scale — an 11×11 grid of a few dozen elements needs no GPU pipeline — and it stands in useful contrast to Screeps' PixiJS/WebGL renderer, which must draw thousands of animated objects per room.

---

## 6. Halite III: Subprocess stdio and the Limits of Host Trust

Halite III was a competitive programming game run as an organised competition; the competition has concluded, but `HaliteChallenge/Halite-III` remains public under the MIT licence, with recent repository activity limited to automated dependency updates. [Source](https://github.com/HaliteChallenge/Halite-III)

Its execution model is the simplest possible one that supports arbitrary languages: the engine spawns the bot as a child process and talks to it over stdin/stdout with a text protocol. A starter bot is unremarkable code:

```python
# Halite-III — starter_kits/Python3/MyBot.py
import hlt
import logging

game = hlt.Game()
# As soon as you call "ready" function below, the 2 second per turn timer will start.
game.ready("MyPythonBot")

while True:
    game.update_frame()
    command_queue = []
    # ... decide moves ...
    game.end_turn(command_queue)
```

[Source](https://github.com/HaliteChallenge/Halite-III/blob/master/starter_kits/Python3/MyBot.py)

The starter kit notes that regular stdout — ordinary `print` — is reserved for engine-bot communication, which is why debugging must go through `logging` to stderr. [Source](https://github.com/HaliteChallenge/Halite-III/blob/master/starter_kits/Python3/MyBot.py) A protocol that occupies stdout makes the bot's most natural debugging facility unusable, and every language kit must document the workaround. Tank Royale's decision in §3.3 to carry `stdOut`/`stdErr` as protocol fields is a direct answer to this class of problem.

A kit consists of `MyBot.{ext}`, a modifiable `/hlt` helper library, and local test scripts, with per-language build files (`Cargo.toml`, `Package.swift`, `stack.yaml`, `mix.exs`, `project.clj`, `.csproj`) and an optional `install.sh`; the documented allowance is that a bot has ten minutes to install dependencies and compile on the game server. [Source](https://github.com/HaliteChallenge/Halite-III/blob/master/starter_kits/README.md) Running an arbitrary `install.sh` on the competition host is the clearest possible statement of where trust sits in this design.

The engine's Unix process launcher shows exactly what isolation is and is not present:

```cpp
// Halite-III — game_engine/networking/unix/UnixConnection.cpp
// Ignore SIGPIPE, as we want to detect bot exit gracefully.
signal(SIGPIPE, SIG_IGN);
// ... three pipes created; write and error ends set O_NONBLOCK via fcntl ...
auto pid = fork();
if (pid == 0) {
    // This is the child
    setpgid(getpid(), getpid());
    if (getppid() != ppid_before_fork) {
        throw NetworkingError("fork failed");
    }

    // Redirect stdin, stdout, and stderr
    CHECK(dup2(write_pipe[PIPE_HEAD], STDIN_FILENO));
    CHECK(close(write_pipe[PIPE_HEAD]));
    CHECK(dup2(read_pipe[PIPE_TAIL], STDOUT_FILENO));
    CHECK(close(read_pipe[PIPE_TAIL]));
    CHECK(dup2(error_pipe[PIPE_TAIL], STDERR_FILENO));
    CHECK(close(error_pipe[PIPE_TAIL]));

    // Execute the command
    CHECK(execl("/bin/sh", "sh", "-c", command.c_str(), nullptr));
    // Nothing past here should be run
    throw NetworkingError("exec failed");
}
```

and teardown:

```cpp
UnixConnection::~UnixConnection() noexcept {
    kill(-process, SIGKILL);
}
```

[Source](https://github.com/HaliteChallenge/Halite-III/blob/master/game_engine/networking/unix/UnixConnection.cpp)

Read this against §1.2's four bounds:

- **Ambient authority: unrestricted.** The child is `execl`'d into `/bin/sh -c <command>` and inherits the parent's user, filesystem view, and network access. There is no `seccomp` filter, no namespace unshare, no `chroot`, and no `setrlimit` in this path. The bot can open files, make outbound connections, and — relevant to this book — access device nodes the invoking user can access, including `/dev/dri/*`.
- **Time: wall-clock only.** Timeouts are enforced by the engine's own I/O logic; `send_string` throws a `TimeoutError` carrying the configured timeout, and reads are non-blocking with `select(2)`. There is no CPU-time accounting comparable to Screeps' `isolate.cpuTime`.
- **Memory: unbounded** by the engine. Nothing in the connection layer limits the child's address space.
- **Output: capped.** `MAX_STDERR_LENGTH = 1024 * 1024` bounds captured stderr, protecting the engine's own memory rather than the host.

The one real isolation primitive here is the **process group**. `setpgid(getpid(), getpid())` in the child makes the bot the leader of its own group, so `kill(-process, SIGKILL)` reaps the bot *and every process it spawned*. Without that, a bot that forked helpers would leave orphans behind after each match. It is a containment mechanism aimed at cleanup, not at confinement.

None of this was negligent — it was appropriate to the deployment. Halite ran matches on infrastructure the organisers controlled, presumably with isolation supplied at a layer below the game engine (containers or VMs) and with submissions subject to review. **Note: needs verification** — the specific host-level isolation used by the competition's match infrastructure is not documented in the repository files consulted here, and should not be assumed. The architectural point stands regardless: `UnixConnection.cpp` is the game engine's complete contribution to sandboxing, and it is a `fork` and an `execl`. Language-agnosticism came for free precisely because the engine made no attempt to understand or constrain the runtime it launched.

Halite's visualisation is likewise decoupled: replays are rendered by `libhaliteviz`, an HTML5-canvas viewer that reads recorded game files, so the engine emits data and the viewer is an independent consumer. [Source](https://github.com/HaliteChallenge/Halite-III/tree/master/libhaliteviz)

---

## 7. Sandbox Architecture Compared

### 7.1 The Comparison Table

| | **Screeps: World** | **Tank Royale** | **RoboCup Sim** | **Battlesnake** | **Halite III** |
|---|---|---|---|---|---|
| **Isolation unit** | V8 isolate, in-process | OS process | OS process (often remote host) | Player-operated HTTP service | OS process, `fork`+`execl` |
| **Boundary mechanism** | `isolated-vm` heap + context separation | WebSocket + JSON Schema validation | UDP/TCP socket + S-expression parse | Network + HTTP request/response | stdin/stdout pipes |
| **Enforced by** | V8 (and the game's own cleanup code) | OS + schema | OS + transport semantics | Nothing — no host execution | OS process group only |
| **Time bound** | V8 script `timeout`; per-isolate `isolate.cpuTime` | Per-turn message deadline | 20 ms cycle (3D); UDP loss = no action (2D) | HTTP client timeout (CLI default 500 ms) | Wall-clock `TimeoutError`; 2 s per turn after `ready()` |
| **Memory bound** | `memoryLimit` (MB, advisory) + heap metering | Host process limits (external to game) | Host process limits | Player's own infrastructure | None in engine |
| **Ambient authority** | Stripped: privileged globals injected then `delete`d | Full on the bot's own process; none over server | Full on the agent's host; none over server | Full on the player's host; none over engine | **Full on the game host** |
| **Action channel** | Priced intents (`intentCpu`), server-resolved | Delta `bot-intent`, server-clamped | S-expression effectors, server-integrated | One of four strings + shout | Text command queue |
| **Languages** | JavaScript / anything → JS or Wasm | JVM, .NET, Python, JS (any, via launch script) | Any with a socket | Any that serves HTTP | Any that runs on the host |
| **Cold-start cost** | Near zero — isolate retained ~3 min, V8 snapshot | Process spawn per battle | Process spawn per match | None (long-lived service) | Process spawn + up to 10 min build |
| **Determinism** | High — server owns clock and CPU accounting | Server integrates all state; `behaviorVersion` flags outcome changes | **Explicitly not guaranteed** in 3D — loop does not compensate for latency or host load | Weakened by network latency | Moderate — wall-clock dependent |
| **Renderer** | PixiJS 7 / WebGL 2D, metadata-driven | Web viewer (separate WS client) | `rcssmonitor` (2D); OpenGL monitor over snapshot/delta stream (3D) | Svelte/SVG board | HTML5 canvas (`libhaliteviz`) |

All cells above are drawn from the sources cited in §2–§6.

### 7.2 Reading the Spectrum

The table has a shape. Reading left to right, isolation strength inside the host decreases monotonically while language freedom increases, and the enforcement burden migrates from the game engine to the operating system and then to the player.

**Screeps pays for precision.** Because player code shares an address space with the server, Screeps gets exactly what an in-process sandbox uniquely offers: nanosecond CPU attribution per tenant, real heap statistics, retained warm state across ticks, and a snapshot-accelerated cold start. Those properties are what make a CPU-budget game economy possible — you cannot price a resource you cannot measure. The cost is that every escape vector is the game's own responsibility, which is why §2.3's inject-then-revoke list exists and why its exhaustiveness is load-bearing.

**Tank Royale pays for durability.** Robocode's history in §3.1 is the empirical case: an in-process language-level sandbox has a dependency on a language facility that may be withdrawn. Moving to processes plus a schema-validated protocol makes the trust boundary something the OS maintains, and turns the validation problem into a data-validation problem that a JSON Schema can express declaratively. What is lost is per-bot resource measurement — nothing in the protocol reports a bot's memory use — and cold-start efficiency.

**RoboCup pays for distribution — and, in the 3D league, for reproducibility.** Sockets and egocentric perception mean agents can run anywhere, in any language, on a competitor's own hardware at a tournament, and the server still owns physics; partial observability is simultaneously a research requirement and a security property. But §4.2's update loop makes the price explicit: because the server executes agent actions immediately and never compensates for latency or host load, identical inputs can produce different matches. That is the one cell in §7.1 where a *stronger* isolation model would not have helped — the non-determinism comes from the agent being on the far side of a network, which is the same property that provides the isolation.

**Battlesnake and Halite are the two extremes of the same decision** — "do not sandbox, delegate" — resolved in opposite directions. Battlesnake delegates *to the player*, who bears all cost and all risk, and the engine achieves perfect safety by never executing anything. Halite delegates *to the host operator*, who must supply isolation beneath the game engine because the engine supplies none. Both are language-agnostic; only one is safe by construction.

The pattern worth generalising: **precise metering requires shared address space, and shared address space requires continuous security engineering.** Every system in this chapter resolves that tension by picking one and accepting the consequence. The interesting question, taken up in §8, is whether a mechanism exists that offers metering *without* shared ambient authority.

### 7.3 What the Open Record Cannot Show: Closed Judges

One prominent programmable-game platform is deliberately excluded from the analysis above. CodinGame publishes `CodinGame/codingame-game-engine`, a Java SDK for authoring games that run on its platform. [Source](https://github.com/CodinGame/codingame-game-engine) The SDK is open source and useful for understanding how a game's rules, referee, and visual replay are defined.

The platform's *execution sandbox* — the judge that compiles and runs submitted code in dozens of languages under resource limits — is not open source, and this chapter makes no claims about it. It is noted here only so that its absence from §7.1 is not read as an omission: a comparison table of sandbox architectures can only contain systems whose sandboxes can be read.

For completeness, several other programmable or code-adjacent games are out of scope as case studies: commercial titles where the sandbox is not published, defunct research projects whose code is no longer retrievable, and single-player games that present programming as a puzzle theme without executing untrusted code against other players.

---

## 8. WebAssembly as the Successor Sandbox

Screeps already runs WebAssembly, but not as a replacement for its sandbox. `rustyscreeps/screeps-game-api` (MIT) provides typed bindings to the Screeps in-game API for Rust AIs compiled to WebAssembly. [Source](https://github.com/rustyscreeps/screeps-game-api)

The layering is worth stating precisely, because it is easy to get backwards: a Rust bot compiles to a Wasm module, that module is instantiated **inside** the player's V8 isolate, and it reaches the game through JavaScript bindings. The isolate remains the trust boundary. WebAssembly here is a *compilation target within* the sandbox, not the sandbox itself — the server's guarantees still come from `isolated-vm`, `memoryLimit`, and the cleanup script of §2.3.

That said, the properties that make Wasm attractive as a compilation target are the same ones that would make it a stronger *primary* boundary, and they map onto §1.2's four bounds more cleanly than any mechanism in §2–§6:

**Ambient authority is zero by default.** A Wasm module can only call what the host explicitly places in its import object. There is no global object to strip, no `require` to shadow, no filesystem or socket API unless one is imported. This is a categorically different situation from §2.3: the failure mode of a forgotten `delete` has no analogue, because capabilities are enumerated rather than removed. For a graphics stack, the relevant corollary is that a Wasm module has no path to a device node — no `/dev/dri` access, no GPU submission — unless the host builds and passes one.

**Memory is a linear buffer the host owns.** A module's memory is a host-allocated region with bounds-checked access, so a maximum size is a property of instantiation rather than an advisory limit. Compare the honest caveat in `isolated-vm`'s documentation that `memoryLimit` is more of a guideline than a strict cap. [Source](https://github.com/laverdet/isolated-vm)

**Execution can be metered deterministically.** Wasmtime supports fuel-based metering: with `Config::consume_fuel` enabled, a `Store` is given a fuel budget via `Store::set_fuel`, most Wasm instructions consume one unit of fuel, the remaining amount is readable with `Store::get_fuel`, and exhausting the budget traps. [Source](https://docs.wasmtime.dev/api/wasmtime/struct.Store.html)

Fuel is the interesting one. Screeps' CPU accounting via `isolate.cpuTime` and V8's wall-clock `timeout` is *wall-clock-derived*, and wall-clock measurements are not reproducible: the same script on a busier machine consumes more of its budget. Fuel is an instruction count, so it is identical across hosts and across runs. For a competitive game that is a materially better primitive — a bot's cost becomes a property of the bot rather than of the server's current load, and a match becomes replayable bit-for-bit. Trapping on exhaustion also gives the host a clean, in-band termination, which is structurally similar to Screeps' `_halt` reference disposing the isolate, but without needing a host closure injected into the guest's global namespace in the first place.

The tradeoffs are real and should not be glossed. Wasm's language support, while broad, is narrower than "anything that speaks HTTP" or "anything the host can `execl`" — the Battlesnake and Halite models remain strictly more permissive. Instruction-count fuel is not proportional to real time, so a fuel budget is a fairness metric rather than a latency guarantee, and a host that also needs bounded wall-clock latency must still impose a deadline. And a host embedding Wasm has to design its import surface deliberately, which is real work — though it is *design-time* work with an enumerable result, rather than the open-ended review burden of keeping an ambient global namespace clean.

The practical toolchain for this pattern is covered elsewhere in this Part: Extism packages the embed-a-Wasm-plugin problem as a set of host SDKs over a Wasm runtime, and is the shape most projects reach for rather than embedding Wasmtime directly. [Source](https://github.com/extism/extism) Chapter 244d treats those mod-sandboxing patterns in detail; Chapter 98 treats the compilation and deployment model from the browser side.

The summary claim, offered as analysis rather than as a documented property of any shipping game: WebAssembly is the first mechanism in this chapter's survey that decouples *metering* from *shared ambient authority*. Every system in §7.1 had to choose between them. A Wasm-based competitive-code sandbox would not.

---

## Integrations

**Chapter 98 — WebAssembly and WebGPU as a Deployment Target** supplies the runtime model that §8 argues is the architectural successor to Screeps' V8-isolate sandbox. Screeps' `isolated-vm` design and a browser's Wasm sandbox solve the same problem with the same lineage — both descend from V8's isolate and linear-memory machinery — but Chapter 98's capability-import model replaces §2.3's inject-then-revoke global cleanup with an enumerable import surface, and its treatment of how a Wasm module reaches WebGPU (and only what the host hands it) is the concrete answer to why a Wasm guest has no path to `/dev/dri`.

**Chapter 244d — Modding Architectures: Scripting, Sandboxing, and Hot-Reload** covers the Extism and Wasmtime embedding patterns that §8 identifies as the practical realisation of a fuel-metered, import-scoped sandbox. The mod-loading problem is the same problem as competitive-code execution with one constraint relaxed — mod authors are semi-trusted, competitors are not — so the fuel budgeting, host-function design, and hot-reload machinery described there transfer directly to a programmable game, with the metering thresholds tightened.

**Chapter 210 — GPU Physics** connects directly to §4.2, and SimSpark's own documentation makes the link explicit: the abstract physics layer exists so that the engine is swappable, ODE sits behind that interface as a plugin, and the wiki records in-progress Bullet work motivated specifically by the prospect of GPU acceleration. [Source](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Abstract-Physical-Layer) Chapter 210's treatment of GPU rigid-body and contact solving is therefore the continuation of a migration the 3D league has already begun, and it explains the ceiling that motivates it: a 20 ms cycle solving 22 controllable joints per humanoid across a full match, with contact forces surfaced to every agent through the `(FRP …)` perceptor.

**Chapter 40 — Bevy and wgpu** is the Rust-native comparison point for §2.6's renderer. Screeps' graphics engine is a PixiJS 7 scene graph over WebGL, dispatching per-object-type through an externalised metadata table so that private servers can render custom object types without forking the renderer. A Bevy equivalent would express the same decoupling through ECS components and `wgpu` render pipelines, with the metadata table becoming component composition — and since `rustyscreeps/screeps-game-api` already compiles Rust bots to Wasm for this game, the Rust-side simulation client and a Rust-side renderer would share a type vocabulary that the current JavaScript/PixiJS split cannot.

---

## References

**Screeps: World**

- [screeps/screeps](https://github.com/screeps/screeps) — private-server package, ISC (§2.1)
- [screeps/engine](https://github.com/screeps/engine) — tick processing and intent resolution, ISC (§2.1)
- [screeps/driver](https://github.com/screeps/driver) — storage, sandbox, and native modules, ISC (§2.1)
- [screeps/driver — package.json](https://github.com/screeps/driver/blob/master/package.json) — `isolated-vm` git-commit pin, `node-gyp` install hook (§2.1)
- [screeps/driver — lib/runtime/user-vm.js](https://github.com/screeps/driver/blob/master/lib/runtime/user-vm.js) — per-player `ivm.Isolate`, snapshot, capability injection and cleanup, heap metrics, idle eviction (§2.2, §2.3, §2.4)
- [screeps/driver — lib/runtime/runtime-driver.js](https://github.com/screeps/driver/blob/master/lib/runtime/runtime-driver.js) — `evalCode` script timeout, module wrapper, timeout error remap (§2.4)
- [screeps/driver — lib/runtime/runtime.js](https://github.com/screeps/driver/blob/master/lib/runtime/runtime.js) — `isolate.cpuTime`, prototype stripping, intent pricing, RawMemory 2 MB cap (§2.3, §2.4, §2.5)
- [laverdet/isolated-vm](https://github.com/laverdet/isolated-vm) — V8 isolate API for Node; `memoryLimit` documented as a guideline (§2.2, §8)
- [screeps/renderer](https://github.com/screeps/renderer) — open-source graphics engine, ISC (§2.6)
- [screeps/renderer — engine/package.json](https://github.com/screeps/renderer/blob/master/engine/package.json) — PixiJS `^7.4.2` dependency (§2.6)
- [screeps/renderer — README](https://github.com/screeps/renderer/blob/master/README.md) — `applyState` pipeline, metadata lookup, processor taxonomy (§2.6)
- [Open source graphics engine — Screeps blog, 2018-08-24](https://blog.screeps.com/2018/08/renderer/) — release announcement; PixiJS basis; private-server and third-party-GUI use cases (§2.6)
- [screeps/launcher — example renderer mods](https://github.com/screeps/launcher/tree/ptr/init_dist/example-mods/renderer) — custom object-type graphics (§2.6)
- [rustyscreeps/screeps-game-api](https://github.com/rustyscreeps/screeps-game-api) — typed Rust→Wasm bindings to the in-game API, MIT (§8)
- [Arena and World Roadmap 2026 — Steam, 2025-12-15](https://store.steampowered.com/news/app/464350/view/1818752592133871) — seasonal cadence, Power Creep classes, Warp Containers, premium fast-tick shard, runtime update (§Roadmap)
- [Node.js v24 upgrade — Steam, 2026-04-01](https://store.steampowered.com/news/app/464350/view/1828894815555845) — all runtime servers moved to Node.js v24; Pixi v7 and backported Arena renderer fixes (§Roadmap)
- [New shard + Unlimited Access Keys Pack — Steam, 2026-04-23](https://store.steampowered.com/news/app/464350/view/1830797770234042) — shardX at 2000 ticks/hour, `Game.shard.access`/`accessTime`/`activateAccess`, `ERR_ACCESS_DENIED` (§Roadmap)
- [10th Anniversary and Free Season — Steam, 2026-05-28](https://store.steampowered.com/news/app/464350/view/1833968530888223) — decorations, Terminal Nodes, two new Power Creep classes, a new Screeps IDE (§Roadmap)
- [Screeps: Arena — Season 3 starts on May 1st, Steam, 2026-04-26](https://store.steampowered.com/news/app/1137320/view/1830797770240825) — three-month Arena season cadence in practice (§Roadmap)

**Robocode**

- [robo-code/robocode](https://github.com/robo-code/robocode) — classic Robocode (§3)
- [robo-code/robocode — LICENSE.txt](https://github.com/robo-code/robocode/blob/master/LICENSE.txt) — Eclipse Public License v1.0 (§3)
- [robo-code/robocode — versions.md](https://github.com/robo-code/robocode/blob/master/versions.md) — v1.9.5.5 and v1.10.0 notes on the Security Manager removal (§3.1)
- [JEP 411: Deprecate the Security Manager for Removal](https://openjdk.org/jeps/411) (§3.1, §7.2)
- [JEP 486: Permanently Disable the Security Manager](https://openjdk.org/jeps/486) (§3.1)
- [robocode-dev/tank-royale](https://github.com/robocode-dev/tank-royale) — Kotlin rewrite, Apache-2.0, v1.0.0 May 2026 (§3.2)
- [tank-royale — schema/schemas](https://github.com/robocode-dev/tank-royale/tree/master/schema/schemas) — JSON Schema 2020-12 protocol definitions (§3.3)
- [tank-royale — bot-handshake.schema.yaml](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/bot-handshake.schema.yaml) — identity, `secret`, `isDroid`, required fields (§3.3)
- [tank-royale — server-handshake.schema.yaml](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/server-handshake.schema.yaml) — `behaviorVersion`, `features` (§3.3)
- [tank-royale — bot-intent.schema.yaml](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/bot-intent.schema.yaml) — delta semantics, firepower bounds, `stdOut`/`stdErr` (§3.3)
- [tank-royale — tick-event-for-bot.schema.yaml](https://github.com/robocode-dev/tank-royale/blob/master/schema/schemas/tick-event-for-bot.schema.yaml) — per-turn state, all fields required (§3.3)
- [Tank Royale — My First Bot for the JVM](https://robocode.dev/tutorial/jvm/my-first-bot-for-jvm.html) — bot skeleton, launch script, JSON descriptor (§3.2)
- [robo-code/robocode — releases](https://github.com/robo-code/robocode/releases) — classic still shipping: v1.10.3 (2026-05-18), v1.11.0 (2026-06-06), v1.11.1 (2026-07-13) (§Roadmap)
- [robocode-dev/tank-royale — releases](https://github.com/robocode-dev/tank-royale/releases) — v1.0.0 (2026-05-03), v1.0.2 (2026-05-18) (§Roadmap)
- [tank-royale issue #232 — Release Tank Royale 1.1.0 with TwinDuel GUI](https://github.com/robocode-dev/tank-royale/issues/232) — opened 2026-08-04 (§Roadmap)
- [tank-royale issue #12 — Bring original robocode bots to new platform](https://github.com/robocode-dev/tank-royale/issues/12) — labelled *in progress* / *huge effort* (§Roadmap)
- [tank-royale issue #100 — Booter to connect to a remote game server](https://github.com/robocode-dev/tank-royale/issues/100) — open community request (§Roadmap)
- [tank-royale issue #106 — Server websocket over wss](https://github.com/robocode-dev/tank-royale/issues/106) — open community request, labelled *huge effort* (§Roadmap)

**RoboCup Soccer Simulation**

- [rcsoccersim/rcssserver](https://github.com/rcsoccersim/rcssserver) — 2D simulator, C++, LGPL-3.0 (§4.1)
- [rcssserver — src/serverparam.cpp](https://github.com/rcsoccersim/rcssserver/blob/master/src/serverparam.cpp) — default ports 6000 / 6001 / 6002 (§4.1)
- [RoboCup Soccer Simulator documentation](https://rcsoccersim.readthedocs.io/en/latest/) — `(init …)`, `(dash …)`, `(see …)`, `(sense_body …)` protocol (§4.1)
- [SimSpark (GitLab)](https://gitlab.com/robocup-sim/SimSpark) — generic physical simulator hosting `rcssserver3d` (§4.2)
- [SimSpark documentation portal](https://robocup-sim.gitlab.io/SimSpark/) — project landing page and wiki index (§4.2)
- [SimSpark wiki — About SimSpark](https://gitlab.com/robocup-sim/SimSpark/-/wikis/About-SimSpark) — generic 3D multiagent simulator; official RoboCup 3D simulation server (§4.2)
- [SimSpark wiki — Network Protocol](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Network-Protocol) — TCP 3100, 32-bit network-order length prefix, ASCII S-expressions, create-then-init handshake (§4.2)
- [SimSpark wiki — Abstract Physical Layer](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Abstract-Physical-Layer) — pluggable physics, ODE plugin, in-progress Bullet work for GPU acceleration (§4.2, Integrations)
- [SimSpark wiki — Simulation Update Loop](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Simulation-Update-Loop) — simcontrol nodes, 20 ms cycle, reproducibility explicitly not guaranteed (§4.2, §7.2)
- [SimSpark wiki — Monitor](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Monitor) — separate rendering client, snapshot/incremental Monitor Format, overlay plugins, log playback (§4.2)
- [SimSpark wiki — GUI Monitor Frame](https://gitlab.com/robocup-sim/SimSpark/-/wikis/GUI-Monitor-Frame) — `SparkGLWidget` OpenGL rendering of the scene graph (§4.2)
- [SimSpark wiki — Effectors](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Effectors) — `(scene …)`, `(init …)`, `(beam …)`, joint effectors, `(say …)`, `(syn)` (§4.2)
- [SimSpark wiki — Perceptors](https://gitlab.com/robocup-sim/SimSpark/-/wikis/Perceptors) — `HJ`, `GYR`, `ACC`, `FRP`, `TCH`, `See`, `GS`, `AgentState`, `hear` (§4.2)
- [Open Dynamics Engine](https://www.ode.org/) — rigid-body dynamics library used by SimSpark (§4.2)
- [RoboCup Federation — Objective](https://www.robocup.org/objective) — 1997 founding mission: beat the human World Cup champions by 2050 (§Roadmap)
- [RoboCup 2026 — Incheon, South Korea](https://2026.robocup.org/) — first Korean hosting of the annual competition, July 2026 (§Roadmap)
- [rcssserver — releases](https://github.com/rcsoccersim/rcssserver/releases) — last protocol release `rcssserver-19.0.0`, 2024-03-25 (§Roadmap)
- [rcssserver issue #90 — UDP ↔ WebRTC bridge](https://github.com/rcsoccersim/rcssserver/issues/90) — open since 2022, browser-direct client connections (§Roadmap)
- [rcssserver issue #148 — AppImage and official Docker container release](https://github.com/rcsoccersim/rcssserver/issues/148) — open packaging request (§Roadmap)
- [rcsoccersim/RoboCup2026](https://github.com/rcsoccersim/RoboCup2026) — 2D competition logs and binaries, published July 2026 (§Roadmap)
- [SimSpark — tags](https://gitlab.com/robocup-sim/SimSpark/-/tags) — `SIMSPARK_0.3.8_RELEASE` and `RCSSSERVER3D_0.7.9_RELEASE`, both 2026-01-30 (§Roadmap)

**Battlesnake**

- [BattlesnakeOfficial/rules](https://github.com/BattlesnakeOfficial/rules) — Go rules engine and CLI, AGPL-3.0 (§5)
- [BattlesnakeOfficial/rules — cli/commands/play.go](https://github.com/BattlesnakeOfficial/rules/blob/main/cli/commands/play.go) — 500 ms default request timeout (§5)
- [Battlesnake API — webhooks](https://docs.battlesnake.com/api/webhooks) — `GET /`, `POST /start`, `/move`, `/end` (§5)
- [BattlesnakeOfficial/board](https://github.com/BattlesnakeOfficial/board) — Svelte game board and playback control, AGPL-3.0 (§5)
- [BattlesnakeOfficial/arena](https://github.com/BattlesnakeOfficial/arena) — Rust rewrite of `play.battlesnake.com`; Axum + PostgreSQL web service with its own built-in Rust rules crate (§5, §Roadmap)
- [BattlesnakeOfficial/rules — releases](https://github.com/BattlesnakeOfficial/rules/releases) — last release v1.2.3, 2023-02-10; superseded as the ranked-play engine by `arena` (§5, §Roadmap)
- [BattlesnakeOfficial/rfcs](https://github.com/BattlesnakeOfficial/rfcs) — public RFC and specification process opened January 2026 (§Roadmap)
- [BattlesnakeOfficial/snake-zoo](https://github.com/BattlesnakeOfficial/snake-zoo) — Rust CLI and Dockerfile registry for running community snakes locally (§Roadmap)

**Halite III**

- [HaliteChallenge/Halite-III](https://github.com/HaliteChallenge/Halite-III) — game engine and starter kits, MIT (§6)
- [Halite-III — game_engine/networking/unix/UnixConnection.cpp](https://github.com/HaliteChallenge/Halite-III/blob/master/game_engine/networking/unix/UnixConnection.cpp) — `fork`/`execl("/bin/sh")`, `setpgid`, `dup2`, `kill(-pgid, SIGKILL)`, stderr cap (§6)
- [Halite-III — starter_kits/Python3/MyBot.py](https://github.com/HaliteChallenge/Halite-III/blob/master/starter_kits/Python3/MyBot.py) — turn loop; stdout reserved for the protocol (§6)
- [Halite-III — starter_kits/README.md](https://github.com/HaliteChallenge/Halite-III/blob/master/starter_kits/README.md) — kit layout, `install.sh`, ten-minute build allowance (§6)
- [Halite-III — libhaliteviz](https://github.com/HaliteChallenge/Halite-III/tree/master/libhaliteviz) — HTML5 canvas replay viewer (§6)
- [Halite-III — commit history](https://github.com/HaliteChallenge/Halite-III/commits/master) — last substantive commit 2020-02-08; Dependabot merges only since (§Roadmap)
- [Halite-III — releases](https://github.com/HaliteChallenge/Halite-III/releases) — no tagged release has ever been published (§Roadmap)

**WebAssembly sandboxing and other platforms**

- [Wasmtime — `wasmtime::Store` API](https://docs.wasmtime.dev/api/wasmtime/struct.Store.html) — `Config::consume_fuel`, `set_fuel`, `get_fuel`, trap on exhaustion (§8)
- [extism/extism](https://github.com/extism/extism) — cross-language Wasm plugin framework (§8)
- [Wasmtime — Release Process](https://docs.wasmtime.dev/stability-release.html) — major version on the 20th of each month; every 12th release is an LTS supported for 24 months (§Roadmap)
- [wasmtime PR #12541 — Add support for fine-grained operator costs](https://github.com/bytecodealliance/wasmtime/pull/12541) — per-operator fuel cost configuration, merged 2026-02-19 (§Roadmap)
- [wasmtime PR #13393 — Insert fuel/epoch checks in bulk copies/fills](https://github.com/bytecodealliance/wasmtime/pull/13393) — fuel proportional to bulk-transfer size, merged 2026-05-18 (§Roadmap)
- [wasmtime PR #13612 — Update WASI to 0.3.0, enable component-model-async](https://github.com/bytecodealliance/wasmtime/pull/13612) — merged 2026-06-12 (§Roadmap)
- [wasmtime PR #13558 — Remove wasi-threads and wasi-common](https://github.com/bytecodealliance/wasmtime/pull/13558) — merged 2026-06-05 (§Roadmap)
- [CodinGame/codingame-game-engine](https://github.com/CodinGame/codingame-game-engine) — open-source Java SDK for authoring CodinGame games; the platform's execution judge is not open source (§7.3)

---

## Roadmap

The five platforms dissected above are not frozen artefacts. Four of them are under active development and one is not, and the direction each is moving in sharpens rather than blurs the chapter's central argument: the sandbox boundary keeps migrating downward, out of the language runtime and into the process, the container, and increasingly the Wasm engine.

### Near-term (6–12 months)

- **Screeps: World is importing Arena's runtime.** The published 2026 roadmap commits to bringing Arena's execution model into World — ES module support, folder support, built-in JS definitions, native script uploads through the Steam client, and an official VSCode/vscode.dev extension carrying editor, console, and memory views. The Node.js half of that work already landed: all runtime servers were moved to Node.js v24 on 2026-04-01, in a release that also pulled the isolate host, the PixiJS 7 renderer, and the backported Arena room-renderer fixes forward together. [Source](https://store.steampowered.com/news/app/464350/view/1818752592133871) [Source](https://store.steampowered.com/news/app/464350/view/1828894815555845) (§2.1, §2.4, §2.6)
- **Arena finishes its pipeline before World work resumes.** Two major Arena updates — Trophies and Decorations — are stated as remaining before development focus shifts back to World, with Arena running a new season every three months; Season 2 opened 2026-02-01 and Season 3 on 2026-05-01. Readers tracking §2's isolate-per-player design should expect World changes to trail Arena's by roughly that interval, because the runtime ideas are being proven in Arena first. [Source](https://store.steampowered.com/news/app/464350/view/1818752592133871) [Source](https://store.steampowered.com/news/app/1137320/view/1830797770240825) (§2.2)
- **Robocode ships two lines in parallel, not one.** §3 traces the SecurityManager collapse into Tank Royale's process-isolated rewrite, but classic Robocode has not been retired: v1.10.3 (2026-05-18), v1.11.0 (2026-06-06), and v1.11.1 (2026-07-13) followed Tank Royale's own v1.0.0 (2026-05-03) and v1.0.2 (2026-05-18). The in-JVM and out-of-process sandboxes are being maintained concurrently, so the migration described in §3.1 is an addition to the ecosystem rather than a replacement of it. [Source](https://github.com/robo-code/robocode/releases) [Source](https://github.com/robocode-dev/tank-royale/releases) (§3.1, §3.2)
- **Tank Royale 1.1.0 and a bridge for classic bots.** A maintainer-opened tracking issue targets 1.1.0 around a TwinDuel GUI, and the long-running request to run original Robocode bots on the new platform is labelled *in progress* and *huge effort* — the compatibility shim that would let the EPL-licensed classic bot corpus survive the move to the WebSocket protocol of §3.3. [Source](https://github.com/robocode-dev/tank-royale/issues/232) [Source](https://github.com/robocode-dev/tank-royale/issues/12) (§3.2, §3.3)
- **The Go rules engine's role is narrowing to reference implementation and CLI.** `BattlesnakeOfficial/rules` has taken no tagged release since v1.2.3 (2023-02-10), while ranked play has already moved to the Rust `arena` engine described in §5. Whether the Go engine and its CLI stay in sync with `arena`'s rules as the RFC process below formalises game-mode semantics is the open question — a divergence would leave local `battlesnake play` testing unrepresentative of ranked outcomes. [Source](https://github.com/BattlesnakeOfficial/rules/releases) [Source](https://github.com/BattlesnakeOfficial/arena) (§5, §7.1)
- **Wasmtime's fuel accounting is becoming finer-grained.** §8 presents fuel as a portable instruction count; two changes refine exactly what gets counted. Per-operator cost configuration merged 2026-02-19, letting an embedder price individual instructions rather than charging uniformly, and bulk memory copies and fills now consume fuel proportional to the size of the operation (merged 2026-05-18) where previously "a constant amount of fuel was consumed regardless" — though that change explicitly does not yet cover `memory.init` or `table.copy`. Anyone building a metered contest engine on Wasmtime should treat fuel as *configurable* and version-sensitive, not as a fixed universal constant. [Source](https://github.com/bytecodealliance/wasmtime/pull/12541) [Source](https://github.com/bytecodealliance/wasmtime/pull/13393) (§8)

### Medium-term (1–3 years)

- **A premium Screeps shard aiming at one-second ticks.** The roadmap describes a new shard gated behind Access Keys where ticks run "potentially even around the 1-second mark". The first instalment shipped on 2026-04-23 as shardX at 2000 ticks/hour against the standard 1000, with `Game.shard.access`, `Game.shard.accessTime`, and `Game.shard.activateAccess` added to the API and a new `ERR_ACCESS_DENIED` return from `claimController`, `reserveController`, and `upgradeController`. Halving the tick budget tightens every constraint in §2.4 — the isolate wake-up, the script timeout, and the CPU bucket all have to fit in less wall-clock time. [Source](https://store.steampowered.com/news/app/464350/view/1818752592133871) [Source](https://store.steampowered.com/news/app/464350/view/1830797770234042) (§2.4, §2.5)
- **New Screeps game objects and a first-party IDE.** Two Power Creep classes, Commander and Executor, were designed years ago but never released; alongside them the announced backlog covers Warp Containers, a Terminal Nodes mechanic, an improved decorations system, and a new Screeps IDE. The IDE matters most to §2.6's argument: the renderer was open-sourced precisely so tooling could be built against it without forking. [Source](https://store.steampowered.com/news/app/464350/view/1818752592133871) [Source](https://store.steampowered.com/news/app/464350/view/1833968530888223) (§2.5, §2.6)
- **Containerised and remote execution are the standing asks in both Robocode and Battlesnake.** Two open Tank Royale requests — a booter that connects to a remote game server, and serving the protocol over `wss://` — are community-filed with no maintainer commitment, but together they describe the deployment shape that reinforcement-learning users want: bots in containers, server elsewhere, transport encrypted. Battlesnake has already taken a step in that direction; its opponent-registry CLI runs community snakes locally from Docker images, the first containerised execution in the project's own tooling. [Source](https://github.com/robocode-dev/tank-royale/issues/100) [Source](https://github.com/robocode-dev/tank-royale/issues/106) [Source](https://github.com/BattlesnakeOfficial/snake-zoo) (§3.2, §5, §7.2)
- **Battlesnake is formalising its rules through a public RFC process.** A dedicated repository for RFCs and specifications opened in January 2026, intended to pin down game-mode semantics that currently live only in engine source. No RFC has been accepted yet, so the normative description of the ruleset remains the Go implementation cited in §5. [Source](https://github.com/BattlesnakeOfficial/rfcs) (§5)
- **Wasmtime is standardising on the component model and shedding legacy sandbox surface.** WASI 0.3.0 and asynchronous component-model support were enabled by default in a change merged 2026-06-12, and `wasi-threads` together with the older `wasi-common` crate were removed outright a week earlier. The direction is the one §8 identifies: a single typed, capability-explicit interface boundary replacing a scatter of ad-hoc host hooks — but embedders pinning older Wasmtime majors should note that the removal is a hard break, not a deprecation. [Source](https://github.com/bytecodealliance/wasmtime/pull/13612) [Source](https://github.com/bytecodealliance/wasmtime/pull/13558) (§8)
- **The RoboCup 2D and 3D servers are diverging in maintenance tempo.** `rcssserver` has published no protocol release since `rcssserver-19.0.0` on 2024-03-25, and its open feature requests — a UDP-to-WebRTC bridge that would let browsers speak the S-expression protocol of §4.1 directly, and official Docker/AppImage packaging — remain unactioned. SimSpark, by contrast, tagged `SIMSPARK_0.3.8_RELEASE` and `RCSSSERVER3D_0.7.9_RELEASE` together on 2026-01-30, so the 2D protocol should be treated as stable-by-inertia while the 3D simulator is where engine work is happening. [Source](https://github.com/rcsoccersim/rcssserver/releases) [Source](https://github.com/rcsoccersim/rcssserver/issues/90) [Source](https://gitlab.com/robocup-sim/SimSpark/-/tags) (§4.1, §4.2)

### Long-term

- **RoboCup's 2050 Grand Challenge remains the organising goal.** The federation's stated mission, set when it was established in 1997, is to field a team of robots capable of winning against the human soccer World Cup champions by 2050. The simulation leagues of §4 are the cheap, fast testbed for that programme — no hardware, no batteries, thousands of matches per night — which is why their protocols evolve conservatively while the physical leagues absorb the rule churn. The 2026 competition was held in Incheon, South Korea, the first Korean hosting, and the 2D league's match logs and binaries are published as a public archive. [Source](https://www.robocup.org/objective) [Source](https://2026.robocup.org/) [Source](https://github.com/rcsoccersim/RoboCup2026) (§4)
- **Halite III has no roadmap: development ceased in 2020.** The repository has never published a tagged release, its last substantive commit is dated 2020-02-08, and everything merged since has been automated dependency bumps. The competition site no longer exists — `halite.io` issues a cross-host redirect to its former sponsor's corporate homepage. Readers should treat §6 as an autopsy of a technique rather than a guide to a running platform: the `fork`/`execl` process-group sandbox is worth studying precisely because it is small, complete, and finished. [Source](https://github.com/HaliteChallenge/Halite-III/commits/master) [Source](https://github.com/HaliteChallenge/Halite-III/releases) (§6)
- **WebAssembly is converging on the role each platform built separately.** Every sandbox in §7.1 was engineered in isolation: a V8 isolate here, a JVM security policy there, a process group, a network socket, an HTTP timeout. Wasmtime now ships a major version on the 20th of each month with every twelfth release designated LTS and supported for 24 months — a support model that makes a Wasm engine a defensible long-lived dependency for a contest platform rather than a moving research target. Screeps already accepts Rust compiled to Wasm through community bindings; the plausible end state is that the *language* runtime stops being the trust boundary entirely and the Wasm engine becomes the only one, with fuel replacing five different notions of a time limit. [Source](https://docs.wasmtime.dev/stability-release.html) [Source](https://github.com/rustyscreeps/screeps-game-api) (§7.1, §7.2, §8)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
