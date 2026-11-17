# Generated history and regions in prior games

Research for issue #3, read on 2026-11-17. The question: how do Dwarf Fortress, Caves of Qud and similar games generate a region, its factions and its history, and what is the smallest approach that gives the alpha a believable reason for who the player fights?

## Answer in brief

The prior games sit on a line between two poles.

- **Dwarf Fortress simulates.** It builds terrain from physical fields, places civilizations, then runs what Tarn Adams calls "a giant zero-player strategy game" and keeps the log. The history is true to the world because it is the world's record. It is also the expensive pole: by Adams's own account it is a lot of work, it needs post-processing to find the good moments, and it is boring without enough mechanisms.
- **Caves of Qud rationalizes.** It picks events at random from a small pool and invents each event's cause afterwards from the state of the entities involved. Prose comes from a replacement grammar. Its developers built it this way because they could not afford a simulation, and they describe it as producing convincing histories "on a tight scope budget".
- **Epitaph sits between.** A table of events with prerequisites, where each event shifts the odds of later ones. Its civilization generator and event table together are about 450 lines.

For the alpha, the smallest thing that works is the Qud pole, run backwards from the present: decide first who is hostile to whom and who holds which location, then generate six to ten past events that explain it, then render each event as one or two sentences. The region is a handful of named, tagged locations, not terrain. This is candidate A below, and it is the recommendation.

## How claims are marked

- **[V]** Verified: I read the cited source during this research and the claim is in it. The source is named next to the mark and listed under Sources.
- **[V-wiki]** Verified against a community-maintained wiki that I read. The wikis are the standard references for these games, and the Qud wiki cites game classes by name, but they are not first-party statements.
- **[R]** Recalled from memory. Not checked against a source in this session.
- **[J]** My judgment or estimate. Not a sourced fact.

## Comparison

| Game                   | Order of generation                                                                                        | Simulated or picked                                                         | How history becomes text                                                                    |
| ---------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Dwarf Fortress         | Terrain fields, biomes, rivers, named regions, wildlife, then civilizations placed, then years of history  | Simulated: agents act, the log is the history                               | Each logged event is listed on the pages of the entities it involves (Legends mode)         |
| Caves of Qud, sultans  | Named regions and locations per period, then one ruler, then about a dozen life events, then a death event | Picked at random from a pool; causes invented afterwards from entity state  | One snippet per event from a replacement grammar keyed on the ruler's properties            |
| Caves of Qud, villages | Fixed world map, villages placed per biome, two history events each, then quests from templates            | Picked; filled in from the context of the place                             | Same grammar; shown on monuments, in dialog, in the journal                                 |
| Epitaph                | Species, language, home planet, three traits, shared vocabulary, then events tick by tick                  | Picked by dice, gated by prerequisites, with odds shifted by earlier events | A template per event with variables fixed when the civilization is created                  |
| Ultima Ratio Regum     | Nations with a style, ideologies and religion, then everything else derived from those                     | Attributes assigned at world generation, then expressed in every artifact   | Large hand-written databases of variants selected by the nation's attributes and references |

Sources for each row are in the sections below.

## Dwarf Fortress

**What is generated, in order.**

- **[V]** Terrain comes from basic fields, not from placing biomes. Elevation is made by midpoint displacement, then temperature (biased by elevation and latitude), rainfall (later adjusted for rain shadows) and drainage. Biomes fall out of the fields: "if rainfall is >=66/100 and drainage is less than 50, then you have a swamp". Then erosion by temporary rivers, then real rivers and lakes, then connected biome groups become named regions, then plant and animal populations are set per region. Adams: "This gives you a canvas for world history." (Gamasutra interview, 2008)
- **[V]** The same order, in his words eleven years later, ends: "History itself can begin at this point. Civilizations and caves are placed." (Game Developer Q&A, 2019). The Game AI Pro 2 chapter gives the order as elevation, map layers, biomes, rivers, plant and animal populations, fantasy creatures, then "the simulation of civilization begins. Through this process settlements are raised, trade routes are formed, and wars are waged."
- **[V-wiki]** Even the smallest preset is far larger than the alpha: a "pocket" world is 17 by 17 region tiles with 3 to 10 civilizations and a cap of 4 to 36 new sites. The wiki recommends 50 or 100 years of history for adventure-mode worlds. (DF wiki, World generation)

**What is simulated.**

