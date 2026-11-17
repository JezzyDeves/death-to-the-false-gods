# Research: simulation-driven prose generation

Resolves issue #5. Researched 2026-11-17. Scope is the alpha fixed by the map (issue #1): templates and grammars filled in by the simulation, no LLM at runtime, same seed gives the same text, two uses (narrating turn-based one-on-one anatomical combat; presenting a short generated history), one developer.

## Answer in brief

Hand-roll one small narration module in Rust rather than adopt a grammar crate. The simulation emits typed events; the module picks a template whose conditions match the event's facts, fills its slots through a handful of English helpers, and draws every random choice from a seeded text-only generator. The only dependency worth taking is `rand` with a named portable generator.

- **Combat** leans on conditions and facts: one sentence per strike that names the deepest tissue reached and the functional consequence, with verbs keyed by damage type, tissue and severity. The complete layer-by-layer list belongs in the body-status panel, not the prose.
- **History** leans on nested fragment pools: one sentence per typed event, with fragments chosen by the properties of the factions and figures involved, as Caves of Qud does.

Both uses share the same engine and differ only in authored content.

## How to read the claims

Each claim carries one of these marks.

- **[V]** Verified: read at the cited URL in this session, as page text or source code. Some pages were read by delegated reading agents. Afterwards I fetched the cited pages again and checked the quoted wording against the raw text with a script; every quote checked matched, allowing for stripped emphasis markup and PDF line breaks. The two exceptions I did not re-check are the Parrigues PDF and the SimpleNLG pages, whose quotes rest on the reading agent's extraction.
- **[M]** Recalled from memory, not verified in this session.
- **[R]** My recommendation or inference, not a sourced fact.

## Techniques compared

| Technique                                    | What it is                                                                      | Agreement                             | Pronouns                                                          | Articles                              | Avoiding repetition                               | Seeding                                        |
| -------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------- | ---------------------------------------------- |
| Slot templates                               | A sentence with named holes                                                     | None; the author writes each form     | Passed in as slots                                                | Passed in or via a helper             | None built in                                     | Not applicable; no randomness                  |
| Tracery (replacement grammar)                | `#symbol#` expands to one of a list of rules, recursively                       | None; `.s` and `.ed` are suffix rules | Author pushes a matched set onto rule stacks                      | `.a` checks the first letter          | A hidden shuffle-deck mode                        | `tracery.setRng` on the `tracery2` branch only |
| Rules with conditions and weights (RimWorld) | Grammar rules guarded by tests on simulation facts, with weights and priorities | Verb forms authored per manoeuvre     | `_possessive`, `_objective` subsymbols                            | `_definite`, `_indefinite` subsymbols | Weighted alternates                               | Not documented in what was read                |
| World-model selection (Improv, Parrigues)    | Fragments carry tags; the most specific fragment matching the world state wins  | Not addressed                         | Not addressed                                                     | Naive `a`/`an` helper                 | History filter drops used phrases; rare-word caps | Improv takes an `rng` option                   |
| Ink variation primitives                     | Sequences, cycles, once-only, shuffles, conditional text                        | Not addressed                         | Not addressed                                                     | Not addressed                         | Built into the primitives                         | `SEED_RANDOM`                                  |
| NLG pipeline (SimpleNLG)                     | Choose content, plan, aggregate, pick referring expressions, realise grammar    | Realiser inflects from features       | Classed as referring-expression generation; not done by SimpleNLG | Orthographic helper with exceptions   | Not addressed                                     | Not addressed                                  |

### Tracery

