# Phase 1: De-bloat & Clean Slate — Implementation Plan

## Overview

Phase 1 removes monetization, adware, legacy OS code, and unprofessional naming. Every step is ordered so the project **compiles after each commit**. Total: ~25 file edits, 4 file deletions, 1 file rename.

> [!IMPORTANT]
> **Prerequisite:** Phase 0 (baseline build verification) must be complete. You must have a working `lazbuild` compilation and a `git tag baseline-builds` to revert to if needed.

---

## Step 1: Delete Monetization & Ad-Serving Units

**Goal:** Remove the 4 units that form the monetization ecosystem, and clean all references so the project compiles.

**Order matters:** Remove references first, then delete the files. Work from leaf → root (units that depend on others first).

### Step 1.1 — Remove `cheatecoins` from `MemoryRecordUnit.pas`

**File:** `Cheat Engine\MemoryRecordUnit.pas` (line 464)

```diff
- processhandlerunit, Parsers, {$ifdef windows}winsapi,{$endif}autoassembler, globals{$ifdef windows}, cheatecoins{$endif};
+ processhandlerunit, Parsers, {$ifdef windows}winsapi,{$endif}autoassembler, globals;
```

> [!NOTE]
> The `cheatecoins` unit is only imported here — verify no procedure calls from this unit exist in the file body. If `cheatecoins` exports are used, those call sites must also be removed.

### Step 1.2 — Remove `cheatecoins` from `MainUnit.pas` and delete the `EnableCheatECoinSystem` call

**File:** `Cheat Engine\MainUnit.pas`

Two changes:

**Line 1126** — uses clause:
```diff
- ,frmFoundlistPreferencesUnit, fontSaveLoadRegistry{$ifdef windows}, cheatecoins{$endif},strutils, iptlogdisplay,
+ ,frmFoundlistPreferencesUnit, fontSaveLoadRegistry,strutils, iptlogdisplay,
```

**Line 8589** — delete the call:
```diff
-     EnableCheatECoinSystem;
```

### Step 1.3 — Remove `cesupport` from `MainUnit.pas`

**File:** `Cheat Engine\MainUnit.pas` (line 41)

```diff
- ceguicomponents, frmautoinjectunit, cesupport, trainergenerator, genericHotkey,
+ ceguicomponents, frmautoinjectunit, trainergenerator, genericHotkey,
```

Search for any calls to `TADWindow` or `adwindow` in `MainUnit.pas` and remove them.

### Step 1.4 — Remove `cesupport` from `LuaHandler.pas`

**File:** `Cheat Engine\LuaHandler.pas` (line 111)

```diff
- cesupport, DBK32functions, sharedMemory, disassemblerComments, disassembler,
+ DBK32functions, sharedMemory, disassemblerComments, disassembler,
```

Search for any calls to `TADWindow` or `adwindow` in `LuaHandler.pas` and remove them.

### Step 1.5 — Gut ad support from `trainergenerator.pas`

**File:** `Cheat Engine\trainergenerator.pas`

This is the most invasive change. The trainer generator has deep ad integration:

**Line 18** — uses clause: remove `frmAdConfigUnit, cesupport`
```diff
- frmAdConfigUnit, cesupport, IconStuff, memoryrecordunit, frmSelectionlistunit,
+ IconStuff, memoryrecordunit, frmSelectionlistunit,
```

**Line 137** — private field: remove `adconfig`
```diff
-     adconfig: TfrmAdConfig;
```

**Delete the `cbSupportCheatEngine` checkbox entirely:**
- Delete `cbSupportCheatEngine: TCheckBox;` from the class declaration
- Delete `cbSupportCheatEngineChange` procedure declaration (line 111)
- Delete entire `cbSupportCheatEngineChange` procedure body (lines 1868–1930) — the "guilt procedure" that shows ads
- Delete `RestoreSupportCE` procedure declaration and body (lines 1861–1866)
- Delete `restoretimer: ttimer;` field declaration
- Delete the `cbSupportCheatEngine` component from `trainergenerator.lfm` (search for `cbSupportCheatEngine` and remove the entire object block)
- Remove the resource string `rsDonTSupportCheatEngineOrYourself` (line 215–216)
- Remove `rsThankYou` and `rsAaaaw` resource strings (lines 217–218)

