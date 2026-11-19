# Fights alternate turns with a chosen defence, and track no speed or distance

A one-on-one fight needs a clock, a sense of position, and a way to defend. We decided that turns pass strictly back and forth with one action each, that the only position a fight tracks is whether each creature is standing or on the ground, and that the defender is asked to choose a defence (dodge, parry, or block) against every strike that comes at it.

## Considered options

- **Action costs on a timeline**, as Dwarf Fortress does, so that fast actions come round sooner. Not chosen because it needs a speed for every creature and a cost for every action, all of which would have to be tuned before a first fight could be played, and a speed number on a creature would have to be argued against ADR 0002.
- **Both creatures choose in secret and resolve together.** Not chosen because the defender cannot answer a strike it has not seen, and because two interleaved outcomes are harder to narrate in order.
- **Distance as well as posture** (apart or in reach, with actions to close and back off). Not chosen because it adds two actions and a reach rule for every weapon and natural weapon, which is too much for the alpha.
- **Automatic defence with a separate Guard action**, where the body uses its best defence unasked. This was the recommendation during the design session. The developer chose a prompt on every incoming strike instead, so that defending is always a decision.

## Consequences

- No creature is faster than another. A wolf's quickness can show only through its odds, never through extra turns.
- Reach does not exist. A spear has no advantage over teeth for being long, and a creature on the ground can strike any part of a standing opponent, at worse odds. The duel prototype is where this gets tested for absurd outcomes.
- The player makes a decision on the opponent's turn as well as on their own, so a fight has about twice as many prompts as it has player actions.
- Every creature that is not the player needs a rule for choosing its defence, not only its action.
- Blocking with a bare part is a deliberate trade: the blow lands on the blocking limb instead of the part that was aimed at.
- The first turn has to be awarded by something outside the fight (who surprised whom, or a seeded coin flip), because nothing inside the fight ranks the two creatures.
