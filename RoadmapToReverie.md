# Reverie Codebase Audit & Transition Roadmap

## Executive Summary — Codebase Health

| Metric | Value |
|---|---|
| **Core Language** | Object Pascal (Lazarus/FPC) |
| **Pascal Source Files** | ~529 `.pas` / ~164 `.lfm` / ~119 `.lrt` |
| **C/C++ Components** | Kernel driver (`DBKKernel`), DBVM hypervisor, UEFI loader, D3D hooks, ceserver, Mono/DotNet collectors, tcclib |
| **Build Systems** | Lazarus `.lpi`, Visual Studio `.sln/.vcxproj`, GNU Make, AppVeyor CI |
| **Localization Languages** | 6 (DE, FR, IT, RU, ZH-CN, ZH-TW) + PO/POT files |
| **Lines of Largest Files** | `disassembler.pas` (776 KB), `disassemblerarm64.pas` (514 KB), `LuaHandler.pas` (421 KB), `MainUnit.pas` (319 KB), `memscan.pas` (286 KB) |

**Overall Health:** The codebase is a mature, ~20-year veteran project. It is functional and feature-rich, but carries significant tech debt: monolithic 300KB+ Pascal units, no automated tests, a stale AppVeyor CI using Lazarus 1.6.4, inline assembly anti-tamper systems, monetization bloat ("Cheat-E-Coins"), and zero separation between UI and logic in core units. The code is, however, well-structured enough that a systematic rebrand and cleanup is tractable.

---

## 1. Rebranding & Renaming: `Cheat Engine` → `Reverie`

### 1.1 Directory & File Renames

| Current Path | Proposed Path | Impact |
|---|---|---|
| `Cheat Engine/` (root source dir) | `Reverie/` | **HIGH** — Every build script, CI config, and `.lpi` references this path |
| `cheatengine.lpi` | `reverie.lpi` | Lazarus project file — main entry point |
| `cheatengine.lpr` | `reverie.lpr` | Main program unit (`program cheatengine;` → `program reverie;`) |
| `cheatengine.ico` | `reverie.ico` | Application icon (needs new artwork) |
| `cheatengine.res` / `cheatengine-x86_64.po` | `reverie.res` / `reverie-x86_64.po` | Resource and localization files |
| `cecore.lpi` / `cecore.lpr` | `reveriecore.lpi` / `reveriecore.lpr` | Core library project |
| `ceserver/` | `reverieserver/` | Linux/Android remote debugging server (C) |
| All `CE*` prefixed DLLs in `bin/` | `Reverie*` or `rv*` prefix | `CED3D9Hook.dll`, `CED3D10Hook.dll`, `CED3D11Hook.dll`, etc. |

### 1.2 Central String Constants (Single Source of Truth)

The branding is already partially centralized via compile-time constants in `MainUnit2.pas` (lines 22–43). This is **the** key file:

```pascal
// MainUnit2.pas — lines 22–43
const
  strCheatEngine = 'Cheat Engine';    // → 'Reverie'
  strCheatTable = 'Cheat Table';      // → 'Reverie Table' or 'Table'
  strCheat = 'Cheat';                 // → 'Mod' or 'Modification'
  strTrainer = 'Trainer';             // → Keep or rename to 'Modifier'
  strMyCheatTables = 'My Cheat Tables'; // → 'My Tables'
  strSpeedHack = 'Speedhack';         // → Keep (functional name)
```

> [!IMPORTANT]
> There is also an `{$ifdef altname}` branch that uses `'Runtime Modifier'`. This conditional compilation flag and the `altname` path in `first.pas` (line 84) should be collapsed — Reverie should be the **only** identity.

**Files referencing `strCheatEngine` (via `grep`):** 119+ matches across ~50 files. Most are registry paths, dialog messages, and temp directory names. Because they use the constant, changing `MainUnit2.pas` will cascade automatically for the Pascal codebase.