- Syntax: `#symbol#` expands by picking one rule; modifiers "are applied, in order, after a tag is fully expanded". `[key:#rule#]` pushes an expansion onto a key's stack for reuse, and `[key:POP]` pops it. [V] https://raw.githubusercontent.com/galaxykate/tracery/tracery2/readme.md
- Selection is uniform by default: `Math.floor(Math.pow(rng(), this.falloff) * this.defaultRules.length)` with `falloff = 1`. A `"shuffle"` distribution exists (a Fisher-Yates deck of indices, popped until empty, then reshuffled) but nothing sets it by default and the README does not document it. `"weighted"` and `"falloff"` only report "not yet implemented". [V] https://raw.githubusercontent.com/galaxykate/tracery/tracery2/js/tracery/tracery.js
- Seeding: `tracery.setRng(myRng)` exists on the `tracery2` branch. The master branch calls `Math.random()` directly in `ruleset.js`. [V] same source, plus https://raw.githubusercontent.com/galaxykate/tracery/master/js/tracery/rule/ruleset.js
- Agreement: there is no number or person mechanism. `.s` adds "es" after s/h/x, turns consonant+y into "ies", otherwise adds "s". `.ed` has matching suffix rules. There are no irregular forms and no consonant doubling. [V] https://raw.githubusercontent.com/galaxykate/tracery/tracery2/js/tracery/mods-eng-basic.js
- Articles: `.a` returns "an" before a vowel letter, with a carve-out for `u?i` words such as "unicorn". It is spelling-based, so "hour" comes out wrong. [V] same source
- Pronouns: the tutorial's own pattern is to push a matched set, `"[heroThey:she][heroThem:her][heroTheir:her][heroTheirs:hers]"`, then write `#heroThey#`. [V] https://tracery.io/archival/crystalcodepalace/js/tutorials/tutorialSrc.js
- Limits in the authors' words: "if-statements, and modifiers with parameters are still in flux". Kate Compton: "The downside of grammars is that they do not have a way to handle constraints, unless the constraints are implicitly encoded". [V] README above and https://galaxykate0.tumblr.com/api/read/json?id=139774965871

What this means for the alpha [R]: Tracery is good at flavour pools and bad at saying a specific true thing. Combat narration must state facts the simulation decided (this tissue, this consequence), which is exactly the conditional, world-state-driven selection Tracery lacks.

### Rules with conditions and weights (RimWorld)

RimWorld's combat log is a grammar whose rules can test simulation facts. The English source rules, read from the comments of an official translation repository: [V] https://raw.githubusercontent.com/Ludeon/RimWorld-fr/master/Core/DefInjected/RulePackDef/RulePacks_CombatIncludes.xml

```
r_logentry->[action], [result].
action->[INITIATOR_definite] [tried] to [damaged_inf] [RECIPIENT_definite] with [implement]
result(p=0.2)->[damaged_present] [RECIPIENT_possessive] [targetlist] and [flavortext_consequence]
targetlist(recipient_part_count==2)->[recipient_part0_label] and [recipient_part1_label]
flavortext_consequence(RECIPIENT_flesh!=Mechanoid)->leaving a disgusting wound
```

- `p=` is a weight, the parenthesised test is a constraint on a simulation fact, and a rule can carry a fallback priority. [V] same source
- Each manoeuvre supplies its own verb forms (`damaged_inf->slash`, `damaged_past->slashed`, `damaged_present->slashing`), so no conjugator is needed. [V] `RulePacks_Maneuvers.xml` in the same directory
- Article subsymbols "will try to obey normal rules of grammar, such as not printing articles in front of proper names in English". [V] https://rimworldwiki.com/wiki/Modding_Tutorials/GrammarResolver

This is the closest prior art to what the alpha needs: a replacement grammar plus guards on facts plus weights. [R]

### World-model selection (Improv, Annals of the Parrigues)

- Bruno Dias's Improv: "Unlike Tracery, however, Improv generators refer to models to build text." Phrase groups carry tags; filters score each group against the model, drop contradictions, and add salience for matches; by default only the top-scoring phrases are eligible. Used phrases merge their tags back into the model so later text can refer to them. [V] https://raw.githubusercontent.com/sequitur/improv/master/README.md and `docs/filters.rst` under the same root
- Emily Short's Parrigues method compiles "a list of possible substitutions that match the present world model", then can "optionally apply salience criteria to pick the option that reflects the greatest number of facts". It also lets the system "recognize, for instance, when a new concept or entity has first been mentioned to the reader". [V] end matter of https://gwern.net/doc/ai/fiction/2015-short-theannalsoftheparrigues.pdf
- Improv's article helper is `[an :animal]`, implemented as a first-letter test that its own docs call "fairly naive". [V] Improv `docs/templating.rst`

