# CampSpot System Reference

The CampSpot system controls character positioning in OgreBot. It defines where characters should stand and return to between combat actions. Understanding this system is essential for writing encounter modules, since nearly every boss fight involves positioning your group.

The system has three layers:

1. **`Ogre_CampSpot` Object** - Local object methods (runs on each session)
2. **`OgreBotAPI` Methods** - API wrapper methods (local or cross-session)
3. **`oc` Commands** - Cross-session commands via OgreConsole

---

## Key Concepts

### CampSpot Must Be Enabled First

You must set a campspot with `-CampSpot` before `-ChangeCampSpotWho` commands will work. Characters ignore campspot change commands if they do not have a campspot enabled.

> **:warning: Warning**
>
> If your campspot change commands seem to be doing nothing, verify that `-CampSpot` was called first.

### Absolute vs Relative CampSpot

- **Absolute CampSpot** (`AbsoluteCampSpotLoc`) - The primary/base campspot set by `-CampSpot` or `-ChangeCampSpotWho`
- **Relative CampSpot** (`RelativeCampSpotLoc`) - A temporary override position. When set, characters move here instead of the absolute campspot.
- **Relative Offset** (`RelativeLoc`) - The offset or absolute position used for relative positioning

> **:memo: Note**
>
> When you clear the relative campspot, characters return to the **current** absolute campspot -- not where it was when relative was set. This allows individuals to temporarily break away and automatically rejoin wherever the group moved to.

### HowClose and HowFar

- **HowClose** (default: 2) - Distance from campspot before the character is considered "at" the campspot. When within this range, the character stops moving toward the campspot.
- **HowFar** (default: 200) - Sanity check distance. If a new campspot command would place the campspot beyond this distance from the character, the command is ignored. Prevents accidental movement across the zone.

> **:bulb: Tip**
>
> Setting HowClose too low (like 1) can cause rubberbanding -- characters constantly move back and forth trying to stay exactly at the campspot. The default of 2 works well for most situations.

### Enable/Disable System

The campspot inherits from `Object_Movement` which provides a collection-based enable/disable system:

- Multiple systems can disable campspot with a named "reason"
- Campspot only works when ALL reasons are cleared
- Prevents conflicts between joust, movement scripts, etc.

---

## Command Reference Table

### Cross-Session (oc) Commands