**Hardcoded string instances** (need manual replacement):
- `cheatengine.lpr` L293: `Application.Title:='Cheat Engine 7.5'`
- `cheatengine.lpr` L416: `OutputDebugString('Starting CE')`
- `first.pas` L84: `'Cheat Engine'` (hardcoded, not using constant)
- `frmmicrotransactionsunit.pas` L73: `'Cheat Engine'` user agent
- All 164 `.lfm` form files — caption properties containing "Cheat Engine"
- All 119 `.lrt` resource translation files
- `aboutunit.lfm` / `aboutunit.pas` — About dialog with Patreon links
- `tcclib/` — **40+ comments** in vendored TinyCC source (`tccrun.c`, `tccpe.c`, `libtcc.c`, `tcc.h`) saying `//Cheat Engine modification` — rebrand to `//Reverie modification`
- `Tutorial/` — hardcoded `cheatengine.org` and `forum.cheatengine.org` URLs in `Unit1.pas`, `Unit4.pas`, `Unit7.pas`, `Unit10.pas`, `frmhelpunit.pas`, `cetranslator.pas`
- `MainUnit.pas` L5902, L7300, L11140 — hardcoded `wiki.cheatengine.org` and `cheatengine.org` URLs
- `DBK32functions.pas` L3417 — hardcoded `cheatengine.org/dbkerror.php` URL
- `sfx/` — standalone trainer launcher with hardcoded CE version strings and `cheatengine.org` URL
- `csharpcompiler.pas` — hardcoded `D:\git\cheat-engine\` paths in `{$ifdef standalonetest}` blocks
- `tcclib.pas` — hardcoded `D:\git\cheat-engine\` and `cheatengine-x86_64.app` paths in test blocks

### 1.3 Registry Keys (Critical for Conflict Avoidance)

The application stores all settings under `HKEY_CURRENT_USER\Software\Cheat Engine`. This is used in **40+ locations** across the codebase.

| Current Registry Path | New Path |
|---|---|
| `\Software\Cheat Engine\` | `\Software\Reverie\` |
| `\Software\Cheat Engine\Plugins32` / `Plugins64` | `\Software\Reverie\Plugins32` / `Plugins64` |
| `\Software\Cheat Engine\Tools` | `\Software\Reverie\Tools` |
| `\Software\Cheat Engine\DissectData` | `\Software\Reverie\DissectData` |
| `\Software\Cheat Engine\Disassemblerview ...` | `\Software\Reverie\Disassemblerview ...` |
| `\Software\Cheat Engine\Hexview ...` | `\Software\Reverie\Hexview ...` |
| `\Software\Cheat Engine\Lua Highlighter` | `\Software\Reverie\Lua Highlighter` |
| `\Software\Cheat Engine\CPP Highlighter` | `\Software\Reverie\CPP Highlighter` |
| `\Software\Cheat Engine\Ignored Exceptions` | `\Software\Reverie\Ignored Exceptions` |
| `\Software\Cheat Engine\Process Window\Font` | `\Software\Reverie\Process Window\Font` |

The central registry wrapper `ceregistry.pas` (lines 49–62) opens `\Software\+strCheatEngine+\`. Changing the constant in `MainUnit2.pas` will fix this automatically, but **40+ other files open subkeys directly** and must be audited individually.

> [!NOTE]
> **Decision: No migration.** Reverie starts fresh under `\Software\Reverie\`. No import from the old CE registry hive.

### 1.4 Named Pipes

| Current Pipe Name | Location | New Name |
|---|---|---|
| `\\.\pipe\CEWINHOOK<pid>` | `winhook/com.pas` L124 | `\\.\pipe\RVWINHOOK<pid>` |
| `\\.\pipe\CEWINHOOKC<pid>` | `winhook/com.pas` L53, `LuaHandler.pas` L13399 | `\\.\pipe\RVWINHOOKC<pid>` |

### 1.5 Temp Directories

| Current Path | Source | New Path |
|---|---|---|
| `%TEMP%\Cheat Engine\` | `memscan.pas` L8765 | `%TEMP%\Reverie\` |
| `%TEMP%\Cheat Engine Symbols\` | `symbolhandler.pas` L1669, `symbolsync.pas` L299 | `%TEMP%\Reverie Symbols\` |

### 1.6 Application Directory Helper

The function `GetCEDir` (defined in `CEFuncProc.pas` L2720, used in **15+ files**) returns the application's working directory. The function name and all call sites reference "CE" and must be renamed to `GetReverieDir` or similar.

### 1.7 File Formats & Extensions

The application uses custom file extensions that contain "CE" or "Cheat" branding:

| Extension | Purpose | Decision |
|---|---|---|
| `.CT` | Cheat Table (primary save format) | **Keep** — ubiquitous, short, not obviously branded |
| `.CETRAINER` | Trainer executable wrapper | **Rename** → `.RVTRAINER` or keep for backward compat |
| `.CESIG` | Table signature file | **Rename** → `.RVSIG` |
| `CheatEngineTableVersion` (XML attribute) | CT file version marker inside saved tables | **Keep for read compat**, add `ReverieTableVersion` for new saves |
| `CheatTable` (XML root node) | Root node name in `.CT` files | **Keep for read compat**, recognize both `CheatTable` and `ReverieTable` |
| `CheatEntries`, `CheatCodes` (XML node names) | Sub-nodes in `.CT` files | **Keep** — changing breaks all existing tables |

### 1.8 Kernel Driver Identity

| File | Current Value | Change |
|---|---|---|
| `DBK32.inf` | `ManufacturerName="Cheat Engine"` | `"Reverie"` |
| `DBK64.inf` (line 75) | `ManufacturerName="Cheat Engine"` | `"Reverie"` |
| `sigcheck.c` | References to "Cheat Engine" in signature verification | Update strings |

### 1.9 Localization (.po/.lrt) Files

There are **6 language directories** under `bin/languages/` (de_DE, fr_FR, it_IT, ru_RU, zh_CN, zh_TW) plus root `.po` files including `cheatengine.po` and `cheatengine-x86_64.po`. All contain translated strings referencing "Cheat Engine" and need systematic `msgstr` updates.

---

## 2. De-bloating & Cleanup

### 2.1 Monetization / Paywall / Ad-Serving Code — **REMOVE ENTIRELY**

> [!CAUTION]
> The following units form a complete monetization ecosystem: virtual currency, microtransactions, ad serving, and anti-debug/anti-tamper systems. This must be **completely removed**.

| File | Purpose | Action |
|---|---|---|
| `cheatecoins.pas` (461 lines) | Virtual currency system with VEH-based anti-debug, hardware breakpoints, inline asm anti-tamper, `ExitProcess` calls | **DELETE** |
| `frmmicrotransactionsunit.pas` + `.lfm` (83KB form) | Microtransaction purchase UI, calls `cheatengine.org/microtransaction.php` | **DELETE** |
| `frmAdConfigUnit.pas` + `.lfm` + `.lrt` | Ad configuration panel for trainers | **DELETE** |
| `cesupport.pas` (~160 lines) | Ad-serving support unit — fetches ads from `cheatengine.org/ceads.php`, referenced by `MainUnit.pas`, `LuaHandler.pas`, `trainergenerator.pas`, `frmAdConfigUnit.pas` | **DELETE** |

**References to remove from:**
- `MainUnit.pas` L8589: `EnableCheatECoinSystem;` call
- `MainUnit.pas` L1126: `cheatecoins` in uses clause
- `MainUnit.pas` L41: `cesupport` in uses clause
- `MemoryRecordUnit.pas` L464: `cheatecoins` in uses clause
- `cheatengine.lpr` L117-118: `cheatecoins, frmMicrotransactionsUnit` in uses
- `cheatengine.lpr` L60: `cesupport` in uses clause
- `cheatengine.lpr` L62: `frmAdConfigUnit` in uses
- `trainergenerator.pas` L18: `frmAdConfigUnit, cesupport` in uses + ~30 lines of ad support logic
- `LuaHandler.pas` L111: `cesupport` in uses clause
- `aboutunit.pas` L159: Patreon `ShellExecute` call

### 2.2 External Phone-Home URLs

| URL | Location | Action |
|---|---|---|
| `https://cheatengine.org/microtransaction.php` | `frmmicrotransactionsunit.pas` | Remove with unit |
| `http://www.cheatengine.org/ceads.php` | `cesupport.pas` L159 | Remove with unit |
| `https://cheatengine.org/dbkerror.php` | `DBK32functions.pas` L3417 | Remove or replace |
| `https://wiki.cheatengine.org/` | `MainUnit.pas` L5902, L11140 | Replace with Reverie docs URL or remove |
| `http://www.cheatengine.org/?referredby=` | `MainUnit.pas` L7300 | Remove (referral tracking) |
| `https://cheatengine.org/tutorial.php` | `Tutorial/frmhelpunit.pas` L87, L89 | Replace or remove |
| `http://forum.cheatengine.org/` | `Tutorial/Unit4.pas`, `Unit1.pas`, `Unit10.pas` | Replace or remove |
| `https://cheatengine.org/help/pointer-scan.htm` | `Tutorial/Unit7.pas` L88 | Replace or remove |
| `https://www.patreon.com/cheatengine` | `aboutunit.pas`, `README.md`, `READMEes.md` | Remove |
| `.github/FUNDING.yml` | Patreon funding link | Replace with Reverie funding info or delete |

