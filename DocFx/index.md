---
_layout: landing
---

> [!NOTE]
> Join the Astra Discord server!  
> There is now a dedicated **Discord server** for Astra Framework and its extensions.
> Join to **receive notifications** about new extension releases and important updates, to **ask for new features**, **report bugs**, **share ideas**, and **showcase your Astra creations** with other developers.  
> <span style="font-size:1.18em; font-weight:600;">💬 Join the Discord Server: https://discord.gg/nJVRMkGrZg</span>

## 💚 Astra Health

### 👉 Introduction

**A complete health and damage system built on Astra Framework's data-driven foundation.**

Astra Health adds robust health management to your entities. Handle damage, healing, death, and resurrection with a flexible pipeline that adapts to your game's mechanics.

Simple to set up, powerful in action. Customize damage types, reduction formulas, and calculation pipelines without touching code.

---

### ✨ Key Features

- ❤️ **Health Management:** Complete system for HP, damage, healing, death, and resurrection
- 🗡️ **Damage Types & Sources:** Define custom damage types with defensive stats and armor penetration. Define Damage Sources for damage tracking, or advanced game mechanics
- 🛡️ **Damage Mitigation:** Multiple reduction functions—flat, percentage, logarithmic, or custom
- ⚙️ **Damage Pipeline:** Fully customizable calculation pipeline with reorderable steps
- 🩹 **Healing System:** Heal sources and mechanics that integrate seamlessly with your game
- 🧛 **Lifesteal:** Configurable lifesteal that works with any damage type
- 💀 **Death & Resurrection:** Flexible strategies for death behavior and bringing entities back
- 📈 **Health Scaling:** New scaling component for formulas based on current, max, or missing HP
- ⚡ **Health Events:** Extended game events for damage, healing, death, and resurrection

---

### ⚙️ Implemented on top of `ScriptableObjects` architecture

- 🎨 **Inspector-driven:** Configure damage types, sources, and strategies directly in the Unity Inspector
- 🧩 **Modular and reusable:** Share damage calculation strategies across entities or customize per-entity
- 🧪 **Test in play mode:** Tweak damage formulas, reduction functions, and pipeline steps without recompiling and without restarting play mode

---

### 🔍 Features Breakdown

#### ❤️ Health Management

The new `EntityHealth` MonoBehaviour enhances your entities with health management. Any entity with this component can take damage, die, and even resurrect.

#### 🗡️ Damage Types

Damage Types are bound to your game's specific mechanics. Being built on top of `ScriptableObjects`, you're free to define your own at will. They model the defensive statistics and damage mitigation mechanics—for example, "Physical Damage" could be reduced by the "Armor" statistic.

While configuring a Damage Type, you can also provide a piercing statistic for its defensive stat. For example, the "Armor Pierce" statistic could be designated to ignore part of "Armor" when calculating net damage.

#### 🎯 Damage Sources

Damage Sources, like Damage Types, are bound to the specific game you're making. They represent the origin of the damage inflicted on an entity—examples include Skill, Trap, Environment, etc. They can be used to implement specific game mechanics, such as resistances or vulnerabilities to certain sources of damage, or to track in a damage log where the damage came from.

#### 🛡️ Damage Mitigation Functions

The damage mitigation granted by defensive stats is modeled via mathematical functions. The framework comes with 4 pre-defined and very common functions to use in your game:

- **Flat damage mitigation:** Each point of the defensive stat reduces incoming damage by *n* points, with *n* ≥ 1. The easiest and most predictable damage reduction function—eases the balancing of your game.
- **Percentage damage mitigation:** The defensive stat expresses the percentage of damage reduction to apply. Useful for a defense that stays effective even when attacker/defender levels or damage-input/defensive-stat values differ greatly.
- **Hyperbolic damage mitigation:** Each point of the defensive stat removes the share `X / (X + K)` of the damage, the formula of League of Legends, Dota, World of Warcraft and Diablo. Diminishing returns without immunity, with an optional level-dependent `K`. Recommended for new projects.
- **Logarithmic damage mitigation (Legacy):** Increments to the defensive stat provide diminishing returns as the value grows more and more. Useful for preventing complete damage negation.

You can also implement your own custom damage reduction formulas and use those instead of the default ones—the framework is highly extensible.

#### 🛡️ Defense Penetration Functions

Similarly to Damage Mitigation Functions, these functions specify how a piercing stat should ignore defense when calculating the net damage to be taken.

The package offers, out of the box, the same flat, percentage, hyperbolic, and logarithmic (legacy) functions.

#### ⚙️ Damage Calculation Pipeline

The Damage Calculation Pipeline models the process of transforming raw damage—the damage an attack/ability/trap/etc. intends to deal to an entity—into net damage, the actual damage taken by the entity. It can be customized so the various steps are applied in the desired order; since the order of application is not idempotent, the pipeline can be tailored to your game's needs.

The package comes with these pre-defined steps:

- **Critical multiplier:** If the damage to be taken is marked as critical, multiplies the damage amount by the critical multiplier passed along with the damage context. For now, it's the only step that increases the damage amount—generally placed first in the pipeline.
- **Defense mitigation:** Takes into consideration the defensive stat for the incoming damage type (if any) and any possible defense penetration.
- **Barrier absorption:** "Barrier" refers to temporary hit points that, upon damage, take precedence over actual HP. This step detracts from the amount of damage to be taken by the entity's available barrier, if any.
- **Percentage damage modifiers:** Analogously to flat damage modifiers, applies the percentage damage modifiers.
- **Flat damage modifiers:** Positive or negative flat damage modifications, intended as a foundation for higher-level abstractions such as weaknesses and resistances. Applies general flat modifiers (for all damage types and sources), as well as ones specific to particular damage types and sources.
- **Damage floor:** Allows imposing a lower limit on the damage amount being processed. Useful for ensuring a minimum amount of damage.
- **Damage cap:** Analogous to damage floor, but imposes an upper limit on the damage amount being processed. Useful for bosses with traits like "cannot take more than 10% of Max HP on a single hit." Usually placed last in the pipeline.

You can configure a default damage calculation strategy to be used, by default, by all entities. If needed, you can also provide an entity with a custom damage calculation strategy—for example, to design certain bosses with additional specific steps. You could use the damage cap step, configured to cap the damage amount at 10% of max HP, and grant it to all bosses; this prevents players from cheesing bosses too quickly. You can even override the damage calculation strategy at runtime—for example, to implement a status effect that doubles all electric damage taken, by swapping in a new electric damage multiplier step that tweaks the damage amount whenever the type is electric. As the debuff wears off, the overriding strategy is removed and the entity restarts using its default or custom calculation strategy.

Finally, you can define a new damage calculation pipeline by starting from another one, promoting modularity and preventing duplication of configuration. A Boss pipeline with a 10% Max HP damage cap, for instance, can be achieved by decorating the default pipeline with an extra damage cap step.

This feature not only lets you tailor the default damage calculation pipeline to your own game, but also eases the implementation of more complex damage-related mechanics, for a robust and flexible damage calculation system.

#### 🩹 Heal Sources

Heal Sources are similar to Damage Sources, but pertain to healing instead of damage. They represent the origin of the healing received by an entity—examples include Potion, Skill, Regeneration, Lifesteal, Resurrection, Weapon Skill, etc. In some games, Heal Sources can also include Self and Ally. They can be used to implement specific game mechanics, such as bonuses to healing coming from or granted to allies or self, or bonuses to healing coming from passive HP regeneration.

#### 🧛 Lifesteal

Lifesteal is a widely used game mechanic that lets an entity regain health based on the damage it deals to others. The package provides a highly customizable Lifesteal system that can be tailored to your game's specific needs. You can configure the lifesteal statistic to associate with each damage type you're interested in, and then choose which step of the damage calculation pipeline it should apply to: before any step (raw damage), after all steps (final net damage), before or after defensive-statistic damage mitigation, before or after damage-modifiers mitigation, before or after barrier absorption, etc.

You can fine-tune lifesteal application timing to fit your game's design.

#### 💀 Death

By default, when an entity's health reaches zero, it is considered dead. The package also allows setting a custom death threshold, which can be useful for implementing mechanics such as "down but not out" or "last stand," where an entity can survive until a certain negative health value before dying.

When an entity dies, a Game Action can be executed, allowing for custom behavior upon death, such as triggering events, playing animations, or dropping loot. Beyond game mechanics, Game Actions let you tailor death behavior to your architecture. For example, if your game uses object pooling, you can implement a Game Action that returns the entity to the pool instead of destroying it. Alternatively, if an entity can resurrect, you could opt for the pre-defined "disable game object" Game Action, which simply disables the entity upon death, allowing for easy resurrection later on.

A default on-death Game Action for all entities can be set, and custom on-death Game Actions can be assigned to specific entities as needed.

The base package and this one come with several Game Actions out of the box—you can define plenty of death behaviors without even writing your own.

#### 😇 Resurrection

Resurrection is the process of bringing a dead entity back to life. An entity can be resurrected via API, providing the amount of health to restore, either as a flat value or as a percentage of its maximum health.

Similarly to on-death Game Actions, on-resurrection Game Actions can be used to define custom behavior when an entity is resurrected. The same Game Actions used for death can be used in response to resurrections as well.

#### 📈 Health Scaling Component

With the base framework, you were able to define scaling formulas based on Stats and Attributes. This package extends the expressiveness of those formulas by introducing the Health Scaling Component, which lets you scale values based on the entity's health—using maximum health, current health, or missing health as variables.

For example, your Berserk abilities could scale based on how much health the entity is missing, becoming more powerful as it gets closer to death.

#### ⚡ More Game Events

By relying on the powerful and flexible event system of the core package, this package introduces more game events related to:

- Health
- Damage
- Healing
- Death
- Resurrection

These events can be used to trigger custom behavior in your game, such as applying complex passive abilities, playing sound effects, spawning particles, updating the UI, and much more.

---

### 🛒 Where to Buy

Unity Asset Store: [Astra Health](https://assetstore.unity.com/packages/package/662643)

---

### 📬 Information & Contact

Questions, feedback, or bugs? Email us at [electricdrill.info@gmail.com](mailto:electricdrill.info@gmail.com).

Unity Forum: [Astra Health Discussion](https://discussions.unity.com/t/tired-of-rigid-combat-systems-meet-astra-health-a-modular-pipeline-based-damage-health-system-for-any-genre/1723594)

Discord: [Astra Discord Server](https://discord.gg/nJVRMkGrZg)