- **[V]** "the basic idea is that there's a giant zero-player strategy game going on with somewhat loose turn rules and bad AI (but thousands of agents), and history is just a record of that." (Game Developer Q&A, 2019)
- **[V]** The cost, same answer: "Procedurally generating stories by recording a log of a simulation is a valid enough approach, though it has drawbacks, of course. It's a lot of work, you need to do post-processing or investigation to find any good moments you'd like to highlight, and if you don't have enough dynamics and mechanisms, the output can be boring (in any number of ways.)"
- **[V-wiki]** Most of the population is abstract. Only a small share is tracked individually as "historical figures": nobility and their entourage, megabeasts, villains, and anyone who gets a name. (DF wiki, Historical figure)
- **[V-wiki]** Why civilizations fight: each entity has "ethic" stances in its data file, relations are based on comparing them, and "Strongly conflicting ethics often trigger wars during world generation." In war, civilizations "attack each other's settlements, occasionally destroying a site or taking it over." (DF wiki, Ethic and War)

**How history becomes text.**

- **[V-wiki]** Legends mode is a browser over the record. Categories are historical figures, sites, artifacts, written works, regions, civilizations and other entities, structures, and chronicles by age. An entry page has a title, a description, the list of events involving that entry, and stats. Names in the text link to other entries. (DF wiki, Legends)
- **[R]** Each event type has a fixed sentence form with slots for the year, the figures and the site, so the output reads as a terse chronicle. I did not check an example against a source.

**Principles Adams states** that bear on scope **[V]** (Game AI Pro 2, chapter 41):

- "Operate at the level of what the player sees or one layer below."
- "Rather than overplanning the model, build it up through iteration."
- "There's no reason to have 50 variables modeling one aspect of a game's behavior if they don't all have a meaningful impact on the game."

## Caves of Qud: sultan histories

All claims in this section are **[V]** from Grinblat and Bucklew's 2017 paper unless marked otherwise.

**Why they did not simulate.** The history system was subordinate to the game's main systems, so "we couldn't devote the resources to research and develop a generative system whose complexity matched the historical and cultural simulations" of Dwarf Fortress and Ultima Ratio Regum. They wanted "history as an artifact", meaning accounts read in the present, and say Dwarf Fortress must simulate because it "allows players to interrupt its simulated history at any point to start play". The alpha has the same exemption: play starts after the history ends.

**The model.** History is "the interplay between historical entities—places, items, and sultans represented as data structures—and historical events that modify the properties of those entities". The steps for one ruler:

1. Create the ruler with a few properties: name, pronouns, birth year, birth region, location, and one _domain_ (ice, glass, stars, might, scholarship and so on).
2. Pick a life event at random from a pool.
3. The event resolves an outcome, partly from the ruler's state and partly by random branching, then writes changes back to the ruler and any other entities affected.
4. The event generates its text.
5. Repeat about twelve times, then add a death event.

**Cause after effect.** "there's no logic behind the choice of events. Historical cause and effect aren't intrinsic." An event that needs a reason looks at the ruler's state for one. If the ruler has an allied faction of frogs, the siege was "Acting against the persecution of frogs". If no suitable state exists, the event creates it: "the effect causes the cause."

**Why it still reads as a story.** Shared state. "The sultan's shared state acts as a glue that holds the disjointed events together", and because only a few properties are shared across many events, "emergent micro-narratives like this one are quite common". The domain appears in almost every text pattern, so one random choice at creation threads through the whole life.

**How history becomes text.** One snippet per event, from "a Tracery-like replacement grammar whose rules map sultan properties to text fragments". Fragments live in "a static, nested JSON file", grouped by domain under keys such as `practices`. A pattern like "over an ordinance prohibiting the practice of #practice#" is filled by looking up the ruler's domain and picking one fragment at random.

**Size.** At the time of the paper: ten domains and nineteen event types, generating five rulers per world.

**History reaches the world.** Places a ruler visits "are modeled as historical entities and instantiated as historic sites during world generation", and a named item lost in an event is placed at that site with matching properties.

**Tooling.** The authors credit their satisfaction with coherence to "the ability to generate several sultans quickly and read through their gospels".

**Later detail from the wiki** **[V-wiki]** (Qud wiki, Sultan histories and World generation; both cite game classes in `XRL.Annals`):