### 2.3 Legacy OS Support to Strip

| Code | Location | Rationale |
|---|---|---|
| `cOsWin98`, `cOsWin98SE` constants | `CEFuncProc.pas` L1534-L1590 | Win98/ME detection is dead code; no modern system hits these branches |
| `iswin2kplus` global | `globals.pas` L144 | Always true on Win7+. Can be replaced with a compile-time constant or removed |
| Windows XP RF flag workarounds | `debugeventhandler.pas` L860, L880 | XP-specific hardware breakpoint workaround |
| `setDPIAware` Win7 fallback | `first.pas` L42 | `Shcore.dll` + `SetProcessDpiAwareness` is available on Win8.1+; the User32 fallpath is for Win7. Consider Win10+ minimum |
| `strNeedNewerWindowsVersion` | `MainUnit2.pas` L76 | References "Windows 2000+" — dead since we target Win10 minimum |
| `EncodePointerNI` / `DecodePointerNI` | `feces.pas` (→ `tablesignature.pas`) L90-103 | XP fallback for missing `EncodePointer` API — remove, always available on Win10 |

### 2.4 Profanity & Unprofessional Code

| Item | Location | Action |
|---|---|---|
| `TFormFucker` class name | `cheatengine.lpr` L262 | Rename to `TFormFontApplier` |
| "fuuuuucking time" comment | `cheatengine.lpr` L270 | Clean up |
| `feces.pas` unit name | `feces.pas` | Determine purpose & rename |
| "Learn to use 14 byte jmps people" | `globals.pas` L170 | Clean comment |

