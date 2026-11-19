# Death to the False Gods

A text-heavy low-fantasy RPG built on two pillars: a procedurally generated world presented through prose, and combat simulated on layered anatomy rather than stats or hit points.

## Language

**Body plan**:
The authored definition of a creature's anatomy, composed from reusable parts and written as data rather than code.
_Avoid_: Species template, creature blueprint, anatomy definition

**Body**:
One living instance of a body plan, carrying its own current injuries and impairments.
_Avoid_: Character sheet, health, HP

**Part**:
One named piece of a body, either attached to another part (a hand to a forearm) or inside another part (a heart in a chest).
_Avoid_: Limb, body part, hit location, bone (bone is a tissue)

**Tissue**:
A kind of living material, such as skin, fat, muscle, bone, or nervous tissue, that a part is made of in ordered layers from the outside in.
_Avoid_: Material, layer type

**Layer plan**:
The recipe that gives each kind of part its ordered tissues, with the tissues swappable so that hide can stand in for skin or chitin for bone.
_Avoid_: Tissue recipe, tissue template

**Assembly**:
A reusable sub-tree of parts, such as an arm with its hand or a pair of lungs, that a body plan mounts instead of restating.
_Avoid_: Bundle, sub-body, prefab

**Function**:
Something a part does that the simulation reads, such as thinking, breathing, pumping blood, seeing, grasping, standing, or biting.
_Avoid_: Tag, capacity, ability

**Nerve**:
A part made of nervous tissue that sits inside another part and serves the parts that move or feel, linking them through other nerves back to a thinking part.
_Avoid_: Nervous system, nerve path, wiring
