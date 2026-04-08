# Phase 2: Core Rebranding — Implementation Plan

## Overview

Phase 2 establishes the Reverie identity across the entire codebase. Phase 1 finished a clean, ad-free, telemetry-free, profanity-free codebase that still calls itself "Cheat Engine" everywhere. Phase 2 fixes that.

The order is **safety-first**: every step is structured so the project compiles after each commit. The single highest-leverage change (Step 1) cascades automatically through 30+ files via the central `strCheatEngine` constant. The directory rename (Step 9) is **last** because it touches every build script and IDE bookmark — doing it earlier inflates every other diff.

> [!IMPORTANT]
> **Prerequisite:** Phase 1 must be complete and committed. You must have a working `lazbuild --build-mode="Release 64-Bit"` baseline and a `git tag phase1-debloat` (or equivalent) to revert to if needed.

> [!CAUTION]
> **No registry migration.** Per `RoadmapToReverie.md` §1.3, Reverie starts fresh under `\Software\Reverie\`. Existing CE users will lose their settings. This is a deliberate decision — do not add a migration path mid-phase.

---

## Decisions (locked in before starting)

| Question | Decision | Source |
|---|---|---|
| **Identity collapse** | Delete every `{$ifdef altname}` branch. Reverie is the only identity. | Roadmap §1.2 |
| **Registry migration** | None. Clean break under `\Software\Reverie\`. | Roadmap §1.3 |
| **`.CT` extension** | Keep — too ubiquitous to break user files. | Roadmap §1.7 |
| **`.CETRAINER`/`.CESIG`** | Defer to Phase 3. Keep current names. | Roadmap §1.7 |
| **XML root nodes (`CheatTable`, `CheatEntries`)** | Keep — changing breaks all existing tables. | Roadmap §1.7 |
| **Driver names (`DBK32`/`DBK64`)** | Defer to Phase 3 (kernel driver work). Only update `ManufacturerName` strings now. | Roadmap §1.8 |
| **Localization `.po` files** | **Stub-translate.** Replace `msgstr` lines that are translations of "Cheat Engine" with the literal "Reverie" in the same language. Do not attempt full re-translation of every string. | Phase 2 scope |
| **Binary filename (`cheatengine-x86_64.exe`)** | **Defer to end of Phase 2.** Step 9 renames the binary along with the directory. | This plan |
| **Lazarus build mode default** | `Release 64-Bit` (already set by Phase 1 cleanup commit). | Phase 1 |

---

## Step 1: Flip the central constants in `MainUnit2.pas`

**Goal:** Change the single source of truth. This step alone cascades through 30+ files for registry paths, named pipes, temp dirs, and every `cename`-derived string.

**File:** `Cheat Engine\MainUnit2.pas`

### Step 1.1 — Replace the `{$ifdef altname}` constant block

**Lines 22–43** — replace the entire dual-identity block with a single Reverie identity:

```diff
- const
-   ceversion=7.51;
-   strVersionPart='7.5.1';
- {$ifdef altname}  //i'd use $MACRO ON but fpc bugs out
-   strCheatEngine='Runtime Modifier'; //if you change this, also change it in first.pas
-   strCheatTable='Code Table';   //because it contains code.... duh.....
-   strCheatTableLower='code table';
-   strCheat='Modification';
-   strTrainer='Modifier';
-   strTrainerLower='modifier';
-   strMyCheatTables='My Mod Tables';
-   strSpeedHack='Speedmodifier';
- {$else}
-   strCheatEngine='Cheat Engine';
-   strCheatTable='Cheat Table';
-   strCheatTableLower='cheat table';
-   strCheat='Cheat';
-   strTrainer='Trainer';
-   strTrainerLower='trainer';
-   strMyCheatTables='My Cheat Tables';
-   strSpeedHack='Speedhack';
- {$endif}
+ const
+   ceversion=7.51;
+   strVersionPart='7.5.1';
+   strCheatEngine='Reverie';
+   strCheatTable='Reverie Table';
+   strCheatTableLower='reverie table';
+   strCheat='Mod';
+   strTrainer='Trainer';
+   strTrainerLower='trainer';
+   strMyCheatTables='My Reverie Tables';
+   strSpeedHack='Speedhack';
```

> [!NOTE]
> The constant name `strCheatEngine` is kept as-is (just its **value** changes). Renaming the identifier itself would be a 119+ file rename for zero behavioral benefit; defer to Phase 3 if at all.

### Step 1.2 — Compile, smoke-test, commit

```
lazbuild --build-mode="Release 64-Bit" "Cheat Engine\cheatengine.lpi"
```

**Expected behavior after this single commit:**
- Window title shows "Reverie 7.5.1"
- Registry path becomes `HKEY_CURRENT_USER\Software\Reverie\`
- Named pipes become `\\.\pipe\CEWINHOOK<pid>` (still — they don't use the constant; see Step 4)
- Temp dirs become `%TEMP%\Reverie\` and `%TEMP%\Reverie Symbols\` automatically (they use `strCheatEngine`)
- `My Reverie Tables` folder appears in `%USERPROFILE%\Documents`
- About dialog group caption reads "Reverie 7.5.1"

**Smoke test:**
- Launch the executable
- Verify the title bar
- Verify `regedit.exe` shows the new key under `HKCU\Software\Reverie`
- Open and re-save a `.CT` table — confirm backward compatibility works

```
git commit -m "rebrand central identity constants to Reverie"
```

> [!WARNING]
> This commit is a **clean break** for any existing CE user running this build. Their settings (registry, custom types, plugins, hotkeys) appear lost because the application now reads from `\Software\Reverie\` instead. This is the intended Phase 2 behavior. Do not add a migration shim.

---

## Step 2: Collapse the `{$ifdef altname}` machinery

**Goal:** Reverie is the only identity. Every `{$ifdef altname}` block is dead and should be deleted to prevent the code from drifting back into a dual-identity state.

**Order:** Delete `altname` *consumers* first, then the `altname` *definition* (if any), then the unused helpers.

### Step 2.1 — `cheatengine.lpr`

**File:** `Cheat Engine\cheatengine.lpr`

**Lines 140–144** — collapse the `Images_alt.res` linkage:

```diff
- {$ifdef altname}
- {$R Images_alt.res}
- {$else}
  {$R Images.res}
