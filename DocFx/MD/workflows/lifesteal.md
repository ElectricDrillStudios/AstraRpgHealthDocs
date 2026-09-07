# Lifesteal

Lifesteal is a mechanic that lets an entity recover health proportional to the damage it deals. When an entity successfully damages a target, a percentage of the damage — determined by a configurable stat — is healed back to the attacker (or, for entities composed of owned sub-entities, to whichever entity is resolved as responsible for the hit — see [Lifesteal and Ownership](#lifesteal-and-ownership)).

The system supports **two lifesteal layers**:

1. **Generic Lifesteal** — configured once in `AstraHealthConfigSO` and applied to all damage dealt by the entity.
2. **Damage-Type Lifesteal** — configured directly inside each `DamageTypeSO` and applied only when that specific damage type is dealt.

Both layers use the same `LifestealStatConfig` structure, so designers configure the same three concepts in both places: the lifesteal stat, the heal source, and the damage amount selector. If both layers are configured for the same hit, their heal amounts stack. Depending on the `Unify Lifesteal Heals` flag in `AstraHealthConfigSO`, those stacked amounts can be applied either as two separate heals or as a single unified heal.

## LifestealStatConfig

`LifestealStatConfig` is the reusable data structure used by both **Generic Lifesteal** and the per-`DamageTypeSO` **Lifesteal** section.

**Lifesteal Stat**  
The stat that drives the lifesteal percentage. At runtime, its value is read according to the configured **Lifesteal Stat Source** (see [Lifesteal and Ownership](#lifesteal-and-ownership)) — by default, from the damage performer's own `StatSet` — and the heal is computed as:
> heal amount = basis damage × (lifesteal stat value / 100)

The stat value is read as a `Percentage` type, meaning the raw `long` value stored in the stat is divided by 100 internally. A stat value of `15` therefore corresponds to 15% lifesteal. For lifesteal to activate, this stat must be present in the `StatSet` of whichever entity or entities the configured **Lifesteal Stat Source** reads from.

> [!NOTE]
> The computed heal amount is rounded to a `long` using the **Lifesteal Rounding Mode** from [`HealthRoundingSettings`](../workflows/package-configuration.md#rounding-settings) (default: **Round**). This rounding is applied independently to each lifesteal contribution (generic and damage-type) before they are summed or applied as separate heals.

**Lifesteal Source**  
The `HealSourceSO` used when applying the lifesteal heal. As with any heal in the framework, the heal source determines which heal modifiers — flat and percentage — are applied to the resulting heal, so lifesteal heals can be boosted or reduced by modifiers associated with the configured `HealSourceSO`. See [Heal Source](healing.md#heal-source) for details on `HealSourceSO` configuration.

**Amount Selector**  
Determines which damage value is used as the basis for the lifesteal computation. See [Amount Selector](#amount-selector) below.

## Generic Lifesteal

`AstraHealthConfigSO` exposes a **Generic Lifesteal** field of type `LifestealStatConfig`.

If its **Lifesteal Stat** is configured and the damage dealer has that stat in its `StatSet`, every successful hit can generate lifesteal from this generic configuration regardless of the hit's `DamageTypeSO`.

Use this when you want a broad mechanic such as "all outgoing damage grants lifesteal", without repeating the same setup on every damage type.

See also: [Lifesteal](package-configuration.md#lifesteal) in Package Configuration.

## Damage-Type Lifesteal

Each `DamageTypeSO` contains its own **Lifesteal** field, also of type `LifestealStatConfig`.

This contribution is evaluated only when the resolved hit uses that `DamageTypeSO`. It stacks with **Generic Lifesteal**, making it possible to define a baseline lifesteal that applies to all damage plus an additional bonus or alternate source for specific damage types.

Use this when lifesteal should depend on the nature of the damage — for example, only physical hits lifesteal, or fire damage lifesteal should use a dedicated `HealSourceSO`.

See also: [Damage Type's Lifesteal](damage.md#damage-types-lifesteal).

### Amount Selector

The Amount Selector controls which damage value is sampled as the lifesteal basis. Three modes are available:

- **Final** *(default)*: uses the damage value after all pipeline steps — the damage actually applied to the target. This is the most intuitive mode for a straightforward "heal for a percentage of damage dealt" mechanic.
- **Initial**: uses the raw damage value before any pipeline modifications. Useful when you want lifesteal to reflect the attacker's power output regardless of the target's defenses — for example, a vampire's lifesteal that scales with their attack power rather than with how much the target's armor reduced.
- **Step**: samples the damage value at a specific pipeline step. You select a step and whether to use the value recorded before (`Pre`) or after (`Post`) that step executes, giving designers precise control over which phase of the calculation drives the heal. Refer to [Damage Calculation Pipeline](damage.md#damage-calculation-pipeline) for an overview of the available pipeline steps.

The following image shows the inspector when Step mode is selected:  
![Amount Selector - Step mode](../../images/AstraRPG/workflows/lifesteal/amount-selector-step.png)

> [!NOTE]
> If **Step** mode is selected but no step is configured, the system falls back to **Final** damage. The inspector displays a warning to indicate this condition.

> [!IMPORTANT]
> **Step** mode reads the per-step damage trace, which is opt-in and **off by default in release builds** (see [Damage Step Trace](package-configuration.md#damage-step-trace)). If you ship step-based lifesteal in a release build, set **Damage Step Trace** to `Always` in `AstraHealthConfigSO`. Otherwise the trace is unavailable at runtime: a one-time console warning is logged and step-based lifesteal falls back to the **Final** damage amount. In the editor and development builds the trace is on by default, so Step mode works out of the box there.

## Lifesteal and Ownership

By default, lifesteal credits the entity that literally dealt the damage — the damage performer. This is straightforward for most entities, but it breaks down once an entity is composed of several owned sub-entities. Consider a `Spaceship` entity (`EntityCore` + `EntityStats` for armor and speed and `EntityHealth` for HP) with a `Primary Weapon` child entity of its own (`EntityCore` + `EntityStats` for bullet damage and a lifesteal stat). When the weapon fires, the weapon — not the ship — is the damage performer, so a lifesteal stat configured on the ship would never trigger: the ship never dealt the damage, only its weapon did.

Astra Framework's [Entity Ownership](https://electricdrill.github.io/AstraRpgFrameworkDocs/MD/workflows.html#entity-ownership) system solves this with an opt-in `Owner` edge on `EntityCore`. Nesting `Primary Weapon` under `Spaceship` and setting its **Owner Resolution** to `NearestAncestor` (or assigning **Owner** explicitly) makes the weapon aware that the ship is responsible for it. Lifesteal resolves the **beneficiary** — the entity whose `EntityHealth` actually receives the heal — through this edge, controlled by the **Lifesteal Attribution** field in `AstraHealthConfigSO` (see [Attribution](package-configuration.md#attribution)):

- **Direct** *(default)*: the beneficiary is the performer itself. Preserves the pre-ownership behavior — only the entity that dealt the damage can lifesteal from it.
- **Owner**: the beneficiary is the performer's immediate `Owner`, or the performer itself if unowned.
- **Root**: the beneficiary is the top-most entity in the performer's ownership chain, or the performer itself if unowned.

> [!IMPORTANT]
> The resolved beneficiary must have its own `EntityHealth` component to actually receive the heal — lifesteal resolution runs per `EntityHealth` and only applies when the beneficiary matches that entity. If **Lifesteal Attribution** resolves to a purely structural entity with no `EntityHealth` (for example, an intermediate mounting point), the lifesteal contribution is silently lost. Reach for **Root** instead of **Owner** when intermediate entities in the chain don't carry their own `EntityHealth`.

### Multiple Nested Entities

**Owner** and **Root** diverge once more than one ownership hop separates the performer from the entity that should benefit. Extend the spaceship example with a `Turret Mount` entity in between: `Spaceship` owns `Turret Mount`, which in turn owns `Primary Weapon`.

- **Lifesteal Attribution = Owner**: a hit from `Primary Weapon` credits `Turret Mount` — its immediate owner — regardless of how deep the chain continues above it.
- **Lifesteal Attribution = Root**: the same hit walks the full chain and credits `Spaceship`, the top-most entity, no matter how many intermediate entities sit in between.

Use **Owner** when each link in the chain is itself a meaningful lifesteal beneficiary (for example, a `Turret Mount` with its own `EntityHealth` and shields to regenerate). Use **Root** when only the outermost entity should ever benefit, which is the more common case for a composed vehicle or creature.

### Lifesteal Stat Source

Resolving the beneficiary decides *who gets healed*; it does not decide *whose stat sets the percentage*. That is controlled separately by **Lifesteal Stat Source** in `AstraHealthConfigSO` (see [Lifesteal Stat Source](package-configuration.md#lifesteal-stat-source)), because the two concerns are often different: the ship may be the one that gets healed, while the rate should still come from the weapon's own lifesteal stat (a "vampiric rounds" mod on the gun itself, rather than a ship-wide trait).

- **Performer** *(default)*: reads the **Lifesteal Stat** from the damage performer only (e.g. `Primary Weapon`). Preserves the pre-ownership behavior.
- **Beneficiary**: reads the **Lifesteal Stat** from the resolved beneficiary only (e.g. `Spaceship`).
- **Performer Then Beneficiary**: reads from the performer if it has the stat, otherwise falls back to the beneficiary's.
- **Sum**: adds the performer's and the beneficiary's **Lifesteal Stat** values together, letting a weapon-level bonus stack additively with a ship-level trait.

For example:
- **Lifesteal Attribution**: `Root`.
- **Lifesteal Stat Source**: `Sum`.
- **Primary Weapon**'s Lifesteal Stat: 5 (5% lifesteal).
- **Spaceship**'s Lifesteal Stat: 10 (10% lifesteal).
- Damage dealt by `Primary Weapon` (Final Amount Selector): 200.

In this case, the beneficiary resolves to `Spaceship` by walking the full ownership chain from the weapon through `Turret Mount`, and the lifesteal rate is 5% (weapon) + 10% (ship) = 15%, so the ship is healed for 200 × 0.15 = **30 HP**. This example assumes no other damage modifications are active.

## Separate vs Unified Heals

When both **Generic Lifesteal** and **Damage-Type Lifesteal** contribute to the same hit, the framework can apply them in two different ways. By default, `Unify Lifesteal Heals` is enabled, so the two contributions are merged into a single heal.

### Separate Heals

When `Unify Lifesteal Heals` is **disabled**, each contribution produces its own heal:

- the generic contribution uses the generic `Lifesteal Source`
- the damage-type contribution uses the damage type's `Lifesteal Source`

This is the right choice when those two heal sources should remain distinguishable for heal modifiers, event listeners, combat logs, analytics, or VFX/UI feedback.

### Unified Heal

When `Unify Lifesteal Heals` is **enabled** *(default)*, the generic and damage-type lifesteal amounts are summed and applied as a **single heal**.

In this mode, the `HealSourceSO` is resolved as follows:

- use the damage type's `Lifesteal Source` if the damage-type contribution is active and that source is configured
- otherwise fall back to the generic `Lifesteal Source`

This is useful when generic lifesteal is meant to be an invisible baseline bonus and you want the final heal to behave as a single gameplay event.

## Performance Considerations

> [!NOTE]
> Before making any considerations about performance, keep in mind that premature optimization is the root of all evil. A real performance problem should be identified through profiling before taking any action.

By default, lifesteal heals raise the standard healing events (`Pre Heal` and `Entity Healed`). In games where many entities deal lifesteal damage frequently — for example, a horde of enemies all with a lifesteal stat, each landing hits several times per second — the volume of heal events generated can become significant, especially when the global `Entity Healed Event` has multiple listeners.

If profiling reveals this to be a bottleneck, the **Suppress Lifesteal Events** flag in `AstraHealthConfigSO` disables healing event emission for lifesteal heals entirely. The heal is still applied; only the events are suppressed. This option is appropriate when you do not need to react to lifesteal heals through event listeners — for example, when your UI or combat log does not track lifesteal healing specifically.

A similar consideration applies to passive health regeneration, which can also generate a high volume of heal events under certain conditions. See [Performance Considerations](healing.md#performance-considerations) in the Healing documentation for a broader discussion.

## Conditions

> [!NOTE]
> Lifesteal is evaluated after every damage application. The following conditions must all be met for it to trigger:
> - The entity checking for lifesteal must be the resolved beneficiary of the hit (see [Lifesteal and Ownership](#lifesteal-and-ownership)). Under the default **Direct** attribution this is the dealing entity itself, so lifesteal does not trigger for damage dealt by unrelated entities.
> - The beneficiary is alive at the moment of damage resolution.
> - At least one lifesteal layer is configured for the hit:
>   - **Generic Lifesteal** in `AstraHealthConfigSO`, and/or
>   - **Lifesteal** on the hit's `DamageTypeSO`.
> - For each configured layer, the corresponding **Lifesteal Stat** is present according to the configured **Lifesteal Stat Source** — on the performer, the beneficiary, or both.
> - The computed heal amount for that layer is greater than zero. A lifesteal stat of 0% produces no heal contribution.

> [!IMPORTANT]
> Lifesteal relies on the **Global Damage Resolution Event** being correctly configured on each entity's `EntityHealth`. This event is used by the framework to propagate damage outcomes across all listening entities — including the attacker, which uses it to detect its own successful hits and trigger lifesteal. Assigning a non-global event to this slot will break lifesteal and other framework features that depend on centralized damage communication. See [Global Events](entity-health.md#global-events) for setup details.