| Command | Description | Example |
|---------|-------------|---------|
| `-CampSpot <who> [close] [far]` | Set campspot at current location (alias: `-cs`) | `oc !c -CampSpot igw:${Me.Name}` |
| `-ChangeCampSpotWho <who> X Y Z` | Move campspot to coordinates | `oc !c -ChangeCampSpotWho igw:${Me.Name} 100.5 25.0 -50.3` |
| `-ccsw <who> X Y Z` | Alias for ChangeCampSpotWho | `oc !c -ccsw igw:${Me.Name} 100.5 25.0 -50.3` |
| `-CS_Set_Relative <who> X Y Z` | Set relative offset | `oc !c -CS_Set_Relative ${Me.Name} 100.0 50.0 200.0` |
| `-CS_Set_ChangeCampSpotBy <who> X Y Z` | Add offset to campspot | `oc !c -CS_Set_ChangeCampSpotBy igw:${Me.Name} 5 0 -3` |
| `-ChangeCampSpotBy <who> X Y Z` | Alias for above | `oc !c -ChangeCampSpotBy igw:${Me.Name} 5 0 -3` |
| `-CS_Set_ChangeRelativeCampSpotBy <who> X Y Z` | Add offset to relative campspot | `oc !c -CS_Set_ChangeRelativeCampSpotBy igw:${Me.Name} 5 0 -3` |
| `-CS_ClearCampSpot <who>` | Clear campspot | `oc !c -CS_ClearCampSpot igw:${Me.Name}` |
| `-CS_ClearRelative <who>` | Clear relative offset only | `oc !c -CS_ClearRelative igw:${Me.Name}` |
| `-CS_ClearAll <who>` | Clear everything | `oc !c -CS_ClearAll igw:${Me.Name}` |
| `-CS_HowClose <who> <distance>` | Set minimum distance | `oc !c -CS_HowClose igw:${Me.Name} 2` |
| `-CS_HowFar <who> <distance>` | Set maximum distance | `oc !c -CS_HowFar igw:${Me.Name} 30` |
| `-CS_Set_Formation_Circle <who> ...` | Circle formation | `oc !c -CS_Set_Formation_Circle igw:${Me.Name} 8 100 50 200 FALSE` |
| `-CS_Set_Formation_MonkeyInMiddle <who> ...` | One in middle formation | `oc !c -CS_Set_Formation_MonkeyInMiddle igw:${Me.Name} 13 ${Me.X} ${Me.Y} ${Me.Z} ${Me.Name}` |
| `-CS_Set_Formation_MonkeyInMiddleRaid <who> ...` | Raid formation with configurable spots | `oc !c -CS_Set_Formation_MonkeyInMiddleRaid igw:${Me.Name} fighter 10 100 50 200 5 FALSE` |
| `-SetCS_PositionNPC <who> <name/ID> [dist] [skip]` | Fighters in front, others behind NPC | `oc !c -SetCS_PositionNPC igw:${Me.Name} "Boss Name" 5` |
| `-SetCS_InFrontNPC <who> <name/ID> [dist] [skip]` | Everyone in front of NPC (180 degrees) | `oc !c -SetCS_InFrontNPC igw:${Me.Name} "Boss Name" 5` |
| `-SetCS_BehindNPC <who> <name/ID> [dist] [skip]` | Everyone behind NPC (0 degrees) | `oc !c -SetCS_BehindNPC igw:${Me.Name} "Boss Name" 5` |
| `-SetCS_NPC <who> <angle> <name/ID> [dist] [skip]` | Custom angle relative to NPC | `oc !c -SetCS_NPC igw:${Me.Name} 90 "Boss Name" 5` |

### Local Object (Ogre_CampSpot) Methods

| Method | Description | Example |
|--------|-------------|---------|
| `:Set_CampSpot[who, close, far]` | Set at current location | `Ogre_CampSpot:Set_CampSpot["all", 2, 200]` |
| `:Set_CS[who, close, far]` | Alias for Set_CampSpot | `Ogre_CampSpot:Set_CS["all", 2, 200]` |
| `:Set_CCS[X, Y, Z]` | Change to coordinates (local) | `Ogre_CampSpot:Set_CCS[100.5, 25.0, -50.3]` |
| `:Set_ChangeCampSpot[who, X, Y, Z]` | Change with targeting | `Ogre_CampSpot:Set_ChangeCampSpot["all", 100.5, 25.0, -50.3]` |
| `:Set_Relative[who, X, Y, Z]` | Set relative offset | `Ogre_CampSpot:Set_Relative["all", 10, 0, 5]` |
| `:ClearCampSpot[who]` | Clear campspot | `Ogre_CampSpot:ClearCampSpot["all"]` |
| `:ClearRelative[who]` | Clear relative only | `Ogre_CampSpot:ClearRelative["all"]` |
| `:ClearAll[who]` | Clear everything | `Ogre_CampSpot:ClearAll["all"]` |
| `:Disable[reason]` | Disable with named reason | `Ogre_CampSpot:Disable["Joust"]` |
| `:Enable[reason]` | Re-enable (remove reason) | `Ogre_CampSpot:Enable["Joust"]` |
| `:Set_HowClose[who, dist]` | Set minimum distance | `Ogre_CampSpot:Set_HowClose["all", 2]` |
| `:Set_HowFar[who, dist]` | Set maximum distance | `Ogre_CampSpot:Set_HowFar["all", 30]` |

### Local Object (Ogre_CampSpot) Members