- {$endif}
```

**Line 293** — fix the hardcoded title:
```diff
- Application.Title:='Cheat Engine 7.5';
+ Application.Title:='Reverie 7.5.1';
```

**Line 416** — fix the debug string:
```diff
- OutputDebugString('Starting CE');
+ OutputDebugString('Starting Reverie');
```

### Step 2.2 — `MainUnit.pas`

**File:** `Cheat Engine\MainUnit.pas`

**Lines ~8470–8480** — collapse the alternate logo resource selection:

```diff
-   {$ifdef altname}
-   rname:='IMAGES_ALT_CELOGO';
-   {$else}
    rname:='IMAGES_CELOGO';
-   {$endif}

    {$ifdef windows}
-   {$ifndef altname}
    if logo.Width>=90 then
-   {$endif}
    {$endif}
```

> [!NOTE]
> If you also rename the logo resource itself (e.g. `IMAGES_REVERIELOGO`) you must rename it inside `Images.res` too. That's an out-of-process Lazarus IDE step. **Easiest path: keep the resource name `IMAGES_CELOGO` for now and rename it in Phase 3 along with the rest of the asset overhaul.**

### Step 2.3 — `cetranslator.pas`

**File:** `Cheat Engine\cetranslator.pas`

Delete four `{$ifdef altname}` blocks:

- **Lines 75–77**: forward declaration of `function altnamer`
- **Lines 285–304**: the entire `TAltnameTranslator` class
- **Lines 327–331**: `SetResourceStrings(@altnamerri,nil); LRSTranslator:= TAltnameTranslator.create;`
- **Lines 399–416**: the `function altnamer(s: string): string;` body

The `cheatengine` filename probes inside `GetLocaleFileName` (lines 102, 105, 108) reference filenames inside `bin/languages/`. Leave them for now — Step 8 handles the rename of those `.po` files.

### Step 2.4 — `first.pas`

**File:** `Cheat Engine\first.pas`

**Lines 70 and 78** — the registry path is currently built via `{$ifdef altname}`. After Step 1, we want the unconditional Reverie identity. But `first.pas` runs at unit initialization *before* `MainUnit2`, so it cannot use `strCheatEngine` from `MainUnit2.pas` (circular). Hardcode it:

```diff
- if r.OpenKey('\Software\'+{$ifdef altname}'Runtime Modifier'{$else}'Cheat Engine'{$endif},false) then
+ if r.OpenKey('\Software\Reverie',false) then
```

```diff
-       if r.OpenKey('\Software\'+{$ifdef altname}'Runtime Modifier'{$else}'Cheat Engine'{$endif},true) then
+       if r.OpenKey('\Software\Reverie',true) then
```

> [!IMPORTANT]
> `first.pas` is one of the **two places** in the codebase where the registry path must be hardcoded (the other is `ceregreset/ceregreset.dpr`, see Step 2.6). Everywhere else uses the `strCheatEngine` constant.

### Step 2.5 — `formsettingsunit.pas`, `LuaHandler.pas`, `LuaInternet.pas`

**File:** `Cheat Engine\formsettingsunit.pas`

**Line 1904** — delete the `{$ifdef altname}` block.

**File:** `Cheat Engine\LuaHandler.pas`

**Lines 10620 and 10654** — delete two `{$ifdef altname}` blocks.

**File:** `Cheat Engine\LuaInternet.pas`

**Lines 372 and 374** — collapse the user-agent strings:
```diff
-     luaclass_newClass(L, TWinInternet.create({$ifdef altname}'Cheat Engine'{$else}cename{$endif}+' : luascript-'+Lua_ToString(L,1)))
+     luaclass_newClass(L, TWinInternet.create(cename+' : luascript-'+Lua_ToString(L,1)))
```
```diff
-     luaclass_newClass(L, TWinInternet.create({$ifdef altname}'Cheat Engine'{$else}cename{$endif}+' : luascript'));
+     luaclass_newClass(L, TWinInternet.create(cename+' : luascript'));
```

### Step 2.6 — `ceregreset/ceregreset.dpr` (separate registry-reset utility)

**File:** `Cheat Engine\ceregreset\ceregreset.dpr`

**Line 21** — collapse the dual-identity:
```diff
- const regname: string={$ifdef altname}'Runtime Modifier'{$else}'Cheat Engine'{$endif};
+ const regname: string='Reverie';
```

This is a standalone .dpr — it doesn't link `MainUnit2.pas`, so it must hardcode its own copy.

### Step 2.7 — `launcher/cheatengine.lpr` (32-bit launcher)

**File:** `Cheat Engine\launcher\cheatengine.lpr`

**Lines 49–53** — collapse the basename selection:
```diff
- {$ifndef altname}
  basename:='cheatengine';
- {$else}
- basename:='rt-mod';
- {$endif}
```

> [!NOTE]
> The `basename:='cheatengine'` itself stays for this step. Step 9 will rename the launcher and update this string when the binary itself is renamed. **Do not change `basename` here** — that would break the current bin/cheatengine-x86_64.exe lookup until Step 9 lands.

**Line 105** — rebrand the user-facing error:
```diff
-   MessageBoxW(0, pwidechar(exename+' could not be found. Please disable/uninstall your anti virus and reinstall Cheat Engine to fix this'),'Cheat Engine launch error',MB_OK or MB_ICONERROR);
+   MessageBoxW(0, pwidechar(exename+' could not be found. Please disable/uninstall your anti virus and reinstall Reverie to fix this'),'Reverie launch error',MB_OK or MB_ICONERROR);
```

### Step 2.8 — Compile, verify, commit

Verify with `grep -rn "altname" "Cheat Engine"` — should return zero hits.

```
git commit -m "collapse altname conditional, Reverie is the only identity"
```

---

## Step 3: Audit and rebrand stray hardcoded "Cheat Engine" strings

**Goal:** Even after Step 1, ~50 files contain literal `'Cheat Engine'` strings that bypass the constant. Fix them.

The safest approach: `grep -rn "'Cheat Engine'" "Cheat Engine"` and triage each hit. The expected categories:

| Category | Examples | Action |
|---|---|---|
| **Application titles / debug strings** | `cheatengine.lpr` L293, L416 | Already fixed in Step 2.1 |
| **Form captions in `.lfm`** | `aboutunit.lfm`, `MainUnit.lfm` window titles | **Step 7** handles all `.lfm` files in bulk |
| **Resource strings (`resourcestring`)** | `tablesignature.pas`, `OpenSave.pas`, etc. | Replace `'Cheat Engine'` with `strCheatEngine` (which is now `'Reverie'`) where the unit can `uses MainUnit2` |
| **Comments / DBK error strings** | `dbk32/DBK32functions.pas` | Already use `strCheatEngine` in most cases — re-verify |
| **Attribution comments** | `lua/lua.pas:38` `(cheatengine.org)`, `frmautoinjectunit.pas:159`, `custombase85.pas:8`, etc. | **Keep** — these are credit lines, not branding |

### Step 3.1 — Generate the audit list

```
grep -rn "'Cheat Engine'" "Cheat Engine" --include='*.pas' --include='*.lpr' \
  | grep -v 'tcclib/' | grep -v '/Tutorial/' | grep -v '/lua/lua.pas' \
  > phase2-strcheatengine-audit.txt
```

Expected scope: ~30–60 hits across ~20 files. Most resolve to either:
- Replace `'Cheat Engine'` → `strCheatEngine` (when the unit `uses MainUnit2`)
- Replace `'Cheat Engine'` → `'Reverie'` literal (when adding a `MainUnit2` import is overkill)

### Step 3.2 — Special-case files known to bypass the constant

Verify and fix these specifically (each should already use `strCheatEngine` after Step 1, but double-check):

- `cheatengine.lpr` L293 (Application.Title) — done in Step 2.1
- `cheatengine.lpr` L416 (OutputDebugString) — done in Step 2.1
- `dbk32/DBK32functions.pas` — uses `strCheatEngine` in most error strings (Phase 1 already cleaned this up); re-verify
- `LuaHandler.pas` — search for any leftover literal `'Cheat Engine'` user-agent or window title strings

### Step 3.3 — Compile, verify, commit

```
git commit -m "replace stray hardcoded 'Cheat Engine' literals with strCheatEngine / Reverie"
```

---

## Step 4: Rename named pipes (`CEWINHOOK` → `RVWINHOOK`)

**Goal:** Update the IPC pipe prefix used by the Win API hook system. Two units, four call sites total.

**File:** `Cheat Engine\winhook\com.pas`

**Line 53**:
```diff
- pipe:=CreateNamedPipe(pchar('\\.\pipe\CEWINHOOKC'+inttostr(GetProcessID)), PIPE_ACCESS_DUPLEX, PIPE_TYPE_BYTE or PIPE_READMODE_BYTE or PIPE_WAIT, 255, 16, 8192, 0, nil);
+ pipe:=CreateNamedPipe(pchar('\\.\pipe\RVWINHOOKC'+inttostr(GetProcessID)), PIPE_ACCESS_DUPLEX, PIPE_TYPE_BYTE or PIPE_READMODE_BYTE or PIPE_WAIT, 255, 16, 8192, 0, nil);
```

**Line 124**:
```diff
- pipename:='CEWINHOOK'+inttostr(GetProcessID);
+ pipename:='RVWINHOOK'+inttostr(GetProcessID);
```

**File:** `Cheat Engine\LuaHandler.pas`

**Line 13297**:
```diff
- ts:='CEWINHOOK'+inttostr(processid);
+ ts:='RVWINHOOK'+inttostr(processid);
```

**Lines 13323 and 13389**:
```diff
- pc:=TLuaPipeClient.create('CEWINHOOKC'+inttostr(processid));
+ pc:=TLuaPipeClient.create('RVWINHOOKC'+inttostr(processid));
```

> [!IMPORTANT]
> All five call sites must update **in the same commit**. The Pascal side (`LuaHandler.pas`) and the injected DLL side (`winhook/com.pas`, which compiles into the speedhack/winhook DLL) must agree on the pipe name or runtime injection breaks silently.

### Step 4.1 — Verify the speedhack/winhook DLL needs a rebuild

If `winhook/com.pas` is compiled into a separate `.dll`, that DLL also needs to be rebuilt and dropped into `bin/`. Check:

```
grep -l "winhook/com.pas" "Cheat Engine"/winhook/winhook.lpi
grep -l "winhook/com.pas" "Cheat Engine"/speedhack/speedhack.lpi
```

Build the affected sub-projects:
```
lazbuild "Cheat Engine\winhook\winhook.lpi"
lazbuild "Cheat Engine\speedhack\speedhack.lpi"
```

### Step 4.2 — Compile, verify, commit

```
git commit -m "rename winhook pipes CEWINHOOK -> RVWINHOOK"
```

---

## Step 5: Rename `GetCEDir` and `CheatEngineDir` (optional — see notes)

**Goal:** The function `GetCEDir` and the global variable `CheatEngineDir` are referenced in 27 files. Decide whether to rename now or defer.

**Recommendation: DEFER to Phase 3.**

Rationale:
- These are *internal* identifiers, not user-visible strings
- Renaming touches 27 files for zero behavioral or branding benefit (no user ever sees the function name)
- The Phase 3 C++ migration will replace this code path entirely with `libreverie::get_app_dir()` or similar
- Renaming now adds 27 file edits to Phase 2's diff with no functional gain

**If you choose to rename anyway**, the mechanical edit is straightforward:

```bash
# Pascal-aware sed (case-insensitive identifier rewrite)
# Run from "Cheat Engine/" root, on .pas/.lpr/.lfm only
find . -type f \( -name '*.pas' -o -name '*.lpr' \) \
  -not -path './Tutorial/*' -not -path './tcclib/*' -not -path './backup/*' \
  -exec sed -i 's/\bGetCEDir\b/GetReverieDir/g; s/\bCheatEngineDir\b/ReverieDir/g' {} +
```

Then verify the function declaration site in `CEFuncProc.pas` is also renamed (lines 94 and 2681), and the global `CheatEngineDir: String;` in `globals.pas` line 54.

> [!WARNING]
> Sed is unsafe across `.dfm` files (some have UTF-16 encoding). Restrict the find to `.pas` and `.lpr` only. The Direct X mess C++ code at `d3dhookshared.h`, `DXHookBase.cpp`, `CED3D11Hook.cpp`, `CED3D10Hook.cpp` also references `cheatenginedir` — those must be updated by hand.

### Step 5.1 — Decision and commit

If deferring: skip this step entirely. Move to Step 6.
If renaming: commit as `rename GetCEDir/CheatEngineDir to GetReverieDir/ReverieDir`.

---

## Step 6: Clean vendored code attribution and dev paths

### Step 6.1 — `tcclib/` vendored TinyCC modifications

25 comments across 5 files inside `Cheat Engine\tcclib\` say `//Cheat Engine modification`. These mark divergence from upstream TinyCC.

**Files:**
- `tcclib/libtcc.c` — 10 occurrences
- `tcclib/tcc.h` — 6 occurrences
- `tcclib/tccrun.c` — 5 occurrences
- `tcclib/tccelf.c` — 2 occurrences
- `tcclib/tccpe.c` — 2 occurrences

**Bulk rewrite (run from repo root):**

```bash
sed -i 's|//Cheat Engine modification|//Reverie modification|g' \
  "Cheat Engine"/tcclib/libtcc.c \
  "Cheat Engine"/tcclib/tcc.h \
  "Cheat Engine"/tcclib/tccrun.c \
  "Cheat Engine"/tcclib/tccelf.c \
  "Cheat Engine"/tcclib/tccpe.c
```

> [!NOTE]
> Don't touch any `//cheat engine fuckery` comments — Phase 1 already cleaned those. Don't touch upstream TinyCC code outside of these `// modification` markers.

### Step 6.2 — Hardcoded dev paths in `tcclib.pas` and `csharpcompiler.pas`

These are inside `{$ifdef standalonetest}` blocks (only active when manually testing the units in isolation). They reference `D:\git\cheat-engine\Cheat Engine\bin\` — a path that exists only on the original CE author's machine.

**File:** `Cheat Engine\tcclib.pas`

**Lines 1233, 1240–1243, 1467** — replace `D:\git\cheat-engine\Cheat Engine\bin\` with a relative-to-repo placeholder. Since `standalonetest` is dev-only and never enabled in normal builds, the cheapest fix is to delete the `{$ifdef standalonetest}` blocks entirely:

```diff
- module:=LoadLibrary({$ifdef standalonetest}'D:\git\cheat-engine\Cheat Engine\bin\'+{$endif}'tcc32-32.dll');
+ module:=LoadLibrary('tcc32-32.dll');
```

Apply the same pattern to lines 1240–1243 and 1467.

**File:** `Cheat Engine\csharpcompiler.pas`

**Lines 234 and 236** — same treatment:
```diff
- r:=DotNetExecuteClassMethod({$ifdef standalonetest}'D:\git\cheat-engine\Cheat Engine\bin\CSCompiler.dll'{$else}CheatEngineDir+'CSCompiler.dll'{$endif},'CSCompiler','Compiler','NewCompiler',inttostr(ptruint(@delegates)));
+ r:=DotNetExecuteClassMethod(CheatEngineDir+'CSCompiler.dll','CSCompiler','Compiler','NewCompiler',inttostr(ptruint(@delegates)));
```

### Step 6.3 — `DotNetInvasiveDataCollector.csproj`

**File:** `Cheat Engine\DotNetInvasiveDataCollector\DotNetInvasiveDataCollector\DotNetInterface.csproj`

**Lines 14 and 18** — `<OutputPath>D:\git\cheat-engine\Cheat Engine\bin\autorun\dlls\</OutputPath>` is hardcoded to the original author's machine. Replace with a relative path:

```diff
- <OutputPath>D:\git\cheat-engine\Cheat Engine\bin\autorun\dlls\</OutputPath>
+ <OutputPath>..\..\bin\autorun\dlls\</OutputPath>
```

### Step 6.4 — Compile, verify, commit

```
lazbuild --build-mode="Release 64-Bit" "Cheat Engine\cheatengine.lpi"
git commit -m "clean tcclib attribution and remove hardcoded dev paths"
```

---

## Step 7: Rebrand `.lfm` form captions

**Goal:** ~199 `.lfm` form files contain "Cheat Engine" in window titles, button captions, group captions, hint text, and About-dialog text. These are the user-visible text on every UI surface.

**Scope:** Bulk-rewrite, then audit.

### Step 7.1 — Bulk caption replacement

The safe substitutions (preserving capitalization patterns):

```bash
# From repo root
find "Cheat Engine" -name '*.lfm' \
  -not -path '*/Tutorial/*' -not -path '*/backup/*' \
  -exec sed -i \
    -e "s/'Cheat Engine'/'Reverie'/g" \
    -e "s/'Cheat Engine /'Reverie /g" \
    -e "s/ Cheat Engine'/ Reverie'/g" \
    -e "s/Caption = 'Cheat Engine\\(.*\\)'/Caption = 'Reverie\\1'/g" \
    {} +
```

> [!WARNING]
> `.lfm` files are **mixed-encoding**. Most are UTF-8, but some embed binary `Picture.Data` blocks. `sed -i` is safe for caption-only edits because the binary blocks contain no `'Cheat Engine'` substring, but **always run a build immediately after** to catch any corrupted form files. If a build fails with `EReadError: Stream read error`, restore from git and switch to a Python script that processes only text caption nodes.

### Step 7.2 — Audit known special cases

**File:** `Cheat Engine\aboutunit.lfm`

The about dialog has hint text and "Made by: Dark Byte" credits. Inspect:
```
grep -n 'Cheat Engine\|Made by' "Cheat Engine\aboutunit.lfm"
```

Decision points:
- "Made by: Dark Byte" — keep (original CE author credit)
- "Cheat Engine" in `Caption` properties — replace with "Reverie"
- About dialog version label — already driven by `cenamewithversion` in `aboutunit.pas`

**File:** `Cheat Engine\MainUnit.lfm`

The main window is the highest-visibility surface:
- Window caption (`Caption = 'Cheat Engine 7.5'`) — replace with `'Reverie 7.5.1'` *or* leave blank and let `MainUnit.pas:initcetitle` set it dynamically
- Menu items containing "Cheat Engine"

### Step 7.3 — Rebuild and visual smoke test

```
lazbuild --build-mode="Release 64-Bit" "Cheat Engine\cheatengine.lpi"
```

**Run the executable and visually verify:**
- Main window title bar
- Help → About dialog
- Settings dialog tab captions
- Right-click context menus
- Open one .CT file and verify the table editor opens cleanly (catches form-load errors from corrupted .lfm)

### Step 7.4 — Commit

```
git commit -m "rebrand .lfm form captions to Reverie"
```

---

## Step 8: Localization files (`.lrt` and `.po`)

**Goal:** Update the 108 `.lrt` resource translation tables and 6 `cheatengine-x86_64.po` language files to use the new strings.

### Step 8.1 — `.lrt` regeneration

`.lrt` files are auto-generated by Lazarus from `resourcestring` declarations. They will be regenerated automatically on the next build. **Do not hand-edit them.** After Step 1 lands, the next `lazbuild` will overwrite all 108 `.lrt` files with the new constant values.

Verify:
```
grep -l "'Cheat Engine'" "Cheat Engine"/*.lrt | wc -l
```
Should return 0 after a clean rebuild.

### Step 8.2 — `.po` translation file scope

**Files affected (6 languages):**
- `bin/languages/de_DE/cheatengine-x86_64.po`
- `bin/languages/fr_FR/cheatengine-x86_64.po`
- `bin/languages/ru_RU/cheatengine-x86_64.po`
- `bin/languages/zh_CN/cheatengine-x86_64.po`
- `bin/languages/zh_CN/cheatengine.po`
- `bin/languages/zh_TW/cheatengine-x86_64.po`

**Strategy:** Stub-translate. Replace any `msgstr` value that is a translation of "Cheat Engine" (in the target language) with the literal `"Reverie"`. Do not attempt full re-translation of the thousands of other strings.

**File renames:** All `cheatengine-x86_64.po` files should be renamed to `reverie-x86_64.po`. **But:** the loader at `cetranslator.pas:105` looks for `cheatengine-x86_64.po`. Decision:

- **Option A (chosen):** Rename `.po` files to `reverie-x86_64.po` AND update `cetranslator.pas:105` to look for `reverie-x86_64.po`
- **Option B:** Keep the `cheatengine-x86_64.po` filename as a "schema name" (defer rename to Step 9 with the binary)

This plan picks **Option A** because the `.po` files are loaded at runtime by exact filename match — defer means we re-touch this code in Step 9.

### Step 8.3 — Rename `.po` files and update the loader

```bash
cd "Cheat Engine/bin/languages"
for d in de_DE fr_FR ru_RU zh_CN zh_TW; do
  if [ -f "$d/cheatengine-x86_64.po" ]; then
    git mv "$d/cheatengine-x86_64.po" "$d/reverie-x86_64.po"
  fi
done
git mv zh_CN/cheatengine.po zh_CN/reverie.po
```

**File:** `Cheat Engine\cetranslator.pas`

**Lines 102, 105, 108** — update the filename probe sequence:
```diff
-     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'cheatengine'+LCEXT;
+     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'reverie'+LCEXT;
      if FileExists(Result) then exit;

-     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'cheatengine-x86_64'+LCEXT;
+     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'reverie-x86_64'+LCEXT;
      if FileExists(Result) then exit;

-     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'cheatengine-i386'+LCEXT;
+     Result := cheatenginedir + {$ifdef Darwin}'../'+{$endif}'Languages' + DirectorySeparator + LangID + DirectorySeparator + 'reverie-i386'+LCEXT;
```

### Step 8.4 — Stub-translate "Cheat Engine" strings inside each `.po`

For each `cheatengine-x86_64.po` (now `reverie-x86_64.po`), find `msgstr` values translating the source string `"Cheat Engine"` and replace with `"Reverie"`. The msgid lines stay as `msgid "Cheat Engine"` because gettext keys must match the `.pas` source — but Step 1 changed the constant value, which means the next `lazbuild` will rewrite the `.lrt` files and the gettext keys. **Re-extract `.po` files from the current source after Step 1 is committed**, using:

```
lazbuild --lazarusdir=C:\lazarus --build-mode="Release 64-Bit" --aslink-resources "Cheat Engine\cheatengine.lpi"
```

(or the IDE Project → Tools → Update PO Files action). Then hand-translate or stub the new "Reverie" entries to the local language equivalent. For Phase 2, **stub with the literal "Reverie"** in every language is acceptable — this is a known interim state and tracked in `RoadmapToReverie.md`.

### Step 8.5 — Commit

```
git commit -m "rename .po files to reverie-*, stub-translate identity strings"
```

---

## Step 9: Rename project files, binaries, and the `Cheat Engine/` directory

**Goal:** Final mechanical rename. Last because every prior step touches the directory and renaming earlier means re-rebasing every commit.

> [!CAUTION]
> This step is **physically destructive** to git history navigation. After this commit, `git log -- "Cheat Engine/MainUnit.pas"` will not show pre-Phase-2 history without `--follow`. Plan accordingly.

### Step 9.1 — Rename project files

```bash
cd "Cheat Engine"

# Main project
git mv cheatengine.lpi reverie.lpi
git mv cheatengine.lpr reverie.lpr
git mv cheatengine.ico reverie.ico
git mv cheatengine.res reverie.res
git mv cheatengine.lps reverie.lps  # (optional, IDE session file)

# Sub-projects
cd launcher && git mv cheatengine.lpi reverie-launcher.lpi && git mv cheatengine.lpr reverie-launcher.lpr && cd ..
```

Then update the inside of each renamed `.lpr`:

**File:** `Cheat Engine\reverie.lpr` (was `cheatengine.lpr`)

**Line 1:**
```diff
- program cheatengine;
+ program reverie;
```

**Line 128 (or wherever `{$R cheatengine.res}` appears):**
```diff
- {$R cheatengine.res}
+ {$R reverie.res}
```

**File:** `Cheat Engine\reverie.lpi` (was `cheatengine.lpi`)

Update `<MainUnit Value="...">` and any `<UnitN><Filename>` referencing `cheatengine.lpr` to `reverie.lpr`.

### Step 9.2 — Update binary output filenames in `.lpi`

**File:** `Cheat Engine\reverie.lpi`

In **every** `<BuildMode>` block (Item1 = Release 64-Bit, Item2 = Release 32-Bit, Item3 = Release 64-Bit O4 AVX2, etc.):

```diff
- <Filename Value="bin\cheatengine-$(TargetCPU)"/>
+ <Filename Value="bin\reverie-$(TargetCPU)"/>
```

```diff
- <Filename Value="bin\cheatengine-i386"/>
+ <Filename Value="bin\reverie-i386"/>
```

```diff
- <Filename Value="bin\cheatengine-$(TargetCPU)-SSE4-AVX2"/>
+ <Filename Value="bin\reverie-$(TargetCPU)-SSE4-AVX2"/>
```

### Step 9.3 — Update launcher's basename

**File:** `Cheat Engine\launcher\reverie-launcher.lpr` (was `cheatengine.lpr`)

**Line 50 (after Step 2.7 collapsed altname):**
```diff
- basename:='cheatengine';
+ basename:='reverie';
```

### Step 9.4 — Rename the top-level directory

```bash
git mv "Cheat Engine" "Reverie"
```

Then sweep all references to the old path:
```bash
grep -rln '"Cheat Engine\\\|"Cheat Engine/' . --include='*.yml' --include='*.yaml' --include='*.md' --include='*.lpi' --include='*.lpr' --include='*.bat' --include='*.sh' --include='*.cfg'
```

Update each match. The known sites:
- `appveyor.yml` (if not yet deleted per Roadmap §4.3)
- `README.md` build instructions (Phase 1 already updated lazbuild path, may still reference `Cheat Engine/`)
- `READMEes.md` build instructions
- Any `.bat`/`.sh` build helper scripts in `lua53/`, `speedhack/`, etc.

### Step 9.5 — Update `MyDocuments` folder name

The new "My Reverie Tables" folder is set by `strMyCheatTables` in Step 1. Verify it works at runtime — `CEFuncProc.GetCEDir` (or `GetReverieDir` if Step 5 was applied) constructs the path using the constant.

### Step 9.6 — Driver `.inf` `ManufacturerName` strings

**File:** `DBKKernel\DBK32.inf` line 75
**File:** `DBKKernel\DBK64.inf` line 75
**File:** `DBKKernel\ultimap2\ultimap2-64.inf` line 78

```diff
- ManufacturerName="Cheat Engine" ;TODO: Replace with your manufacturer name
+ ManufacturerName="Reverie"
```

> [!NOTE]
> The driver **filenames** (`DBK32.sys`, `DBK64.sys`) are NOT renamed in Phase 2 — that requires re-signing the driver and is deferred to Phase 3 along with the `DBK32`/`DBK64` → `Reverie32`/`Reverie64` rename.

### Step 9.7 — Final build verification

```
lazbuild --build-mode="Release 64-Bit" "Reverie\reverie.lpi"
lazbuild --build-mode="Release 32-Bit" "Reverie\reverie.lpi"
lazbuild "Reverie\speedhack\speedhack.lpi"
lazbuild "Reverie\VEHDebug\vehdebug.lpi"
lazbuild "Reverie\luaclient\luaclient.lpi"
lazbuild "Reverie\winhook\winhook.lpi"
```

Confirm `Reverie\bin\reverie-x86_64.exe` exists, runs, and shows "Reverie 7.5.1" in the title bar.

### Step 9.8 — Tag the milestone

```
git commit -m "rename project files, binaries, and Cheat Engine -> Reverie directory"
git tag phase2-rebrand
```

---

## Summary of Files Modified

| Step | Files Modified | Files Renamed / Deleted |
|---|---|---|
| **1** (Constants) | `MainUnit2.pas` (1 file) | — |
| **2** (Collapse altname) | `cheatengine.lpr`, `MainUnit.pas`, `cetranslator.pas`, `first.pas`, `formsettingsunit.pas`, `LuaHandler.pas`, `LuaInternet.pas`, `ceregreset/ceregreset.dpr`, `launcher/cheatengine.lpr` (9 files) | — |
| **3** (Stray literals) | ~20–60 files (audit-driven) | — |
| **4** (Pipes) | `winhook/com.pas`, `LuaHandler.pas` (2 files) | — |
| **5** (GetCEDir) | **DEFERRED to Phase 3** | — |
| **6** (Vendored code + dev paths) | 5 `tcclib/*` files, `tcclib.pas`, `csharpcompiler.pas`, `DotNetInterface.csproj` (8 files) | — |
| **7** (LFM captions) | ~199 `.lfm` files (bulk sed) | — |
| **8** (Localization) | `cetranslator.pas`, 6 `.po` files | 6 `cheatengine-*.po` → `reverie-*.po` |
| **9** (Project rename) | `reverie.lpi`, `reverie.lpr`, `launcher/reverie-launcher.lpr`, 3 `DBKKernel/*.inf`, `README.md`, `READMEes.md` | `cheatengine.{lpi,lpr,ico,res,lps}` → `reverie.*`, `Cheat Engine/` → `Reverie/` |

## Risk Assessment

| Risk | Mitigation |
|---|---|
| **Step 1 cascades a wrong default into 30+ files at once** | Verify by running the exe immediately. If the title shows "Reverie", registry under `\Software\Reverie\` is created, and the table editor opens cleanly, the cascade is correct. Rollback is `git revert` of the single Step 1 commit. |
| **Existing CE users lose their settings on first run** | This is the **intended** clean-break behavior per Roadmap §1.3. Document this in the next release notes. Do not add a migration shim. |
| **`.lfm` bulk sed corrupts a form with embedded UTF-16 or binary data** | Build immediately after Step 7.1. If it fails, `git checkout HEAD -- "Cheat Engine"/*.lfm` and switch to a Python script that processes only `Caption =`/`Hint =`/`Lines.Strings = (` lines. |
| **`first.pas` runs before `MainUnit2.pas` is initialized** | Hardcode `'Reverie'` in `first.pas` Step 2.4 — do **not** try to use `strCheatEngine` there (circular dependency). |
| **Renaming `cheatengine.lpr` breaks the 32-bit launcher's lookup** | Step 2.7 explicitly defers the launcher's `basename` change to Step 9.3 so the binary is renamed and the lookup is updated in the same commit. |
| **`tcclib/` bulk sed accidentally edits upstream TinyCC code** | The sed filter targets `//Cheat Engine modification` exactly. Upstream TinyCC has no such marker. Verify with `git diff --stat "Cheat Engine"/tcclib/` after Step 6.1 — should show only comment-line changes. |
| **Pipe rename breaks runtime injection because the speedhack DLL is stale** | Step 4.1 explicitly rebuilds `winhook.lpi` and `speedhack.lpi`. Smoke-test by attaching to a 32-bit process and triggering a Lua hook. |
| **`.po` regeneration overwrites hand-edited stub translations** | Stub-translate **after** the rebuild that updates `.lrt` files. Document that the `.po` files are intentionally English-stubbed for Phase 2 and contributors are welcome to fill them in. |
| **Driver INF rename triggers a Windows driver re-installation prompt** | Updating `ManufacturerName` in the INF does not invalidate the driver signature unless the driver is re-built. Phase 2 does **not** rebuild the driver. The new `ManufacturerName` appears in Device Manager only after the user reinstalls. |
| **Step 9 directory rename breaks IDE bookmarks, CI configs, and external tooling** | This is unavoidable. Mitigate by doing it last so there's only one breaking change for downstream consumers. Tag `phase2-rebrand` immediately after the directory rename so contributors can compare against a stable baseline. |

## Verification Plan

After **each step**, run:
```
lazbuild --build-mode="Release 64-Bit" "Cheat Engine\cheatengine.lpi"
```
(after Step 9, the path becomes `"Reverie\reverie.lpi"`).

After **Step 1, 2, 4, 7, 9** specifically, also run a smoke test:
- Launch the executable
- Verify window title shows "Reverie"
- Open `regedit.exe` and confirm `HKCU\Software\Reverie` exists
- Open and re-save a `.CT` file (backward compatibility check)
- Open Help → About and verify the version label
- For Step 4: attach to a test process and trigger a Lua hook (verifies the renamed pipe IPC works end-to-end)
- For Step 7: open every form at least once (Settings, Trainer Generator, Memory View, Pointer Scanner, Structure Dissector) to catch any corrupted `.lfm`

Final acceptance after Step 9:
- `Reverie\bin\reverie-x86_64.exe` exists and runs
- `grep -rn "'Cheat Engine'" Reverie --include='*.pas' --include='*.lpr'` returns only attribution comments
- `grep -rn 'altname' Reverie` returns zero hits
- `grep -rn 'CEWINHOOK' Reverie` returns zero hits
- `grep -rn 'D:\\git\\cheat-engine' .` returns zero hits
- Registry under `HKCU\Software\Cheat Engine\` is **untouched** by Reverie (verify)
- Registry under `HKCU\Software\Reverie\` is created on first run

## Decisions (Resolved)

| Question | Decision |
|---|---|
| **`{$ifdef altname}`** | **Delete entirely** — Reverie is the only identity. |
| **Registry migration** | **None** — clean break. |
| **`.CT` file extension and XML schema** | **Keep** — backward compatibility with existing tables. |
| **`.CETRAINER` / `.CESIG` extensions** | **Defer** to Phase 3. |
| **Driver names (`DBK32`/`DBK64`)** | **Defer** filename rename to Phase 3. Update only `ManufacturerName` strings now. |
| **`GetCEDir` / `CheatEngineDir` identifier rename** | **Defer** to Phase 3 (or skip entirely — the C++ port replaces this anyway). |
| **`.po` localization** | **Stub-translate** "Cheat Engine" → "Reverie" only. Defer full re-translation indefinitely. |
| **`Cheat Engine/` directory rename** | **Last step** of Phase 2. |
| **Binary filename (`cheatengine-x86_64.exe`)** | **Rename in Step 9** alongside the directory. |

---

## After Phase 2

Phase 2 leaves the codebase as a **fully-rebranded Reverie identity** with:
- Single source of truth in `MainUnit2.pas` constants
- Zero `{$ifdef altname}` machinery
- Reverie registry, pipes, temp dirs, file folders
- Reverie binary names, project files, and directory layout
- Reverie driver `ManufacturerName` (driver filenames unchanged pending Phase 3)
- English-stubbed localization (full translation deferred)

Phase 3 (C++ Migration — see `RoadmapToReverie.md` §3) starts from the `phase2-rebrand` tag with no Pascal-era branding noise to wade through.
