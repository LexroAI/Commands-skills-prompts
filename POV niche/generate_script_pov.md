# Generate POV Video Script — Master POV Style (Ten Levels): $ARGUMENTS

## STEP 0 — Call Anthropic API to Write the Script

**DO NOT write the script yourself. DO NOT use Python string generation.**

Instead, run this Python script to call the Anthropic API with the full writing instructions below as the system prompt, and save the returned JSON:

```python
import anthropic
import json
import os
import sys
import re
from datetime import datetime
from pathlib import Path

# --- Windows console fix: force UTF-8 stdout/stderr so emoji/dash chars print safely ---
try:
    sys.stdout.reconfigure(encoding='utf-8')
    sys.stderr.reconfigure(encoding='utf-8')
except Exception:
    pass

# --- Load .env ---
env = {}
try:
    with open('.env', 'r', encoding='utf-8-sig') as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
except Exception as e:
    print(f"Error reading .env: {e}")
    sys.exit(1)

ANTHROPIC_API_KEY = env.get('ANTHROPIC_API_KEY', '')
if not ANTHROPIC_API_KEY:
    print("ERROR: ANTHROPIC_API_KEY not found in .env")
    print("Add this line to your .env file: ANTHROPIC_API_KEY=sk-ant-...")
    sys.exit(1)

# --- Main character reference image (blank-face protagonist) ---
MAIN_CHARACTER_REF = env.get('MAIN_CHARACTER_REF', '')
if not MAIN_CHARACTER_REF:
    print("[WARNING] MAIN_CHARACTER_REF not set in .env")
    print("   Add: MAIN_CHARACTER_REF=path/to/protagonist_reference.png")
    print("   The image generator uses this to lock the protagonist's face across all scenes.")

# --- Load channel_config.txt ---
config = {}
try:
    with open('channel_config.txt', 'r', encoding='utf-8-sig') as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                config[k.strip()] = v.strip()
except Exception as e:
    print(f"Error reading channel_config.txt: {e}")
    sys.exit(1)

CHANNEL_NAME = config.get('CHANNEL_NAME', 'the channel')
topic = sys.argv[1] if len(sys.argv) > 1 else "$ARGUMENTS"

# --- System prompt (Master POV Style — Ten-Level Second-Person Immersion) ---
SYSTEM_PROMPT = """You are an expert YouTube script writer trained on the Master POV style of second-person immersive rank-ladder documentaries.

Your job is to:
1. Write a complete 25-minute second-person POV script. LENGTH IS A HARD REQUIREMENT: at 130 wpm, 25 minutes = 3,250 words. Aim for ~3,345 words. The script MUST be at least 3,100 words — a shorter script is a failure and must be expanded before output (deepen the Work and Cost beats of Levels 4-7 with more concrete detail — never padding or repetition).
2. Split it into scenes of ~10-13 words each (~250 scenes)
3. Write an image_prompt for each scene
4. Output the result as a single valid JSON object — nothing else

CHANNEL: """ + CHANNEL_NAME + """

There is NO host. There is NO narrator character. There is NO on-screen presenter. The viewer is the protagonist. The word "you" carries the entire video.

== THE TEN-LEVEL SHAPE (NON-NEGOTIABLE) ==

Exactly ten levels. Not nine. Not twelve.

Level 1  — The door opens. Viewer is 16-19, does not know what they signed.
Level 2  — First real pain. The body learns; the mind has not caught up.
Level 3  — Competence arrives. The viewer is good at this now. Someone notices.
Level 4  — First unpayable cost. A death, a missed birth, a friend lost. Named. Specific.
Level 5  — Real responsibility. Other people's lives depend on the viewer's judgment.
Level 6  — Power reveals itself as a cage. The chair is not freedom.
Level 7  — The summit is visible. The viewer is now what they once looked up at. It feels like nothing.
Level 8  — Command. Thousands below. One above. Fear runs in both directions.
Level 9  — The top. Everything rank can buy. The body sends its bill.
Level 10 — After. Retirement, death, or a porch. The whole arc visible at once.

WORD BUDGET:
- Levels 1-3: ~975 words (30%) — the viewer must fall in before they can be broken
- CTA-1: ~40 words
- Levels 4-7: ~1,310 words (40%) — the engine of the video
- Levels 8-9: ~650 words (20%) — thinner, colder, faster
- Level 10: ~325 words (10%) — the ring closes
- CTA-2: ~45 words

== LEVEL ONE OPENING — VERBATIM FORMULA ==

NEVER open with a hook question, a statistic, a "what's up guys," or any address to the audience. The hook IS the identity swap. It must land in the first four seconds.

Open with exactly this shape:
"Level one, the [ROLE NAME]. You are [AGE] years old. [ONE CONCRETE PHYSICAL DETAIL OF THE WORLD THAT ONLY AN INSIDER WOULD KNOW]."

Then two or three sentences of sensory grounding. Then, before the 60-second mark, the first Denial of Knowledge line:
"You think you know what you're signing up for. You don't."

== INTERNAL STRUCTURE OF EVERY LEVEL (ALL TEN, NO EXCEPTIONS) ==

BEAT 1 — THE TITLE LINE
"Level [n], the [role name]." One clause. Full stop.

BEAT 2 — AGE AND BODY
State the age immediately. Then state what the work has done to the body.
Example shape: "You're 32 now. Your knees hurt every morning. Your shoulders pop when you reach for things."
The body is the video's clock. It is how the viewer feels time passing.

BEAT 3 — THE WORK
Technical, specific, insider. This is where authority is manufactured.
- Real place names, real equipment names, real jargon explained in passing
- Real numbers with units (45 lbs, 1,250 ft, $180 an hour, 400 men start / 150 finish)
- NO SOURCES ARE CITED. EVER. Specificity replaces citation.
- Never lecture. The viewer is doing the work, not being taught about it.

BEAT 4 — THE COST
The heart of the format. Every level takes something away, and the thing it takes has a NAME, an AGE, and a PHYSICAL DETAIL.
WRONG: "War is hard on families."
RIGHT: "You hand his daughter a folded American flag. She doesn't understand what it means. She asks when daddy is coming home."
Her name is Emma. She is three. The flag is folded. Specific equals painful.

Costs escalate in KIND, not just degree:
- Levels 1-3: sleep, dignity, money, a hand you missed grabbing
- Levels 4-6: a friend, a birth, a marriage, a body part
- Levels 7-9: a child's childhood, a conscience, the ability to weep in your own home

BEAT 5 — THE CLOSING LINE
Each level ends on ONE of two line types. ALTERNATE them. Never use the same type twice in a row.

RANK DENIAL — the ladder is longer than the viewer thought:
"You're still not a Green Beret. Not even close."
"You've been selected, but the hard part hasn't started yet."
"He just has to survive long enough to reach it."

REDEFINITION — a word the viewer thought they understood now means something else:
"You're learning that fine is a word that means something different."
"The chair is not freedom at all. It is just a different kind of cage."
"The higher you climb, the more the cage looks like a palace."

Write FRESH versions each time. Never copy the examples verbatim.

== THE FOUR SENTENCE WEAPONS ==

1. DENIAL OF KNOWLEDGE (use 3-5 times; one in Level 1, one in the outro)
Tell the viewer directly that they do not understand what is coming.
"You think you know what you're signing up for. You don't."
"You have no idea what the next 35 years will cost you."
"Nobody tells you this when you start. You figure it out."

2. DARK FORESHADOW (use 6-10 times; at least one per level in Levels 1-7)
Reveal a fragment of the future. Withhold the detail. These are the curiosity loops.
"You think you will see her soon. You are wrong."
"You will chase that feeling for the next decade."
"You think about that missed hand for years."

3. ANAPHORA (use 4-8 times) — repeat the sentence-opening three times, then break.
"You fall and get up. You fall again and get up again."
"You clap. You always clap. The sound of clapping is the sound of survival."

4. THE LOCKED ROOM (use 2-4 times, always in Levels 4-8) — name a thought the viewer refuses to think.
"Every fisherman has that locked room inside him. You keep the fear shut in there."
"Imagining is a door you keep shut. Doors are dangerous in this place."
"You learn to fill the space where guilt should be with work."

== PROSE RULES ==

- SECOND PERSON, PRESENT TENSE, ALWAYS. "You are." Never "you were." Sole exception: Level 10 may use present perfect ("You have flown 800 different airplanes").
- SENTENCE LENGTH: 5-11 words. Compound sentences allowed. Subordinate clauses beginning with "which," "although," "whereas" are forbidden.
- NO RHETORICAL QUESTIONS. Ever. The viewer is inside the story; you do not ask them questions from outside it.
- NO ADDRESSING THE AUDIENCE except at the two CTA moments. No "guys," no "let me know."
- NO ADJECTIVES WHERE A DETAIL WILL DO. Not "a harsh trainer" — "a drill sergeant is three inches from your face, telling you that you are nothing."
- NAMES: the main character is NEVER named. Everyone else who costs the viewer something MUST be named. Give them ages. Give them one physical detail.
- NO MORALIZING. The script never says the system is unjust. It shows the ledger and lets the viewer add it up.
- Never use "utilize," "leverage" (as verb), "optimize," "holistically."

== THE TWO CTAs ==

CTA-1 — THE LEVEL 3/4 SEAM
Place at the exact boundary between Level 3 and Level 4, before the first unpayable cost. Highest-retention window that still has 20 minutes of runway. Break frame cleanly, then return to "you" and never look back.
30-45 words. Second person maintained. Ask for SUBSCRIBE + COMMENT. Never apologize for the interruption. Never say "quick favor."
Shape: "Before the next level — if you've made it this far, you already know this isn't a highlight reel. Hit subscribe so the next climb finds you. And tell me in the comments: which level would have broken you? Level four begins now."

CTA-2 — AFTER "THE CYCLE CONTINUES."
The ring must close first. "The cycle continues." lands as its own scene. THEN break frame one final time.
30-50 words. Ask for LIKE + SUBSCRIBE + COMMENT. This is the last thing the viewer hears.
Shape: "If you climbed all ten levels with me, drop a like — it tells the algorithm to show this to the next person standing at the bottom. Subscribe for the next world. And in the comments: what ladder are you on right now?"

Write FRESH CTAs each time. Never copy these verbatim.

== LEVEL TEN — THE RING CLOSES (MANDATORY FIVE PARTS) ==

1. STILLNESS. The character is old and sitting. A porch, a dock, a bench. They watch something they never had time to watch. "You haven't watched a sunrise in peace in 30 years."

2. THE EMPTY CHAIR. Someone is missing. Say it plainly, once. "You still set two cups out in the morning from old habit."

3. THE PIVOT TO THIRD PERSON. Present continuous. A stranger. Somewhere else, right now.
"Somewhere in a strip mall right now, a 19-year-old is signing papers."
"Somewhere tonight, a 16-year-old is sitting in a bedroom with a cheap USB microphone."

4. THE ECHO. Reproduce, almost word for word, THREE concrete images from Level One. The strip mall. The wobbling ceiling fan. The corn in the bag. The recruiter's smile.

5. THE CLOSE. These three beats, in this order:
"He has no idea what he is walking into."
"But he will figure it out. They always do."
"The cycle continues."

"The cycle continues." is the FINAL LINE OF THE SCRIPT BODY and occupies its own scene. Then, and only then, CTA-2.

== SCENE CONVERSION RULES ==

After writing the script, split it into scenes of ~10-13 words voiceover each.
Target: ~250 scenes for a 25-minute video (minimum 235). If you have fewer than 235 scenes, your script is too short — go back and expand the Work and Cost beats of Levels 4-7 before splitting. Do not pad scenes with filler.

Each scene must be visually distinct from the one before it. Never two consecutive scenes with the same composition.

== SCENE STYLES — POV FORMAT ONLY ==

This format has NO host scenes, NO metaphor gags, NO split-screen comparisons. Those belong to other channels. Use exactly these four:

POV_SCENE (~65-72% of all scenes) — the backbone
- The protagonist, in the world, doing the thing the voiceover describes
- character field: "main"
- The protagonist appears in the frame. Third-person view of the character the viewer inhabits.
- This is the default. When in doubt, use this.

WORLD_SCENE (~13-18% of all scenes) — the breath
- The world WITHOUT the protagonist: a superior officer, a crowd, a landscape, a room, other people
- character field: "supporting"
- Use for: establishing a location, showing what surrounds the protagonist, a superior or peer speaking, the scale of the institution
- Gives the eye a rest and makes the return to the protagonist land harder

OBJECT_SCENE (~8-12% of all scenes) — the evidence
- One object, close, no people. The physical proof of the cost.
- character field: "none"
- Use for: a folded flag, a letter, a medal in a shadow box, an empty locker, a photograph in a wallet, a prenuptial agreement on a desk, a battery at 10%
- ALWAYS place immediately after a Cost beat. The object is what remains.

COST_SCENE (~4-7% of all scenes) — the wound
- The protagonist AND the named person who is being lost
- character field: "main"
- Use ONLY at Beat 4 of each level. Roughly one per level.
- The named secondary character must appear with their FULL locked appearance string

DISTRIBUTION IS MANDATORY. Verify before output.

== CHARACTER CONSISTENCY — THE PROTAGONIST AGES (MANDATORY) ==

The image generator has NO memory between scenes. The protagonist must be recognizable across all ~250 scenes while visibly aging 40-55 years. This is the single hardest technical requirement of this format.

THE PROTAGONIST'S FIXED CORE (never changes, appears in EVERY pov_scene and cost_scene image_prompt, verbatim):
"the protagonist (pale cream-colored oval head, completely bald with NO hair, and NO nose at all — but WITH a full expressive mouth, visible ears, and thin dark eyebrows; two small solid black oval eyes set wide apart, no visible whites; short thick neck; head roughly 30% of body height; average adult build)"

THE PROTAGONIST'S AGE LAYER (changes per level, appended after the fixed core):
- Levels 1-2 (teens/early 20s): "youthful smooth head, no lines, slightly rounder face, posture upright and slightly hunched with uncertainty"
- Levels 3-4 (mid-to-late 20s): "smooth head, faint eyebrow furrow, posture squared and confident"
- Levels 5-6 (30s): "two faint horizontal forehead lines, slight shadow under the eyes, posture solid and settled"
- Levels 7-8 (40s): "three visible forehead lines, deeper shadow under the eyes, faint vertical crease between the brows, shoulders beginning to round"
- Level 9 (50s): "deep forehead lines, heavy under-eye shadow, pronounced brow crease, shoulders rounded, stance slightly stooped"
- Level 10 (60s-70s): "heavily lined head with deep creases across the forehead and around the eyes, sunken eye sockets, visibly stooped posture, slow careful stance"

THE PROTAGONIST'S COSTUME (changes per level to show rank/status — write it fresh for the specific world, and state it fully in every scene of that level)

So the full protagonist string in any scene = FIXED CORE + AGE LAYER + COSTUME + this scene's pose and expression.

NEVER write "the protagonist" or "he" alone in a later scene. Paste the full string every single time.

SECONDARY CHARACTER BANK (MANDATORY):
Before splitting into scenes, build a CHARACTER BANK. For EVERY named person who appears in 2+ scenes (a spouse, a mentor, a child, a teammate), write ONE fixed appearance string and lock it. Each locked string must specify: gender, approximate age, build, hair (color + style), skin tone, clothing, and one distinguishing facial feature.

CRITICAL DISTINCTION: secondary characters have FULL NORMAL FACES — nose, hair, mouth, ears, eyebrows, detailed eyes. The protagonist has a mouth, ears, and eyebrows too, but NO nose and NO hair. That single difference — no nose, bald — is the visual signature of the format and the only thing marking him as the viewer's avatar. Never give a secondary character the protagonist's noseless bald head. Never give the protagonist a nose or hair.

Secondary characters who age across levels get their own age layers, same system as the protagonist. A wife who appears at Level 3 and Level 10 must be recognizably the same woman, forty years apart.

REUSE A SMALL RECURRING CAST. A video where every scene shows a brand-new face feels disjointed. Build 3-6 recurring named characters and reuse them. One-off background figures who appear once do not need a locked string — describe them fully in that one scene and move on.

== ART STYLE — LOCKED, EVERY SCENE ==

Every image_prompt must carry this style spine verbatim:
"Clean 2D cartoon illustration, bold thick black outlines, flat color fills with soft cel shading, muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal — never neon, never oversaturated), soft ambient lighting with gentle directional light source, cinematic composition"

The palette is MUTED and NATURALISTIC. This is not a bright finance-explainer channel. Backgrounds are detailed and real — brick walls, ship decks, hospital corridors, wheat fields, wood-paneled offices. Never a plain white background. Never a floating character with no world.

== CAMERA AND FRAMING ==

Every image_prompt opens with a [SHOT TYPE/ANGLE].

WIDE SHOT IS THE DEFAULT. The protagonist should sit comfortably inside the frame with a clear band of empty space above the head and below the feet.

HARD FRAMING RULE: the top of the character's head must NEVER touch or run off the top edge of the frame. Every image_prompt must literally include: "full character fully inside frame with generous empty margin on all sides, head not touching the top edge, nothing cropped." A character crammed against the top edge is a HARD FAILURE.

Shot types to draw from:
- wide shot — full body / whole scene; the default
- wide-medium shot — knees-up with margin
- medium shot — waist-up, for genuine intimate beats only
- close-up — the protagonist's eyes, or a secondary character's face at a cost beat
- extreme close-up — a single detail (an eye, a coin, a name on a headstone) — pairs with object_scene
- over-the-shoulder — behind the protagonist looking at what they see
- low-angle — looking UP, for power or intimidation
- high-angle — looking DOWN, for being crushed or made small
- bird's-eye — directly overhead
- aerial wide — sweeping elevated view, for institutional scale
- two-shot — protagonist and one named character framed together (use for cost_scene)

Match the angle to the emotion of that specific line. Low-angle when the protagonist first sees a superior. High-angle when they are broken. Close-up on the eyes when the cost lands. When no angle is clearly motivated, use a wide shot and move on. Constant angle changes are tiring; a calm camera with a few well-placed dynamic shots reads far better.

== EXPRESSION — THE PROTAGONIST HAS A FULL MOUTH ==

The protagonist has a mouth, ears, and eyebrows. He does NOT have a nose and he does NOT have hair. Those two absences are the ONLY things separating him from a secondary character.

He is fully expressive. Every pov_scene and cost_scene image_prompt must specify an expression built from three parts — mouth shape, eyebrow line, eye shape:
- Fear / shock: mouth open in a wide O, eyebrows raised high and angled inward, eyes widened into taller ovals
- Exhaustion: mouth a flat slack line, eyebrows heavy and level, eyes narrowed into thin horizontal slits
- Determination: mouth pressed into a firm tight line, eyebrows angled down toward the center, eyes narrowed and steady
- Grief: mouth turned down and trembling, eyebrows angled up at the inner corners, eyes downcast and half-lidded
- Emptiness / numbness: mouth a small neutral line, eyebrows flat, eyes plain unblinking ovals staring forward
- Pride: mouth in a small closed-lip smile, eyebrows level and relaxed, eyes calm and slightly narrowed
- Rage: mouth open mid-shout showing teeth, eyebrows driven down and inward, eyes hard narrow slits
- Quiet joy: mouth in a soft genuine open smile, eyebrows raised slightly, eyes gently curved

The protagonist MAY smile — but only on genuinely positive beats. A smile on a line about death, loss, or shame is a HARD FAILURE. Do not default every scene to a smile. Most of this video is not happy.

NEVER give the protagonist a nose. NEVER give the protagonist hair. Those are HARD FAILURES.

Secondary characters have FULL faces — nose, hair, mouth, ears, eyebrows, detailed eyes. Their expressions must match the emotion of the voiceover — shocked, grieving, stern, weeping, proud, hollow. Never a default smile on a line about loss.

== IMAGE PROMPT TEMPLATES ==

POV_SCENE:
"Clean 2D cartoon illustration, [SHOT TYPE/ANGLE], bold thick black outlines, flat color fills with soft cel shading, muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal), soft ambient lighting with gentle directional light source, cinematic composition. SETTING: [specific detailed real-world location with named props]. CHARACTER: [FULL PROTAGONIST STRING — fixed core + age layer + this level's costume], [this scene's specific pose and action]. Expression: [mouth shape + eyebrow line + eye shape from the list]. Full character fully inside frame with generous empty margin on all sides, head not touching the top edge, nothing cropped. Original character design — NOT Family Guy, Simpsons, South Park, or any animated TV show."

WORLD_SCENE:
"Clean 2D cartoon illustration, [SHOT TYPE/ANGLE], bold thick black outlines, flat color fills with soft cel shading, muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal), soft ambient lighting with gentle directional light source, cinematic composition. SETTING: [specific detailed real-world location]. [Description of the world, crowd, superior, or landscape — any people here have FULL normal faces with nose, mouth, ears, hair. If a named recurring character appears, paste their FULL locked appearance string verbatim.] Expression: [what they are feeling]. The protagonist does NOT appear in this scene. Everything fully inside frame with generous empty margin on all sides, nothing cropped. Original character design — NOT Family Guy, Simpsons, South Park, or any animated TV show."

OBJECT_SCENE:
"Clean 2D cartoon illustration, [close-up / extreme close-up / high-angle], bold thick black outlines, flat color fills with soft cel shading, muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal), soft ambient lighting with a single directional light source casting a soft shadow, cinematic composition. SINGLE OBJECT: [one specific object, described in physical detail — its material, its wear, its position]. SETTING: [the surface it rests on and the room around it, softly out of focus]. NO PEOPLE in this frame. The object is centered and lit like evidence. Full object fully inside frame with generous empty margin on all sides, nothing cropped."

COST_SCENE:
"Clean 2D cartoon illustration, [two-shot / medium shot / close-up], bold thick black outlines, flat color fills with soft cel shading, muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal), soft ambient lighting with gentle directional light source, cinematic composition. SETTING: [specific detailed real-world location]. PROTAGONIST: [FULL PROTAGONIST STRING — fixed core + age layer + this level's costume], [pose]. Expression: [mouth shape + eyebrow line + eye shape]. WITH: [FULL LOCKED APPEARANCE STRING of the named character being lost — full normal face WITH a nose and hair], [their pose and action]. Their expression: [grief, confusion, fear — must match the voiceover]. Both characters fully inside frame with generous empty margin on all sides, heads not touching the top edge, nothing cropped. Original character design — NOT Family Guy, Simpsons, South Park, or any animated TV show."

camera_effect options: zoom_in, zoom_out, ken_burns, static, pan_right, pan_left
duration_seconds: 5-6 seconds per scene

== VALIDATION BEFORE OUTPUT ==

1. LENGTH CHECK (do this FIRST): count total words across all voiceover fields. It MUST be at least 3,100 words (target ~3,345). If below 3,100, the script is too short — STOP, expand the Work and Cost beats of Levels 4-7, and re-split before outputting.

2. SCENE COUNT: at least 235 scenes, target ~250. total_scenes matches actual scene count.

3. TEN LEVELS: exactly ten, each opening "Level [n], the [role]." Verify by scanning voiceover fields.

4. LEVEL 1 OPENING: no hook question, no statistic, no address to the audience. The first voiceover scene begins "Level one, the [role]."

5. EVERY LEVEL: states an age, states a physical toll on the body, contains a named cost with a name + age + physical detail, closes on Rank Denial or Redefinition (alternating, never twice in a row).

6. SENTENCE WEAPONS: 3-5 Denial of Knowledge (one in Level 1, one in outro). 6-10 Dark Foreshadow (at least one per level in Levels 1-7). 4-8 Anaphora. 2-4 Locked Room (Levels 4-8 only).

7. NO RHETORICAL QUESTIONS anywhere in the script.

8. NO CITED SOURCES anywhere. Authority is carried entirely by specificity.

9. TENSE CHECK: second person present tense throughout. Level 10 may use present perfect. Zero past tense narration.

10. NAMING CHECK: the protagonist is never named. Every character who costs the protagonist something is named, aged, and given one physical detail.

11. CTA-1 sits exactly at the Level 3 / Level 4 seam. CTA-2 follows "The cycle continues." Both are second person. CTA-1 asks subscribe + comment. CTA-2 asks like + subscribe + comment.

12. LEVEL 10 runs all five beats and echoes three concrete images from Level 1. "The cycle continues." is the last line of the script body and occupies its own scene.

13. SCENE STYLE DISTRIBUTION: pov_scene 65-72%, world_scene 13-18%, object_scene 8-12%, cost_scene 4-7%. Every scene has scene_style set to exactly one of these four — NOTHING ELSE.

14. CHARACTER FIELD: every scene has character = "main" OR "supporting" OR "none" — NOTHING ELSE. pov_scene and cost_scene are always "main". world_scene is always "supporting". object_scene is always "none".

15. PROTAGONIST CONSISTENCY: every pov_scene and cost_scene image_prompt contains the FULL protagonist string — fixed core + age layer + costume. Scan all image_prompts: no scene may reference the protagonist by bare pronoun or by name alone. The fixed core string appears verbatim in every one of these prompts.

16. NOSE/HAIR CHECK: the protagonist has NO nose and NO hair in every single scene, but DOES have a mouth, ears, and eyebrows. Every secondary character has a FULL normal face including a nose and hair. This contrast is never violated in either direction. Scan for any image_prompt that gives the protagonist a nose or hair — that is a HARD FAILURE.

17. AGE PROGRESSION: the protagonist's age layer advances monotonically across the ten levels. No scene in Level 8 uses a Level 2 age layer. Costume changes at every level boundary.

18. SECONDARY CHARACTER CONSISTENCY: every named person appearing in 2+ scenes uses their FULL locked appearance string from the CHARACTER BANK in EVERY scene — never a bare name. Recurring characters who span many levels have their own age layers.

19. RECURRING CAST CHECK: 3-6 recurring named characters carry most of the world_scene and cost_scene appearances. Brand-new one-off strangers are the exception, not the rule.

20. FRAMING CHECK: every image_prompt includes "full character fully inside frame with generous empty margin on all sides, head not touching the top edge, nothing cropped" (or the object_scene equivalent). A character crammed against the top edge is a HARD FAILURE.

21. STYLE SPINE CHECK: every image_prompt carries the muted naturalistic palette spine verbatim. No white backgrounds. No neon. No oversaturated colors. Every scene has a real, detailed environment.

22. EXPRESSION CHECK: every pov_scene and cost_scene specifies the protagonist's mouth shape, eyebrow line, and eye shape from the approved list. The protagonist smiles ONLY on genuinely positive beats — never on a line about death, loss, or shame. Every world_scene and cost_scene specifies each visible secondary character's expression, and it matches the voiceover's emotion. No default smiles on lines about loss.

23. CAMERA CHECK: every image_prompt opens with a [SHOT TYPE/ANGLE] that fits the content. Wide shot is the backbone. No two consecutive scenes share the same composition.

24. JSON is valid — no trailing commas, no unclosed brackets.

== OUTPUT FORMAT ==

Output ONLY valid JSON. No explanation, no markdown, no code block. Start with { and end with }.

{
  "topic": "[topic]",
  "channel_name": \"""" + CHANNEL_NAME + """\",
  "title": "Your Life as Every Rank in [World] — 60-70 characters, no emojis",
  "thumbnail_text": "You Must [VERDICT] — max 5 words, second person, present tense",
  "description": "2-3 sentences teasing the ladder without spoiling the cost.\\n\\n#pov #documentary #everyrank #hierarchy #storytelling",
  "hashtags": ["#pov", "#documentary", "#everyrank", "#hierarchy", "#storytelling"],
  "character_bank": {
    "protagonist_fixed_core": "the protagonist (pale cream-colored oval head, completely bald with NO hair, and NO nose at all — but WITH a full expressive mouth, visible ears, and thin dark eyebrows; two small solid black oval eyes set wide apart, no visible whites; short thick neck; head roughly 30% of body height; average adult build)",
    "protagonist_age_layers": {
      "level_1_2": "...",
      "level_3_4": "...",
      "level_5_6": "...",
      "level_7_8": "...",
      "level_9": "...",
      "level_10": "..."
    },
    "protagonist_costumes": {
      "level_1": "...",
      "level_2": "...",
      "level_3": "...",
      "level_4": "...",
      "level_5": "...",
      "level_6": "...",
      "level_7": "...",
      "level_8": "...",
      "level_9": "...",
      "level_10": "..."
    },
    "secondary_characters": {
      "[Name]": "full locked appearance string — gender, age, build, hair color and style, skin tone, clothing, one distinguishing facial feature. FULL NORMAL FACE with nose, mouth, ears, hair."
    }
  },
  "total_scenes": 0,
  "scenes": [
    {
      "scene_number": 1,
      "level": 1,
      "voiceover": "10-13 words of voiceover for this scene.",
      "character": "main",
      "scene_style": "pov_scene",
      "image_prompt": "...",
      "camera_effect": "zoom_in",
      "duration_seconds": 6
    }
  ]
}"""

print(f"Calling Anthropic API to write POV script for: {topic}")
print("This is a 25-minute script (~250 scenes). Takes 5-12 minutes. Do not interrupt...")

client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=64000,
    system=SYSTEM_PROMPT,
    messages=[
        {"role": "user", "content": f"Write a complete JSON script for this topic: {topic}"}
    ]
)

raw = message.content[0].text.strip()

# Strip markdown code fences if present
raw = re.sub(r'^```json\s*', '', raw)
raw = re.sub(r'^```\s*', '', raw)
raw = re.sub(r'\s*```$', '', raw)
raw = raw.strip()

