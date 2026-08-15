# Chapter 205d: Modding Architectures — Scripting, Sandboxing, and Hot-Reload on Linux

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Engine developers designing a third-party extension or mod system, who need to choose between an embedded scripting runtime, a WebAssembly sandbox, and a native ABI. Also security-conscious developers evaluating what a sandbox model actually guarantees when the code being loaded is untrusted.
> **Status**: First draft — 2026-08-08

Every engine that survives contact with a community eventually grows an extension surface. The interesting question is not *whether* to have one but what the extension is allowed to touch. A mod that can register a new item type is a content author; a mod that can call into the renderer is a co-developer of the engine; a mod that can open a file descriptor is a process with the user's full authority. These are three different products with three different threat models, and the architecture chosen at the start determines which one ships.

This chapter surveys the architectures actually in production on Linux: embedded Lua via `mlua` and LuaJIT, with Luanti as the mature reference implementation of a full-fledged Lua modding API; WebAssembly mod sandboxes via Extism and Veloren's plugin system; .NET IL patching via Lib.Harmony and BepInEx, which is what modders reach for when the engine deliberately offers no extension point; and engine-native shared-library extension via Godot's GDExtension, contrasted with Bevy's decision to remove its dynamic-plugin path outright.

The through-line is a security argument developed concretely in §5: WebAssembly's linear-memory isolation plus WASI capability gating gives a mod loader something that `dlopen` structurally cannot, which is a mod with no ambient access to a file descriptor, a syscall, or a GPU handle. Everything a WASM guest can reach, the host handed it deliberately. Everything a native mod can reach, the process can reach. That difference is the whole design space.

---

## Table of Contents

