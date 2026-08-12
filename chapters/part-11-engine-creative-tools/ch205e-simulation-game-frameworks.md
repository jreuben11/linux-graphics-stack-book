# Chapter 205e: FOSS Simulation-Game Frameworks — Data Models, Rules Engines, and Modding as SDKs

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Systems and application developers evaluating a data-driven rules engine and modding architecture for a domain-specific simulation — 4X/strategy-game developers, and readers of Ch205c/Ch205d looking for the data-model half of that story rather than the rendering half.
> **Status**: First draft — 2026-08-10

A general-purpose engine like Godot or Bevy gives you a scene graph, a renderer, and a scripting layer, and leaves the question of *what a "city" or a "unit" is* entirely to you. The five projects in this chapter answer that question first and build everything else around the answer. Each one ships a fixed domain model — civilizations, techs, cities, units; or UFOs, soldiers, research; or planets, fleets, orders — plus a rules engine that operates over instances of that model, plus a modding surface that lets a third party redefine the content without recompiling the host program. None of them is trying to be a general engine, and judging them as failed attempts at one misses what they actually are: SDKs for a specific genre of simulation, where the "SDK" is the ruleset format.

This chapter covers five such frameworks on their own architectural terms — data model, rules engine, client/server networking, modding system — deliberately not framed around rendering or GPU integration. OpenCiv3 splits UI, engine, and data into three C#/Godot components connected by message queues, and lets mods extend the rules engine itself via embedded Lua. Freeciv keeps a genuinely separate network-authoritative server reading plain-text `.ruleset` files, and freeciv-web wraps that same server behind a WebSocket bridge without touching its rules format. OpenXcom Extended (OXCE) pushes an entire game's content — including its own base game — through one YAML ruleset format, to the point that "mod" and "base ruleset" are the same kind of file. Thousand Parsec, dead since 2011, is included as an instructive dead end: it went a step further than any live project here, making the ruleset itself a swappable compiled module behind a network protocol. A closing survey covers FreeCol, FIFE/Unknown Horizons, and Battle for Wesnoth's WML more briefly.

---

## Table of Contents