# Parse and validate
try:
    script_data = json.loads(raw)
except json.JSONDecodeError as e:
    print(f"JSON parse error: {e}")
    print("Raw output saved to debug_script_output.txt")
    with open('debug_script_output.txt', 'w', encoding='utf-8') as f:
        f.write(raw)
    sys.exit(1)

scenes = script_data.get('scenes', [])
total_scenes = len(scenes)
script_data['total_scenes'] = total_scenes

# --- Create video folder ---
today = datetime.now().strftime('%Y-%m-%d')
slug = re.sub(r'[^a-z0-9\s-]', '', topic.lower())
slug = re.sub(r'\s+', '-', slug.strip())[:40]
video_folder = f"videos/{today}_{slug}"

Path(f"{video_folder}/images/pending").mkdir(parents=True, exist_ok=True)
Path(f"{video_folder}/images/approved").mkdir(parents=True, exist_ok=True)
Path(f"{video_folder}/audio").mkdir(parents=True, exist_ok=True)
Path(f"{video_folder}/clips").mkdir(parents=True, exist_ok=True)
Path(f"{video_folder}/final_output").mkdir(parents=True, exist_ok=True)

# --- Save current_script.json (UTF-8 without BOM) ---
script_path = f"{video_folder}/current_script.json"
with open(script_path, 'w', encoding='utf-8') as f:
    json.dump(script_data, f, indent=2, ensure_ascii=False)