---

## 3. Code Modernization & Technical Debt

### 3.1 C/C++ Components

#### Kernel Driver (`DBKKernel/`) — 51 files, ~500KB C
- **Current standard:** C89/C99 with WDK headers
- **Key concern:** Uses `ExAllocatePool` (deprecated since WDK 10.0.19041) — should migrate to `ExAllocatePool2`.
- **Memory safety:** Raw pointer arithmetic throughout `memscan.c`, `debugger.c`, `vmxoffload.c`. No bounds checking or RAII patterns.
- **Recommendation:** Upgrade to WDK-compatible C11. Since this is a kernel driver, C++ is limited but `ExAllocatePool2` and `POOL_FLAG_*` should be adopted.

#### D3D Hook DLLs (`Direct x mess/`) — Visual Studio 2017 solution
- 4 sub-projects: DXHookBase, CED3D9Hook, CED3D10Hook, CED3D11Hook
- **Current standard:** C++11/14 (VS2017 default)
- **Recommendation:** Upgrade to VS2022 with C++17. Replace raw `new`/`delete` with smart pointers. Add D3D12 hook support.

#### ceserver (Linux/Android) — ~300KB C
- Clean POSIX C with pthread usage. Cross-platform architecture is solid.
- **Recommendation:** Modernize to C11, add AddressSanitizer/ThreadSanitizer CI runs.

#### MonoDataCollector / DotNetDataCollector — VS solutions
- Injected DLLs for .NET/Mono inspection. Relatively modern C/C++.
- **Recommendation:** Convert to CMake for unified build.

#### tcclib (bundled TinyCC) — Vendored library
- Used for `{$C}` and `{$CCODE}` auto-assembler support.
- **Recommendation:** Keep vendored but tag the version. Consider making it a git submodule.

### 3.2 Object Pascal / Lazarus Codebase

#### Monolithic Units — Critical Refactoring Targets

| Unit | Size | Problem |
|---|---|---|
| `MainUnit.pas` | 319 KB | God-class: contains scan logic, UI event handlers, process management, and hotkey handling in one unit |
| `MemoryBrowserFormUnit.pas` | 181 KB | Full debugger UI + disassembly + hex view logic tightly coupled |
| `LuaHandler.pas` | 421 KB | **Largest file in the project** — all Lua bindings in one file. Should be split into domain-specific binding modules |
| `StructuresFrm2.pas` | 220 KB | Structure dissector with UI and logic interleaved |
| `memscan.pas` | 286 KB | Memory scanner — core engine logic that should be in a separate non-UI library |
| `disassembler.pas` | 776 KB | x86/x64 disassembler — single-file, but inherently complex |

**Recommended refactoring priority:**
1. Extract scan engine from `MainUnit.pas` → new `ScanEngine.pas`
2. Split `LuaHandler.pas` into ~10 domain binding units (process, memory, assembler, debugger, UI, etc.)
3. Separate `memscan.pas` business logic from any UI dependencies
4. Create a `ReverieCore` library package containing all non-UI logic

#### Deprecated Patterns
- **Broad `except` without handler:** Dozens of `try ... except end;` blocks that silently swallow errors (e.g., `ceregistry.pas` L88-91)
- **`{$mode delphi}` vs `{$mode objfpc}`:** Mixed modes across units. Should standardize on `{$mode objfpc}{$H+}` for consistency
- **Direct registry access scattered everywhere:** Should be centralized through `ceregistry.pas`
- **Untyped pointers:** `Memory: ^Byte` in `globals.pas` L108 — should use proper typed arrays

### 3.3 Lua Scripting Engine

- **Current Lua version:** 5.3 (bundled as `lua53-32.dll` / `lua53-64.dll`)
- **Engine integration:** `LuaHandler.pas` (421 KB) — the largest file. Contains ~10,000+ lines of `lua_register` calls.
- **JIT support:** `luaJit.pas` (~3KB) — minimal LuaJIT bindings exist but appear optional.

