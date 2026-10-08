# Project Overview — Astra Health

> **For AI assistants**: read all four files in this folder at the start of each session before touching any documentation or source code.

## What This Is

**Astra Health** is a Unity package built on top of **Astra Framework**. It adds health, damage, healing, death/resurrection, lifesteal, barrier, experience collection, and health scaling mechanics to entities managed by the Framework's stat/attribute system.

The user is writing English documentation for this package using DocFx. All conversations happen in Italian; documentation is always written in English.

---

## Folder Paths

On Windows workstation, the relevant paths are:
| Asset | Path |
|---|---|
| Health package source | `C:\Users\emaci\Documents\AstraRpgPublishing_6_3\Packages\com.electricdrill.astra-health\` |
| Framework source | `C:\Users\emaci\Documents\AstraRpgPublishing_6_3\Packages\com.electricdrill.astra-rpg-framework\` |
| Documentation markdown | `C:\Users\emaci\Documents\AstraRpgHealthDocs\DocFx\MD\` |
| Documentation images | `C:\Users\emaci\Documents\AstraRpgHealthDocs\DocFx\images\AstraRPG\` |
| AI context (this folder) | `C:\Users\emaci\Documents\AstraRpgHealthDocs\_ai-context\` |

On Ubuntu workstation, the paths are:
| Asset | Path |
|---|---|
| Health package source | `/home/emanuele/AstraTesting/Packages/individual.emanuele-cisotto.astra-health/` |
| Framework source | `/home/emanuele/AstraTesting/Packages/com.electricdrill.astra-rpg-framework/` |
| Documentation markdown | `/home/emanuele/Documents/Personal/ElectricDrill/AstraRpgHealthDocs/DocFx/MD/` |
| Documentation images | `/home/emanuele/Documents/Personal/ElectricDrill/AstraRpgHealthDocs/DocFx/images/AstraRPG/` |
| AI context (this folder) | `/home/emanuele/Documents/Personal/ElectricDrill/AstraRpgHealthDocs/_ai-context/` |

---

## Astra Framework — Key Concepts

The Framework provides the foundation. Key systems relevant to the Health package:

- **EntityCore**: base component all entities share
- **Stats (`Stat`)**: numeric values (e.g., Armor, Magic Resistance, Armor Penetration). Used as defensive stats, piercing stats, modifier stats.
- **StatSet**: a collection of stats; entities have StatSets, allowing centralized stat management.
- **Attributes**: derived values (e.g., HP max); attributes drive `EntityHealth`.
- **AttributesScalingComponent**: links attributes to stats via scaling formulas.
- **GrowthFormula**: how stats grow with level; used by `DmgReductionGraphWindow` for piercing stat simulation.
- **GameAction\<TContext\>**: async action SO; used for on-death and on-resurrection hooks.
- **Percentage**: framework utility type for percentage values.
- **IExpSource**: interface implemented by entities that can yield experience on death.

---

## Astra Health — Systems Overview

### 1. Config System
Global configuration asset accessed via `AstraHealthConfigProvider` (static singleton). Covers all subsystems.

### 2. Damage System
Fluent builder (`PreDamageContext`) → pipeline of `DamageStep` instances defined in `DamageCalculationStrategySO` → result in `DamageResolutionContext`.

### 3. Heal System
Fluent builder (`PreHealContext`) → modifiers applied → result in `ReceivedHealContext`.

### 4. Barrier System
Temporary HP absorbed before real health by `ApplyBarrierStep` in the damage pipeline. Damage types can opt out via `IgnoresBarrier`.

### 5. Death & Resurrection
`EntityHealth` fires death events; `OnDeathGameAction` executes; `Resurrect()` restores HP and fires resurrection events.

### 6. Lifesteal
`LifestealConfigSO` maps `DamageTypeSO` → (stat, heal source, amount basis). Amount basis can be initial damage, a specific pipeline step value (pre/post), or final damage.

### 7. Experience Collection
`ExpCollector` MonoBehaviour listens for `EntityDiedGameEvent`; delegates to `ExpCollectionStrategySO` to determine who collects and how much.

### 8. Health Scaling
`HealthScalingComponentSO` (a Framework `ScalingComponent`) computes scaling values based on current HP, max HP, or missing HP fractions.

---

## Runtime — Key Classes

### Root
| Class | Purpose |
|---|---|
| `EntityHealth` | Core health component; `TakeDamage`, `Heal`, `Resurrect`, barriers, passive regen, HP management |

### Config (`Config/`)
| Class | Purpose |
|---|---|
| `IAstraHealthConfig` | Interface: all configurable parameters for every subsystem |
| `AstraHealthConfigSO` | Concrete SO implementation of `IAstraHealthConfig` |
| `AstraHealthGlobalSettingsSO` | Holds pointer to active config; must live in `Resources/` |
| `AstraHealthConfigProvider` | Static service; lazy-loads config with 3-step fallback |
| `AstraHealthBootstrap` | `[RuntimeInitializeOnLoadMethod]` warm-load on startup |

### Damage (`Damage/`)
| Class | Purpose |
|---|---|
| `IDamageable` | Interface: `TakeDamage(PreDamageContext)` → `DamageResolutionContext` |
| `DamageTypeSO` | Defines a damage category; all damage reduction / modifier / true-damage fields |
| `DamageSourceSO` | Identifies damage origin; source-specific modifier stats |
| `PreDamageContext` | Fluent builder for damage input (amount, type, source, target, dealer, critical) |
| `DamageResolutionContext` | Result: outcome (Applied/Prevented), prevention reasons, final damage info |
| `DamageOutcome` | Enum: `Applied`, `Prevented` |
| `DamagePreventionReason` | Flags enum: all reasons damage can be stopped (immunity, barrier, defense, zero, dead, etc.) |
| `DamageAmountContext` | Tracks damage value at each pipeline stage with before/after records; `InitialAmount`, `Current`, `Records` |

### Damage Pipeline (`Damage/CalculationPipeline/`)
| Class | Purpose |
|---|---|
| `DamageInfo` | Mutable pipeline state container passed through each step; holds `DamageAmountContext`, type, source, prevention reasons |
| `DamageStep` | Abstract base: enforces skip-if-prevented, calls `ProcessStep()`, records amounts |
| `DamageCalculationStrategySO` | Ordered list of `DamageStep` instances; executes pipeline; per-entity override supported |
| `ApplyBarrierStep` | Consumes target's barrier before real health; skips if `IgnoresBarrier` |
| `ApplyCriticalMultiplierStep` | Multiplies damage when critical; skips if multiplier = 1 |
| `ApplyDefenseStep` | Applies `DefensiveStat` → `DamageReductionFn`; applies piercing via `DefensiveStatPiercedBy` → `DefenseReductionFn` |
| `ApplyFlatDmgModifiersStep` | Sums generic + source + type flat modifiers |
| `ApplyPercentageDmgModifiersStep` | Sums generic + source + type % modifiers; checks immunity at ≤ −100 per category |

### Damage Reduction Functions (`DamageReductionFunctions/`)
| Class | Formula |
|---|---|
| `DamageReductionFnSO` | Abstract base; `CalculateReducedDamage(amount, defensiveStatValue)` |
| `FlatDamageReductionFnSO` | `Damage - (DefensiveStat × Factor)` |
| `PercentageDamageReductionFnSO` | `Damage - (Damage × DefensiveStat / 100)` |
| `LogDamageReductionFnSO` | `Damage × (BaseValue / (BaseValue + Log(1 + DefensiveStat × ScaleFactor)))` |

### Defense Reduction Functions (`DefenseReductionFunctions/`)
| Class | Formula |
|---|---|
| `DefenseReductionFnSO` | Abstract base; `CalculateReducedDefense(piercingStat, defensiveStat, stat, applyClamp)` |
| `FlatDefenseReductionFnSO` | `Defense - (PiercingStat × Factor)` |
| `PercentageDefenseReductionFnSO` | `Defense - (Defense × PiercingStat / 100)` |
| `LogDefenseReductionFnSO` | `Defense × (BaseValue / (BaseValue + Log(1 + PiercingStat × ScaleFactor)))` |

### Heal (`Heal/`)
| Class | Purpose |
|---|---|
| `IHealable` | Interface: `Heal(PreHealContext)` → `ReceivedHealContext` |
| `HealSourceSO` | Identifies heal origin; flat + percentage heal modifier stats |
| `HealAmountContext` | Records raw, after-modifier, and net heal amounts |
| `PreHealContext` | Fluent builder for heal input (amount, source, healer, target, critical) |
| `ReceivedHealContext` | Result of heal application; contains `HealAmountContext` |
| `LifestealConfigSO` | Maps `DamageTypeSO` → (`Stat`, `HealSourceSO`, `LifestealAmountSelector`); `LifestealBasisMode` enum (Initial/Step/Final) |

### Resurrection (`Resurrection/`)
| Class | Purpose |
|---|---|
| `IResurrectable` | Interface: two `Resurrect()` overloads (flat HP or percentage) |
| `ResurrectionContext` | Event payload: target, previous HP, new HP, heal source |

### Scaling (`Scaling/ScalingComponents/`)
| Class | Purpose |
|---|---|
| `HealthScalingComponentSO` | Scaling component using `Hp`, `MaxHp`, or `MissingHp` metric fractions |

### Experience (`Experience/`)
| Class | Purpose |
|---|---|
| `ExpCollectionStrategySO` | Abstract base for exp strategies; `TryCollectExp(collector, victim, dmgContext)` |
| `ExpCollector` | MonoBehaviour; listens for kill events; delegates to strategy |
| `DirectKillExpStrategySO` | Grants exp to the entity that dealt the killing blow |
| `DamageSourceKillExpStrategySO` | Grants exp when kill matches a configured `DamageSource` |
| `FirstMatchExpStrategySO` | Composite: tries sub-strategies in order, uses first success |

### Events (`Events/`)
| Context class | Purpose |
|---|---|
| `EntityHealthChangedContext` | Payload for health gain/loss events |
| `EntityMaxHealthChangedContext` | Payload for max health change events |
| `EntityDiedContext` | Payload for death events; includes damage resolution context |
| `ResurrectionContext` | Payload for resurrection events |

Game events (SO assets) and listeners (MonoBehaviour) are auto-generated for: `EntityDied`, `EntityGainedHealth`, `EntityLostHealth`, `EntityMaxHealthChanged`, `EntityHealed`, `EntityResurrected`, `PreHeal`, `PreDamage`, `DamageResolution`.

### Game Actions (`GameActions/`)
| Class | Purpose |
|---|---|
| `ResurrectGameActionSO<TContext>` | Generic base; configurable flat/percentage HP restore |
| `ResurrectComponentGameActionSO` | Concrete SO for `Component` context |

### Utils / Exceptions
| Class | Purpose |
|---|---|
| `EntityCoreHealthExtensions` | Extension method `GetEntityHealth(this EntityCore)` with cached lookup |
| `DeadEntityException` | Thrown when health ops are attempted on a dead entity |

---

## Editor — Key Classes

### Root
| Class | Purpose |
|---|---|
| `AstraRpgHealthAssetCreation` | 13 asset-creation menu items under `Assets/Create/Astra Health/` |
| `EntityHealthEditor` | Complex inspector: Health, Damage, Death, Events sections; foldouts; extra event channel management |

### Window (`Window/`)
| Class | Opened by | Purpose |
|---|---|---|
| `LogDamageReductionGraphWindow` | `LogDamageReductionFnEditor` button or `Window → Astra Health → Log Damage Reduction Graph` | Two graphs: Base Value varying / Scale Factor varying; 5 curves each; hover tooltip |
| `DmgReductionGraphWindow` | **"Make a Damage Reduction Simulation" button in `DamageTypeEditor`** | Shows actual clamped (green) + unclamped (yellow) damage reduction for a `DamageTypeSO`; supports piercing stat + growth formula |
| `LogDefenseReductionGraphWindow` | `LogDefenseReductionFnEditor` button | Same layout as LogDamageReductionGraphWindow but for defense (piercing) reduction |

### Config (`Config/`)
| Class | Purpose |
|---|---|
| `AstraHealthSettingsProviderEditor` | Project Settings panel; active config selection with ✓/⚠/🛑 status indicators |
| `AstraHealthConfigEditor` | Inspector for `AstraHealthConfigSO`; 6 organized sections |
| `AstraHealthEditorBootstrapper` | `[InitializeOnLoad]` auto-creates GlobalSettings and assigns fallback config |

### Damage (`Damage/`)
| Class | Purpose |
|---|---|
| `DamageTypeEditor` | Inspector for `DamageTypeSO`; 4 sections + consistency warnings + "Make a Damage Reduction Simulation" button |
| `DamageSourceEditor` | Inspector for `DamageSourceSO` |
| `ConfigurableDamageStrategyEditor` | Reorderable list inspector for `DamageCalculationStrategySO`; type-picker for adding steps |

### Damage/Defense Reduction Function Editors
Each built-in function has a matching `CustomEditor` that shows a formula help box. `LogDamageReductionFnEditor` and `LogDefenseReductionFnEditor` also add an "Open Graph Visualizer" button.

### Other Editors
| Class | Purpose |
|---|---|
| `HealthScalingComponentEditor` | Inspector with add/remove list for scaling metric entries |
| `ExpCollectorEditor` | Shows active strategy with ✓/⚠ status boxes |
| `LifestealStatConfigDrawer` | Property drawer for `LifestealStatConfig`; conditional step picker |

---

## DamageTypeSO Inspector Sections

Defined in `DamageTypeEditor.cs`:

1. **Damage Reduction**: `DefensiveStat`, `DamageReductionFn`, `DefensiveStatPiercedBy`, `DefenseReductionFn`
   - Button: **"Make a Damage Reduction Simulation"** → opens `DmgReductionGraphWindow`
2. **Damage Modifiers**: `PercentageDamageModificationStat`, `FlatDamageModificationStat`
3. **True Damage Options**: `IgnoresBarrier`, `IgnoreGenericPercentageDamageModifiers`, `IgnoreGenericFlatDamageModifiers`

---

## Key Business Rules (from source)

- **Pairing rule (Damage Reduction)**: `DefensiveStat` + `DamageReductionFn` must both be set or both null → `LogWarning` + step skipped.
- **Pairing rule (Defense Penetration)**: `DefensiveStatPiercedBy` + `DefenseReductionFn` must both be set or both null → same.
- **Missing stat (Damage Reduction)**: target entity lacks `DefensiveStat` → error logged.
- **Missing stat (Defense Penetration)**: dealer lacks `DefensiveStatPiercedBy` → error logged.
- **Modifier stacking**: generic + source + type modifiers for both flat and % are summed additively.
- **Percentage immunity**: generic % ≤ −100 → `AllDamageImmune`; source % ≤ −100 → `DamageSourceImmune`; type % ≤ −100 → `DamageTypeImmune`.
- **True Damage — IgnoresBarrier**: `ApplyBarrierStep` is skipped entirely.
- **True Damage — IgnoreGenericFlatDamageModifiers / IgnoreGenericPercentageDamageModifiers**: generic modifier steps skip the global config stat for this damage type.
- **Pipeline short-circuit**: `DamageStep.Process()` skips execution if damage is already prevented or reduced to 0 by a previous step.
- **Config fallback**: `AstraHealthConfigProvider` tries GlobalSettings → Resources conventional location → logs error.
- **Dead entity**: `TakeDamage` returns `Prevented` with reason `EntityDead` immediately; `DeadEntityException` is thrown on illegal health ops.

---

## AstraHealthConfig — Sections

| Section | Key fields |
|---|---|
| Health | `HealthAttributesScaling`, `GenericFlatHealAmountModifierStat`, `GenericPercentageHealAmountModifierStat` |
| Damage | `DefaultDamageCalculationStrategy`, `GenericFlatDamageModificationStat`, `GenericPercentageDamageModificationStat` |
| Health Regeneration | `PassiveHealthRegenerationStat`, `PassiveHealthRegenerationInterval`, `ManualHealthRegenerationStat`, `HealthRegenerationSource`, suppression flags |
| Lifesteal | `LifestealConfig`, suppress events flag |
| Death | `DefaultOnDeathGameAction`, `DefaultOnResurrectionGameAction`, `DefaultResurrectionSource` |
| Experience | `DefaultExpCollectionStrategy` |