The idea to take [R]: when several templates match, prefer the one that uses the most facts. A template that needs "bone fractured and the limb is a grasping limb" should beat a generic "the blow lands" whenever both apply.

### Ink variation primitives

Ink names the standard repetition-control devices: sequences (advance, then stick on the last), cycles (loop), once-only (then nothing), shuffles (randomised; the multiline form will "shuffle the content, play through it, then reshuffle"), and conditional text `{cond: a|b}`. `~ SEED_RANDOM(235)` fixes the seed. [V] https://raw.githubusercontent.com/inkle/ink/master/Documentation/WritingWithInk.md

### The NLG pipeline

The textbook pipeline is: a document planner (content determination, document structuring), a microplanner (lexicalisation, referring expressions, aggregation), and a surface realiser. SimpleNLG implements only the last stage, where the author sets features and the realiser inflects verbs to agree. Its a/an helper is a vowel-letter test with a short exception list. [V] https://raw.githubusercontent.com/wiki/simplenlg/simplenlg/ (Appendix A) and https://aclanthology.org/W09-0613.pdf

The stage names are a useful checklist even for a small system [R]: decide what is worth saying, merge related facts into one sentence, decide how to refer to each thing, then worry about grammar.

### Avoiding repetition: what practitioners say

- Compton on shuffling: "'deck shuffling' (putting all the options in a stack and discarding when they are used), which prevents accidentally picking the same option twice." [V] https://galaxykate0.tumblr.com/api/read/json?id=139774965871
- Compton on the oatmeal problem: 10,000 bowls of oatmeal are each unique, "But the user will likely just see a lot of oatmeal. Perceptual uniqueness is the real metric". [V] same source
- Short's test for oatmeal is "whether the generation is connected to anything mechanical". [V] https://emshort.blog/2016/09/21/bowls-of-oatmeal-and-text-generation/
- Short's "venom" rule: vary "the most statistically implausible, meaning-bearing words", capped at "maximally two" such elements per sentence. [V] Parrigues end matter
- Improv's `dryness()` filter "Removes phrases that are already present in the generator's history", with a warning not to run out of valid phrases. [V] Improv `docs/filters.rst`

The lesson for a combat sim [R]: synonyms alone read as oatmeal. Text should differ because the simulation result differs, and a deck plus a short recency memory handles the rest.

## How prior games narrate a wound

### Dwarf Fortress

The DF wiki refused every request during this session (HTTP 429), so the findings below come from three other sources: bug reports that paste real logs, a dump of the strings in the game executable, and the vanilla raw files.

Example lines from pasted logs:

- "The Axedwarf hacks The Gorlak in the head with her \*steel battle axe\*, tearing apart the muscle!" followed by "An artery has been opened by the strike!" and "The \*steel battle axe\* has lodged firmly in the wound!" [V] https://dwarffortressbugtracker.com/view.php?id=2524
- "The force pulls the right ankle, tearing apart the muscle and bruising the bone and tearing apart the muscle and bruising the bone!" followed by "A ligament has been torn and a tendon has been torn!" [V] https://dwarffortressbugtracker.com/view.php?id=10095

How it is built:

- The sentence is concatenated from fragments. The executable holds them as separate strings: connectors (`in the`, `with`), severity participles (` bruising`, ` denting`, ` chipping`, ` fracturing`, ` shattering`, ` tearing`, ` tearing apart`), outcome tails (`, but the attack is deflected by `, ` and the severed part sails off in an arc!`, ` is cloven asunder!`), and consequence subjects (`an artery`, `a tendon`, `a ligament`, `a motor nerve`, `a sensory nerve`). [V] https://raw.githubusercontent.com/dfint/stringdumps/master/stringdump_steam_53_15.txt
- The data files supply the nouns and verbs. An attack carries both verb persons: `[ATTACK_VERB:punch:punches]`. A weapon attack is documented in the raws as "ATTACK:EDGE/BLUNT:contact area:penetration size:verb2nd:verb3rd:noun:velocity multiplier", for example `[ATTACK:EDGE:40000:6000:hack:hacks:NO_SUB:1250]`. Body parts and tissues carry their names: `[BP:RH:right hand:STP]`, `[TISSUE_NAME:muscle:muscles]`. [V] https://github.com/BenLubar/raws (branch `archive`, `objects/item_weapon.txt`, `creature_standard.txt`, `body_default.txt`, `tissue_template_default.txt`)
- Second person is handled by duplicated fragments, not a conjugator: ` gives` and ` give`, `has` and `have`, and whole lines such as "You are stunned." [V] string dump above