- A ruler's history is built as: one origin event, eight core events drawn from a pool of 17 types, then **guarantee passes**. If no event has yet made the ruler sultan, one is added. If a historic region of that period has no event set in it, one is added. Then a death event. A history has 10 to 22 events, typically 11 to 14.
- If a random core event kills the ruler early, a "death reversal" event follows at once (a faked death, a twin, a clone).
- Regions and locations are generated per period before the ruler, so events have places to refer to. "Ultimately, most of the regions and locations that are generated as part of the world history will not be accessible in game." They are names.
- The exception is historic sites, which "will be inhabited by sultan cults composed of members from the sultans' favored factions". History decides who lives where.
- Each event has two texts, a "gospel" and a "tomb inscription" that praises the ruler and leaves out shortcomings. The 2017 paper listed conflicting accounts as future work.

## Caves of Qud: villages and quests

- **[V]** The world map is static and hand-made. Only individual areas are procedural. (2017 paper, section 2)
- **[V-wiki]** Each generated village gets a random name, a standard set of roles (mayor, tinker, apothecary, warden, merchant), one quest-giver, and "two generated events which form the 'history' of the village". Each of those events has a 1 in 20 chance of leaving the village abandoned. (Qud wiki, Village)
- **[V]** Village and quest generation "were built on top of a framework we previously developed for generating historical biographies". The parts: a catalog of hand-written quest templates, an interface through which a place answers questions about itself (who can give a quest, what matters to them), a factory that picks a template and fills it from those answers, and a replacement grammar per template for names and dialog. The stated principle: "eschewing causal logic in favor of randomness that's parameterized by the richly narrative world the generators are embedded in." (Grinblat and Bucklew, 2020)
- **[V]** The GDC 2019 talk abstract says villages "include their own histories, cultures, architectural styles, storytelling traditions, NPCs, and quests". I read the abstract only, not the talk.

## Epitaph

All **[V]** from Max Kreminski's 2018 post and the game's source.

- Two kinds of thing happen to a civilization: techs (with prerequisites) and events. "Many techs influence the odds that certain events will happen", and events can shift the odds of later events too. An event can only happen once something has raised its likelihood above zero, which acts as an implicit prerequisite.
- "On each tick, the game just looks at the odds that certain things will happen to each civilization, and rolls the dice".
- Text: a template per event with `$VARIABLES`. "any civilization-specific bit of text that appears in the descriptions of multiple events or techs is set up front as a global variable for that civilization at the moment the civilization is first generated", for consistency across separate snippets. In the source these are `$CITY`, `$CROP`, `$BEAST`, `$RELIGION`, `$CONQUEROR` and a few more.
- Size: the civilization generator is 153 lines of ClojureScript and the event table is 300 lines holding 25 events. (`src/epitaph/civs.cljs`, `src/epitaph/events.cljs`)
- It uses unseeded random calls (`rand`, `rand-nth`), so it is not a model for reproducible worlds.

## Ultima Ratio Regum

- **[V]** Mark Johnson's term "qualitative procedural generation" means "the algorithmic creation of cultures, social norms, practices, beliefs, etc". (Johnson, 2016)
- **[V]** Each nation gets attributes at world generation (a style such as feudal, nomadic or tribal, ideologies, a religion). By the end of world generation each culture holds a database of "references" drawn from its geography, plants, animals, historical events and gods, which its people use in names, greetings, insults and conversation. (Johnson, 2016)
- **[V]** The text comes from "a large database of hand-written variations for civilizations of certain sorts", varied by those references. (Johnson, 2016)
- **[V]** Scale warning: in 2016 the paper calls it a "five-year ongoing" project "a little over half finished". The developer's blog was still posting numbered development updates in November 2026.
- I found no primary description of how the game generates its history as such. See Not verified.

## Tracery

**[V]** (Tracery README) A grammar is a JSON object mapping a symbol to an array of expansions. `#symbol#` in a string is replaced by a random expansion, recursively. A choice can be saved and reused, as in `#[hero:#name#]story#`, so the same name appears throughout one output. Qud's grammar follows this shape with a query step added to read entity state.

## What carries over to the alpha

The alpha needs one region, a few settlements and factions, and a short history whose job is to explain who the player fights. Against that:

| From           | Take                                                                                                                                                               | Leave                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| Dwarf Fortress | The data model: entities plus an event log, with every line of prose rendered from an event. Value conflicts as a stated reason for hostility.                     | Terrain fields, erosion, yearly agent simulation, population tracking, a browser over thousands of events |
| Qud, sultans   | Random events that read and write shared state. Causes invented from state. One domain per leader threaded through the text. Guarantee passes. Batch-reading tool. | Five rulers, nineteen event types, shrines and journal discovery                                          |
| Qud, villages  | Two or three events per place. Occupants of a place decided by history. Places described by answering a fixed set of questions about themselves.                   | Quest generation (out of scope on the map)                                                                |
| Epitaph        | Per-faction vocabulary fixed at creation. Prerequisites on events.                                                                                                 | Tick-based dice rolling, tech tree                                                                        |
| URR            | A faction's attributes should show in its names and in what it says before a fight, not only in the history.                                                       | Breadth. Everything else.                                                                                 |