| Member | Description | Example |
|--------|-------------|---------|
| `.AtCampSpot` | TRUE if within HowClose of Get_CampSpot | `if ${Ogre_CampSpot.AtCampSpot}` |
| `.Get_CampSpot` | Where character moves to (relative if set, otherwise absolute) | `${Ogre_CampSpot.Get_CampSpot.XYZ[" "]}` |
| `.Get_AbsoluteCampSpot` | Primary campspot (before relative offset) | `${Ogre_CampSpot.Get_AbsoluteCampSpot.X}` |
| `.Get_RelativeMods` | The relative offset values (0,0,0 when no offset) | `${Ogre_CampSpot.Get_RelativeMods}` |
| `.Get_X`, `.Get_Y`, `.Get_Z` | Individual coordinates of Get_CampSpot | `${Ogre_CampSpot.Get_X}` |
| `.Get_HowClose`, `.Get_HowFar` | Current HowClose and HowFar values | `${Ogre_CampSpot.Get_HowClose}` |
| `.DistanceFromCampSpot` | Distance from Get_CampSpot | `${Ogre_CampSpot.DistanceFromCampSpot}` |
| `.DistanceFromAbsoluteCampSpot` | Distance from Get_AbsoluteCampSpot | `${Ogre_CampSpot.DistanceFromAbsoluteCampSpot}` |
| `.Disabled[reason]` | TRUE if disabled by named reason | `if ${Ogre_CampSpot.Disabled["Joust"]}` |
| `.UsingRelative` | TRUE if Get_CampSpot differs from Get_AbsoluteCampSpot | `if ${Ogre_CampSpot.UsingRelative}` |

---

## Cross-Session Commands (oc)

Use these in Instance Controllers and fight scripts for group coordination.

> **:bulb: Tip**
>
> Examples use `oc !c` (visible) for readability. Switch to `oc !ci` (invisible) once your code is working to reduce console spam.

### Setting CampSpot at Current Location

```lavishscript
; Set campspot at current location for everyone (recommended - uses defaults)
oc !c -CampSpot igw:${Me.Name}

; Defaults: HowClose=2, HowFar=200
; Only specify values when defaults don't work for your situation

; With custom HowClose and HowFar - only when needed
oc !c -CampSpot igw:${Me.Name} 3 100

; For specific character
oc !c -CampSpot ${Me.Name}
```

### Changing CampSpot to Specific Coordinates

```lavishscript
; Move everyone to specific X Y Z coordinates
oc !c -ChangeCampSpotWho igw:${Me.Name} 100.5 25.0 -50.3

; Shorthand alias
oc !c -ccsw igw:${Me.Name} 100.5 25.0 -50.3

; Move fighters to one spot, others to another
oc !c -ChangeCampSpotWho "igw:${Me.Name}+fighter" 100.0 10.0 50.0
oc !c -ChangeCampSpotWho "igw:${Me.Name}+notfighter" 95.0 10.0 55.0

; Move to current leader's position
oc !c -ChangeCampSpotWho igw:${Me.Name} ${Me.X} ${Me.Y} ${Me.Z}

; Move to mob's location (use space separator for XYZ)
oc !c -ChangeCampSpotWho igw:${Me.Name} ${Actor[NamedMob].Loc.XYZ[" "]}
```

### Using point3f Variables

```lavishscript
variable point3f MySpot
MySpot:Set[100.5, 25.0, -50.3]

; Using XYZ with space separator (preferred)
oc !c -ChangeCampSpotWho igw:${Me.Name} ${MySpot.XYZ[" "]}

; Using individual components (when you need to modify a value, e.g., Y+5)
oc !c -ChangeCampSpotWho igw:${Me.Name} ${MySpot.X} ${Math.Calc[${MySpot.Y}+5]} ${MySpot.Z}
```

### Relative CampSpot (Temporary Override)

