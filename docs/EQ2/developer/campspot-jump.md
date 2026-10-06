# CampSpot Jump System

The CampSpot Jump system automates navigating characters across gaps, ledges, or platforms that require a precisely-timed jump. It runs as a background script that takes over character movement, walks to a start point, runs toward an end point, and triggers a jump at the exact right location.

---

## Overview

Normal campspot movement walks characters in a straight line to a destination. This does not work when the path requires jumping across a gap or off a ledge. The CampSpot Jump system solves this by defining a four-point path:

1. **Start Point** -- Walk here first to line up the approach
2. **Jump Point** -- When the character reaches this location, trigger a jump
3. **End Point** -- The destination after the jump (stop moving when reached)
4. **Completed Point** -- Where to set the campspot after the jump is finished (defaults to End Point if not specified)

The system temporarily disables campspot movement, takes direct control of the character using forward key press and facing commands, executes the jump, then re-enables campspot at the Completed Point.

---

## How It Works

1. Campspot movement is **disabled** (reason: `"CSJump"`) so it does not interfere
2. The character **faces and walks toward the Start Point**
3. Once within 2 units of the Start Point, the character **faces the End Point** and begins running forward
4. When within **JumpDistance** (default: 2) of the Jump Point, `OgreBotAPI:Jump` is called
5. Once within 2 units of the End Point, forward movement is **released**
6. Campspot is **set to the Completed Point** and re-enabled
7. A **MaxTimer** (default: 10 seconds) acts as a safety timeout -- if the jump is not completed in time, the system resets

> **:warning: Warning**
>
> The character must not be following another character when a jump starts. The system automatically calls `/stopfollow` if needed.

---

## Cross-Session Command (oc)

The primary way to use CampSpot Jump in encounter modules and instance controllers.

### Syntax

```
oc !c -Set_CampSpotJump_Handle <ForWho> <Start> <Jump> <End> <Completed> [options]
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ForWho` | string | Standard ForWho targeting (e.g., `igw:${Me.Name}`, `${Me.Name}`) |
| `Start` | point3f | The approach/lineup position (X,Y,Z) |
| `Jump` | point3f | Where to trigger the jump (X,Y,Z) |
| `End` | point3f | Destination after the jump (X,Y,Z) |
| `Completed` | point3f | Where to set campspot after completion (X,Y,Z). Use `0,0,0` to default to End |
| `options` | variadic | Optional flags: `-JumpDistance <float>`, `-MaxTimer <int>` |

Coordinates can be passed as either:

- **Comma-separated point3f**: `"X,Y,Z"` (e.g., `"-157.38,-24.15,42.65"`)
- **Space-separated components**: `X Y Z` (e.g., `-157.38 -24.15 42.65`)

### Basic Example

```lavishscript
; Jump across a gap - everyone in the group
oc !ci -Set_CampSpotJump_Handle igw:${Me.Name} "-157.38,-24.15,42.65" "-157.18,-24.40,38.73" "-156.81,-25.48,31.01" "-158.37,-25.35,32.25"
```

### With Optional Parameters

```lavishscript
; Jump with custom jump distance and timeout
oc !ci -Set_CampSpotJump_Handle igw:${Me.Name} "Start" "Jump" "End" "Completed" -JumpDistance 1 -MaxTimer 15
```

### Chaining Multiple Jumps

```lavishscript
; Navigate across a series of platforms
; Each jump's Start point = previous jump's Completed point
oc !ci -Set_CampSpotJump_Handle igw:${Me.Name} "-157.38,-24.15,42.65" "-157.18,-24.40,38.73" "-156.81,-25.48,31.01" "-158.37,-25.35,32.25"
wait 25

oc !ci -Set_CampSpotJump_Handle igw:${Me.Name} "-158.37,-25.35,32.25" "-155.42,-25.35,29.64" "-149.30,-26.56,24.05" "-149.02,-26.56,23.97"
wait 25

oc !ci -Set_CampSpotJump_Handle igw:${Me.Name} "-149.02,-26.56,23.97" "-147.03,-26.44,21.59" "-141.75,-27.77,16.19" "-143.97,-27.65,18.21"
wait 25
```