**Recommendations:**
1. **Upgrade to Lua 5.4** — Better integer handling, generational GC, warning system
2. **Split LuaHandler.pas** into modular binding files (see 3.2 above)
3. **Add a Lua plugin manifest system** — Currently scripts are loaded from `bin/autorun/`. Add a `plugins.json` manifest for dependency resolution
4. **Sandbox untrusted scripts** — The `miLuaExecSignedOnly` setting exists but is rudimentary. Add proper sandboxing with restricted API access
5. **Type-safe bindings** — Consider auto-generating bindings from annotated Pascal interfaces instead of hand-writing thousands of `lua_push*` calls

---

## 4. Build System & CI/CD

### 4.1 Current State

| Component | Build System | Toolchain |
|---|---|---|
| Main GUI (`cheatengine.lpi`) | Lazarus IDE / `lazbuild` | FPC 3.2.2 + Lazarus 2.2.2 |
| Speedhack DLL | Lazarus (`speedhack.lpi`) | FPC (32+64 bit DLL) |
| VEH Debugger DLL | Lazarus (`vehdebug.lpi`) | FPC (32+64 bit DLL) |
| Lua client DLL | Lazarus (`luaclient.lpi`) | FPC (32+64 bit DLL) |
| Kernel driver (`DBKKernel.sln`) | VS2017 / WDK | MSVC |
| D3D Hooks (`Direct x mess.sln`) | VS2017 | MSVC |
| DotNet Compiler/Collector | VS Solutions | MSVC / .NET |
| Mono Collector | VS Solution | MSVC |
| tcclib | VS Solution | MSVC |
| ceserver | GNU Make | GCC |
| DBVM hypervisor | GNU Make | GCC (freestanding) |
| CI | AppVeyor (`appveyor.yml`) | Lazarus 1.6.4 (!!) — **severely outdated**, downloads from Dropbox |

> [!WARNING]
> The AppVeyor CI downloads Lazarus 1.6.4 installers from **personal Dropbox links** that may be taken down at any time. The README says Lazarus 2.2.2 is required, but CI uses 1.6.4. This discrepancy means CI builds may not match developer builds.

### 4.2 Proposed: Unified Build with GitHub Actions

#### Pascal Components (Lazarus)
```yaml
# .github/workflows/build.yml — Pascal components
- uses: gcarreno/setup-lazarus@v3
  with:
    lazarus-version: "3.4"     # Latest stable
    include-packages: ""
- run: lazbuild --build-mode="Release Win64" Reverie/reverie.lpi
- run: lazbuild --build-mode="Release Win32" Reverie/reverie.lpi
- run: lazbuild Reverie/speedhack/speedhack.lpi  # for each target
```

#### C/C++ Components (CMake)
Create `CMakeLists.txt` files for:
- `DBKKernel/` — Kernel driver (requires WDK, possibly separate workflow)
- `Reverie/Direct x mess/` — D3D hook DLLs
- `Reverie/MonoDataCollector/`
- `Reverie/DotNetDataCollector/`
- `Reverie/tcclib/`
- `Reverie/ceserver/` (Linux target only)

```yaml
# .github/workflows/build-native.yml
- uses: ilammy/msvc-dev-cmd@v1
  with:
    arch: x64
- run: cmake -B build -G "Visual Studio 17 2022"
- run: cmake --build build --config Release
```

#### Proposed Workflow Matrix

| Workflow | Trigger | Targets |
|---|---|---|
| `build-pascal.yml` | push, PR | Win32, Win64 (main + sub-projects) |
| `build-native.yml` | push, PR | D3D hooks, Mono/DotNet collectors, tcclib |
| `build-driver.yml` | manual, tag | ReverieKernel (requires WDK + signing) |
| `build-ceserver.yml` | push, PR | Linux x64, ARM64, Android |
| `release.yml` | tag | Aggregate all artifacts → GitHub Release |

### 4.3 Remove AppVeyor

Delete `appveyor.yml` after GitHub Actions are validated.

---

## 5. Phased Execution Roadmap

> [!NOTE]
> **Design principle:** Lazarus/FPC is a *temporary local dev tool* used only to verify Pascal changes compile during Phases 0–2. It is never a CI target. The C++ migration in Phase 3 is the path to a zero-external-dependency build (`cmake` + MSVC/GCC is all you need).

### Phase 0: Baseline Build Verification (Day 1)

Confirm every existing build target compiles locally before touching anything. This gives a known-good baseline to diff against if something breaks later.