So the shape of a DF wound is: a strike clause (attacker, verb, target, body part, weapon), then one severity participle and tissue name per layer from the outside in, then separate sentences for each consequence (artery, tendon, nerve, pain, falling). [V]

Why it reads as a log [R, from the quoted lines]: one sentence per mechanical event, a fixed clause order, every layer listed, an exclamation mark on everything, full noun phrases repeated instead of pronouns, and no variation between two identical outcomes. The duplicated clause in the second example shows the cost of pure concatenation with no aggregation step.

What is worth copying [R]: the vocabulary lives in creature and item data, so a new creature brings its own words; and severity is expressed by the choice of verb per tissue rather than by a number.

### RimWorld

RimWorld reports the same kind of event far more loosely: an action clause, then a result naming the damaged parts as a joined list, with an optional consequence one time in five ("leaving a horrific wound", "throwing [RECIPIENT_objective] off balance"). Verb pools are weighted and keyed by manoeuvre. It reads better than DF because alternates are weighted and facts gate the flavour, but it says less about tissue. [V] rule packs cited above

### Caves of Qud

What was reachable covers Qud's grammar helpers rather than its wound text. The modding wiki documents pronoun replacers of the form `=pronouns.(pronounterm)=` (terms include `subjective`, `objective`, `possessive`, `reflexive` and `personTerm`, and a capitalised key such as `=pronouns.PersonTerm=` gives capitalised output) and a verb replacer `=verb:(word)[:afterpronoun]=` that conjugates from the object's pronoun set. [V] https://wiki.cavesofqud.com/wiki/Modding:Grammar

The point to take [R]: each entity stores its own pronoun set, and verbs conjugate from whoever is the subject, so "you strike" and "the bear strikes" come from one template.

## How prior games write history

### Caves of Qud

From Grinblat and Bucklew, "Subverting Historical Cause & Effect: Generation of Mythic Biographies in Caves of Qud": [V] https://www.pcgworkshop.com/archive/grinblat2017subverting.pdf

- "Our system first decides that a particular historical event will occur, then it rationalizes a cause based on the states of the historical entities at play." Events "are chosen at random, though their gospels often profess causes."
- Entities are "places, items, and sultans represented as data structures". A sultan starts with "name, pronouns, birth year, birth region, location in birth region, and domain", then passes through about a dozen events.
- Each event produces one sentence-sized "gospel" via "a Tracery-like replacement grammar whose rules map sultan properties to text fragments".
- Fragments live "in a static, nested JSON file" and are looked up by property, for example `<domains.sultan$domains[random].practices.!random >`, which for the ice domain yields "encasing things in ice". Queries can nest inside rules.
- If no existing state justifies an event, the event may create one: "the effect causes the cause."
- Sample output: "In 230, Antixerpur assassinated the sultan of Qud over an ordinance prohibiting the practice of encasing things in ice."
- Coherence comes from shared state: "The sultan's shared state acts as a glue that holds the disjointed events together".

### Dwarf Fortress legends

- History is stored as typed event records. A real export block has `<historical_event>` with `<id>`, `<year>`, `<seconds72>`, `<type>peace rejected</type>`, `<site_id>` and participant ids. [V] https://dwarffortressbugtracker.com/view.php?id=13512
- The executable contains 165 distinct event type strings (`hf died`, `hf wounded`, `attacked site`, `artifact created` and so on) and prose fragments consistent with one sentence assembled per event type: `In the `, `early `, `late `, `attacked`, `, struck down by `, ` was the victor.`. [V] string dump above
- A typical rendered sentence has the form "In the early spring of 103, X attacked Y of Z at W." [M]

The two games agree on the structure [R]: history is a list of typed records, and prose is one sentence per record. Qud's sentences read better because fragments are chosen by the properties of the entities involved.