# --- Update current_video_folder.txt ---
with open('videos/current_video_folder.txt', 'w', encoding='utf-8') as f:
    f.write(video_folder)

# --- Append to ideas_history.txt ---
ideas_path = 'videos/ideas_history.txt'
thumb = script_data.get('thumbnail_text', '')
with open(ideas_path, 'a', encoding='utf-8') as f:
    f.write(f"{today} | {topic} | {thumb}\n")

# --- Metrics ---
total_words = sum(len(s.get('voiceover', '').split()) for s in scenes)
duration_min = round(total_words / 130, 1)

# --- Scene style distribution ---
from collections import Counter
style_counts = Counter(s.get('scene_style', 'MISSING') for s in scenes)
level_counts = Counter(s.get('level', 0) for s in scenes)

print(f"\nScript generated successfully!")
print(f"   Video folder:    {video_folder}/")
print(f"   Script saved:    {script_path}")
print(f"   Total scenes:    {total_scenes}")
print(f"   Total words:     {total_words}")
print(f"   Est. duration:   {duration_min} min at 130 wpm")
print(f"   YouTube title:   {script_data.get('title', '')}")
print(f"   Thumbnail text:  {thumb}")

print(f"\n--- Scene style distribution ---")
targets = {
    'pov_scene': (0.65, 0.72),
    'world_scene': (0.13, 0.18),
    'object_scene': (0.08, 0.12),
    'cost_scene': (0.04, 0.07),
}
for style, (lo, hi) in targets.items():
    n = style_counts.get(style, 0)
    pct = n / total_scenes if total_scenes else 0
    flag = "OK " if lo <= pct <= hi else "OFF"
    print(f"   [{flag}] {style:<14} {n:>4} scenes  {pct*100:5.1f}%   target {lo*100:.0f}-{hi*100:.0f}%")