- [ ] **Pascal GUI:** Build `cheatengine.lpi` with `lazbuild` locally (Win32 + Win64)
- [ ] **Speedhack DLL:** Build `speedhack/speedhack.lpi` (32-bit + 64-bit)
- [ ] **VEH Debugger DLL:** Build `VEHDebug/vehdebug.lpi` (32-bit + 64-bit)
- [ ] **Lua Client DLL:** Build `luaclient/luaclient.lpi` (32-bit + 64-bit)
- [ ] **Kernel Driver:** Open `DBKKernel/DBKKernel.sln` in VS, build x64
- [ ] **D3D Hooks:** Open `Direct x mess/Direct x mess.sln` in VS, build Win32 + x64
- [ ] **ceserver:** Run `make` in `ceserver/` on a Linux box or WSL
- [ ] Document exact toolchain versions (FPC, Lazarus, VS, WDK)
- [ ] Tag the repo: `git tag baseline-builds`

> [!TIP]
> Budget **one day**. If anything fails, fix it before proceeding.

---

### Phase 1: De-bloat & Clean Slate (Week 1–2)

Remove monetization, adware, legacy OS code, and unprofessional naming. Verify with local `lazbuild` after each change.

**Monetization & Ad-Serving:**
- [ ] Delete `cheatecoins.pas`, `frmmicrotransactionsunit.*`, `frmAdConfigUnit.*`
- [ ] Delete `cesupport.pas` (ad-serving unit fetching `cheatengine.org/ceads.php`)
- [ ] Remove all references in `uses` clauses (`MainUnit.pas`, `LuaHandler.pas`, `trainergenerator.pas`, `MemoryRecordUnit.pas`, `cheatengine.lpr`)
- [ ] Remove `EnableCheatECoinSystem` call from `MainUnit.pas`
- [ ] Remove Patreon links from `aboutunit.pas`, `README.md`, `READMEes.md`
- [ ] Remove or replace `.github/FUNDING.yml`

**External URLs:**
- [ ] Remove all `cheatengine.org` hardcoded URLs (see Section 2.2 for complete list)
- [ ] Decide: replace Tutorial URLs with Reverie equivalents, or strip Tutorial module entirely
- [ ] Remove referral tracking URL in `MainUnit.pas` L7300

**Legacy OS Code (Win10 minimum):**
- [ ] Strip Win98/Win2K/XP dead code paths
- [ ] Remove `iswin2kplus` global, `cOsWin98`/`cOsWin98SE` constants
- [ ] Remove XP RF flag workarounds in `debugeventhandler.pas`
- [ ] Remove `EncodePointerNI`/`DecodePointerNI` XP fallbacks in `feces.pas`
- [ ] Remove Win7 DPI fallback in `first.pas`
- [ ] Remove `strNeedNewerWindowsVersion` (references Win2000)

**Naming & Profanity:**
- [ ] Clean up profane identifiers (`TFormFucker` → `TFormFontApplier`, etc.)
- [ ] Rename `feces.pas` → `tablesignature.pas` + update all references
- [ ] Clean profane comment in `tcc.h` L200 (`"cheat engine fuckery"`)

- [ ] **Verify project compiles cleanly after all removals**

---

### Phase 2: Core Rebranding (Week 2–3)

Establish the Reverie identity across the entire codebase. Last phase that touches Pascal heavily.

**Constants & Compiler Flags:**
- [ ] Change constants in `MainUnit2.pas` (single source of truth)
- [ ] Collapse `{$ifdef altname}` branches — Reverie is the only identity
- [ ] Update `Application.Title`, `OutputDebugString` calls

**Directory & File Renames:**
- [ ] Rename directory `Cheat Engine/` → `Reverie/`
- [ ] Rename project files: `cheatengine.lpi` → `reverie.lpi`, etc.
- [ ] Rename `GetCEDir` → `GetReverieDir` in `CEFuncProc.pas` + all call sites
- [ ] Rename `CEFuncProc.pas` → `ReverieFuncProc.pas` (or similar)

**File Formats:**
- [ ] Decide on `.CETRAINER` → `.RVTRAINER` rename (with backward-compat read support)
- [ ] Decide on `.CESIG` → `.RVSIG` rename
- [ ] Add `ReverieTableVersion` XML attribute alongside `CheatEngineTableVersion` for new saves

**Driver:**
- [ ] Rename kernel driver: `DBK32`/`DBK64` → `Reverie32`/`Reverie64` (`.inf`, `.sln`, `.vcxproj`, `.sys`, loader code)

**IPC:**
- [ ] Update named pipe prefixes (`CEWINHOOK` → `RVWINHOOK`)