## Rust crates today, versus hand-rolled

Versions and dates are from the crates.io API and repository metadata from the GitHub API, both queried 2026-11-17. [V] `https://crates.io/api/v1/crates/<name>` and `https://api.github.com/repos/<owner>/<repo>`

| Crate                                     | Latest                | Published               | What it gives                                                       | Fit                                                    |
| ----------------------------------------- | --------------------- | ----------------------- | ------------------------------------------------------------------- | ------------------------------------------------------ |
| `tracery`                                 | 0.2.1                 | 2021-04-24              | Tracery port; RNG passed in as `rand::Rng`                          | Poor: stale, depends on `rand ^0.8`, no conditions     |
| `rant`                                    | 4.0.0-alpha.33        | 2022-05-20              | Procedural templating language with a seedable RNG                  | Poor: alpha, no release since 2022                     |
| `minijinja`                               | 3.1.0                 | 2026-11-07              | Jinja2-style engine; custom filters; conditionals, loops, macros    | Works for slot filling; selection logic is still yours |
| `tera`                                    | 2.4.0                 | 2026-09-11              | Jinja2/Django-style engine                                          | Same as above                                          |
| `upon`                                    | 0.11.0                | 2026-07-25              | Small template engine                                               | Slot filling only                                      |
| `fluent` / `fluent-bundle`                | 0.17.0 / 0.16.0       | 2025-05                 | Localisation messages with plural and variant selectors             | Overkill for English only                              |
| `bladeink`                                | 2.1.0                 | 2026-10-17              | Rust port of the Ink runtime                                        | Wrong shape: authored branching, not event-driven      |
| `indefinite` / `in_definite`              | 0.1.12 / 1.1.2        | 2026-11-15 / 2025-09-06 | a/an selection                                                      | Optional; a small exception table does the same        |
| `pluralizer`                              | 0.6.0                 | 2026-10-19              | Pluralise and singularise                                           | Optional; authored plurals are safer                   |
| `rand`                                    | 0.10.4                | 2026-11-15              | RNG traits, `ChaCha8Rng`, `Xoshiro256PlusPlus`, `choose`, `shuffle` | Take it                                                |
| `rand_chacha`, `rand_pcg`, `rand_xoshiro` | 0.10.0, 0.10.2, 0.8.1 | 2026                    | Named portable generators                                           | Only if not using the ones in `rand::rngs`             |
| `rand_seeder`                             | 0.5.0                 | 2026-02-02              | SipHash-based seeding from any hashable value                       | Optional; useful for deriving named streams            |

Notes on the two grammar crates:

- `tracery` 0.2.1: the repository's last push was 2021-04-24. `Grammar::flatten<R: ?Sized + Rng>(&self, rng: &mut R)` and `execute` take the generator explicitly, so it can be seeded. Its dependencies include `rand ^0.8` (two major versions behind 0.10), `pest`, and `Inflector ^0.11` (itself last published 2019-01-20). The `a` modifier is a bare first-letter vowel test and `s` delegates to Inflector. The modifier registry is a private field and the docs describe no way to add modifiers. [V] https://docs.rs/tracery/0.2.1/tracery/, `src/grammar.rs` and `src/modifiers.rs` at https://github.com/caranatar/tracery-rs, and `https://crates.io/api/v1/crates/tracery/0.2.1/dependencies`
- `rant`: the README says "This project is in alpha" and "do not use in production environments until a stable version is released". It does state that "The internal RNG can be manually seeded to produce repeatable outputs." [V] https://github.com/rant-lang/rant

Notes on the general template engines:

- `minijinja` registers filters with `env.add_filter` and loads templates from strings with `env.add_template`. Its docs page mentions no randomness. [V] https://docs.rs/minijinja/3.1.0/minijinja/
- `tera` 2.x adds templates with `add_raw_template` and has `Filter`, `Function` and `Test` traits. [V] https://docs.rs/tera/2.4.0/tera/
- `fluent-bundle` wraps every inserted value in Unicode isolation marks by default; the documented output for a name is `"Welcome, \u{2068}John\u{2069}."`. [V] https://docs.rs/fluent-bundle/0.16.0/fluent_bundle/

### The comparison