Relative campspot sets a **temporary** position without losing the primary campspot. When cleared, characters return to the **current** primary campspot -- even if it changed while they were away.

**Use Case:** Individual characters need to temporarily break from the group (joust, grab add, handle mechanic) then automatically rejoin wherever the group is now.

```lavishscript
; Move character to temporary location (primary campspot preserved)
oc !c -CS_Set_Relative ${Me.Name} ${JoustSpot.XYZ[" "]}

; Clear relative - returns to current primary campspot
oc !c -CS_ClearRelative ${Me.Name}
```

**Example - Joust then return:**

```lavishscript
; Character gets debuff, needs to move away
oc !c -CS_Set_Relative ${Me.Name} ${JoustSpot.XYZ[" "]}
wait 60  ; wait for debuff

; Return to wherever group is now
oc !c -CS_ClearRelative ${Me.Name}
```

**Relative offset commands:**

```lavishscript
; Set relative to absolute coordinates
oc !c -CS_Set_Relative igw:${Me.Name} 100.0 50.0 200.0

; Change campspot by offset (adds to current position)
oc !c -CS_Set_ChangeCampSpotBy igw:${Me.Name} 5 0 -3
oc !c -ChangeCampSpotBy igw:${Me.Name} 5 0 -3

; Change relative offset by amount
oc !c -CS_Set_ChangeRelativeCampSpotBy igw:${Me.Name} 5 0 -3
```

### Clearing CampSpot

```lavishscript
; Clear campspot for group (disables dynamic campspot checkbox)
oc !c -CS_ClearCampSpot igw:${Me.Name}

; Clear relative offset only (keeps absolute position)
oc !c -CS_ClearRelative igw:${Me.Name}

; Clear everything (campspot, relative, zone tracking)
oc !c -CS_ClearAll igw:${Me.Name}
```

### Setting HowClose and HowFar

```lavishscript
; Set HowClose - distance from campspot to be considered "at" campspot
oc !c -CS_HowClose igw:${Me.Name} 2

; Set HowFar - sanity check, campspot changes beyond this distance are ignored
oc !c -CS_HowFar igw:${Me.Name} 30
```

### Formation Commands

```lavishscript
; Circle formation around a point
; Parameters: ForWho, Distance, X, Y, Z, UseRelative
oc !c -CS_Set_Formation_Circle igw:${Me.Name} 8 100 50 200 FALSE

; Monkey in the middle (one person center, others around)
; Parameters: ForWho, Distance, X, Y, Z, MiddleMan, UseRelative
oc !c -CS_Set_Formation_MonkeyInMiddle igw:${Me.Name} 13 ${Me.X} ${Me.Y} ${Me.Z} ${Me.Name}

; Raid version with configurable spots
; Parameters: ForWho, MiddleForWho, Distance, X, Y, Z, CircleSpots, UseRelative
oc !c -CS_Set_Formation_MonkeyInMiddleRaid igw:${Me.Name} fighter 10 100 50 200 5 FALSE
```

### NPC-Relative Positioning

Position characters relative to an NPC's location and facing. Useful for mechanics requiring front/behind positioning.

**Angle system:** 180 degrees = in front of NPC, 0 degrees = behind NPC

```lavishscript
; SetCS_PositionNPC - Fighters in front (180), non-fighters behind (0)
; Parameters: ForWho, NameOrID, Distance (default 3), SkipIfAggro (default FALSE)
oc !c -SetCS_PositionNPC igw:${Me.Name} "Boss Name" 5
oc !c -SetCS_PositionNPC igw:${Me.Name} ${Actor[${MobName}].ID} 4

; SetCS_InFrontNPC - Everyone in front of NPC (180)
; Parameters: ForWho, NameOrID, Distance, SkipIfAggro
oc !c -SetCS_InFrontNPC igw:${Me.Name} "Boss Name" 5

; SetCS_BehindNPC - Everyone behind NPC (0)
; Parameters: ForWho, NameOrID, Distance, SkipIfAggro
oc !c -SetCS_BehindNPC igw:${Me.Name} "Boss Name" 5

; SetCS_NPC - Custom angle (0-360)
; Parameters: ForWho, Angle, NameOrID, Distance, SkipIfAggro
oc !c -SetCS_NPC igw:${Me.Name} 90 "Boss Name" 5
```

