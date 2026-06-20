# Chapter 171: Linux Gaming Anti-Cheat — EasyAntiCheat, BattlEye, and the Ring-0 Problem

**Part VIII — Gaming Layer**

**Audiences:** Linux gaming developers and Proton users who need to understand why some games refuse to start; systems engineers working at the Wine/kernel boundary who need a precise model of what is and is not possible; game developers evaluating whether to enable Linux support for their existing EAC- or BattlEye-protected title.

---

## Table of Contents

1. [Introduction: The Anti-Cheat Ecosystem on Linux](#1-introduction-the-anti-cheat-ecosystem-on-linux)
2. [Why Ring-0 Anti-Cheat Cannot Work on Linux](#2-why-ring-0-anti-cheat-cannot-work-on-linux)
3. [EasyAntiCheat on Linux](#3-easyanticheat-on-linux)
4. [BattlEye on Linux](#4-battleye-on-linux)
5. [Steam Linux Runtime and Anti-Cheat Containers](#5-steam-linux-runtime-and-anti-cheat-containers)
6. [Anti-Cheat Systems That Block Linux](#6-anti-cheat-systems-that-block-linux)
7. [The Remaining Gap: Kernel-Level Integrity](#7-the-remaining-gap-kernel-level-integrity)
8. [Game Developer Guidance: Enabling Linux Anti-Cheat](#8-game-developer-guidance-enabling-linux-anti-cheat)
9. [Integrations](#9-integrations)
10. [References](#10-references)

---

## 1. Introduction: The Anti-Cheat Ecosystem on Linux

Anti-cheat software is one of the most consequential differentiators between a game being playable on Linux and being effectively blocked. It is not a matter of graphics API support or driver quality — Vulkan translation through DXVK and VKD3D-Proton is mature enough to run the vast majority of Windows titles at competitive performance. What blocks Linux players from many of the most popular online multiplayer titles is a single architectural decision made by their anti-cheat vendor: whether to ship a kernel-mode driver that runs only on Windows, or to provide a user-space or native Linux implementation.

The major anti-cheat systems in the PC gaming ecosystem in 2026 are:

- **Easy Anti-Cheat (EAC)** — developed by Kamu, acquired by Epic Games in 2018, now distributed as part of the **Epic Online Services (EOS)** SDK. Used by Fortnite, Dead by Daylight, Halo: The Master Chief Collection, and hundreds of other titles.
- **BattlEye** — an independent vendor, long-standing partner of Bohemia Interactive (DayZ, Arma). Used by PUBG: Battlegrounds, DayZ, Arma 3, Rainbow Six Siege, and others.
- **Riot Vanguard** — proprietary to Riot Games; used exclusively in Valorant and League of Legends.
- **XIGNCODE3** — from Wellbia; common in Korean MMOs and Asian-published online games.
- **nProtect GameGuard** — from INCA Internet; historically very common in Korean online games; used in Helldivers 2 on PC.
- **Valve Anti-Cheat (VAC)** — Valve's own system, used in Counter-Strike 2, Team Fortress 2, Dota 2, and others.

Of these, **EAC** and **BattlEye** have provided working Linux support since late 2021, making the vast majority of titles that use them technically runnable on Linux if their developers opt in. **Vanguard**, **XIGNCODE3**, and **nProtect GameGuard** rely on Windows kernel-mode drivers that are architecturally incompatible with Wine/Proton. **VAC** is server-side and works on Linux without any special handling.

The story of anti-cheat on Linux is therefore not a single technical narrative but a spectrum: at one end, systems that have solved the problem; at the other, systems that are structurally incapable of porting; and in the middle, a large number of games that have capable anti-cheat but whose publishers have not taken the few configuration steps required to enable Linux support.

---

## 2. Why Ring-0 Anti-Cheat Cannot Work on Linux

To understand why some anti-cheat systems are incompatible with Linux and Wine, it is necessary to understand what kernel-mode anti-cheat software actually does on Windows.

### 2.1 The Windows Kernel-Mode Driver Model

On Windows, an anti-cheat system such as Vanguard or the full EAC Windows driver registers itself as a **Kernel-Mode Driver Framework (KMDF)** driver (`WDFDriver`). This driver runs at **privilege ring 0** — the same privilege level as the Windows kernel itself — giving it unrestricted access to every byte of physical memory, every CPU register, and every hardware interface. On 64-bit Windows 10 and 11, any kernel driver must be digitally signed with an **Extended Validation (EV)** certificate enrolled through Microsoft's Hardware Developer Center. PatchGuard (Kernel Patch Protection) additionally enforces the integrity of core kernel data structures at runtime. The combination means that a Windows kernel-mode driver, once loaded, operates in a verified, signed environment with a known-good starting state.

Kernel-mode anti-cheat software uses this position to:

1. **Block cross-process memory reads.** By registering `ObRegisterCallbacks`, the driver intercepts every call to `NtOpenProcess` and strips `PROCESS_VM_READ`, `PROCESS_VM_WRITE`, `PROCESS_VM_OPERATION`, and `PROCESS_DUP_HANDLE` from the requested access mask before the handle is granted. Cheat software that uses `ReadProcessMemory` or `WriteProcessMemory` obtains a handle without the necessary rights. [Source](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/)

2. **Enumerate loaded kernel modules.** By walking kernel data structures, the driver detects unsigned or malicious kernel modules that a cheat might inject.

3. **Monitor system calls.** Via SSDT hooks or `PsSetLoadImageNotifyRoutine`, the driver observes DLL loads and injections into the game process.

4. **Attest boot-chain integrity.** With UEFI Secure Boot and TPM 2.0 integration, modern anti-cheat systems can ask the TPM for a **measured boot log** covering every component from firmware through the OS kernel, providing cryptographic proof that the system has not been tampered with before the game launched. [Source](https://andrewmoore.ca/blog/post/anticheat-secure-boot-tpm/)

### 2.2 Why None of This Can Run on Linux via Wine

Wine is a **user-space compatibility layer**. It implements the Windows API surface (Win32, NT native API, COM) as Linux ELF code making Linux system calls. Critically, Wine does not implement a Windows kernel, and it does not run any code in ring 0. The architectural gap has several concrete dimensions:

**No Windows kernel to host a driver.** A KMDF driver calls `WdfDriverCreate`, communicates via IOCTLs (`DeviceIoControl`), and depends on `ntoskrnl.exe` exports. Wine implements a subset of the Win32 API but has no `ntoskrnl.exe`, no kernel object manager at ring 0, and no driver verifier. A `.sys` file loaded into Wine would immediately fault on its first call into kernel infrastructure that simply does not exist. [Source](https://sam4k.com/whats-the-deal-with-anti-cheat-on-linux/)

**Linux's deliberately open debugging interfaces.** On Linux, any process that has the appropriate capabilities can attach to another process via `ptrace(2)`, read its memory via `/proc/PID/mem`, and inspect its state via `/proc/PID/maps`. This is by design — Linux's philosophy is that a process owner has full control over their processes, and the ability to debug and introspect programs is considered a feature, not a vulnerability. Windows's **protected process** model (`PPL`/`PP`) is the inverse: the OS enforces that only trusted, signed processes can open handles to protected processes. Wine has no mechanism to replicate PPL semantics because it would require ring-0 code on the Linux kernel side. [Source](https://tulach.cc/the-issue-of-anti-cheat-on-linux/)

**No enforcement authority for kernel module signing.** Linux does support kernel module signing (`CONFIG_MODULE_SIG`, `CONFIG_MODULE_SIG_FORCE`), and the Integrity Measurement Architecture (IMA) can extend TPM PCRs with measurements of loaded modules. [Source](https://lwn.net/Articles/137306/) However, on a typical consumer Linux desktop — and on SteamOS — these features are not enforced: users can freely load unsigned modules, recompile the kernel, or use `LD_PRELOAD` to intercept any library call. An anti-cheat system that deploys a kernel module on Linux would be fighting against a kernel that any sufficiently motivated user can recompile with the anti-cheat's detection hooks removed. Riot Games stated precisely this concern: "Linux does not currently afford us sufficient ability to attest boot state or kernel modules" and "the difficulty in securing it is only compounded by all the frustrating differences between distributions." [Source](https://www.gamingonlinux.com/2024/04/riot-games-talk-vanguard-anti-cheat-for-league-of-legends-and-why-its-a-no-for-linux/)

**Valve's explicit position.** Valve has stated unequivocally that Proton will not support kernel-module anti-cheat software. No kernel-mode anti-cheat running in a Linux kernel module is feasible without requiring every Linux gaming user to run a proprietary, closed-source module as root — a security commitment that the Linux gaming community and major distributors have consistently refused to make. [Source](https://www.phoronix.com/news/Easy-Anti-Cheat-Linux/)

### 2.3 The Scope of What Remains Possible

The consequence of the above is that Linux anti-cheat must operate entirely in **user space**, either as a native Linux library or as a Windows DLL running under Wine's PE loader. This eliminates the most powerful classes of kernel-level detection: cross-process handle stripping, kernel module enumeration, and cryptographic boot attestation. What remains is user-space memory scanning of the game process itself (which the anti-cheat library shares an address space with), network-level heuristics enforced server-side, and behavioral analysis. Whether these measures are sufficient to deter cheating is a game design and security question; what is technically certain is that this is the only architecture possible on Linux without a kernel module.

---

## 3. EasyAntiCheat on Linux

### 3.1 The September 2021 Announcement

On September 23, 2021, Epic Games announced native Linux and macOS support for Easy Anti-Cheat as part of its **Epic Online Services (EOS)** SDK. The announcement stated that "developers can activate anti-cheat support for Linux via Wine or Proton with just a few clicks in the Epic Online Services Developer Portal." [Source](https://onlineservices.epicgames.com/news/epic-online-services-launches-anti-cheat-support-for-linux-mac-and-steam-deck)

This was motivated directly by the impending launch of the **Steam Deck**, which runs SteamOS (a Linux-based OS) and uses Proton for Windows game compatibility. Valve had been in dialogue with anti-cheat vendors to make this transition smooth.

### 3.2 Two Generations: Kamu (v1) and EOS (v2)

There are two distinct generations of EAC integration, and they require different enablement procedures. Confusing them explains many forum discussions about broken EAC support:

**EAC v1 (Kamu/standalone):** The older integration uses a standalone setup executable (`EasyAntiCheat_Setup.exe`) that installs `EasyAntiCheat.dll` directly into the game folder. This was the dominant architecture before Epic's acquisition. The Steamworks documentation still covers the Kamu enablement path for games that have not migrated. [Source](https://partner.steamgames.com/doc/steamhardware/proton)

**EAC v2 (EOS SDK):** The newer integration ships as `EasyAntiCheat_EOS_Setup.exe` bundled with the EOS SDK. Games using this integration dynamically load EAC through the EOS anti-cheat interfaces. Apex Legends' November 2024 breakage on Linux stemmed from EA migrating from the v1 standalone EAC to the EOS version while simultaneously changing their EAC configuration to exclude Linux. [Source](https://www.gamingonlinux.com/2024/05/apex-legends-upheaval-update-live-with-eos-anti-cheat-breaking-it-on-steam-deck-linux/)

### 3.3 Architecture: The Native Linux Library

EAC's Linux support is built around a **native ELF shared library**, `easyanticheat_x64.so`. This file is extracted from the EAC SDK at `\Client\Assets\Plugins\x86_64\libeasyanticheat.so` and renamed to `easyanticheat_x64.so` before being shipped in the game's Steam depot alongside the Windows DLL `EasyAntiCheat_x64.dll`. [Source](https://partner.steamgames.com/doc/steamhardware/proton)

When Proton launches a game that includes `easyanticheat_x64.so`, it uses the **Proton EasyAntiCheat Runtime** (a separate Steam tool, discussed in §5) to bridge the game's PE-side EAC invocation to the native ELF library. The precise mechanism by which Wine's PE loader delegates to the native `.so` is not publicly documented by Epic or Valve. **Note: needs verification.** What is publicly confirmed is the file naming convention and the fact that no source code changes are required to the game executable — the enablement is purely a matter of including the `.so` file and enabling the Linux platform flag in the EAC portal.

Running as a native Linux process, `easyanticheat_x64.so` has access to the same process memory as the game itself (they share the same address space under Wine's process model). It can therefore scan for known cheat signatures in the game's memory regions, validate the integrity of the game's executable pages, and report results to EAC's server-side backend. What it **cannot** do is anything requiring a kernel-mode driver: it cannot intercept handle creation in other processes, enumerate all kernel modules, or perform boot attestation. [Source](https://tulach.cc/the-issue-of-anti-cheat-on-linux/)

### 3.4 The Developer Opt-In Process (EOS Version)

Enabling EAC Linux support for an EOS-integrated game requires changes in the Epic Online Services Developer Portal:

```text
1. Log into the EOS Developer Portal at dev.epicgames.com.
2. Navigate to the anti-cheat configuration for your product.
3. Enable "Linux" as a supported client platform.
4. Download the latest EAC SDK.
5. Locate the Linux library in the SDK:
     SDK/Client/Assets/Plugins/x86_64/libeasyanticheat.so
6. Rename it to: easyanticheat_x64.so
7. Place it in your game depot alongside:
     EasyAntiCheat_EOS_Setup.exe
     EasyAntiCheat_x64.dll
8. Publish a new build on Steamworks containing the updated depot.
   No changes to the game source code are required.
```

For the older Kamu/v1 standalone integration, the steps go through the EAC partner site rather than the EOS portal:

```text
1. In SDK Configuration settings: enable "Linux" as a client platform.
2. In Client Module Releases: select the Unix platform and activate a module.
3. Extract libeasyanticheat.so from the SDK, rename to easyanticheat_x64.so,
   and place it next to EasyAntiCheat_x64.dll in the depot.
4. Publish a new Steamworks build.
```
[Source](https://partner.steamgames.com/doc/steamhardware/proton)

### 3.5 Current Game Compatibility

The [Are We Anti-Cheat Yet?](https://areweanticheatyet.com/) community database tracks Linux anti-cheat compatibility across over 1,100 games. As of mid-2026, of the games using EAC, notable titles with working Linux support include:

- **Halo: The Master Chief Collection** — EAC, fully supported [Source](https://areweanticheatyet.com/)
- **Halo Infinite** — EAC + Arbiter, fully supported
- **Dead by Daylight** — EAC, fully supported
- **Dune: Awakening** — BattlEye (see §4), Linux support enabled at launch in May 2025 [Source](https://www.gamingonlinux.com/2025/05/dune-awakening-will-have-battleye-enabled-for-linux-steamos-steam-deck/)

Notable titles where EAC is present but Linux support has been **withdrawn** or **never enabled**:

- **Apex Legends** — EA disabled Linux/Steam Deck support in November 2024 following internal analysis identifying Linux as a vector for hard-to-detect cheats, citing the "openness of Linux operating systems" making cheat development easier. The blocking followed a shift from the Kamu EAC to the EOS version. EA reported a 30% drop in cheat incidents after the change. [Source](https://forums.ea.com/blog/apex-legends-game-info-hub-en/dev-team-update-linux--anti-cheat/9468053)
- **Rust** — Facepunch uses EAC but has stated no plans to enable Linux support, citing engine-level concerns with Unity.
- **Fortnite** — EAC, but Linux support is actively blocked by Epic Games itself; the title appears in the "Denied" category of the Are We Anti-Cheat Yet? database, meaning Epic has taken a positive decision to exclude Linux, not merely deferred a decision. [Source](https://areweanticheatyet.com/)

The pattern across these cases is significant: the technical infrastructure for EAC Linux support is present in all of these titles, but the publisher's decision to opt in (or not) is entirely independent of that technical readiness.

### 3.6 The Apex Legends Case Study

The Apex Legends situation deserves its own treatment because it illustrates the policy dimension of Linux anti-cheat more clearly than any purely technical analysis can. For a period between 2021 and 2024, Apex Legends worked on Linux through Proton using the v1 Kamu EAC integration, with the Linux platform flag enabled. EA then upgraded to the EOS-integrated version of EAC in the "Upheaval" update in May 2024, which broke Linux compatibility as a side effect of the migration — the EOS version requires an explicit re-enablement of the Linux platform flag. At this point, rather than re-enable Linux support, EA's anti-cheat team conducted an internal investigation.

EA's stated conclusion, published in the Dev Team Update in November 2024, was that Linux OS represented "a path for a variety of impactful exploits and cheats" and that "Linux cheats are harder to detect with data showing they are growing at a rate requiring outsized focus for a relatively small platform." [Source](https://forums.ea.com/blog/apex-legends-game-info-hub-en/dev-team-update-linux--anti-cheat/9468053) The implicit reasoning: because `easyanticheat_x64.so` on Linux cannot perform the same kernel-level isolation and detection that EAC's Windows driver can, Linux effectively operates at a lower detection sensitivity threshold. A player running a cheat on Linux that doesn't trigger EAC's user-space checks will not be caught.

This is the technical argument against EAC's Linux user-space mode at its most concrete: it is not that the Linux EAC library does nothing, but that what it can do is a subset of what the Windows kernel driver does, and that subset may be insufficient for a highly competitive title where the financial and reputational costs of a cheat-infested player base are high.

The EA decision generated significant community pushback, with critics noting that blocking Linux affects Steam Deck owners who purchased the game in good faith. Valve issued refunds to some affected users. The incident set a precedent that is likely to influence other publishers operating highly competitive free-to-play titles.

---

## 4. BattlEye on Linux

### 4.1 The September 2021 Announcement

BattlEye announced Proton and Steam Deck support on September 24, 2021, the day after Epic's EAC announcement, via a post on X (formerly Twitter): "BattlEye has provided native Linux and Mac support for a long time and we can announce that we will also support the upcoming Steam Deck (Proton). This will be done on an opt-in basis with game developers choosing whether they want to allow it or not." [Source](https://x.com/TheBattlEye/status/1441477816311291906)

This confirmed that BattlEye had carried native Linux support for some time — particularly used by dedicated Linux game servers — and was now extending that to client-side Proton play.

### 4.2 Architecture: Native Library via the BattlEye Runtime

Like EAC, BattlEye's Proton support is built around a native ELF library (`BattlEye.so` or the platform equivalent) shipped alongside the Windows DLL. The specific internal filenames are not publicly specified in developer-facing documentation. **Note: needs verification.** The runtime is delivered as a separate Steam tool: the **Proton BattlEye Runtime**, which must be installed and pointed to by the `PROTON_BATTLEYE_RUNTIME` environment variable. [Source](https://www.gamingonlinux.com/2021/11/supporting-linux-proton-and-the-steam-deck-with-battleye-is-just-an-email-away/)

### 4.3 Developer Enablement: An Email

BattlEye's enablement process is strikingly simple compared to EAC's portal workflow. Valve confirmed that as of late 2021, a developer needs only to **contact their BattlEye technical representative** and request that Linux/Proton support be activated for their title. No SDK changes, no depot modifications, no source code changes — BattlEye flips a configuration flag on its side. [Source](https://store.steampowered.com/news/group/4145017/view/3104663180636096966) This makes BattlEye arguably the easiest anti-cheat to enable on Linux.

### 4.4 Current Game Compatibility

The GamingOnLinux BattlEye compatibility tracker documents games that have and have not enabled Linux support. As of mid-2026 the working list includes approximately 18 titles:

- **DayZ**, **Arma 3**, **ARK: Survival Evolved / Ascended**, **Unturned**, **PlanetSide 2**, **Enlisted**, **Albion Online**, **Conan Exiles**, **War Thunder**, **Mount & Blade II: Bannerlord**, **Crossout**, **Skull and Bones**, **Riders Republic**, **Dune: Awakening**

[Source](https://www.gamingonlinux.com/anticheat/vendor/battleye/)

Broken or not-enabled titles include approximately 11 tracked by the same database:

- **Destiny 2** — BattlEye but no Linux support enabled
- **PUBG: Battlegrounds** — BattlEye but no Linux support enabled
- **Tom Clancy's Rainbow Six Siege X** — BattlEye but no Linux support enabled
- **Escape from Tarkov** — BattlEye but no Linux support enabled
- **Grand Theft Auto V / GTA Online** — a particularly illustrative case (see below)

**The GTA V case.** In September 2024, Rockstar added BattlEye anti-cheat to GTA Online's PC build. This broke online play on Linux and Steam Deck. The key detail: BattlEye itself fully supports Linux and the enablement would require nothing more than contacting BattlEye, yet Rockstar chose not to enable it. GTA V single-player remains functional on Linux, while GTA Online is blocked. Valve responded by issuing refunds to some Steam Deck users. [Source](https://www.gamingonlinux.com/2024/09/grand-theft-auto-v-gets-battleye-anti-cheat-breaks-online-play-on-steam-deck-linux/) This case is the clearest example of the "capable but not enabled" category — a publisher leaving Linux support on the table when the technical lift is trivially small.

---

## 5. Steam Linux Runtime and Anti-Cheat Containers

### 5.1 The Steam Linux Runtime Container Architecture

Proton runs games inside a **pressure-vessel** container that mounts a Valve-curated rootfs, isolating the game from host library differences. There are three runtime container generations, distinguished by their Ubuntu base:

- **scout** (Steam Runtime 1): Ubuntu 12.04 base; legacy; used by older Proton versions.
- **soldier** (Steam Runtime 2): Ubuntu 18.04 base; intermediate.
- **sniper** (Steam Runtime 3): Ubuntu 22.04 base; current default for Proton 8+ and Proton Experimental.

The anti-cheat runtimes are installed as separate Steam tools and delivered alongside these containers.

### 5.2 The Proton EasyAntiCheat Runtime and PROTON_EAC_RUNTIME

The **Proton EasyAntiCheat Runtime** is a distinct item in the Steam library (separate from Proton itself), installed to:

```
~/.local/share/Steam/steamapps/common/Proton EasyAntiCheat Runtime/
```

When Proton detects that a game includes `easyanticheat_x64.so`, it uses the path specified in `PROTON_EAC_RUNTIME` to locate this runtime. The environment variable can be set as a Steam launch option to override the default location:

```bash
PROTON_EAC_RUNTIME="/home/$USER/.local/share/Steam/steamapps/common/Proton EasyAntiCheat Runtime" %command%
```

[Source](https://pulsegeek.com/articles/enable-proton-eac-runtime-on-steam-deck-steps/)

Steam automatically sets `PROTON_EAC_RUNTIME` for games whose Steamworks configuration marks them as requiring EAC, as long as the runtime tool is installed. Manual setting is only needed when the automatic detection fails or for non-Steam Proton environments.

### 5.3 The Proton BattlEye Runtime and PROTON_BATTLEYE_RUNTIME

The **Proton BattlEye Runtime** (a separate Steam tool) installs to:

```
~/.local/share/Steam/steamapps/common/Proton BattlEye Runtime/
```

The `PROTON_BATTLEYE_RUNTIME` environment variable points to this directory. The Dune: Awakening launch instructions explicitly note: "To access multiplayer areas, you must install 'Proton BattlEye Runtime'." [Source](https://www.gamingonlinux.com/2025/05/dune-awakening-will-have-battleye-enabled-for-linux-steamos-steam-deck/) As with EAC, Steam handles this automatically for Steamworks-registered BattlEye games.

```bash
# Manual override if auto-detection fails:
PROTON_BATTLEYE_RUNTIME="/home/$USER/.local/share/Steam/steamapps/common/Proton BattlEye Runtime" %command%
```

### 5.4 Proton Version and Anti-Cheat Compatibility

Anti-cheat library behaviour can be sensitive to the specific Proton version. The EOS version of EAC introduced a **bootstrapper version 1.6.0** that broke EAC loading under Proton 8.x in certain combinations. [Source](https://github.com/ValveSoftware/Proton/issues/6740) Games using the Kamu/v1 EAC integration (where `PROTON_EAC_RUNTIME` points to the v1 runtime variant) are affected differently from EOS-integrated games. When a game's EAC stops working after a Proton update, the first diagnostic step is to check the anti-cheat log file (§8.4) and try switching to **Proton Experimental**, which typically carries the most recent anti-cheat compatibility fixes.

---

## 6. Anti-Cheat Systems That Block Linux

### 6.1 Riot Vanguard

Riot Vanguard is the kernel-mode anti-cheat used in **Valorant** (since launch in 2020) and **League of Legends** (since January 2024, when Riot replaced its previous client-side checks with Vanguard). Vanguard consists of two components: a userland client process and a kernel-mode driver (`vgk.sys`) that starts at system boot, before the game launches.

Riot's technical rationale for why Vanguard cannot work on Linux, stated in April 2024, is precise:

- **Boot-state attestation:** Vanguard requires the ability to attest that the system's boot chain — firmware, OS loader, kernel — has not been tampered with. On Linux, this requires Secure Boot enforcement and UEFI Secure Boot key management that is not universally configured on consumer systems.
- **Kernel module verification:** Linux's kernel module signing is not enforced by default; an adversary can boot with a custom kernel and hide cheat software from any userland scanner.
- **Distribution fragmentation:** The wide variety of Linux kernel versions and configurations makes it impractical to ship a Vanguard kernel driver that handles all variants safely. [Source](https://www.gamingonlinux.com/2024/04/riot-games-talk-vanguard-anti-cheat-for-league-of-legends-and-why-its-a-no-for-linux/)

The practical impact: League of Legends, previously playable on Linux via Wine/Lutris (using an unofficial Lutris script), became unplayable after Vanguard was enabled. Valorant has never worked on Linux through Wine. The virtual machine bypass Riot is concerned about (running the game in a VM while the cheat runs on the host) is genuine: Vanguard inside a VM cannot see the host kernel modules that might contain the cheat.

### 6.2 XIGNCODE3 and nProtect GameGuard

**XIGNCODE3** (Wellbia) and **nProtect GameGuard** (INCA Internet) are kernel-mode anti-cheat systems common in Korean MMOs and Asian-published online games. Both ship Windows kernel drivers that function as rootkits, intercepting system calls and monitoring process memory from ring 0. Wine's lack of a Windows kernel means these drivers cannot be loaded — as the WineHQ forums noted years ago: "Wine will never support nProtect GameGuard because it's a kernel-level rootkit, and Wine does not have a fully featured kernel." [Source](https://forum.winehq.org/viewtopic.php?t=1077)

**Helldivers 2** (Sony Interactive Entertainment / Arrowhead Game Studios) added nProtect GameGuard on PC. This initially broke Steam Deck and Linux compatibility. Valve released a Proton Hotfix in February 2024 to partially address the launch sequence, but the GameGuard integration on Linux remains problematic and the compatibility situation has fluctuated across game updates. **Note: needs verification** — the precise current status as of mid-2026 should be confirmed against ProtonDB reports for App ID 553850, as it changes with game patches.

The fundamental technical limitation for all of these systems is identical to Vanguard's: they require a ring-0 driver communicating with `ntoskrnl.exe` infrastructure that does not exist in Wine's process model.

### 6.3 Valve Anti-Cheat (VAC) — The Working Counter-Example

**Valve Anti-Cheat (VAC)** is the anti-cheat system in Counter-Strike 2, Team Fortress 2, Dota 2, and dozens of other Valve and third-party titles on Steam. VAC is a user-space, non-kernel system — it runs as a module loaded into the game process and performs memory scanning and signature matching at the user-space privilege level. [Source](https://developer.valvesoftware.com/wiki/Valve_Anti-Cheat) The detection backend (`VACnet`) is a server-side machine learning system that analyzes gameplay telemetry for behavioural anomalies.

Because VAC has no kernel component, it works on Linux without modification. Counter-Strike 2 is fully VAC-enabled on Linux (where it runs natively, not through Proton), and TF2 and Dota 2 run without anti-cheat issues under either native or Proton builds.

VAC's architecture represents one viable path forward for Linux gaming: move the adversarial-detection logic server-side or into user-space, where it is platform-independent, rather than building it into a kernel driver. The trade-off is that VAC cannot block kernel-mode cheats (cheats that run as kernel modules themselves), which is why Valve has supplemented VAC with the VACnet ML backend for its most competitive titles.

### 6.4 EA Anticheat / RICOCHET and the Kernel Trend

Beyond the systems above, EA has developed **EA Anticheat** (codenamed "Javelin") and Activision uses **RICOCHET** for Call of Duty titles. Both are kernel-level on Windows, with RICOCHET requiring TPM 2.0 and Secure Boot since Call of Duty: Black Ops 7. [Source](https://support.activision.com/articles/trusted-platform-module-and-secure-boot) Neither system currently provides Linux support.

The industry trend toward TPM/Secure Boot attestation as a baseline represents a new technical barrier: even if a kernel driver could be made Linux-compatible, the attestation chain would need to run through Linux's TPM2 stack and IMA framework, requiring specific kernel configurations not present on typical consumer installs.

---

## 7. The Remaining Gap: Kernel-Level Integrity

### 7.1 What Kernel-Mode Anti-Cheat Checks That User-Space Cannot

Even the best user-space anti-cheat running as a native Linux library faces a fundamental limitation: it is a peer of the process it is protecting, not a superior. A sophisticated cheat operating from a Linux kernel module or a hypervisor layer (running the game inside a VM) is invisible to any user-space scanner. Specifically, user-space anti-cheat cannot:

- **Intercept cross-process handle creation.** Without `ObRegisterCallbacks` — a Windows kernel API — there is no equivalent Linux mechanism for a user-space library to prevent another process from reading game memory via `process_vm_readv(2)` or `/proc/PID/mem`. Linux does allow `PR_SET_DUMPABLE` to restrict `/proc/PID/mem` access, but this is a per-process flag that the game can set for itself, not a system-wide enforcement mechanism. [Source](https://tulach.cc/the-issue-of-anti-cheat-on-linux/)

- **Enumerate all kernel modules.** A user-space library can read `/proc/modules` and `/sys/module/`, but a kernel-level cheat can hide itself from these by modifying the in-kernel module list (a technique equivalent to a Windows rootkit modifying `PsLoadedModuleList`).

- **Verify boot chain integrity cryptographically.** Without a TPM endorsement that the anti-cheat software trusts, there is no way to confirm that the kernel hasn't been patched.

### 7.2 Linux IMA and Hypothetical Kernel Module Anti-Cheat

Linux does have integrity infrastructure that could theoretically underpin a kernel-level anti-cheat:

- **IMA (Integrity Measurement Architecture)** extends TPM PCRs with hashes of every executed file and loaded kernel module, providing a measurement log that a remote attestation server can verify. [Source](https://lwn.net/Articles/137306/)
- **`CONFIG_MODULE_SIG_FORCE`** can require that all kernel modules be signed with a key embedded in the kernel build.
- **UEFI Secure Boot** with a distribution's signing key can prevent unsigned bootloaders and kernels.

In principle, a game publisher could require that players boot with a kernel built with `CONFIG_MODULE_SIG_FORCE`, IMA enabled in enforcing mode, and Secure Boot active — and then verify this state via TPM attestation before allowing game access. This would provide a Linux equivalent of the Windows measured-boot chain that Vanguard and RICOCHET rely on.

In practice, this would require players to use a locked-down distribution kernel with no ability to load third-party modules (including DKMS-built GPU drivers or network drivers), which is incompatible with the needs of the vast majority of Linux users. It would also require game publishers to maintain an attestation server that tracks trusted kernel builds across every major Linux distribution and kernel version — a significant ongoing engineering commitment. No publisher has pursued this approach as of mid-2026.

The open-source nature of the Linux kernel is also cited by anti-cheat vendors as a reason kernel-level protection is harder: any detection technique that requires kernel code can be read by anyone with access to the published kernel source, allowing cheat developers to understand and bypass the checks faster than the anti-cheat can update them. On Windows, the kernel itself is a trade secret.

### 7.3 Server-Side Mitigation as the Viable Alternative

Given the ceiling on what client-side user-space anti-cheat can achieve on Linux, a server-authoritative game design is the most robust defence available. The principle is straightforward: any game state that matters for fairness should be verified on the server, not trusted from the client. Concrete techniques include:

- **Movement validation.** The server rejects position updates that are physically impossible given the player's last-known state (teleportation hacks, speed hacks). This requires careful handling of latency and interpolation but is entirely server-side and platform-agnostic.

- **Damage validation.** The server verifies that a claimed hit was geometrically possible given the shooter's position, aim, and the target's hitbox at the relevant tick. Aim-assist cheats and trigger bots produce hit patterns that are statistically anomalous and can be detected by server-side ML.

- **Behavioural telemetry.** Systems like Valve's VACnet analyse action sequences (reaction times, movement patterns, aim trajectories) over extended game histories, flagging accounts whose behaviour falls outside the distribution of human performance. This is entirely server-side and therefore fully Linux-compatible.

The trade-off is engineering cost: server-authoritative validation requires careful game architecture from the start (not easily retrofit onto an existing client-authoritative design) and increases server compute costs. Games that launch with this architecture are much better positioned to support Linux without the security concerns raised by EA's experience with Apex Legends.

### 7.4 The Community Debate

The Linux gaming community has a long-standing debate about whether accepting a proprietary kernel module for anti-cheat purposes is preferable to being locked out of major titles. The majority position, shared by Valve and major distributions, is that requiring a proprietary ring-0 module as a condition of gaming sets an unacceptable precedent for the security and freedom of the Linux platform. The counter-argument — that users already accept proprietary GPU drivers and firmware blobs — has not prevailed in policy discussions.

There is a meaningful distinction between proprietary GPU drivers and proprietary anti-cheat kernel modules, however. An NVIDIA or AMD driver grants the GPU vendor code the ability to manage GPU hardware that the user owns. A proprietary anti-cheat kernel module grants a third-party game publisher's code unrestricted access to the user's entire system memory, process list, and hardware state — with no recourse if the module misbehaves. This is the security objection, and it is why the majority of Linux distributions (including Valve's SteamOS) would not include such a module by default even if one were developed.

The practical consequence is that the EAC/BattlEye user-space model remains the ceiling of what Linux gaming can offer for anti-cheat, and games whose publishers require kernel-level integrity will remain unavailable on Linux through Wine/Proton. The path forward for the Linux gaming ecosystem runs through server-side validation, behavioural analytics, and publishers making a deliberate business decision to enable the user-space anti-cheat options that are already available — not through a Linux kernel anti-cheat module.

---

## 8. Game Developer Guidance: Enabling Linux Anti-Cheat

This section is addressed to developers of games that already integrate EAC or BattlEye and are evaluating whether to enable Linux/Steam Deck support.

### 8.1 Enabling EAC Linux Support (EOS Integration)

The following assumes your game uses the current EOS SDK version of EAC. If you are on the older Kamu/v1 standalone integration, the portal steps differ (see the Steamworks documentation linked in §10) but the file-deployment principle is the same.

**Step 1: Enable Linux in the EOS Developer Portal.**

```text
1. Log in at dev.epicgames.com/portal
2. Select your product → Anti-Cheat → Platform Configuration
3. Toggle "Linux" to Enabled
4. Download the latest EOS Anti-Cheat SDK
```

**Step 2: Extract and deploy the Linux library.**

```bash
# In your SDK download:
SDK_PATH="./SDK/Client/Assets/Plugins/x86_64"
GAME_DEPOT="./YourGame/depot"

# Rename and copy the ELF library alongside the Windows DLL
cp "${SDK_PATH}/libeasyanticheat.so" "${GAME_DEPOT}/easyanticheat_x64.so"

# Verify the Windows DLL is present
ls "${GAME_DEPOT}/EasyAntiCheat_x64.dll"
ls "${GAME_DEPOT}/easyanticheat_x64.so"
```

**Step 3: Publish a Steamworks build.** Upload the updated depot to Steamworks and set the build live on the default branch. No executable changes are needed. [Source](https://partner.steamgames.com/doc/steamhardware/proton)

**Step 4: Enable Steam Deck compatibility in Steamworks.** Under the Steam Deck compatibility section of your Steamworks app admin, mark your title as compatible (or submit for Valve's review). This causes Steam to automatically install the Proton EasyAntiCheat Runtime for players.

### 8.2 Enabling BattlEye Linux Support

BattlEye's process requires no source code changes and no depot modifications:

```text
1. Contact your BattlEye technical contact and request
   Proton/Linux support enablement for your App ID.
   
2. If you do not have a BattlEye technical contact,
   reach out via your Valve partner contact or the
   Steamworks partner forums.
   
3. BattlEye enables the configuration on their server side.
   No SDK update, no depot change, no game rebuild required.
```

[Source](https://store.steampowered.com/news/group/4145017/view/3104663180636096966)

Mark your title as Steam Deck compatible in Steamworks. Steam will then automatically install the Proton BattlEye Runtime for your players.

### 8.3 Testing Anti-Cheat in a Proton Development Environment

Set up a local testing environment that mirrors what Steam Deck users will see:

```bash
# Install both anti-cheat runtimes via Steam (search in the Library,
# show "Tools" category):
#   "Proton EasyAntiCheat Runtime"   (for EAC games)
#   "Proton BattlEye Runtime"         (for BattlEye games)

# Launch your game with explicit runtime paths for testing:
STEAM_COMPAT_DATA_PATH="$HOME/.steam/steam/steamapps/compatdata/YOUR_APP_ID" \
STEAM_COMPAT_CLIENT_INSTALL_PATH="$HOME/.steam/steam" \
PROTON_EAC_RUNTIME="$HOME/.steam/steam/steamapps/common/Proton EasyAntiCheat Runtime" \
"$HOME/.steam/steam/steamapps/common/Proton - Experimental/proton" run \
  "$HOME/.steam/steam/steamapps/common/YourGame/YourGame.exe"

# For BattlEye, substitute:
PROTON_BATTLEYE_RUNTIME="$HOME/.steam/steam/steamapps/common/Proton BattlEye Runtime" \
```

Confirm that:
1. The game launches and reaches the main menu (anti-cheat initialisation).
2. Online multiplayer functions (the anti-cheat reports back to the game server successfully).
3. A clean exit without anti-cheat error dialogs.

### 8.4 Debugging Anti-Cheat Failures

When EAC or BattlEye fails to initialise under Proton, the log files are the first diagnostic resource.

**EAC log locations (within the Wine prefix):**

```bash
# Modern EOS-based EAC:
~/.steam/steam/steamapps/compatdata/YOUR_APP_ID/pfx/drive_c/users/steamuser/AppData/Roaming/EasyAntiCheat/anticheatlauncher.log

# Legacy Kamu/v1 EAC:
~/.steam/steam/steamapps/compatdata/YOUR_APP_ID/pfx/drive_c/users/steamuser/AppData/Roaming/EasyAntiCheat/service.log
~/.steam/steam/steamapps/compatdata/YOUR_APP_ID/pfx/drive_c/users/steamuser/AppData/Roaming/EasyAntiCheat/loader.log
```

Common error patterns and their meanings:

```text
"Failed to load the anti-cheat module" (error code 206)
→ Typically indicates that easyanticheat_x64.so is missing from
  the game depot or lacks execute permissions. Verify the file is
  present: ls -l "$GAME_DIR/easyanticheat_x64.so"

"Anti-cheat module reported initialization failure"
→ Typically indicates the Linux platform has not been enabled in
  the EAC/EOS developer portal. Confirm with the game developer.
  Note: needs verification — internal EAC error text is not
  publicly documented.

"EAC bootstrapper version mismatch" (or similar version error)
→ The game's bundled EAC SDK may be incompatible with the Proton
  EasyAntiCheat Runtime. Try switching to Proton Experimental.
  Note: needs verification — exact wording varies across EAC
  versions and is not publicly documented.
```
[Source](https://github.com/ValveSoftware/Proton/issues/6740) (error code 206 documented in Proton issue; remaining patterns are indicative, drawn from community reports — see ProtonDB)

**BattlEye:** BattlEye logs are typically written within the Wine prefix, often at a path such as:

```bash
# Note: needs verification — BattlEye's log path is not publicly
# documented and may vary by game. The path below is commonly
# reported in community bug threads but should be confirmed per title.
~/.steam/steam/steamapps/compatdata/YOUR_APP_ID/pfx/drive_c/Program Files/BattlEye/BEClient.log
```

If BattlEye blocks launch with no log output, confirm that the `PROTON_BATTLEYE_RUNTIME` path points to a valid installation of the Proton BattlEye Runtime. Check Proton issue tracker and ProtonDB for your specific title's log location.

**Enabling Proton diagnostic output:**

```bash
# Print anti-cheat related Wine debug channels:
PROTON_LOG=1 WINEDEBUG=+loaddll,+module %command%

# This shows which DLLs are loaded and which .so overrides are applied,
# helping confirm that easyanticheat_x64.so is being picked up.
```

### 8.5 Steam Deck Review Criteria for Anti-Cheat Games

Valve's Steam Deck compatibility review evaluates:
- Whether the anti-cheat runs and does not block gameplay.
- Whether the game is playable in online multiplayer mode (not just single-player with anti-cheat bypassed in offline mode).

A game that uses EAC or BattlEye but has not enabled Linux support will receive a **"Unsupported"** rating if the anti-cheat is the blocking factor, regardless of graphics/input compatibility. Enabling Linux anti-cheat support is therefore a prerequisite for a "Playable" or "Verified" badge. [Source](https://partner.steamgames.com/doc/steamhardware/proton)

---

## 9. Integrations

- **Ch28 (Windows Compatibility Layer)** — The Wine PE loader, pressure-vessel container, and Proton architecture that hosts the EAC and BattlEye native `.so` libraries. The `WINEDLLOVERRIDES` mechanism and the PE load sequence are prerequisites for understanding how the anti-cheat bridge operates.
- **Ch104 (DXVK and VKD3D-Proton)** — The D3D translation layers that run in the same Wine process as EAC/BattlEye. The anti-cheat `.so` shares address space with DXVK's Vulkan dispatch, and compatibility between specific DXVK/VKD3D-Proton versions and EAC/BattlEye versions can affect stability.
- **Ch167 (NTSYNC)** — The `ntsync` kernel driver (`/dev/ntsync`, merged in Linux 6.14) provides NT synchronisation semantics that improve Wine's overall process model, benefiting the correctness of threading in EAC/BattlEye PE-side code running under Wine.
- **Ch78 (Gamescope and the Steam Deck Compositor)** — On Steam Deck, Proton games run as nested Wayland clients inside gamescope. Anti-cheat libraries must not interfere with gamescope's frame submission model; the EAC/BattlEye runtimes are tested specifically against this configuration.
- **Ch80 (GPU Security)** — The Linux security model covering `ptrace`, `/proc/PID/mem`, `PR_SET_DUMPABLE`, and the absence of Windows's protected process model. These interfaces define the ceiling of what user-space anti-cheat can detect and block on Linux.

---

## 10. References

- Epic Online Services launch announcement (September 2021): [Epic Online Services Launches Anti-Cheat Support for Linux, Mac, and Steam Deck](https://onlineservices.epicgames.com/news/epic-online-services-launches-anti-cheat-support-for-linux-mac-and-steam-deck)
- Phoronix coverage of EAC Linux support: [Epic Games Announces Easy Anti-Cheat For Linux](https://www.phoronix.com/news/Easy-Anti-Cheat-Linux)
- GamingOnLinux EAC announcement coverage: [Epic Games announce full Easy Anti-Cheat support for Linux including Wine & Proton](https://www.gamingonlinux.com/2021/09/epic-games-announce-full-easy-anti-cheat-for-linux-including-wine-a-proton/)
- BattlEye Proton announcement tweet (September 24, 2021): [BattlEye on X](https://x.com/TheBattlEye/status/1441477816311291906)
- Phoronix coverage of BattlEye announcement: [BattlEye To Support Valve's Steam Deck / Proton](https://www.phoronix.com/news/BattlEye-Proton-Steam-Deck)
- GamingOnLinux on BattlEye enablement process: [Supporting Linux / Proton and the Steam Deck with BattlEye is just an email away](https://www.gamingonlinux.com/2021/11/supporting-linux-proton-and-the-steam-deck-with-battleye-is-just-an-email-away/)
- Valve Steamworks Update on BattlEye + Proton: [Steamworks Development — Update on BattlEye + Proton support](https://store.steampowered.com/news/group/4145017/view/3104663180636096966)
- Steamworks documentation (Proton + anti-cheat): [Steam Hardware and Proton — Steamworks Documentation](https://partner.steamgames.com/doc/steamhardware/proton)
- Riot Games on Vanguard and Linux (April 2024): [Riot Games talk Vanguard anti-cheat for League of Legends and why it's a no for Linux](https://www.gamingonlinux.com/2024/04/riot-games-talk-vanguard-anti-cheat-for-league-of-legends-and-why-its-a-no-for-linux/)
- Riot Vanguard Wikipedia: [Riot Vanguard](https://en.wikipedia.org/wiki/Riot_Vanguard)
- League of Linux — post-Vanguard impact: [LoL After Vanguard — League of Linux](https://leagueoflinux.org/post_vanguard/)
- EA official statement on Apex Legends Linux anti-cheat (November 2024): [Dev Team Update: Linux & Anti-Cheat — Apex Legends](https://forums.ea.com/blog/apex-legends-game-info-hub-en/dev-team-update-linux--anti-cheat/9468053)
- GamingOnLinux — Apex Legends EOS anti-cheat breakage (May 2024): [Apex Legends Upheaval update with EOS anti-cheat breaking Linux](https://www.gamingonlinux.com/2024/05/apex-legends-upheaval-update-live-with-eos-anti-cheat-breaking-it-on-steam-deck-linux/)
- GamingOnLinux — GTA V BattlEye breaks Linux (September 2024): [Grand Theft Auto V gets BattlEye anti-cheat, breaks online play on Steam Deck / Linux](https://www.gamingonlinux.com/2024/09/grand-theft-auto-v-gets-battleye-anti-cheat-breaks-online-play-on-steam-deck-linux/)
- GamingOnLinux — Dune: Awakening BattlEye Linux enabled (May 2025): [Dune: Awakening will have BattlEye enabled for Linux](https://www.gamingonlinux.com/2025/05/dune-awakening-will-have-battleye-enabled-for-linux-steamos-steam-deck/)
- GamingOnLinux BattlEye compatibility tracker: [BattlEye compatibility list for Steam Deck, Linux, SteamOS](https://www.gamingonlinux.com/anticheat/vendor/battleye/)
- Are We Anti-Cheat Yet? community database: [areweanticheatyet.com](https://areweanticheatyet.com/)
- nProtect GameGuard WineHQ forum thread: [nProtect GameGuard Engine — WineHQ Forums](https://forum.winehq.org/viewtopic.php?t=1077)
- Helldivers 2 Proton Hotfix (February 2024): [Proton Hotfix updated to support HELLDIVERS 2 on Steam Deck / Linux](https://www.gamingonlinux.com/2024/02/proton-hotfix-updated-to-support-helldivers-2-on-steam-deck-linux/)
- Valve Anti-Cheat developer wiki: [Valve Anti-Cheat — Valve Developer Community](https://developer.valvesoftware.com/wiki/Valve_Anti-Cheat)
- Samuel Tulach — The issue of anti-cheat on Linux: [The issue of anti-cheat on Linux](https://tulach.cc/the-issue-of-anti-cheat-on-linux/)
- Sam4k — What's the deal with anti-cheat on Linux: [What's The Deal With Anti-Cheat On Linux?](https://sam4k.com/whats-the-deal-with-anti-cheat-on-linux/)
- Adrian's Security Research — How kernel anti-cheats work: [How Kernel Anti-Cheats Work: A Deep Dive](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/)
- Linux Integrity Measurement Architecture (IMA): [The Integrity Measurement Architecture — LWN.net](https://lwn.net/Articles/137306/)
- Activision TPM 2.0 / Secure Boot requirement (RICOCHET): [Trusted Platform Module (TPM) 2.0 and Secure Boot for Call of Duty](https://support.activision.com/articles/trusted-platform-module-and-secure-boot)
- Andrew Moore — Secure Boot, TPM and Anti-Cheat: [Secure Boot, TPM and Anti-Cheat Engines](https://andrewmoore.ca/blog/post/anticheat-secure-boot-tpm/)
- Proton issue tracker — EAC bootstrapper v1.6.0 regression: [ValveSoftware/Proton issue #6740](https://github.com/ValveSoftware/Proton/issues/6740)
- Wikipedia — Easy Anti-Cheat: [Easy Anti-Cheat](https://en.wikipedia.org/wiki/Easy_Anti-Cheat)
- Wikipedia — BattlEye: [BattlEye](https://en.wikipedia.org/wiki/BattlEye)
- Wikipedia — Kernel-level anti-cheat: [Kernel-level anti-cheat](https://en.wikipedia.org/wiki/Kernel-level_anti-cheat)
