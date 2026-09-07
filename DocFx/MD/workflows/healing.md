# Healing

An entity can be healed in 4 different ways:
- Direct healing through the `Heal` method.
- Health regeneration. This can be passive, through the dedicated `EntityPassiveHpRegeneration` component plus the [Health Regeneration](package-configuration.md#health-regeneration) configuration, or it can be manually activated via the `ManualHealthRegenerationTick` method. Also for the latter, you must configure the statistic to consider for the calculation of healing through the configuration.
- Through lifesteal, that is healing the entity based on the damage dealt. However, lifesteal functioning is detailed in [Lifesteal](lifesteal.md) as it deserves a dedicated section.
- Via resurrection with the two `Resurrect` methods. One to resurrect the entity with a percentage of HP, and one to resurrect it with a fixed amount of health points. Applicable only if the entity is dead. Also resurrection is detailed in a dedicated section, [Resurrection](resurrection.md).

> [!NOTE]
> Regeneration (both passive and manual), lifesteal, and resurrection all use, behind the scenes, the `Heal` method to effectively heal the entity.

We will first introduce some concepts that are common to all the four healing methods, and then we will analyze direct healing and health regeneration. As aforesaid, lifesteal and resurrection are analyzed in their respective sections, [Lifesteal](lifesteal.md) and [Resurrection](resurrection.md).

## Heal Source
*Relative path:* `Heal Source`  

A `HealSourceSO` represents the source of healing. Some examples of `HealSourceSO` could be: healing potions, skills, passive regeneration, lifesteal, system (e.g., the player is healed to max health during the tutorial by a support machine), and so on. The `HealSourceSO` is an important concept as it allows distinguishing between different sources of healing, and thus applying specific healing modifiers for each healing source, in addition to the generic healing modifiers that apply to all healing sources.
Furthermore, `HealSources` are important because they are communicated in healing contexts, and therefore in the listeners of healing events, allowing the latter to react specifically based on the healing source that triggered the event. Thanks to this, you could, for example, implement a buff that increases the healing received from healing potions.
Finally, `HealSources` pave the way for the implementation of a combat log that tracks not only the amount of healing received but also the healing source, allowing the player to have a deeper understanding of what healed them most effectively in a specific game situation.

An instance of `HealSourceSO` in the inspector looks like this:  
![](../../images/AstraRPG/workflows/healing/heal-source-resurrection.png)

You can notice that there are two properties to configure for each `HealSourceSO`:
- **Flat Heal Modification Stat**: the statistic to consider in an entity to apply flat, specific healing modifiers for this healing source. Positive values of this statistic increase the healing received from this source, while negative values decrease it. If the entity does not have this statistic, a value of 0 will be considered for this statistic, and therefore no specific flat healing modifier will be applied for this healing source.
- **Percentage Heal Modification Stat**: the statistic to consider in an entity to apply percentage, specific healing modifiers for this healing source. Positive values of this statistic increase the healing received from this source, while negative values decrease it. If the entity does not have this statistic, a value of 0 will be considered for this statistic, and therefore no specific percentage healing modifier will be applied for this healing source.

## Heal Modifiers
Heal modifiers are statistics that influence the amount of healing received by an entity. They can influence it positively or negatively, depending on the value. There are two types of healing modifiers: generic heal modifiers, which apply to all healing sources, and source-specific heal modifiers, which apply only to a specific healing source.
Furthermore, healing modifiers can be flat or percentage-based. Flat healing modifiers influence the amount of healing received by adding or subtracting a fixed amount of HP to the healing, while percentage healing modifiers influence the amount of healing received by multiplying the healing by a certain factor. It is important to note that flat healing modifiers are applied before percentage healing modifiers. Therefore, percentage modifications will also affect the impact of flat healing modifiers.  
For percentage modifiers, a statistic with a value of 120 increases the healing by 120%, while a statistic with a value of -25 decreases the healing by 25%.

Generic healing modifiers are summed with the source-specific ones: flat modifiers are added together, as are the percentage ones.

Finally, if an entity does not possess the statistic associated with a healing modifier in its Stat Set, a value of 0 will be considered for that statistic, and therefore no healing modifier will be applied for that statistic.

Clearly, to provide healing modifiers to entities, since they are statistics, you must proceed through the Stat Modifiers APIs, as described in the [Understanding Stat Modifier Types](https://electricdrill.github.io/AstraRpgFrameworkDocs/MD/workflows.html#understanding-stat-modifier-types) section of the base framework documentation.

### Example of Heal Modifiers

Suppose we want to calculate the healing received by an entity using a healing potion. Here are the example values:

**Definition of the statistics used in the example:**
- **Potion Heal Flat Modifier**: Source-specific statistic (the potion) that adds or subtracts a fixed value to the HP healed by the potion.
- **Potion Heal Percentage Modifier**: Source-specific statistic (the potion) that increases or decreases the potion's healing by a percentage.
- **Generic Flat Heal Modifier**: Generic statistic that applies to all healing sources, adding or subtracting a fixed value to the HP healed.
- **Generic Percentage Heal Modifier**: Generic statistic that applies to all healing sources, increasing or decreasing the healing by a percentage.

| Phase | Value | Calculation | Result |
|------|--------|---------|-----------|
| Base healing | 80 HP | - | 80 HP |
| Flat modifiers | Potion Heal Flat Modifier: 50<br>Generic Flat Heal Modifier: -10 | 50 - 10 = 40 HP<br>80 + 40 | 120 HP |
| Percentage modifiers | Potion Heal Percentage Modifier: 20%<br>Generic Percentage Heal Modifier: 10% | 20% + 10% = 30%<br>120 × 1.3 | 156 HP |

**Final healing received: 156 HP**

This table clearly shows how flat and percentage modifiers are summed, and how the final result is reached.



## Direct Healing
The `Heal` method is the most straightforward way to heal an entity. It takes a `PreHealContext` as input, which contains all relevant information about the healing you intend to apply and operates as follows:
1. It checks if the entity is dead. If the entity is dead, will raise an exception, as you cannot heal a dead entity. If you want to resurrect a dead entity, check the [Resurrection](resurrection.md) section.
2. It checks if the healing to be applied is marked as a critical hit and, if so it applies the critical multiplier to the healing amount.
3. It retrieves the generic and `HealSourceSO`-specific **flat** heal modifiers for the entity.
4. It retrieves the generic and `HealSourceSO`-specific **percentage** heal modifiers for the entity.
5. It retrieves the heal flat modifiers specific to the heal source, and sums them with the generic flat heal modifiers.
6. It retrieves the heal percentage modifiers specific to the heal source, and sums them with the generic percentage heal modifiers.
7. It applies the flat modifiers to the base healing amount, and then applies the percentage modifiers to the result, to calculate the final healing amount.
8. It increases the entity's current HP by the final healing amount.
9. It raises the _Entity Healed Event_ with a `HealResolutionContext` containing all relevant information about the healing that was just applied.
10. It returns the `HealResolutionContext` created in the previous step so that the caller can use it for further processing if needed.

The `Heal` method, differently from the `TakeDamage` method, cannot be configured to reorder the steps of the healing pipeline. This choice was made as healing is generally more straightforward than damage, and there are usually no mechanics that require reordering the steps of the healing pipeline. Therefore, I evaluated simplicity and ease of use more important than flexibility for this method.

### `Heal` Usage Example
Suppose we want to heal an entity for 20% of its total max HP. The code could be the following:

```csharp
// Assuming that:
// - healSource is a HealSource representing the healing coming from a skill
// - skillCaster is the EntityCore that casts the healing skill
// - target is the EntityCore that we want to heal

if (target.TryGetComponent(out EntityHealth targetHealth)) {
    // in case the target has a EntityHealth component...
    long healAmount = target.GetMaxHpPortion(0.2d);

    PreHealContext preHealContext = PreHealContext.Builder
        .WithAmount(healAmount)
        .WithSource(healSource)
        .WithTarget(target.EntityCore)
        .WithHealer(skillCaster)       // optional
        .Build();

    targetHealth.Heal(preHealContext);
}
```

It is not a good practice to hardcode the healing amount directly in code (20% of max hp in the example). Hardcoded values make tuning and balancing difficult, prevent reuse across different skills or items, and force code changes and recompilation for values that designers or content creators should be able to tweak. Prefer one of these approaches instead:

- **Serialize the amount** as an inspector-editable field so designers can tweak values without touching code.
- **Use a `ScalingFormula`** (or equivalent) to compute the heal amount dynamically from the caster's or target's stats, allowing the effect to scale with progression and remain data-driven.

Example using a `ScalingFormula` to calculate the heal amount at runtime:

```csharp
// Assuming that scalingFormula is a ScalingFormula that calculates the heal amount
// based on the skill caster's stats...
long healAmount = scalingFormula.CalculateValue(skillCaster);

PreHealContext preHealContext = PreHealContext.Builder
    .WithAmount(healAmount)
    .WithSource(healSource)
    .WithTarget(target.EntityCore)
    .WithHealer(skillCaster)       // optional
    .Build();

targetHealth.Heal(preHealContext);
```

Recommended pattern when healing programmatically:
1. Construct a `PreHealContext` with all relevant information using the fluent builder.
2. Call `Heal` passing the constructed context.

The `PreHealContext` fluent builder is helpful because it guides you through required inputs first — the IDE will typically present required builder steps one at a time. When the builder presents multiple fields together, those fields are optional and can be omitted. Optional fields include the healer entity (`WithHealer`), the critical hit flag, the critical multiplier, the instigator (`WithInstigator`), and the reactability flag (`WithIsReactable`). `IsReactable` defaults to `true`; set it to `false` when building a secondary heal that reacts to another event, to prevent chain reactions. See [Preventing Infinite Cycles](./game-actions.md#preventing-infinite-cycles) for details.

> [!TIP]
> **The fluent `Builder` is the recommended default.** On very high-frequency heal paths (for example a heal-over-time applied to many entities every tick) you can use `PreHealContext.Create(...)` instead: it builds the context directly, without allocating the intermediate step-builder object. It accepts the same values the builder does. Reach for it only where profiling shows the builder allocations are a real cost.

## Health Regeneration
Health regeneration is a mechanism that allows an entity to passively regenerate its health over time. In games generally, this mechanic is implemented through discrete regeneration ticks. Ticks, depending on the game, can be marked by the passage of time, or by certain events, like the end of a turn in a turn-based game.  
This package supports both passive and manual regeneration.

### Passive Health Regeneration
Passive regeneration is handled by the dedicated `EntityPassiveHpRegeneration` MonoBehaviour. Attach `EntityPassiveHpRegeneration` alongside `EntityHealth` on every entity that should regenerate automatically over time.

![Entity Passive HP Regeneration Component](../../images/AstraRPG/workflows/healing/entity-passive-hp-regeneration-component.png)


The component reads its automatic-tick settings from the [Health Regeneration](package-configuration.md#health-regeneration) section of the package configuration. The following parameters should be configured there:
- **Health Regeneration Source**: the `HealSourceSO` to associate with regeneration (passive and manual). The package will use this `HealSourceSO` to create the healing contexts to pass to the `Heal` method when it's time to regenerate health.
- **Passive Health Regeneration Stat (HP/10s)**: the amount of HP the entity regenerates every 10 seconds. The package will take care of scaling this amount based on the time that passes between consecutive ticks, to ensure smooth and consistent regeneration.
- **Passive Health Regeneration Interval**: the time interval, in seconds, between one regeneration tick and the next.

Configuring these values alone is not enough: only entities that also have `EntityPassiveHpRegeneration` attached will receive automatic regeneration ticks.

To give an example, if we want an entity to regenerate 5 HP every 0.5 seconds, we must configure the "Passive Health Regeneration Interval" parameter to 0.5, ensure the entity has `EntityPassiveHpRegeneration` attached, and set the statistic associated with "Passive Health Regeneration Stat (HP/10s)" to a value of 100 (5 HP every 0.5 seconds correspond to 10 HP per second, which correspond to **100 HP every 10 seconds**).

#### Performance Considerations
Before making any considerations about performance, I want to remind you that premature optimization is the root of all evil. Before taking actions to optimize performance, a real performance problem should be identified through profiling.
That said, since by default passive regeneration raises the `Entity Healed` Event Channel, which in turn raises the global `Entity Healed` event and any extra `Entity Healed` events assigned to that entity, it's important to consider the performance impact if you have a large number of entities with `EntityPassiveHpRegeneration` regenerating health simultaneously with a very short regeneration interval. If there were also several listeners subscribed to the global `Entity Healed` event, you would get several function calls at each regeneration tick X each entity.
If this becomes a performance issue for your game, you have several levers available to optimize performance:
- **Increase the regeneration interval**: by increasing the regeneration interval, the number of regeneration ticks per time unit is reduced, and therefore the number of Entity Healed Events raised.
- **Disable the raising of Entity Healed Events for passive regeneration**: if you don't need to react to passive regeneration through the listeners of Entity Healed Events, you can disable the raising of these events for passive regeneration through the configuration. This way, even if you have a large number of entities regenerating health simultaneously with a very short regeneration interval, you won't have a significant performance impact due to raised events and associated function calls.
- **Use Extra Entity Healed Events**: if you only need to react to the passive regeneration of one or some specific entities, you can configure these entities with a dedicated extra `Entity Healed` event on their `Entity Healed` Event Channel, and subscribe your listeners to this extra event instead of the global `Entity Healed` event. This way, the only function calls you'll have at each tick will be those of the listeners subscribed to the specific extra event of the entities configured to raise it, thus reducing the performance impact.

I would like to reiterate that, unless you have identified a real performance problem through profiling, it is not necessary to take any of these actions to optimize performance. You probably won't have any performance issues even with a thousand entities and dozens of listeners subscribed to the global Entity Healed Event.

### Manual Health Regeneration
Manual regeneration is instead triggered manually through the [`ManualHealthRegenerationTick()`](xref:ElectricDrill.AstraHealth.Core.EntityHealth.ManualHealthRegenerationTick) API method.

This method, when called, triggers a regeneration tick for the entity. Note that the statistic considered for the health calculation does NOT coincide with the one configured for passive regeneration, but is a separate statistic, also set via the package configuration, specifically: [Manual Health Regeneration Stat](package-configuration.md#manual-health-regeneration-stat).
The value of this statistic determines the amount of HP regenerated at each manual regeneration tick. Obviously, healing modifiers, both generic and those specific to the heal source configured for regeneration, will also affect manual regeneration, just like passive regeneration.

### Passive vs Manual Health Regeneration: Which One Should I Use?
If you have doubts about which type of regeneration to use, here are some guidelines that might help you choose:
- **Real Time**: if your game is in real-time, passive regeneration is generally more suitable, as it integrates better with continuous gameplay flow.
- **Turn-Based**: if your game is turn-based, manual regeneration is generally more suitable, as you can trigger it at the end of each turn, ensuring more precise control over the game flow.
- **Regeneration in the base**: if your game involves a regeneration mechanic that only happens when the character is in a specific location (e.g., a base), you can use passive regeneration, but enable `EntityPassiveHpRegeneration` only when the character is in that location, and disable it when they leave. If, on the other hand, you want it to be active even when the character is outside the base, but still want it to be stronger when in the base, you can assign a specific Heal Modifier for the regeneration's Heal Source when the character is in the base, to increase the amount of HP regenerated when the character is at the base. Upon leaving the base, you can remove this Heal Modifier to reduce the amount of HP regenerated.
- **Regeneration out of combat**: if your game features regeneration that only happens when the character is out of combat, you can use passive regeneration, enabling `EntityPassiveHpRegeneration` when out of combat, and disabling it when entering combat. Or, if you want it to be active during combat as well, but stronger out of combat, you can (as with base regeneration) add a specific Heal Modifier for the regeneration's Heal Source when the character is out of combat.
- **Regeneration as a buff**: if the entity receives a buff providing HP regeneration according to a specific timing, you can use manual regeneration to trigger regeneration ticks perfectly aligned with the timing intended by the buff.

Clearly, these are just guidelines, not strict rules. For every problem, there are always several possible solutions, and choosing the best one always depends on the specific context of your game and the mechanics you want to implement. So, if you feel a solution different from the suggested ones fits your game better, feel free to use it.