No maintained Rust crate provides the part that matters here: choosing among templates by simulation facts, with weights, specificity, and no-repeat memory, under a seeded generator. The two crates aimed at procedural text are stale or alpha. The maintained template engines solve slot filling and conditionals, which is the easy part, and bring a second language (Jinja) into the data files. [R]

A hand-rolled system needs four small pieces [R]:

1. A template format: literal text plus `{slot}` references, loaded from data files alongside creature data.
2. A selector: filter templates by conditions on the event, prefer the most specific, break ties with a shuffle deck.
3. English helpers: a/an with an exception list, possessive, list joining ("A, B and C"), sentence capitalisation, and verb forms looked up from data.
4. A narration context: who has been mentioned, what was said last, and the deck states.

My estimate is a few hundred lines of Rust and no dependency beyond `rand`. That estimate is unverified; the prototype ticket should test it. [R]

## Seeded randomness and reproducibility

What the `rand` project guarantees:

- `StdRng` is "deterministic but not portable: the algorithms may be changed in any release and may be platform-dependent." `SmallRng` is likewise non-portable. [V] https://docs.rs/rand/0.10.4/rand/rngs/index.html
- The named generators (`ChaCha8Rng`, `ChaCha12Rng`, `ChaCha20Rng`, `Xoshiro128PlusPlus`, `Xoshiro256PlusPlus`) are "similar to the standard generators, but with the additional guarantees of reproducibility." The ChaCha types sit behind the `chacha` feature. [V] same page
- Portable items should be "reproducible across platforms" and "across patch releases". Value-breaking changes are permitted in minor versions for portable items, where they "must be well motivated and should be clearly mentioned in the CHANGELOG." [V] https://rust-random.github.io/book/crate-reprod.html
- "if portability is required, _never_ sample a `usize` or `isize` value directly." Index sampling in `rand::seq` uses `u32` "where possible" for this reason. [V] same page and https://docs.rs/rand/0.10.4/rand/seq/index.html
- Floating-point results "depend on rounding modes and implementation details", so float-based distributions "may not be portable." [V] reproducibility page above
- `SeedableRng::seed_from_u64` is the convenience constructor; "Changing the implementation of this function should be considered a value-breaking change." [V] https://docs.rs/rand_core/0.10.1/rand_core/trait.SeedableRng.html
- In 0.10 the slice method is `IndexedRandom::choose`, with `choose_weighted` and `sample`; `choose_multiple` is deprecated in favour of `sample`. [V] https://docs.rs/rand/0.10.4/rand/seq/trait.IndexedRandom.html
- `rand_seeder` turns any hashable value into a seed: `let mut rng: SipRng = Seeder::from("stripy zebra").into_rng();`. [V] https://docs.rs/rand_seeder/0.5.0/rand_seeder/

One trap outside `rand`: Rust's `HashMap` hashing "is randomly seeded" and its iterators visit entries "in arbitrary order". Iterating a `HashMap` to build a candidate list makes the same seed produce different text between runs. [V] https://doc.rust-lang.org/std/collections/struct.HashMap.html

What follows for this project [R]:

1. Use a named portable generator, never `StdRng`, `SmallRng` or the thread generator.
2. Keep the text generator separate from the simulation generator. Otherwise adding one template variant shifts every later dice roll and changes who wins the fight.
3. Derive the text generator's seed per unit of narration (per fight, or per history) from the world seed and a stable label, so editing combat templates cannot change history text.
4. Store candidates in `Vec` or `BTreeMap`, and sort before choosing.
5. Use integer weights, not floats.
6. Commit `Cargo.lock` and treat a `rand` minor-version upgrade as a possible text-changing event.
7. Add golden tests: a fixed seed and a fixed event list must reproduce a stored transcript. They catch all of the above.
8. The narration context (decks, mention history) is state. Either it resets at a fixed boundary such as the start of a fight, or it must be saved. Save and load is still open on the map; this is an input to it.

## Recommended approach

### Shared engine

A narration module in the Rust backend with one entry point: an event plus a narration context in, a string out. Templates and vocabulary live in data files next to the creature data. [R]

