# Space Shooter UNIR – Feature Implementation Plan

> Engine: **Unreal Engine 5.4** · Project type: **Blueprint only** · Time budget: **4 days × 4 h = 16 h**

This document describes every new feature, **which Blueprint it lives in**, **which variables / functions / events to create**, and **how the pieces connect**.

> ⚠️ Names such as `Progress`, `Level`, `EnemiesDestroyed`, `NumLifes`, `PlayerDamage`, `UpdateWidget`, `SpawnEnemy`, `SpawnTime`, `DelayShoot`, `Speed`… come from the project's current assets. The exact wiring of your existing nodes may differ slightly — when a step says *"after the node that does X"*, find that spot in your graph and plug in there.

---

## Table of contents

1. [Current project map](#1-current-project-map)
2. [Schedule (4 days)](#2-schedule-4-days)
3. [F1 – Display the current level](#f1--display-the-current-level)
4. [F2 – Save best score & best level (SaveGame) and show them on the HUD](#f2--save-best-score--best-level-savegame-and-show-them-on-the-hud)
5. [F3 – Pause with the **P** key](#f3--pause-with-the-p-key)
6. [F4 – Quit with the **Q** key](#f4--quit-with-the-q-key)
7. [F5 – Difficulty scaling per level](#f5--difficulty-scaling-per-level) *(extra)*
8. [F6 – Power-ups: Shield, Triple Shot, Extra Life](#f6--power-ups-shield-triple-shot-extra-life) *(extra)*
9. [F7 – Enemy health system + 3rd enemy type ("Tank")](#f7--enemy-health-system--3rd-enemy-type-tank) *(extra)*
10. [F8 – Boss every N levels with health bar](#f8--boss-every-n-levels-with-health-bar) *(extra)*
11. [Final testing checklist](#11-final-testing-checklist)
12. [Common pitfalls](#12-common-pitfalls)
13. [F9 – Player loses a life when hit by an enemy bullet or when colliding with an enemy](#f9--player-loses-a-life-when-hit-by-an-enemy-bullet-or-when-colliding-with-an-enemy) *(bug fix, do it at the start of Day 2)*

---

## 1. Current project map

All game assets are in `Content/00_Space_Shooter/`.

| Asset | Type | What it does today | Relevant members |
|---|---|---|---|
| `SS_GameMode` | GameModeBase | Score by ticks, level up by kills | `Progress` (score), `Level`, `EnemiesDestroyed`, `IncrementPerSecond`, event `Enemy Destroyed` |
| `SS_PlayerController` | PlayerController | Empty | — |
| `UI/BP_SS_HUD` | HUD | Creates the game widget, forwards updates | var `GameWidget`, `UpdateWidget`, `GameOver` |
| `UI/BP_SS_GameWidget` | Widget | Lives, score, game-over overlay | `Overlay_Life` (Image1-3), `Overlay_Progress` (`ProgressTitle`, `ProgressValue`), `Overlay_GameOver`, functions `UpdateWidget`, `UpdateLifes`, `ShowGameOver`, *Any Key → Open Level MainMap* |
| `Player/BP_SS_Character` | Character (default pawn) | Movement, fire, lives | `NumLifes`, `CanFire`, `FireDelay`, `FireSpawn`, `PlayerDamage`, `Execute Fire`, key `Escape → Quit Game` |
| `Player/BP_Actor_Ammo` | Actor | Player bullet | `Speed`, explosion on hit |
| `Enemies/BP_EnemyBase` | Actor | Enemy type 1 (forward movement + shooting) | `Speed`, `MovementVector`, `DelayShoot`, `FirePoint`, dispatcher `OnEnemyDestroy` |
| `Enemies/BP_EnemyH` | Child of `BP_EnemyBase` | Enemy type 2 (lateral movement) | `Lateral`, fires `BP_EnemyAmmo2` |
| `Enemies/BP_RandomGenerator` | Actor | Spawns enemies on a looping timer | `SpawnTime`, `MaxNumEnemies`, `SpawnEnemy`, `SpawnPoint` components |
| `Input/IMC_Player` | Input Mapping Context | Movement + fire | `IA_Player_Movement`, `IA_Player_Fire` |

### New assets you'll create

```
Content/00_Space_Shooter/
├── Core/
│   └── BP_SS_SaveGame          (SaveGame)                 F2
├── Input/
│   ├── IA_Pause                (Input Action, bool)       F3
│   └── IA_Quit                 (Input Action, bool)       F4
├── PowerUps/
│   ├── E_PowerUpType           (Enumeration)              F6
│   ├── BP_PowerUp              (Actor)                    F6
│   ├── M_PowerUp               (Material)                 F6
│   └── M_Shield                (Material, translucent)    F6
└── Enemies/
    ├── BP_EnemyTank            (Child of BP_EnemyBase)    F7
    └── BP_EnemyBoss            (Child of BP_EnemyBase)    F8
```

---

## 2. Schedule (4 days)

| Day | Block | Feature | Est. |
|---|---|---|---|
| **Day 1** | 0:00–0:15 | Create the `Core`, `PowerUps` folders; commit a clean starting point | 0.25 h |
| | 0:15–0:45 | **F1** Level display | 0.5 h |
| | 0:45–2:15 | **F2** SaveGame + Best on HUD | 1.5 h |
| | 2:15–3:15 | **F3** Pause (P) | 1 h |
| | 3:15–3:45 | **F4** Quit (Q) | 0.5 h |
| | 3:45–4:00 | Test required features, commit | 0.25 h |
| **Day 2** | 0:00–1:00 | **F9** Player damage from enemy bullets & collisions (bug fix) | 1 h |
| | 1:00–2:30 | **F5** Difficulty scaling | 1.5 h |
| | 2:30–4:00 | **F6** Power-ups part 1: enum, `BP_PowerUp`, drop from enemies | 1.5 h |
| **Day 3** | 0:00–2:00 | **F6** Power-ups part 2: Extra Life, Shield, Triple Shot | 2 h |
| | 2:00–4:00 | **F7** Enemy health + `BP_EnemyTank` | 2 h |
| **Day 4** | 0:00–3:00 | **F8** Boss + health bar + spawn every N levels | 3 h |
| | 3:00–4:00 | Full playtest, tuning, packaging test, final commit | 1 h |

> 💡 F1–F4 are the **required** features: finish them on Day 1 so everything after is a bonus. **F9 is a gameplay bug fix:** do it first on Day 2, because you need working damage to test lives, the shield, Extra Life and game over. If you run late, drop F8 first, then F7.

> 💡 `.uasset` files are binary and can't be merged. **Commit after each feature** so you can roll back.

---

## F1 – Display the current level

**Goal:** show `LEVEL 3` on screen, updated whenever the player levels up.

**Blueprints touched:** `BP_SS_GameWidget`, `BP_SS_HUD`

### Step 1 — Widget (`BP_SS_GameWidget` → Designer)
1. Inside `Overlay_Progress` (or a new Overlay in the top-left corner), add a **Text** block.
2. Rename it **`LevelValue`** and tick **Is Variable**.
3. Default text: `LEVEL 1`.

### Step 2 — Widget (Graph): extend `UpdateWidget`
1. Open the function **`UpdateWidget`** → select the entry node → **Inputs → +** → add `NewLevel` (**Integer**).
2. Add at the end of the function:

```
… existing ProgressValue SetText …
  → LevelValue.SetText( Format Text "LEVEL {0}"  [ {0} ← NewLevel ] )
```

### Step 3 — HUD (`BP_SS_HUD`)
`UpdateWidget` now has a new pin. Wherever the HUD calls `GameWidget → UpdateWidget(Progress)` (probably on **Event Tick** after **Get Game Mode → Cast To SS_GameMode**), connect:

```
As SS Game Mode → Get Level → UpdateWidget.NewLevel
```

✅ **Test:** kill enemies until you level up; the text changes.

---

## F2 – Save best score & best level (SaveGame) and show them on the HUD

**Goal:** keep `BestScore` and `BestLevel` in a file that survives closing the game, show them on the HUD when the game starts, and update them when a run ends.

**File location:**
- Editor (PIE): `<Project>/Saved/SaveGames/SS_Records.sav`
- Packaged build on macOS: `~/Library/Application Support/Epic/Space_Shooter_UNIR/Saved/SaveGames/SS_Records.sav`

**Blueprints touched:** new `BP_SS_SaveGame`, `SS_GameMode`, `BP_SS_GameWidget`, `BP_SS_HUD`

### Step 1 — Create the SaveGame class
1. Content Browser → `Core/` → **Right click → Blueprint Class → All Classes → `SaveGame`** → name **`BP_SS_SaveGame`**.
2. Add variables:

| Variable | Type | Default |
|---|---|---|
| `BestScore` | Integer | 0 |
| `BestLevel` | Integer | 1 |

Compile & save. That's all it does; it's only a data container.

### Step 2 — GameMode (`SS_GameMode`): new variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `SaveSlotName` | String | `SS_Records` | File name |
| `SaveData` | `BP_SS_SaveGame` (Object Reference) | None | Cached save object |
| `bIsGameOver` | Boolean | false | Stops score, blocks pause (F3) |

### Step 3 — GameMode: function `GetSaveData` (loads lazily)

Loading lazily (when first needed) avoids **BeginPlay order** problems: the HUD may run BeginPlay before the GameMode has loaded the file.

- **Outputs:** `Result` (`BP_SS_SaveGame` ref)

```
GetSaveData
 ├─ Is Valid (SaveData)? ── true ──► Return SaveData
 └─ false
     └─ Does Save Game Exist (Slot = SaveSlotName, User Index = 0)
          ├─ true  → Load Game from Slot (SaveSlotName, 0)
          │           → Cast To BP_SS_SaveGame → SET SaveData
          └─ false → Create Save Game Object (Class = BP_SS_SaveGame)
                      → Cast To BP_SS_SaveGame → SET SaveData
     → Return SaveData
```

### Step 4 — GameMode: function `SaveRecords`

- **Outputs:** `bNewRecord` (Boolean)
- **Local variable:** `bChanged` (Boolean, default false)

```
SaveRecords
  Data = GetSaveData()
  Branch ( Progress > Data.BestScore )
     true → SET Data.BestScore = Progress ; SET bChanged = true
  Branch ( Level > Data.BestLevel )
     true → SET Data.BestLevel = Level   ; SET bChanged = true
  Branch ( bChanged )
     true → Save Game to Slot (Data, SaveSlotName, 0)
  Return bChanged
```

> To set a variable on another object, drag from the `Data` pin and type `Set BestScore`.

> Design choice: best score and best level are tracked separately, so they can come from different runs.

### Step 5 — GameMode: stop the score after game over
In **Event Tick**, put a `Branch (NOT bIsGameOver)` **before** the node that adds to `Progress`, so the score freezes on the game-over screen.

### Step 6 — Widget (`BP_SS_GameWidget`): show the records
1. **Designer:** add a Text block in the top-right corner → rename it **`BestValue`**, tick **Is Variable**, default `BEST 0  ·  LV 1`.
2. Inside `Overlay_GameOver`, add a Text block **`NewRecordText`** = `NEW RECORD!` (yellow), **Visibility = Collapsed**.
3. **Graph:** create function **`SetBest(BestScore: int, BestLevel: int)`**:

```
BestValue.SetText( Format Text "BEST {0}  ·  LV {1}"  [ {0} ← BestScore, {1} ← BestLevel ] )
```

4. Create function **`ShowNewRecord(bShow: bool)`**:

```
NewRecordText.SetVisibility( Select(bShow): Visible / Collapsed )
```

### Step 7 — HUD (`BP_SS_HUD`): connect everything

**On BeginPlay** (right after `Create Widget → Add to Viewport`):

```
Get Game Mode → Cast To SS_GameMode → GetSaveData
   → GameWidget.SetBest( Data.BestScore, Data.BestLevel )
```

**In the `GameOver` event/function** (the one that calls `ShowGameOver`):

```
Get Game Mode → Cast To SS_GameMode
   → SET bIsGameOver = true
   → SaveRecords → (bNewRecord)
   → GameWidget.ShowNewRecord( bNewRecord )
   → GetSaveData → GameWidget.SetBest( BestScore, BestLevel )   // refresh numbers
   → … existing ShowGameOver …
```

### Connection diagram

```
 BP_SS_HUD.BeginPlay ──► SS_GameMode.GetSaveData ──► BP_SS_SaveGame (file)
           │                                                   │
           └──────────── GameWidget.SetBest ◄──────────────────┘

 BP_SS_Character (lives = 0) ──► BP_SS_HUD.GameOver
           ├──► SS_GameMode.bIsGameOver = true
           ├──► SS_GameMode.SaveRecords ──► Save Game to Slot ──► SS_Records.sav
           └──► GameWidget.ShowNewRecord / SetBest / ShowGameOver
```

✅ **Test:** play, die → `NEW RECORD!`. Stop PIE, play again → the top-right shows the previous best. Check that `Saved/SaveGames/SS_Records.sav` exists. To reset the records, delete that file.

---

## F3 – Pause with the **P** key

**Goal:** P toggles pause; a "PAUSED" overlay is shown; the score, enemies and timers freeze. P must work **while paused** (that's the tricky part).

**Blueprints touched:** new `IA_Pause`, `IMC_Player`, `SS_PlayerController`, `BP_SS_HUD`, `BP_SS_GameWidget`

### Step 1 — Input Action
1. `Input/` → **Right click → Input → Input Action** → **`IA_Pause`**. Value Type: **Digital (bool)**.
2. ⚠️ In its Details tick **`Trigger when Paused`**. Without this, the key is ignored once the game is paused and you can never unpause.
3. Open **`IMC_Player`** → **Mappings +** → `IA_Pause` → key **P**.

### Step 2 — Widget: pause overlay (`BP_SS_GameWidget`)
1. **Designer:** add an **Overlay** named **`Overlay_Pause`** (Is Variable) covering the screen, with:
   - an **Image** (black, alpha 0.5) as background,
   - a **Text** `PAUSED` (big, centered),
   - a **Text** `P – Resume    Q – Quit` (small, below).
2. Set `Overlay_Pause` **Visibility = Collapsed**.
3. **Graph:** function **`ShowPause(bShow: bool)`** → `Overlay_Pause.SetVisibility( Select(bShow): Visible / Collapsed )`.

### Step 3 — HUD: forwarding function (`BP_SS_HUD`)
Function **`SetPauseVisible(bShow: bool)`** → `GameWidget.ShowPause(bShow)`.
(The controller talks to the HUD and the HUD talks to its widget. This matches how the project already uses `UpdateWidget` and `GameOver`.)

### Step 4 — PlayerController (`SS_PlayerController`)
The controller is the right owner for "meta" input (pause and quit), because it survives even if the pawn dies.

```
EnhancedInputAction IA_Pause  ── (Started pin) ──►
  Get Game Mode → Cast To SS_GameMode → Branch( bIsGameOver )
     true  → (do nothing)
     false → SET local bNewPaused = NOT( Is Game Paused )
             → Set Game Paused( bNewPaused )
             → Get HUD → Cast To BP_SS_HUD → SetPauseVisible( bNewPaused )
```

> Use the **Started** pin, not **Triggered**. With no trigger configured, Triggered fires every frame while the key is held and the game would flicker between paused and unpaused.

> `IMC_Player` is added in `BP_SS_Character`'s BeginPlay ("Setup Enhanced Input"). Because it is added to the **local player's** subsystem, the controller also receives the `IA_Pause` events. You don't need a second mapping context.

### What gets paused automatically
`Set Game Paused` stops **Tick** on all actors (GameMode score, enemy movement, bullets) and **all timers** (enemy spawner, fire delay, power-up durations). You don't need any extra logic.

✅ **Test:** P → overlay shown, everything frozen, score stops. P again → resumes. P on the game-over screen → nothing happens.

---

## F4 – Quit with the **Q** key

**Goal:** Q saves the records and closes the game, both while playing and while paused.

**Blueprints touched:** new `IA_Quit`, `IMC_Player`, `SS_PlayerController`, (`BP_SS_Character` cleanup), (`BP_SS_GameWidget` check)

### Step 1 — Input Action
1. Create **`IA_Quit`** (Digital bool) → tick **`Trigger when Paused`**.
2. `IMC_Player` → add mapping `IA_Quit` → key **Q**. *(Optional: also map **Escape** here and delete the old `Escape` key event in `BP_SS_Character`, so all exits save.)*

### Step 2 — PlayerController (`SS_PlayerController`)

```
EnhancedInputAction IA_Quit ── (Started) ──►
  Get Game Mode → Cast To SS_GameMode → SaveRecords
  → Quit Game ( Specific Player = Self, Quit Preference = Quit, Ignore Platform Restrictions = false )
```

> Saving on quit means a mid-run score also counts as a record. If you only want finished runs (game over) to count, remove `SaveRecords` here.

> In the editor (PIE), **Quit Game** just stops the play session. That's expected.

### Step 3 — Check the "Press any key to continue" logic
`BP_SS_GameWidget` uses an **Any Key** event to reload `MainMap` on game over. On the game-over screen, pressing **Q** could fire both *Quit* and *Restart*. In that Any Key handler, add:

```
Any Key (Key) → Branch( Key == Q  OR  Key == P )  → true: do nothing / false: existing OpenLevel logic
```

(Use **Equal (Key)** nodes and an **OR** node.)

✅ **Test:** Q while playing → closes; Q while paused → closes; reopen → best values were saved.

---

## F5 – Difficulty scaling per level

**Goal:** each level makes the game harder: enemies spawn faster, move faster and shoot more often, up to a cap.

**Blueprints touched:** `SS_GameMode`, `BP_RandomGenerator`, `BP_EnemyBase`

### Step 1 — GameMode: dispatcher + difficulty function
1. **Event Dispatcher** `OnLevelUp` with input `NewLevel` (Integer).
2. In **`Enemy Destroyed`**, right after the node that does `Level = Level + 1`, add **Call OnLevelUp(Level)**.
3. Variables: `DifficultyStep` (Float, 0.15), `MaxDifficulty` (Float, 2.5).
4. **Pure** function **`GetDifficulty`** → returns Float:

```
Return  Min( 1.0 + (Level - 1) * DifficultyStep ,  MaxDifficulty )
```

Level 1 → 1.0×, Level 5 → 1.6×, Level 11+ → 2.5× (cap).

### Step 2 — Enemies (`BP_EnemyBase`) read difficulty when spawned
At the **end of Event BeginPlay** (after the existing "Setup initial direction and Destroy Event" logic):

```
Get Game Mode → Cast To SS_GameMode → GetDifficulty → (D)
  SET Speed       = Speed * D
  SET DelayShoot  = Max( DelayShoot / D , 0.4 )
```

Every enemy spawned after a level-up is faster automatically, so the enemies never have to listen to the level-up event.

> ⚠️ `BP_EnemyH` (and the new children in F7/F8) **must call `Parent: BeginPlay`** in their own BeginPlay. Right-click the BeginPlay node → **Add Call to Parent Function**. Otherwise they skip this code.

### Step 3 — Spawner (`BP_RandomGenerator`) speeds up
1. New variables: `BaseSpawnTime` (Float), `MinSpawnTime` (Float, 0.4), `SpawnTimerHandle` (Timer Handle), `BaseMaxNumEnemies` (Integer).
2. **BeginPlay:**
   - `SET BaseSpawnTime = SpawnTime`, `SET BaseMaxNumEnemies = MaxNumEnemies`.
   - On the existing **Set Timer by Event** node (the looping one that calls `SpawnEnemy`), drag its **Return Value** → **Promote to variable** → `SpawnTimerHandle`.
   - `Get Game Mode → Cast To SS_GameMode → Bind Event to OnLevelUp` → custom event **`HandleLevelUp(NewLevel)`**.
3. **`HandleLevelUp`:**

```
HandleLevelUp(NewLevel)
  D = GameMode.GetDifficulty
  SET SpawnTime     = Max( BaseSpawnTime / D , MinSpawnTime )
  SET MaxNumEnemies = BaseMaxNumEnemies + (NewLevel - 1)          // only if your spawner uses this cap
  Clear and Invalidate Timer by Handle ( SpawnTimerHandle )
  Set Timer by Event ( Event = same SpawnEnemy delegate, Time = SpawnTime, Looping = true )
      → SET SpawnTimerHandle
```

(Copy and paste your existing *Create Event → SpawnEnemy* node so the new timer calls the same thing.)

### Optional polish (15 min)
Add a `LEVEL UP!` text with a **Widget Animation** (fade in and out) to `BP_SS_GameWidget`, plus an event `PlayLevelUp`. Bind it in the HUD to `OnLevelUp` as well.

✅ **Test:** set `Level` default to 8 in `SS_GameMode` temporarily. Enemies should be clearly faster and spawn more often. Set it back to 1.

---

## F6 – Power-ups: Shield, Triple Shot, Extra Life

**Goal:** destroyed enemies sometimes drop a pickup. Touching it gives:

| Type | Color | Effect | Duration |
|---|---|---|---|
| **Extra Life** | Green | `NumLifes + 1` (max 3) | instant |
| **Shield** | Blue | Ignore all damage, visible bubble | 6 s |
| **Triple Shot** | Orange | Fire 3 bullets in a spread | 8 s |

**Blueprints touched:** new `E_PowerUpType`, `BP_PowerUp`, `M_PowerUp`, `M_Shield`; `BP_EnemyBase`, `BP_SS_Character`

### Step 1 — Enum
`PowerUps/` → **Right click → Blueprint → Enumeration** → **`E_PowerUpType`** with: `ExtraLife`, `Shield`, `TripleShot`.

### Step 2 — Materials
- **`M_PowerUp`**: Vector Parameter **`Color`** → connect to **Base Color** and (× 5) to **Emissive Color**.
- **`M_Shield`**: Blend Mode **Translucent**, Shading Model **Unlit**, Emissive = cyan, Opacity = `0.3` (optionally Fresnel → Opacity for a bubble edge).

### Step 3 — `BP_PowerUp` (Actor)

**Components**
```
DefaultSceneRoot
└── Sphere (Sphere Collision)  radius ~60, Collision Preset = OverlapOnlyPawn
    └── Mesh (Static Mesh = StarterContent/Shapes/Shape_Sphere, scale 0.5, NoCollision)
RotatingMovement (Rotation Rate Z = 180)
```

> **OverlapOnlyPawn** means only the player ship detects the pickup, so the player's bullets won't explode on it.

**Variables**

| Variable | Type | Default | Flags |
|---|---|---|---|
| `Type` | `E_PowerUpType` | ExtraLife | **Instance Editable + Expose on Spawn** |
| `Speed` | Float | 300 | Same value as `BP_EnemyAmmo` so it drifts the same way |

**Event BeginPlay**
```
Set Life Span (10)
Mesh → Create Dynamic Material Instance (Parent = M_PowerUp)
  → Set Vector Parameter Value ( "Color",
        Select( Type ): ExtraLife = Green, Shield = Blue, TripleShot = Orange ) )
```

**Event Tick** — copy the movement from `BP_EnemyAmmo` ("Update Movement"):
```
Add Actor World Offset ( GetActorForwardVector * Speed * DeltaSeconds )
```

**Event ActorBeginOverlap**
```
Other Actor → Cast To BP_SS_Character → ApplyPowerUp( Type ) → Destroy Actor (self)
```

### Step 4 — Drop from enemies (`BP_EnemyBase`)

New variables: `DropChance` (Float, **0.15**), `PowerUpClass` (Class ref of `BP_PowerUp`, default `BP_PowerUp`).

Find the spot where the enemy dies **because the player shot it** (just before `Call OnEnemyDestroy` / `Destroy Actor` in the overlap logic). Don't use the place where it is destroyed for leaving the screen. Insert:

```
Branch ( Random Float < DropChance )
  true → Spawn Actor from Class ( PowerUpClass,
            Spawn Transform = GetActorTransform,
            Type = Byte To Enum E_PowerUpType( Random Integer(3) ) )
→ … existing death logic …
```

> If you do F7 first, put this inside the `Die` function instead (see F7).

### Step 5 — Player (`BP_SS_Character`)

**New components**
- **`ShieldMesh`**: Static Mesh `Shape_Sphere`, material `M_Shield`, scale so it surrounds the ship, **Collision = NoCollision**, **Visible = false**.

**New variables**: `bShieldActive` (Bool), `bTripleShot` (Bool), `ShieldDuration` (Float, 6), `TripleShotDuration` (Float, 8), `MaxLifes` (Int, 3), `SpreadAngle` (Float, 15).

**Function `ApplyPowerUp(Type: E_PowerUpType)`** → **Switch on E_PowerUpType**:

```
ExtraLife  → SET NumLifes = Min( NumLifes + 1, MaxLifes )
             → (same HUD call PlayerDamage uses) GetHUD → Cast BP_SS_HUD → GameWidget.UpdateLifes(NumLifes)

Shield     → SET bShieldActive = true → ShieldMesh.SetVisibility(true)
             → Set Timer by Event ( Time = ShieldDuration, Looping = false ) → custom event EndShield

TripleShot → SET bTripleShot = true
             → Set Timer by Event ( Time = TripleShotDuration, Looping = false ) → custom event EndTripleShot
```

- `EndShield` → `bShieldActive = false`, `ShieldMesh.SetVisibility(false)`
- `EndTripleShot` → `bTripleShot = false`

> Calling **Set Timer by Event** again with the same event restarts the timer, so picking up a second shield refreshes the duration.

> `ApplyPowerUp` uses timers, which are latent-ish. If Blueprint complains about the function, make **`ApplyPowerUp` a Custom Event** in the Event Graph instead.

**Shield blocks damage** — at the very start of **`PlayerDamage`**:
```
Branch ( bShieldActive ) → true: Return (no damage) / false: existing logic
```

**Triple shot** — refactor the bullet spawn in `Execute Fire`:

1. New function **`SpawnBullet(AngleOffset: float)`**:
```
T = FireSpawn → Get World Transform
R = Combine Rotators ( T.Rotation , Make Rotator(Roll 0, Pitch 0, Yaw AngleOffset) )
Spawn Actor from Class ( BP_Actor_Ammo, Make Transform(T.Location, R, T.Scale) )
```
2. Replace the existing *Spawn Actor BP_Actor_Ammo* node in `Execute Fire` with:
```
Branch ( bTripleShot )
  true  → SpawnBullet(-SpreadAngle) → SpawnBullet(0) → SpawnBullet(+SpreadAngle)
  false → SpawnBullet(0)
```

> If the bullets spread along the wrong axis (because of how the camera looks at the play area), put the offset in **Pitch** or **Roll** instead of **Yaw**.

### Connection diagram
```
BP_Actor_Ammo hits enemy ─► BP_EnemyBase (dies) ─► 15% ─► spawn BP_PowerUp(Type)
                                                               │ overlaps player
                                                               ▼
                                       BP_SS_Character.ApplyPowerUp(Type)
                                ┌──────────────┼──────────────────────┐
                           ExtraLife         Shield               TripleShot
                       NumLifes+1 → HUD   bShieldActive + timer   bTripleShot + timer
                                          (PlayerDamage checks)   (Execute Fire checks)
```

✅ **Test:** temporarily set `DropChance = 1.0` so every kill drops something. Test each type, then set it back to 0.15.

---

## F7 – Enemy health system + 3rd enemy type ("Tank")

**Goal:** enemies can take several hits. A new slow, big enemy, **Tank**, needs 3 hits. The existing enemies keep 1 hit.

**Blueprints touched:** `BP_EnemyBase`, new `BP_EnemyTank`, `BP_RandomGenerator`

### Step 1 — Health in `BP_EnemyBase`

**Variables**: `MaxHealth` (Int, **1** so the current enemies behave the same), `CurrentHealth` (Int).

**BeginPlay** (start): `SET CurrentHealth = MaxHealth`.

**Function `Die`** — **move** your existing death nodes here (the F6 power-up drop + `Call OnEnemyDestroy` + `Destroy Actor`).

**Function `OnDamaged`** — leave it **empty**. Children override it (Tank: flash, Boss: health bar).

**Function `ApplyDamage(Amount: int)`**:
```
SET CurrentHealth = CurrentHealth - Amount
→ OnDamaged
→ Branch ( CurrentHealth <= 0 ) → true: Die
```

**In ActorBeginOverlap:** where it currently runs the death logic after a hit from the player, call **`ApplyDamage(1)`** instead.

> Make sure the **player bullet is still destroyed** on hit. Otherwise one bullet passes through and hits again. `BP_Actor_Ammo` already explodes and destroys itself on overlap, so this should already work. Just check it.

### Step 2 — `BP_EnemyTank`
1. Right-click `BP_EnemyBase` → **Create Child Blueprint Class** → **`BP_EnemyTank`**.
2. **Class Defaults:** `MaxHealth = 3`, `Speed` ≈ 60% of the base value, `DropChance = 0.3`, `DelayShoot` a bit higher.
3. **Components:** scale the mesh to ~1.5 and give it a different material (for example `StarterContent/Materials/M_Metal_Rust`).
4. **Override `OnDamaged`** (hit feedback):
```
Play Sound 2D (StarterContent/Audio/Collapse_Cue)       // or any short sound
Set Actor Scale 3D ( current * 0.9 )  → Delay 0.05 → Set back   // quick "squish"
```
5. If you add a BeginPlay → **call Parent: BeginPlay first**.

### Step 3 — Spawn it (`BP_RandomGenerator`)
In `SpawnEnemy`, the class comes from a **Select** node (`Option 0` = `BP_EnemyBase`, `Option 1` = `BP_EnemyH`) driven by **Random Integer**.

1. Right-click the Select node → **Add Option Pin** → `Option 2` = `BP_EnemyTank`.
2. Make Tanks appear only from **level 3**:
```
Random Integer ( Max = Select( GameMode.Level >= 3 ): 3 / 2 )   → Select index
```

✅ **Test:** set the `Level` default to 3 temporarily. Tanks appear, take 3 hits, and still count as kills.

---

## F8 – Boss every N levels with health bar

**Goal:** every **5 levels** normal spawning stops and a big boss appears. It moves side to side, fires a fan of bullets, and has a **health bar** on the HUD. Killing it gives **+500 score**, a guaranteed power-up, and resumes normal spawning.

**Blueprints touched:** new `BP_EnemyBoss`, `BP_SS_GameWidget`, `BP_SS_HUD`, `BP_RandomGenerator`, `SS_GameMode`

### Step 1 — GameMode helper
Function **`AddScore(Amount: int)`** → `SET Progress = Progress + Amount`.

### Step 2 — Widget: boss bar (`BP_SS_GameWidget`)
1. **Designer:** top-center **Overlay** **`Overlay_Boss`** (Collapsed) with a Text `BOSS` and a **Progress Bar** **`BossHealthBar`** (Is Variable, fill color red, Percent 1).
2. Functions:
   - **`ShowBossBar(bShow: bool)`** → `Overlay_Boss.SetVisibility( Visible / Collapsed )`
   - **`SetBossHealth(Percent: float)`** → `BossHealthBar.SetPercent(Percent)`

### Step 3 — `BP_EnemyBoss` (child of `BP_EnemyBase`)

**Class Defaults:** `MaxHealth = 30`, `DropChance = 1.0`, `Speed` (entry speed) ≈ base value, mesh scale ~3, a distinct material.

**New variables:**

| Variable | Type | Default |
|---|---|---|
| `EnterTime` | Float | 2.0 (seconds moving forward before strafing) |
| `StrafeAmplitude` | Float | 600 |
| `StrafeFrequency` | Float | 1.2 |
| `FireInterval` | Float | 1.2 |
| `BulletsPerVolley` | Int | 5 |
| `FanAngle` | Float | 60 |
| `BossBonus` | Int | 500 |
| `TimeAlive` | Float | 0 |
| `StrafeCenter` | Vector | — |

**Event BeginPlay**
```
Parent: BeginPlay
→ GetHUD (Get Player Controller 0 → Get HUD → Cast BP_SS_HUD) → GameWidget.ShowBossBar(true) → SetBossHealth(1.0)
→ Set Timer by Event ( FireInterval, Looping ) → custom event FireVolley
```

**Event Tick** — ⚠️ **do NOT call Parent: Tick** here. The boss replaces the normal movement and shooting.
```
SET TimeAlive += DeltaSeconds
Branch ( TimeAlive < EnterTime )
  true  → Add Actor World Offset( GetActorForwardVector * Speed * DeltaSeconds )
          → SET StrafeCenter = GetActorLocation
  false → Set Actor Location(
              StrafeCenter + GetActorRightVector * Sin( TimeAlive * StrafeFrequency ) * StrafeAmplitude )
```
(`BP_EnemyH` already moves laterally with `GetActorRightVector`, so this axis matches your game.)

**Custom event `FireVolley`** — fan of bullets:
```
Step = FanAngle / (BulletsPerVolley - 1)
For Loop ( 0 → BulletsPerVolley - 1 ) (Index)
   Angle = -FanAngle/2 + Index * Step
   R = Combine Rotators ( FirePoint.WorldRotation , Make Rotator(Yaw = Angle) )
   Spawn Actor from Class ( BP_EnemyAmmo, Make Transform(FirePoint.WorldLocation, R) )
```

**Override `OnDamaged`**
```
GameWidget.SetBossHealth( CurrentHealth / MaxHealth )        // convert both to Float before dividing
Branch ( CurrentHealth <= 0 )
  true → Get Game Mode → Cast SS_GameMode → AddScore(BossBonus)
```

**Event Destroyed**
```
GameWidget.ShowBossBar(false)
```

### Step 4 — Spawn the boss (`BP_RandomGenerator`)
Variables: `BossEveryNLevels` (Int, 5), `BossClass` (Class of `BP_EnemyBoss`), `bBossActive` (Bool).

Extend the **`HandleLevelUp`** event from F5 (**at the end**):
```
Branch ( NewLevel % BossEveryNLevels == 0  AND  NOT bBossActive )
  true →
    SET bBossActive = true
    Pause Timer by Handle ( SpawnTimerHandle )
    Spawn Actor from Class ( BossClass, Transform = middle SpawnPoint's world transform )
      → Bind Event to OnEnemyDestroy → custom event OnBossDefeated
```
```
OnBossDefeated
  SET bBossActive = false
  Unpause Timer by Handle ( SpawnTimerHandle )
```

> The boss is a child of `BP_EnemyBase`, so its death also goes through your existing `OnEnemyDestroy → GameMode.Enemy Destroyed` path and counts as a kill.

> Because F5 resets the timer with `Clear and Invalidate` + `Set Timer` on each level-up, a level-up **during** a boss fight would restart spawning. Add `Branch(NOT bBossActive)` around the F5 timer reset, or store the new time and apply it in `OnBossDefeated`.

### Connection diagram
```
SS_GameMode.Enemy Destroyed → Level++ → OnLevelUp(NewLevel)
                                              │
                         BP_RandomGenerator.HandleLevelUp
                ┌──────────────┴─────────────────┐
         update SpawnTime (F5)        Level % 5 == 0 → Pause spawn timer
                                                 → Spawn BP_EnemyBoss
                                                         │
                     ┌───────────────────────────────────┼───────────────────────┐
              BeginPlay: ShowBossBar           ApplyDamage → OnDamaged       Die → OnEnemyDestroy
                                               SetBossHealth(%)              → OnBossDefeated: unpause timer
                                               AddScore(500) on kill         → Destroyed: hide bar
```

✅ **Test:** set `BossEveryNLevels = 1` (or the `Level` default to 4) to reach the boss quickly. Check the bar goes down, the score jumps by +500, a power-up drops, and normal spawning resumes.

---

## 11. Final testing checklist

| # | Check | ✔ |
|---|---|---|
| 1 | Level text updates on level up | ☐ |
| 2 | First launch with no save: Best shows `0 · LV 1` | ☐ |
| 3 | Game over with higher score → `NEW RECORD!` + file in `Saved/SaveGames` | ☐ |
| 4 | Restart / reopen → Best values persist | ☐ |
| 5 | Score stops increasing on game over | ☐ |
| 6 | P pauses: enemies, bullets, score, spawner frozen; overlay shown | ☐ |
| 7 | P resumes; P on game-over screen does nothing | ☐ |
| 8 | Q quits while playing and while paused; records saved | ☐ |
| 9 | Q on game-over screen doesn't also restart the level | ☐ |
| 10 | Level 5+ visibly harder; spawn time never below `MinSpawnTime` | ☐ |
| 11 | Each power-up works; shield blocks damage; lives capped at 3 | ☐ |
| 12 | Power-ups don't drop when enemies leave the screen | ☐ |
| 13 | Tank takes 3 hits; only appears from level 3 | ☐ |
| 14 | Boss appears at level 5, bar works, +500, spawning resumes | ☐ |
| 15 | Pausing during a boss fight / power-up keeps timers correct | ☐ |
| 16 | **All debug values restored** (`DropChance`, `Level` default, `BossEveryNLevels`) | ☐ |
| 17 | Packaged build (Platforms → Mac → Package) runs and saves | ☐ |

---

## 12. Common pitfalls

- **Pause can't be undone** → `Trigger when Paused` isn't ticked on `IA_Pause` (and `IA_Quit`).
- **Pause flickers** → you used the *Triggered* pin; use **Started**.
- **Child enemy ignores difficulty / health** → its BeginPlay doesn't call **Parent: BeginPlay**.
- **Boss moves like a normal enemy** → its Tick calls **Parent: Tick**; remove that call.
- **"Accessed None" errors** → a **Cast** failed or `GameWidget` isn't created yet. Check that `SS_GameMode` is the GameMode in *World Settings → GameMode Override* or *Project Settings → Maps & Modes*, and that it uses `BP_SS_HUD` and `SS_PlayerController`.
- **Best score always 0** → you set the values but never called **Save Game to Slot**, or the slot name differs between load and save (always use `SaveSlotName`).
- **Bullets explode on power-ups** → set the `BP_PowerUp` collision to `OverlapOnlyPawn`.
- **Integer division** → `CurrentHealth / MaxHealth` on integers gives 0. Convert to Float first.
- **Binary assets** → don't edit the same `.uasset` in two git branches; commit after each feature.

---

## F9 – Player loses a life when hit by an enemy bullet or when colliding with an enemy

**Goal:** the player loses **one life** when:
1. an **enemy bullet** (`BP_EnemyAmmo` / `BP_EnemyAmmo2`) touches the ship, or
2. the ship **collides with an enemy** (`BP_EnemyBase` and all its children).

After a hit the player is **invulnerable for 1.5 s** and the ship blinks, so one collision can't remove several lives in a row.

**Blueprints touched:** `BP_EnemyAmmo`, `BP_EnemyAmmo2`, `BP_EnemyBase`, `BP_SS_Character`

### Why it doesn't work today
- `BP_EnemyAmmo` and `BP_EnemyAmmo2` have an **ActorBeginOverlap** event, but it never talks to the player.
- `BP_SS_Character` has a **`PlayerDamage`** function (it subtracts from `NumLifes`), but nothing calls it on a hit. Only the debug key uses it.
- The collision settings may make the ship **block** bullets and enemies instead of **overlapping** them. Blocking never fires an overlap event.

### Design: handle the damage in one place, the player
All "something hurt me" logic goes in **`BP_SS_Character`'s ActorBeginOverlap**. Bullets and enemies only need a **tag** that says what they are. That gives one place to check the shield, invulnerability and game over, so you don't repeat the checks in every enemy Blueprint.

```
Enemy bullet ──overlap──┐
                        ├──► BP_SS_Character.ActorBeginOverlap
Enemy ship  ──overlap───┘         │
                                  ├─ tag "EnemyAmmo" → destroy bullet → PlayerDamage
                                  └─ tag "Enemy"     → enemy.Die / ApplyDamage(1) → PlayerDamage
                                                                        │
                                          shield? invulnerable? already dead? → ignore
                                          else NumLifes-1 → UpdateLifes → (0 → GameOver)
                                                       → StartInvulnerability (blink 1.5 s)
```

### Step 1 — Tags
Tags are set in **Class Defaults → Actor → Advanced → Tags** (click **+**).

| Blueprint | Add tag |
|---|---|
| `BP_EnemyAmmo` | `EnemyAmmo` |
| `BP_EnemyAmmo2` | `EnemyAmmo` |
| `BP_EnemyBase` | `Enemy` *(children such as `BP_EnemyH`, `BP_EnemyTank` and `BP_EnemyBoss` inherit it; check their Class Defaults still show it)* |

> Keep the existing tags (`Player`, `Ammo`, `Base`…). Your current overlap logic uses them.

### Step 2 — Collision settings
An overlap event fires only if **both** components have **Generate Overlap Events ✔** and their channels are set to **Overlap** each other, not Block.

| Blueprint → component | Collision Preset | Generate Overlap Events |
|---|---|---|
| `BP_SS_Character` → `CapsuleComponent` (CollisionCylinder) | `Pawn` (keep) | ✔ |
| `BP_EnemyAmmo` / `BP_EnemyAmmo2` → root Capsule | **`OverlapAllDynamic`** | ✔ |
| `BP_EnemyBase` → root Capsule | **`OverlapAllDynamic`** | ✔ |
| Their **Static Mesh** children | `NoCollision` | — |

> The ship moves with **CharacterMovement**, which **sweeps**. If an enemy's collision **blocks** the Pawn channel, the ship just stops against it and no overlap fires. That's why enemies must use **OverlapAllDynamic**.

> Turning off collision on the mesh children avoids **double overlap events** (capsule + mesh both triggering).

### Step 3 — Prepare the enemy: function `Die` in `BP_EnemyBase`
If you haven't done F7 yet, do its first part now: create the function **`Die`** and **move** the existing death nodes into it (`Call OnEnemyDestroy` → `Destroy Actor`, plus the F6 drop if it exists). Then make the old place in the overlap graph call `Die`.

> Calling `Die` instead of only `Destroy Actor` matters: `OnEnemyDestroy` tells the GameMode and the spawner the enemy is gone. If you skip it, the spawner's enemy count gets out of sync.

### Step 4 — Player: new variables (`BP_SS_Character`)

| Variable | Type | Default |
|---|---|---|
| `bInvulnerable` | Boolean | false |
| `InvulnerabilityTime` | Float | 1.5 |
| `BlinkTimerHandle` | Timer Handle | — |

### Step 5 — Player: guard at the start of `PlayerDamage`
Put this **before** the existing `NumLifes - 1` logic. It replaces the F6 shield-only check.

```
PlayerDamage
  Branch ( bShieldActive  OR  bInvulnerable  OR  NumLifes <= 0 )
     true  → Return                              // no damage
     false → … existing logic: NumLifes - 1 → UpdateLifes → (NumLifes <= 0 → GameOver) …
             → Branch ( NumLifes > 0 ) → true: StartInvulnerability
```

> Because the guard is inside `PlayerDamage`, the debug key, bullets and collisions all follow the same rules.

> `bShieldActive` only exists after F6. If you do F9 first, leave it out of the OR and add it when you build the shield.

### Step 6 — Player: invulnerability + blink
**Custom event `StartInvulnerability`**
```
SET bInvulnerable = true
Set Timer by Event ( Time = 0.1, Looping = true ) → custom event Blink → SET BlinkTimerHandle
Set Timer by Event ( Time = InvulnerabilityTime, Looping = false ) → custom event EndInvulnerability
```
**Custom event `Blink`**
```
<ship mesh component> → Toggle Visibility
```
**Custom event `EndInvulnerability`**
```
SET bInvulnerable = false
Clear and Invalidate Timer by Handle ( BlinkTimerHandle )
<ship mesh component> → Set Visibility ( true )     // make sure it ends visible
```

> Use the component that draws the ship, not the whole actor. Toggling the actor's visibility would also hide the shield bubble from F6.

### Step 7 — Player: Event ActorBeginOverlap (`BP_SS_Character`)
```
Event ActorBeginOverlap (Other Actor)
  Branch ( Other Actor → Actor Has Tag "EnemyAmmo" )
     true → Other Actor → Destroy Actor           // the bullet is consumed (even if the shield absorbs it)
          → PlayerDamage
     false →
  Branch ( Other Actor → Actor Has Tag "Enemy" )
     true → Other Actor → Cast To BP_EnemyBase
              → Die                     // before F7
              → ApplyDamage(1)          // after F7: normal enemies die, the Tank loses 1 HP, the Boss loses 1 HP
          → PlayerDamage
```

> Use **ApplyDamage(1)** once F7 exists. With it, ramming the **Boss** doesn't instantly kill it; the player loses a life and the boss loses one hit point. The 1.5 s invulnerability stops the player from losing all lives while touching the boss.

> Ramming an enemy calls `Die`, which fires `OnEnemyDestroy`, so it **counts as a kill** for level-up. If you don't want that, add a `bKilledByPlayer` Boolean input to `Die` and only fire the dispatcher's level-up path when it's true. That takes more work; the default above is fine for this project.

### Step 8 — Remove any old, conflicting logic
- If `BP_EnemyAmmo` / `BP_EnemyAmmo2` **destroy themselves on any overlap**, make sure they ignore other enemies and other enemy bullets: `Branch(Other Actor has tag "Enemy" OR "EnemyAmmo") → do nothing`. Otherwise a bullet spawned at an enemy's `FirePoint` can vanish immediately.
- In `BP_EnemyBase`'s overlap, if colliding with the player (tag `Player`) currently destroys the enemy, **remove that branch**. The Character now handles that case, and keeping both would run `Die` twice.

### Optional — bullets vs bullets
At the moment the player's bullet (`BP_Actor_Ammo`) may explode when it touches an enemy bullet. Choose one behaviour and make it explicit in `BP_Actor_Ammo` → ActorBeginOverlap:
- **Shoot bullets down (fun):** `Other Actor has tag "EnemyAmmo"` → destroy both + explosion.
- **Pass through (simpler):** `Other Actor has tag "EnemyAmmo"` → do nothing.

### Debugging tip
If nothing happens on a hit, add a **Print String** (`Other Actor → Get Display Name`) as the first node of the Character's ActorBeginOverlap.
- **Nothing prints:** it's a collision settings problem (Step 2).
- **It prints but there's no damage:** check the tag names (they're case-sensitive) or the `PlayerDamage` guard.

### ✅ Tests
| # | Check | ✔ |
|---|---|---|
| 1 | Enemy bullet hits ship → 1 life lost, bullet disappears, ship blinks | ☐ |
| 2 | Several bullets within 1.5 s → still only 1 life lost | ☐ |
| 3 | Ship rams a normal enemy → enemy explodes/dies, 1 life lost | ☐ |
| 4 | Ship touches the Boss / Tank → 1 life lost, enemy loses 1 HP (after F7) | ☐ |
| 5 | With the shield active → bullets disappear, no life lost | ☐ |
| 6 | Last life lost → game over once (no double GameOver call) | ☐ |
| 7 | No damage while paused or after game over | ☐ |
| 8 | Enemy bullets don't disappear when spawned inside their own enemy | ☐ |
