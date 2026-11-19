# Bones are tissue, and nerves are parts

A body plan has to say where bone and nerves live. We decided that bone is a tissue layer of a part (the "ribs" are the bone layer of the chest) and that a nerve is a part of its own, sitting inside another part, with every part that moves or feels naming the nerve that serves it.

## Considered options

- **Bones as parts** (a humerus, a femur), as RimWorld does. Rejected because it roughly doubles the part count and splits one strike across two models: layers for flesh, a separate part for the bone beneath it. With bone as a layer, a hit is one walk from the outside in.
- **Nerves that follow the attachment tree**, so control flows from the thinking part outward and no nerve is authored. Rejected because it ties what a creature can move to how its parts are bolted together, which forces the tree into a shape chosen for nerve routing, and because a nerve that is not a part cannot be struck, shown in the status panel, or named in prose.
- **Nerves as routes**, a separate list of the parts each nerve passes through. Rejected because it needs its own hit rules and its own display, where a nerve that is a part reuses both.

## Consequences

- There are no named bones and no joints. Prose that wants "the collarbone" has to get it from the part and its bone layer.
- Every creature has to be wired: each part that moves or feels must trace through its nerves to a thinking part, and a body plan that fails this is rejected when loaded. Assemblies carry their own nerves so the wiring is authored once per assembly, not once per creature.
- One nerve carries both movement and feeling, so a limb whose nerve is severed stops working and stops hurting together.