- [1. What Is a Simulation-Game Framework](#1-what-is-a-simulation-game-framework)
  - [1.1 Scope Against General-Purpose Engines](#11-scope-against-general-purpose-engines)
  - [1.2 The Data-Driven Rules + Modding Pattern](#12-the-data-driven-rules--modding-pattern)
- [2. OpenCiv3: Three Tiers and Embedded Lua](#2-openciv3-three-tiers-and-embedded-lua)
  - [2.1 C7 / C7Engine / C7GameData](#21-c7--c7engine--c7gamedata)
  - [2.2 Message-Passing Between UI and Engine](#22-message-passing-between-ui-and-engine)
  - [2.3 A JSON DTO Layer Under the Live Game State](#23-a-json-dto-layer-under-the-live-game-state)
  - [2.4 MoonSharp: BehaviorEngine and GameMode](#24-moonsharp-behaviorengine-and-gamemode)
- [3. Freeciv: Network-Authoritative Client/Server and Text Rulesets](#3-freeciv-network-authoritative-clientserver-and-text-rulesets)
  - [3.1 civclient/civserver Separation](#31-civclientcivserver-separation)
  - [3.2 The .ruleset Text-File System](#32-the-ruleset-text-file-system)
  - [3.3 freeciv-web as a Bridge, Not a Reimplementation](#33-freeciv-web-as-a-bridge-not-a-reimplementation)
  - [3.4 Lua Scripting Underneath the Text Rulesets](#34-lua-scripting-underneath-the-text-rulesets)
  - [3.5 AI Opponents: A Deliberately Out-of-Scope Axis](#35-ai-opponents-a-deliberately-out-of-scope-axis)
- [4. OpenXcom/OXCE: YAML Rulesets as the Entire Content Pipeline](#4-openxcomoxce-yaml-rulesets-as-the-entire-content-pipeline)
  - [4.1 The bin/standard/&lt;ModName&gt;/*.rul Convention](#41-the-binstandardmodnamerul-convention)
  - [4.2 Total-Conversion Mods and the Mod Portal](#42-total-conversion-mods-and-the-mod-portal)
  - [4.3 OXCE vs. Base OpenXcom](#43-oxce-vs-base-openxcom)
- [5. Thousand Parsec: Protocol-First Rulesets (An Instructive Dead End)](#5-thousand-parsec-protocol-first-rulesets-an-instructive-dead-end)
- [6. Data-Driven Rules Compared](#6-data-driven-rules-compared)
- [7. Brief Survey: FreeCol, FIFE/Unknown Horizons, Wesnoth WML](#7-brief-survey-freecol-fifeunknown-horizons-wesnoth-wml)
- [8. How Mods Compose: Load Order and Conflict Resolution](#8-how-mods-compose-load-order-and-conflict-resolution)
  - [8.1 OpenCiv3: Declaration-Order Pipeline, No Conflict Detection](#81-openciv3-declaration-order-pipeline-no-conflict-detection)
  - [8.2 OXCE: Explicit master Chains and Per-Field Override Keywords](#82-oxce-explicit-master-chains-and-per-field-override-keywords)
  - [8.3 Freeciv and Thousand Parsec: No Runtime Composition](#83-freeciv-and-thousand-parsec-no-runtime-composition)
- [9. Save-Game Version Compatibility](#9-save-game-version-compatibility)
  - [9.1 Freeciv: Forward Migration Through a Versioned Compat Table](#91-freeciv-forward-migration-through-a-versioned-compat-table)
  - [9.2 OXCE: No Version Gate, Silent Best-Effort Loading](#92-oxce-no-version-gate-silent-best-effort-loading)
  - [9.3 OpenCiv3: A Version Field That Is Dead Code](#93-openciv3-a-version-field-that-is-dead-code)
  - [9.4 Thousand Parsec: A Hard Version Gate, With One Exception](#94-thousand-parsec-a-hard-version-gate-with-one-exception)
- [10. Mod Distribution and Discovery Channels](#10-mod-distribution-and-discovery-channels)
  - [10.1 Freeciv: A Dedicated Modpack Installer and Repository](#101-freeciv-a-dedicated-modpack-installer-and-repository)
  - [10.2 OpenCiv3: No Channel Yet — The Packaging Format Itself Is Unresolved](#102-openciv3-no-channel-yet--the-packaging-format-itself-is-unresolved)
  - [10.3 Thousand Parsec: No Third-Party Distribution Ever Emerged](#103-thousand-parsec-no-third-party-distribution-ever-emerged)
  - [10.4 FreeCol: A Separate Community-Mod Repository](#104-freecol-a-separate-community-mod-repository)
- [11. Turn-Based Multiplayer Beyond Client/Server: Longturn, PBEM, and Deadline Resolution](#11-turn-based-multiplayer-beyond-clientserver-longturn-pbem-and-deadline-resolution)
  - [11.1 Freeciv Longturn: Concurrent Play With a Very Long Timer](#111-freeciv-longturn-concurrent-play-with-a-very-long-timer)
  - [11.2 Freeciv PBEM: Strict Alternation, Not Longturn Under Another Name](#112-freeciv-pbem-strict-alternation-not-longturn-under-another-name)
  - [11.3 FreeCol and OXCE: Outside This Axis Entirely](#113-freecol-and-oxce-outside-this-axis-entirely)
  - [11.4 Thousand Parsec: Simultaneous Resolution on a Deadline](#114-thousand-parsec-simultaneous-resolution-on-a-deadline)
- [Integrations](#integrations)
- [References](#references)

---

## 1. What Is a Simulation-Game Framework

### 1.1 Scope Against General-Purpose Engines

A general-purpose engine is content-agnostic by design: Godot's `Node` and Bevy's ECS `Component` make no assumption about whether the thing being simulated is a spaceship, a city, or a spreadsheet. The frameworks in this chapter make exactly that assumption, up front, in the type system. OpenCiv3's domain model has a `City`, a `Player`, a `MapUnit`, and a `Tile`, because it is a Civilization III remake and nothing else. Freeciv's server has `advance` (a tech), `nation`, `unit_type`, and `terrain`, because it is Freeciv. Fixing the domain model is what makes a data-driven rules engine possible at all: once "city" and "tech" are stable concepts in the code, a ruleset file only needs to fill in the *values* — a tech's prerequisites, a unit's attack strength — without touching the engine that interprets them.

This is a narrower and, for a fixed genre, more productive proposition than a general engine. It is also why these projects should not be read as attempts at reusable multi-game SDKs that fell short. OpenXcom's rules engine is extraordinarily moddable, to the point of hosting total-conversion games with almost no resemblance to the original X-COM, but it is still one game's rules engine — its data model never stops being UFOs, soldiers, and X-COM's specific stat block. The genuinely general-purpose entry in this chapter, Thousand Parsec, tried to make even the *ruleset itself* swappable behind a network protocol — and it is the one project here that is dead. That is one data point, not a proof, but it is a real result worth sitting with in §5 and §6.

### 1.2 The Data-Driven Rules + Modding Pattern

Every framework surveyed here shares a three-part structure:

1. **A fixed domain model** — a set of engine-native types (in C#, C++, or Java) representing the entities the simulation reasons about.
2. **A rules engine** that reads instances of that model out of a content format — INI-style text, YAML, or embedded Lua — and drives turn processing, combat, research, and AI against it.
3. **A modding surface**, which is simply *how far into the rules engine an unprivileged content author can reach* without recompiling the host program. This ranges from filling in blanks in a text template (Freeciv's `.ruleset`) to writing arbitrary Lua that manipulates live engine objects by reflection (OpenCiv3) to writing and compiling a new C++ shared module that the server loads by name (Thousand Parsec).

A fourth axis, orthogonal to the three above, is *where the authoritative simulation state lives relative to the process rendering it*. Freeciv's `civclient`/`civserver` split (§3.1) and freeciv-web's WebSocket bridge over that same split (§3.3) put the rules engine in a separate, network-authoritative process from the outset. OpenCiv3's `MessageToEngine`/`MessageToUI` queues (§2.2) are explicitly shaped with that same split in mind for later — "exploring networking options," per its own `readme.md` — but today run in a single process. OXCE and Thousand Parsec sit at the two extremes of this axis with little middle ground between them: OXCE's rules engine runs entirely client-side, with no server process at all, while Thousand Parsec (§5) makes the network protocol itself the whole integration surface — ruleset modules run only inside `tpserver-cpp`, and every client, admin tool, and AI player is a separate program that never touches ruleset code directly. §6 folds this into its own comparison column alongside the modding axis.

§6 returns to both axes directly. What varies across the five projects is not whether the data-vs-code pattern exists — it exists everywhere — but where the line falls between "data a mod can edit" and "code the engine's maintainers must write," and, independently, whether that line is enforced across a network boundary or inside one process.

---

## 2. OpenCiv3: Three Tiers and Embedded Lua

OpenCiv3 (formerly codenamed "C7") is "an open-source, mod-oriented remake of *Civilization III* by the fan community built with Godot and C#," currently in "an early pre-alpha state" — playable but missing late-game content [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/README.md). It is MIT-licensed, developed against the `Development` branch rather than `main`, and lives at `github.com/C7-Game/OpenCiv3`.

### 2.1 C7 / C7Engine / C7GameData

The project's own documentation describes a three-part split. The root README lists the top-level folders as "C7 - The core game, which runs on the Godot engine," "C7Engine - The mechanics of the game, including AI logic," and "C7GameData - Stores native game data, which will be saved to disc when the save feature is merged" [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/README.md). `C7Engine/readme.md` is more specific about the intent: "The UI (in the C7 folder) will call the engine when the player takes actions, and the engine will update the game state and perhaps send a reply back (e.g. a result of combat)... This should hopefully keep the UI and the engine and the data somewhat decoupled, and facilitate both maintenance and exploring networking options" [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/readme.md). `C7GameData/readme.md` frames the data tier as "the master copy of the game state," updated and read by the engine as needed [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7GameData/readme.md).

The README's list treats C7, C7Engine, and C7GameData as three parallel top-level folders, but the repository's actual layout does not match that framing: `C7GameData` is physically nested at `C7Engine/C7GameData`, not a sibling directory, and there is no separate `C7GameData.csproj` — it builds as part of the `C7Engine` assembly. The three-tier split described in the two readme files is real and intentional, but it is a *namespace and convention* boundary within one assembly, not an assembly boundary. The coupling runs both directions: `C7Engine/C7GameData/Save/SaveGame.cs`, the file responsible for the data tier's serialization, opens with `using C7Engine;` and `using C7Engine.Lua;` [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7GameData/Save/SaveGame.cs) — the data layer depends back on the engine layer it is nominally beneath. This is a more honest description of the architecture than "explicit three-tier separation" would suggest, and it does not make the design less useful; it means the boundary is enforced by discipline and code review rather than by the compiler.

### 2.2 Message-Passing Between UI and Engine

The UI-to-engine boundary is implemented as two message hierarchies queued through a shared `EngineStorage` object. `MessageToEngine` is abstract, requiring subclasses to implement `process()`, and its `send()` method enqueues the instance onto `EngineStorage.pendingMessages`:

```csharp
// C7Engine/EntryPoints/MessageToEngine.cs, OpenCiv3 (Development branch)
public abstract class MessageToEngine {
	public abstract void process();

	public void send() {
		EngineStorage.pendingMessages.Enqueue(this);
	}
}

public class MsgSetFortification : MessageToEngine {
	private ID unitID;
	private bool fortifyElseWake;

	public override void process() {
		MapUnit unit = EngineStorage.gameData.GetUnit(unitID);
		if (unit != null) {
			if (fortifyElseWake) { /* ... */ }
		}
	}
}
```
[Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/EntryPoints/MessageToEngine.cs)

The reverse direction, `MessageToUI`, is the mirror: a plain interface with a `send()` that enqueues onto `EngineStorage.messagesToUI`, plus an `AnimationMessage` subclass that enqueues onto a separate `animationMessages` queue and can be marked completed against a `pendingAnimations` table of `TaskCompletionSource` handles keyed by a `Guid` — the mechanism by which the engine can `await` a UI-side animation finishing before continuing turn processing [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/EntryPoints/MessageToUI.cs). The pattern is a pair of typed, discriminated-union-like message queues rather than direct method calls across the UI/engine boundary — exactly the shape a networked client/server split would need, which lines up with `C7Engine/readme.md`'s stated motivation of "exploring networking options."

### 2.3 A JSON DTO Layer Under the Live Game State

Saves are not serialized `GameData`/`City`/`Player` objects directly. `C7Engine/C7GameData/Save/SaveGame.cs` defines a separate `SaveGame` class with a static `SaveGame.FromGameData(GameData data)` factory that copies fields across into `Save*`-prefixed counterparts (`SaveMap`, and by the same pattern `SaveCity`, `SavePlayer`, etc.), and configures a dedicated `JsonSerializerOptions`:

```csharp
// C7Engine/C7GameData/Save/SaveGame.cs, OpenCiv3 (Development branch)
private static JsonSerializerOptions JsonOptions {
	get => new JsonSerializerOptions {
		PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
		WriteIndented = true,
		IncludeFields = true,
		Converters = {
			new Json2DArrayConverter(),
			new IDJsonConverter(),
			new JsonStringEnumConverter(JsonNamingPolicy.CamelCase),
		},
		TypeInfoResolver = new DefaultJsonTypeInfoResolver {
			Modifiers = { TypeInfoResolver.IgnoreDefaultValues },
		},
	};
}
```
[Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7GameData/Save/SaveGame.cs)

`TypeInfoResolver.IgnoreDefaultValues` walks every property of every serialized type and installs a `ShouldSerialize` predicate — skip empty strings, empty collections, empty enumerables, and null references — which keeps save files close to a diff of non-default state rather than a full dump of every field. The DTO layer buys three things a direct serialization of `GameData` would not: custom converters can encode engine-internal representations (a 2D array, a custom `ID` type, enums-as-strings) independently of how the live objects hold that data; default-value pruning is centralized in one resolver instead of scattered `[JsonIgnore]` attributes; and — as §2.4 shows — the same `SaveGame` type doubles as the target format that mod-authored Lua rulesets compile down to.

### 2.4 MoonSharp: BehaviorEngine and GameMode

OpenCiv3's modding system is built on MoonSharp, a Lua interpreter for .NET, pinned in `C7Engine.csproj` as `<PackageReference Include="moonsharp" Version="2.0.0" />` alongside `net8.0` [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7Engine.csproj). Two classes carry the design: `BehaviorEngine`, which turns Lua functions into callable C# delegates, and `GameMode`, which loads and layers Lua-authored scenario/ruleset content.

`BehaviorEngine` is documented in its own header comment as "A rules engine that loads game behaviors defined in a Lua script... The intended use case is during the game initialization (e.g., when loading a JSON save), to transform given Lua function paths into C# delegates." Its core method, `ImportFunc<T>`, takes a dot-separated path like `"building.production_rules.must_be_coastal"`, walks a Lua `Table` segment by segment, and on reaching a leaf function converts it to a strongly typed `Delegate` via MoonSharp's `DelegateConverter`, caching the result:

```csharp
// C7Engine/Lua/BehaviorEngine.cs, OpenCiv3 (Development branch)
public T ImportFunc<T>(string functionPath) where T : Delegate {
	if (funcCache.TryGetValue(functionPath, out Delegate func))
		return (T)(object)func;

	Closure closure = ResolveFunctionPath(functionPath);
	Delegate del = DelegateConverter.CreateDelegate(script, closure, typeof(T));
	funcCache[functionPath] = del;
	return (T)(object)del;
}

Closure ResolveFunctionPath(string functionPath) {
	string[] parts = functionPath.Split('.');
	Table current = rules;
	for (int i = 0; i < parts.Length; i++) {
		DynValue value = current.Get(parts[i]);
		if (value.Type == DataType.Function && i == parts.Length - 1)
			return value.Function;
		if (value.Type == DataType.Table)
			current = value.Table;
		else
			throw new ArgumentException($"Unexpected type at '{parts[i]}': '{value.Type}'");
	}
	throw new ArgumentException($"Function path '{functionPath}' did not resolve to a function.");
}
```
[Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/BehaviorEngine.cs)

`GameMode` sits above this, loading a base scenario directory plus a list of addon directories (`GameMode.Config.baseModeDir` / `addonPaths`), and pulling three script kinds per directory: `textures.lua`, `behaviors.lua`, and `ruleset.lua`. Each addon's script "must return a function" that receives the accumulated table from earlier layers and returns a modified one — `current = lua.SafeCall(addon.Function, current)` — a middleware pipeline where each mod transforms the output of the mods loaded before it [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/GameMode.cs). The ruleset load path has a notable special case: it checks for a `ruleset.json` file first and falls back to `ruleset.lua` only if no JSON file exists, then — regardless of which source it started from — converts the final accumulated Lua table back to JSON text and deserializes it through `SaveGame.LoadFromJSON`. Lua is therefore a *content-authoring* language for OpenCiv3, not a runtime scripting language in the usual sense: a ruleset written in Lua compiles down to the same JSON `SaveGame` representation described in §2.3, and the engine only ever loads rulesets as data.

The reach given to that Lua is unusually broad. `ScriptInitializer`, run once at Lua-state initialization, reflects over `Assembly.GetExecutingAssembly().GetTypes()` filtered to `t.Namespace == "C7GameData"` and public visibility, and registers every one of them as MoonSharp `UserData` — plus every public enum in that namespace, flattened into a Lua `ENUMS` table with nested enums renamed `Outer_EnumName` (e.g. `Tile_YieldType`) to avoid collisions. It also installs a `GAME_DATA()` global function returning `EngineStorage.gameData` directly [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/GameMode.cs). Because registration is driven by reflection over an entire namespace rather than a hand-written binding table, a mod script has access to essentially the full `C7GameData` object graph through `GAME_DATA()`, not a curated subset of it — a much wider modding surface than a purpose-built, hand-curated API like Freeciv's `tolua`-bound `api_edit_*`/`api_server_*` functions (§3.4) or Luanti's `core.*` table (Ch205d, §3.2), at the cost of exposing internal engine state that a hand-curated binding could have kept private.

---

## 3. Freeciv: Network-Authoritative Client/Server and Text Rulesets

Freeciv (`github.com/freeciv/freeciv`, GPL, C) is the oldest project in this chapter and the one whose client/server split is the most literal. Its repository root splits cleanly into `client/`, `server/`, `common/`, `ai/`, `data/`, and `lua/` as top-level directories [Source](https://github.com/freeciv/freeciv) — the `ai/` directory is picked up in §3.5, and `lua/` and the Lua-scripting parts of `server/` and `data/` are picked up in §3.4.

### 3.1 civclient/civserver Separation

Freeciv's client (`civclient`) and server (`civserver`) are genuinely separate programs communicating over a network protocol. The project's own hacker's guide states this plainly: "Freeciv is a client/server civilization style of game. The client is pretty dumb. Almost all calculations are performed on the server" — and adds that this was a deliberate simplification over time, not the original design: "It wasn't like this always. Originally more code was placed in the common/ dir, allowing the client to do some of the world updates itself... Little by little we moved more code to the server" [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/doc/HACKING). This is the property freeciv-web (§3.3) inherits and builds a bridge around rather than reimplementing.

### 3.2 The .ruleset Text-File System

Freeciv's content lives in per-ruleset directories under `data/` — `data/classic`, `data/civ2civ3`, `data/civ1`, `data/civ2`, `data/multiplayer`, `data/sandbox`, `data/alien`, `data/goldkeep`, `data/granularity`, `data/hex2t`, `data/trident`, `data/default`, and `data/stub` all exist in the repository as of this writing, each a full parallel ruleset directory rather than a diff against a base [Source](https://github.com/freeciv/freeciv/tree/master/data). Individual `.ruleset` files are INI-style text with an explicit, in-file instruction for how to mod them. `data/classic/techs.ruleset` opens:

```
; Modifying this file:
; You should not modify this file except to make bugfixes or
; for other "maintenance".  If you want to make custom changes,
; you should create a new datadir subdirectory and copy this file
; into that directory, and then modify that copy.  Then use the
; command "rulesetdir <mysubdir>" in the server to have freeciv
; use your new customized file.

[datafile]
description = "Classic technology data for Freeciv (as Civ2, minus a few)"
options = "+Freeciv-ruleset-3.2-Devel-2022.Feb.02 web-compatible"
format_version = 30
```
[Source](https://raw.githubusercontent.com/freeciv/freeciv/master/data/classic/techs.ruleset)

Two things are worth pulling out of that header. First, the modding instruction is procedural, not declarative: there is no override/patch mechanism — a mod is a full copy of the ruleset directory, selected wholesale at server start via `rulesetdir`. Second, the `web-compatible` token inside `options` ties the ruleset format directly to freeciv-web's constraints (§3.3) — it is a marker some rulesets carry and others may not, meaning not every Freeciv ruleset is guaranteed to work unmodified through the web bridge.

### 3.3 freeciv-web as a Bridge, Not a Reimplementation

`freeciv-web` (`github.com/freeciv/freeciv-web`) is explicit in its own README that it wraps rather than replaces the C server: "Freeciv-web is an open-source turn-based strategy game... The Freeciv C server is released under the GNU General Public License, while the Freeciv-web client is released under the GNU Affero General Public License" [Source](https://github.com/freeciv/freeciv-web/blob/develop/README.md), a split confirmed by `LICENSE.txt`, which carries the full AGPLv3 text for the web client [Source](https://github.com/freeciv/freeciv-web/blob/master/LICENSE.txt). The README's own architecture description lists four components: "Freeciv-web" (a Java web application built with Maven, running on Tomcat 10 and nginx, implementing the browser-side UI and the Metaserver); "Freeciv" ("the Freeciv C server, which is checked out from the official Git repository, and patched to work with a WebSocket/JSON protocol"); "Freeciv-proxy" ("a WebSocket proxy which allows WebSocket clients in Freeciv-web to send socket requests to Freeciv servers... Implemented in Python"); and "Publite2" ("a process launcher for Freeciv C servers, which manages multiple Freeciv server processes and checks capacity through the Metaserver... Implemented in Python") [Source](https://github.com/freeciv/freeciv-web/blob/develop/README.md). The repository layout matches: `freeciv-proxy/` contains `civcom.py` and `freeciv-proxy.py`; `publite2/` contains `civlauncher.py`, `publite2.py`, and a set of paired `longturn_*.ruleset` / `pubscript_longturn_*.serv` files that configure specific long-running game instances [Source](https://github.com/freeciv/freeciv-web). Nothing in this stack forks or reimplements Freeciv's ruleset format; it patches the server's transport layer and puts a WebSocket/JSON proxy in front of the same `.ruleset`-driven server described in §3.2.

The two projects' maintenance cadence is not close. Querying Freeciv's commit API with `since=2025-08-01T00:00:00Z&per_page=1` and reading the commit count off the `Link: rel="last"` page number returns 502 commits to `master` [Source](https://api.github.com/repos/freeciv/freeciv/commits?since=2025-08-01T00:00:00Z&per_page=1); the equivalent query against freeciv-web's `develop` branch over the same twelve-month window returns an empty result set — **zero** commits [Source](https://api.github.com/repos/freeciv/freeciv-web/commits?sha=develop&since=2025-08-01T00:00:00Z&per_page=100). That is a wide and easily reproduced gap, not a subjective impression: the bridge project has gone a full year with no commits to `develop` while the server it wraps continues active development. (A reader checking only the repository's overall activity indicators — issues, pull requests, other branches — may see more recent-looking signals than this; the commit count above is scoped specifically to `develop`, the branch `github.com/freeciv/freeciv-web` redirects to as the repository's default.)

### 3.4 Lua Scripting Underneath the Text Rulesets

§3.2's `.ruleset` files are plain-data INI text, but they are not Freeciv's whole content-authoring surface, and describing the project as "pure data, no code" (as §6 previously did) understates it. `server/scripting/` compiles a curated, `tolua`-generated C API into the server's embedded Lua state: `api_server_edit.h` alone declares dozens of `api_edit_*` functions — `api_edit_unleash_barbarians`, `api_edit_unit_teleport`, `api_edit_create_city`, `api_edit_change_gold`, and more — each taking the live `lua_State *L` plus native `Tile`/`Player`/`Unit`/`City` pointers [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/server/scripting/api_server_edit.h). Every ruleset directory loads a `default.lua` shared across all rulesets (`data/default/default.lua`) plus, optionally, a ruleset-specific `script.lua` layered on top of it; `data/classic/script.lua` explains the relationship in its own header comment: "This file is for lua-functionality that is specific to a given ruleset. When freeciv loads a ruleset, it also loads script file called 'default.lua'. The one loaded if your ruleset does not provide an override is default/default.lua" [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/data/classic/script.lua).

The scripting model is event-driven rather than free-running. `data/classic/script.lua`'s only substantive content is a `city_destroyed_callback(city, loser, destroyer)` function registered against the engine's signal bus with `signal.connect("city_destroyed", "city_destroyed_callback")` [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/data/classic/script.lua) — a ruleset hooks named engine events and calls back into the curated `api_edit_*`/`api_server_*` surface, rather than the engine's live object graph being open to arbitrary reflection the way OpenCiv3's `GAME_DATA()` is (§2.4). Both projects embed Lua as a modding surface; they differ in how much of the engine that Lua can reach, and by what mechanism it is invoked. §6's comparison table reflects this rather than treating Freeciv as code-free.

### 3.5 AI Opponents: A Deliberately Out-of-Scope Axis

Both Freeciv and OpenCiv3 name an AI-opponent implementation as a first-class part of their architecture — Freeciv's own README-level folder list calls out `ai/`, and C7Engine's root README describes the C7Engine folder as covering "the mechanics of the game, including AI logic" (§2.1). Freeciv's `ai/` directory holds engine-wide difficulty and trait tuning (`difficulty.c`, `handicaps.c`, `aitraits.c`) alongside per-ruleset AI modules — `ai/classic/classicai.c` and `classicai.h` implement the classic ruleset's opponent, distinct from the `ai/default` and `ai/stub` variants [Source](https://github.com/freeciv/freeciv/tree/master/ai). OpenCiv3's `C7Engine/AI/` directory is organized around an `IAI.cs` interface with `PlayerAI.cs` and `BarbarianAI.cs` implementations, plus `StrategicAI`, `UnitAI`, and `Pathing` subdirectories for the layered decision-making beneath them [Source](https://github.com/C7-Game/OpenCiv3/tree/Development/C7Engine/AI).

Neither AI system is a data-driven ruleset in the sense §1.2 and §6 use the term — both are engine-native code a mod cannot redirect without recompiling, which is precisely why this chapter, scoped to data models, rules engines, networking, and modding surfaces, does not analyze them further here. The classical game-AI techniques underneath implementations like these — navmeshes, behaviour trees, GOAP, utility systems — are covered on their own terms in Chapter 205b, which this chapter defers to rather than duplicating.

---

## 4. OpenXcom/OXCE: YAML Rulesets as the Entire Content Pipeline

OpenXcom (`github.com/OpenXcom/OpenXcom`) and its most actively developed fork, OpenXcom Extended (OXCE, `github.com/MeridianOXC/OpenXcom`, `oxce-plus` branch), are both GPL-3.0 [Source](https://api.github.com/repos/OpenXcom/OpenXcom) [Source](https://api.github.com/repos/MeridianOXC/OpenXcom). This section is honest about scope up front: OpenXcom's rules engine is a deeply moddable single game's rules engine, not a generalized multi-game SDK the way OpenCiv3 or Freeciv nominally are. What makes it worth a full section anyway is how far "deeply moddable" turns out to go.

### 4.1 The bin/standard/&lt;ModName&gt;/*.rul Convention

Every unit of content in the OXCE tree — official and third-party alike — is a directory under `bin/standard/` containing a `metadata.yml` and one or more `.rul` YAML files. A small, complete example, `Smarter_Equip`, patches one property of one existing inventory-priority list:

```yaml
# metadata.yml, bin/standard/Smarter_Equip
name: "Smarter Equip"
version: 1.1
description: "Custom priority order of inventory slots for auto-equip and ctrl-click-equip."
master: xcom1
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/Smarter_Equip/metadata.yml)

`master: xcom1` declares that this mod depends on, and is layered over, the base game's own ruleset directory, `bin/standard/xcom1`, which is itself just another directory under `bin/standard/`, containing files like `items.rul` in exactly the same YAML `.rul` format any third-party mod uses [Source](https://github.com/MeridianOXC/OpenXcom/tree/oxce-plus/bin/standard/xcom1). There is no privileged "engine format" distinct from "mod format": the base game ships its own ruleset through the identical pipeline a total-conversion mod uses. `Smarter_Equip`'s own `.rul` file shows what a mod actually overrides — a per-item inventory placement, and a set of category-level display-order lists:

```yaml
# 1_Smarter_Equip.rul, bin/standard/Smarter_Equip
items:
  - type: STR_HIGH_EXPLOSIVE
    defaultInventorySlot: STR_RIGHT_LEG
  - type: STR_MOTION_SCANNER
    defaultInventorySlot: STR_BELT
  - type: STR_MEDI_KIT
    defaultInventorySlot: STR_BELT
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/Smarter_Equip/1_Smarter_Equip.rul)

A second, smaller example shows the same key-based merge in miniature — `XCOM_Damage` overrides a single global constant rather than an item field:

```yaml
# XCOM_Damage.rul, bin/standard/XCOM_Damage
constants:
  - damageRange: 100
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/XCOM_Damage/XCOM_Damage.rul)

Both examples show the same pattern: a mod's `.rul` file names an existing key under an existing top-level section (`items`, or a single entry under `constants`) and supplies a new value; the loader's merge semantics, not a scripting layer, do the actual patching. This is a materially different modding model from OpenCiv3's addon-as-Lua-function pipeline (§2.4): there is no code path a mod author can execute, only declarative overrides the engine's YAML loader applies in dependency order via each mod's `master` field.

### 4.2 Total-Conversion Mods and the Mod Portal

OXCE's own README points modders at a dedicated portal: "There is also a [mod portal website](https://openxcom.mod.io/) with a thriving mod community with hundreds of innovative mods to choose from" [Source](https://github.com/MeridianOXC/OpenXcom/blob/oxce-plus/README.md). Two of the best-known total conversions built on this pipeline, XPiratez and X-Com Files, both have active listing pages on that portal (`openxcom.mod.io/xpiratez`, `openxcom.mod.io/x-com-files`) — total conversions that replace nearly all of the base game's content while still loading through the same `bin/standard/<ModName>/*.rul` mechanism described in §4.1, distinguished from a small tweak like `Smarter_Equip` only by how much of the ruleset tree they touch, not by any different loading mechanism.

### 4.3 OXCE vs. Base OpenXcom

Both repositories' push timestamps are only about seven weeks apart as of this writing, which is not, by itself, evidence of a maintenance gap. Commit *volume* over the same window is a sharper test: querying each repository's commit API with `since=2025-08-01T00:00:00Z&per_page=1` and reading the commit count off the `Link: rel="last"` page number returns 175 commits to OXCE's `oxce-plus` branch [Source](https://api.github.com/repos/MeridianOXC/OpenXcom/commits?sha=oxce-plus&since=2025-08-01T00:00:00Z&per_page=1) versus 41 commits to base OpenXcom's `master` in the same period [Source](https://api.github.com/repos/OpenXcom/OpenXcom/commits?since=2025-08-01T00:00:00Z&per_page=1) — a factor of roughly four. OXCE is, by that measure, the practically live branch of this codebase, consistent with it being the fork most total-conversion mods target and document compatibility against.

---

## 5. Thousand Parsec: Protocol-First Rulesets (An Instructive Dead End)

Thousand Parsec is GPL-2.0 and organizationally scattered across a set of small repositories under the `thousandparsec` GitHub organization, the most central of which — `tpserver-cpp`, the C++ server — was last pushed on 2011-05-03 and has not been archived by its maintainers, but has recorded no commits since that date [Source](https://api.github.com/repos/thousandparsec/tpserver-cpp). Unlike every other project in this chapter, it separates not just client from server but *ruleset from engine*, at the level of a network protocol rather than a data format.

The organization's repository list is itself the evidence for this design: `libtpproto-cpp` ("a client-side protocol and game library in C++"), `libtpproto-py` and `libtpproto2-py` (Python implementations of the same protocol), `tpclient-pywx` ("the primary client, uses wxPython to look like a native application on Windows, Mac OS X and Linux"), `tpadmin-cpp` ("an administration client for servers supporting the configuration protocol"), and `daneel-ai` ("An advanced rule based AI for RFTS (and eventually other rulesets)") [Source](https://api.github.com/orgs/thousandparsec/repos) — a client, a server, an admin tool, and an AI player, each an independent program speaking the "TP protocol" rather than linked-in modules of one binary. `daneel-ai`'s own description makes the ruleset-scoping explicit: it is written for one ruleset (RFTS) with other rulesets as a stated future goal, because a different ruleset can change what a valid AI decision even looks like.

The ruleset itself lives inside `tpserver-cpp` as compiled, swappable C++ modules under `modules/games/`: `minisec`, `mtsec`, `rfts`, `risk`, and `tae` [Source](https://github.com/thousandparsec/tpserver-cpp/tree/master/modules/games). `minisec` alone comprises over a dozen `.cpp`/`.h` pairs — `build`, `colonise`, `combatant`, `fleet`, `intercept`, `mergefleet`, `move`, `planet`, `rspcombat`, `spaceobject`, `splitfleet`, `universe`, plus a `minisec`/`minisecturn` pair tying the module together [Source](https://github.com/thousandparsec/tpserver-cpp/tree/master/modules/games/minisec) — a full compiled ruleset implementation, not a data file. Where Freeciv's `.ruleset` and OXCE's `.rul` let a content author fill in an existing engine's blanks, Thousand Parsec's rulesets are themselves game logic written in the server's own implementation language, loaded as a named module at server startup. That is the maximum point on §6's moddability-vs-recompilation axis: nothing is expressible without a C++ compiler, but in exchange nothing about the game's *rules* is fixed by the engine at all — only the protocol between programs is. It is the one project surveyed here that pushed this far, and it is also the one project surveyed here with no commits in over a decade. That correlation is not proof that protocol-level ruleset-as-module design is unworkable — one project is one data point, and small open-source games fail for many reasons having nothing to do with their modding architecture — but it is the most architecturally ambitious entry in this chapter and the only one that did not survive.

---

## 6. Data-Driven Rules Compared

| | Format | Moddable without recompiling? | Ceiling before engine code changes | Client/server topology | Maintenance/tooling burden |
|---|---|---|---|---|---|
| **Freeciv** `.ruleset` + Lua | INI-style text per-directory, plus a curated `tolua`-bound Lua API for event-driven logic (§3.4) | Yes — copy a ruleset directory, edit values, select via `rulesetdir`; a ruleset's `script.lua` can add `signal.connect`-hooked logic without a server rebuild | High but bounded: plain-data fields are capped by what the format exposes, and Lua hooks are capped by what `api_edit_*`/`api_server_*` exposes — a genuinely new mechanic still needs C server changes | Genuinely separate `civclient`/`civserver` processes over a network protocol (§3.1); freeciv-web proxies that same protocol over WebSocket (§3.3) without altering it | Low to author values; moderate to author Lua hooks, which require learning the signal-event model and the curated API surface |
| **OXCE** `.rul` | YAML, per-directory with `master` dependency declarations | Yes — the base game and every mod use the identical format and merge mechanism | Similarly bounded by what keys the loader recognizes, but the surface is large enough that total conversions (XPiratez, X-Com Files) replace nearly all content without touching engine code | None — a single process; OpenXcom has no server/client split | Low to author individual mods; YAML key/merge semantics scale to total conversions without new tooling |
| **OpenCiv3** embedded Lua | Lua scripts (`ruleset.lua`, `behaviors.lua`, `textures.lua`) compiled to a JSON `SaveGame` at load time | Yes, and further: mods are executable code with reflection-derived access to the live `C7GameData` object graph via `GAME_DATA()`, not just declarative values | Very high — a mod can implement arbitrary logic MoonSharp can express, bounded only by what `ScriptInitializer` exposes as userdata | Single process today; the `MessageToEngine`/`MessageToUI` queues (§2.2) are structured for a future client/server split but do not implement one | Higher: authoring requires understanding both Lua and the reflected C# type surface; the addon-pipeline model (§2.4) also makes load order across mods significant |
| **Thousand Parsec** protocol modules | Compiled C++ (`modules/games/*`), selected by the server at startup, speaking a network protocol to independent client/AI programs | No — a new ruleset is a new compiled module, not a data file | None in principle — a module can implement anything the server process can, since it *is* server code | Network protocol *is* the integration surface — client, server, admin tool, and AI player (`daneel-ai`) are independent programs (§5) | Highest: requires a C++ toolchain and the server's internal module API; the project did not survive to demonstrate this at scale |

Reading down the "moddable without recompiling" column against the "maintenance burden" column shows the trade-off directly: the two projects whose primary modding path is plain data-file editing (OXCE's YAML, and Freeciv's `.ruleset` text for most day-to-day balance work) are also the two still under active development, while the project offering the most powerful modding surface (Thousand Parsec, compiled swappable rulesets) is the one that stopped. OpenCiv3 and Freeciv both also embed Lua, and comparing them is instructive precisely because it isolates the code-vs-data axis from the active-vs-dead one: OpenCiv3 exposes its entire `C7GameData` object graph to mod-authored Lua by reflection, while Freeciv's Lua reaches the engine only through the curated, `tolua`-bound `api_edit_*`/`api_server_*` surface and only in response to named signals (§3.4) — a hand-curated API versus an open reflection surface, not "no code" versus "code." OXCE's YAML merging remains the one format in this chapter with no code path at all for a mod author to execute. The client/server column is a separate axis again: it tracks with neither modding power nor project health — the actively developed Freeciv is network-split and the actively developed OXCE is not — which is the point of listing it separately rather than folding it into "maintenance burden." All of this is addressed more generally for engine modding across languages in Ch205d.

---

## 7. Brief Survey: FreeCol, FIFE/Unknown Horizons, Wesnoth WML

**FreeCol** (`github.com/freecol/freecol`, Java) is a Colonization-inspired 4X built on a directory structure that closely parallels Freeciv's: `data/rules/` holds ruleset variants (`classic`, `freecol`, `testing`), and a separate `data/mods/` holds nineteen distinct mod directories layered over a base ruleset [Source](https://github.com/freecol/freecol/tree/master/data). Its `LICENSE` file carries the GPLv2 text; GitHub's own SPDX detection reports the license as plain `gpl-2.0` rather than "or later" — this chapter states GPL-2.0 rather than the "GPL-2.0-or-later" sometimes cited for the project, since an "or later" grant was not independently confirmed in the fetched license text.

**FIFE** (`github.com/fifengine/fifengine`, LGPL-2.1, C++/Python) is a nominally reusable isometric-engine SDK, and its most prominent adopter, **Unknown Horizons**, is the caution rather than FIFE itself: FIFE's repository was pushed within the last two weeks and is not archived, so describing the engine itself as dead would overstate the evidence. What is well documented is that Unknown Horizons moved away from it — the `unknown-horizons` GitHub organization hosts a separate, still-active `godot-port` repository (last pushed 2025-12-04, not archived) alongside the original FIFE-based `unknown-horizons` repository (last pushed 2026-04-14, also not archived) [Source](https://github.com/unknown-horizons). Both repositories are alive by GitHub's activity metrics; the FIFE-based version has simply not been the project's forward direction since a Godot rewrite began. The caution this chapter draws from that is narrow and specific: a "reusable engine" claim is only as strong as its evidence, and a single flagship adopter migrating away — even without the underlying engine being abandoned — is worth checking rather than assuming still holds.

**Battle for Wesnoth** (`github.com/wesnoth/wesnoth`, GPL-2.0) is the most actively maintained repository surveyed in this chapter and the most widely cited example of declarative, macro-driven campaign scripting in a FOSS game. Its WML (Wesnoth Markup Language) format uses a C-preprocessor-like layer of `#define`/`{MACRO}` expansion and `#ifdef` conditionals over a tag-based content language, visible in a representative campaign scenario file:

```
#textdomain wesnoth-h2tt

#define SCENARIO_TURN_LIMIT
{ON_DIFFICULTY4 18 17 16 15}#enddef

[scenario]
    id=01_The_Elves_Besieged
    map_file=01_The_Elves_Besieged.map
    name=_"The Elves Besieged"
    next_scenario=00_The_Great_Continent # no-syntax-rewrite
```
[Source](https://github.com/wesnoth/wesnoth/blob/master/data/campaigns/Heir_To_The_Throne/scenarios/01_The_Elves_Besieged.cfg)

`{ON_DIFFICULTY4 ...}` is a macro call resolving to a per-difficulty-level turn limit, and `#ifdef` conditionals elsewhere in the same file gate content on which mainline eras or add-ons are active. WML sits architecturally closer to Freeciv's text-file rulesets than to OpenCiv3's embedded Lua — it is a declarative content format with a macro preprocessor layered on top, not a general-purpose scripting language — but the preprocessor gives mod authors more structural reuse (parameterized macros, conditional compilation) than a flat INI file offers, at the cost of a format that is Wesnoth-specific rather than a general pattern the way YAML or Lua are.

---

## 8. How Mods Compose: Load Order and Conflict Resolution

Every section so far describes how *one* ruleset or mod plugs into its engine (§2.4, §3.2, §4.1, §5). It says nothing about what happens once a player enables more than one at a time — whether order is significant, whether one mod can see what an earlier mod already changed, and whether the engine notices when two mods edit the same thing. That turns out to range from "not a concept that exists yet" to an explicit, engine-enforced dependency chain with per-field override semantics.

### 8.1 OpenCiv3: Declaration-Order Pipeline, No Conflict Detection

`GameMode.Config.addonPaths` is a plain `List<string>`, and `GameMode.Load`'s addon loader walks it with a `foreach`:

```csharp
// C7Engine/Lua/GameMode.cs, OpenCiv3 (Development branch)
foreach (string addonPath in config.addonPaths) {
    // ...
    current = lua.SafeCall(addon.Function, current);
}
```
[Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/GameMode.cs)

There is no alphabetical sort, no dependency graph, and no explicit dependency-declaration field anywhere in `GameMode.Config` — load order is exactly declaration order in that list. The companion doc confirms this is the whole mechanism: "Addons are applied in the order listed in `addonPaths`, and each one only needs to provide the scripts it actually wants to change" [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7/Lua/README.GameModes.md). The same doc's own TODO admits there is no user-facing mod system yet to assemble that list from installed mods: "TODO: Implement a code-change free mod loading mechanism" — today `addonPaths` lists are hardcoded in C# (`C7/GlobalSingleton.cs`), not player configuration.

Because each addon's script "must return a function" that receives the accumulated table produced by every mod loaded before it and returns a modified one (§2.4), a later addon has unrestricted read/write access to whatever an earlier addon already produced. That gives OpenCiv3 no engine-level conflict detection at all: if two addons both target the same key, the later one in `addonPaths` order simply overwrites it, silently, by construction — correctness is left entirely to authors coordinating addon order themselves, with nothing in the engine to warn them if they don't.

### 8.2 OXCE: Explicit master Chains and Per-Field Override Keywords

OXCE's `master` field (§4.1) is not limited to declaring a dependency on the base game — it chains. `ModInfo.cpp`'s own comment states this directly: "masters can still have masters, but they must be explicitly declared" [Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Engine/ModInfo.cpp). `FileMap::setup()` walks that chain for every active mod and expands it into the actual load order:

```cpp
// src/Engine/FileMap.cpp, OXCE (oxce-plus branch)
while (true) {
    insert_before = map_order.insert(insert_before, currentId);
    masterId = /* ...modInfo.getMaster()... */;
    if (masterId.empty()) break;
    currentId = masterId;
}
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Engine/FileMap.cpp)

— a comment in the same function describes exactly what this is for: "expand the mod list into map order, listing all the mod dependencies." No independent `requires:`/`dependencies:` field exists beyond `master`, `requiredMasterModVersion`, `requiredExtendedVersion`, and `requiredExtendedEngine` — a targeted search of `ModInfo.h`/`.cpp` found nothing broader. Among mods that don't have a `master`-chain relationship to each other, load order is simply the order of the `mods:` sequence in `options.cfg`, and it is directly reorderable in-game: `ModListState::moveModUp`/`moveModDown` let a player drag entries up and down a list with the mouse wheel or buttons [Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Menu/ModListState.cpp).

Conflict resolution, unlike OpenCiv3's silent overwrite, is a designed part of the loader. `Mod::loadRule` in `src/Mod/Mod.cpp` reuses the existing rule object if a prior-loaded mod already created an entry of the same `type:`, so a later mod's fields overwrite in place on the same object rather than replacing it wholesale:

```cpp
// src/Mod/Mod.cpp, OXCE (oxce-plus branch) — loadRule, abbreviated
if (i != map->end()) {
    rule = i->second; // reuse the object an earlier-loaded mod already created
}
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Mod/Mod.cpp)

Authors get finer-grained control than plain overwrite-in-place through explicit `new`/`override`/`update`/`delete`/`ignore` keywords on an entry; combining two of these on the same node is caught and rejected with `"Conflict of main node X and Y"` [Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Mod/Mod.cpp) — though that check is a syntax validation inside one mod's own file, not detection of a genuine cross-mod collision between two independently authored mods touching the same key.

### 8.3 Freeciv and Thousand Parsec: No Runtime Composition

Neither of the other two projects in this chapter has a runtime mod-stacking concept to compare against §8.1/§8.2 at all. Freeciv's `.modpack` files are consumed only by a separate installer tool, not by `civserver` itself, and dependency resolution happens strictly at *install* time: `doc/README.modpack_installer` states "the tool will download the files for the selected modpack, and any other modpacks it depends on" [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/doc/README.modpack_installer) — once installed, a player still selects one whole ruleset directory at server start via `rulesetdir` (§3.2), and nothing composes multiple rulesets together at runtime. Thousand Parsec is narrower still: `tpserver-cpp`'s `main.cpp` reads a single `ruleset` config value and `PluginManager::loadRuleset` `dlopen`s exactly one shared library named after it [Source](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/tpserver/main.cpp) — there is no list to order, and `sample.conf` documents the setting as singular: "ruleset - sets which ruleset to load... If it is not set, no ruleset is loaded" [Source](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/tpserver/pluginmanager.cpp).

Read together with §8.1 and §8.2, the four projects span a genuine spectrum on this axis, not just a yes/no split: Thousand Parsec and Freeciv treat "the ruleset" as a single, atomically-selected unit with no composition step at all; OpenCiv3 composes multiple mods but leaves ordering and conflict resolution entirely to author discipline; OXCE composes multiple mods *and* gives the engine an explicit dependency chain (`master`), a reorderable load list, and per-field override keywords with syntax-level conflict detection. That last point is also OXCE's most direct architectural link to §4's total-conversion mods (XPiratez, X-Com Files): a mod ecosystem with dozens of simultaneously-installable mods stacked over one `master` base is only tractable because the engine, not just convention, enforces load order and merge semantics.

---

## 9. Save-Game Version Compatibility

A ruleset or mod format is only half the compatibility story; the other half is what happens when a save file was written against an older version of that format, or against a mod combination that has since changed. The four projects take four distinct, and instructively different, positions on this — from genuine forward migration to an explicit refusal to even try.

### 9.1 Freeciv: Forward Migration Through a Versioned Compat Table

Freeciv has a dedicated compatibility module, `server/savegame/savecompat.c`. `sg_load_compat()` reads the save's `savefile.version` and then walks an ordered `compat[]` table, running every migration function whose version number exceeds the save's own:

```c
/* server/savegame/savecompat.c, Freeciv (master) */
loading->version = secfile_lookup_int_default(loading->file, -1, "savefile.version");
sg_failure_ret(0 < loading->version && loading->version <= compat[compat_current].version,
               "Unknown savefile format version (%d).", loading->version);
for (i = 0; i < compat_num; i++) {
    if (loading->version < compat[i].version && compat[i].load != NULL) {
        compat[i].load(loading, format_class);
    }
}
```
[Source](https://raw.githubusercontent.com/freeciv/freeciv/master/server/savegame/savecompat.c)

A save newer than the running build's own `compat_current` is refused outright in release builds (a `FREECIV_DEBUG` build instead logs a warning and attempts to load it anyway). Ruleset compatibility is handled by an analogous mechanism one layer down: `server/ruleset/rscompat.c`'s `rscompat_check_cap_and_version()` checks each `.ruleset` file's `datafile.format_version` against the current `RSFORMAT_3_4` constant defined in `server/ruleset/ruleload.h`, and a whole ruleset directory older than that is migrated forward via `compat_mode` functions gated on the same version check [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/server/ruleset/rscompat.c). `savegame3.c`'s `sg_load_savefile()` reloads the exact ruleset directory named in the save's own `savefile.rulesetdir` field, so a save's ruleset dependency is resolved by re-loading that ruleset by name — through the same forward-migration path — rather than by re-checking it against whatever ruleset happens to be configured on the server at load time [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/server/savegame/savegame3.c).

### 9.2 OXCE: No Version Gate, Silent Best-Effort Loading

OXCE takes a much looser position. `SavedGame::save()` writes each active mod into the save as `modId + " ver: " + modVersion` in a `mods` YAML list, but `SavedGame::load()` only uses that list for save-browser filtering, not as a load-time gate — and even that filtering check strips the version before comparing:

```cpp
// src/Savegame/SavedGame.cpp, OXCE (oxce-plus branch) — _isCurrentGameType, abbreviated
std::string name = SavedGame::sanitizeModName(modName);
if (name == curMaster) { matchMasterMod = true; break; }
```
[Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Savegame/SavedGame.cpp)

`sanitizeModName` discards the version suffix entirely — a save's declared mod *version* is never actually checked, only whether the current master mod's name is present in the list at all. Inside the field-by-field YAML load itself, an entity that no longer exists in the currently loaded ruleset is dropped with a log line rather than aborting the load: `if (mod->getCountry(type)) { ... } else { Log(LOG_ERROR) << "Failed to load country " << type; }` [Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Savegame/SavedGame.cpp). `src/version.h` separately defines `MIN_REQUIRED_RULESET_VERSION_NUMBER 8,6,3,0`, but that constant gates a *mod's* compatibility against the *engine* at mod-load time — it has nothing to do with save-file version checking [Source](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/version.h). Loading a save written under a different mod configuration is, in short, a best-effort partial load with per-entity error logging, not a version-gated operation.

### 9.3 OpenCiv3: A Version Field That Is Dead Code

`SaveGame` declares `public string Version = "0.0.0";`, which reads like the beginning of exactly the kind of compat mechanism §9.1 and §9.2 have — except it is never assigned inside `SaveGame.FromGameData()` and never read or compared anywhere in the load path (`Load`/`LoadFromJSON`); a repository-wide search turns up no other reference to `SaveGame.Version` at all [Source](https://raw.githubusercontent.com/C7-Game/OpenCiv3/Development/C7Engine/C7GameData/Save/SaveGame.cs). The consequence shows up directly in the issue tracker: closed issue #434, "Loading Incompatible Save Ruins New Game," describes exactly the failure mode an unenforced version field predicts — "Data loaded from an incompatible save persists until the program is restarted... Here I have altered terrain from the scenario I loaded. However, the units can not be moved due to incompatibilities" [Source](https://github.com/C7-Game/OpenCiv3/issues/434). That is consistent with the root README's own framing of save/load as a feature still being merged in incrementally (§2.1) — save-format compatibility here is not a designed policy at all yet, just an open bug.

### 9.4 Thousand Parsec: A Hard Version Gate, With One Exception

Thousand Parsec's persistence is not purely in-memory: `tpserver-cpp`'s `modules/persistence/mysql/mysqlpersistence.cpp` backs the server with a MySQL database and tracks its own schema version in a `tableversion` table. On reconnecting to an existing database with an old schema, it refuses outright rather than attempting a migration:

```cpp
// modules/persistence/mysql/mysqlpersistence.cpp, tpserver-cpp (master)
Logger::getLogger()->error("Old database format detected.");
Logger::getLogger()->error("Incompatable old table formats and missing tables detected.");
Logger::getLogger()->error("Changes to most stored classes means there is no way to update "
                            "from your current database to the newer format");
Logger::getLogger()->error("I cannot stress this enough: Please shutdown your game, delete "
                            "the contents of the database and start again. Sorry");
Logger::getLogger()->error("Mysql persistence NOT STARTED");
```
[Source](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/modules/persistence/mysql/mysqlpersistence.cpp)

There is exactly one carve-out: a single `ALTER TABLE gameinfo ADD COLUMN turnname` migration runs when the `gameinfo` table's own tracked version is `0`, the one place in the file that migrates forward instead of refusing [Source](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/modules/persistence/mysql/mysqlpersistence.cpp). That makes Thousand Parsec's posture the hard-refuse end of the spectrum this section surveys: not silent best-effort loading (OXCE), not an unenforced field masking real incompatibility bugs (OpenCiv3), and not general forward migration (Freeciv) — a version check that stops the server rather than risk loading against a schema its own code no longer understands, with one narrow, explicitly-versioned exception rather than a general migration framework.

Read across all four, the spread is really a spread of engineering investment rather than four unrelated designs: Freeciv's `savecompat.c`/`rscompat.c` pair represents the only project here that treats "an old save/ruleset combination should still load" as a maintained, ongoing engineering commitment; Thousand Parsec's hard refusal is the same underlying judgment — that migrating an unknown-shape save silently is worse than not loading it — applied without the migration half; OXCE's silent best-effort load optimizes for a save *usually* still loading fine across small mod-version bumps at the cost of no hard guarantee; and OpenCiv3's dead `Version` field is what §9.1–§9.4's design space looks like before anyone has had to solve it yet.

---

## 10. Mod Distribution and Discovery Channels

§4.2 already covers OXCE's mod-sharing channel, the dedicated `openxcom.mod.io` portal hosting "hundreds of innovative mods" including total conversions like XPiratez and X-Com Files. That leaves an open question for the rest of this chapter's projects: once a mod or ruleset exists, how does anyone else find it? The answer ranges from a purpose-built installer tool to no channel existing at all — including, in one case, no settled definition yet of what a distributable unit of content even is.

### 10.1 Freeciv: A Dedicated Modpack Installer and Repository

Distinct from the "copy the ruleset directory manually" workflow §3.2 documents for `.ruleset` text-file edits, Freeciv ships a separate GUI tool — `freeciv-mp-gtk3`/`freeciv-mp-qt`, the "modpack installer" — that fetches a curated listing from `modpack.freeciv.org` and installs rulesets, scenarios, tilesets, soundsets, or musicsets automatically into the right data directory. The tool is not restricted to that official list: the project's own doc states modpacks can also be installed from arbitrary third-party `.mpdl`/`.list` URLs [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/doc/README.modpack_installer). Alongside the installer, an active forum sub-board — "Rulesets and modpacks," described in its own header as a place to "contribute, display and discuss rulesets and modpacks for use in Freeciv" — currently holds 329 topics [Source](https://forum.freeciv.org/f/viewforum.php?f=11). The honest picture is two channels serving two different needs: a tool-mediated, curated repository for polished, install-ready content, and a forum for raw sharing and in-progress discussion.

### 10.2 OpenCiv3: No Channel Yet — The Packaging Format Itself Is Unresolved

No mod-sharing forum, wiki page, or directory convention exists for OpenCiv3 mods, and the clearest evidence for why is that the packaging question a distribution channel would presuppose is still open. GitHub issue #133, "Prototype Mod," open since 2022-03-12, asks directly "what form this should take" and "is a 'mod' or 'scenario' a collection of those [files]?" [Source](https://github.com/C7-Game/OpenCiv3/issues/133) — the same open question §8.1 found reflected in `README.GameModes.md`'s own "code-change free mod loading mechanism" TODO. The project wiki's page list (Home, Contributing, Developer Guide, Game Assets, Notes on game features) has nothing on mod sharing [Source](https://github.com/C7-Game/OpenCiv3/wiki). A distribution channel is downstream of a stable definition of what gets distributed, and OpenCiv3 does not have the latter yet.

### 10.3 Thousand Parsec: No Third-Party Distribution Ever Emerged

The `thousandparsec/wiki` repository's `Clone_Ruleset.mediawiki` page discusses ruleset *concepts* — a Diplomacy-style ruleset, a Master-of-Orion-style ruleset — as ideas for future implementation directly inside the server codebase, never as something to download or share separately from the project's own source tree [Source](https://github.com/thousandparsec/wiki/blob/master/Clone_Ruleset.mediawiki). Given §5's and §8.3's finding that a Thousand Parsec ruleset is a compiled C++ module `dlopen`ed by name, this is the expected consequence rather than a separate gap: authoring a new ruleset already requires being a `tpserver-cpp` contributor with a C++ toolchain, which is a fundamentally different starting point from a text-file or YAML mod a player installs after the fact. No distribution mechanism beyond the project's own repository ever developed before development stopped in 2011.

### 10.4 FreeCol: A Separate Community-Mod Repository

FreeCol is the one project in this section with a genuine external distribution channel, structurally distinct from the nineteen mods `data/mods/` already ships (§7). `github.com/FreeCol/freecol-mods` is a separate repository specifically for third-party content, with README instructions telling authors to "submit a pull-request with your mod and description added, along with a link to the download," adding that maintainers "can also create a github release with your mod package on request" [Source](https://github.com/FreeCol/freecol-mods/blob/master/mods/README.md). Its one tracked entry as of this writing, "New Nations by Mazim," is sourced from the FreeCol SourceForge forums — genuinely third-party content brought into a curated repository, rather than first-party content the core team wrote and bundled.

Read against §4.2's OXCE mod.io portal, the picture across all five projects splits cleanly by whether distribution infrastructure was ever built at all, and that split tracks project activity more closely than it tracks how expressive the modding format itself is: OXCE (a full portal) and Freeciv (an installer plus a forum) both actively invest in discovery infrastructure; FreeCol has a minimal but real one; OpenCiv3 and Thousand Parsec have none, for two different reasons — OpenCiv3 because the underlying packaging question is still unsettled, Thousand Parsec because its ruleset format never lowered the bar enough for a non-contributor to produce one in the first place.

---

## 11. Turn-Based Multiplayer Beyond Client/Server: Longturn, PBEM, and Deadline Resolution

§3.1's `civclient`/`civserver` split and §5's protocol-first Thousand Parsec architecture both describe *how* a client talks to a server, but neither says anything about *when* a turn actually advances once more than one human is involved and they are not all online at once. Three genuinely different concurrency models turn up across this chapter's projects, plus two projects that never left synchronous, all-players-present play at all.

### 11.1 Freeciv Longturn: Concurrent Play With a Very Long Timer

"Longturn" is a server-configuration mode built entirely from settings Freeciv already exposes, not a separate code path. The `timeout` server setting — "the number of seconds that is allowed for each player to make their moves. If this is set to zero, then players will have unlimited time" — can be set as high as 8,640,000 seconds (100 days) [Source](https://raw.githubusercontent.com/freeciv/freeciv/master/server/settings.c), and community Longturn servers document setting it to roughly 23 hours per turn [Source](https://longturn.readthedocs.io/en/latest/Playing/turn-change.html). Movement stays concurrent rather than alternating — moves from all connected players are processed in the order received, and a turn ends either when the timer expires or when every player has ended their turn [Source](https://longturn.readthedocs.io/en/latest/Playing/faq.html). freeciv-web's own `publite2/` directory, already cited in §3.3 for its process-launcher role, ships the concrete evidence of this in production: paired `longturn_*.ruleset`/`pubscript_longturn_*.serv` files configuring specific long-running game instances. Longturn is, in short, ordinary concurrent Freeciv multiplayer with the per-turn clock stretched from minutes to a day — a persistent server players check in on periodically, not a different network model.

### 11.2 Freeciv PBEM: Strict Alternation, Not Longturn Under Another Name

Freeciv also ships a genuinely distinct third mode, easy to conflate with longturn but architecturally different. freeciv-web's own README lists them as separate offerings side by side: "Single player, Multiplayer, Longturn, Play-by-Email, Hotseat" [Source](https://github.com/freeciv/freeciv-web/blob/develop/README.md). Where longturn keeps concurrent movement with a long timer, PBEM enforces strict per-player alternation — a `pbem/pbem.py` daemon in the same repository watches for saved-game files, determines whose turn it now is, and emails that player a link to continue. The mechanism is server-backed turn notification via email, not literal save-file-as-email-attachment exchange the "play-by-email" name might suggest historically. The distinction matters architecturally: longturn is a concurrency-model choice (everyone moves within a long window), while PBEM is a turn-order enforcement choice (only one player may act at a time) layered with an asynchronous notification mechanism — two independent axes that happen to both solve the same underlying problem of players who are never online simultaneously.

### 11.3 FreeCol and OXCE: Outside This Axis Entirely

Two of this chapter's projects simply never built an asynchronous mode to compare against §11.1/§11.2/§11.4. FreeCol's server code (`src/net/sf/freecol/server/{control,networking,...}`) describes only live client/server multiplayer, with nothing in its README, wiki, or javadoc resembling a PBEM or longturn-style deadline mode [Source](https://www.freecol.org/javadoc/net/sf/freecol/server/FreeColServer.html) — an absence-of-evidence finding, not a developer statement that async play is unsupported, but a consistent one across every primary source checked. OXCE's position is more structurally certain: its `src/` tree has no networking directory at all — `Basescape`, `Battlescape`, `Engine`, `Geoscape`, `Interface`, `Menu`, `Mod`, `Savegame`, `Ufopaedia` cover the whole engine [Source](https://github.com/MeridianOXC/OpenXcom/tree/oxce-plus/src) — confirming §4's existing characterization of OXCE as having no server process, synchronous or otherwise, to build a deadline-based mode into. Both projects sit outside this section's comparison for the same underlying reason: an asynchronous turn model presupposes a server-authoritative process to hold state between sessions, which FreeCol's architecture technically has but never used this way, and which OXCE's architecture does not have at all.

### 11.4 Thousand Parsec: Simultaneous Resolution on a Deadline

Thousand Parsec's model is a third, architecturally distinct pattern from both of Freeciv's. The *Architecture of Open Source Applications* chapter on Thousand Parsec, written by the project's own developers, describes turn advancement as genuinely simultaneous and deadline-bounded: "When a player has finished performing actions, he or she may signal readiness for the next turn via the `Finished Turn` request; the next turn is computed when all players have done so," and turns additionally "have a time limit imposed by the server, so that slow or unresponsive players cannot hold up a game" [Source](https://aosabook.org/en/v1/thousandparsec.html). That is neither Freeciv longturn's concurrent-with-long-timer model (where individual moves apply as they arrive) nor its PBEM's strict alternation — every player submits orders independently and the server resolves them all at once, either when everyone has explicitly finished or when the deadline forces resolution regardless. It is the only one of the three models surveyed here where no player's action is ever visible to another before the turn resolves.

Across all four live comparison points, the chapter's projects turn out to cover three structurally different answers to "how do turn-based strategy games handle players who are never online at the same time": process-moves-as-they-arrive-with-a-long-clock (Freeciv longturn), strict-alternation-with-async-notification (Freeciv PBEM), and simultaneous-orders-resolved-on-a-deadline (Thousand Parsec) — with FreeCol and OXCE never needing to pick any of the three, because neither ever built the server-authoritative persistence layer the choice presupposes.

---

## Integrations

**Chapter 205d — Modding Architectures** covers the general engineering problem this chapter's frameworks each answer once, for one domain: embedded Lua as a modding substrate (§2.4's OpenCiv3/MoonSharp is a second, independently engineered instance of the `mlua`-style embedded-interpreter pattern Ch205d surveys generically, though built on MoonSharp rather than `mlua`; Freeciv's `tolua`-bound `api_edit_*`/`api_server_*` surface, §3.4, is a third, hand-curated instance of the same pattern), and data-driven rulesets more broadly as the domain-specific, code-free counterpart to Ch205d's scripting-and-sandboxing survey. Where Ch205d asks what an extension is allowed to *reach* — the file descriptor, the GPU, another mod's private state — OXCE's declarative YAML merging (§4) sidesteps that question by never running arbitrary code in the first place. OpenCiv3 and Freeciv both reopen it, but at different points on the same axis: a mod script reaching live engine objects through OpenCiv3's `GAME_DATA()` is a trust decision with almost no boundary, while Freeciv's `signal.connect`-hooked Lua is scoped to whatever the curated `api_edit_*`/`api_server_*` functions choose to expose — the hand-curated-binding pattern Ch205d treats as the norm, against which OpenCiv3's reflection-based approach is the outlier.

**Chapter 205c — Open-Source 2D Simulation-Game Engines** is this chapter's companion "reimplementation" chapter, there scoped to rendering architecture for a related family of projects and here scoped to rules and data-model architecture. Read together, the two chapters cover the same genre of open-source strategy/simulation project from its two structural halves: how it draws itself, and what it is actually simulating underneath.

**Chapter 205b — AI Agents in Games** covers the classical game-AI techniques (navmeshes, behaviour trees, GOAP, utility systems) and their modern/LLM-driven successors that sit underneath opponent implementations like Freeciv's `ai/classic/classicai.c` and OpenCiv3's `C7Engine/AI/PlayerAI.cs` (§3.5) — this chapter deliberately stops at naming those AI modules as engine-native, non-data-driven code, and defers the AI architecture itself to Ch205b.

**Chapter 205f — Artificial Life on the GPU** is a sibling Part XI chapter covering a different class of simulation software — deliberately GPU-compute-focused where this chapter is deliberately non-graphics. The two chapters share little architectural overlap but sit side by side in the Part XI survey of simulation-oriented software; Ch205f links back to this chapter as its data-driven, rules-engine-first counterpart.

---

## References

- [GitHub — C7-Game/OpenCiv3](https://github.com/C7-Game/OpenCiv3) — MIT license, pre-alpha status, Godot+C#, `Development` default branch, top-level folder descriptions (§2)
- [OpenCiv3 `C7Engine/readme.md` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/readme.md) — UI/engine/data decoupling rationale, networking-exploration motivation (§2.1, §2.2)
- [OpenCiv3 `C7Engine/C7GameData/readme.md` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7GameData/readme.md) — Game data as "master copy of the game state" (§2.1)
- [OpenCiv3 `C7Engine/C7GameData/Save/SaveGame.cs` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7GameData/Save/SaveGame.cs) — `SaveGame.FromGameData`, `JsonSerializerOptions`, `TypeInfoResolver.IgnoreDefaultValues`, `using C7Engine`/`using C7Engine.Lua` (§2.1, §2.3)
- [OpenCiv3 `C7Engine/EntryPoints/MessageToEngine.cs` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/EntryPoints/MessageToEngine.cs) — `MessageToEngine` abstract base, `EngineStorage.pendingMessages` queue, `MsgSetFortification` example (§2.2)
- [OpenCiv3 `C7Engine/EntryPoints/MessageToUI.cs` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/EntryPoints/MessageToUI.cs) — `MessageToUI`, `AnimationMessage`, `EngineStorage.messagesToUI`/`animationMessages` queues (§2.2)
- [OpenCiv3 `C7Engine/C7Engine.csproj` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/C7Engine.csproj) — MoonSharp 2.0.0 dependency, `net8.0` target (§2.4)
- [OpenCiv3 `C7Engine/Lua/BehaviorEngine.cs` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/BehaviorEngine.cs) — `ImportFunc<T>`, `ResolveFunctionPath`, Lua-function-path-to-delegate conversion (§2.4)
- [OpenCiv3 `C7Engine/Lua/GameMode.cs` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/GameMode.cs) — `GameMode.Config`, `GameModeLoader`, addon pipeline, JSON/Lua ruleset fallback, `ScriptInitializer` reflection-based userdata registration (§2.4)
- [GitHub — freeciv/freeciv](https://github.com/freeciv/freeciv) — GPL license, `client`/`server`/`common`/`ai`/`data`/`lua` top-level layout (§3, §3.2)
- [Freeciv `doc/HACKING` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/doc/HACKING) — Client/server model, "the client is pretty dumb," history of server-side centralization (§3.1)
- [Freeciv commit count since 2025-08-01 (`master`)](https://api.github.com/repos/freeciv/freeciv/commits?since=2025-08-01T00:00:00Z&per_page=1) — 502 commits via `Link: rel="last"` page count (§3.3)
- [Freeciv `data/classic/techs.ruleset` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/data/classic/techs.ruleset) — In-file modding instructions, `[datafile]` section, `web-compatible` options token (§3.2)
- [Freeciv `data/` directory listing (master)](https://github.com/freeciv/freeciv/tree/master/data) — Per-ruleset subdirectories: classic, civ2civ3, civ1, civ2, multiplayer, sandbox, alien, goldkeep, granularity, hex2t, trident, default, stub (§3.2)
- [GitHub — freeciv/freeciv-web](https://github.com/freeciv/freeciv-web) — AGPLv3 license, redirects to `develop` as default branch (§3.3)
- [freeciv-web commit count since 2025-08-01 (`develop`)](https://api.github.com/repos/freeciv/freeciv-web/commits?sha=develop&since=2025-08-01T00:00:00Z&per_page=100) — zero commits returned (§3.3)
- [freeciv-web `README.md` (develop)](https://github.com/freeciv/freeciv-web/blob/develop/README.md) — Component breakdown: Freeciv-web (Java/Tomcat/nginx), Freeciv (patched C server), Freeciv-proxy (Python WebSocket proxy), Publite2 (Python process launcher) (§3.3)
- [freeciv-web `LICENSE.txt` (master)](https://github.com/freeciv/freeciv-web/blob/master/LICENSE.txt) — GPL for the C server vs. AGPLv3 for the web client (§3.3)
- [Freeciv `server/scripting/api_server_edit.h` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/server/scripting/api_server_edit.h) — Curated, `tolua`-bound `api_edit_*` C functions exposed to the embedded Lua state (§3.4)
- [Freeciv `data/classic/script.lua` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/data/classic/script.lua) — Ruleset-specific Lua layered over `default.lua`, `signal.connect("city_destroyed", ...)` example (§3.4)
- [Freeciv `ai/` directory listing (master)](https://github.com/freeciv/freeciv/tree/master/ai) — `difficulty.c`, `handicaps.c`, `aitraits.c`, and per-ruleset `classic/classicai.c` AI modules (§3.5)
- [OpenCiv3 `C7Engine/AI/` directory listing (Development)](https://github.com/C7-Game/OpenCiv3/tree/Development/C7Engine/AI) — `IAI.cs`, `PlayerAI.cs`, `BarbarianAI.cs`, `StrategicAI`, `UnitAI` (§3.5)
- [GitHub — OpenXcom/OpenXcom](https://github.com/OpenXcom/OpenXcom) — GPL-3.0 license, base repository (§4, §4.3)
- [OpenXcom commit count since 2025-08-01 (`master`)](https://api.github.com/repos/OpenXcom/OpenXcom/commits?since=2025-08-01T00:00:00Z&per_page=1) — 41 commits via `Link: rel="last"` page count (§4.3)
- [GitHub — MeridianOXC/OpenXcom](https://github.com/MeridianOXC/OpenXcom) — GPL-3.0 license, `oxce-plus` branch, README mod-portal reference (§4, §4.2, §4.3)
- [OXCE commit count since 2025-08-01 (`oxce-plus`)](https://api.github.com/repos/MeridianOXC/OpenXcom/commits?sha=oxce-plus&since=2025-08-01T00:00:00Z&per_page=1) — 175 commits via `Link: rel="last"` page count (§4.3)
- [OXCE `bin/standard/Smarter_Equip/metadata.yml` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/Smarter_Equip/metadata.yml) — `master: xcom1` dependency declaration (§4.1)
- [OXCE `bin/standard/Smarter_Equip/1_Smarter_Equip.rul` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/Smarter_Equip/1_Smarter_Equip.rul) — Per-item `defaultInventorySlot` overrides (§4.1)
- [OXCE `bin/standard/XCOM_Damage/XCOM_Damage.rul` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/bin/standard/XCOM_Damage/XCOM_Damage.rul) — Single-key `constants` override (§4.1)
- [OXCE `bin/standard/xcom1/` directory listing (oxce-plus)](https://github.com/MeridianOXC/OpenXcom/tree/oxce-plus/bin/standard/xcom1) — Base game ruleset in the same `.rul`/YAML format as third-party mods (§4.1)
- [openxcom.mod.io — XPiratez](https://openxcom.mod.io/xpiratez) and [X-Com Files](https://openxcom.mod.io/x-com-files) — Total-conversion mod listings (§4.2)
- [Thousand Parsec organization repositories](https://api.github.com/orgs/thousandparsec/repos) — `libtpproto-cpp`, `libtpproto-py`, `tpclient-pywx`, `tpadmin-cpp`, `daneel-ai` as independent client/server/AI/admin programs (§5)
- [GitHub — thousandparsec/tpserver-cpp](https://github.com/thousandparsec/tpserver-cpp) — GPL-2.0, last pushed 2011-05-03, not archived, no commits since (§5)
- [tpserver-cpp `modules/games/` directory (master)](https://github.com/thousandparsec/tpserver-cpp/tree/master/modules/games) — Compiled ruleset modules: minisec, mtsec, rfts, risk, tae (§5)
- [tpserver-cpp `modules/games/minisec/` directory (master)](https://github.com/thousandparsec/tpserver-cpp/tree/master/modules/games/minisec) — Full C++ implementation of one ruleset module (§5)
- [GitHub — freecol/freecol](https://github.com/freecol/freecol) — GPL-2.0 license text, `data/rules/` and `data/mods/` directory structure (§7)
- [GitHub — fifengine/fifengine](https://github.com/fifengine/fifengine) — LGPL-2.1, active commit history (§7)
- [GitHub — unknown-horizons organization](https://github.com/unknown-horizons) — `unknown-horizons` (FIFE-based) and `godot-port` repositories, both active and unarchived (§7)
- [GitHub — wesnoth/wesnoth](https://github.com/wesnoth/wesnoth) — GPL-2.0, most active repository surveyed (§7)
- [Wesnoth `data/campaigns/Heir_To_The_Throne/scenarios/01_The_Elves_Besieged.cfg` (master)](https://raw.githubusercontent.com/wesnoth/wesnoth/master/data/campaigns/Heir_To_The_Throne/scenarios/01_The_Elves_Besieged.cfg) — WML macro definition, `#enddef`, `[scenario]` tag (§7)
- [OpenCiv3 `C7/Lua/README.GameModes.md` (Development)](https://github.com/C7-Game/OpenCiv3/blob/Development/C7/Lua/README.GameModes.md) — Addon declaration-order composition, "code-change free mod loading mechanism" TODO (§8.1, §10.2)
- [OXCE `src/Engine/ModInfo.cpp` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Engine/ModInfo.cpp) — "masters can still have masters" chained-dependency comment (§8.2)
- [OXCE `src/Engine/FileMap.cpp` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Engine/FileMap.cpp) — `FileMap::setup()` master-chain expansion into load order (§8.2)
- [OXCE `src/Menu/ModListState.cpp` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Menu/ModListState.cpp) — `moveModUp`/`moveModDown` in-game mod reordering (§8.2)
- [OXCE `src/Mod/Mod.cpp` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Mod/Mod.cpp) — `Mod::loadRule` reuse-and-overwrite merge, `new`/`override`/`update`/`delete`/`ignore` keyword conflict check (§8.2)
- [Freeciv `doc/README.modpack_installer` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/doc/README.modpack_installer) — Modpack installer, install-time dependency resolution, third-party `.mpdl`/`.list` URLs (§8.3, §10.1)
- [tpserver-cpp `tpserver/main.cpp` (master)](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/tpserver/main.cpp) — Single `ruleset` config value read at server startup (§8.3)
- [tpserver-cpp `tpserver/pluginmanager.cpp` (master)](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/tpserver/pluginmanager.cpp) — `PluginManager::loadRuleset`, single `dlopen`ed ruleset module (§8.3)
- [Freeciv `server/savegame/savecompat.c` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/server/savegame/savecompat.c) — `sg_load_compat()`, versioned `compat[]` forward-migration table (§9.1)
- [Freeciv `server/ruleset/rscompat.c` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/server/ruleset/rscompat.c) — `rscompat_check_cap_and_version()`, ruleset format-version migration (§9.1)
- [Freeciv `server/savegame/savegame3.c` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/server/savegame/savegame3.c) — `sg_load_savefile()`, ruleset reload keyed off `savefile.rulesetdir` (§9.1)
- [OXCE `src/Savegame/SavedGame.cpp` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/Savegame/SavedGame.cpp) — `sanitizeModName`/`_isCurrentGameType` version-stripped mod-name check, silent per-entity load failures (§9.2)
- [OXCE `src/version.h` (oxce-plus)](https://raw.githubusercontent.com/MeridianOXC/OpenXcom/oxce-plus/src/version.h) — `MIN_REQUIRED_RULESET_VERSION_NUMBER` (mod-vs-engine gate, not save-file gate) (§9.2)
- [OpenCiv3 issue #434 — "Loading Incompatible Save Ruins New Game"](https://github.com/C7-Game/OpenCiv3/issues/434) — Observed incompatible-save failure mode (§9.3)
- [tpserver-cpp `modules/persistence/mysql/mysqlpersistence.cpp` (master)](https://raw.githubusercontent.com/thousandparsec/tpserver-cpp/master/modules/persistence/mysql/mysqlpersistence.cpp) — `tableversion` schema-version table, hard-refusal error log, single `gameinfo.turnname` migration exception (§9.4)
- [Freeciv forum — Rulesets and modpacks board](https://forum.freeciv.org/f/viewforum.php?f=11) — 329-topic community sharing/discussion board (§10.1)
- [OpenCiv3 issue #133 — "Prototype Mod"](https://github.com/C7-Game/OpenCiv3/issues/133) — Open since 2022-03-12; unresolved mod-packaging-format question (§10.2)
- [OpenCiv3 wiki](https://github.com/C7-Game/OpenCiv3/wiki) — Page listing with no mod-sharing/distribution page (§10.2)
- [thousandparsec/wiki `Clone_Ruleset.mediawiki` (master)](https://github.com/thousandparsec/wiki/blob/master/Clone_Ruleset.mediawiki) — Ruleset concepts discussed as in-tree future work, not external distribution (§10.3)
- [FreeCol `freecol-mods` repository, `mods/README.md`](https://github.com/FreeCol/freecol-mods/blob/master/mods/README.md) — Separate community-mod repository, pull-request submission workflow (§10.4)
- [Freeciv `server/settings.c` (master)](https://raw.githubusercontent.com/freeciv/freeciv/master/server/settings.c) — `timeout` server setting, up to 8,640,000-second maximum (§11.1)
- [Longturn documentation — turn change](https://longturn.readthedocs.io/en/latest/Playing/turn-change.html) — ~23-hour longturn timer convention (§11.1)
- [Longturn documentation — FAQ](https://longturn.readthedocs.io/en/latest/Playing/faq.html) — Concurrent move processing, turn-end conditions (§11.1)
- [freeciv-web `README.md` (develop)](https://github.com/freeciv/freeciv-web/blob/develop/README.md) — "Single player, Multiplayer, Longturn, Play-by-Email, Hotseat" mode list (§11.2) — also cited at §3.3
- [FreeColServer javadoc](https://www.freecol.org/javadoc/net/sf/freecol/server/FreeColServer.html) — No PBEM/longturn-style async mode found in server architecture (§11.3)
- [OXCE `src/` directory listing (oxce-plus)](https://github.com/MeridianOXC/OpenXcom/tree/oxce-plus/src) — No networking directory present, confirming single-process/no-server architecture (§11.3)
- [AOSA — "Thousand Parsec" chapter](https://aosabook.org/en/v1/thousandparsec.html) — `Finished Turn` request, simultaneous deadline-bounded turn resolution, written by project developers (§11.4)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