**Vendored Code:**
- [ ] Update 40+ `//Cheat Engine modification` comments in `tcclib/` → `//Reverie modification`
- [ ] Update hardcoded dev paths in `tcclib.pas`, `csharpcompiler.pas` (`D:\git\cheat-engine\`)
- [ ] Update Tutorial module URLs and strings
- [ ] Update `sfx/` standalone trainer launcher strings

**Assets & Documentation:**
- [ ] Create new application icon
- [ ] Write `README.md` for Reverie
- [ ] Update localization `.po` files for all 6 languages
- [ ] Delete `appveyor.yml`

- [ ] **Verify full build one final time with `lazbuild`**

---

### Phase 3: C++ Migration (Week 4+) — **Core Mission**

Incrementally replace Pascal/Lua with C++. Each ported module gets a CMake build immediately. The Pascal codebase shrinks with every PR until `lazbuild` is no longer needed.

#### 3.1 Architecture & Foundation
- [ ] Design modular C++ architecture: `libreverie` (core engine DLL/SO) + UI executable
- [ ] Set up root `CMakeLists.txt` with project-wide settings (C++17 minimum, MSVC/GCC, static analysis flags)
- [ ] Create `src/` directory structure alongside existing `Reverie/` Pascal code
- [ ] Define the **C-ABI boundary** for the Pascal→C++ transition (see 3.2)

> [!IMPORTANT]
> **UI Framework recommendation: Dear ImGui.** For a tool dealing with dense data visualization — memory editors, hex views, pointer maps, structure dissectors — Dear ImGui is the strongest candidate. It is lightweight, trivial to integrate into a CMake pipeline, excels at high-frequency updating data-dense UIs, and natively supports DirectX/OpenGL backends (aligning with the existing D3D hooking architecture). Evaluate alongside native Win32/WinUI as fallback.

#### 3.2 The FFI Bridge (Critical Transitional Piece)

During migration, the legacy Pascal UI must call the new C++ backend. You cannot drop a C++ class into a Lazarus form.

**Solution:** `libreverie` is compiled as a dynamic library (`.dll` / `.so`) with a strict, flat **C-API** (`extern "C"`) during the transitional phase. Pascal header bindings wrap this C-API so the legacy UI can call the new C++ engine.

```
┌──────────────────┐     extern "C" API      ┌──────────────────┐
│  Pascal UI       │ ◄──────────────────────► │  libreverie.dll  │
│  (legacy, dying) │    LoadLibrary +         │  (C++ engine)    │
│                  │    GetProcAddress         │                  │
└──────────────────┘                          └──────────────────┘
         │                                             │
         └──── gradually replaced by ──────────────────┘
                   Dear ImGui UI (C++)
```

- [ ] Define `reverie_api.h` — flat C function signatures for scanner, disassembler, process attach, memory read/write
- [ ] Write Pascal import unit `reverieapi.pas` — `external 'libreverie.dll'` declarations
- [ ] Implement versioned API (`REVERIE_API_VERSION`) so Pascal and C++ can evolve independently
- [ ] Gradually migrate Pascal callers from internal units to the C-API
- [ ] Once the Dear ImGui UI replaces all Pascal forms, the C-API becomes internal and can be C++-ified

> [!WARNING]
> Manual C-API wrappers are safer than auto-generators (SWIG, etc.) for memory-critical applications. Keep the API surface small and explicit.

#### 3.3 Port Pure-Logic Modules First (no UI dependencies)
These Pascal units are self-contained logic with no form/UI coupling — ideal first targets:

| Pascal Unit | Size | C++ Target | Priority |
|---|---|---|---|
| `disassembler.pas` + ARM variants | ~1.6 MB total | `libreverie/disasm/` | High — already mostly lookup tables |
| `Assemblerunit.pas` + ARM | ~485 KB | `libreverie/asm/` | High |
| `memscan.pas` | 286 KB | `libreverie/scanner/` | High — core feature, SIMD opportunity |
| `autoassembler.pas` | 155 KB | `libreverie/autoasm/` | Medium |
| `symbolhandler.pas` | 207 KB | `libreverie/symbols/` | Medium |
| `pointerscancontroller.pas` | 157 KB | `libreverie/ptrscan/` | Medium |
| `addressparser.pas` | 14 KB | `libreverie/parser/` | Low — small, port anytime |

> [!TIP]
> **Scanner SIMD Optimization:** The `memscan.pas` port is an opportunity for a major performance win. The Pascal implementation uses basic pointer arithmetic and legacy inline assembly for bulk memory comparisons. The C++ port should:
> - Enforce strict memory alignment (16/32/64-byte aligned buffers)
> - Use AVX2 intrinsics (`_mm256_cmpeq_epi8`, `_mm256_movemask_epi8`) for byte-pattern scanning
> - Provide AVX-512 codepath for supported CPUs (runtime detection via `__cpuid`)
> - Fall back to SSE2 as minimum baseline (available on all x64 CPUs)
>
> A vectorized scanner doing 32 or 64 bytes per comparison vs. the current byte-at-a-time approach can yield **10–30x throughput improvement** on large address spaces.

#### 3.4 Port Injected DLLs & Native Components
Already C/C++ — modernize and bring under CMake:

- [ ] D3D Hooks → CMake project, upgrade to C++17, add D3D12 support
- [ ] MonoDataCollector → CMake project
- [ ] DotNetDataCollector → CMake project
- [ ] tcclib → CMake project (or git submodule)
- [ ] ceserver → CMake project (replace GNU Makefile)
- [ ] Kernel driver → CMake + WDK, rename `DBK` → `Reverie`, migrate `ExAllocatePool` → `ExAllocatePool2`

#### 3.5 Port UI Layer
This is the largest and last piece. Powered by Dear ImGui (or chosen framework).

- [ ] Scaffold Dear ImGui application shell with DirectX 11 backend
- [ ] Port main window (currently `MainUnit.pas` — 319 KB god-class)
- [ ] Port memory browser / hex view (currently `MemoryBrowserFormUnit.pas` — 181 KB)
- [ ] Port structures dissector (currently `StructuresFrm2.pas` — 220 KB)
- [ ] Port all remaining form units (~164 `.lfm` forms)
- [ ] Remove Pascal UI completely — `lazbuild` is no longer needed

#### 3.6 Replace Lua Scripting
- [ ] Design C++ plugin/scripting interface (evaluate: native C++ plugins, Python embedding, or keep Lua-C API without Pascal wrapper)
- [ ] Port `LuaHandler.pas` (421 KB) bindings to direct Lua-C++ or replacement system
- [ ] Migrate `bin/autorun/` scripts to new system

---

### Phase 4: Build System & CI/CD (alongside Phase 3)

Built incrementally as C++ modules land. Each ported module adds to the CMake build.

- [ ] GitHub Actions: CMake build for all C++ modules (MSVC on Windows, GCC on Linux)
- [ ] GitHub Actions: Kernel driver build (WDK, manual trigger)
- [ ] GitHub Actions: Release workflow — aggregate artifacts → GitHub Release
- [ ] **PDB symbol archiving** — generate and archive `.pdb` files alongside release binaries for all Windows builds (critical for analyzing crash dumps from kernel drivers and injected DLLs)
- [ ] Static analysis in CI (clang-tidy, MSVC `/analyze`)
- [ ] AddressSanitizer / ThreadSanitizer for ceserver and `libreverie`
- [ ] Document build in `BUILD.md`: `cmake -B build && cmake --build build` — that's it
- [ ] Create `CONTRIBUTING.md`
- [ ] Create `LICENSE` file (verify CE's Apache 2.0 + proprietary components)

> [!IMPORTANT]
> **End-state build experience:** Clone, run `cmake --preset default`, done. No Lazarus, no FPC, no special IDE. Just CMake + a C++ compiler.

---

## Decisions (Resolved)

| Question | Decision | Impact |
|---|---|---|
| **Minimum Windows version** | **Windows 10** | Strip all Win98/2K/XP/7/8 codepaths, DPI fallbacks, `iswin2kplus` checks |
| **Registry migration** | **No** — clean break | No migration utility needed. Reverie starts fresh under `\Software\Reverie\` |
| **Driver naming** | **Rename** `DBK32`/`DBK64` → `Reverie32`/`Reverie64` | Update `.inf`, `.sys`, `.sln`, `.vcxproj`, IOCTL device names, and loader code |
| **`feces.pas`** | **Rename** → `tablesignature.pas` | The file is a **table signing & verification system** (ECDSA P-521 + SHA-512). The name "feces" was a joke acronym for "Friends Endorsing Cheat Engine System". Rename to `tablesignature.pas` |
| **Lua version** | **Keep Lua 5.3** for now | No breaking changes to existing scripts. **Long-term vision: migrate entire codebase from Pascal/Lua to C++** |

> [!NOTE]
> **Architecture vision:** The C++ migration in Phase 3 is the core mission, not a distant future goal. Phases 1–2 clean and rebrand the existing Pascal codebase as a practical first step to reduce the surface area being ported.

---

## Verification Plan

### Phase 1–2 Verification (Pascal era)
- Lazarus project compiles with local `lazbuild` (Win32 + Win64)
- All sub-projects (`speedhack`, `vehdebug`, `luaclient`) compile independently
- Launch executable and verify window title, About dialog, icon
- Verify registry entries go to `\Software\Reverie\`
- Verify temp files go to `%TEMP%\Reverie\`
- Load a `.CT` table file to verify backward compatibility

### Phase 3+ Verification (C++ era)
- `cmake -B build && cmake --build build` succeeds on clean checkout (MSVC + GCC)
- `libreverie` unit tests pass (scanner, disassembler, assembler)
- Pascal UI can call `libreverie.dll` through C-API bridge without crashes
- Dear ImGui UI renders memory browser, hex view, scan results correctly
- SIMD scanner benchmarks show measurable improvement over Pascal baseline
- PDB symbols are archived alongside every CI release build
- Attach to a test process and verify basic scan → write workflow end-to-end