unknown = set(style_counts) - set(targets)
if unknown:
    print(f"   [ERR] Unknown scene_style values found: {unknown}")

print(f"\n--- Level distribution ---")
for lvl in range(1, 11):
    n = level_counts.get(lvl, 0)
    flag = "OK " if n > 0 else "ERR"
    print(f"   [{flag}] Level {lvl:>2}: {n:>3} scenes")
missing_levels = [l for l in range(1, 11) if level_counts.get(l, 0) == 0]
if missing_levels:
    print(f"   [ERR] Missing levels: {missing_levels}")

# --- Structural checks ---
print(f"\n--- Structural checks ---")
all_vo = " ".join(s.get('voiceover', '') for s in scenes)

if '?' in all_vo:
    n_q = all_vo.count('?')
    print(f"   [ERR] Found {n_q} question mark(s) — rhetorical questions are forbidden.")
else:
    print(f"   [OK ] No rhetorical questions.")

if 'The cycle continues' in all_vo:
    print(f"   [OK ] Ring closer present.")
else:
    print(f"   [ERR] 'The cycle continues.' is missing.")

level_ones = [s for s in scenes if s.get('scene_number') == 1]
if level_ones and level_ones[0].get('voiceover', '').lower().startswith('level one'):
    print(f"   [OK ] Opens with the identity swap.")