**Role-based positioning example:**

```lavishscript
; Fighters tank from front, everyone else behind
oc !c -SetCS_InFrontNPC "igw:${Me.Name}+fighter" "${MobName}" 3
oc !c -SetCS_BehindNPC "igw:${Me.Name}+-fighter" "${MobName}" 5
```

---

## Local Object Methods (Ogre_CampSpot)

Use these for local-only operations within OgreBot or helper scripts.

### Setting CampSpot

```lavishscript
; Set campspot at current location
Ogre_CampSpot:Set_CampSpot["all", 1, 125]
Ogre_CampSpot:Set_CS["all", 2, 200]

; Change to specific coordinates (local only)
Ogre_CampSpot:Set_CCS[100.5, 25.0, -50.3]
Ogre_CampSpot:Set_CCS[${Me.Loc}]
Ogre_CampSpot:Set_CCS[${Actor[BossName].Loc}]

; Change with ForWho targeting
Ogre_CampSpot:Set_ChangeCampSpot["all", 100.5, 25.0, -50.3]
```

### Relative Positioning

```lavishscript
; Set relative offset
Ogre_CampSpot:Set_Relative["all", 10, 0, 5]

; Change campspot by offset
Ogre_CampSpot:Set_ChangeCampSpotBy["all", 5, 0, -3]

; Clear relative
Ogre_CampSpot:ClearRelative["all"]
```

### Clearing

```lavishscript
; Clear campspot
Ogre_CampSpot:ClearCampSpot["all"]

; Clear everything
Ogre_CampSpot:ClearAll["all"]
```

### Enable/Disable (Movement Lock)

```lavishscript
; Disable campspot movement with a named reason
Ogre_CampSpot:Disable["Joust"]
Ogre_CampSpot:Disable["MyFightScript"]

; Re-enable (remove the reason)
Ogre_CampSpot:Enable["Joust"]
Ogre_CampSpot:Enable["MyFightScript"]

; Check if disabled by a specific reason
if ${Ogre_CampSpot.Disabled["Joust"]}
    echo Campspot disabled by Joust
```

### HowClose and HowFar

```lavishscript
Ogre_CampSpot:Set_HowClose["all", 2]
Ogre_CampSpot:Set_HowFar["all", 30]
```

### Reading CampSpot Values

**Understanding Get_CampSpot vs Get_AbsoluteCampSpot:**

- `Get_CampSpot` - Where the character will actually move to. Returns the relative position if set, otherwise the absolute position.
- `Get_AbsoluteCampSpot` - The primary/base campspot, before any relative offset is applied.
- When no relative is set, both return the same value.

```lavishscript
; Get where character will actually move to
; (returns relative position if set, otherwise absolute)
${Ogre_CampSpot.Get_CampSpot}           ; Returns point3f
${Ogre_CampSpot.Get_CampSpot.X}         ; X coordinate
${Ogre_CampSpot.Get_CampSpot.XYZ[" "]}  ; "X Y Z" string

; Get the primary campspot (ignores relative offset)
; Only different from Get_CampSpot when using relative
${Ogre_CampSpot.Get_AbsoluteCampSpot}

; Get relative offset values
${Ogre_CampSpot.Get_RelativeMods}

; Get individual components
${Ogre_CampSpot.Get_X}
${Ogre_CampSpot.Get_Y}
${Ogre_CampSpot.Get_Z}

; Get HowClose and HowFar values
${Ogre_CampSpot.Get_HowClose}
${Ogre_CampSpot.Get_HowFar}
```

### Position Checking

