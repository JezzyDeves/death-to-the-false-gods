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
One named piece of a body, either attached to another part (a hand to a forearm) or inside another part beneath one of its tissue layers (a heart in a chest).
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
Something a part does that the simulation reads, such as thinking, breathing, pumping blood, seeing, grasping, standing, or biting. A creature has a function for as long as enough of its parts with that function still work.
_Avoid_: Tag, capacity, ability, capability

**Nerve**:
A part made of nervous tissue that sits inside another part and serves the parts that move or feel, linking them through other nerves back to a thinking part.
_Avoid_: Nervous system, nerve path, wiring

**Wound**:
The record of one strike on one part, holding the attack type and the state it left each tissue layer it reached: bruised, breached, or destroyed.
_Avoid_: Damage, hit, injury points

**Attack type**:
One of the three ways a strike harms tissue: a cut, a stab, or a blunt blow.
_Avoid_: Damage type, attack mode

**Severed**:
Said of an attached part that a cut has removed from the body for good, along with everything attached beyond it.
_Avoid_: Dismembered, amputated, destroyed (a destroyed part is still attached)

**Blood**:
The volume of blood a body holds, which open wounds drain; too little weakens a creature, then knocks it out, then kills it.
_Avoid_: Health, HP, blood points

**Pain**:
What a body feels at this moment from the wounds on parts whose nerves still reach a thinking part.
_Avoid_: Shock, stun, morale

**Air**:
How many turns a body can last once it cannot breathe or its blood has stopped moving.
_Avoid_: Breath meter, oxygen, stamina

**Way to die**:
One of the four failures that kill a body: loss of thought, blood loss, suffocation, or stopped circulation.
_Avoid_: Death condition, kill threshold, zero HP

**Dying**:
Said of a creature that is alive with a way to die under way that will kill it without help.
_Avoid_: Critical, bleeding out (that is one cause)

**Out of the fight**:
Said of a creature that is unconscious or dead.
_Avoid_: Downed, defeated, incapacitated
