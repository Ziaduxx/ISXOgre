# Production Script Patterns (Learned from Real Scripts)

This document distills actual coding patterns from the live scripts folder at `C:\Program Files (x86)\InnerSpace\Scripts`. It is a **secondary, informational reference** — patterns observed in the wild, not a specification. Where it conflicts with `coding-practices.md` or the API docs (`ogrebot-api.md`, `isxogre-api.md`), **the API docs remain authoritative for signatures and behavior**; this document exists to show how those APIs actually get used together, and to flag real bugs seen repeatedly so they aren't copied forward.

**Method:** The corpus was filtered to exclude `eq2a/`, `EQ2Craft/`, `Eq2JCommon/`, `EQ2OgreCodex/`, `init-session/`, `init-uplink/`, `ISBoxer Images/`, and root-level `ISBOXER*` files, leaving ~1,175 `.iss` files. A stratified sample of 488 files (~41%) was drawn across folder, file size, and file age, then read in full by independent reviewers. Every claim below marked "confirmed corpus-wide" was additionally validated with a grep across all ~1,175 filtered files, not just the sample. Some scripts in this corpus are unfinished, experimental, or buggy — a pattern's confidence note reflects how many independent files/authors exhibited it, not whether it's correct.

---

## 1. Core Structure: The Instance-Controller Checkpoint State Machine

This is the dominant convention in the corpus — **confirmed corpus-wide: `_StartingPoint` appears in 230 files.** It's not just emergent convention: the corpus contains an actual annotated template file, `EQ2OgreBot\InstanceController\Instance_Files\Sample_Instance_File.iss`, with comments like *"This object definition must be exactly this name"* — this is a designed spec that scripts are built from.

```lavishscript
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
        if ${_StartingPoint}==0
        {
            ; zone-in / buff / setup
            _StartingPoint:Inc
        }
        if ${_StartingPoint}==1
        {
            call This.Named1 "Boss Name"
            if !${Return}
            {
                Obj_OgreIH:Message_FailedZone["#1: Boss Name"]
                return FALSE
            }
            _StartingPoint:Inc
        }
        ; ... one independent `if` block per boss/step (not elseif) ...
    }

    function:bool NamedN(string _NamedNPC="Doesnotexist")
    {
        ; "already dead" guard -> return TRUE (skip)
        ; move via chained move_to_next_waypoint calls
        ; kill (see Tank_n_Spank, §2)
        ; "still alive" verification -> return FALSE on failure
        ; loot chest, return TRUE
    }
}
```

The integer checkpoint lets the controller resume mid-zone after a disconnect/wipe instead of restarting from scratch. A frequently-seen debug scaffold immediately re-derives `_StartingPoint` with a hardcoded override for testing a specific boss ("for debugging only" comment) — meant to be hand-edited then reset to 0 before shipping; don't copy the override value itself.

**Thin per-difficulty wrapper split:** most zones have one real `<Zone>_Common.iss` holding the logic above, and a 2–5 line wrapper per difficulty (Solo/Heroic I/Heroic II/Expert/Event) that just sets `variable string sZoneName="..."`, `variable string sDifficulty="Normal"`, and `#include "../<Zone>_Common.iss"`. This is why so many `.iss` files in the corpus are tiny — it's DRY-by-convention, not incomplete work. The template visibly grew fields over time: oldest-quartile shims are a bare 2-line form (no `sDifficulty`, no header comment); current-era shims add `sDifficulty="Normal"` and a leading `; ${Ogre_Instance_Controller.CleanZoneName}` comment line. Older leaf files were never backfilled when the template gained fields — that's expected, not a bug to fix on sight.

A verbatim warning comment — `; DO NOT MESS WITH THE FOLLOWING CODE.` — recurs across many `_Common.iss` files right after the zone-name variables, marking the boundary between per-zone config and shared/templated framework glue.