- **Selection.** Each template lists conditions on the event's facts. Keep the matches, prefer the highest specificity (the count of conditions satisfied, as in Improv and Parrigues), and draw from a per-rule shuffle deck among the equals.
- **Slots.** A slot names an entity or fact and a form: subject, object, possessive, a verb in the subject's person. Slots may also name another rule set, which gives Tracery-style nesting.
- **Verbs.** Each verb is authored with its forms (base, third-person singular, participle, past), as DF's `punch:punches` and RimWorld's `damaged_inf`/`damaged_present` do. A closed, authored vocabulary never needs an inflection library.
- **Articles.** Nouns default to the first-letter rule and carry an explicit override where that is wrong. The context tracks first mention: "a wolf" once, then "the wolf".
- **Pronouns.** Every entity stores a pronoun set. The player is always "you", which removes most ambiguity in a duel: pronominalise the opponent only when it was the subject of the previous sentence.
- **Validation.** At load time, check that every slot a template uses exists for its event type, and expand every template once in a test. A typo should fail at startup, not mid-fight.

### Combat narration

1. **The simulation emits one structured strike result**: attacker, attack (with verb forms), target body part, each layer reached with its tissue and damage severity, and the functional consequences (bleeding, a severed nerve, lost grip, falling).
2. **Select what to say.** Lead with the deepest or most consequential layer, not all of them. Outer layers are implied: "opens your forearm to the bone" already says the skin and muscle parted. The body-status panel carries the complete list.
3. **Aggregate.** One sentence for the strike and its worst wound; fold one consequence into the same sentence as a clause, and give a second sentence to anything that changes what the player can do next. DF's one-line-per-event habit is the main thing to avoid.
4. **Choose verbs by damage type, tissue and severity.** This is DF's best idea: `bruising`, `tearing`, `tearing apart` for soft tissue and `chipping`, `fracturing`, `shattering` for bone. Author a small pool per cell so the deck has something to rotate.
5. **Take body-part and tissue names from the body plan data**, so the strange creature narrates with no code change.
6. **Vary structure, not only words.** Alternate between attacker-led, wound-led and consequence-led sentences, and bar the same structure twice running. Cap vivid words at one or two per sentence.
7. **Give misses, blocks and glancing blows their own short templates**, kept plainer than wounds so the wounds stand out.

Illustrative shape only; the real schema belongs to the prototype:

```
when: { damage: cut, deepest_tissue: bone, bone_severity: fractured, part_role: grasp }
text: "{attacker.Subject} {verb.3sg} {target.possessive} {part}, {cut.participle} to the bone. {consequence.sentence}"
```

Example output: "The wolf's jaws close on your left forearm, tearing through to the bone. Your fingers go slack."

### History text

1. **Worldgen produces a list of typed events** with entity ids and a year, like DF's records. The text layer never invents facts.
2. **One sentence per event**, from three to six templates per event type.
3. **Fragments keyed by entity properties**, as in Qud: a faction's creed, a settlement's trade, or a god-claimant's domain selects the motive phrases and epithets. This is where nested rule pools earn their place.
4. **Group sentences into short paragraphs** by era or by faction, with computed connectives ("Two years later", "In answer") where one event references another.
5. **First mention gives the full name and an epithet; later mentions use the short name or a pronoun.**
6. **Accept some visible pattern.** The history is skimmed once per world, so a page of consistent chronicle voice is fine; spend the authoring effort on combat, which is read hundreds of times.

One option to raise in the worldgen ticket, not to decide here: Qud picks events first and invents their causes afterwards, which gives the appearance of causality from a very thin simulation. That fits "worldgen stays thin", but it must still produce the factions the player actually fights. [R]

## Not verified

