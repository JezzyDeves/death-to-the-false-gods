# Anatomical body and wound models in prior games

Research for issue #2. Sources were read on 2026-11-17.

**Question.** How do Dwarf Fortress, Caves of Qud, RimWorld, and Cataclysm: Dark Days Ahead model bodies and wounds, and what should this game take or avoid from each? The answer is sized to the alpha fixed in issue #1: turn-based one-on-one combat, creatures authored as data files from reusable parts, three body plans (human, wolf, one strange creature), and no hit-point pool.

## How to read the evidence marks

- **[V]** means I read the cited source during this research and the claim is stated there. The URL follows the claim or sits in the source list of that section.
- **[R]** means I am recalling the claim from memory and did not check it against a source. Treat it as a lead, not a fact.
- A wiki is a community record of a game's behaviour, not the game itself. Where a wiki describes mechanics that its own editors reverse-engineered or marked uncertain, I say so.

## Answer in brief

Only Dwarf Fortress derives death from anatomy alone. The other three keep a pool somewhere: RimWorld gives every body part hit points and adds a whole-body lethal damage total, Cataclysm gives each limb hit points and gives monsters one number, and Caves of Qud uses ordinary hit points with anatomy as equipment slots and severable limbs.

The recommendation is to combine three ideas:

1. **Compose bodies the way Dwarf Fortress does, but written as explicit trees the way RimWorld does.** Reusable part definitions, reusable sub-assemblies (a pair of lungs, a five-fingered hand), and a tissue-layer plan applied by part category with the tissues passed in as arguments.
2. **Resolve a hit by walking tissue layers from the outside in**, recording a small discrete wound state per layer. Use a short resistance table per tissue and attack mode, not Dwarf Fortress's material physics.
3. **Derive consequences from tags, not from totals.** Bleeding, pain, and loss of function come from properties of the damaged tissues and parts. Death comes only from a named physiological failure: no working thought centre, no breathing, or no blood.

RimWorld contributes the readable health panel (a part list with per-injury lines plus a short list of capacities). Caves of Qud contributes part-type variants for naming strange anatomy. Cataclysm contributes a clean way to turn part state into action modifiers (limb scores), and a warning about what happens when the player and the monsters use different models.

## Dwarf Fortress

### Primitives and composition

A creature's body is assembled from three kinds of raw definition **[V]** (https://dwarffortresswiki.org/index.php/Body_token, https://dwarffortresswiki.org/index.php/Body_detail_plan_token, https://dwarffortresswiki.org/index.php/Tissue_definition_token):

- **Body templates** (`BODY`) are named bundles of **body parts** (`BP`). Each part says what it connects to: a specific part (`CON`), any part of a type (`CONTYPE`), or any part in a user-defined category (`CON_CAT`). Parts carry function tags such as `GRASP`, `STANCE`, `BREATHE`, `SIGHT`, `THOUGHT`, `NERVOUS`, `CIRCULATION`, `INTERNAL`, `EMBEDDED`, and `SMALL`, and a relative size (`DEFAULT_RELSIZE`).
- **Tissue templates** define a tissue's properties: relative thickness, healing rate, `VASCULAR` (how much it bleeds), `PAIN_RECEPTORS`, and flags such as `STRUCTURAL`, `CONNECTS`, `FUNCTIONAL`, `ARTERIES`, `MAJOR_ARTERIES`, `MUSCULAR`, and `CONNECTIVE_TISSUE_ANCHOR`.
- **Body detail plans** assign ordered tissue layers to parts by category (`BP_LAYERS`), with the tissues supplied as arguments, and set positions and relations between parts (`BP_POSITION`, `BP_RELATION`).