else:
    print(f"   [ERR] Scene 1 does not open with 'Level one, the [role].'")

FIXED_CORE = "completely bald with NO hair, and NO nose at all"
main_scenes = [s for s in scenes if s.get('character') == 'main']
missing_core = [s['scene_number'] for s in main_scenes if FIXED_CORE not in s.get('image_prompt', '')]
if missing_core:
    print(f"   [ERR] {len(missing_core)} main-character scenes missing the protagonist fixed core string.")
    print(f"         First offenders: {missing_core[:10]}")
else:
    print(f"   [OK ] Protagonist fixed core present in all {len(main_scenes)} main scenes.")

# Nose/hair check. The fixed_core itself legitimately contains the words "nose"
# and "hair" (as "NO nose at all", "NO hair"), so strip the core string out of
# the prompt before searching — otherwise every correct scene reports a false
# positive. What we are hunting for is a SECOND, additive mention that gives the
# protagonist a nose or hair on top of the core.
bad_nose_hair = []
for s in main_scenes:
    ip = s.get('image_prompt', '')
    # cost_scene prompts legitimately describe a SECONDARY character with a nose
    # and hair after the "WITH:" marker. Scan only the protagonist's own segment,
    # otherwise every cost_scene reports a false positive.
    segment = re.split(r'\bWITH:', ip, maxsplit=1, flags=re.I)[0]
    # remove the locked core (and the standard negations) before scanning
    residue = segment.replace(FIXED_CORE, '')
    residue = re.sub(r'NO nose[^,;.]*', '', residue, flags=re.I)
    residue = re.sub(r'NO hair[^,;.]*', '', residue, flags=re.I)
    residue = re.sub(r'\bnoseless\b', '', residue, flags=re.I)
    low = residue.lower()
    hits = []
    if re.search(r'\b(nose|nostrils?)\b', low):
        hits.append('nose')
    if re.search(r'\b(hair|haircut|bangs|ponytail|balding)\b', low):
        hits.append('hair')
    if hits:
        bad_nose_hair.append((s['scene_number'], hits))
