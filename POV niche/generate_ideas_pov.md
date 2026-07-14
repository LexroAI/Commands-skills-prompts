# Generate Video Topic Ideas — Hierarchy POV Channel

Generate fresh "Your Life at Every Rank" video ideas that haven't been covered yet on this channel.

## CHANNEL FORMAT — Read This First

This channel produces long-form (20–30 min) second-person immersive documentaries.
Every video takes ONE closed hierarchical world and walks the viewer up (or down) its ladder,
rank by rank, making them *live* each level rather than observe it.

The viewer is never "they." The viewer is always **YOU**.

Three pillars make this format work. Every idea must satisfy all three:

**PILLAR 1 — CLOSED WORLD**
The subject must be a world outsiders cannot casually enter or observe.
It has gates, initiation, insider language, and consequences for leaving.
Good: submarine crew, cartel, monastery, oil rig, royal court.
Bad: "being a freelancer," "living in a big city" — no gate, no ladder.

**PILLAR 2 — VISIBLE LADDER**
There must be named, ordered tiers that a person moves through.
Ranks, belts, grades, clearance levels, castes, seniority, sentence length.
If you cannot list at least 6 distinct rungs, the topic fails.

**PILLAR 3 — ESCALATING COST**
Each rung up must take something away — freedom, identity, sleep, family, name, conscience.
The ladder is not a reward structure. It is a trade.
If climbing only brings benefits, the topic has no emotional engine.

---

## STEP 1 — Read Channel Config

Read `channel_config.txt` to understand the channel name and narrator identity.

## STEP 2 — Read Past Topics

Read `videos/ideas_history.txt`.
If the file doesn't exist yet, treat the history as empty.

Each line follows this format:
```
YYYY-MM-DD | topic title | thumbnail_line
```

Extract all past topics AND all past thumbnail lines.
Both must be checked for repetition in Step 4.

## STEP 3 — Read Reference World Bank

Read `worlds_covered.txt` if it exists.
This lists broad *categories* already used (military, crime, medicine, finance, etc.).
Aim for category diversity — do not generate five military topics in one batch.

---

## STEP 4 — Generate 20 New Ideas

Generate 20 original video topic ideas following these rules.

### Title construction

Use one of these four title frames. Rotate them — do not use the same frame more than 7 times in a batch:

- `Your Life as Every Rank in [World]`
- `Your Life at Every Level of [World]`
- `Your Life at Every Stage of [World]`
- `Your Life as Every [Role] in [World]`

Rules:
- The `[World]` must be a noun phrase a stranger instantly pictures. No abstractions.
- Never use "the" unless the world genuinely requires it (the CIA, the Roman Empire).
- Keep titles under 55 characters where possible.

### Thumbnail line construction

Every idea must ship with a **thumbnail line** — this is the single most important asset.

Structure: `[2–3 word setup] + [1–2 word verdict in caps]`

The verdict is a **sentence passed on the viewer**, not a description of the video.
It should read like something whispered by someone who cannot leave.

Verdict must be one of these emotional registers:
- **Command** — you have no choice: `YOU MUST OBEY`, `YOU MUST SERVE`
- **Erasure** — you cease to exist: `YOU DON'T EXIST`, `YOU ARE A GHOST`
- **Ownership** — someone owns you: `THEY OWN EVERYTHING`, `THE STATE OWNS YOU`
- **Entrapment** — you cannot leave: `NO WAY OUT`, `NO SECOND CHANCE`
- **Deprivation** — something is taken: `YOU DON'T SLEEP`, `YOU DON'T MATTER`
- **Surveillance** — you are seen: `YOU ARE WATCHED`, `YOU ARE MARKED`

Hard rules for thumbnail lines:
- Second person ONLY. The word "you" or "your" must appear.
- Present tense. Never past tense unless the world is historical.
- Maximum 5 words total.
- Never explain. `YOU MUST HIDE` beats "Life as a lottery winner is dangerous."
- Do not reuse any verdict word already in `ideas_history.txt`.

### Content rules

- Target audience: English-comprehending viewers worldwide, with weight toward US, UK, Canada, Australia.
- No topic requiring specialist knowledge to *enter*. A viewer with zero background must understand rung one within 10 seconds.
- Mix the batch: roughly 8 exotic/inaccessible worlds, 8 professional/institutional worlds, 4 historical worlds.
- Historical worlds must have a documented rank system, not a vibe.
- Avoid anything that glamorizes real ongoing violence, names real living private individuals, or reads as instructional.
- Do not generate any topic that is the same as, or a near-neighbor of, past topics.
  A near-neighbor is any topic sharing the same closed world at a different scale.
  (If "Your Life as Every Rank in CIA" exists, "Your Life as Every Rank in MI6" is a near-neighbor. Reject it.)

### Rung check

For each idea, silently verify you could name 6+ ordered rungs.
If you cannot, discard the idea and generate a replacement. Do not show this work.

---

## STEP 5 — Output Format

Output exactly this. No preamble, no commentary, no explanation.

```
1. Your Life as Every Rank in [World]
   Thumbnail: You Must SURRENDER
   World type: Exotic | Institutional | Historical

2. Your Life at Every Level of [World]
   Thumbnail: You Don't DECIDE
   World type: Institutional
```

...through 20.

---

## STEP 6 — Prompt the User

After outputting the list, tell the user:

"Pick a topic and run: /generate_script \"your chosen title here\"

If you want a different angle on the same world, run: /generate_ideas --expand \"world name\""
