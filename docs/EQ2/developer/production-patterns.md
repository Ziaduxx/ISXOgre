# Production Script Patterns (Learned from Real Scripts)

This document distills coding patterns from the live scripts folder at `C:\Program Files (x86)\InnerSpace\Scripts`. It is a **secondary, informational reference**: patterns observed in the wild, not a specification. Where it conflicts with `coding-practices.md` or the API docs (`ogrebot-api.md`, `isxogre-api.md`), **the API docs win for signatures and behavior**. This document shows how those APIs are actually combined, which "dialect" a given file is written in, and which bugs recur often enough to avoid copying.

**Method.**
- **Corpus:** every `.iss` file in the Scripts folder, excluding `eq2a/`, `EQ2Craft/`, `Eq2JCommon/`, `EQ2OgreCodex/`, `init-session/`, `init-uplink/`, `ISBoxer Images/`, root-level `ISBOXER*` files (matched case-insensitively), `.venv/` and `.git/` folders. That leaves **1,066 files**, about 8 MB.
- **Full read:** all 1,066 files were read by 13 independent reviewers, about 82 files each. This was a full read, not a sample.
- **Validation:** counts marked "corpus-wide" were re-checked with grep across all filtered files. One file that turned out to be a saved web page was excluded from the counts and has since been deleted.
- **How to weight evidence:** the corpus has heavy copy-paste. One author often duplicates a whole file across expansions, or into both `EQ2OgreBot\...\Default` and `EQ2RAW\IC`. A raw file count can therefore overstate how widespread a convention is. Where it matters, this document says how many independent authors use a pattern.

---

## 1. Know Which Dialect You're Editing

The corpus is several codebases sharing one folder, written by different authors in different eras. Before editing a file, identify its generation and match its style. Mixing dialects inside one file is the most common source of subtle breakage.

| Generation | How to recognize it | Helper calls | Files (corpus-wide) |
|---|---|---|---|
| **A: Inline copy-paste** (oldest) | Helper functions pasted at the bottom of every `_Common.iss`. Often a `; DO NOT MESS WITH THE FOLLOWING CODE.` banner. British spelling `initialise_move_to_next_boss`. | `call move_to_next_waypoint`, `call Tank_n_Spank`, `call mend_and_rune_swap "<adorn>"`, `call HO "<Mode>"` | Most Reign of Shadows / Visions of Vetrovia / Renewal of Ro / Blood of Luclin files |
| **B: `Obj_Kord` library** (author: Kordulek) | `#include .../Support_Files_Common/Kordulek_ICFunctions.iss`, `; Author: Kordulek` header, cleanup in `method Shutdown()` | `call Obj_Kord.Move_to_next_waypoint`, `Obj_Kord.Tank_n_Spank "${n}" "${spot}" [TRUE\|FALSE]`, `Obj_Kord.HO`, `Obj_Kord.Initial_Settings_for_SoD_Zones` | 70 files call `Obj_Kord.` (Scars of Destruction, Rage of Cthurath) |
| **C: `IC_Helper.iss` library** | `#include .../IC_Helper.iss`, header "This IC file requires HOHelperScript/MendRuneSwapHelperScript/IC_Helper files", `raw_main` pre-step, best comments in the corpus | US spelling `initialize_move_to_next_boss`, `KillSpot:Set[x,y,z]`, `CheckGroupSetup`, `SetInitialInstanceSettings`, `EnterPortal`, 4-arg `mend_and_rune_swap` | 79 files include an `IC_Helper` (Ballads of Zimara Custom, EQ2RAW chrono) |
| **D: Zone-reset loop scripts** (authors: Jiim / Squeeze / "Marty Party") | Under `EQ2RAW\IC\TSO\`, `\SF\`, chrono. Unicode banner comments, `ogre ica`, `gcsRetValue` | `Movetoloc`, `HandleNamed`, `PostNamed`, `CheckZoneResetStatus`, `ResetZone` | 35 files use `gcsRetValue` |
| **E: JSON path routes** (newest) | `variable jsonvalue route`, `route:SetValue["$$>[ '<x,y,z>', ... ]<$$"]`, `#include .../Path_Helper_Include.iss` | `call Obj_PathHelper.GroupFollowCampSpotPath route` | 8 files |

**Zone-in has three API generations**, which follow the eras above:

| Call | Files (corpus-wide) | Era |
|---|---|---|
| `Obj_OgreIH.ZoneNavigation.GetIntoZone` / `.ZoneOut` | 133 | Most common |
| `Obj_OgreIH.Handle_Mesh_ZoneIn_Process` | 45 | Kordulek / Rage of Cthurath |
| `Obj_OgreIH.CD.GetIntoZone` / `.CD.ZoneOut` | 27 | Chaos Descending, also in the official template |

**Timers have four coexisting idioms**, never unified:
- `Object_Timer` (`OgreCommon/Object_Timer.iss`): the canonical shared implementation, but only 11 declarations exist corpus-wide. It reschedules itself via `timedcommand` plus a random `UniqueID`, so many instances can share one global event. Real instances are named `Timer_<Description>` (e.g. `Timer_SuccessWait`, `Timer_CurseCalled`), not the documented `Obj_Timer_<Description>`.
- `EQ2RAW_AbilityUseTimer` (24 files, "Trouble" raid scripts): created with `declare CureTimer EQ2RAW_AbilityUseTimer 2000`, polled via `.IsReady`. Note that `IsReady` re-arms itself when it returns TRUE.
- `wait <deciseconds> <condition>`: an early-exit timed wait such as `wait 100 ${EQ2.Zoning}` or `wait 600 ${Actor[...].ID(exists)}`. In practice this is the most common timing tool.
- Counter cooldowns: a loop-scoped `int` set to `-N` and incremented each pass, or bounded polls like `while !${cond} && ${Counter:Inc} <= N { wait 1 }`.

---

## 2. The Instance-Controller Contract

**Corpus-wide: `_StartingPoint` appears in 229 files.** This is a designed contract, not just an emergent habit. `EQ2OgreBot\InstanceController\Instance_Files\Sample_Instance_File.iss` is the annotated template, with comments such as *"This object definition must be exactly this name"* and *"0 = get into zone, 1-# named, last = zone out"*.

```lavishscript
; zone-name/config variables FIRST (the include consumes them)
#include "${LavishScript.HomeDirectory}/Scripts/EQ2OgreBot/InstanceController/Support_Files_Common/Ogre_Instance_Include.iss"

function main(int _StartingPoint=0, ... Args)
{
    call function_Handle_Startup_Process ${_StartingPoint} "-NoAutoLoadMapOnZone" ${Args.Expand}
}

atom atexit()
{
    echo ${Time}: ${Script.Filename} done
}

objectdef Object_Instance
{
    function:bool RunInstance(int _StartingPoint=0)
    {
        if ${_StartingPoint} == 0
        {
            call Obj_OgreIH.ZoneNavigation.GetIntoZone "${sZoneName}"
            if !${Return}
            {
                Obj_OgreIH:Message_FailedZone
                return FALSE
            }
            Ogre_Instance_Controller:ZoneSet
            call Obj_OgreIH.Set_VariousOptions
            call Obj_OgreIH.Set_PriestAscension FALSE
            Obj_OgreIH:Set_NoMove
            Obj_OgreIH:SetCampSpot
            call Obj_OgreUtilities.PreCombatBuff 5
            _StartingPoint:Inc
        }
        if ${_StartingPoint} == 1
        {
            call This.Named1 "Boss Name"
            if !${Return}
            {
                Obj_OgreIH:Message_FailedZone["#1: Boss Name"]
                return FALSE
            }
            call Obj_OgreIH.Get_Chest
            _StartingPoint:Inc
        }
        ; ... one independent `if` per named (NOT elseif) ...
        ; final block: ZoneNavigation.ZoneOut, Message_FailedZoneOut on failure, return TRUE
    }

    function:bool Named1(string _NamedNPC="Doesnotexist")
    {
        ; 1. KillSpot + Ob_AutoTarget:AddActor["add name",0,FALSE,FALSE]
        ; 2. initialise_move_to_next_boss + chained move_to_next_waypoint "x,y,z"
        ; 3. already dead?  -> Obj_OgreIH:Message_NamedDoesNotExistSkipping["${_NamedNPC}"], return TRUE
        ; 4. kill (Tank_n_Spank or bespoke per-difficulty branch)
        ; 5. still alive?   -> Obj_OgreIH:Message_FailedToKill["${_NamedNPC}"], return FALSE
        ; 6. return TRUE
    }
}
```