**Line 753** — `FormClose`: remove the forced re-check:
```diff
-   if not cbSupportCheatEngine.checked then
-     cbSupportCheatEngine.checked:=true;
```

> [!WARNING]
> Both `trainergenerator.pas` AND `trainergenerator.lfm` must be edited in sync. If you remove the checkbox from Pascal but leave it in the `.lfm`, Lazarus will crash on form load.

### Step 1.6 — Remove from `cheatengine.lpr`

**File:** `Cheat Engine\cheatengine.lpr`

**Line 60** — remove `cesupport`:
```diff
- LuaSyntax, cesupport, trainergenerator, genericHotkey,
+ LuaSyntax, trainergenerator, genericHotkey,
```

**Line 62** — remove `frmAdConfigUnit`:
```diff
- ExtraTrainerComponents, frmAdConfigUnit, IconStuff, cetranslator,
+ ExtraTrainerComponents, IconStuff, cetranslator,
```

**Line 117** — remove `cheatecoins`:
```diff
- LuaHeaderSections, frmDebuggerAttachTimeoutUnit, cheatecoins,
+ LuaHeaderSections, frmDebuggerAttachTimeoutUnit,
```

**Line 118** — remove `frmMicrotransactionsUnit`:
```diff
- frmMicrotransactionsUnit, frmSyntaxHighlighterEditor, LuaCustomImageList,
+ frmSyntaxHighlighterEditor, LuaCustomImageList,
```

### Step 1.7 — Delete the files

After all references are removed:

```
git rm "Cheat Engine/cheatecoins.pas"
git rm "Cheat Engine/frmmicrotransactionsunit.pas"
git rm "Cheat Engine/frmmicrotransactionsunit.lfm"
git rm "Cheat Engine/frmAdConfigUnit.pas"
git rm "Cheat Engine/frmAdConfigUnit.lfm"
git rm "Cheat Engine/frmAdConfigUnit.lrt"
git rm "Cheat Engine/cesupport.pas"
```

### Step 1.8 — Compile & verify

```
lazbuild --build-mode="Debug Win64" "Cheat Engine\cheatengine.lpi"
```

If it compiles, commit:
```
git commit -m "phase1: remove monetization, ad-serving, and microtransaction units"
```

---

## Step 2: Remove External Phone-Home URLs

**Goal:** Eliminate all hardcoded URLs pointing to `cheatengine.org`, `patreon.com`, and `forum.cheatengine.org`.

### Step 2.1 — `aboutunit.pas`

| Line | Current | Action |
|---|---|---|
| L159 | `shellexecute(0,'open','https://www.patreon.com/cheatengine'...)` | Delete the entire `Label4Click` procedure body (leave empty begin/end) |
| L164 | `ShellExecute(0,'open','https://cheatengine.org/'...)` | Delete body of `Label8Click` |
| L169 | `ShellExecute(0,'open','http://forum.cheatengine.org/'...)` | Delete body of `Label9Click` |

Also remove the `{$ifdef altname}` block in `FormShow` (lines 141–151) — Reverie is the only identity.

### Step 2.2 — `MainUnit.pas`

| Line | Current | Action |
|---|---|---|
| L5902 | `wikipath:='https://wiki.cheatengine.org/index.php'` | Replace with `''` or a Reverie docs URL placeholder |
| L7300 | `s:=format('http://www.cheatengine.org/?referredby=CE%.2f',[ceversion])` | Delete the referral block |
| L11140 | `ShellExecute(0,'open','https://wiki.cheatengine.org/index.php'...)` | Delete or replace |

### Step 2.3 — `DBK32functions.pas`

| Line | Current | Action |
|---|---|---|
| L3417 | `shellexecute(0, 'open', 'https://cheatengine.org/dbkerror.php'...)` | Delete or replace with a local error message |

### Step 2.4 — Strip Tutorial module entirely

The Tutorial module contains hardcoded `cheatengine.org` URLs throughout and is tightly coupled to the CE identity. **Delete it entirely.**

```
git rm -r "Cheat Engine/Tutorial/"
```

Then remove any references to Tutorial units from the main project:
- Search `cheatengine.lpr` for Tutorial-related units in the `uses` clause and remove them
- Search `cheatengine.lpi` for Tutorial source file entries and remove them
- Verify no other unit imports from `Tutorial/`

### Step 2.5 — `cesupport.pas` — already deleted in Step 1