Three points follow from the sources **[J]**:

1. **No terrain generation.** Adams's rule is to model what the player sees or one layer below. The alpha has no tile map and presents travel as prose, so the layer below is a graph of four to six locations, each with a few tags (terrain, a feature, who holds it). Qud's history refers mostly to places that exist only as names.
2. **Generate the present first.** Qud lets the effect precede the cause, and adds events afterwards to guarantee the outcomes the game needs. The alpha needs exactly one outcome from history: a set of hostilities and holdings that encounters can read. So fix that end state first and generate events that account for it.
3. **The False Gods premise maps onto Qud's domain.** A god-claimant gets one aspect (ash, tide, hunger). It appears in the cult's name, in every event the cult takes part in, and in the reasons given. That is flavor at almost no mechanical cost, which is what the map asks for.

## Candidate approaches

Costs are **[J]**: rough solo-developer estimates for a first working version, including authored text. Treat them as relative sizes.

### A. Rationalized chronicle (Qud pole)

1. From the seed, make the region: a name, a terrain theme, four to six locations in a graph with tags.
2. Make three or four factions, each with a kind (settlement folk, cult, outlaw band), a named leader, an aspect or creed, a home location, and two or three value stances.
3. Decide the present: which factions are hostile to the player's home, who holds each location, and where the wolves and the strange creature are.
4. Draw six to ten events from a pool of about ten types (founding, schism, god-claim, raid, seizure of a site, burning, broken pact, exile, hard winter, the thing that came to the barrow, death of a leader). Each event reads and writes entity state and invents its cause from that state, creating the state if it is missing.
5. Run guarantee passes: every hostility has at least one event behind it, every location and every leader appears at least once.
6. Render each event with a replacement grammar. The history screen is the list in date order. The same events are quoted again at the point of an encounter.

- Cost: about 4 to 6 days of code and 2 to 3 days of writing (ten event types with two or three phrasings each, six aspects with a few fragments per key).
- Strength: every run produces a usable answer to "who do I fight and why". Smallest of the three. Output quality is a writing problem, which suits a prose-first game.
- Weakness: events do not cause each other, so a careful reader will find coincidences where they expected consequences. Variety is bounded by the event pool.

### B. Chained event deck (Epitaph, with Qud's text)

Same entities and same text layer as A. Events carry prerequisites and change the odds of later events: a schism makes a raid likely, a raid makes a reprisal likely, a burning makes a blood feud near certain. Hostility in the present is read off the final state.

- Cost: about 6 to 9 days of code and tuning, plus the same writing as A.
- Strength: real causal chains at low cost, so the history survives closer reading.
- Weakness: the end state is not guaranteed. Some seeds will produce a quiet region with nobody to fight unless a repair pass forces a conflict, at which point it is A with extra tuning. Odds tables need play to balance.

### C. Small faction simulation (Dwarf Fortress pole)

Factions are agents with population, stores, holdings and value stances. Simulate 30 to 50 years on the location graph: they expand, trade, quarrel and raid. Log everything, then select the handful of events worth showing.

- Cost: about 3 to 5 weeks, with open-ended tuning. It still needs the text layer from A and a selection step on top.
- Strength: hostility has true causes, and the same simulation could keep running during play later.
- Weakness: Adams's caveats apply in full. With three or four agents on six locations there are too few interacting mechanisms for the log to be interesting, and the map states that worldgen stays thin.

## Recommendation

Build **A**, with two borrowings **[J]**:

- From B, give each event type a short list of the state it requires (a reprisal needs a prior grievance). This is a validity check on the random draw, not an odds system, and it removes the worst non sequiturs.
- From Dwarf Fortress, keep entities and an event log as the single record of the world, and render all history prose from it. Encounters, the history screen and any later journal read the same record.

Also:

- Generate with one seeded random stream passed through every step, including grammar expansion, and avoid any dependence on unordered collection iteration. That is what makes the same seed give the same world. **[V-wiki]** Dwarf Fortress logs its seeds for the same purpose.
- Build the batch tool on day one: generate fifty histories from fifty seeds and print them. The Qud authors credit this for their output quality **[V]**.
- Leave two-sided accounts (the cult's telling against the village's) for later. Qud shows it is one extra text pattern per event, and it suits the premise, but the alpha does not need it.

What A hands to the open "History feeding encounters" item on the map: for each location, who occupies it, whether they are hostile to the player, and the ids of the events that explain why.

## Not verified

- The content of the two GDC talks (Grinblat 2018, Bucklew and Grinblat 2019). I read the GDC Vault abstracts only. The 2018 slides link returned an access error.
- The form of Dwarf Fortress's Legends sentences, recalled from memory.
- How Dwarf Fortress's world-generation simulation works inside (succession, sieges, migration, megabeast attacks). Adams declines to detail it in the 2019 Q&A, and I found no first-party description. The ethics and war claims come from the community wiki, not from Adams.
- The event types Qud uses for village histories. The wiki confirms two events per village and the abandonment chance, and nothing more.
- How Ultima Ratio Regum generates history. Johnson's "Towards Qualitative Procedural Generation" (2016), cited by the Qud paper, could not be found online, and the developer's older blog posts on the subject did not turn up in the site's search.
- The Tracery paper (Compton, Kybartas and Mateas, 2015). I read the README, not the paper.
- James Ryan's dissertation "Curating Simulated Storyworlds" (2018), which I recall as the main academic treatment of the need to curate simulation output. Not read here. Adams's own statement on post-processing covers the point.
- All cost estimates.

## Sources

Read in this session:

- Tarn Adams, "Simulation Principles from Dwarf Fortress", Game AI Pro 2, chapter 41 (2015). https://www.gameaipro.com/GameAIPro2/GameAIPro2_Chapter41_Simulation_Principles_from_Dwarf_Fortress.pdf
- "Q&A: Dissecting the development of Dwarf Fortress with creator Tarn Adams", Game Developer (2019-06-03). https://www.gamedeveloper.com/design/q-a-dissecting-the-development-of-i-dwarf-fortress-i-with-creator-tarn-adams
- "Interview: The Making Of Dwarf Fortress", Gamasutra, now Game Developer (2008-02-27). https://www.gamedeveloper.com/design/interview-the-making-of-dwarf-fortress
- Dwarf Fortress wiki (community): World generation https://dwarffortresswiki.org/index.php/World_generation, Legends https://dwarffortresswiki.org/index.php/Legends, Historical figure https://dwarffortresswiki.org/index.php/Historical_figure, Ethic https://dwarffortresswiki.org/index.php/Ethic, War https://dwarffortresswiki.org/index.php/War
- Jason Grinblat and C. Brian Bucklew, "Subverting Historical Cause & Effect: Generation of Mythic Biographies in Caves of Qud", FDG 2017. https://pcgworkshop.com/archive/grinblat2017subverting.pdf
- Jason Grinblat and C. Brian Bucklew, "Warm Rocks for Cold Lizards: Generating Meaningful Quests in Caves of Qud", Experimental AI in Games workshop, AIIDE 2020. https://ceur-ws.org/Vol-2862/paper5.pdf
- Caves of Qud wiki (hosted on the game's own domain, community-edited): Sultan histories https://wiki.cavesofqud.com/wiki/Sultan_histories, World generation https://wiki.cavesofqud.com/wiki/World_generation, Village https://wiki.cavesofqud.com/wiki/Village
- GDC Vault abstracts: "Procedurally Generating History in 'Caves of Qud'" https://gdcvault.com/play/1025379/Procedurally-Generating-History-in-Caves and "End-to-End Procedural Generation in 'Caves of Qud'" https://gdcvault.com/play/1026313/Math-for-Game-Developers-End
- Max Kreminski, "History generation in Epitaph" (2018-01-09). https://mkremins.github.io/blog/history-generation-epitaph
- Epitaph source. https://github.com/mkremins/epitaph/blob/master/src/epitaph/civs.cljs and https://github.com/mkremins/epitaph/blob/master/src/epitaph/events.cljs
- Mark R Johnson, "Procedural Generation of Linguistics, Dialects, Naming Conventions and Spoken Sentences", DiGRA/FDG 2016. https://pcgworkshop.com/archive/johnson2016procedural.pdf
- Mark R Johnson's development blog, post listing. https://www.markrjohnsongames.com/
- Kate Compton, Tracery README. https://github.com/galaxykate/tracery