- **Why the integer checkpoint:** the controller can resume mid-zone after a disconnect or wipe, by re-entering with a saved `_StartingPoint`.
- **Debug scaffold:** many files keep `_StartingPoint:Set[0]` immediately followed by `_StartingPoint:Inc`, under "for debugging only". You hand-edit the `0` to jump to a boss. Leave it at `0` when shipping.
- **Thin difficulty wrappers:** most zones have one `<Zone>_Common.iss` plus a 2–5 line wrapper per difficulty: `variable string sZoneName="Zone: Name [Heroic I]"`, `variable string sDifficulty="Normal"`, `#include "../<Zone>_Common.iss"`.
  - **Wrapper prefix ≠ Common prefix:** the wrapper's numeric prefix is a menu sort key and needn't match its Common file. For example, `_26_..._Heroic_II` includes `_13_..._Common`.
  - **`sDifficulty`:** it is `"Normal"` almost everywhere, even for Heroic and Expert, so it is effectively vestigial.
  - **Template drift:** newer wrappers start with `;    ${Ogre_Instance_Controller.CleanZoneName}`, while the oldest omit it and sometimes omit `sDifficulty`. That's template drift, not a bug.
- **Difficulty branching:** older files test `${Zone.Name.Equals["${Solo_Zone_Name}"]}`. Kordulek files test `${OgreBotAPI.ZoneName.Equal["${Heroic_2_Zone_Name}"]}`. Both styles are in heavy production use (`.Equal[` 2,513 lines, `.Equals[` 1,299 lines corpus-wide).
- **Two cleanup hooks:** most files clean up in top-level `atom atexit()`. **70 files** instead, or additionally, define `method Shutdown()` inside `Object_Instance`, mostly Kordulek-era. Check which one a file uses before adding cleanup.
- **The `raw_main` route pre-step (EQ2RAW):** `main` calls `raw_main ${Args.Expand}` first. If not in the zone, it runs the expansion's `*_zone_routes` script on the group (`oc !ci -RunScriptRequiresOgreBot igwbn:${Me.Name} ".../xxx_zone_routes" "${sZoneName}" FALSE FALSE`), runs it locally, then `while ${Script[xxx_zone_routes](exists)} wait 10`. `atexit` ends it. **The `if` must be braced** (see §6).
- **RAW vs Default:** a diff of matched pairs shows `EQ2RAW\IC` copies are the work-in-progress versions and `EQ2OgreBot\...\Default` copies are the settled ones. When both exist, trust Default.

---

## 3. Core API Idioms

### Movement is a camp-spot change, not a nav call
```lavishscript
Obj_OgreIH:SetCampSpot
Obj_OgreIH:ChangeCampSpot["x,y,z"]
call Obj_OgreUtilities.HandleWaitForCampSpot 10
call Obj_OgreUtilities.HandleWaitForCombat
call Obj_OgreUtilities.WaitWhileGroupMembersDead
call Obj_OgreUtilities.HandleWaitForGroupDistance 5
```
- **Coordinate formats:** Obj_ API methods take one quoted `"x,y,z"` string. `oc !ci -ChangeCampSpotWho <who> x y z` takes **space-separated** coordinates. Mixing them up is a frequent slip.
- **Per-character scripts:** standalone helpers use the low-level `Ogre_CampSpot:Set_CampSpot[name]` / `Set_HowClose[name,N]` / `Set_ChangeCampSpot[name,x,y,z]` / `ClearAll[name]`.
- **Navmesh legs:** `Obj_OgreUtilities.NavToLoc "<x,y,z>"` after `Obj_OgreIH:LetsGo`.

### The pull / kill / loot sequence
```lavishscript
oc ${Me.Name} is pulling ${_NamedNPC}
Obj_OgreIH:SetCampSpot
Obj_OgreIH:ChangeCampSpot["${KillSpot}"]
call Obj_OgreUtilities.HandleWaitForCampSpot 10
oc !ci -PetOff igw:${Me.Name}
wait 10
oc !ci -PetAssist igw:${Me.Name}
Actor["${_NamedNPC}"]:DoTarget
wait 50
call Obj_OgreUtilities.HandleWaitForCombat
call Obj_OgreUtilities.WaitWhileGroupMembersDead
eq2execute summon
wait 10
call Obj_OgreIH.Get_Chest
```

### Group control: the `oc` command bus
**Corpus-wide: 9,532 `oc !ci` lines vs 1,509 `oc !c` lines.** The authoritative list of `-Flag` commands is the `ParseMessage` switch in `OgreConsole\Everquest2\Object_Everquest2CommandParser.iss`. It has 569 `case` labels, and every one dispatches to a method that resolves the target and calls the matching `OgreBotAPI:` method.

