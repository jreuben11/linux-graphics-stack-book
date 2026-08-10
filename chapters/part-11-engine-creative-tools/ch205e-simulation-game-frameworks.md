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
- [4. OpenXcom/OXCE: YAML Rulesets as the Entire Content Pipeline](#4-openxcomoxce-yaml-rulesets-as-the-entire-content-pipeline)
  - [4.1 The bin/standard/&lt;ModName&gt;/*.rul Convention](#41-the-binstandardmodnamerul-convention)
  - [4.2 Total-Conversion Mods and the Mod Portal](#42-total-conversion-mods-and-the-mod-portal)
  - [4.3 OXCE vs. Base OpenXcom](#43-oxce-vs-base-openxcom)
- [5. Thousand Parsec: Protocol-First Rulesets (An Instructive Dead End)](#5-thousand-parsec-protocol-first-rulesets-an-instructive-dead-end)
- [6. Data-Driven Rules Compared](#6-data-driven-rules-compared)
- [7. Brief Survey: FreeCol, FIFE/Unknown Horizons, Wesnoth WML](#7-brief-survey-freecol-fifeunknown-horizons-wesnoth-wml)
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

§6 returns to this axis directly. What varies across the five projects is not whether this pattern exists — it exists everywhere — but where the line falls between "data a mod can edit" and "code the engine's maintainers must write."

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

The reach given to that Lua is unusually broad. `ScriptInitializer`, run once at Lua-state initialization, reflects over `Assembly.GetExecutingAssembly().GetTypes()` filtered to `t.Namespace == "C7GameData"` and public visibility, and registers every one of them as MoonSharp `UserData` — plus every public enum in that namespace, flattened into a Lua `ENUMS` table with nested enums renamed `Outer_EnumName` (e.g. `Tile_YieldType`) to avoid collisions. It also installs a `GAME_DATA()` global function returning `EngineStorage.gameData` directly [Source](https://github.com/C7-Game/OpenCiv3/blob/Development/C7Engine/Lua/GameMode.cs). Because registration is driven by reflection over an entire namespace rather than a hand-written binding table, a mod script has access to essentially the full `C7GameData` object graph through `GAME_DATA()`, not a curated subset of it — a much wider modding surface than a purpose-built API like Freeciv's server commands or Luanti's `core.*` table (Ch205d, §3.2), at the cost of exposing internal engine state that a hand-curated binding could have kept private.

---

## 3. Freeciv: Network-Authoritative Client/Server and Text Rulesets

Freeciv (`github.com/freeciv/freeciv`, GPL, C) is the oldest project in this chapter and the one whose client/server split is the most literal. Its repository root splits cleanly into `client/`, `server/`, `common/`, `ai/`, `data/`, and `lua/` as top-level directories [Source](https://github.com/freeciv/freeciv).

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

| | Format | Moddable without recompiling? | Ceiling before engine code changes | Maintenance/tooling burden |
|---|---|---|---|---|
| **Freeciv** `.ruleset` | INI-style text, per-directory | Yes — copy a ruleset directory, edit values, select via `rulesetdir` | High but bounded: any field the format doesn't expose (a new combat formula, a new UI concept) needs C server changes | Low to author; the format is stable and self-documenting via in-file comments |
| **OXCE** `.rul` | YAML, per-directory with `master` dependency declarations | Yes — the base game and every mod use the identical format and merge mechanism | Similarly bounded by what keys the loader recognizes, but the surface is large enough that total conversions (XPiratez, X-Com Files) replace nearly all content without touching engine code | Low to author individual mods; YAML key/merge semantics scale to total conversions without new tooling |
| **OpenCiv3** embedded Lua | Lua scripts (`ruleset.lua`, `behaviors.lua`, `textures.lua`) compiled to a JSON `SaveGame` at load time | Yes, and further: mods are executable code with reflection-derived access to the live `C7GameData` object graph via `GAME_DATA()`, not just declarative values | Very high — a mod can implement arbitrary logic MoonSharp can express, bounded only by what `ScriptInitializer` exposes as userdata | Higher: authoring requires understanding both Lua and the reflected C# type surface; the addon-pipeline model (§2.4) also makes load order across mods significant |
| **Thousand Parsec** protocol modules | Compiled C++ (`modules/games/*`), selected by the server at startup, speaking a network protocol to independent client/AI programs | No — a new ruleset is a new compiled module, not a data file | None in principle — a module can implement anything the server process can, since it *is* server code | Highest: requires a C++ toolchain and the server's internal module API; the project did not survive to demonstrate this at scale |

Reading down the "moddable without recompiling" column against the "maintenance burden" column shows the trade-off directly: the two projects offering pure data-file modding (Freeciv, OXCE) are also the two still under active development, while the project offering the most powerful modding surface (Thousand Parsec, compiled swappable rulesets) is the one that stopped. OpenCiv3 sits in an interesting middle position — Lua is data in the sense that it loads without recompiling the host, but it is code in the sense that a mod author needs real programming skill and can touch far more of the live engine than a YAML key can, which is a genuinely different risk/power trade-off from either INI text or YAML merging, addressed more generally for engine modding across languages in Ch205d.

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

## Integrations

**Chapter 205d — Modding Architectures** covers the general engineering problem this chapter's frameworks each answer once, for one domain: embedded Lua as a modding substrate (§2.4's OpenCiv3/MoonSharp is a second, independently engineered instance of the `mlua`-style embedded-interpreter pattern Ch205d surveys generically, though built on MoonSharp rather than `mlua`), and data-driven rulesets more broadly as the domain-specific, code-free counterpart to Ch205d's scripting-and-sandboxing survey. Where Ch205d asks what an extension is allowed to *reach* — the file descriptor, the GPU, another mod's private state — this chapter's rulesets mostly sidestep that question by never running arbitrary code in the first place; OpenCiv3's Lua layer is the one framework here that reopens it, since a mod script reaching live engine objects through `GAME_DATA()` is a trust decision in exactly Ch205d's sense.

**Chapter 205c — Open-Source 2D Simulation-Game Engines** is this chapter's companion "reimplementation" chapter, there scoped to rendering architecture for a related family of projects and here scoped to rules and data-model architecture. Read together, the two chapters cover the same genre of open-source strategy/simulation project from its two structural halves: how it draws itself, and what it is actually simulating underneath.

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

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
