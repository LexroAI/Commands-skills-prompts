---
name: generate-pov-ideas
description: >
  Generate viral YouTube video topic ideas in the "Your Life as Every Rank in X" second-person
  immersive documentary format (Master POV style). Use this skill whenever the user asks to
  brainstorm, generate, or draft video topic ideas or titles for a hierarchy POV / rank-ladder
  channel. Also trigger when the user mentions "generate pov ideas", "Hierarchy POV", "Master POV
  style", "Your Life as Every Rank", "rank ladder video", or asks for closed-world documentary
  topics with escalating-cost structure. Each idea ships as a title plus a 1–2 sentence
  explanation of the world, its ladder, and what climbing it costs. Target audience:
  English-comprehending viewers worldwide, weighted toward US, UK, Canada, and Australia.
---

# Generate POV Ideas — Video Topic Idea Generator

## What This Skill Does

Produces batches of original video topic ideas for a long-form (20–30 min) second-person
immersive documentary channel. Every video takes ONE closed hierarchical world and walks the
viewer up (or down) its ladder, rank by rank, making them *live* each level rather than
observe it.

Each idea ships as a two-part package:
1. **Title** — locked to a franchise frame
2. **Explanation** — 1–2 sentences: what the world is, what the ladder is, and what it costs

The viewer is never "they." The viewer is always **YOU**.

---

## Why This Format Works

Before generating anything, internalize the three mechanics that make this format spread:

**Franchise title lock.** Every title uses the same frame. Viewers recognize the brand from
the text alone. The algorithm learns the pattern and cross-recommends the catalog.

**Second-person immersion.** "Your Life" not "The Life of." The viewer isn't watching someone
else's story — they're inside it.

**Ladder = built-in retention.** "Every Rank" promises a climb. Each rung is a natural
cliffhanger. The viewer stays to see what the next level looks like.

---

## The Three Pillars

Every idea must satisfy all three. No exceptions.

### PILLAR 1 — CLOSED WORLD
The subject must be a world outsiders cannot casually enter or observe. It has gates,
initiation, insider language, and consequences for leaving.

- **Good:** submarine crew, cartel, monastery, oil rig, royal court, seminary, whaling ship
- **Bad:** "being a freelancer," "living in a big city" — no gate, no ladder

### PILLAR 2 — VISIBLE LADDER
There must be named, ordered tiers a person moves through. Ranks, belts, grades, clearance
levels, castes, seniority, sentence length.

If you cannot list at least 6 distinct rungs, the topic fails.

### PILLAR 3 — ESCALATING COST
Each rung up must take something away — freedom, identity, sleep, family, name, conscience.
The ladder is not a reward structure. It is a trade.

If climbing only brings benefits, the topic has no emotional engine.

---

## Title Construction

Use one of these four frames. Rotate them — no frame more than 7 times per batch of 20.

- `Your Life as Every Rank in [World]`
- `Your Life at Every Level of [World]`
- `Your Life at Every Stage of [World]`
- `Your Life as Every [Role] in [World]`

Rules:
- `[World]` must be a noun phrase a stranger instantly pictures. No abstractions.
- Never use "the" unless the world genuinely requires it (the CIA, the Roman Empire).
- Keep titles under 55 characters where possible.

---

## Explanation Construction

One or two sentences. No more. Its only job is to make the idea legible at a glance so the
user can pick a topic without asking follow-up questions.

Each explanation must do two things:

1. **Name the ladder.** State where the climb starts and where it ends — the bottom rung and
   the top rung, by name. This proves the world has ordered tiers.
2. **Name the cost.** State what the climb takes away. This proves the world has an emotional
   engine.

### Hard Rules

- Maximum 2 sentences. Aim for 30–45 words total.
- Write in second person where it reads naturally ("you start as...", "you climb from...").
- Concrete nouns for rungs. `deckhand → harpooner → captain` beats "various seafaring roles."
- Name the specific thing lost — sleep, your name, your family, your conscience. Never
  "a lot of sacrifice" or "personal challenges."
- Do not pitch, hype, or editorialize. No "this one is fascinating," no "viewers will love."
- Do not describe the video. Describe the world.

### Example

```
Your Life as Every Rank in a Nuclear Submarine
   You climb from seaman recruit to captain inside a steel tube that will not surface for
   six months. Every promotion buys you more responsibility for a boat you can never leave,
   and less sleep than the rank below you.
```

---

## Content Rules

- **Audience:** English-comprehending viewers worldwide, weighted toward US, UK, Canada, Australia.
- **Zero-entry barrier.** A viewer with no background must understand rung one within 10 seconds.
- **Batch mix:** roughly 8 exotic/inaccessible worlds, 8 professional/institutional worlds,
  4 historical worlds. Keep this balance, but do not label the ideas with their type.
- **Historical worlds** must have a documented rank system, not a vibe.
- **Avoid:** glamorizing ongoing real violence, naming real living private individuals,
  or anything that reads as instructional.

### Near-Neighbor Rejection

Do not generate any topic that is the same as, or a near-neighbor of, past topics.

A near-neighbor shares the same closed world at a different scale.
If `Your Life as Every Rank in CIA` exists, then `Your Life as Every Rank in MI6` is a
near-neighbor. Reject it.

### Rung Check

For each idea, silently verify you can name 6+ ordered rungs. If you cannot, discard the idea
and generate a replacement. Never show this work to the user.

---

## Workflow

### Step 1 — Read Channel Config
Read `channel_config.txt` for channel name and narrator identity. If absent, proceed with defaults.

### Step 2 — Read Past Topics
Read `videos/ideas_history.txt`. If it doesn't exist, treat history as empty.

Format per line:
```
YYYY-MM-DD | topic title
```

### Step 3 — Read World Bank
Read `worlds_covered.txt` if present. Lists broad categories already used (military, crime,
medicine, finance). Aim for category diversity — never five military topics in one batch.

### Step 4 — Generate
Produce 20 ideas obeying every rule above.

### Step 5 — Output
Output exactly this shape. No preamble, no commentary, no explanation of your process.
No thumbnail lines. No world-type labels.

```
1. Your Life as Every Rank in [World]
   [One to two sentences: the ladder from bottom rung to top, and what the climb costs.]

2. Your Life at Every Level of [World]
   [One to two sentences: the ladder from bottom rung to top, and what the climb costs.]
```

...through 20.

### Step 6 — Close
After the list, tell the user:

> Pick a topic and run: `/generate_script "your chosen title here"`
>
> If you want a different angle on the same world, run: `/generate_ideas --expand "world name"`

---

## Quality Checklist

Before returning a batch, confirm:

- [ ] All 20 titles use a franchise frame; no frame used more than 7 times
- [ ] Every idea has an explanation of 2 sentences or fewer
- [ ] Every explanation names a bottom rung and a top rung by name
- [ ] Every explanation names a specific thing the climb takes away
- [ ] No thumbnail lines and no world-type labels appear in the output
- [ ] Batch mix approximates 8 exotic / 8 institutional / 4 historical, unlabeled
- [ ] Every topic passes the 6-rung test
- [ ] No near-neighbors of past topics
- [ ] No topic glamorizes ongoing violence or names real private individuals
