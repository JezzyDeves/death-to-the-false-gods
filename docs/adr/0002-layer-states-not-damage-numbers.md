# Layers have states, not damage numbers, and a creature has three quantities

The game has no hit-point pool, so something else has to hold what a strike did. We decided that each tissue layer of a part is in one of four states (intact, bruised, breached, destroyed), that a wound is the record of one strike and the states it left behind, and that a creature carries exactly three quantities: blood, pain, and air. Death comes only from one of four named failures: loss of thought, blood loss, suffocation, or stopped circulation.

## Considered options

- **A damage fraction per layer**, as Dwarf Fortress keeps (cut, dent, and effect from 0 to 10000). Rejected because a number that fills up on a layer is a small hit-point pool, and because the player cannot read it: "the muscle is breached" can be shown and narrated, "the muscle is at 6200" cannot.
- **Hit points per part with injury records on top**, as RimWorld and Cataclysm do. Rejected because the hit points decide the outcome and the wounds only decorate it, and because RimWorld still needed a whole-body damage total on top.
- **Blood and pain as the only quantities**, as the research recommended. Rejected because suffocating over several turns needs a clock. Air is that clock, and stopped circulation reuses it, so a pierced heart needs no fourth quantity.
- **Pain as a stored total** that hits add to. Rejected because a stored total drifts away from the body it describes. Pain is recomputed from the wounds that are still there, so severing a nerve removes the pain of everything it served at once.
- **The heart mattering only through bleeding**, as in Dwarf Fortress. Rejected because a stab through the heart should kill by itself, and because a creature whose only blood-pumping part is destroyed should not fight on until it happens to bleed out.

## Consequences

- A second strike on an already breached layer cannot make that layer "more breached". It either destroys the layer or leaves it as it is. Repeated strikes still add up through their own wounds, each with its own bleeding and pain.
- Any new number proposed for a creature, a part, or a layer has to name a physical quantity and be argued against this record. Thresholds on blood, pain, and air are tuning values, not new quantities.
- Every creature needs at least one of the four failures to apply to it. A creature with no blood cannot bleed out, and one that does not breathe cannot suffocate, so a body plan with none of the four is rejected when loaded.
- Pain never kills. It can only put a creature out of the fight.
- Fine gradations of injury are lost. A shallow gash and a deep one in the same layer are both "breached", and the difference shows only in how many layers the strike reached.