```lavishscript
; Check if at campspot (within HowClose distance of Get_CampSpot)
if ${Ogre_CampSpot.AtCampSpot}
    echo At campspot

; Get distance from where character will move (Get_CampSpot)
${Ogre_CampSpot.DistanceFromCampSpot}

; Get distance from primary campspot (ignores relative)
${Ogre_CampSpot.DistanceFromAbsoluteCampSpot}

; Check if using relative positioning
; Returns TRUE if Get_CampSpot differs from Get_AbsoluteCampSpot
${Ogre_CampSpot.UsingRelative}

; More specific checks (used internally by UsingRelative)
${Ogre_CampSpot.UsingRelativeCS}      ; TRUE if positions differ
${Ogre_CampSpot.UsingRelativeByCS}    ; TRUE if offset values are non-zero
```

### Waiting for CampSpot

```lavishscript
; Wait until character reaches campspot (with optional initial delay)
call Ogre_CampSpot.HandleWaitForCampSpot 10
; Returns TRUE if at campspot, FALSE if too far or dead
```

### 3D Distance Mode

```lavishscript
; Enable 3D distance checking (includes Y axis)
Ogre_CampSpot:Set_Use3D[TRUE]

; Check current mode
${Ogre_CampSpot.Get_Use3D}
```

---

## OgreBotAPI Methods

```lavishscript
; Change campspot (local)
OgreBotAPI:ChangeCampSpot[100.5, 25.0, -50.3]

; Change campspot with ForWho targeting
OgreBotAPI:ChangeCampSpotWho["igw:${Me.Name}", 100.5, 25.0, -50.3]
OgreBotAPI:ChangeCampSpotWho["@Healer1", 95.0, 10.0, 55.0]
```

---

## Common Patterns

### Fight Script Setup

```lavishscript
function DoSetup()
{
    if ${OgreBotAPI.IsGroupLeader}
    {
        ; Set initial campspot at current position
        oc !ci -CampSpot igw:${Me.Name} 2
    }
}
```

### Moving Group to Named Position

```lavishscript
variable point3f MySpot

function MoveToPosition()
{
    if ${OgreBotAPI.IsGroupLeader}
    {
        MySpot:Set[150.0, 10.0, -200.0]
        oc !ci -ChangeCampSpotWho igw:${Me.Name} ${MySpot.XYZ[" "]}
    }
}
```

### Tank vs Group Positioning

```lavishscript
function PositionForFight()
{
    if ${OgreBotAPI.IsGroupLeader}
    {
        ; Tank in front
        oc !ci -ChangeCampSpotWho "igw:${Me.Name}+fighter" 155.0 10.0 -195.0

        ; Everyone else behind
        oc !ci -ChangeCampSpotWho "igw:${Me.Name}+notfighter" 145.0 10.0 -205.0
    }
}
```

### Waiting for Group to Arrive

```lavishscript
function MoveAndWait()
{
    oc !ci -ChangeCampSpotWho igw:${Me.Name} ${MySpot.XYZ[" "]}

    ; Wait for self to arrive
    wait 100 ${Ogre_CampSpot.AtCampSpot}

    ; Or use the helper function
    call Ogre_CampSpot.HandleWaitForCampSpot 10
}
```

### Temporary Movement Lock (Jousting)

```lavishscript
function JoustOut()
{
    ; Disable campspot return
    Ogre_CampSpot:Disable["MyJoust"]

    ; Move away
    ; ... movement code ...

    ; Wait
    wait 30

    ; Re-enable campspot
    Ogre_CampSpot:Enable["MyJoust"]
}
```

---

## Related Documentation

- [CampSpot Jump System](campspot-jump.md) - Automated jump navigation across gaps and platforms
- [OgreBotAPI Reference](ogrebot-api.md) - Movement and positioning methods on OgreBotAPI
- [Coding Practices](coding-practices.md) - Variable naming conventions and code style
