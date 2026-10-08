# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).


## [1.1.0] - 2026-10-07
### Added
#### Runtime Features
- Added **hyperbolic damage mitigation and defense penetration** (`HyperbolicDamageMitigationFnSO` and `HyperbolicDefensePenetrationFnSO`), the `X / (X + K)` formula used by League of Legends, Dota, World of Warcraft and Diablo. Defense gives diminishing returns without ever granting immunity, and every `K` points of defense add the same number of survivable hits, which makes the stat easy to budget.
- `K` is the defense that gives 50% mitigation. It is configured as `Base Value + Per Level × level`: a constant such as League of Legends' 100, or a level-dependent value such as World of Warcraft's `400 + 85 × level`. *Level Source* chooses whether the attacker's (default) or the target's level is used.
- Added `ReductionContext`, which lets custom damage mitigation and defense penetration functions depend on more than the stat value: it gives them the target, the performer, their stats and their levels. The performer is the entity the hit is attributed to, so with a *Damage Stats Attribution* other than `Direct` a level 1 weapon owned by a level 50 hero counts as level 50. Functions written for 1.0.0 keep working unchanged.

#### Editor Features
- Added `Hyperbolic Dmg Mitigation` and `Hyperbolic Def Penetration` to the `Assets > Create > Astra Health` menu, with League of Legends-like and World of Warcraft-like presets in their inspectors.
- Added the **Hyperbolic Damage Mitigation Graph** and **Hyperbolic Defense Penetration Graph** windows (`Tools > Astra Health`). They plot the formula for a constant `K` or, for a level-dependent `K`, one curve per level up to a *Max Level* of your choice. The *Open Graph Visualizer* button in each hyperbolic asset opens the window already filled with its values.
- Added **Attacker Level** and **Defender Level** fields to the Damage Mitigation Graph, so level-dependent functions are drawn at the right level. Levels are read from the assigned entities when there are any.

### Changed
#### Editor Features
- The **logarithmic** damage mitigation and defense penetration functions are now marked **Legacy**: a warning in their inspectors and "(Legacy)" in the create menu. Their behavior is unchanged and existing assets keep working. Use the hyperbolic functions for new work. The two families are not interchangeable (at high stat values the logarithmic curve grows far more slowly), so rebalance your stats rather than swapping the assets.

## [1.0.1] - 2026-09-25
### Changed
#### *Samples*
- The Example scene is not depending on Astra Framework samples any more

## [1.0.0] - 2026-09-11
### Added
- Initial release of Astra Health.