- [1. The Extension-Surface Problem](#1-the-extension-surface-problem)
  - [1.1 Three Axes: Trust, ABI Stability, Hot-Reload](#11-three-axes-trust-abi-stability-hot-reload)
  - [1.2 The Native `dlopen` Baseline](#12-the-native-dlopen-baseline)
- [2. Embedded Lua: mlua and LuaJIT](#2-embedded-lua-mlua-and-luajit)
  - [2.1 mlua as the Rust↔Lua Binding](#21-mlua-as-the-rustlua-binding)
  - [2.2 rlua's Deprecation into a Shim](#22-rluas-deprecation-into-a-shim)
  - [2.3 A Language Sandbox Is Not an OS Sandbox](#23-a-language-sandbox-is-not-an-os-sandbox)
- [3. Luanti: A Production Lua Modding API](#3-luanti-a-production-lua-modding-api)
  - [3.1 Engine and Scripting Substrate](#31-engine-and-scripting-substrate)
  - [3.2 The Registration-Time API](#32-the-registration-time-api)
  - [3.3 Node and Item Definitions](#33-node-and-item-definitions)
  - [3.4 Formspecs: Declarative UI Shipped Over the Network](#34-formspecs-declarative-ui-shipped-over-the-network)
  - [3.5 Environment and Player Callbacks](#35-environment-and-player-callbacks)
  - [3.6 Luanti's Capability Gate: Trusted Mods and the Insecure Environment](#36-luantis-capability-gate-trusted-mods-and-the-insecure-environment)
- [4. WebAssembly Mod Sandboxing](#4-webassembly-mod-sandboxing)
  - [4.1 Extism as a General Plugin Framework](#41-extism-as-a-general-plugin-framework)
  - [4.2 Veloren's WASM Plugin System](#42-velorens-wasm-plugin-system)
  - [4.3 wasmtime and wasmer as Capability-Gated Hosts](#43-wasmtime-and-wasmer-as-capability-gated-hosts)
- [5. Why Linear Memory Beats dlopen](#5-why-linear-memory-beats-dlopen)
  - [5.1 The Linear Memory Model, Concretely](#51-the-linear-memory-model-concretely)
  - [5.2 Control-Flow Integrity and the Unreachable Callstack](#52-control-flow-integrity-and-the-unreachable-callstack)
  - [5.3 WASI: No Ambient Authority](#53-wasi-no-ambient-authority)
  - [5.4 Resource Exhaustion: Fuel and Epochs](#54-resource-exhaustion-fuel-and-epochs)
  - [5.5 The Native Contrast](#55-the-native-contrast)
- [6. .NET IL Patching: Harmony and BepInEx](#6-net-il-patching-harmony-and-bepinex)
  - [6.1 What a Harmony Patch Does at the IL Level](#61-what-a-harmony-patch-does-at-the-il-level)
  - [6.2 Prefixes, Postfixes, and Injected Parameters](#62-prefixes-postfixes-and-injected-parameters)
  - [6.3 Transpilers: Rewriting the Instruction Stream](#63-transpilers-rewriting-the-instruction-stream)
  - [6.4 Finalizers](#64-finalizers)
  - [6.5 BepInEx: Preloader, HarmonyX, and IL2CPP](#65-bepinex-preloader-harmonyx-and-il2cpp)
- [7. Engine-Native Extension](#7-engine-native-extension)
  - [7.1 Godot 4 GDExtension](#71-godot-4-gdextension)
  - [7.2 Bevy: Dynamic Plugin Removal and Hot-Patching](#72-bevy-dynamic-plugin-removal-and-hot-patching)
  - [7.3 dexterous_developer: Dynamic-Library Hot-Reload for Bevy](#73-dexterous_developer-dynamic-library-hot-reload-for-bevy)
- [8. Comparison](#8-comparison)
- [9. Why There Is No Standard](#9-why-there-is-no-standard)
- [Integrations](#integrations)
- [References](#references)
- [Roadmap](#roadmap)

---

## 1. The Extension-Surface Problem

### 1.1 Three Axes: Trust, ABI Stability, Hot-Reload

Mod architectures are usually discussed as a language choice — "should we embed Lua or use WASM?" — but the language is downstream of three independent decisions.

**Trust.** Does the mod run with the process's authority, or with authority the host explicitly grants? This is a binary, and it is not negotiable after the fact. If a mod can call `dlsym` or `open`, no amount of code review at distribution time converts it into a sandbox, because the enforcement point does not exist. If a mod's only route to the outside world is a host function table, the host controls the entire surface by construction.

**ABI stability.** Can a mod compiled once keep working across engine versions? Text-based scripting trivially can, because the interface is a set of function names in a table. A native shared library can only do so if the engine commits to a stable ABI — a much stronger promise than a stable API, and one that a language like Rust does not currently support at all (§7.2).

**Hot-reload.** Can the mod change while the process runs? This axis splits into two use cases that are routinely conflated: the *developer loop*, where the author of the game wants to edit a system and see it live, and *mod distribution*, where an end user drops a third-party artifact into a folder. A solution to the first is not a solution to the second, because the developer loop can assume matching compiler versions and a cooperating build tool, while mod distribution cannot.

Each architecture in this chapter picks a different point in that cube, and the comparison table in §8 is organised along these axes rather than by language.

### 1.2 The Native `dlopen` Baseline

The simplest possible mod loader scans a directory, calls `dlopen` on each shared object, resolves an entry-point symbol, and calls it. This is worth stating explicitly because it is the baseline every other architecture is trying to improve on, and because it remains common.

Its properties follow directly from the dynamic linker's semantics, and they are unforgiving: the loaded object shares the address space and the descriptor table, can issue syscalls, can run code through ELF constructors before the engine's entry point, and is linked against the engine's internal layout. §5.5 sets those properties against WASM's in detail once the sandbox mechanisms are on the table.

ioquake3 is a working, GPL-2.0-or-later-licensed example of exactly this baseline, and it is worth reading because the mechanism is only a few lines of code once the platform macros are stripped away. The engine's loader macros resolve straight to `dlopen`/`dlsym` on dedicated-server builds:

```c
// code/sys/sys_loadlib.h
#ifdef DEDICATED
#	ifdef _WIN32
	/* ... LoadLibrary/GetProcAddress ... */
#	else
#	include <dlfcn.h>
#		define Sys_LoadLibrary(f) dlopen(f,RTLD_NOW)
#		define Sys_UnloadLibrary(h) dlclose(h)
#		define Sys_LoadFunction(h,fn) dlsym(h,fn)
#		define Sys_LibraryError() dlerror()
#	endif
#else
	/* normal client builds route through SDL_LoadObject/SDL_LoadFunction,
	   which is itself a thin dlopen/dlsym wrapper on Linux */
#endif
```
[Source](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/sys/sys_loadlib.h)

`Sys_LoadGameDll` calls those macros, resolves two named symbols out of the loaded object, and hands the mod a raw function pointer — there is no capability table, no permission check, and no mediation between this call and the mod's own code:

```c
// code/sys/sys_main.c
void *Sys_LoadGameDll(const char *name,
	vmMainProc *entryPoint,
	intptr_t (*systemcalls)(intptr_t, ...))
{
	void *libHandle;
	void (*dllEntry)(intptr_t (*syscallptr)(intptr_t, ...));

	libHandle = Sys_LoadLibrary(name);
	if (!libHandle) { /* ... error path ... */ }

	dllEntry = Sys_LoadFunction(libHandle, "dllEntry");
	*entryPoint = Sys_LoadFunction(libHandle, "vmMain");
	if (!*entryPoint || !dllEntry) { /* ... error path ... */ }

	dllEntry(systemcalls);
	return libHandle;
}
```
[Source](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/sys/sys_main.c)

On the mod side, the exported `dllEntry` symbol just stashes that function pointer in a file-static, and every `trap_*` call the mod makes is a call through it:

```c
// code/game/g_syscalls.c — compiled into the mod's .so, not the QVM bytecode target
static intptr_t (QDECL *syscall)( intptr_t arg, ... ) = (intptr_t (QDECL *)( intptr_t, ...))-1;

Q_EXPORT void dllEntry( intptr_t (QDECL *syscallptr)( intptr_t arg,... ) ) {
	syscall = syscallptr;
}

void	trap_Print( const char *text ) {
	syscall( G_PRINT, text );
}
```
[Source](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/game/g_syscalls.c)

That last file's own comment is the point: the same source compiles either into this native `.so` — sharing the process's address space, reachable by `dlsym`, bound by nothing but the two symbol names `dllEntry` and `vmMain` — or, with `Q3_VM` defined instead, into bytecode for ioquake3's sandboxed QVM interpreter. One source tree, two points on the trust axis; §5 covers the WASM equivalent of the sandboxed side.

The last of the properties listed above — layout coupling — is what usually kills such systems in practice, before the security properties become anyone's problem. Bevy's removal of its dynamic-plugin path (§7.2) is exactly this story.

---

## 2. Embedded Lua: mlua and LuaJIT

### 2.1 mlua as the Rust↔Lua Binding

For a Rust engine, `mlua` is the current standard binding. It describes itself as "high level Lua 5.5/5.4/5.3/5.2/5.1 (including LuaJIT) and Luau bindings to Rust with async/await support," is MIT-licensed, and supports "Lua 5.5, 5.4, 5.3, 5.2, 5.1 (including LuaJIT) and Luau" selected through mutually exclusive Cargo features `lua55`, `lua54`, `lua53`, `lua52`, `lua51`, `luajit`, and `luau` [Source](https://github.com/mlua-rs/mlua).

The breadth of that version matrix matters for a mod system, because the choice of Lua flavour is a performance and compatibility decision the engine makes once and the mod ecosystem then depends on forever. LuaJIT's tracing JIT is the reason it remains the default in game engines despite tracking Lua 5.1 semantics rather than the current language. Luau adds gradual typing and was designed for the same untrusted-script problem this chapter is about.

The core embedding shape is small:

```rust
// mlua README example
use mlua::prelude::*;

fn main() -> LuaResult<()> {
    let lua = Lua::new();

    let map_table = lua.create_table()?;
    map_table.set(1, "one")?;

    lua.globals().set("map_table", map_table)?;

    lua.load("for k,v in pairs(map_table) do print(k,v) end").exec()?;

    Ok(())
}
```

[Source](https://github.com/mlua-rs/mlua)

The `Lua` value owns an independent interpreter state. Everything a script can reach is what the host put into `lua.globals()`, which is the crucial architectural property: the default global table is the *only* ambient surface, and the host can replace it. Additional features relevant to engine integration include `async` — "mlua supports async/await for all Lua versions including Luau," implemented over Lua coroutines — plus `serde` for `serde`-based value conversion, `send` for making the Lua handle `Send`, `vendored` to build Lua from source rather than link a system library, and `userdata-wrappers` [Source](https://github.com/mlua-rs/mlua).

Coroutine-backed async is what allows a scripted mod to yield mid-tick without blocking the engine's schedule, which is the usual reason a naive `lua_pcall`-per-frame design fails once mods start doing anything slow.

*Note: needs verification* — a specific shipped commercial Rust game using `mlua` as its end-user modding runtime could not be confirmed from primary sources. `mlua`'s own documentation frames standalone mode as "adding scripting support to your application," and community write-ups discuss it for modding, but no vendor-confirmed shipping title was identified.

### 2.2 rlua's Deprecation into a Shim

`rlua` was the earlier Rust binding and is still widely referenced in older material. It should not be used for new work. The repository is archived read-only as of 2025-09-12, and states plainly: "rlua is now deprecated in favour of mlua: see below for migration information." Version 0.20 "includes some utilities to help transition to `mlua`, but is otherwise just re-exporting `mlua` directly" [Source](https://github.com/mlua-rs/rlua).

That is a genuine shim rather than a parallel implementation, so an engine on `rlua` 0.20 is already running `mlua` and the migration is mechanical.

### 2.3 A Language Sandbox Is Not an OS Sandbox

An embedded Lua interpreter is often described as sandboxed, and in a narrow sense it is: a script cannot forge a pointer, cannot execute arbitrary machine code through the language's own semantics, and cannot see a host value that was not placed in a table it can reach.

But those guarantees are *language-level*, and they hold only as long as the host actually restricts the global table. A stock Lua interpreter's standard library includes the `io` and `os` libraries, whose documented functions cover opening files and, via `os.execute`, handing "command to be executed by an operating system shell" [Source](https://www.lua.org/manual/5.4/manual.html#6.9). That is ambient authority in exactly the sense §1.1 defines: file and process access reachable from any script without the host granting anything. The isolation therefore depends on the engine's discipline in constructing the environment, not on a mechanism the runtime enforces. Two further leaks are worth naming because they are easy to miss:

- **Bytecode loading.** Lua's loaders accept precompiled bytecode, and the verifier is not a security boundary. Luanti's own documentation is explicit about the consequence in the context of deserialising data: functions loaded this way "cannot directly access the global environment," but "they could bypass this restriction with maliciously crafted Lua bytecode if mod security is disabled" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).
- **A shared environment across mods.** If all mods load into one interpreter state, mod A can reach mod B's globals and monkey-patch its functions. That is often a feature for modding, and simultaneously means the trust boundary is around the whole mod set rather than around each mod.

Luanti's answer to the first problem, and its explicit capability gate for the ambient-authority problem, are covered in §3.6. It is the most developed instance of a Lua host taking the sandbox question seriously, which is why it is worth examining in depth.

---

## 3. Luanti: A Production Lua Modding API

Luanti, formerly Minetest, is the best available reference for what a mature Lua modding API looks like after long-term maintenance. It is a voxel game engine rather than a single game — the shipped content is games and mods written against the Lua API — and it is LGPL-2.1-or-later licensed [Source](https://github.com/luanti-org/luanti). The API surface described below is the one third-party mods actually target, which makes it a useful concrete counterweight to abstract discussion of scripting design.

One naming note that affects every code example: the API namespace was `minetest.*` and is now `core.*`. The reference documentation states that "if you're looking for the `minetest` namespace (e.g. `minetest.something`), it's now called `core` due to the renaming of Luanti (formerly Minetest)" [Source](https://api.luanti.org/). Both spellings appear in the wild; new code uses `core`.

### 3.1 Engine and Scripting Substrate

The engine is C++ with a Lua scripting layer, and LuaJIT is a first-class build configuration rather than an afterthought. The top-level build file gates a pure-Lua bit-operations shim on the absence of LuaJIT:

```cmake
# CMakeLists.txt, luanti master
find_package(Lua REQUIRED)
if(NOT USE_LUAJIT)
	add_subdirectory(lib/bitop)
endif()
```

[Source](https://raw.githubusercontent.com/luanti-org/luanti/master/CMakeLists.txt)

`src/CMakeLists.txt` carries additional `USE_LUAJIT`-conditional logic, including a feature probe for `luaopen_string_buffer` used as a proxy for LuaJIT recency, with an in-tree comment noting that "LuaJIT provides exactly zero ways to determine how recent it is (the version is unchanged since 2017)" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/src/CMakeLists.txt). This is a small but honest detail about the cost of depending on LuaJIT: version detection is a hack because upstream stopped bumping the version string.

Mods are directories containing `init.lua` plus a `mod.conf`, and the reference documentation describes the load model directly: "Mods are loaded during server startup from the mod load paths by running the `init.lua` scripts in a shared environment" [Source](https://api.luanti.org/). "Shared environment" is the design choice discussed in §2.3 — mods can see and modify each other, deliberately.

The load model has a second consequence that shapes the entire API: mods are *server-side*, and the server transfers definitions and media to clients. A mod author registers a node with a texture name; the engine ships the texture and the node definition to every connected client. The mod itself never runs client-side rendering code and never touches a GPU resource. This is why Luanti's row in §8 shows no GPU reachability despite mods having full visual authorship: rendering is entirely declarative.

### 3.2 The Registration-Time API

The API is organised around a registration phase. The documentation heads the section with an unambiguous constraint — "Call these functions only at load time!" — and the environment registration list includes:

```text
* `core.register_node(name, nodedef)`
* `core.register_craftitem(name, itemdef)`
* `core.register_tool(name, tooldef)`
* `core.override_item(name, redefinition, del_fields)`
* `core.unregister_item(name)`
* `core.register_entity(name, entity definition)`
* `core.register_abm(abm definition)`
* `core.register_lbm(lbm definition)`
* `core.register_alias(alias, original_name)`
* `core.register_ore(ore definition)`
* `core.register_biome(biome definition)`
```

[Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md)

Two details in that list are load-bearing for anyone designing a comparable API.

First, `register_node` carries the note: "you must pass a clean table that hasn't already been used for another registration to this function, as it will be modified." The engine takes ownership of and mutates the definition table. That is an efficient choice in a Lua host — no deep copy per registration — and it is exactly the kind of aliasing hazard that a naive mod author trips over by defining one table and registering it twice.

Second, `core.override_item` and `core.unregister_item` exist:

```lua
core.override_item("default:mese", {light_source=core.LIGHT_MAX}, {"sounds"})
```

This "Overwrites the `light_source` field, removes the sounds from the definition of the mese block," where `redefinition` is "a table of fields `[name] = new_value`, overwriting fields of or adding fields to the existing definition" and `del_fields` is "a list of field names to be set to `nil`" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md). This is the sanctioned equivalent of what Harmony does by IL patching in §6 — mutating another party's definition — except that here it is a first-class API operation on a data table rather than a rewrite of compiled code. That contrast is the cleanest illustration in this chapter of why an engine with a real extension API does not need an IL patcher, and why one without gets one anyway.

`core.clear_craft` carries a comparable override semantics for recipes, and "Will erase existing craft based either on output item or on input recipe" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

### 3.3 Node and Item Definitions

A node definition is a plain Lua table. From the in-tree `devtest` game, which is the engine's own test content and therefore the most reliable style reference:

```lua
-- games/devtest/mods/basenodes/init.lua, luanti master
core.register_node("basenodes:stone", {
	description = "Stone",
	tiles = {"default_stone.png"},
	groups = {cracky=3},
})

core.register_node("basenodes:dirt_with_grass", {
	description = "Dirt with Grass",
	-- Using overlays here has no real merit here but we do it anyway so
	-- overlay-related bugs become more apparent in devtest.
	tiles = {"default_dirt.png"},
	overlay_tiles = {
		"default_grass.png",
		-- a little dot on the bottom to distinguish it from dirt
		"basenodes_dirt_with_grass_bottom.png",
		{name = "default_grass_side.png", tileable_vertical = false},
	},
	groups = {crumbly=3, soil=1},
})
```

[Source](https://raw.githubusercontent.com/luanti-org/luanti/master/games/devtest/mods/basenodes/init.lua)

The `modname:itemname` naming convention is enforced by the engine's registration namespace, which is how a mod ecosystem avoids collisions without a central registry. `groups` is the generic tagging mechanism the engine's interaction rules key off; `tiles` and `overlay_tiles` are texture-name strings, optionally tables carrying per-tile flags such as `tileable_vertical`.

Note what is absent. There is no shader, no material, no draw call, and no GPU handle anywhere in this definition. The mod names a texture and some flags; the engine's renderer decides what that means. Node definitions also carry `drawtype`, `paramtype`, and `paramtype2` fields whose value enumerations are documented in the API reference, including `paramtype2 = "color"` with an associated `palette` for per-node colourisation [Source](https://api.luanti.org/nodes/). Even the most visually specific thing a mod can say is a selection from an engine-defined enumeration, not code.

### 3.4 Formspecs: Declarative UI Shipped Over the Network

Mods need UI, and a server-side mod cannot draw. Luanti's answer is the formspec, which the reference describes candidly: "Formspec defines a menu. This supports inventories and some of the typical widgets like buttons, checkboxes, text input fields, etc. It is a string, with a somewhat strange format." A formspec "is made out of formspec elements, which includes widgets like buttons but also can be used to set stuff like background color," and "Many formspec elements have a `name`, which is a unique identifier which is used when the server receives user input" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

Elements are bracket-delimited with documented signatures, among them:

```text
formspec_version[<version>]
size[<W>,<H>,<fixed_size>]
field[<X>,<Y>;<W>,<H>;<name>;<label>;<default>]
button[<X>,<Y>;<W>,<H>;<name>;<label>]
button_exit[<X>,<Y>;<W>,<H>;<name>;<label>]
```

[Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md)

An inventory formspec from the reference's own examples:

```text
size[8,7.5]
image[1,0.6;1,2;player.png]
list[current_player;main;0,3.5;8,4;]
list[current_player;craft;3,0;3,3;]
list[current_player;craftpreview;7,1;1,1;]
```

[Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md)

The server pushes a formspec to a specific player:

```lua
core.show_formspec(player_name, "mymod:config", table.concat({
    "formspec_version[4]",
    "size[8,4]",
    "field[0.5,1;7,0.8;greeting;Greeting;Hello]",
    "button_exit[0.5,2.5;3,0.8;save;Save]",
}))
```

`core.show_formspec(playername, formname, formspec)` takes the "name of player to show formspec" and a `formname` that is the "name passed to `on_player_receive_fields` callbacks," which "should follow the `modname:<whatever>` naming convention" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md). `core.close_formspec(playername, formname)` closes it, and the `formname` "has to exactly match the one given in `show_formspec`, or the formspec will not close"; passing an empty formspec string to `show_formspec` is equivalent.

Input comes back as a callback:

```lua
core.register_on_player_receive_fields(function(player, formname, fields)
    if formname ~= "mymod:config" then return end
    if fields.save then
        -- fields.greeting holds the text field contents
    end
end)
```

The documentation enumerates precisely which events trigger this: "a button was pressed," "Enter was pressed while the focus was on a text field," "a checkbox was toggled," "something was selected in a dropdown list," "a different tab was selected," "selection was changed in a textlist or table," "an entry was double-clicked in a textlist or table," "a scrollbar was moved, or the form was actively closed by the player." Node metadata formspecs are excluded and instead use the node definition's own `on_receive_fields` [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

The formspec is a string protocol with an explicitly versioned dialect, and the version history is instructive about the cost of shipping UI over a network to heterogeneous clients. Version 2 (engine 5.1.0) "Forced real coordinates"; version 3 (5.2.0) made "Formspec elements are drawn in the order of definition" and enabled clipping by default for `box[]` and `image[]`. Ten versions had shipped by engine 5.13.0. `formspec_version[<version>]` "Must be specified before `size` element," and "Clients older than this version can neither show newer elements nor display elements with new arguments correctly" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

That last sentence is the architectural point. Because the mod's UI is data rather than code, an old client degrades by failing to render new widgets — it does not execute anything it does not understand. A mod system that shipped UI as client-side code would have no equivalent graceful-degradation story, and would have handed the mod a rendering context.

### 3.5 Environment and Player Callbacks

Beyond registration, mods attach to engine lifecycle events. Confirmed signatures include:

```lua
core.register_globalstep(function(dtime) end)
core.register_on_joinplayer(function(ObjectRef, last_login) end)
core.register_on_leaveplayer(function(ObjectRef, timed_out) end)
core.register_on_authplayer(function(name, ip, is_success) end)
```

For `register_on_joinplayer`, `last_login` is "The timestamp of the previous login, or nil if player is new." For `register_on_leaveplayer`, `timed_out` is "True for timeout, false for other reasons," and the callback "Does not get executed for connected players on shutdown." `register_on_authplayer` is "Called when a client attempts to log into an account" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

Deferred work uses `core.after(time, func, ...)`, which "returns job table" for later cancellation. Map interaction has its own asynchrony: `core.compare_block_status(pos, condition)` "Checks whether the mapblock at position `pos` is in the wanted condition," where conditions are `"unknown"` (not in memory), `"emerging"` (in the queue for loading from disk or generating), `"loaded"` (in memory but inactive, with no ABMs executed), and `"active"` (in memory and active), returning `nil` for an unsupported condition value [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

This is a small but revealing API. It tells mod authors that the world is not resident, and gives them a way to ask rather than a way to force. A mod system that let scripts synchronously demand arbitrary world state would be a denial-of-service surface; exposing the load state as a query is the sandbox-friendly shape.

### 3.6 Luanti's Capability Gate: Trusted Mods and the Insecure Environment

This is the part of Luanti most worth copying, because it is a Lua host implementing exactly the capability model that WASI formalises in §5.3 — and it demonstrates that the model is a design choice, not a property of WebAssembly.

By default, mods do not get ambient OS access. A mod that needs it must ask, at init time, and the request only succeeds if the server operator has named that mod in a configuration setting:

```lua
-- init.lua, mod main scope only
local ie = core.request_insecure_environment()
```

The reference specifies the semantics tightly. `core.request_insecure_environment()` "returns an environment containing insecure functions if the calling mod has been listed as trusted in the `secure.trusted_mods` setting or security is disabled, otherwise returns `nil`." It "Only works at init time and must be called from the mod's main scope (ie: the init.lua of the mod, not from another Lua file or within a function)." And the documentation shouts the consequence: "**DO NOT ALLOW ANY OTHER MODS TO ACCESS THE RETURNED ENVIRONMENT, STORE IT IN A LOCAL VARIABLE!**" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md)

Each of those restrictions is doing real work. Requiring init-time main-scope invocation means the grant cannot be obtained later from an arbitrary callback, which bounds when privilege escalation can occur and lets the engine identify the requesting mod unambiguously from the load context. Requiring the operator to list the mod in `secure.trusted_mods` puts the grant decision with the person who bears the risk. And the shouted warning about storing it in a local exists precisely because of the shared-environment property from §3.1: a trusted mod that leaks its insecure environment into a global has silently granted OS access to every other mod on the server.

Network access is gated separately and more finely, which is the mark of a capability system rather than a single privilege bit. `core.request_http_api()` "returns `HTTPApiTable` containing http functions if the calling mod has been granted access by being listed in the `secure.http_mods` or `secure.trusted_mods` setting, otherwise returns `nil`." It carries the same init-time main-scope restriction, the same all-caps warning against leaking the returned table, and one additional caveat: the "Function only exists if Luanti server was built with cURL support." The returned table provides `fetch`, `fetch_async`, and `fetch_async_get`, where `HTTPApiTable.fetch(HTTPRequest req, callback)` "Performs given request asynchronously and calls callback upon completion" and is recommended as the default — "Use this HTTP function if you are unsure, the others are for advanced use" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

Note the shape: a mod can be granted HTTP without being granted the filesystem. `secure.http_mods` is a narrower capability than `secure.trusted_mods`, and both are operator-controlled. This is the same least-privilege decomposition WASI achieves with preopened directories, expressed in Lua.

Finally, `core.deserialize(string[, safe])` shows the bytecode hazard from §2.3 handled explicitly. The input "is loaded in an empty sandbox environment," and it "Will load functions if `safe` is `false` or omitted." With `safe` true it "Will silently strip functions embedded via calls to `loadstring`," and the documentation supplies the check: `core.deserialize("return loadstring('')", true)` yields `nil`. But it also refuses to oversell the mitigation, warning "You should not rely on this if possible" and, decisively: "This function should not be used on untrusted data, regardless of the value of `safe`" [Source](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md).

That admission is the honest limit of a language-level sandbox, and the reason the next section exists. Luanti gates capabilities well, but the enforcement lives in the host's discipline about what goes in the environment table, and a bytecode-level escape defeats it. A WASM guest's isolation does not depend on the guest's own bytecode being well-formed, because the failure mode of malformed WASM is a validation error or a trap, not an escape.

---

## 4. WebAssembly Mod Sandboxing

### 4.1 Extism as a General Plugin Framework

Extism is the most direct answer to "I want a plugin system and I do not want to write a WASM host from scratch." It is BSD-3-Clause licensed and describes itself as "The framework for building with WebAssembly (wasm). Easily & securely load wasm modules, move data, call functions, and build extensible apps," with the explicit goal of making untrusted code "safe and practical" to run [Source](https://github.com/extism/extism).

Its architecture is two-sided. Host SDKs embed the runtime into an application, covering Rust, JavaScript (Web, Node, Deno, Bun), Elixir, Go, Haskell, Java, .NET (C# and F#), OCaml, Perl, PHP, Python, Ruby, Zig, and C/C++. Plugin Development Kits (PDKs) are used to *write* plugins, covering Rust, JavaScript, Python, Go, Haskell, AssemblyScript, .NET, C, C++, and Zig [Source](https://github.com/extism/extism).

That matrix is the real product. A mod ecosystem's practical constraint is not usually the host's capability but the modder's language, and a system that only accepts Rust mods will get far fewer mods than one accepting Python and JavaScript. WASM's compilation-target model is what makes the cross-product tractable at all.

The underlying engine is Wasmtime. Extism's runtime crate declares it directly:

```toml
# runtime/Cargo.toml, extism main
wasmtime = { version = "43", default-features = false, features = [
  'anyhow',
  'cache',
  ...
```

[Source](https://raw.githubusercontent.com/extism/extism/main/runtime/Cargo.toml)

This matters because everything §5 says about Wasmtime's security model applies transitively to an Extism-based mod loader. Extism layers plugin ergonomics — a calling convention, data marshalling, runtime limiters and timers, and host-controlled HTTP — on top of Wasmtime's isolation rather than reimplementing it.

The HTTP point deserves emphasis: network access is host-controlled, matching Luanti's `secure.http_mods` gate in §3.6 and WASI's model in §5.3. Three independently designed systems converge on "the plugin cannot open a socket; the host offers a function."

### 4.2 Veloren's WASM Plugin System

Veloren is an open-source voxel RPG whose plugin system is the clearest published example of WASM mods in a game, including the part most systems avoid: pushing mod code to clients.

A plugin is a Rust crate compiled to WebAssembly:

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
veloren-plugin-rt = { git = "https://gitlab.com/veloren/veloren.git" }
```

```bash
rustup target add wasm32-wasi
rustup override set nightly
```

[Source](https://book.veloren.net/contributors/modders/writing-a-plugin.html)

*Note: needs verification* — the documented target triple `wasm32-wasi` predates its rename to `wasm32-wasip1` in the Rust toolchain, and the nightly requirement may no longer apply. These lines are reproduced as the documentation states them, not as current toolchain advice.

Handlers are declared with attribute macros over a runtime crate:

```rust
use veloren_plugin_rt::{*, api::{*, event::*}};

#[event_handler]
pub fn on_load(load: PluginLoadEvent) {
    emit_action(Action::Print(String::from("Hello, Veloren!")));
}

#[event_handler]
pub fn on_command_ping(chat_cmd: ChatCommandEvent)
    -> Result<Vec<String>, String> {
    Ok(vec![String::from("Pong!")])
}

#[global_state]
#[derive(Default)]
struct State {
    ping_count: u64,
}
```

[Source](https://book.veloren.net/contributors/modders/writing-a-plugin.html)

Two things are notable about this shape. `emit_action(Action::Print(...))` is not a `println!` — the guest cannot write to stdout, so it emits a typed action that the host interprets. That is the host-function pattern from §5.3 made visible in the API surface: every effect is a value the host chooses to honour. And `#[global_state]` exists because a WASM guest's state lives in *its own* linear memory across calls; the macro gives the plugin persistent state without the host having to expose engine memory.

Distribution is a plain archive:

```toml
name = "my_plugin"
modules = ["my_plugin.wasm"]
dependencies = []
```

```bash
tar -cvf ../my_plugin.plugin.tar *
```

[Source](https://book.veloren.net/contributors/modders/writing-a-plugin.html)

A `.plugin.tar` is an uncompressed tar of `plugin.toml` plus the `.wasm` module. The payoff is stated by the project directly: plugins "are sandboxed, and so are safe to run client-side automatically," and "They are portable and will work on all architectures and platforms" [Source](https://book.veloren.net/contributors/modders/writing-a-plugin.html).

"Safe to run client-side automatically" is the sentence that justifies this entire architecture. A server can push mod code to every connecting client without the client operator auditing it, because the sandbox — not the trustworthiness of the server — is the enforcement mechanism. No architecture in this chapter other than WASM can make that claim. Luanti's server-to-client transfer is deliberately restricted to *data*, definitions and media, precisely because it has no way to sandbox transferred code. Portability is the second-order consequence: a WASM module needs no per-platform build matrix, whereas a native mod would need one per target triple.

The runtime is Wasmtime, evidenced in-tree by the error type the plugin module surfaces — `HostError(wasmtime::Error)` in `common/state/src/plugin/mod.rs` [Source](https://gitlab.com/veloren/veloren). *Note: needs verification* — this is inferred from a single error variant rather than from a stated architectural document.

The system is explicitly pre-stable. The project warns that "the plugin API and the process for plugin development is under active development and we currently make no stability guarantees about APIs, tooling, etc." [Source](https://book.veloren.net/contributors/modders/writing-a-plugin.html). The relevant ABI observation for §8 is that shared linear memory plus typed event handlers is a much easier interface to keep stable than a Rust vtable layout, because the boundary is bytes and integers rather than compiler-dependent struct layout — but "easier" is not "stable," and Veloren has not committed yet.

### 4.3 wasmtime and wasmer as Capability-Gated Hosts

Wasmtime and Wasmer are the two general-purpose standalone WASM runtimes an engine would embed. Both implement WASI, and the reason they are appropriate for mod hosting rather than merely for running WASM fast is capability gating: the host constructs the guest's entire view of the outside world.

Wasmtime states the filesystem property plainly: with WASI, "applications can only access files and directories they've been given access to" [Source](https://docs.wasmtime.dev/security.html). There is no `open("/etc/passwd")` that succeeds because the process has permission; there is only a preopened directory handle the host granted, and paths resolved relative to it.

The mechanism generalises. A WASM module declares imports, and the host resolves them. If the host does not supply an import, instantiation fails; if it supplies a function, that function is the only route to the capability behind it. There is no equivalent of `dlsym` reaching into the host for something the host did not export, which §5.1 explains structurally.

---

## 5. Why Linear Memory Beats dlopen

This section makes the security argument concrete, because "WASM is sandboxed" is asserted far more often than it is explained, and the explanation is what tells you where the boundary actually is.

### 5.1 The Linear Memory Model, Concretely

A WebAssembly instance's memory is not the process address space. The specification defines it as a distinct object: "A *memory instance* is the runtime representation of a linear memory. It records its type and holds a sequence of bytes." Its length is always a multiple of 65,536 — the 64 KiB page size [Source](https://webassembly.github.io/spec/core/exec/runtime.html).

The consequences of "holds a sequence of bytes" are the entire sandbox.

A WASM pointer is an *index into that byte sequence*, not a machine address. There is no arithmetic a guest can perform on a pointer that yields an address outside its own memory, because the value is not an address in the first place — it is an offset that the runtime adds to a base it controls, with a bound it enforces. Wasmtime states the check directly: "All accesses within linear memory are checked to ensure they stay in bounds" [Source](https://docs.wasmtime.dev/security.html).

This is categorically different from the memory-safety story of a native mod. A native mod with an out-of-bounds write has, by definition, written to some other part of the process — possibly an engine struct, possibly a function pointer. A WASM guest with an out-of-bounds write has triggered a trap. The failure is contained not because the guest was careful but because the address space the guest can name does not extend past its own memory.

Wasmtime hardens the bounds check with a large guard region: a 2 GB guard is reserved before each linear memory, so that a mis-signed or wildly out-of-range offset lands in unmapped pages and faults rather than aliasing anything real [Source](https://docs.wasmtime.dev/security.html). That is a performance optimisation as much as a security one — it lets many bounds checks be elided in favour of hardware faults — but its effect is the same containment.

The second half of the model is how a guest reaches anything at all. A module instance holds "runtime representations of all entities that are imported, defined, or exported by the module," and those entities are referenced through *store addresses* organised by category: functions, tables, memories, globals, tags, data segments, and element segments. An export is a name paired with an external address [Source](https://webassembly.github.io/spec/core/exec/runtime.html).

Read that as an access-control statement. The guest's universe is exactly the set of store addresses its instance holds. Those come from two places: what the module defines itself, and what the host supplied to satisfy its imports. There is no third source. There is no ambient namespace to look things up in, no dynamic symbol table spanning the host, no way to discover a capability that was not handed over. The guest's only route to anything on the host side is a function the host placed in that import table — so a GPU handle, a file descriptor, or a socket that sits behind no such function has no route to the guest at all.

That is the precise sense in which a WASM mod "has no ambient access to a GPU handle." It is not that the runtime filters GPU calls. It is that a GPU handle is a host-side value living in host memory, reachable only through a host function the host chose to export, and the guest's memory contains bytes rather than references.

### 5.2 Control-Flow Integrity and the Unreachable Callstack

Memory isolation would be insufficient if a guest could redirect execution. WASM's control flow is structurally constrained: "All control transfers—direct and indirect branches, as well as direct and indirect calls—are to known and type-checked destinations" [Source](https://docs.wasmtime.dev/security.html).

This confines rather than eliminates computed control flow. `call_indirect` genuinely takes a runtime-supplied table index, so a guest that corrupts its own function-pointer-shaped integer can reach a different function — but only one in its own table, and only one with a matching signature, because the index is bounds-checked against the table and the type is verified before transfer. The destination is therefore always some function already reachable through that table — one the module defined, or one the host itself imported into it. It is never an arbitrary byte offset, and never host code the host did not already hand over. The native primitive of overwriting a function pointer to land on attacker-chosen bytes has no expression here.

The callstack is also not in the guest's memory. Wasmtime notes that the callstack is inaccessible to guests, so stack smashing is not possible, and that guests cannot read return addresses or spilled register values [Source](https://docs.wasmtime.dev/security.html). Return addresses live in machine stack frames the guest cannot address. Whatever a buggy WASM mod does to its own data, it cannot rewrite where it will return to.

The remaining guarantees Wasmtime enumerates are worth stating together, because a mod author's bug becomes a host's incident only if one of them fails: guests cannot access raw syscalls, cannot interact with the outside world except through explicitly imported interfaces, and cannot execute undefined behaviour [Source](https://docs.wasmtime.dev/security.html).

"Cannot access raw syscalls" is the one that most directly separates this from every other architecture in this chapter. A Lua script in a badly-configured host reaches `os.execute`. A native mod issues `syscall` directly. A WASM guest has no instruction that transfers to the kernel; the only way out is a call to an imported function, and imports are the host's list.

### 5.3 WASI: No Ambient Authority

WASI is the standardised set of host interfaces, and its design principle is the absence of ambient authority. In a POSIX process, authority is a property of the process: any code in it can open any path the user can. Under WASI, authority is a property of a *handle*, and handles are granted.

Concretely, a host preopens directories and hands the guest their descriptors. Paths are resolved relative to those, so "applications can only access files and directories they've been given access to" [Source](https://docs.wasmtime.dev/security.html). A guest asked to load a mod asset gets a handle to the mod's own asset directory and cannot express a path outside it — not because a filter rejects `../../`, but because resolution starts from a granted handle rather than from a global root.

For a mod loader this maps onto the requirement almost exactly. A mod needs to read its own assets and persist its own save data. It does not need the user's home directory, the engine's configuration, or `/dev/dri/renderD128`. Under WASI those are not denied; they are unnameable.

The convergence noted in §4.1 is worth restating as a design lesson. Luanti gates OS access behind `secure.trusted_mods` and HTTP behind the narrower `secure.http_mods` (§3.6); Extism routes plugin HTTP through the host; WASI grants filesystem access per preopened directory. The capability model is not a WebAssembly invention. What WebAssembly adds is that the model is enforced by the execution substrate rather than by the host remembering to leave `io` out of a table.

### 5.4 Resource Exhaustion: Fuel and Epochs

Isolation says nothing about liveness. A mod that runs `while true do end` inside a perfect sandbox still hangs the frame. This is the failure mode that most often takes down real modded games, and it is worth being clear that it is a *scheduling* problem, not an isolation one.

Wasmtime provides two mechanisms for bounding guest execution: fuel, where the guest is charged for operations and execution terminates when the budget is exhausted, and epoch-based interruption, where the host periodically bumps a counter that the guest's compiled code checks, allowing termination of malicious or long-running guests [Source](https://docs.wasmtime.dev/security.html). Extism surfaces equivalent controls to plugin hosts as runtime limiters and timers [Source](https://github.com/extism/extism).

Epoch interruption is generally the right choice for a game loop, because it is cheap in the common case — a counter comparison rather than per-operation accounting — while still giving the host a hard deadline. The architectural point is that this mechanism exists at all and is host-controlled. A native mod cannot be preempted this way. There is no fuel for a `dlopen`ed function; interrupting it means signals or thread termination, neither of which leaves engine invariants intact.

### 5.5 The Native Contrast

Setting the two side by side makes the structural difference explicit rather than rhetorical.

A native `dlopen`ed mod shares the address space, so engine internals are readable and writable regardless of visibility. It shares the file-descriptor table, inheriting every open device node, socket, and pipe — including DRM device fds, which is the concrete GPU-reachability answer for its row in §8. It can issue syscalls directly, so a host function table is a convention rather than a boundary. It can execute code at load time through ELF constructors, before the engine's entry point runs. It cannot be resource-limited or preempted cleanly. And it is coupled to the engine's compiled layout, so it breaks across versions.

A WASM mod's memory is a separate bounds-checked byte sequence, so engine internals are unreachable. It has no descriptor table except granted WASI handles. It has no syscall instruction. Its imports are resolved by the host at instantiation, so load-time code cannot reach anything the host did not supply. It can be bounded by fuel or epochs. And its interface is types and bytes rather than struct layout, which is why the same artifact runs on every platform.

Two honest caveats. First, the sandbox's strength is the host function table: a host that exports a function taking a raw pointer, or one that exports something equivalent to "execute this shell command," has handed over the authority regardless of the runtime's properties. The boundary is only as narrow as the imports. Second, WASM guests are slower than native for compute-heavy work, and the isolation cost is real. That is the trade, and for mod code — which is usually orchestration rather than inner loops — it is normally the right one.

---

## 6. .NET IL Patching: Harmony and BepInEx

The architectures so far assume the engine wanted to be extended. Harmony addresses the opposite case: a shipped .NET or Unity game with no mod API, whose behaviour the community modifies anyway. It is the clearest demonstration in this chapter of what happens when an engine declines to provide an extension surface — the community manufactures one, at the IL level, with full process trust.

### 6.1 What a Harmony Patch Does at the IL Level

Lib.Harmony is MIT-licensed, currently version 2.4, and describes itself as "a library for patching, replacing and decorating .NET and Mono methods during runtime," offering "an elegant and high level way to alter the functionality in applications written in C#" [Source](https://github.com/pardeike/Harmony).

Its distinguishing property is stated against the alternative: "Where other patch libraries simply allow you to replace the original method, Harmony goes one step further and gives you: A way to keep the original method intact • Execute your code before and/or after the original method • Modify the original with IL code processors" [Source](https://github.com/pardeike/Harmony).

Mechanically, Harmony operates on the CLR's method-body indirection. A managed method is JIT-compiled from IL on first call. Harmony generates a replacement dynamic method whose body weaves together the original IL and the patch methods, then redirects invocation of the original to that replacement — which is why multiple independent mods can patch the same method and all take effect, whereas naive detouring means the last mod wins. The composition is the reason Harmony rather than raw detours became the ecosystem standard.

Games listed as using Harmony include Rust, RimWorld, Stardew Valley, Subnautica, Cities: Skylines, Kerbal Space Program, Genshin Impact, Ravenfield, Sheltered, and SCP: Secret Laboratory [Source](https://github.com/pardeike/Harmony). Valheim, frequently associated with Harmony, is not on that list; its mod scene is built on BepInEx, which bundles HarmonyX (§6.5), so the attribution runs through BepInEx rather than through Lib.Harmony directly.

### 6.2 Prefixes, Postfixes, and Injected Parameters

A prefix runs before the original body. Per the documentation, prefixes can "access and edit the arguments of the original method," "set the result of the original method," "skip the original method," "set custom state that can be recalled in the postfix," and "run a piece of code at the beginning that is guaranteed to be executed" [Source](https://harmony.pardeike.net/articles/patching.html).

A postfix runs after. Postfixes can "read or change the result of the original method," "access the arguments of the original method," and "read custom state from the prefix" [Source](https://harmony.pardeike.net/articles/patching.html).

The mechanism connecting patch methods to the original's frame is a set of magic parameter names, resolved by name when the composite method is generated:

- `__instance` — "the instance value for non-static methods," functioning like `this`.
- `__result` — the return value; "The type must match the return type of the original or be assignable from it," and modifying it requires declaring it as a `ref` parameter.
- `__resultRef` — modifies `ref return` references, via `ref RefResult<T>`.
- `__state` — carries information from prefix to postfix; "It can be any type and you are responsible to initialize its value in the prefix."
- `__args` — "all arguments at once" as an `object[]`, where writing to the array updates the corresponding arguments.
- `___fieldName` — three leading underscores grant read/write access to a private field, with the underscores stripped to give the field name.
- `__originalMethod` — the original's `MethodBase`, "useful for conditional logic when a patch applies to multiple methods."
- `__runOriginal` — a readonly boolean indicating whether the original will run (in a prefix) or was run (in a postfix).

Original parameters are matched by name, or by `__n` where `n` is the zero-based argument index [Source](https://harmony.pardeike.net/articles/patching-injections.html).

`___fieldName` is worth pausing on. C#'s `private` is a compile-time visibility rule, and the generated composite method emits a direct field access regardless. Nothing about the runtime prevents it. This is the same class of fact as §1.2's observation that a `dlopen`ed mod can read any engine struct: within a single trust domain, encapsulation is a convention. Harmony simply exposes it ergonomically.

### 6.3 Transpilers: Rewriting the Instruction Stream

Prefixes and postfixes wrap. Transpilers rewrite. A transpiler receives the original method's IL and returns a modified stream:

```csharp
static IEnumerable<CodeInstruction> Transpiler(<arguments>)
// or
[HarmonyTranspiler]
static IEnumerable<CodeInstruction> MyTranspiler(<arguments>)
```

Arguments "are identified by their type and can have any name": `IEnumerable<CodeInstruction> instructions` is required, while `ILGenerator generator` and `MethodBase original` are optional [Source](https://harmony.pardeike.net/articles/patching-transpiler.html).

A `CodeInstruction` is a mutable container for one IL operation, carrying an `opcode` and an `operand`. Typical use materialises the sequence into a `List`, locates a recognisable instruction pattern, splices, and returns via `codes.AsEnumerable()`.

The execution model is the part most often misunderstood: "A transpiler is executed only once before the original is run... Harmony will run it once when you patch the method and *again* every time someone else adds a transpiler for the same methods" [Source](https://harmony.pardeike.net/articles/patching-transpiler.html). A transpiler is a compile-time transformation over IL, not a runtime hook — it costs nothing per call, because its output is what gets JIT-compiled. And it re-runs on later patching, which means a transpiler must be written to tolerate having already-transpiled IL as input.

This is genuinely powerful and genuinely brittle. A transpiler that pattern-matches on IL is coupled to the C# compiler's codegen for that method body, so a game update that changes an unrelated line can shift the instruction sequence and break the match. It is the deepest extension mechanism in this chapter and the least stable, which is the honest trade for extending software that never intended to be extended.

### 6.4 Finalizers

A finalizer "is a method that executes after all postfixes. It wraps the original method, all prefixes, and postfixes in try/catch logic" [Source](https://harmony.pardeike.net/articles/patching.html).

For a mod loader this is the exception-safety primitive. Without it, an exception thrown from a prefix propagates into engine code that never expected it, from a call site the engine's authors believed could not throw. Finalizers give a patch a place to observe and suppress that, which is the difference between one misbehaving mod logging an error and the process terminating.

### 6.5 BepInEx: Preloader, HarmonyX, and IL2CPP

Harmony patches methods; something must load Harmony before the game starts. BepInEx is that something — "Bepis Injector Extensible," LGPL-2.1 licensed, a preloader and plugin framework for Unity Mono, Unity IL2CPP, and .NET Framework games including XNA, FNA, and MonoGame [Source](https://github.com/BepInEx/BepInEx).

Its documented platform matrix is directly relevant to Linux modding: Unity Mono is supported on Windows, macOS, and Linux; Unity IL2CPP is supported on Windows and Linux but not macOS or ARM; .NET Framework and XNA titles are supported natively on Windows and via Mono on macOS and Linux [Source](https://github.com/BepInEx/BepInEx). Linux IL2CPP support is the notable entry, since IL2CPP is the harder target.

BepInEx bundles HarmonyX v2.10.2 and MonoMod v22.7.31.1 [Source](https://github.com/BepInEx/BepInEx). HarmonyX is a fork of Harmony built on MonoMod's runtime detour infrastructure rather than Harmony's own, which is what extends the model beyond stock Mono. The patch-authoring surface — prefixes, postfixes, transpilers, injected parameters — is the same as §6.1–6.4.

IL2CPP is worth explaining because it looks like it should defeat this entire approach. Unity's IL2CPP backend compiles IL ahead of time to C++ and then to native code, so there is no managed method body to patch at runtime and no JIT to redirect. Supporting it requires a fundamentally different mechanism: native function detouring plus reconstruction of managed metadata, so patch code written against a C#-shaped API can target functions that are now native. MonoMod.RuntimeDetours provides the detour layer.

The security position of this whole family is §1.2's, with no sandbox and no pretence of one: installing a BepInEx plugin is running an arbitrary program as yourself. That is not a criticism of BepInEx, which exists precisely because the alternative is not modding the game at all.

---

## 7. Engine-Native Extension

### 7.1 Godot 4 GDExtension

GDExtension is Godot's sanctioned native-code path: "GDExtension is a Godot-specific technology that lets the engine interact with native shared libraries at runtime." The documentation is careful that "it is not a scripting language and has no relation to GDScript" [Source](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/what_is_gdextension.html).

An extension has three parts: a native library — `.dll`, `.so`, `.dylib`, or `.wasm` depending on platform; a `.gdextension` manifest describing entry point and per-platform library paths; and a C entry-point function that initialises the extension and registers its classes.

The architectural difference from GDNative, its Godot 3 predecessor, is registration. GDExtension's "new registration system is now part of Godot's ClassDB. This means that classes implemented in plugins are indistinguishable from core classes. Help pages are automatically made available for your classes, detailing their properties, methods, signals, etc." [Source](https://godotengine.org/article/introducing-gd-extensions/) At its core GDExtension is a C API enabling registration of classes implemented within a dynamic library, rather than being limited to backing scripts, and the result "allows extending Godot to nearly the same level as statically linked C++ modules can."

That is the substantive change. GDNative code was reachable as a script implementation; GDExtension code becomes a class in the engine's own type registry, appearing in the editor's node list, in documentation, and in the inspector like anything built in. Godot 3 GDNative plugins do not run on Godot 4 and vice versa; they require alteration and recompilation.

The reason GDExtension does not require recompiling Godot is that it does not participate in the engine's build. A C++ module is compiled into the engine binary, so shipping one means shipping a custom engine build and every consumer rebuilding from source. GDExtension inverts this: the engine ships with a C ABI, the extension is built separately against that ABI, and the engine loads it at runtime. The documentation lists the practical consequences — no compiling of the engine source is needed, which makes distribution easier; multiple languages are usable through bindings; the same library works identically in the editor and in exported projects; and only your library needs compiling rather than the whole engine. C++ modules retain the advantage of deeper engine integration [Source](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/what_is_gdextension.html).

"Works identically in the editor and in exports" is easy to skim past and matters a great deal, because it means an extension can provide editor tooling and runtime behaviour from one artifact — something a scripting-only extension point handles poorly.

The C ABI choice is also what makes the language matrix possible. C++ is the officially supported binding via godot-cpp; community bindings exist for D, Go, Nim, Rust, Swift, and Odin, with the documentation's own caveat that "not all bindings mentioned here may be production-ready" [Source](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/what_is_gdextension.html). A C ABI is the one interface every language can speak, which is precisely the property Rust lacks natively and which §7.2 is about.

Compatibility is asymmetric and bounded: extensions built for an earlier Godot version typically work in later minor versions but not the reverse, and "since GDExtension remains experimental, breaking changes can occur between major versions" [Source](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/what_is_gdextension.html).

Registration into ClassDB is also what sets GDExtension's trust position: because its classes are indistinguishable from core ones, an extension reaches the rendering stack directly — `RenderingDevice` and the explicit Vulkan path included — with no sandbox in between.

### 7.2 Bevy: Dynamic Plugin Removal and Hot-Patching

Bevy's story is the most instructive negative result in this chapter, and it is two separate stories that are routinely merged into a misleading one.

**Dynamic plugins were removed.** Bevy issue #11969 proposed removing `bevy_dynamic_plugin`, and it was implemented by PR #13080. The reasoning is worth reading as a general argument about native mod ABIs, not as a Bevy-specific complaint [Source](https://github.com/bevyengine/bevy/issues/11969):

1. The mechanism loads and executes arbitrary compiled code, potentially running `ctor` initialisation code without an explicit call. The issue states it directly: "It works by loading and executing foreign code, which is drastically unsafe." This is §1.2's ELF-constructor observation, in Rust.
2. Plugins must be compiled with identical Bevy *and* Rust versions, which blocks compiler optimisations and bugfixes for users, because vtable layout is not stable across compiler versions.
3. Bevy makes heavy use of generics and `TypeId`, which lack guaranteed consistent codegen across compiler versions — "even within the same release."

The third point is the deepest. `TypeId` is how Bevy's ECS identifies component types at runtime. If a plugin and the host disagree about the `TypeId` of a component — which they may, because its value is not a stable contract even between two builds of the same rustc release — the plugin registers components the host cannot match. That is not a bug to be fixed; it is a consequence of Rust having no stable ABI, and it cannot be worked around from the engine side.

The disposition was to deprecate `dynamically_load_plugin` and `DynamicPluginExt::load_plugin` in Bevy 0.14, remove them in 0.15, move the crate to `bevy-crate-reservations`, and point users at the third-party `dexterous_developer` project. That crate exists as a hot-reload system for Bevy [Source](https://github.com/lee-orr/dexterous_developer), though it is currently published under a minimal-maintenance marker [Source](https://lib.rs/crates/bevy_dexterous_developer).

**Hot-patching, separately, does exist.** Bevy 0.17 added first-class system hot-patching: "Bevy now supports hot patching systems via subsecond and the `dx` command line tool from the Dioxus project. When the cargo feature `hotpatching` is enabled, every system can now be modified during execution, and the changes are immediately visible in your game" [Source](https://bevy.org/news/bevy-0-17/). Usage is via the Dioxus CLI, `dx serve --hot-patch --features "bevy/hotpatching"`.

The documented limitations define its scope precisely: it only works on the binary crate, it is not supported in WebAssembly, hot reload will not work if a system's parameters change, and it may be sensitive to Rust and linker configuration [Source](https://bevy.org/news/bevy-0-17/). Asset hot-reload via `bevy_asset` is a separate and long-standing capability.

So the accurate statement — and the one §8's hot-reload column encodes — is not "Bevy cannot hot-reload." It is that Bevy has a **developer-loop** hot-patching feature and no **mod-distribution** plugin ABI. Those serve different users. Hot-patching requires the developer's own build environment and toolchain, assumptions a third-party mod author downloading a release binary cannot satisfy. It is the §1.1 distinction made concrete by a real engine: solving the developer loop does not yield a mod system, and Bevy's own analysis in #11969 is that a safe native mod ABI is not reachable at all while Rust lacks a stable one.

Which is precisely why a Rust engine wanting third-party mods reaches for `mlua` or a WASM runtime. Both give a boundary made of names and bytes rather than of vtable layout — and Veloren, a Rust game, chose WASM for exactly this reason.

### 7.3 dexterous_developer: Dynamic-Library Hot-Reload for Bevy

`dexterous_developer` is the third-party project Bevy's own deprecation notice pointed users at (§7.2). It is worth a closer look because it is not a mod system at all — it is a **developer-loop** hot-reload tool, and its own design documents the same dynamic-linking hazards that #11969 gave as the reason to remove `bevy_dynamic_plugin` in the first place, just accepted deliberately in exchange for iteration speed.

The README describes it as "a modular hot-reload system for Rust" with "a first-party Bevy adapter," not a Bevy-specific tool [Source](https://github.com/lee-orr/dexterous_developer). Its own future-work list is the clearest statement of its current mechanism: "Supporting the use of inter-process communication in addition to the current dynamic-library approach" [Source](https://github.com/lee-orr/dexterous_developer) — i.e., today it reloads by rebuilding and reloading a dynamic library into the running process, the same `dlopen`-class mechanism as §1.2's baseline, not by running the changed code out-of-process. That is exactly the trade Bevy's core team declined to make part of the engine: it is workable for a developer rebuilding their own crate with a matching toolchain, and unworkable as a distribution format for a third party's compiled mod, for the identical compiler/ABI reasons #11969 gives.

Usage is CLI-driven rather than a library a mod loads unprompted: the `dexterous_developer_cli` builds and watches the project, and hot reload only compiles in at all behind an explicit `hot` Cargo feature — "designed to work only inside a reloadable library," and deliberately excluded from ordinary release builds [Source](https://github.com/lee-orr/dexterous_developer). Bevy code opts in with a `reloadable_main!` wrapper around the app and marks specific systems, components, events, and resources as reloadable via `reloadable_scope!` and `setup_reloadable_elements`; reloadable resources can be reset to a default or serialized and restored across the reload via `rmp_serde`, which is how it survives a schema change to a `struct` between one build and the next without simply losing that state [Source](https://github.com/lee-orr/dexterous_developer).

Its own Bevy compatibility table stops at Bevy 0.14 [Source](https://github.com/lee-orr/dexterous_developer), the same release in which `bevy_dynamic_plugin` was deprecated, and the GitHub repository is now archived, with its last push on 2025-05-03 — after which Bevy shipped its own first-party `hotpatching` feature in 0.17 (§7.2) [Source](https://github.com/lee-orr/dexterous_developer). Read together with the "minimal-maintenance" marker already noted on its `bevy_dexterous_developer` crate, the project is a reasonable illustration of a pattern this chapter returns to more than once: a third-party crate can hold a gap open for a while, but a native ABI problem this fundamental tends to get solved, if at all, by the engine absorbing it as a first-class feature rather than by an ecosystem crate carrying it indefinitely.

---

## 8. Comparison

| | Sandboxing model | Host-API surface | GPU-resource reachability | Hot-reload |
|---|---|---|---|---|
| **Lua / LuaJIT (`mlua`)** | Language-level. No pointer forging; isolation depends entirely on what the host places in the globals table. Bytecode loading is an escape route. | Host-defined tables and userdata. Stock `io`/`os` are ambient authority unless removed. | None by default — a script sees only exposed host functions. A host *may* expose GPU bindings; nothing structurally prevents it. | Strong. Re-execute a chunk; scripts are text with no ABI. |
| **Luanti (`core.*`)** | Language-level plus an operator-controlled capability gate: `secure.trusted_mods` for OS access via `core.request_insecure_environment()`, narrower `secure.http_mods` for HTTP. Mods share one environment. | Large declarative API: registration, formspecs, lifecycle callbacks. OS and network access require an explicit operator grant. | None. Mods are server-side; visuals are declared as texture names, `drawtype`, and `paramtype2` enumerations. Definitions and media transfer to clients as data, never code. | Yes for content by reload; registration is load-time only by design. |
| **WASM — Extism** | Linear-memory isolation on Wasmtime 43. Bounds-checked memory, type-checked control transfers, no syscalls, imports resolved by host. | Host-defined function imports; PDKs in 10 languages, host SDKs in 15+. HTTP is host-controlled. Runtime limiters and timers. | None without an explicit host export. A GPU handle behind no host function has no route to the guest. | Yes — swap the `.wasm` and re-instantiate. Interface is bytes and integers, not struct layout. |
| **WASM — Veloren** | Same, on Wasmtime. Sandbox is strong enough that plugins are "safe to run client-side automatically" and are pushed server→client. | Typed event handlers (`#[event_handler]`) over shared linear memory; effects emitted as `Action` values the host interprets. `#[global_state]` for guest-side persistence. | None. The guest cannot even write stdout — it emits `Action::Print`. | Yes in principle (`.plugin.tar` reload); API explicitly pre-stable with no stability guarantees. |
| **.NET IL patching (Harmony / BepInEx)** | **None.** Managed code in-process at full trust. Private fields reachable via `___field`; any method patchable. | Unbounded — the entire game assembly plus the whole .NET BCL and OS. | **Full.** Direct access to the graphics API and any engine renderer state. | Patches applied at load; a transpiler runs once at patch time and re-runs when others patch the same method. Not a live-edit loop. |
| **Native — Godot GDExtension** | **None.** Native shared library in-process at full trust. | Full engine API via ClassDB; registered classes are "indistinguishable from core classes." C ABI, so bindings exist for C++, D, Go, Nim, Rust, Swift, Odin. | **Full.** Direct `RenderingDevice` and explicit Vulkan path access. | Library loads at runtime and works identically in editor and exports; forward-compatible across minor versions, breaking across majors. Not live per-function editing. |
| **Native — Bevy** | N/A — no supported third-party native plugin path. | N/A for mods. `hotpatching` targets the engine developer's own systems. | N/A for mods; a developer's own code has full `wgpu` access. | Split: `bevy_asset` hot-reload works; system hot-patching via `subsecond`/`dx` behind the `hotpatching` feature (binary crate only, no Wasm, breaks if system params change). **No mod-distribution ABI** — removed in 0.15 for `TypeId`/vtable instability. |

Three patterns fall out of the table.

**GPU reachability tracks trust exactly, with nothing in between.** Every architecture either grants full graphics access (Harmony, GDExtension) or none at all (Lua, Luanti, WASM). No system in production offers a *restricted* GPU surface to mods — no capability-gated subset of Vulkan, no bounded shader-authoring interface. Sandboxed mods get declarative visual authorship at best, as in Luanti's texture names and `drawtype` enumerations. This is a real gap, and it is not an accident: safely exposing a GPU API means mediating a device queue and a driver that were never designed as a trust boundary.

**The sandboxed architectures are the hot-reloadable ones, for the same structural reason.** Text and bytes carry no ABI. A Lua chunk or a `.wasm` module can be replaced because the interface is names and integers, and that same property is why the host can enumerate and restrict the interface. Bevy's dynamic plugins failed both tests together, because a vtable is neither inspectable nor stable.

**Full-trust IL patching is what an ecosystem produces when the engine offers no extension point.** Harmony's transpilers are the most powerful mechanism in this table and the most brittle, pattern-matching against compiler output that any patch release can shift. Luanti's `core.override_item` accomplishes the same *intent* — redefining another party's content — as a documented operation on a data table. The choice is not really between Lua and IL patching; it is between designing an extension surface and having one reverse-engineered onto you.

---

## 9. Why There Is No Standard

Every architecture in this chapter is a genuinely different engineering decision, not a dialect of one underlying "mod format." That is worth stating plainly, because adjacent parts of the graphics stack do have real, cross-vendor standards — glTF for asset interchange (Chapter 64), OpenUSD for scene composition (Chapter 69), OpenXR for the runtime/compositor boundary (Chapter 27) — and modding conspicuously does not. Three separate observations explain why, and none of them is "nobody has gotten around to it yet."

**What looks like standardization is distribution, not architecture.** Steam Workshop and mod.io are the two platforms an outside observer is most likely to point to as evidence of a modding standard, and both are explicit that they standardize logistics rather than a mod's technical shape. Valve's own Workshop documentation puts the boundary directly on the developer: "you'll need a tool for item authors to upload their entries to your Workshop using the ISteamUGC API," and "since the items you are accepting should be ready-to-use, then your submission tool should accept just the file formats your game client expects to load" [Source](https://partner.steamgames.com/doc/features/workshop). Steam supplies upload, hosting, discovery, and payment — "the Steam Workshop takes care of collecting bank and tax info from authors, provides the tools for specifying pricing... and handles all the backend payment processing" [Source](https://partner.steamgames.com/doc/features/workshop) — and stops there; the file format inside a Workshop item is whatever the game already expected.

mod.io makes the same split even more visible, because it operates across engines rather than inside one game. Its own framing is "our service provides the framework for player customization and creativity within your game" [Source](https://docs.mod.io/) — a hosting and identity layer, sold separately per engine. The Unreal Engine integration is explicit that it does not replace Unreal's own packaging: "the Unreal Engine plugin is a wrapper around the mod.io C++ SDK," covering "connecting your title to mod.io," "browsing, searching, and filtering UGC," and "subscribing and managing local UGC (downloads, installs, updates, removals)" [Source](https://docs.mod.io/unreal), while the actual UGC content is still built and loaded through Unreal's own cooking and `.pak` pipeline underneath. mod.io standardizes *how a player finds and installs* a mod across a dozen engines; it has no opinion on what runs when that mod loads, because that is exactly the §1.1 trust/ABI/hot-reload decision each engine already made independently.

**Where something like a standard does exist, it is scoped to one engine's install base, not the industry, and it is usually not a standard the vendor wrote.** Bethesda's Creation Engine plugin format (`.esp`/`.esm`) is the clearest case: every Skyrim or Fallout mod targets the same binary record format because every copy of the game shares the same engine build, not because Bethesda published a specification for it. The community documentation project that fills that gap says so about as plainly as a README can: "the Oblivion, Skyrim, Fallout 3 and Fallout: New Vegas plugin file formats are all very similar, but while there exists good documentation for Oblivion, there is no equivalent for Fallout 3 and Fallout: New Vegas. The aim is for this repository to become that equivalent" [Source](https://tes5edit.github.io/fopdoc/). xEdit — "an advanced graphical module editor and conflict detector for Bethesda games" spanning Oblivion through Fallout 4 [Source](https://tes5edit.github.io/) — exists because the format had to be reverse-engineered before it could be edited safely, and it remains the tool the entire Bethesda modding community depends on for load-order conflict resolution. The format is real, stable, and shared across an enormous mod ecosystem; it is a standard in every practical sense except that no standards process produced it.

Unreal Engine shows the identical pattern from the vendor side. Epic ships no first-party in-game modding API comparable to Luanti's `core.*` or Bevy's `hotpatching` — its officially supported user-generated-content surface is Verse, scoped to Fortnite/UEFN, not to Unreal licensees generally. The gap is filled the same way BepInEx fills it for Unity (§6.5): a single third-party project, UE4SS, becomes the de facto standard for every other Unreal game's modding scene, injecting "a Lua scripting system platform, C++ Modding API, SDK generator, blueprint mod loader, live property editor and other dumping utilities for UE4/5 games" [Source](https://github.com/UE4SS-RE/RE-UE4SS), explicit that "the goal of UE4SS is not to be a plug-n-play solution that always works with every game" but "to have an underlying system that works for most games" [Source](https://github.com/UE4SS-RE/RE-UE4SS). BepInEx and UE4SS are not competitors — they are the same response to the same vendor silence, arising independently in two different engine communities because neither Unity nor Epic ships anything for third parties to standardize around.

**The deeper reason is that §1.1's three axes are not a policy choice a standard could unify — they are consequences of what the engine already is.** A standards body can require every glTF exporter to emit the same JSON schema because the *asset* is inert data regardless of which engine renders it. It cannot require every engine to land on the same point of the trust/ABI/hot-reload cube, because that point falls out of decisions made for reasons that have nothing to do with modding: Bethesda's format is what a C++ engine with in-house tools produces; Unity and Unreal's absence of a modding API is what "modding wasn't a launch requirement" produces; Bevy's lack of a native plugin ABI is a direct, unavoidable consequence of Rust having no stable ABI at all (§7.2), not a feature the Bevy team declined to build. Standardizing "how mods work" across engines would mean standardizing engine internals — memory layout, type identity, scripting runtime — which is a far larger claim than anyone modding a single game actually needs. The narrower thing that *would* generalize, a shared sandboxing substrate like WASM's linear-memory model (§5), is exactly what is starting to happen — but it standardizes the isolation mechanism, not the mod, and adoption is still per-engine and voluntary (Extism, Veloren) rather than mandated by anything resembling a spec.

---

## Integrations

**Chapter 98 — WebAssembly and WebGPU as a Deployment Target** covers the Emscripten and WASM compilation model for shipping a whole graphics application into a browser sandbox. This chapter applies the identical machinery inward: the same linear-memory instance, imports-resolved-by-host, and no-ambient-authority properties that make a browser safe to run a stranger's renderer in are what make Extism and Veloren safe to load a stranger's mod into a native game (§5.1–5.3). The difference is only which side of the boundary the engine sits on — in Ch98 the engine is the guest, here the engine is the host writing the import table.

**Chapter 40 — Bevy and wgpu: A Rust-Native Vulkan Client** describes the engine whose native mod story is the concrete open problem in §7.2. Bevy's removal of `bevy_dynamic_plugin` (issue #11969, PR #13080) is not a project-management decision but a direct consequence of Rust having no stable ABI: the `TypeId` values that Ch40's ECS uses to identify render components are not guaranteed consistent across compiler builds, so a plugin and the host can disagree about what a component *is*. That is why a Rust engine reaching for third-party extension lands on `mlua` or WASM rather than a dylib.

**Chapter 41 — Godot 4: RenderingDevice and the Explicit Vulkan Path** documents the renderer that GDExtension exposes without restriction. Because GDExtension registers classes into ClassDB such that they are "indistinguishable from core classes," a native extension reaches `RenderingDevice` and Ch41's explicit Vulkan path directly — making it the concrete instance of this chapter's highest-GPU-reachability, zero-sandbox row (§7.1, §8), and the sharpest available contrast with a WASM guest that cannot name a GPU handle at all.

**Chapter 205a — Programmable Games and Competitive-Code Sandboxes** covers Screeps, whose V8-isolate sandbox solves this chapter's exact problem — executing untrusted third-party code inside a running game — a generation before WASM was available. The V8 isolate is the same architectural idea as §5.1's memory instance: a separate heap with a host-controlled global object, where the guest reaches only what was injected. Comparing the two shows which of WASM's properties are genuinely novel (language-agnostic compilation targets, capability-gated WASI filesystem access, fuel-based preemption) and which were already achieved by a scripting-engine isolate.

**Chapter 205c — Open-Source 2D Simulation-Game Engines** surveys engines embedding Squirrel, JavaScript/TypeScript, and Lua for scripting, which are the alternative points in §2's design space evaluated under a much narrower rendering model. Because a 2D sprite-blitting engine's visual surface is fundamentally declarative — sprites, atlases, batching parameters — the GPU-reachability question that dominates §8 largely dissolves there, which is a useful control case for how much of a mod system's security posture comes from the sandbox versus from the renderer simply having no dangerous API to expose.

---

## References

- [GitHub — mlua-rs/mlua](https://github.com/mlua-rs/mlua) — Rust↔Lua bindings, MIT; version matrix, Cargo features, async support (§2.1)
- [GitHub — mlua-rs/rlua](https://github.com/mlua-rs/rlua) — Archived 2025-09-12; deprecation notice and re-export-only 0.20 (§2.2)
- [Lua 5.4 Reference Manual — Operating System Facilities](https://www.lua.org/manual/5.4/manual.html#6.9) — `os.execute` shell access as standard-library ambient authority (§2.3)
- [ioquake3 `code/sys/sys_loadlib.h`](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/sys/sys_loadlib.h) — `dlopen`/`dlsym` and SDL-loadso platform macros, GPL-2.0-or-later (§1.2)
- [ioquake3 `code/sys/sys_main.c`](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/sys/sys_main.c) — `Sys_LoadGameDll`, symbol resolution by name, `dllEntry`/`vmMain` (§1.2)
- [ioquake3 `code/game/g_syscalls.c`](https://raw.githubusercontent.com/ioquake/ioq3/588393618dbc82e7207c21c6ddecca229944a03a/code/game/g_syscalls.c) — Mod-side `dllEntry`, `trap_Print`, native-DLL vs. QVM-bytecode compile targets (§1.2)
- [GitHub — luanti-org/luanti](https://github.com/luanti-org/luanti) — Luanti engine, LGPL-2.1-or-later (§3)
- [Luanti `doc/lua_api.md` (master)](https://raw.githubusercontent.com/luanti-org/luanti/master/doc/lua_api.md) — Registration functions, formspec elements and version history, player callbacks, `core.request_insecure_environment`, `core.request_http_api`, `core.deserialize`, `core.compare_block_status` (§2.3, §3.2–3.6)
- [Luanti API reference — api.luanti.org](https://api.luanti.org/) — `minetest`→`core` namespace rename; shared-environment mod loading (§3)
- [Luanti API reference — nodes](https://api.luanti.org/nodes/) — `drawtype`, `paramtype`, `paramtype2` value enumerations and `palette` (§3.3)
- [Luanti `CMakeLists.txt` (master)](https://raw.githubusercontent.com/luanti-org/luanti/master/CMakeLists.txt) — `USE_LUAJIT` gating of the `lib/bitop` fallback (§3.1)
- [Luanti `src/CMakeLists.txt` (master)](https://raw.githubusercontent.com/luanti-org/luanti/master/src/CMakeLists.txt) — LuaJIT recency probe via `luaopen_string_buffer` (§3.1)
- [Luanti `games/devtest/mods/basenodes/init.lua` (master)](https://raw.githubusercontent.com/luanti-org/luanti/master/games/devtest/mods/basenodes/init.lua) — In-tree `core.register_node` examples (§3.3)
- [GitHub — extism/extism](https://github.com/extism/extism) — WASM plugin framework, BSD-3-Clause; host SDK and PDK language matrix, limiters and timers, host-controlled HTTP (§4.1, §5.4)
- [Extism `runtime/Cargo.toml` (main)](https://raw.githubusercontent.com/extism/extism/main/runtime/Cargo.toml) — Wasmtime 43 dependency (§4.1)
- [Veloren Book — Writing a Plugin](https://book.veloren.net/contributors/modders/writing-a-plugin.html) — `cdylib`/`wasm32-wasi` build, `#[event_handler]`, `#[global_state]`, `.plugin.tar`, client-side sandbox claim, no-stability-guarantee caveat (§4.2)
- [GitLab — veloren/veloren](https://gitlab.com/veloren/veloren) — `HostError(wasmtime::Error)` in `common/state/src/plugin/mod.rs` (§4.2)
- [WebAssembly Core Specification — Runtime Structure](https://webassembly.github.io/spec/core/exec/runtime.html) — Memory instance definition, 65,536-byte page multiple, module instance and store addresses (§5.1)
- [Wasmtime — Security](https://docs.wasmtime.dev/security.html) — Bounds checking, 2 GB guard region, type-checked control transfers, inaccessible callstack, no raw syscalls, WASI file access, fuel and epoch interruption (§4.3, §5.1–5.4)
- [GitHub — pardeike/Harmony](https://github.com/pardeike/Harmony) — Lib.Harmony, MIT, v2.4; capability statement and games list (§6.1)
- [Harmony docs — Patching](https://harmony.pardeike.net/articles/patching.html) — Prefix, postfix, and finalizer semantics (§6.2, §6.4)
- [Harmony docs — Injections](https://harmony.pardeike.net/articles/patching-injections.html) — `__instance`, `__result`, `__resultRef`, `__state`, `__args`, `___field`, `__originalMethod`, `__runOriginal`, `__n` (§6.2)
- [Harmony docs — Transpiler](https://harmony.pardeike.net/articles/patching-transpiler.html) — Transpiler signature, required and optional arguments, single-execution and re-run model (§6.3)
- [GitHub — BepInEx/BepInEx](https://github.com/BepInEx/BepInEx) — LGPL-2.1; Unity Mono / IL2CPP / .NET platform matrix, bundled HarmonyX v2.10.2 and MonoMod v22.7.31.1 (§6.5)
- [Godot docs 4.4 — What is GDExtension?](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/what_is_gdextension.html) — Definition, C++ module comparison, language bindings and production-readiness caveat, version compatibility (§7.1)
- [Godot Engine — Introducing GDNative's successor, GDExtension](https://godotengine.org/article/introducing-gd-extensions/) — ClassDB registration; classes indistinguishable from core classes; parity with statically linked C++ modules (§7.1)
- [Bevy issue #11969 — Remove bevy_dynamic_plugin](https://github.com/bevyengine/bevy/issues/11969) — Three unsoundness categories, `TypeId`/generic codegen instability, deprecate-0.14/remove-0.15 plan (§7.2)
- [Bevy 0.17 release notes](https://bevy.org/news/bevy-0-17/) — `hotpatching` cargo feature via `subsecond` and Dioxus `dx`; documented limitations (§7.2)
- [GitHub — lee-orr/dexterous_developer](https://github.com/lee-orr/dexterous_developer) — Third-party Bevy hot-reload system, Apache-2.0, archived (last push 2025-05-03); dynamic-library reload mechanism, `hot` feature gate, `reloadable_main!`/`reloadable_scope!`, `rmp_serde` state serialization, Bevy 0.14-max compatibility table (§7.2, §7.3)
- [lib.rs — bevy_dexterous_developer](https://lib.rs/crates/bevy_dexterous_developer) — Minimal-maintenance status (§7.2, §7.3)
- [Steamworks docs — Steam Workshop](https://partner.steamgames.com/doc/features/workshop) — ISteamUGC upload tooling, Workshop-handles-payment/hosting vs. developer-defined file formats (§9)
- [mod.io Documentation — Welcome](https://docs.mod.io/) — Cross-engine UGC hosting/identity framing (§9)
- [mod.io Documentation — Unreal Engine Plugin](https://docs.mod.io/unreal) — Plugin as a wrapper around the mod.io C++ SDK; auth, browsing, subscription management layered over Unreal's own cooking/`.pak` pipeline (§9)
- [fopdoc — Fallout 3/New Vegas plugin format documentation](https://tes5edit.github.io/fopdoc/) — Community-authored spec filling the gap left by no official Bethesda documentation (§9)
- [xEdit GitHub Page](https://tes5edit.github.io/) — Module editor and conflict detector spanning Oblivion through Fallout 4 (§9)
- [GitHub — UE4SS-RE/RE-UE4SS](https://github.com/UE4SS-RE/RE-UE4SS) — Injectable Lua/C++ modding system for UE4/5, MIT; "works for most games" rather than a vendor-sanctioned API (§9)
- [mlua v0.12.0 release notes](https://github.com/mlua-rs/mlua/releases/tag/v0.12.0) — Derive-based `UserData`, module reorganisation, thread lifecycle callbacks, Rust 2024 edition (Roadmap)
- [mlua v0.11.6 release notes](https://github.com/mlua-rs/mlua/releases/tag/v0.11.6) — Lua 5.5 support behind the `lua55` feature, external-string optimisation (Roadmap)
- [mlua issue #23 — wasm32-unknown-unknown support](https://github.com/mlua-rs/mlua/issues/23) — Long-running request; only `wasm32-unknown-emscripten` is supported today (Roadmap)
- [Luanti `doc/direction.md`](https://github.com/luanti-org/luanti/blob/master/doc/direction.md) — Official Direction Document; medium-term roadmap covering SSCSM, input handling, UI improvements (Roadmap)
- [Luanti `doc/sscsm_security.md`](https://github.com/luanti-org/luanti/blob/master/doc/sscsm_security.md) — SSCSM threat model, non-binary `enable_sscsm` setting, planned SECCOMP process isolation (Roadmap)
- [Luanti PR #15818 — Create SSCSM skeleton and scripting](https://github.com/luanti-org/luanti/pull/15818) — Merged 2026-01-27; first landed piece of server-sent client-side modding (Roadmap)
- [Luanti issue #6527 — Replace formspecs](https://github.com/luanti-org/luanti/issues/6527) — Roadmap-tracked replacement of the declarative UI string protocol described in §3.4 (Roadmap)
- [Wasmtime v46.0.0 release notes](https://github.com/bytecodealliance/wasmtime/releases/tag/v46.0.0) — WASI 0.3.0 and the `component-model-async` feature enabled by default (Roadmap)
- [Wasmtime v47.0.0 release notes](https://github.com/bytecodealliance/wasmtime/releases/tag/v47.0.0) — Wasm GC and exception handling on by default; wasi-threads and `wasi-common` removed (Roadmap)
- [Bytecode Alliance — WASI 0.3 Launched](https://bytecodealliance.org/articles/WASI-0.3) — Async made native to Components via `stream<T>`/`future<T>`; guest-toolchain work in progress (Roadmap)
- [Bytecode Alliance — The Road to Component Model 1.0](https://bytecodealliance.org/articles/the-road-to-component-model-1-0) — Five workstreams to 1.0, including lazy ABI and the two-browser-engine requirement (Roadmap)
- [Extism issue #666 — runtime: wasi preview2](https://github.com/extism/extism/issues/666) — Tracked migration that would bring per-directory permissions and wasi-sockets into the manifest (Roadmap)
- [Extism `runtime/Cargo.toml`](https://github.com/extism/extism/blob/main/runtime/Cargo.toml) — Wasmtime dependency pinned at version 43, behind the WASI 0.3.0 default (Roadmap)
- [BepInEx v6.0.0-pre.2 release](https://github.com/BepInEx/BepInEx/releases/tag/v6.0.0-pre.2) — Latest tagged v6 build, from 2024-08-27; BepInEx 5 plugins do not yet load under it, users directed to Bleeding Edge builds (Roadmap)
- [BepInEx commit 6abdba4 — Revert Cpp2IL because of regressions](https://github.com/BepInEx/BepInEx/commit/6abdba47eeebe08552282e7a58ef0f4a9ab60b62) — Most recent `master` commit as of 2026-06-28; IL2CPP toolchain churn (Roadmap)
- [godot-docs PR #10827 — Stop referring to GDExtension as experimental](https://github.com/godotengine/godot-docs/pull/10827) — Merged 2025-05-02; removed the experimental caveat quoted in §7.1 (Roadmap)
- [Godot docs — About godot-cpp, Version compatibility](https://docs.godotengine.org/en/4.7/tutorials/scripting/cpp/about_godot_cpp.html) — Current compatibility promise: minor-version forward compatibility only (Roadmap)
- [Godot — Introducing the Godot Asset Store](https://godotengine.org/article/introducing-the-godot-asset-store/) — Replacement for the Asset Library, integrated into Godot 4.7; paid assets planned (Roadmap)
- [Bevy 0.19 release notes](https://bevy.org/news/bevy-0-19/) — Release announcement and "What's Next" list; no hot-patching or dynamic-plugin item (Roadmap)
- [Bevy issue #24832 — hotpatching linker issue](https://github.com/bevyengine/bevy/issues/24832) — Representative of the open `hotpatching` issues, which are uniformly toolchain and linker failures (Roadmap)

---

## Roadmap

### Near-term (6–12 months)

- **WASI 0.3 moves from runtime to toolchain.** WASI 0.3 launched on 2026-06-11, rebasing the standard onto Component Model async primitives — `stream<T>`, `future<T>`, and `async` as first-class parts of the canonical ABI [Source](https://bytecodealliance.org/articles/WASI-0.3) — and Wasmtime 46 enabled WASI 0.3.0 and the `component-model-async` feature by default [Source](https://github.com/bytecodealliance/wasmtime/releases/tag/v46.0.0), with Wasmtime 47 turning on Wasm GC and exception handling and removing wasi-threads and the `wasi-common` crate [Source](https://github.com/bytecodealliance/wasmtime/releases/tag/v47.0.0). The host side is therefore already shipped; the near-term work named by the Bytecode Alliance is guest-toolchain enablement for Rust, Go, JavaScript, and Python, which is what actually determines whether a mod author can target the capability model of §5.3 (§4.3, §5.3).
- **Extism has not yet followed Wasmtime's WASI generation.** Extism's runtime still pins `wasmtime = { version = "43", ... }` [Source](https://github.com/extism/extism/blob/main/runtime/Cargo.toml), predating the release in which WASI 0.3.0 became the default, so the host functions and manifest described in §4.1 remain on the older preview generation. The tracked migration is issue #666, whose stated payoff is wasi-sockets reusing the existing allowed-hosts list and "more advanced permissions for files/directories," for which "we will need to think about how to integrate that information into the manifest" [Source](https://github.com/extism/extism/issues/666) — that is, the manifest gains finer capability granularity than the current directory-preopen model (§4.1, §5.3).
- **Luanti's SSCSM ships, but deliberately clamped.** The SSCSM skeleton and scripting layer merged on 2026-01-27 [Source](https://github.com/luanti-org/luanti/pull/15818), putting server-sent client-side mods in-tree with their own `doc/sscsm_api.md` and `doc/sscsm_security.md`. The security document caps deployment rather than declaring victory: `enable_sscsm` is a graduated setting over `nowhere`, `singleplayer`, `localhost`, `lan`, and everywhere, and "until sufficient security measures are in place, users are disallowed to set this setting to anything higher than `localhost`" [Source](https://github.com/luanti-org/luanti/blob/master/doc/sscsm_security.md) — the same conservative posture as the `secure.trusted_mods` gate in §3.6, applied to a new surface before it is exposed (§3.6).
- **mlua has shipped its Lua-version work and publishes no roadmap.** Lua 5.5 support landed in v0.11.6 behind the `lua55` feature [Source](https://github.com/mlua-rs/mlua/releases/tag/v0.11.6), and v0.12.0 followed with derive-based `UserData`, a reorganised module tree, thread create/resume/yield lifecycle callbacks across all Lua versions, and Rust 2024 edition [Source](https://github.com/mlua-rs/mlua/releases/tag/v0.12.0). The project maintains no published roadmap or release milestones, so the forward-looking claim available here is narrow: the sandboxing story of §2.3 is unchanged, because `Lua::sandbox` remains Luau-only and no equivalent exists for PUC-Rio Lua or LuaJIT [Source](https://github.com/mlua-rs/mlua) (§2.1, §2.3).

### Medium-term (1–3 years)

- **Luanti plans an OS-level fallback beneath its Lua sandbox.** The SSCSM threat model states the position §2.3 argues for in the abstract: "we do not trust the Lua implementation to not have bugs," therefore an "additional process isolation layer as fallback" is required. That layer is explicitly "not yet implemented" and is specified as a separate SSCSM process sandboxed with SECCOMP on Linux [Source](https://github.com/luanti-org/luanti/blob/master/doc/sscsm_security.md). This is the clearest statement in any project surveyed here that a language-level sandbox is treated as insufficient on its own (§2.3, §5).
- **Luanti's formspec protocol is slated for replacement.** The project's Direction Document names UI improvements as one of three medium-term roadmap areas, alongside SSCSM and input handling, and tracks formspec replacement as its own long-running issue [Source](https://github.com/luanti-org/luanti/blob/master/doc/direction.md). The Direction Document is reviewed roughly every two years and functions as a gate rather than a wish list — pull requests outside it are closed absent concept approval — so the versioned string protocol dissected in §3.4 should be read as a format with a scheduled successor rather than a settled interface [Source](https://github.com/luanti-org/luanti/issues/6527) (§3.4).
- **The Component Model's lazy ABI is staged for a default flip.** The path to Component Model 1.0 is described as five workstreams, one of which introduces a lazy ABI as opt-in during a 0.3.x minor release and then as the default at 1.0 [Source](https://bytecodealliance.org/articles/the-road-to-component-model-1-0). For mod hosting this is the ABI-stability axis of §1.1 being addressed by specification rather than by convention, which is precisely the property `dlopen`-based loading cannot obtain (§1.1, §5).
- **Godot retired the "experimental" label without extending its ABI promise.** The caveat quoted in §7.1 — that breaking changes can occur between major versions because GDExtension remains experimental — was deliberately removed from the documentation by a pull request titled "Stop referring to GDExtension as experimental," merged on 2025-05-02 [Source](https://github.com/godotengine/godot-docs/pull/10827). What replaced it is narrower than a stability guarantee: current documentation promises only that extensions targeting an earlier version work in later *minor* versions and not the reverse, names the 4.0→4.1 break as the one exception, and adds that extensions are compatible only with engine builds using the same floating-point precision [Source](https://docs.godotengine.org/en/4.7/tutorials/scripting/cpp/about_godot_cpp.html). No cross-major ABI commitment is published anywhere, and no Godot 5 plans are announced (§7.1).
- **BepInEx 6 has no announced stabilisation date.** The v6 line has stood at `v6.0.0-pre.2` since 2024-08-27, a build explicitly labelled a pre-release under which BepInEx 5 plugins do not yet load [Source](https://github.com/BepInEx/BepInEx/releases/tag/v6.0.0-pre.2); the stable line remains v5 LTS, and the README still qualifies the platform matrix with "currently only Unity Mono has stable releases" [Source](https://github.com/BepInEx/BepInEx). Development is active but flows through Bleeding Edge builds rather than tagged releases, which is what the v6 release page itself directs users to [Source](https://github.com/BepInEx/BepInEx/releases/tag/v6.0.0-pre.2), and the most recent commit on `master` reverts a Cpp2IL bump "because of regressions" [Source](https://github.com/BepInEx/BepInEx/commit/6abdba47eeebe08552282e7a58ef0f4a9ab60b62) — IL2CPP toolchain churn rather than stabilisation work, consistent with §6.5's account of IL2CPP support as the harder half of the problem (§6.5).
- **Bevy has announced nothing further for hot-patching.** Neither the 0.18 nor the 0.19 release notes mention hot-patching, `subsecond`, or dynamic plugins, and the "What's Next" list published with 0.19 — scene format, unified 2D/3D rendering internals, entity inspector, assets-as-entities, WESL shaders — contains no hot-reload or extension item [Source](https://bevy.org/news/bevy-0-19/). The project publishes no roadmap page. The open `hotpatching` issues are uniformly linker and toolchain failures rather than design work [Source](https://github.com/bevyengine/bevy/issues/24832), which corroborates §7.2's characterisation of the feature as a development-loop tool sensitive to Rust and linker configuration rather than a shipping mod-loading mechanism (§7.2).

### Long-term

- **Component Model 1.0 is gated on browser adoption that has not been committed.** Reaching 1.0 is described as requiring native implementation in at least two browser engines, with the current state of engine interest characterised as signals rather than commitments, and no target date is given [Source](https://bytecodealliance.org/articles/the-road-to-component-model-1-0). The remaining workstreams — implementation simplification, a proposed `lower-components` tool, generated guest and host C ABI headers, and unresolved WIT expressivity gaps that will not all land before 1.0 — set the horizon on which the mod-hosting substrate of §5 becomes a stable versioned target rather than a moving one (§5).
- **Shared-nothing components are the only cross-engine convergence with evidence behind it.** §9 argues that the narrow thing that could generalise across engines is the isolation mechanism rather than the mod format, and the Component Model work is explicit that shared-nothing architecture is what makes capabilities such as record/replay debugging tractable [Source](https://bytecodealliance.org/articles/the-road-to-component-model-1-0). Nothing surveyed here suggests a common mod *format*; adoption remains per-engine and voluntary, as with Extism and Veloren (§9, §5).
- **No cross-engine mod standard is on any published roadmap.** Neither mod.io nor Steam Workshop publishes a forward-looking roadmap; mod.io's documentation describes only its existing per-engine integrations and hosting service [Source](https://docs.mod.io/), and Valve's Workshop documentation continues to place file-format responsibility on the developer [Source](https://partner.steamgames.com/doc/features/workshop). Godot's Asset Store, now integrated into 4.7 as the Asset Library's replacement, is a distribution and payments layer with paid assets as its headline planned feature, not an extension-architecture standard [Source](https://godotengine.org/article/introducing-the-godot-asset-store/) — which leaves §9's conclusion intact: the layer that standardises is distribution, and the layer that does not is architecture (§9).

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