The vanilla v50 raws show how far the reuse goes **[V]** (read through the wiki's raw-file template, which serves `creature_standard.txt`, `creature_large_temperate.txt`, `creature_subterranean.txt`, `body_default.txt`, `b_detail_plan_default.txt`, and `tissue_template_default.txt`; creature pages such as https://dwarffortresswiki.org/index.php/Dwarf embed the same files in their Raws box):

```
dwarf   [BODY:HUMANOID_NECK:2EYES:2EARS:NOSE:2LUNGS:HEART:GUTS:ORGANS:HUMANOID_JOINTS:THROAT:NECK:SPINE:BRAIN:SKULL:5FINGERS:5TOES:MOUTH:TONGUE:FACIAL_FEATURES:TEETH:RIBCAGE]
        [BODY_DETAIL_PLAN:VERTEBRATE_TISSUE_LAYERS:SKIN:FAT:MUSCLE:BONE:CARTILAGE]
wolf    [BODY:QUADRUPED_NECK:TAIL:2EYES:2EARS:NOSE:2LUNGS:HEART:GUTS:ORGANS:THROAT:NECK:SPINE:BRAIN:SKULL:4TOES_FQ_REG:4TOES_RQ_REG:MOUTH:TONGUE:GENERIC_TEETH_WITH_LARGE_EYE_TEETH:RIBCAGE]
        [BODY_DETAIL_PLAN:VERTEBRATE_TISSUE_LAYERS:SKIN:FAT:MUSCLE:BONE:CARTILAGE]
hydra   [BODY:BASIC_2PARTBODY:7HEADNECKS:BASIC_FRONTLEGS:BASIC_REARLEGS:TAIL:2EYES:NOSE:2LUNGS:HEART:...]
        [BODY_DETAIL_PLAN:VERTEBRATE_TISSUE_LAYERS:SCALE:FAT:MUSCLE:BONE:CARTILAGE]
giant cave spider  [BODY:SPIDER:2EYES:HEART:GUTS:BRAIN:MOUTH]
        [BODY_DETAIL_PLAN:EXOSKELETON_TISSUE_LAYERS:CHITIN:FAT:MUSCLE]   plus [NOPAIN]
bronze colossus    [BODY:HUMANOID_NECK:2EYES:2EARS:NOSE:HUMANOID_JOINTS:5FINGERS:5TOES]
        [TISSUE_LAYER:BY_CATEGORY:ALL:BRONZE]   plus [NOPAIN][NOBREATHE][NOTHOUGHT], no organs, no blood
```

Fifteen of the wolf's twenty body templates are the same ones the dwarf uses. The hydra's seven heads are one template (`7HEADNECKS`) and its scales are one argument (`SCALE` in place of `SKIN`). This is the strongest prior evidence that the alpha's three body plans can share most of their data.

Inside `VERTEBRATE_TISSUE_LAYERS`, a line such as `[BP_LAYERS:BY_CATEGORY:ARM_UPPER:ARG4:25:ARG3:25:ARG2:5:ARG1:1]` gives an upper arm bone, muscle, fat, and skin with relative thicknesses 25, 25, 5, and 1. Organs get a single tissue (`[BP_LAYERS:BY_CATEGORY:HEART:HEART:100]`), and the spine is nervous tissue wrapped in bone (`[BP_LAYERS:BY_CATEGORY:SPINE:NERVE:2:ARG4:2]`) **[V]**.

Sample tissue values from `tissue_template_default.txt` **[V]**: bone has `PAIN_RECEPTORS:50`, `VASCULAR:3`, `STRUCTURAL`, `CONNECTS`, `SETTABLE`, `SPLINTABLE`. Muscle has `PAIN_RECEPTORS:5`, `VASCULAR:5`, `ARTERIES`, `MUSCULAR`. Heart tissue has `VASCULAR:10`, `FUNCTIONAL`, `STRUCTURAL`. Nervous tissue has `FUNCTIONAL` and no `HEALING_RATE`, and the wiki notes that omitting that token "will result in a tissue that never heals".

### How a hit becomes a wound

The wiki's combat page states the premise directly: "rather than having hit points, units have a collection of body parts" whose sub-parts take damage that leads to "fainting, vomiting, loss of mobility due to bone fractures or nerve damage, and eventually leading to death from organ failure or blood loss" **[V]** (https://dwarffortresswiki.org/index.php/Combat).

- **Layer penetration.** An attack carries momentum and a contact area. For each armor or tissue layer in turn, an edged attack must exceed a threshold built from the ratio of the layer's shear yield and fracture to the weapon's, scaled by contact area and sharpness. If it does, the layer is cut, momentum drops by about 5%, and the next layer is tested. If it does not, the attack is tested as blunt against the layer's impact yield and fracture, and a failed attack is converted to blunt with greatly reduced momentum. A weapon's penetration depth caps how deep a cut goes, and a cut deeper than the part is thick can sever it **[V]** (https://dwarffortresswiki.org/index.php/Material_science). The wiki says these formulas were reverse-engineered by players, dates its description to version 0.40.05, and marks the penetration-depth paragraph as needing verification.
- **Wound state is per layer.** The same page describes per-layer cut, dent, and effect fractions on a 0 to 10000 scale. A part appears to be "pulped" when every layer is fully bruised, 250% dented, or fully cut **[V]** (the wiki's own hedge). These are continuous damage accumulators per tissue layer. They are not a health pool, but they are numbers that fill up.
- **Bleeding.** Tissue `VASCULAR` values set how much a wound bleeds, and `ARTERIES` and `MAJOR_ARTERIES` add severe bleeding when cut **[V]** (tissue token page). A creature is "Faint" below 50% of its blood volume and "Pale" below 25%, and "at zero blood, the creature dies from blood loss" **[V]** (https://dwarffortresswiki.org/index.php/Blood).
- **Pain.** Bruising causes one times the tissue's `PAIN_RECEPTORS` value, partial damage two times, broken tissue three times. "Having 150+ pain causes unconsciousness." A broken bone under bruised skin, fat, and muscle is 15 + 50 × 3 = 165, so one broken bone knocks a creature out. Pain is independent of body part size, and a severed part causes none **[V]** (https://dwarffortresswiki.org/index.php/Wound).
- **Nerves.** A part tagged `NERVOUS` (the spine sections) is "the hub of nervous function"; damage "disables everything in the parent bodypart and what's below it" **[V]** (body token page). Only torn nervous tissue paralyses; a fractured spinal bone without nerve damage heals. In limbs, a wound that reaches muscle can sever a motor nerve (the limb stops working) or a sensory nerve (pain in that part disappears) **[V]** (wound page).
- **Organs.** Structural damage to the brain is instant death. The heart matters only through bleeding ("the heart itself isn't required for life"). One damaged lung impairs breathing and two usually suffocate. Gut damage causes nausea and vomiting, and pierced guts can spill out **[V]** (wound page).
- **Death.** The body token page gives the anatomical conditions: every `HEAD` part severed, every `UPPERBODY` or `LOWERBODY` part pulped or severed, every `BREATHE` part damaged or destroyed (suffocation), or damage to the `THOUGHT` part (the wiki flags this last one as needing verification). The blood page adds blood at zero **[V]**. Multiple heads are redundant, which is why a hydra survives decapitation.
- **Status vocabulary.** Each part is shown in one of six listed states: none, minor, inhibited, function loss, broken, missing **[V]** (wound page, which says the definitions are copied from the game's `init.txt`).

**Where a pool sneaks back in.** Nowhere at the level of a creature or a part. The scalars that exist are blood volume, a pain total, and the per-layer damage fractions above.

### Known weaknesses

- **Unreadable output.** Tarn Adams on the Steam release's health screen: "People miss the old health interface... We'd need to draw 100 more icons... Do you have sutures, do you have an overlapping fracture, are your intestines inside or outside your body? Sensory nerve damage, motor nerve damage, impaired ability to stand… and then the different levels of bleeding, arterial bleeds and whether or not your lungs are functioning properly. It just adds up." **[V]** (PC Gamer, 13 December 2022, https://www.pcgamer.com/dwarf-fortress-now-needs-about-100-icons-for-spilled-intestines-torn-arteries-and-bruised-tendons/)
- **Cost to build.** Adams on why the release containing the rewrite took over a year: "down in the under carcass of the game with the materials and tissues and combat stuff and all that, it had more ramifications... they're taking longer than I thought to get through every single ramification", and he hoped for "less fussing around with 'what tissues make up the toe?'" **[V]** (DF Talk 4 transcript, https://www.bay12games.com/media/df_talk_4_transcript.html). That this was the 2010 release numbered 0.31 is **[R]**.
- **Too expensive to run everywhere.** World generation does not use the full model: "it would just grind to a halt if you had thousands of people fighting each other", so it "classifies things as wounds or killing blows and then assesses what a wound might be" from a list of plausible wounds **[V]** (DF Talk 8, https://www.bay12games.com/media/df_talk_8_transcript.html; DF Talk 10 says the same, https://www.bay12games.com/media/df_talk_10_transcript.html).
- **Parts that do nothing.** The wiki records "Pancreas/liver: No effect" and, for the `CIRCULATION` tag, "Exact effects not known" **[V]** (wound page, body token page).
- **Creatures that cannot be killed sensibly.** "Creatures with no blood pose a greater challenge to kill, as they can't die of blood loss, and they will continue to fight no matter how much damage they suffer... Giant sponges were particularly famous for this" **[V]** (blood page).
- **Abrupt outcomes.** One broken bone exceeds the unconsciousness threshold (arithmetic above), and bruising both lungs to function loss "can be fatal" **[V]** (wound page).
- **Opaque rules.** Players had to reverse-engineer the penetration formulas, and the wiki still flags parts of them as unverified **[V]** (material science page).
- **Grouped parts.** Parts stacked with `NUMBER` (teeth) "cannot be individually damaged"; knocking out one knocks out the group **[V]** (body token page).

On intent, Adams in 2008: "Hit points are depressing to me. It's sort of a reflex to just have HP/MP, like a game designer stopped doing their job." He credits the tabletop game Cyberpunk for the damage system **[V]** (Game Developer, 27 February 2008, https://www.gamedeveloper.com/design/interview-the-making-of-dwarf-fortress).

## RimWorld

### Primitives and composition

- A **BodyDef** is one explicit nested tree. The human body's `corePart` is the torso, and each child lists a part definition, a `coverage` fraction, a `depth` (`Inside` or `Outside`), an optional `height`, and the apparel `groups` that protect it **[V]** (`Bodies_Humanlike.xml` in an unofficial 2018 mirror of the game's Core Defs: https://github.com/RimWorld-zh/RimWorld-Core/blob/master/Core/Defs/Bodies/Bodies_Humanlike.xml).
- A **BodyPartDef** is reusable across bodies and holds `hitPoints`, an optional `bleedRate` multiplier, `skinCovered`, and `tags` such as `BloodPumpingSource` or `BreathingSource`. The heart is 15 hit points with `bleedRate` 5 **[V]** (same mirror, `BodyParts_Organs.xml`). Tags marked `vital` are the ones whose loss kills **[V]** (`BodyPartTagDefs.xml`).
- There are no tissue layers. Depth is the two-valued inside/outside flag. Bones are parts in their own right (humerus, femur, ribcage), not layers **[V]** (https://rimworldwiki.com/wiki/Table_of_Human_Body_Parts, which lists 40 kinds of human part with hit points, coverage, and effect if destroyed).
- **Coverage** picks the part that is hit: it is "the percentage of the super-part that this part covers, before its own sub-parts claim their own percentage" **[V]** (footnote on the same table).

### How a hit becomes a wound

- **Injuries are records on parts.** Each hit adds an injury (a "hediff") of a type set by the damage (cut, crush, gunshot, burn) with a severity equal to the damage dealt. "Every body part has a health value, which is lowered every time it takes damage. When it reaches 0, the part is completely destroyed", and a part's efficiency falls in proportion to health lost **[V]** (https://rimworldwiki.com/wiki/Injury).
- **Inside and outside.** A blunt hit on an outside part has a 40% chance to also strike an inside part; some damage types propagate from an inside part outward through its parents; a cut hits up to three additional parts **[V]** (https://rimworldwiki.com/wiki/Damage_Types).
- **Pain** is the sum over injuries of severity times a per-type factor (a cut is 0.0125 per point), and a pawn is downed when pain reaches its shock threshold, 80% by default **[V]** (https://rimworldwiki.com/wiki/Pain, https://rimworldwiki.com/wiki/Injury; the `Cut` definition in the mirror's `Hediffs_Local_Injuries.xml` carries `painPerSeverity` 0.0125 and `bleedRate` 0.06).
- **Bleeding** is summed across injuries into one whole-body blood loss condition; "if blood loss reaches 100%, the pawn will die". A damaged heart bleeds five times as much and a neck four times **[V]** (injury page).
- **Capacities.** Part efficiencies roll up into about a dozen capacities (consciousness, moving, manipulation, breathing, blood pumping, blood filtration, sight, and so on). Consciousness is a formula over brain efficiency, pain, blood pumping, breathing, and blood filtration **[V]** (https://rimworldwiki.com/wiki/Health, https://rimworldwiki.com/wiki/Capacity).
- **Death** follows from losing a vital part, from consciousness, breathing, blood pumping, blood filtration, or digestion reaching 0%, or from a condition such as blood loss reaching 100% **[V]** (https://rimworldwiki.com/wiki/Death).

**Where a pool sneaks back in.** Three places.

1. Every part has hit points, so the body is dozens of small pools.
2. There is a whole-body total: death follows "taking over a total amount of damage across all body parts. For adult humans, this is 150 HP", scaled by health scale for other creatures **[V]** (death page).
3. Enemies "have an additional chance to die when downed", set by storyteller settings and colony population, not by the wound **[V]** (death page, https://rimworldwiki.com/wiki/Downed).

That these checks live in `Pawn_HealthTracker` in the game's C# is **[R]**; I did not read decompiled code.

### Known weaknesses

- The 150-point total and the death-on-downed roll are exactly the abstraction this project rules out. Both exist to serve colony pacing and prisoner capture, which a one-on-one duel does not need.
- Pain shock ends most fights before anatomy does. The wiki notes a baseline human is downed by at most 80 points of damage, well below the lethal total, which makes pain "the most common way to down pawns in combat" **[V]** (death and injury pages).
- The wiki's own pages on body parts and bleeding carry stub and rewrite notices, and the bleeding formula is described as incompletely known **[V]** (https://rimworldwiki.com/wiki/Body_Parts, injury page). The model is simpler than Dwarf Fortress's and still not fully legible to its players from outside.

### The designer's stated principle

Tynan Sylvester, RimWorld's designer, argues that "anything in the Game Model that doesn't copy into the Player Model is worthless", that a system should be judged by its story-richness, "the percentage of interactions in a game that are interesting to the player", and that detail which feeds nothing else ("hair complexity") is acceptable only as cheap flavour **[V]** (https://tynansylvester.com/2013/06/the-simulation-dream/). RimWorld's health tab is this principle applied: a list of bodily systems with their current effectiveness and a list of injuries by part, with a blood-drop icon sized by bleed rate and a bandage icon for tended wounds **[V]** (health page).

## Cataclysm: Dark Days Ahead

### Primitives and composition

- A **body_part** JSON object carries `base_hp`, `hit_size`, `hit_difficulty`, `connected_to` (the path toward a root part), `main_part`, `opposite_part`, `is_vital`, `is_limb`, `limb_types` (head, torso, sensor, mouth, arm, hand, leg, foot, wing, tail, other), `limb_scores`, `effects_on_hit`, `armor`, and `sub_parts` used for armor coverage **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/JSON/JSON_INFO.md, section "Body_parts"; https://github.com/CleverRaven/Cataclysm-DDA/blob/master/data/json/body_parts.json).
- An **anatomy** is a flat list of part ids. `human_anatomy` has twelve: eyes, head, mouth, torso, two arms, two hands, two legs, two feet **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/data/json/anatomy.json).
- There are no tissues and no organs. The project's suggestion list says that as of 2024 characters can have "non-humanoid limb configurations" **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/FREQUENTLY_MADE_SUGGESTIONS.md).

### How a hit becomes a wound

- Damage reduces the struck part's hit points. The header comments the rule: "if a limb is vital and at 0 hp, you die" **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/src/bodypart.h). The torso's `base_hp` is 60 **[V]** (body_parts.json). I did not locate the function that performs the death check.
- **Limb scores** turn part state into ability. Each part contributes named scores (`manip`, `lift`, `grip`, `block`, `breathing`, `vision`, `balance`, `footing`, `move_speed`, and others), scores fall with limb health and encumbrance, and `character_mod` objects turn scores into modifiers on specific actions **[V]** (JSON_INFO.md, sections "Limb scores" and "Character Modifiers").
- **Effects on hit** are data: a part lists effects (winded, staggered, downed) with a damage type, a damage threshold, and chance and duration scaling **[V]** (JSON_INFO.md, "On-hit Effects").
- **Wounds** are a newer data type layered on top of limb hit points. A wound has the damage types and damage range that can cause it, a weight for random selection, a per-limb limit, pain, healing time, limb score penalties, progression into worse wounds on further damage or over time, and periodic effects such as bleeding. A `wound_fix` removes or converts wounds and can restore limb hit points. The documentation suggests tiers named small, medium, bad, and terrible **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/JSON/WOUNDS.md; example data in https://github.com/CleverRaven/Cataclysm-DDA/blob/master/data/json/wounds/cut_wounds.json).
- **Blood** is tracked as a quantity that bleeding drains: "vitamin_blood ranges from -50k(death) to 0(no hypovolemia)" **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/src/character_body.cpp).

**Where a pool sneaks back in.** Per-part hit points are the base model, and wounds decorate them. Monsters do not use the body model at all: a monster has a required integer `hp` ("Monster hit points") and optional weakpoints with hit chances, damage multipliers, and effects **[V]** (https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/JSON/MONSTERS.md).

### Known weaknesses, in the developers' words

- **Two models.** The developers reject "separate limbs for monsters" as "not useful", because weakpoints already let monsters be crippled, and they prefer that "the player declare the effect they want rather than something indirect like targeting a limb" **[V]** (FREQUENTLY_MADE_SUGGESTIONS.md). This is a deliberate position, and it is the opposite of this project's fixed decision that every creature uses the same body simulation. It is worth knowing the argument: declared effects are easier to balance than emergent ones.
- **Limb loss.** It is kept out of the main game because "a lot of our stuff is missing being tied to limb scores", and because losing a limb in a fight means "your chances of surviving such an event are close to zero" without help **[V]** (same file). A model that can sever limbs must already have every action reading part state, and severing is close to a death sentence in a duel.

## Caves of Qud

### Primitives and composition

- `Bodies.xml` has three sections: **body part types** ("the core kinds of body parts"), **variants** ("sub-types that are more or less interchangeable with other parts of the same main type"), and **anatomies** (nested `<part>` trees). A creature picks one with `<part Name="Body" Anatomy="Spider" />` **[V]** (https://wiki.cavesofqud.com/wiki/Modding:Bodies; I read the wiki's quoted snippets, not the game file).
- The wiki states the purpose plainly: bodies organise creatures "primarily for purposes of equipment slots" **[V]**.
- A type carries flags such as `Appendage` (can be severed), `Mortal` (a part such as a head, whose loss can kill), `Mobility` (a weight toward movement speed), and the blueprint of the object dropped when it is severed. A variant renames a type without changing its function: `Tentacle` is a variant of `Hand`. Parts take a `Laterality`, so same-type parts are told apart as left, right, fore, and hind **[V]**.
- A 2018 developer update added body plans for birds, snakes, slugs, insects, spiders, several robot chassis, plants, fungi, and animated furniture, made the loss of mobility limbs slow a creature "in proportion to the number of mobility-granting limbs you have left", and had creature descriptions "dynamically modified with descriptions of the creatures' missing body parts" **[V]** (https://freeholdgames.itch.io/cavesofqud/devlog/50066/feature-friday-september-28-2018).

### How a hit becomes a wound

- It does not, in the anatomical sense. Hitpoints "measures how close a creature is to death". Attacks that penetrate armor reduce the total **[V]** (https://wiki.cavesofqud.com/wiki/Hitpoints_(HP)).
- Bleeding is hit-point damage per turn until a Toughness save succeeds **[V]** (https://wiki.cavesofqud.com/wiki/Bleeding).
- Dismemberment is a special event, not an outcome of damage: the axe skill gives penetrating hits a 3% or 6% chance to sever a random limb, and with Decapitate "if you chop off the only head of an organic opponent, they're killed instantly" **[V]** (https://wiki.cavesofqud.com/wiki/Dismember, https://wiki.cavesofqud.com/wiki/Decapitate).

**Where a pool sneaks back in.** It never left. Caves of Qud is a hit-point game with a composable anatomy beside it.

### Known weaknesses

- The anatomy has no depth: no tissues, no organs, no wounds other than a missing part. It cannot answer this project's injury question.
- The modding page for bodies is marked as a stub, and many of its attribute descriptions are blank **[V]**. I found no developer statement about weaknesses of the body model. Brian Bucklew's roguelike-conference talk on the game's data-driven engine is a likely source for the design reasoning, but I did not read or watch it **[R]**.

## Comparison

|                              | Dwarf Fortress                                                                   | RimWorld                                                                                   | Cataclysm: DDA                                    | Caves of Qud                                   |
| ---------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------- | ---------------------------------------------- |
| Body data                    | Part bundles attached by rule, plus tissue-layer plans with arguments            | One explicit part tree per body, reusable part definitions                                 | Flat part list per anatomy, reusable part objects | Types, variants, and a nested tree per anatomy |
| Depth inside a part          | Ordered tissue layers; internal parts                                            | Inside or outside flag; bones and organs are parts                                         | None; sub-parts exist for armor only              | None                                           |
| What a hit records           | Damage state per tissue layer                                                    | Injury record with severity; part hit points drop                                          | Limb hit points drop; a wound may be added        | Hit points drop; rarely a limb is severed      |
| Bleeding                     | From tissue vascularity and arteries into blood volume                           | Summed injury bleed rates into one blood-loss condition                                    | Bleed effect drains a blood quantity              | Hit-point damage per turn                      |
| Pain                         | Tissue pain receptors × severity tier; 150 knocks out                            | Severity × type factor; shock at 80%                                                       | Scalar, added by hits and wounds                  | None                                           |
| Nerves                       | Spine sections disable everything below; motor and sensory nerve damage in limbs | Spine is a part that feeds Moving                                                          | None                                              | None                                           |
| Death                        | Brain, suffocation, blood at zero, loss of head or trunk                         | Vital part, vital capacity at 0%, blood loss 100%, 150 total damage, or a roll when downed | Vital limb at 0 hit points, or blood exhausted    | Hit points at 0, or decapitation               |
| Hidden pool                  | None (blood, pain, per-layer fractions)                                          | Per-part hit points and a whole-body total                                                 | Per-part hit points; one number for monsters      | The whole model                                |
| Same model for all creatures | Yes                                                                              | Yes                                                                                        | No                                                | Yes                                            |
| Named weakness               | Unreadable, slow to build, opaque rules, some absurd outcomes                    | Pool and pacing rules override anatomy                                                     | Player and monsters differ; limb loss unsupported | Anatomy is slots, not injury                   |

## Design options

### A composable body plan

**Option A1: rule-attached bundles (Dwarf Fortress).** A creature lists bundles; each part in a bundle attaches by rule to whatever matching part exists ("every part in category X", "the upper body"). Tissue layers come from a separate plan keyed by category.

- For: the most reuse. A new creature is two lines.
- Against: the body is not visible in any one file, a typo in a category silently attaches nothing, and layer overrides become edit scripts (`SELECT_TISSUE_LAYER`, `PLUS_TISSUE_LAYER`). That generality is built for hundreds of creatures.

**Option A2: one explicit tree per body (RimWorld, Caves of Qud).** Each body plan is a nested tree that names every part instance, referring to shared part definitions.

- For: the file is the body. Easy to read, validate, and diff.
- Against: sub-structures are copied between bodies. The mirror's `Bodies_Animal_Quadruped.xml` holds ten bodies (with paws, with paws and tail, with hooves, with hooves and hump, and so on), and each restates the same organs **[V]** (https://github.com/RimWorld-zh/RimWorld-Core/blob/master/Core/Defs/Bodies/Bodies_Animal_Quadruped.xml).

**Option A3: explicit tree with reusable sub-assemblies and a layer plan (hybrid, recommended).** Three kinds of data:

1. **Part definitions**: name, function tags, relative size, whether internal.
2. **Assemblies**: named sub-trees that can be mounted at a stated attachment point and repeated with a side ("arm with hand and five fingers", "pair of lungs", "head with brain, eyes, mouth").
3. **Layer plans**: tissue layers by part category, with the tissues as arguments, as in `VERTEBRATE_TISSUE_LAYERS:SKIN:FAT:MUSCLE:BONE`.

A body plan is then a short explicit tree that mounts assemblies and names one layer plan. Attachment is always explicit, so there is no rule matching to debug. Qud's variants cover naming: the strange creature's grasping part can be a variant of the hand type and inherit every rule written for hands.

An illustration only, with no file format implied:

```
body_plan wolf
  layers: vertebrate(hide, fat, muscle, bone)
  trunk
    mount neck_and_head(teeth: fangs)
    mount chest_organs          # heart, two lungs
    mount gut_organs
    mount spine
    mount leg(side: left,  position: fore)
    mount leg(side: right, position: fore)
    mount leg(side: left,  position: hind)
    mount leg(side: right, position: hind)
    mount tail
```

### A no-hit-point injury model

**Option B1: layered tissue with material physics (Dwarf Fortress).** Momentum, contact area, and material yield and fracture values decide which layers are cut, dented, or bruised.

- For: armor and weapon materials fall out of the same rules.
- Against: the developer spent most of a year on it, players had to reverse-engineer it, and its results are hard to predict or tune.

**Option B2: part hit points with injury records and capacities (RimWorld).**

- For: simple, proven, and it gives a very readable panel.
- Against: it is a hit-point model. Part hit points are small pools, and the game needed a whole-body total on top. It conflicts with this project's central rule.

**Option B3: limb hit points with wound overlays (Cataclysm).** Same objection as B2, and the wounds do not determine outcomes; the hit points do.

**Option B4: layered tissue with discrete states and a small resistance table (recommended).** Keep Dwarf Fortress's structure and drop its physics.

- An attack has a mode (cut, stab, blunt) and a penetration budget.
- The struck part is chosen by relative size, in the manner of RimWorld's coverage. Layers are then walked from the outside in. Each tissue has a resistance per mode, and the budget is spent layer by layer. If the strike passes the last layer of a part that contains internal parts, one of them is chosen by relative size and the walk continues.
- Each layer ends in one of a few discrete states, for example intact, bruised, cut or cracked, and severed or shattered. There is no number that fills up.
- Tissue tags then produce the consequences. Vascularity and artery flags set a bleed rate. Pain receptors set pain. Structural, connecting, and functional flags decide whether the part still holds together and still does its job.
- Part tags produce the creature-level consequences: a creature with no working grasping part drops its weapon, one with no working stance part falls, one with no working breathing part suffocates over a few turns, and one with no working thought part dies.
- Nerves follow Dwarf Fortress: severing nervous tissue in a spine section disables every part further from the thought centre, and a deep limb wound may cut a motor nerve (the limb stops working) or a sensory nerve (it stops hurting).
- Blood volume and pain are the only running totals. Both name a physical quantity that the prose can describe.

## Recommendation for the alpha

1. **Body plans: Option A3.** Shared part definitions, shared assemblies, and one parameterised layer plan per body type, with an explicit tree per body plan. Expect the human and wolf to share their organ, head, and spine assemblies, as the dwarf and wolf do in Dwarf Fortress.
2. **Injury: Option B4.** Per-layer discrete wound states, a resistance table of tissue against attack mode, and consequences from tags.
3. **Allow exactly two creature-level scalars: blood volume and pain.** Any other number proposed for a creature or part should be challenged as a hit-point pool. Do not copy RimWorld's part hit points, its 150-point total, or its death-on-downed roll.
4. **Death is always a named failure**: thought centre destroyed, breathing stopped, or blood exhausted. Add a load-time check that every body plan has at least one reachable way to die, so the strange creature cannot be Dwarf Fortress's unkillable sponge.
5. **Every authored part must do something.** If a part has no function tag that the simulation reads, either give it one or leave it out. For the human this suggests brain, eyes, heart, lungs, guts, and spine as internal parts, and no liver or pancreas until something uses them.
6. **Keep the part count modest.** About thirty parts for the human sits between Cataclysm's twelve and RimWorld's forty kinds. Treat fingers and toes as one grouped part per hand or foot unless a later ticket needs them singly.
7. **Design the panel and the prose before the detail.** Use Dwarf Fortress's six part states (none, minor, inhibited, function loss, broken, missing) as the status vocabulary and RimWorld's two lists: a short list of functions and a list of parts with their wounds. The narrator should report the deepest layer reached and its consequence, not every layer passed.
8. **Use one model for every creature**, as already fixed. Cataclysm shows the cost of the alternative: limb loss cannot be added there because monsters and many actions never read part state. Make every action read part function from the start.
9. **Take Qud's variants for strange anatomy**, so a tentacle is a hand to the rules and a tentacle to the prose.
10. **Give generated history its own cheap wound picker.** Dwarf Fortress does not run its combat model in world generation; it picks a plausible wound from a list. The same split lets a generated opponent arrive with an old injury without simulating the fight that caused it.

Two findings bear on tickets that are still open on the map. One broken bone knocks a Dwarf Fortress creature unconscious, and a severed limb is close to fatal in Cataclysm's developers' judgement, so pain and blood thresholds will decide how long a duel lasts (lethality and tuning). Nervous tissue has no healing rate in Dwarf Fortress's raws, which gives a ready rule for what never heals (recovery and lasting injury).

## Not verified

- **[R]** Tarn Adams's chapter "Simulation Principles from Dwarf Fortress" in Game AI Pro 2 gives four principles: do not overplan your model, break down and understand the system, do not overcomplicate, and base your model on real-world analogs. I downloaded the PDF (http://www.gameaipro.com/GameAIPro2/GameAIPro2_Chapter41_Simulation_Principles_from_Dwarf_Fortress.pdf) but could not extract its text, so this wording is from memory.
- **[R]** The version number (0.31) and year (2010) of the Dwarf Fortress release that introduced the material, tissue, and wound rewrite. DF Talk 4 confirms the rewrite and its cost, not the version.
- **[R]** The names of the RimWorld C# classes that perform the death checks. The wiki confirms the rules; I read no game code.
- **[R]** Brian Bucklew's talk on Caves of Qud's data-driven engine as a source for the body model's design reasoning. I found no developer statement on that model's weaknesses.
- The RimWorld XML comes from an unofficial mirror last updated in September 2018. Structure and the quoted values match the current wiki, but any other value may have changed since.
- Dwarf Fortress's penetration formulas are the community's reverse-engineering as recorded on the wiki, dated there to version 0.40.05, with one paragraph flagged by the wiki as unverified. The motor and sensory nerve rules are likewise the wiki's description of observed behaviour.
- I read Caves of Qud's `Bodies.xml` only through the snippets quoted on its wiki, and the wiki page is a stub.
- I did not find the Cataclysm function that kills a character when a vital part reaches zero; the claim rests on the comment in `bodypart.h`.
- I did not play any of the four games for this research. Claims about how output reads to a player rest on the developer quotes and wiki text cited above.