> **:bulb: Tip**
>
> Add a wait between chained jumps to let the previous jump fully settle before starting the next one.

---

## Local Object (Obj_CampSpotJump)

For direct access within OgreBot or raid modules. The `Obj_CampSpotJump` global object is created when the `Ogre_CampSpotJump.iss` script is launched.

### Methods

| Method | Description |
|--------|-------------|
| `:Handle[Start, Jump, End, Completed]` | Execute a jump with four point3f coordinates. If Completed is `0,0,0`, defaults to End |
| `:HandleAdvanced[Start, Jump, End, Completed, ...Args]` | Execute a jump with optional `-JumpDistance` and `-MaxTimer` flags |
| `:Stop[]` | Cancel an in-progress jump |
| `:Set_JumpDistance[float]` | Set how close to the Jump Point before jumping (default: 2) |
| `:Set_MaxTimer[int]` | Set timeout in seconds (default: 10) |

### Members

| Member | Type | Description |
|--------|------|-------------|
| `.Active` | bool | TRUE if a jump is currently in progress |

### Example -- Direct Object Usage

```lavishscript
; From a raid module - manually launch the script and configure
ogre ogre_campspotjump
Obj_CampSpotJump:Set_JumpDistance[1]
Obj_CampSpotJump:Set_MaxTimer[10]

; Execute the jump
Obj_CampSpotJump:Handle["-22.44,182.43,-419.56", "-14.99,182.42,-423.75", "-12.25,183.71,-425.55"]
```

### Script Launching

The jump script runs as a background script. When invoked via the `oc` command or `OgreBotAPI:Set_CampSpotJump_Handle`, the system automatically launches it. To launch it manually:

```lavishscript
ogre ogre_campspotjump
```

To stop it:

```lavishscript
ogre end ogre_campspotjump
```

---

## Events

| Event | Description |
|-------|-------------|
| `CampSpotJump_Completed` | Fires when a jump finishes. Attach to this event to perform actions after a jump completes. |

### Example -- Event Handler

```lavishscript
; In your encounter setup
Event[CampSpotJump_Completed]:AttachAtom[This:CampSpotJump_Completed]

method CampSpotJump_Completed()
{
    ; Jump finished - do something
    OgreBotDebug:Echo["CampSpotJump_Completed", "Jump navigation complete"]
}
```

> **:memo: Note**
>
> The `CampSpotJump_Completed` event must be registered with `LavishScript:RegisterEvent[CampSpotJump_Completed]` before it can be used. The `Obj_CampSpotJump.Initalize` method handles this registration.

---

## Safety Features

- **MaxTimer timeout** -- If the jump is not completed within the timeout (default 10 seconds), the system deactivates and re-enables campspot. Prevents characters from running endlessly.
- **Death/Zoning check** -- If the character dies or zones during a jump, the system waits for recovery then resets.
- **Campspot restore** -- On cleanup, the campspot is set to the Completed Point so the character returns to the correct location.
- **Follow cancellation** -- Automatically stops following before starting a jump.

---

## Tips for Getting Coordinates

To set up a CampSpot Jump path, you need precise coordinates for all four points. Here is a recommended approach:

1. **Manually walk the path** with a single character
2. At each key position (start, jump trigger, landing, final), note your coordinates using `${Me.X} ${Me.Y} ${Me.Z}` or `${Me.Loc}`
3. The **Start Point** should be far enough back to give the character time to line up and build momentum
4. The **Jump Point** should be right at the edge where you need to jump
5. The **End Point** should be on the other side of the gap, where you land
6. The **Completed Point** can be the same as End, or a safer position slightly further from the edge

> **:bulb: Tip**
>
> When chaining multiple jumps, use the previous jump's Completed Point as the next jump's Start Point for smooth transitions.

---

## Related Documentation

- [CampSpot System](camp-spot.md) -- Core campspot positioning system
- [OgreBotAPI Reference](ogrebot-api.md) -- Movement and positioning methods