- **Target selectors:** `igw:${Me.Name}` (whole group), `igw:${Me.Name}+fighter` / `+notfighter` / `+scout` / `+priest` / `+mage` / `+-ranger` / `+@healer1`, `igwbn:` (group but not me), `irw:` (raid), and the special `AUTO`.
- **Most common verb:** `-ChangeOgreBotUIOption <who> checkbox_<name> TRUE TRUE`. The trailing booleans vary between 1, 2 and 3 values across files, and their meaning isn't documented in the corpus, so match whatever the surrounding file already does.
- **Two equivalent call styles:** command strings (`oc !ci -CastAbility ...`) dominate the IC files. The "Trouble" raid scripts call the API object directly (`OgreBotAPI:ChangeOgreBotUIOption[${sMyName},"checkbox_...",TRUE,TRUE]`). Both work; stay consistent within a file.
- **The interrupt bracket:** `oc !ci -Pause` → `wait` → `eq2execute clearabilityqueue` → `oc !ci -CancelCasting` → action → `oc !ci -Resume`. **Every early `return` between Pause and Resume leaves the group paused** (seen in `Hate_Rune_Helper.iss`).
- **Set-then-restore:** whatever a fight disables should be re-enabled after the fight, and again in `atexit`/`Shutdown` in case of abort.

### Detrimental detection by icon-ID pair
```lavishscript
; Name: Auriac Toxin   MainIconID: 909  BackDropIconID: 315
if ${OgreBotAPI.DetrimentalInfo[909, 315]}
; or
if ${Me.Effect[Query, Type == "Detrimental" && MainIconID == 909 && BackDropIconID == 315](exists)}
```
- **Argument order is Main first, BackDrop second.** Comments in raid scripts often paste the game's "Examining Detriment" line, which lists BackDrop first. Several files consequently swap the order.
- **Extended form:** `DetrimentalInfo[main, backdrop, ${ActorID}, "exists"|"CurrentIncrements"|"Duration"]`.
- **One file uses a space instead of a comma** (`[262 316]`). Always use a comma.

### Cross-character state and helper scripts
```lavishscript
oc !ci -Set_Variable igw:${Me.Name} "${Me.Name}_RuneSwapNeeded" "TRUE"     ; publisher
wait 1
${OgreBotAPI.Get_Variable["${Toon}_RuneSwapNeeded"]}                         ; reader
```
- **Launching helpers:** helper scripts run on every member via an end-then-run pair: `oc !ci -EndScriptRequiresOgreBot igw:${Me.Name} ${HelperScript}` followed by `oc !ci -RunScriptRequiresOgreBot igw:${Me.Name} ${HelperScript} "${_NamedNPC}"`.
- **Helper shape:** `function main(string _NamedNPC)` → `switch ${_NamedNPC}` → per-boss `while ${Actor[namednpc,"..."].ID(exists)} { ...; wait 10 }`.
- **Reporting back:** helpers report completion by setting a `..._Complete` variable in their own `atom atexit()`.

### Text-trigger mechanic watcher
```lavishscript
variable bool bJoustIncoming=FALSE
atom JoustWatcher(string Text)
{
    if ${Text.Find["exact phrase"](exists)}
        bJoustIncoming:Set[TRUE]
}
; in the boss function:
Event[EQ2_onIncomingText]:AttachAtom[JoustWatcher]
; ... fight loop polls and clears bJoustIncoming ...
Event[EQ2_onIncomingText]:DetachAtom[JoustWatcher]
```
- **Detach is rare:** corpus-wide there are **432 `AttachAtom` lines and only 125 `DetachAtom` lines**, and **145 files attach without ever detaching**. Script exit cleans up, but attaching inside a `NamedN` that can re-run registers duplicate handlers. In new code, detach at the end of the fight.
- **Atom and variable sharing one name** is common in production (`atom Joust` + `variable bool Joust`). It appears to work because atoms and variables live in different namespaces, but it's easy to misread. Use distinct names in new code, as the example above does.
- **Chat-text handlers:** `EQ2_onIncomingChatText` takes six parameters: `(int ChatType, string Message, string Speaker, string TargetName, string SpeakerIsNPC, string ChannelName)`.

---

## 4. Naming: What the Corpus Actually Does

`coding-practices.md` describes the intended house style. The table compares it with real usage, counted across all 1,066 files.