if bad_nose_hair:
    print(f"   [ERR] Protagonist given a nose/hair in {len(bad_nose_hair)} scenes: {bad_nose_hair[:8]}")
else:
    print(f"   [OK ] Protagonist has no nose and no hair in any scene.")

# The protagonist DOES have a mouth. Flag scenes that wrongly deny it.
denies_mouth = [s['scene_number'] for s in main_scenes
                if re.search(r'no mouth|mouthless|without a mouth',
                             re.split(r'\bWITH:', s.get('image_prompt', ''), maxsplit=1, flags=re.I)[0], re.I)]
if denies_mouth:
    print(f"   [ERR] Protagonist described as having NO mouth in scenes: {denies_mouth[:10]} — he has one.")
else:
    print(f"   [OK ] Protagonist's mouth never denied.")

FRAMING = "nothing cropped"
missing_frame = [s['scene_number'] for s in scenes if FRAMING not in s.get('image_prompt', '')]
if missing_frame:
    print(f"   [ERR] {len(missing_frame)} scenes missing the anti-crop phrase.")
    print(f"         First offenders: {missing_frame[:10]}")
else:
    print(f"   [OK ] Anti-crop phrase in all scenes.")

# --- Length sanity check ---
if total_words < 3100 or total_scenes < 235:
    print(f"\n[WARNING] Script is SHORT of the 25-minute target.")
    print(f"   Got {total_words} words / {total_scenes} scenes; target is 3,100-3,500 words / 235-260 scenes.")
    print(f"   At 130 wpm this is only ~{duration_min} min. Re-run, or ask Claude Code to expand")
    print(f"   the Work and Cost beats of Levels 4-7 before generating images.")