**Competing cleanup mechanism:** most files clean up via the top-level `atom atexit()` shown above, but a newer sub-lineage (`Kordulek`-authored Scars-of-Destruction-era zones) instead — or additionally — defines `method Shutdown()` inside `objectdef Object_Instance`. Both mechanisms are real; check which one a file you're editing actually uses rather than assuming.

---

## 2. Three Coexisting "Dialects" for the Same Job

The single most useful thing to know before writing a new Instance Controller script: **the exact same job — move to a boss, tank-and-spank it, manage rune-swaps/HO, handle zone reset — has at least three parallel, non-shared implementations**, and which one a given file uses correlates with its era/author, not with correctness. Match whichever dialect the file you're editing already uses; don't mix them.

| Dialect | How it's invoked | Where it shows up |
|---|---|---|
| **Bare global functions** (oldest/most common) | `call move_to_next_waypoint "x,y,z"`, `call Tank_n_Spank`, `call mend_and_rune_swap`, `call HO(mode)` — duplicated locally in each `_Common.iss`, ~150 lines per file | Renewal of Ro, Ballads of Zimara era and most files generally |
| **`Obj_Kord` singleton** | `#include Kordulek_ICFunctions.iss`, then `call Obj_Kord.Move_to_next_waypoint`, `Obj_Kord.Tank_n_Spank`, `Obj_Kord.Mend_and_rune_swap`, `Obj_Kord.HO` | Scars of Destruction / Rage of Cthurath era, author "Kordulek" |
| **Decorative third lineage** | `Movetoloc`, `HandleNamed`, `PostNamed`, `CheckZoneResetStatus`/`ResetZone`, `gcsRetValue` (global collection:string), unicode section banners | TSO-era zones under `EQ2RAW\IC\TSO\` and `EQ2RAW\IC\SF\` (Erudin Research Halls, Kurn's Tower, Mistmyr Manor, The Deep Forge, Covenant District, etc. — **confirmed in 33 files**), authors "Jiim"/"Marty Party" |

The third dialect is also the **one place the documented container-prefix convention is actually followed** — see §4.

Similarly, **zone-in/zone-out has three coexisting generations**: `Obj_OgreIH.ZoneNavigation.GetIntoZone`/`.ZoneOut` (older/common), `Obj_OgreIH.Handle_Mesh_ZoneIn_Process` (Kordulek/Rage-of-Cthurath-era), and `Obj_OgreIH.CD.GetIntoZone`/`.CD.ZoneOut` (Chaos Descending expansion, also used in the official `Sample_Instance_File.iss` template). Pick based on the zone's era/expansion, not arbitrarily.

**And three coexisting timer/cooldown objects**, never unified:
- `Object_Timer` (`OgreCommon/Object_Timer.iss`) — the canonical shared implementation. Self-reschedules via `timedcommand N Event[OgreEvent_Timer]:Execute["${This.UniqueID}"]` rather than a persistent loop; a random `UniqueID` (concatenated `Math.Rand[...]` calls, since `Rand` is capped at int16 range) lets many instances share one global event. `Object_StopWatch` in the same folder is the identical idea under a different name.
- `EQ2RAW_AbilityUseTimer` — `declare CureTimer EQ2RAW_AbilityUseTimer 2000`, polled via `.IsReady`. Tied to the "Trouble"-authored RaidScripts lineage.
- `EQ2RAW_CountdownTimer` (`Countdown.iss`) — `Initialize(ReportFrequency)` / `:Set[NextTrigger]` / `member:uint Check()`.

In practice, **plain `wait N <condition>`** (an early-exit timed poll — `wait 100 ${EQ2.Zoning}`, `wait 50 !${Ogre_CampSpot.AtCampSpot}`) is more common than any dedicated timer object for simple waits. A fourth, even more manual idiom also appears: a loop-scoped `variable int FooCounter=0` incremented once per iteration, used as an N-iteration cooldown. All four idioms coexist even within single files.

---

## 3. API Usage Idioms

### `oc !ci -Command <target> args...` — the dominant control-plane idiom

Scripts drive OgreBot almost entirely by broadcasting a chat/console command rather than calling an object API directly. **Targeting/addressing is not standardized** — expect to see, sometimes within the same file family:

- `igw:${Me.Name}` / `igw:${Me.Name}+fighter` / `igw:${Me.Name}+notscout` / `igw:${Me.Name}+@healer1` (role/archetype filters)
- The literal keyword `AUTO`
- A bare role name with no prefix (`enchanter`)
- Pipe-delimited class lists (`Warrior|Crusader|Ranger|Bard|Brigand`)
- Command prefix itself varies: `oc !ci`, `oc !c`, rarely `irc !c`

Don't assume one universal targeting syntax when writing new code — match whichever scheme the specific command already uses elsewhere in the file.

### Named-boss kill loop ("Tank_n_Spank")

```lavishscript
Obj_OgreIH:SetCampSpot
Obj_OgreIH:ChangeCampSpot["x,y,z"]
call Obj_OgreUtilities.HandleWaitForCampSpot 10
Ob_AutoTarget:Clear
Ob_AutoTarget:AddActor["Boss Name",dist,bool,bool]
Actor[namednpc,"${_NamedNPC}"]:DoTarget
; ... pull/engage ...
call Obj_OgreUtilities.HandleWaitForCombat
call Obj_OgreUtilities.WaitWhileGroupMembersDead
```

This exact body is typically duplicated per-file rather than shared, with naming variants (`Tank_n_Spank`, `Tank_n_Spank2`, `Tank_at_KillSpot`, `Tank_n_Spank_Ensure_Group_Behind`) rather than parameters.

### Manual ability-injection bracket

To force a cast outside the bot's normal rotation: `oc !ci -Pause` → `wait N` → `eq2execute clearabilityqueue` → `oc !ci -CancelCasting` → one or more `oc !ci -CastAbility` calls → `oc !ci -Resume`. Whatever gets paused/disabled is always symmetrically resumed/re-enabled, usually mirrored between `main()` and `atexit()`.

### Detrimental/buff detection by icon-ID pair, not name

```lavishscript
; Name: Auriac Toxin
if ${Me.Effect[Query, Type=="Detrimental" && MainIconID==909 && BackDropIconID==315].ID(exists)}
```
or the shortcut `${OgreBotAPI.DetrimentalInfo[MainIconID, BackDropIconID]}`. Always preceded by a comment naming the human-readable effect. **Both forms coexist for the identical mechanic in different files** — e.g. the same "Queen Era'selka" curse is checked via the shortcut in one file and the full query in another. If you're adding a new check, either form is acceptable; check what the surrounding file already uses.

### Cross-character coordination via shared variables

```lavishscript
oc !ci -Set_Variable igw:${Me.Name} "KeyName" "Value"      ; publisher
${OgreBotAPI.Get_Variable["KeyName"]}                        ; subscriber
```
Often wrapped in a bounded poll: `while !${...} && ${Counter:Inc} <= N { wait 1 }`. This is the dominant idiom, but not the only one — some files instead use `declare Var TYPE globalkeep DEFAULT` plus a double-hop `relay`, or a plain `declare Var TYPE script DEFAULT` for same-script sharing. All three solve the same problem and were never unified; prefer `Set_Variable`/`Get_Variable` for new code since it's the most common.

### Text-trigger mechanic watcher

The atom handler and the boolean flag it sets share the same name (legal — atoms and variables are different namespaces):

```lavishscript
atom SomeMechanicName(string Text)
{
    if ${Text.Find["exact trigger phrase"](exists)}
        SomeMechanicName:Set[TRUE]
}
Event[EQ2_onIncomingText]:AttachAtom[SomeMechanicName]   ; attached right before the relevant fight
; ...
Event[EQ2_onIncomingText]:DetachAtom[SomeMechanicName]   ; not always present — see §5
```

### Variadic argument parsing

Stable across the whole history of the corpus (files from 2008 through the present use this identically):

```lavishscript
function main(... Args)
{
    for (i:Set[1]; ${i}<=${Args.Used}; i:Inc)
    {
        switch ${Args[${i}]}
        {
            case -Debug
                ; ...
                break
            case -SP
                _StartingPoint:Set[${Args[${i:Inc}]}]   ; consume the following token as the flag's value
                break
        }
    }
}
```

---

## 4. Naming Conventions: What The Corpus Actually Does

`coding-practices.md` documents a house style. Real adherence is uneven and correlates with **which subsystem/era wrote the file**, not with file age. Numbers below are corpus-wide (all ~1,175 filtered files) unless noted otherwise.

| Convention | Documented rule | What the corpus actually does |
|---|---|---|
| Parameter prefix | `_` prefix, e.g. `_sName` | **The most reliably-followed rule in the corpus** — essentially universal on function/method parameters (`_NamedNPC`, `_StartingPoint`) across every author and era. The type letter after the underscore is almost always dropped, though (`_StartingPoint`, not `_iStartingPoint`) — that's the real dominant form, not a violation. |
| Type prefix (`s`/`i`/`f`/`b`) | Every variable gets one | Followed reasonably well for **procedural locals** in gameplay code (`sZoneName`, `iCounter`, `bEnableDebug`) but **not on `objectdef` member variables** — even the canonical `Object_Timer.iss` itself declares `EndTime`, `Name`, `UniqueID`, `TriggerEngine` with no prefix. Bool locals fare worst overall (`Enabled`, `Flying`, `Mute`, `HasEffect` — rarely `bEnabled`). Rule of thumb: type prefixes are a *procedural-code* convention, not a *class-member* convention, in this corpus. |
| `loc` postfix for `point3f` | e.g. `locDestination` | Not the dominant real-world pattern. The dominant pattern is a **domain-name suffix instead**: `KillSpot`, `TankSpot`, `PrePullSpot`, `GroupSpot`. Where a `Loc` suffix *does* appear (`ActorLoc`, `BanditLoc`, `NewLoc`), it's specifically for **transient "current position of an actor" reads**, not author-defined named spots — so `Loc` and `Spot` aren't interchangeable stylistic choices, they mark different kinds of point3f. |
| Container prefix (`ci`/`cs`/`is`/`ii`/`iloc`/`ib`/`set`) | e.g. `isPlayerNames` for `index:string` | Essentially absent everywhere **except** one identifiable lineage: `gcsRetValue` (global collection:string) is used consistently across **33 files** in the TSO/SF "third dialect" zones (§2). Everywhere else, plain descriptive names are used instead (`Bandits`, `NodesEnabled`, `ActorIndex`). If you're extending a TSO/SF-era file, follow `gcsRetValue`'s example; elsewhere, following the documented convention in new code is still good practice even though it's rare in the existing corpus. |
| `.Between[min,max]` range checks | Preferred over chained comparisons | **Confirmed corpus-wide: only 8 occurrences in ~1,175 files.** The near-universal real idiom is chained comparisons (`if ${x} >= min && ${x} <= max`). Good practice for new code; its absence elsewhere isn't a defect. |
| `Object_X` → `Obj_X`/`Ob_X` | Singleton naming | **Two real, comparably-common prefixes coexist corpus-wide: `Obj_` (305 files) and `Ob_` (255 files)** — e.g. `Obj_OgreIH`/`Obj_JSONHelper`/`Obj_Kord` vs. `Ob_AutoTarget`/`Ob_EQ2Chars`/`Ob_UplinkInfo`. Both appear together in the same files, so this isn't a typo of one or the other — it's two historical standards that never converged. When adding a new singleton, match whichever prefix its sibling objects in the same file/subsystem already use. |

**Bottom line:** when writing new code, follow `coding-practices.md` as given for parameters and procedural locals — that part is genuinely close to real practice. Don't invent adherence to the `loc`/container-prefix/`.Between[]` rules on existing files as a drive-by "fix"; they were rarely followed to begin with outside the specific lineages noted above.

---

## 5. Known Recurring Bugs — Don't Copy These

These appeared independently across multiple files/authors, which is what makes them worth calling out rather than dismissing as one-off typos.

- **`function atexit()` instead of `atom atexit()`.** LavishScript only auto-invokes `atexit` when declared as an `atom`. **Confirmed corpus-wide: 55 files** (against 318 correctly using `atom`) — their cleanup code (UI checkbox resets, event detaches) likely never runs. Always declare it as `atom atexit()` in new code; worth fixing if you're already editing one of the 55.
- **Missing braces on multi-statement `if` bodies.** LavishScript only governs the single next statement without `{ }`. Seen at least twice independently from the same author ("Kordulek"): a zone-in guard (`if !${Zone.Name.Equals[...]}`) without braces let the zone-travel `RunScript` call and its wait-loop run unconditionally on every invocation, not just when actually needed. Always brace multi-line `if` bodies.
- **A repeated missing closing quote on `#include`/`#includeoptional`.** Found independently in at least three different files/zones (`Lyceum_of_the_Recondite.iss`, `Covenant_District.iss`, `EQ2OgreTradeskillApprenticeManager.iss`) — `#includeoptional "${LavishScript.HomeDirectory}/Scripts/.../file.iss` missing its trailing `"`. Easy to miss visually; worth double-checking any `#include` line you write or copy.
- **Silent method-shadowing from duplicate method names.** `Object_EQ2Chars.iss` and `Object_UplinkInfo.iss` (a template-derived pair) each define `method ClearUI()` **twice** in the same `objectdef` with two unrelated bodies — the second silently wins, so a caller invoking `This:ClearUI` doesn't get the behavior the first definition implies. Check for duplicate method names when an objectdef isn't behaving as one of its methods suggests it should.
- **Dead "placeholder" mechanic atoms.** A recurring template (same author, multiple zone files) defines a boss-mechanic atom (`Joust`, `Intercept`, `Bleat`) that checks for the literal text `"placeholder"` with its `AttachAtom` call commented out — scaffolding that was never finished. If a mechanic matters in a file like this, it needs actual implementation, not more copying.
- **Stale copy-pasted file headers.** Several files' documentation header (Script Name / File Path, or even the boss-failure message text) doesn't match the actual file/content — evidence of copy-paste-without-updating. If you copy a file as a starting point, update its header and any embedded self-referential strings.

---

## 6. Corpus Caveats

- A handful of files under `Documentation/LERN/` are **tutorial snippets**, not field-tested production code — good for canonical LavishScript/LGUI2 syntax, not for "this is how OgreBot does it in practice."
- Several files are self-admitted work-in-progress (explicit comments like "NOT FINISHED YET" or a whole function whose real logic is commented out) or literal empty stubs — don't treat everything in the folder as a finished reference. A few (`status.iss`, `guild_harvesters.iss`, `sod_phobis.iss`) have their stated purpose entirely commented out, leaving only argument validation.
- `repeater.iss`, `keymapper.iss`, `isboxer.iss`, `isboxerui.iss` are generic InnerSpace/ISBoxer add-on code, not EQ2/OgreBot-specific — useful only as general LavishScript style examples (they have zero `OgreBotAPI`/`ISXEQ2` references, confirmed by grep), not OgreBot convention evidence.
- The corpus contains duplicate/near-duplicate files across `EQ2OgreBot` and `EQ2RAW` trees (the same zone content mirrored in both locations) — if editing one, check whether a duplicate exists elsewhere.
- At least one file (`OgreMCP_Buttons.iss`) is a personal plain-English notes/snippet collection saved with a `.iss` extension — not intended to run as a script.