| Convention | Documented | Actual |
|---|---|---|
| `_` parameter prefix | `_sName` | **The most reliably followed rule**, on IC contract signatures (`_StartingPoint`, `_NamedNPC`). The type letter after the underscore is almost never used (`_NamedNPC`, not `_sNamedNPC`). Helper-function parameters (`waypoint`, `ScanRadius`, `KillSpot`, `Mode`) usually skip the underscore entirely. |
| Type prefixes `s`/`i`/`f`/`b` | every variable | Followed in framework globals (`sZoneName`, `iZoneResetTime`, `bEnableDebug`, `sMyName`) and in "Trouble" scripts' globals. Rare elsewhere: bools almost never get `b`, and floats essentially never get `f`. Not applied to `objectdef` members, even in `Object_Timer.iss`. |
| `loc` for point3f | `locDestination` | **0 of 1,562 `point3f` declarations use a `loc` prefix.** 960 end in `Spot` (`KillSpot`, `TankSpot`), 119 in `Point`, 102 in `Loc`. `...Loc` is used for transient "actor's current position" reads, `...Spot` for author-defined destinations. |
| Container prefixes | `isPlayerNames` | Essentially unused, **except** `gcsRetValue` (global `collection:string`) in 35 files of the Generation-D lineage. |
| `.Between[min,max]` | preferred | **8 occurrences in 1,066 files.** Chained comparisons are universal. |
| Singleton naming | `Obj_X` | Two real prefixes coexist: `Obj_` (303 files: `Obj_OgreIH`, `Obj_Kord`, `Obj_JSONHelper`) and `Ob_` (254 files: `Ob_AutoTarget`, `Ob_EQ2Chars`). These are framework-owned names; use them exactly as they are. |

**Rule of thumb:**
- **New scripts:** follow `coding-practices.md`.
- **Existing scripts:** match the surrounding file's dialect (§1). Don't rename variables to the "correct" convention unless asked.

---

## 5. Model Code Worth Imitating

These are the cleanest examples in the corpus, useful as templates for new work:
- **Kannkor's core libraries** (`OgreCommon/Object_Timer.iss`, `Object_StopWatch.iss`, `Object_Debug.iss`, `LoadOgreMCP.iss`): `#ifndef` include guards, `method Initialize`/`Shutdown` pairs, and events registered in `Initialize` and detached in `Stop`/`Shutdown`.
- **`EQ2CooldownTracker.iss`:** the textbook readiness guard (`${ISXEQ2(exists)}` → `${ISXEQ2.IsReady}` → `${EQ2.Zoning}`) and symmetric attach/detach.
- **"Trouble" raid scripts** (`EQ2RAW\RaidScripts\Renewal_of_Ro\Scripts\ror_*`): version banner, `-Debug` flag parsing with an "Ignoring unknown option" default, a `do { wait ${WaitTime} } while ${Me.InCombat} || ${Actor[Query, Name == "X" && !IsDead](exists)}` loop, and an `atexit` that reverses every setting it changed.
- **Generation C IC files** (e.g. `Vaashkaani_Alcazar_Crescendo_Common.iss`): the best-commented zone scripts.
- **`Aether_Wroughtlands_Native_Mettle_Common.iss`:** boss-prefixed atom names (`NuggetIncomingText`) attached before each fight and detached after it.
- **`EQ2OgreBagManager.iss`:** dated comments explaining a breaking API change (ISXEQ2 `.Slot` moving from 0-based to 1-based) and the fix.

---

## 6. Recurring Bugs — Don't Copy These

Each of these appears independently in several files. Counts are corpus-wide.