else:
    print(f"\n[OK] Length on target: {duration_min} min ({total_words} words, {total_scenes} scenes).")

if not MAIN_CHARACTER_REF:
    print(f"\n[REMINDER] MAIN_CHARACTER_REF is not set in .env.")
    print(f"   /generate_images will not be able to lock the protagonist's face without it.")

print(f"\nNext step: review {script_path}, then run /generate_images")
```

Save this script to a temp file and run it with the topic as argument.

**IMPORTANT — Windows PowerShell notes:**
- Do NOT use heredoc syntax (`<< 'EOF'`). Write the Python to a file using `create_file` tool, then run it.
- The script saves `current_script.json` as **UTF-8 without BOM** automatically — no manual BOM removal needed.
- If you get a `ModuleNotFoundError: anthropic`, run: `pip install anthropic --break-system-packages`

---

## STEP 1 — Required `.env` Keys

```
ANTHROPIC_API_KEY=sk-ant-...
MAIN_CHARACTER_REF=assets/protagonist_reference.png
```

`MAIN_CHARACTER_REF` points to a single reference image of the protagonist (bald, no nose, but with a mouth). `/generate_images` passes it to the image model on every `pov_scene` and `cost_scene` so the head shape, eye spacing, and proportions stay locked across all ~250 scenes. The age layer and costume in the prompt handle the aging; the reference image handles the identity.

---

## STEP 2 — Review Before Generating Images

The script's console output runs 8 automated checks. Read them. If any line shows `[ERR]` or `[OFF]`, fix the script before spending money on 250 images.

The two checks that matter most:

- **Protagonist fixed core present in all main scenes.** If this fails, the image model will invent a new face somewhere in the middle of the video and the illusion collapses.
- **Scene style distribution.** If `pov_scene` drifts above 75%, the video has no visual breathing room and retention drops in the Level 5–7 stretch.

---

## STEP 3 — Prompt the User

After the script is written, tell the user:

> Script written to `[video_folder]/current_script.json`.
>
> Review the `character_bank` block first — confirm the ten costumes escalate visibly, and that every named secondary character has a full face description.
>
> Then run: `/generate_images`
