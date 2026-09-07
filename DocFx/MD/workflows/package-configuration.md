# Package Configuration

Astra Framework needed no configuration. The health package, however, needs to be configured to have the system work around the specific instances of your game. For example, if you defined a "Super Duper All-damage resistance" `Stat` in your game, you need to tell Astra Health to use it when calculating damage.

The package uses a flexible configuration system that balances convenience with explicit control. This page explains how to set up and configure the health system for your game.

## Configuration Overview

The Astra Health system uses a **two-tier configuration architecture**:

1. **Global Settings** (`AstraHealthGlobalSettings`) - A lightweight pointer stored in `Resources` that references your active configuration
2. **Gameplay Configuration** (`AstraHealthConfigSO`) - The actual configuration containing all gameplay parameters

This separation allows you to:
- Switch between different configuration profiles easily (e.g., for testing, different game modes)
- Keep configuration data separate from the loading mechanism
- Support convention-based fallbacks for quick prototyping

---

## Global Settings

### Automatic Setup

The package automatically creates the Global Settings asset on first import or when the editor loads. You don't need to do anything manually.

**What happens automatically:**
1. The `AstraHealthGlobalSettings.asset` is created in `Assets/Resources/`
2. If a default configuration exists (e.g., from imported samples), it's automatically assigned
3. The system is immediately ready to use

> [!NOTE]
> Default location: `Assets/Resources/AstraHealthGlobalSettings.asset`

### Project Settings

You can manage the package configuration through Unity's Project Settings window:

1. Open **Edit → Project Settings**
2. Navigate to **Astra Health**
3. Assign your desired **Active Config Profile**

**Status Indicators:**
- ✅ **Gray "Using Explicit Configuration"** - A configuration is explicitly assigned
- ⚠️ **Yellow "Using Fallback"** - No explicit configuration; using convention-based fallback
- 🛑 **Red "No Configuration Found"** - Critical: No configuration available

**Quick Actions:**
- **Create New Config Asset** - Opens a save dialog to create a new configuration

### Manual Configuration (alternative to Project Settings)

You can also manage the package configuration manually by directly editing the `AstraHealthGlobalSettings` asset in the `Assets/Resources` folder. This approach allows you to assign a specific `AstraHealthConfigSO` asset without using the Project Settings window.

### Convention Over Configuration

The package follows a **convention-over-configuration** philosophy to reduce setup friction:

#### Fallback Resolution

If no explicit configuration is assigned in Project Settings, the system automatically searches for a configuration named:

> **`Astra Health Config`**

located in any **`Resources`** folder in your project.

#### Search Order

The configuration provider uses a **three-step loading strategy**:

1. **Explicit Configuration** (Project Settings)
   - Loads `AstraHealthGlobalSettings` from `Resources/AstraHealthGlobalSettings`
   - If it has an `ActiveConfig` assigned, use it
   
2. **Convention-Based Fallback**
   - Searches for `Astra Health Config` in any `Resources` folder
   - Logs a warning indicating fallback usage
   
3. **Error State**
   - If neither is found, logs an error with instructions
   - System will not function until a configuration is provided

> [!TIP]
> For production projects, always use **explicit configuration** via Project Settings for clarity and control.

---

## Configuration Loading Strategy

The health configuration is loaded lazily on first access and cached for performance:

```csharp
// Automatically loads configuration on first access
var config = AstraHealthConfigProvider.Instance;

// Pre-load during initialization to avoid runtime overhead
AstraHealthConfigProvider.WarmUp();

// Force reload (useful for testing)
AstraHealthConfigProvider.Reset();
```

**When is the configuration loaded?**
- Automatically before the first scene loads (via `RuntimeInitializeOnLoadMethod`)
- Lazily when first accessed via `AstraHealthConfigProvider.Instance`
- Explicitly when calling `WarmUp()`

---

## Creating Configuration Assets
If for any reason you need to create a new `AstraHealthConfigSO` asset, you can do so via two methods:

### Via Project Settings

1. Open **Edit → Project Settings → Astra Health**
2. Unassign any existing configuration, if any
3. Click **Create New Config Asset**
4. Choose a save location
5. The new configuration is automatically assigned

### Via Asset Menu

1. Right-click in the **Project Window**
2. Select **Create → Astra Health → Config → Astra Health Config**
3. Name your configuration
4. Assign it in Project Settings or in the Global Settings asset

---

The red asterisks (<span style="color:red;">*</span>) indicate fields that are required for the system to function properly. Make sure to assign them before entering Play Mode to avoid runtime errors.

---

## Health Configuration Reference
> [!NOTE]
> In the `AstraHealthConfigSO` asset, you can hover over each field to see a tooltip with a brief description.

The `AstraHealthConfigSO` asset contains all gameplay parameters for the health system. Below is a detailed explanation of each field.

### Health

#### Health Attributes Scaling
**Type:** `AttributesScalingComponent`  
**Required:** No  
**Description:** Defines how entities' maximum health scales based on character attributes (e.g., Vitality, Endurance).

See also: [Astra Framework Scaling documentation](https://electricdrill.github.io/AstraRpgFrameworkDocs/MD/workflows.html#create-scaling-formulas)

#### Generic Flat Heal Amount Modifier Stat
**Type:** `Stat`  
**Required:** No  
**Description:** The stat that modifies **all** healing received by an entity as a flat amount.

**Stacking Behavior:**
- Combines **additively** with source-based flat amount modifications

**How it works:**
- The stat value represents a **flat amount modifier**
- Positive values increase healing, negative values decrease it
- Example: A value of `25` means +25 extra HP healing received

**Example:**
- Base Heal: 100 HP
- Generic Heal Amount Modifier: 25 (means +25 HP healing received)
- Final Heal: 100 + 25 = **125 HP**

### Generic Percentage Heal Amount Modifier Stat
**Type:** `Stat`
**Required:** No
**Description:** The stat that modifies **all** healing received by an entity as a percentage. It is applied on top of the flat heal amount modifiers.

**Stacking Behavior:**
- Combines **additively** with source-based percentage modifications

**How it works:**
- The stat value represents a **percentage modifier**
- Positive values increase healing, negative values decrease it
- Example: A value of `20` means +20% extra healing received

**Example:**
- Base Heal: 100 HP
- Generic Flat Heal Amount Modifier: 20 (means +20 extra HP healing received)
- Generic Percentage Heal Amount Modifier: 20 (means +20% healing received)
- Final Heal: (100 + 20) × 1.20 = **144 HP**

> [!WARNING]
> Remember to remove the default lower bound of 0 from your flat and percentage heal modifiers if you want to allow negative values for healing reduction effects. By default, the base framework sets a lower bound of 0 on all stats.

---

### Damage

#### Default Damage Calculation Strategy
**Type:** `DamageCalculationStrategy`  
**Required:** No  
**Description:** The default strategy used to calculate net damage when an entity doesn't specify its own strategy.

The default strategy applied to entities that do not have a custom one assigned in their `EntityHealth` component. In most cases a single global default is sufficient; override it per-entity when a different calculation pipeline is required. See [Damage Calculation Strategy](./damage.md#damage-calculation-strategy) for details on how strategies work.

#### Generic Flat Damage Modification Stat
**Type:** `Stat`  
**Required:** No  
**Description:** A universal flat damage modifier that applies to **all damage received**, regardless of type or source.

Applied by `ApplyFlatDmgModifiersStep` when that step is included in the active strategy.

**Stacking Behavior:**
- Combines **additively** with Type and Source **flat** modifications

**Example:**
- Incoming Damage: 150
- Generic Flat Damage Modifier: -20 (means -20 HP of damage taken)
- **Damage Mitigation:** -20 → Final Damage: **130**

#### Generic Percentage Damage Modification Stat
**Type:** `Stat`
**Required:** No
**Description:** A universal percentage damage modifier that applies to **all damage received**, regardless of type or source.

Applied by `ApplyPercentageDmgModifiersStep` when that step is included in the active strategy.

**Stacking Behavior:**
- Combines **additively** with Type and Source **percentage** modifications

**Example:**
- Incoming Damage: 150
- Generic Percentage Damage Modifier: -20 (means -20% damage taken)
- **Damage Mitigation:** -20% → Final Damage: 150 × 0.80 = **120**

> [!NOTE]
> If the entity to be healed doesn't have the specified flat/percentage heal modification stats, they will be considered as having a value of 0 for those stats, and therefore no modification will be applied to the healing amount for that entity.

#### Rounding Settings
**Type:** `HealthRoundingSettings`  
**Required:** No  
**Description:** Controls how fractional intermediate results in health combat calculations are rounded to integer gameplay amounts. All five modes default to **Round** (midpoint-away-from-zero), preserving the behavior of versions that did not expose this setting.

> [!NOTE]
> All rounding settings — including lifesteal — are configured here, under **Damage**, because they are part of the global combat configuration asset (`AstraRpgHealthConfigSO`).

`HealthRoundingSettings` exposes one `RoundingMode` field per mechanic:

| Field | When it is applied |
|---|---|
| **Percentage Damage Modifier Rounding Mode** | After all percentage modifiers (generic, source-specific, type-specific) are combined and applied to the current damage amount in `ApplyPercentageDmgModifiersStep`. |
| **Defense Penetration Rounding Mode** | After the defense-penetration function reduces the target's effective defensive stat value inside `ApplyDefenseStep`. |
| **Damage Mitigation Rounding Mode** | After the damage-mitigation function converts the effective defense into a final reduced damage amount inside `ApplyDefenseStep`. |
| **Critical Damage Multiplier Rounding Mode** | After the critical-hit multiplier is applied to the current damage amount in `ApplyCriticalMultiplierStep`. |
| **Lifesteal Rounding Mode** | Applied independently to each lifesteal contribution (generic and damage-type) before amounts are summed or applied as heals. |

`RoundingMode` has three values:

| Value | Behaviour | Example |
|---|---|---|
| **Round** *(default)* | Nearest integer; midpoint rounds away from zero. | 2.5 → 3, 2.4 → 2 |
| **Floor** | Largest integer ≤ the value. | 2.9 → 2, 2.1 → 2 |
| **Ceil** | Smallest integer ≥ the value. | 2.1 → 3, 2.9 → 3 |

> [!TIP]
> **Floor** ensures defensive calculations never accidentally round up in the attacker's favour. **Ceil** guarantees that small offensive or heal amounts are never silently truncated to zero. **Round** (the default) provides neutral, balanced behaviour suitable for most games.

#### Damage Step Trace
**Type:** `DamageStepTracePolicy`  
**Default:** `EditorAndDevelopmentBuilds`  
**Required:** No  
**Description:** Controls whether the damage calculation pipeline records its per-step amount trace (`DamageAmountContext.Records` — the before/after damage value at each pipeline step). Recording the trace costs one list allocation per damage instance, so it can be turned off where it is not needed.

| Value | Behaviour |
|---|---|
| **Always** *(default)* | The trace is always recorded, including in release builds. |
| **EditorAndDevelopmentBuilds** | The trace is recorded in the editor and in development builds, and skipped in release builds. |
| **Never** | The trace is never recorded. |

The trace is used for debugging and diagnostics, and by **Step**-mode lifesteal ([Amount Selector](lifesteal.md#amount-selector)), which samples the damage value at a chosen pipeline step. If you use step-based lifesteal in a release build, set this to **Always** — otherwise a one-time console warning is logged and step-based lifesteal falls back to the final damage amount. When the trace is disabled, `DamageAmountContext.Records` is empty.

> [!WARNING]
> If you defined custom code that inspects damage steps, you NEED to leave it turned ON with `Always`, or your game logic will break.

---

### Health Regeneration

These settings are shared by both automatic and manual regeneration. `EntityPassiveHpRegeneration` consumes the passive-tick settings described below, while `EntityHealth.ManualHealthRegenerationTick()` uses the same regeneration `HealSourceSO` together with the dedicated manual-regeneration stat.

> [!NOTE]
> Configuring the Health Regeneration section does not enable automatic regeneration on any entity by itself. To opt an entity into time-based regeneration, add the `EntityPassiveHpRegeneration` component alongside `EntityHealth`.

#### Health Regeneration Source
**Type:** `HealSourceSO`  
**Required:** No  
**Description:** The heal source used for passive regeneration effects and manual regeneration ticks.

**Use cases:**
- Allows tracking and modifying regeneration separately from active healing
- Can be used for effects like "Increase Regeneration by 50%"
- Tracking passive healing in analytics or combat logs

#### Passive Health Regeneration Stat (HP/10s)
**Type:** `Stat`  
**Required:** No  
**Description:** Determines the amount of health regenerated passively.

> [!WARNING]
> The stat value represents health regenerated **per 10 seconds**.

**Calculation:**
```
Health Per Tick = (Stat Value / 10) * Interval In Seconds
```

**Example:**
- Stat Value: 50 HP/10s
- Interval: 1 second
- Health Per Tick: (50 / 10) * 1 = **5 HP per second**

#### Passive Health Regeneration Interval
**Type:** `float`  
**Default:** 1.0 seconds  
**Required:** Yes (must be > 0)  
**Description:** The time (in seconds) between passive regeneration ticks.

**Configuration Examples:**

| Interval | Stat Value | Result |
|----------|------------|--------|
| 1.0s | 50 HP/10s | 5 HP every 1 second |
| 0.5s | 50 HP/10s | 2.5 HP every 0.5 seconds |
| 2.0s | 50 HP/10s | 10 HP every 2 seconds |

> [!WARNING]
> Smaller intervals increase CPU overhead. Recommended range: 0.5s - 2.0s.

#### Suppress Passive Regeneration Events
**Type:** `bool`  
**Required:** No  
**Description:** If enabled, prevents triggering any heal events during passive regeneration ticks generated by `EntityPassiveHpRegeneration`. Keep it disabled if you need to track passive regeneration in a combat log or if you have effects that trigger on heal events and should also apply to passive regeneration.
If your game doesn't require any of the above and you want to minimize overhead, you can enable this option and skip all heal-related logic during regeneration ticks. Useful if your game has a lot of entities with passive regeneration and you want to optimize performance.
If you need both to keep sending regeneration events and to minimize overhead, I advise to increase the regeneration interval and to keep the "Suppress Passive Regeneration Events" option disabled. This way you can have less regeneration ticks and still trigger events for each tick.

#### Manual Health Regeneration Stat
**Type:** `Stat`  
**Required:** No  
**Description:** Determines health regenerated when triggering manual regeneration via API. The amount of health regenerated is equal to the value of this stat.

**Use Cases:**
- **Turn-based systems:** Regenerate health at the end of each turn
- **Rest mechanics:** Trigger regeneration when resting at campfires
- **Time-skip systems:** Apply regeneration for elapsed time

**API Usage:**
```csharp
// Trigger manual regeneration
entityHealth.ManualHealthRegenerationTick();
```

---

### Attribution

Astra Framework lets one entity be marked as owned by another via `EntityCore.Owner` — for example, a `Primary Weapon` child entity owned by the `Spaceship` entity that carries it. See [Entity Ownership](https://electricdrill.github.io/AstraRpgFrameworkDocs/MD/workflows.html#entity-ownership) in the Framework documentation for how to configure **Owner** and **Owner Resolution**.

When damage is dealt by an owned entity, several Astra Health systems need to decide whether the literal performer (e.g., the weapon) or its resolved owner (e.g., the ship) is the one that matters for a given outcome. Three independent fields control this, each an `EntityAttribution` value:

| Value | Resolves to |
|---|---|
| **Direct** *(default)* | The performer itself, ignoring ownership. |
| **Owner** | The performer's immediate `Owner`, or the performer itself if unowned. |
| **Root** | The top-most entity in the performer's ownership chain, or the performer itself if unowned. |

> [!NOTE]
> **Direct** ignores ownership entirely and preserves the behavior of every project created before this feature existed. Set any of the three fields below to **Owner** or **Root** only for the systems that should look past the performer — they are independent, so redirecting lifesteal does not redirect kill credit or vice versa.

#### Lifesteal Attribution
**Type:** `EntityAttribution`  
**Required:** No  
**Description:** Resolves the lifesteal beneficiary from the damage performer. See [Lifesteal and Ownership](lifesteal.md#lifesteal-and-ownership).

#### Kill Credit Attribution
**Type:** `EntityAttribution`  
**Required:** No  
**Description:** Resolves the entity credited with a kill from the damage performer. Drives `EntityDiedContext.Performer` and the [Direct Kill](experience-collection.md#direct-kill) experience collection strategy.

#### Damage Stats Attribution
**Type:** `EntityAttribution`  
**Required:** No  
**Description:** Resolves the fallback entity whose stats supplement the damage performer's own stats in the [Damage Calculation Pipeline](damage.md#damage-calculation-pipeline) — for example, when a `DamageTypeSO`'s Defense Penetration relies on a stat that the performer doesn't define. The performer's own stats always take precedence; the resolved owner's stats are consulted only for stats the performer doesn't have. See [Defense Penetration](damage.md#defense-penetration).

---

### Lifesteal

Lifesteal has **two configuration layers**:

- a **generic** layer configured in `AstraHealthConfigSO`
- an optional **type-specific** layer configured directly on each `DamageTypeSO`

Both layers use `LifestealStatConfig` and can stack on the same hit.

#### Generic Lifesteal
**Type:** `LifestealStatConfig`  
**Required:** No  
**Description:** Generic lifesteal configuration applied to all damage dealt by an entity, regardless of `DamageTypeSO`.

**Typical Settings:**
- Lifesteal percentage stat
- Heal source for generic lifesteal
- Amount selector (initial, step, or final damage)

If the embedded **Lifesteal Stat** is not assigned, generic lifesteal is effectively disabled. This generic contribution stacks with any lifesteal configured on the specific `DamageTypeSO` of the hit.

#### Lifesteal Stat Source
**Type:** `LifestealStatSource`  
**Required:** No  
**Description:** Selects whose stats feed the lifesteal rate lookup when **Lifesteal Attribution** resolves the performer to a different entity. See [Lifesteal and Ownership](lifesteal.md#lifesteal-and-ownership).

| Value | Behavior |
|---|---|
| **Performer** *(default)* | Reads the **Lifesteal Stat** from the damage performer only. Preserves the pre-ownership behavior. |
| **Beneficiary** | Reads the **Lifesteal Stat** from the resolved beneficiary only. |
| **Performer Then Beneficiary** | Reads from the performer if it has the stat, otherwise falls back to the beneficiary's. |
| **Sum** | Adds the performer's and the beneficiary's **Lifesteal Stat** values together. |

#### Suppress Lifesteal Events
**Type:** `bool`  
**Required:** No  
**Description:** If enabled, prevents triggering heal events for lifesteal heals. Useful when many entities can trigger lifesteal frequently and you do not need listeners, combat-log entries, or UI feedback for those heals.

#### Unify Lifesteal Heals
**Type:** `bool`  
**Required:** No  
**Description:** Controls how **Generic Lifesteal** and **Damage-Type Lifesteal** are applied when both contribute to the same hit. It is enabled by default.

**Behavior:**
- **Enabled** *(default)*: the two amounts are summed into a single heal; the damage type's `HealSourceSO` takes precedence, with fallback to the generic one
- **Disabled**: generic and type-specific lifesteal are applied as two separate heals, each with its own `HealSourceSO`

See also: [Lifesteal](./lifesteal.md)

---

### Experience

#### Default Exp Collection Strategy
**Type:** `ExpCollectionStrategySO`  
**Required:** No  
**Description:** The fallback strategy used by any `ExpCollector` that does not have a custom strategy assigned on the component itself.

Setting this once is usually all you need: every collector in the project that relies on the default will automatically use it without requiring per-entity configuration. Override it on individual `ExpCollector` components only when a specific entity needs different award logic.

See also: [Experience Collection](./experience-collection.md)

---

### Death

#### Default On Death Game Action <span style="color:red;">*</span>
**Type:** `GameAction`  
**Required:** Yes  
**Description:** The game action executed when an entity dies (if the entity doesn't have its own on-death game action). Use a composite game action to chain multiple effects.

**Common on-death Game Action Ideas:**
- **Destroy GameObject** - Removes the entity from the scene
- **Ragdoll** - Enables ragdoll physics
- **Respawn** - Respawns the entity after a delay
- **Loot Drop** - Spawns loot and destroys the entity

**Example:**
```
Death → Execute composite on-death Game Action → [Spawn Death VFX → Drop Loot → Destroy GameObject]
```

#### Default On Resurrection Game Action
**Type:** `GameAction`  
**Required:** No  
**Description:** The game action executed when an entity is resurrected (if the entity doesn't have its own on-resurrection game action). Use a composite game action to chain multiple effects.

**Common on-resurrection Game Action Ideas:**
- **Simple Resurrection** - Restores health and enables the entity
- **Resurrection VFX** - Plays visual effects during resurrection
- **Stat Penalties** - Applies temporary debuffs after resurrection

#### Default Resurrection Source <span style="color:red;">*</span>
**Type:** `HealSourceSO`  
**Required:** Yes  
**Description:** The heal source used when an entity is resurrected via convenience overload `Resurrect` methods, or via the built-in [Resurrect Game Action](resurrection.md#the-resurrect-game-action).

**Use cases:**
- Categorizes resurrection healing separately from normal healing
- Allows effects like "Increase Resurrection Healing by 50%"
- Used for analytics and gameplay feedback

---

### Global Events

Global Events are `GameEvent` ScriptableObjects that broadcast health-related information to the **entire game**. They are assigned once in the configuration asset and shared by all entities, ensuring a single authoritative source for framework-level communication.

> [!WARNING]
> Three events are **required** — the framework logs an assertion error at startup if they are missing. Assign them before entering Play Mode.

#### Global Pre Damage Info Event <span style="color:red;">*</span>
**Type:** `PreDamageGameEvent`  
**Required:** Yes  
**Description:** Raised before any entity takes damage and before the [damage calculation pipeline](./damage.md#damage-calculation-pipeline) runs. Carries the full `PreDamageContext` (raw amount, type, source, dealer, critical state). Subscribe to this event to implement effects that inspect or modify incoming damage, such as damage-amplifying debuffs or conditional immunity logic.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<PreDamageGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<PreDamageGameEvent>(myEntitySpecificEvent);
```

#### Global Damage Resolution Event <span style="color:red;">*</span>
**Type:** `DamageResolutionGameEvent`  
**Required:** Yes  
**Description:** Raised after any entity takes or ignores damage. Carries the full `DamageResolutionContext` including the outcome (`Applied` or `Prevented`), net amount, and prevention reasons. Also used internally by the framework for lifesteal resolution.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<DamageResolutionGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<DamageResolutionGameEvent>(myEntitySpecificEvent);
```

#### Global Entity Died Event <span style="color:red;">*</span>
**Type:** `EntityDiedGameEvent`  
**Required:** Yes  
**Description:** Raised when an entity's HP reaches the death threshold. Carries `EntityDiedContext` with the entity reference and the damage context that caused the death. Used by the [Experience Collection](experience-collection.md) system.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<EntityDiedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<EntityDiedGameEvent>(myEntitySpecificEvent);
```

#### Global Max Health Changed Event
**Type:** `EntityMaxHealthChangedGameEvent`  
**Required:** No  
**Description:** Raised when any entity's total max HP changes (both increases and decreases). Carries `EntityMaxHealthChangedContext`. Useful for updating UI, recalculating derived stats, or triggering game-wide reactions to stat changes.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<EntityMaxHealthChangedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<EntityMaxHealthChangedGameEvent>(myEntitySpecificEvent);
```

#### Global Health Changed Event
**Type:** `EntityHealthChangedGameEvent`<br>
**Required:** No<br>
**Description:** Raised whenever any entity's current HP changes, whether increasing or decreasing. Carries `EntityHealthChangedContext`. This is the global event resolved by the corresponding Health Changed Event Channel on each `EntityHealth`, and replaces the previous split between gained-health and lost-health global events.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<EntityHealthChangedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<EntityHealthChangedGameEvent>(myEntitySpecificEvent);
```

#### Global Pre Heal Event
**Type:** `PreHealGameEvent`  
**Required:** No  
**Description:** Raised before any entity is healed and before healing modifiers are applied. Carries `PreHealContext` (base amount, source, healer, target). Subscribe to modify heal amounts or trigger pre-heal effects.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<PreHealGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<PreHealGameEvent>(myEntitySpecificEvent);
```

#### Global Entity Healed Event
**Type:** `EntityHealedGameEvent`  
**Required:** No  
**Description:** Raised after any entity is healed. Carries `ReceivedHealContext` with the final net heal amount after all modifiers.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<EntityHealedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<EntityHealedGameEvent>(myEntitySpecificEvent);
```

#### Global Entity Resurrected Event
**Type:** `EntityResurrectedGameEvent`  
**Required:** No  
**Description:** Raised after any entity is resurrected. Carries `ResurrectionContext` with HP before and after resurrection.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<EntityResurrectedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<EntityResurrectedGameEvent>(myEntitySpecificEvent);
```

#### Global Health Ratio Changed Event
**Type:** `HealthRatioChangedGameEvent`  
**Required:** No  
**Description:** Raised whenever the HP/MaxHP ratio changes — either because current HP changed (`AddHealth`/`RemoveHealth`) or because MaxHP changed (`RecalculateMaxHp`). Carries `HealthRatioChangedContext` with raw HP and MaxHP values before and after the change, plus `PreviousValue` and `NewValue` as integer HP percentages (0–100). Use this event to implement HP-ratio-based reactions such as entering a low-health state or triggering threshold-gated abilities.

**Per-entity extra events API:**
```csharp
entityHealth.Subscribe<HealthRatioChangedGameEvent>(myEntitySpecificEvent);
entityHealth.Unsubscribe<HealthRatioChangedGameEvent>(myEntitySpecificEvent);
```

---

## Troubleshooting

### "No Configuration found!" error

**Cause:** No configuration is assigned and no fallback exists.

**Solution:**
1. Check **Project Settings → Astra Health**
2. Assign a configuration or create a new one
3. Alternatively, create a config named `Astra Health Config` in a `Resources` folder

### "Using Fallback" warning

**Cause:** No explicit configuration assigned in Project Settings.

**Solution:**
1. Open **Project Settings → Astra Health**
2. Assign the fallback configuration explicitly
3. This warning is just informational and won't break functionality

### Configuration not updating in Play Mode

**Cause:** Configuration is cached on first access.

**Solution:**
```csharp
// Force reload
AstraHealthConfigProvider.Reset();
```

### Missing Resources folder

**Cause:** The `Assets/Resources/` folder doesn't exist.

**Solution:** The package creates it automatically. If it's missing, create it manually:
1. Right-click `Assets`
2. Create → Folder → Name it `Resources`
3. Restart Unity to trigger the bootstrapper