| Bug | Files | What happens |
|---|---|---|
| `call` inside `atexit` | 17 (15 are `call Obj_Kord.HO "Disable" TRUE`) | `atexit` always runs as an **atom**, whichever keyword is used (see the note below the table), and atoms cannot use delays. The guide states that atoms cannot use `call`; not yet confirmed in-game. If true, that cleanup step silently doesn't run. Prefer plain `oc !ci ...` commands in `atexit`, or `Script:QueueCommand`. |
| `${Args.Expand}` used in `main(int _StartingPoint=0)` with no `... Args` | 33 | Expands to empty, so arguments are silently dropped. |
| `HandleNamed()` reads `${_NamedNpc}`, which is the caller's parameter and out of scope | 33 | The "already dead" check tests an empty name. |
| `CheckZoneResetStatus` (a `function:bool`) returns TRUE only on the "wait" path | ~15 (same template) | When the zone is already resettable the function returns nothing, and the caller's `if !${Return}` aborts. |
| `Heroic_3_Zone_Name` referenced but never declared | ~24 (39 use it, 15 declare it) | That difficulty branch never matches. |
| Braceless `if` guarding a multi-line `raw_main` body | many, mostly Kordulek | Only the first statement is conditional. `RunScript` and the wait loop run unconditionally. **Always brace multi-statement `if` bodies.** |
| Dead "placeholder" atoms (`Text.Find["placeholder"]`, attach commented out) | 15 | Unfinished template scaffolding; the mechanic is not handled. |
| Stray character at the end of a coordinate string (`"...-215.163055w"`, `...}"`) | 14 | Malformed coordinate. |
| `OgreIH:Set_Debug_Mode[TRUE]` missing the `Obj_` prefix | 18 | References a nonexistent object; debug mode never turns on. |
| `#include` of misspelled `IC_Helper_Extened.iss` | 9 | A hard `#include` of a file that doesn't exist. The real file is `IC_Helper_Extended.iss`. |
| Missing closing quote on `#includeoptional "...aod_zone_routes.iss` | several | `#includeoptional` hides the failure; the target folder `Scripts/ZoneRoutes/` doesn't exist anyway. |
| Blocking `Messagebox` in automated flow | 32 | Halts the script until a human clicks OK. Sometimes intentional (a manual step), but it can stall a group. |
| Duplicate `method` names in one objectdef (`ClearUI` in `Object_EQ2Chars.iss` and `Object_UplinkInfo.iss`) | 2+ | The second definition silently wins. |
| Arithmetic in `:Set[${A} - ${B}]` without `Math.Calc` | several | Likely stores the literal expression text, not the result. |
| `${NamedNPC}` (no underscore) where the parameter is `_NamedNPC` | several | The wait-while-alive loop never runs. |
| Wrong atom name in `AttachAtom` (attaches `ChestSpawned`, defines `ActorSpawned`) | 2+ | The handler never fires, with no error. |
| Literal names wrapped in `${}` (`AddActor["${Meditation of a Hundred Strikes}",...]`) | 1 file, 8 lines | Expands to empty; no actor is added. |

**Copy-paste text tells:** many files print "Chanter's repair bot not available, trying your priests." three times in a row for shaman, cleric and druid. Failure messages often name the wrong boss, and headers name the wrong script. When copying a file, update every self-referential string.

**`function atexit()` vs `atom atexit()` is not a bug.** 54 files declare `function atexit()` and 312 declare `atom atexit()`. Both run at script end: the LavishScript 1.67 release notes say *"'atexit' is now atomic. If it is defined as a function, it will automatically be converted to an atom."* What matters is the **body**, because it always runs atomically. `wait`/`waitframe` are documented errors there (corpus-wide, 0 atexit bodies use them), and `call` is the open question in the table above. Prefer `atom atexit()` in new code, since it states the real behavior.

---

## 7. Corpus Caveats

- **Generic InnerSpace/ISBoxer code, not OgreBot evidence:** `keymapper.iss`, `menuman.iss`, `repeater.iss`, `vfxlayout.iss`, `IRCLib.iss`, `IRCSample*.iss`, `windowsnapper2.iss`, `uireset.iss`, `OgreCommon/obj_LSTypeIterator.iss` (imported EVEBot code), and `EQ2NavCreator.iss` (2008).
- **Tutorial or notes files, not runnable:** `Documentation/LERN/*` (canonical LavishScript syntax), `__partial-scripts.iss`, `whileloop.iss`, `OgreMCP_Buttons.iss` (a plain-English notes file with a `.iss` extension).
- **Stubs and work in progress:** several are self-described (e.g. `combat_mastery.iss`, `status.iss`, `boz_avatar_raid_brell_contested.iss`, `sod_underdepths_ziaduz.iss`). A few zero-byte files exist (`blank_holder.iss`, `rune_swap_stiffle_and_stun.iss`).
- **Duplicates:** zone content is often duplicated between `EQ2OgreBot\...` and `EQ2RAW\IC`, and between `Custom\` and `Default\`. Two `Kordulek_ICFunctions.iss` copies have drifted apart. When editing one copy, check for the others.
- **Security note:** `OgreConsole\IRC\OgreConsoleIRC.iss` executes commands such as `-OSExecute` and `-KillSession` received over IRC, gated only by membership in `AuthList`. Treat its authorization setup as security-sensitive.