### Step 2.6 — `.github/FUNDING.yml`

```
git rm .github/FUNDING.yml
```

Or replace its contents with Reverie funding info.

### Step 2.7 — `README.md` and `READMEes.md`

Remove Patreon links and any `cheatengine.org` references.

### Step 2.8 — Compile, verify, commit

```
git commit -m "phase1: remove all cheatengine.org phone-home URLs and Patreon links"
```

---

## Step 3: Strip Legacy OS Code

**Goal:** Remove Win98/2K/XP dead code. Target: Windows 10 minimum.

### Step 3.1 — `globals.pas`

**Line 144** — remove the variable declaration:
```diff
-   iswin2kplus: boolean;
```

Then grep for all uses of `iswin2kplus` in the codebase. Every `if iswin2kplus then` becomes unconditionally true. Every `if not iswin2kplus` becomes dead and should be deleted.

### Step 3.2 — `CEFuncProc.pas`

**Lines 1534–1590** — remove the `cOsWin98`, `cOsWin98SE` constants and the `GetSystemType` function (or the Win98/ME detection branches within it).

**Line 4008** — remove `iswin2kplus:=GetSystemType>=5;`

### Step 3.3 — `debugeventhandler.pas`

**Lines 860, 880** — remove XP-specific RF flag workaround blocks.

### Step 3.4 — `first.pas`

**Line 42** — remove the Win7 DPI fallback (`User32.dll` → `SetProcessDPIAware` path). Keep only the `Shcore.dll` → `SetProcessDpiAwareness` path (Win8.1+), or use the Win10 `SetProcessDpiAwarenessContext` directly.

### Step 3.5 — `MainUnit2.pas`

**Line 76** — remove `strNeedNewerWindowsVersion` or update it to reference Windows 10.

### Step 3.6 — `feces.pas` (will become `tablesignature.pas` in Step 4)

**Lines 90–103** — remove `EncodePointerNI` / `DecodePointerNI` functions (XP fallbacks for missing `EncodePointer`).

**Lines 782–790** — simplify initialization:
```diff
- var k32: HMODULE;
- initialization
-   k32:=GetModuleHandle('kernel32.dll');
-   pointer(encodepointer):=GetProcAddress(k32,'EncodePointer');
-   pointer(decodepointer):=GetProcAddress(k32,'DecodePointer');
-   if not assigned(encodepointer) then
-     (encodepointer):=@EncodePointerNI;
-   if not assigned(decodepointer) then
-     (decodepointer):=@DecodePointerNI;
+ initialization
+   pointer(encodepointer):=GetProcAddress(GetModuleHandle('kernel32.dll'),'EncodePointer');
+   pointer(decodepointer):=GetProcAddress(GetModuleHandle('kernel32.dll'),'DecodePointer');
```

### Step 3.7 — Compile, verify, commit

```
git commit -m "phase1: strip Win98/2K/XP legacy code, target Windows 10 minimum"
```

---

## Step 4: Clean Profane Identifiers & Rename `feces.pas`

**Goal:** Professional naming throughout.

### Step 4.1 — `cheatengine.lpr`: Rename `TFormFucker`

5 occurrences across lines 262, 268, 286, 363, 381:

```diff
- type TFormFucker=class
+ type TFormFontApplier=class
```
```diff
- procedure TFormFucker.addFormEvent(Sender: TObject; Form: TCustomForm);
+ procedure TFormFontApplier.addFormEvent(Sender: TObject; Form: TCustomForm);
```
```diff
-   ff: TFormFucker;
+   ff: TFormFontApplier;
```
```diff
-             ff:=TFormFucker.Create;
+             ff:=TFormFontApplier.Create;
```
(both at L363 and L381)

**Line 270** — clean the comment:
```diff
-   //fuuuuucking time
+   //apply font override to new forms
```

### Step 4.2 — Rename `feces.pas` → `tablesignature.pas`

```
git mv "Cheat Engine/feces.pas" "Cheat Engine/tablesignature.pas"
```

Then edit line 1 of the renamed file:
```diff
- unit feces;
- //friends endorsing cheat engine system
+ unit tablesignature;
+ //Table signing and signature verification system (ECDSA P-521 + SHA-512)
```

Update all files that reference `feces` in their `uses` clauses:

| File | Line | Change |
|---|---|---|
| `OpenSave.pas` | L202 | `feces` → `tablesignature` |
| `MainUnit.pas` | L1124 | `feces` → `tablesignature` |
| `LuaHandler.pas` | L135 | `feces` → `tablesignature` |
| `formsettingsunit.pas` | L404 | `feces` → `tablesignature` |
| `cheatengine.lpr` | L106 | `feces` → `tablesignature` |

### Step 4.3 — `globals.pas`

**Line 170** — clean the comment:
```diff
- //Learn to use 14 byte jmps people
+ //14-byte jump encoding required
```

### Step 4.4 — `tcclib/tcc.h`

**Line 200** — clean the profane comment:
```diff
- #define TCC_IS_NATIVE //cheat engine fuckery
+ #define TCC_IS_NATIVE //Reverie: required for native compilation mode
```

### Step 4.5 — Compile, verify, commit

```
git commit -m "phase1: rename TFormFucker, feces.pas, clean profane comments"
```

---

## Step 5: Final Verification & Phase Commit

### 5.1 — Full build verification

```powershell
# Win64
lazbuild --build-mode="Debug Win64" "Cheat Engine\cheatengine.lpi"

# Win32
lazbuild --build-mode="Debug Win32" "Cheat Engine\cheatengine.lpi"

# Sub-projects
lazbuild "Cheat Engine\speedhack\speedhack.lpi"
lazbuild "Cheat Engine\VEHDebug\vehdebug.lpi"
lazbuild "Cheat Engine\luaclient\luaclient.lpi"
```

### 5.2 — Smoke test

- Launch the compiled executable
- Verify no crash at startup (the `EnableCheatECoinSystem` call is gone)
- Verify the About dialog opens without a Patreon link
- Open the Trainer Generator — confirm no ad-related UI or crash

### 5.3 — Tag the milestone

```
git tag phase1-debloat
```

---

## Summary of Files Modified

| Step | Files Modified | Files Deleted |
|---|---|---|
| **1** (Monetization) | `MemoryRecordUnit.pas`, `MainUnit.pas`, `LuaHandler.pas`, `trainergenerator.pas`, `trainergenerator.lfm`, `cheatengine.lpr` | `cheatecoins.pas`, `frmmicrotransactionsunit.pas`, `frmmicrotransactionsunit.lfm`, `frmAdConfigUnit.pas`, `frmAdConfigUnit.lfm`, `frmAdConfigUnit.lrt`, `cesupport.pas` |
| **2** (URLs) | `aboutunit.pas`, `MainUnit.pas`, `DBK32functions.pas`, `Tutorial/frmhelpunit.pas`, `Tutorial/Unit1.pas`, `Tutorial/Unit4.pas`, `Tutorial/Unit7.pas`, `Tutorial/Unit10.pas`, `Tutorial/cetranslator.pas`, `README.md`, `READMEes.md` | `.github/FUNDING.yml` |
| **3** (Legacy OS) | `globals.pas`, `CEFuncProc.pas`, `debugeventhandler.pas`, `first.pas`, `MainUnit2.pas`, `feces.pas` | — |
| **4** (Naming) | `cheatengine.lpr`, `tablesignature.pas` (renamed), `OpenSave.pas`, `MainUnit.pas`, `LuaHandler.pas`, `formsettingsunit.pas`, `globals.pas`, `tcclib/tcc.h` | — |

## Risk Assessment

| Risk | Mitigation |
|---|---|
| Removing `cesupport` breaks trainer generation | The trainer generator will work without ads. The `cbSupportCheatEngine` checkbox and ad window are purely cosmetic — they don't affect trainer functionality. |
| `iswin2kplus` removal breaks conditional logic | Every branch guarded by this is dead on Win10. Search globally and verify each site. |
| `feces.pas` rename misses a reference | The 5 `uses` clause references were found via grep. The `.lpi` project file may also reference it — check and update. |
| `.lfm` form file refers to deleted component | The `trainergenerator.lfm` must have `cbSupportCheatEngine` removed. Lazarus will refuse to load the form otherwise. |

## Decisions (Resolved)

| Question | Decision |
|---|---|
| **Tutorial module** | **Strip entirely** — delete `Tutorial/` directory and all references. Too tightly coupled to CE identity. |
| **`cbSupportCheatEngine` checkbox** | **Delete entirely** — remove from both `.pas` and `.lfm`. No replacement needed. |