- **Dwarf Fortress wiki content.** Every page returned HTTP 429. Not confirmed: which damage threshold produces which severity verb, and the meaning of `STP` and `NP` in the raws (I believe "standard plural" and "no plural"). [M]
- **Dwarf Fortress log grouping.** That repeated announcements collapse with a count such as "x3", and how the combat report screen groups lines. [M]
- **A complete legends sentence.** Only fragments were read. [M]
- **Tarn Adams on text generation.** The Game AI Pro 2 chapter was read and says nothing about text; no talk or interview on the subject was read.
- **Caves of Qud combat and dismemberment messages**, and the `=subject.T=`, `=object.t=`, `Grammar.A()`, `Grammar.Pluralize`, `Grammar.MakePossessive` helpers. I recall these from the game's modding API but found no reachable page documenting them. [M]
- **The GDC 2018 talk "Procedurally Generating History in Caves of Qud"** beyond its abstract.
- **The Tracery paper's full text** (Compton, Kybartas and Mateas, ICIDS 2015). Only the abstract was reachable; Tracery's limits above come from its README, source and Compton's essay.
- **Reiter and Dale, "Building Natural Language Generation Systems".** The pipeline description is SimpleNLG's summary of it.
- **Template engines and randomness.** The word "random" appears nowhere on the `minijinja` 3.1.0 or `tera` 2.4.0 crate front pages; I did not search their full API docs, so that is not proof no such function exists. Tera 1.x had a `get_random` function. [M]
- **`tracery` custom modifiers.** The registry is private in the source read, but I did not read every file, so a public hook may exist.
- **The size of a hand-rolled system** is my estimate, not a measurement.

## Sources

Tracery and grammar practice

- https://raw.githubusercontent.com/galaxykate/tracery/tracery2/readme.md
- https://raw.githubusercontent.com/galaxykate/tracery/tracery2/js/tracery/tracery.js
- https://raw.githubusercontent.com/galaxykate/tracery/tracery2/js/tracery/mods-eng-basic.js
- https://raw.githubusercontent.com/galaxykate/tracery/master/js/tracery/rule/ruleset.js
- https://tracery.io/archival/crystalcodepalace/js/tutorials/tutorialSrc.js
- https://galaxykate0.tumblr.com/api/read/json?id=139774965871 (Compton, "So you want to build a generator")
- https://gwern.net/doc/ai/fiction/2015-short-theannalsoftheparrigues.pdf (Short, Annals of the Parrigues, end matter)
- https://emshort.blog/2016/09/21/bowls-of-oatmeal-and-text-generation/
- https://raw.githubusercontent.com/sequitur/improv/master/README.md and `docs/` under the same root
- https://raw.githubusercontent.com/inkle/ink/master/Documentation/WritingWithInk.md
- https://raw.githubusercontent.com/wiki/simplenlg/simplenlg/ and https://aclanthology.org/W09-0613.pdf

Games

- https://raw.githubusercontent.com/dfint/stringdumps/master/stringdump_steam_53_15.txt
- https://github.com/BenLubar/raws (branch `archive`)
- https://dwarffortressbugtracker.com/view.php?id=2524, `?id=10095`, `?id=13512`
- http://www.gameaipro.com/GameAIPro2/GameAIPro2_Chapter41_Simulation_Principles_from_Dwarf_Fortress.pdf
- https://www.pcgworkshop.com/archive/grinblat2017subverting.pdf
- https://wiki.cavesofqud.com/wiki/Modding:Grammar
- https://raw.githubusercontent.com/Ludeon/RimWorld-fr/master/Core/DefInjected/RulePackDef/RulePacks_CombatIncludes.xml
- https://rimworldwiki.com/wiki/Modding_Tutorials/GrammarResolver

Rust

- https://crates.io/api/v1/crates/ (per-crate metadata, queried 2026-11-17)
- https://docs.rs/tracery/0.2.1/tracery/ and https://github.com/caranatar/tracery-rs
- https://github.com/rant-lang/rant
- https://docs.rs/minijinja/3.1.0/minijinja/
- https://docs.rs/tera/2.4.0/tera/
- https://docs.rs/fluent-bundle/0.16.0/fluent_bundle/ and https://projectfluent.org/fluent/guide/selectors.html
- https://docs.rs/rand/0.10.4/rand/rngs/index.html, `/seq/index.html`, `/seq/trait.IndexedRandom.html`
- https://docs.rs/rand_core/0.10.1/rand_core/trait.SeedableRng.html
- https://docs.rs/rand_seeder/0.5.0/rand_seeder/
- https://rust-random.github.io/book/crate-reprod.html
- https://doc.rust-lang.org/std/collections/struct.HashMap.html
