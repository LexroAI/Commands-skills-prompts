# Hierarchy POV Claude Code Automation System — v38
## Complete Setup Guide for Windows & Mac

Build AI-generated **"Your Life at Every Rank in X"** second-person immersive documentary videos (Master POV style) with a pipeline that **stops at per-scene audio files** — you then assemble the final video manually in CapCut. This avoids the timestamp drift problem that occurs when you re-import a pre-rendered FFmpeg MP4 into CapCut and re-export it (preview looks fine, exported video has audio/subtitle desync in the second half).

Each video takes one closed hierarchical world (a submarine crew, a cartel, a monastery, an airline cockpit) and walks the viewer up its ten-rung ladder rank by rank. There is no host and no narrator character on screen — the camera is the viewer's eyes, and the word "you" carries the whole story. The protagonist you *see* is a single silent figure who ages across the ten levels; the voice you *hear* is a flat, unhurried second-person narration.

**What changed from v37 → v38:**
- v37 used FFmpeg to combine images + audio into a final MP4. Re-editing that MP4 in CapCut caused export desync due to accumulated VFR/sample-rate/timebase rounding errors.
- v38 stops the pipeline after generating one MP3 per scene + one combined SRT subtitle file. You import the raw audio + raw images directly into CapCut, where there is only one encode pass on export — no drift accumulates.
- FFmpeg is no longer required to assemble the video. `ffprobe` (which ships with ffmpeg) is still used purely as a read-only metadata tool to measure audio durations for the SRT file. If you install ffmpeg you get ffprobe automatically.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Why Not Text-to-Video?](#why-not-text-to-video)
3. [Tech Stack](#tech-stack)
4. [System Requirements](#system-requirements)
5. [Installation (Windows)](#installation-windows)
6. [Installation (Mac)](#installation-mac)
7. [API Keys Setup](#api-keys-setup)
8. [Creating Your Protagonist Reference](#creating-your-protagonist-reference)
9. [Project Folder Structure](#project-folder-structure)
10. [Claude Code Configuration](#claude-code-configuration)
11. [The 5 Command Files — Heart of the System](#the-5-command-files)
12. [Running Your First Video](#running-your-first-video)
13. [Handling Images You Don't Like](#handling-images-you-dont-like)
14. [Cost Breakdown](#cost-breakdown)
15. [Troubleshooting](#troubleshooting)

---

## System Overview

### What Does This System Do?

You type one command:
```
/generate_script_pov "Your Life as Every Rank in a Nuclear Submarine Crew"
```

The system automatically:
1. Writes a full **~25 minute** second-person POV script in the Master POV style — ten levels of one closed world, each rung costing the viewer something (freedom, identity, sleep, name, conscience)
2. Structures the ladder so every level escalates the trade, ending on "the cycle continues"
3. Splits the script into scenes (~**6 seconds each** — a POV video runs **~250 total scenes**, give or take)
4. Generates one cinematic image per scene with a consistent protagonist who **visibly ages 40–55 years** across the ten levels
5. Generates voiceover audio for each scene via ElevenLabs as **separate MP3 files**, one per scene
6. Generates a combined `.srt` subtitle file with timing pre-computed from the actual audio durations
7. **Stops there (default).** You assemble the final video manually in CapCut.
   — OR —
   Run `/generate_final_pov` to have FFmpeg assemble a ready-to-upload MP4 automatically (optional).

**Total time:** ~1–1.5 hours (mostly image generation on ~250 scenes; no FFmpeg encoding step in the default flow) plus your manual CapCut editing time
**Cost per video:** ~$8–9 (images via GPT Image 2 on Kie.ai — script writing via Anthropic API)

### Why Stop at Audio Instead of Producing a Final MP4?

When v37 produced a final MP4 via FFmpeg and you imported it into CapCut to add intro clips or B-roll, CapCut's export pass would re-encode that MP4. The combination of:
- FFmpeg's concat producing variable-frame-rate output
- ElevenLabs MP3 sample rates (44.1kHz or 22.05kHz) being resampled to CapCut's project rate (48kHz)
- MP4 container timestamp rounding accumulating across 100+ scene clips

...produces 100-200ms of drift by the end of the video. Preview hides it because CapCut re-times on the fly. Export shows it.

v38 sidesteps this entirely. Audio files and image files go into CapCut as raw assets. CapCut lays them out on its native timeline. There is only one encode pass — the final CapCut export — and no drift accumulates.

---

## Why Not Text-to-Video?

This is the most important design decision of the entire system.

**Text-to-video does NOT work for this use case because:**
- Each scene is generated independently — the character's face, hair, and outfit drift between scenes
- There is no reliable way to lock a custom character 100% across a fully automated pipeline
- Cost is significantly higher than text-to-image

**Text-to-image is the better approach because:**
- GPT Image 2 has built-in character consistency — the protagonist stays recognizable across ~250 image generations while still being aged deliberately, level by level, via a per-scene character description
- A hard cut from one held still to the next reads as deliberate, cinematic pacing for this format — the POV pipeline intentionally uses **no** Ken Burns zoom (a slow push across 250 detailed scenes reads as seasick drift, not motion)
- Roughly 60% cheaper than text-to-video tools, which matters far more at ~250 scenes per video

---

## Tech Stack

| Component | Tool | Role |
|---|---|---|
| AI Orchestrator | Claude Code | Writes scripts, creates prompts, calls APIs |
| Image Generation | GPT Image 2 (OpenAI) | One image per scene, protagonist consistent + aged per level |
| Voiceover | ElevenLabs | Second-person narration synced to each scene |
| Video Assembly | FFmpeg | Combines images + audio into static hard-cut clips (no zoom); optional in default flow |
| Script AI | Anthropic API (claude-opus-4-8) | Writes the full ten-level POV script via API call — produces high-quality JSON scripts |

---

## System Requirements

**All platforms need:**
- Claude Pro subscription ($20/month) — required for Claude Code
- Python 3.12 or higher
- Git
- FFmpeg
- Stable internet connection

**API accounts needed:**
- Kie.ai — for GPT Image 2 image generation ($0.03/image, pay-as-you-go)
- ElevenLabs — for text-to-speech voiceover
- Anthropic API — for script generation via claude-opus-4-8 (~$0.50–0.80 per POV script — these run long, up to 64k tokens streaming, pay-as-you-go)

> **Why Anthropic API instead of Claude Code built-in?**
> When Claude Code writes a script directly, it uses Python string generation — not the language model's creative writing ability. Calling the Anthropic API ensures the script is written by claude-opus-4-8 with full writing quality, exactly as if you had typed the prompt in claude.ai.

**Disk space:** ~4GB per video generation (~250 scenes; temporary files cleaned up after)

---

## Installation (Windows)

### Step 1: Install Python

1. Go to https://www.python.org/downloads/
2. Download **Python 3.12+** (latest stable version)
3. Run the installer
4. **IMPORTANT:** Check the box that says "Add Python to PATH"
5. Click "Install Now"
6. Verify installation:
   - Open Command Prompt (Win + R, type `cmd`)
   - Type: `python --version`
   - Should show: `Python 3.12.x` or higher

### Step 2: Install Git

1. Go to https://git-scm.com/download/win
2. Download Git for Windows
3. Run the installer, click Next through all options
4. Verify:
   - Open Command Prompt
   - Type: `git --version`

### Step 3: Install FFmpeg

1. Go to https://ffmpeg.org/download.html
2. Download the Windows build (static version recommended)
3. Extract the zip to a permanent location (e.g., `C:\ffmpeg`)
4. Add FFmpeg to PATH:
   - Right-click "This PC" → Properties
   - Click "Advanced system settings"
   - Click "Environment Variables"
   - Under "System variables," find "Path" → Edit → New
   - Add: `C:\ffmpeg\bin`
   - Click OK three times
5. Verify: open a new Command Prompt and type `ffmpeg -version`

### Step 4: Install Claude Desktop App

1. Go to https://claude.ai/download
2. Click **Windows**
3. Download and run the installer
4. Sign in with your Claude Pro account

---

## Installation (Mac)

### Step 1: Install Python

1. Go to https://www.python.org/downloads/
2. Download **Python 3.12+** (macOS installer)
3. Run the installer and follow prompts
4. Verify in Terminal: `python3 --version`

### Step 2: Install Homebrew, Git, and FFmpeg

Open Terminal and run these commands one by one:

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Git
brew install git

# Install FFmpeg
brew install ffmpeg
```

Verify:
```bash
git --version
ffmpeg -version
```

### Step 3: Install Claude Desktop App

1. Go to https://claude.ai/download
2. Click **Mac**
3. Download and drag Claude to your Applications folder
4. Open Claude and sign in with your Claude Pro account

---

## API Keys Setup

You need 3 API keys. Gather all of them before moving forward.

### Kie.ai API Key (GPT Image 2 Image Generation)

1. Go to https://kie.ai/
2. Sign in or create an account
3. Click **API Keys** in the left sidebar
4. Click **+ Create New Key**, name it anything
5. Copy the key
6. Save it — this is your `OPENAI_API_KEY` (Kie.ai uses OpenAI-compatible API format)

> **Pricing on Kie.ai for GPT Image 2:**
> - 1K resolution (1024px): **$0.03/image** — text-to-image
> - 1K resolution image-to-image: **$0.03/image** — used for main character scenes
>
> **Add credits:** Go to Billing in Kie.ai → add credits. Pay-as-you-go, no subscription required.

### ElevenLabs API Key + Voice ID

1. Go to https://elevenlabs.io/
2. Sign in to your existing account
3. Click your profile icon (top right) → **Profile**
4. Scroll to the **API Key** section → Copy your key
5. Go to https://elevenlabs.io/voice-lab
6. Pick a voice (Adam or Brian recommended for authority and trust)
7. Click the voice → Copy the **Voice ID**
8. Save both the API key and Voice ID

> ⚠️ **Never paste API keys into any AI chat interface.** Keys go only in the `.env` file on your local machine. Claude Code reads that file directly — you never need to share keys with anyone.

---

## Creating Your Protagonist Reference

This is the **only manual step** you do once and never repeat.

### What Is character_ref.png?

This is the reference image of your **protagonist** — the single silent figure the viewer inhabits in every POV video. GPT Image 2 uses it to keep him recognizable across all ~250 scenes of a video and across every video you produce.

Two things make the POV protagonist different from a normal channel mascot, and both matter for the reference image:

1. **His only visual signature is that he is completely bald.** He has a full face — a small rounded button nose, visible rounded ears, thick dark eyebrows, and a full expressive mouth — exactly like any other character, *except* he has no hair at all. Baldness is the single feature marking him as the viewer's avatar. (Because of this, the script generator gives every recurring secondary character hair — a bald secondary would read as the protagonist.)
2. **He ages 40–55 years across the ten levels.** The reference image should show him at a **neutral middle age (~40s)**. The pipeline ages him up and down at runtime by appending a per-level "age layer" to his description — you do not need separate reference images for young and old.

### Option A — You Already Have the Protagonist

If you've already created him, place the image here:

```
pov-automation/character/character_ref.png
```

Make sure the image is:
- **16:9 aspect ratio**
- Shows him clearly (full body or upper body), completely bald, with a full face
- Clean, white or neutral background
- High resolution (at least 1920×1080px)

Then upload the same image to ImgBB and paste its direct link as `PROTAGONIST_URL` in `channel_config.txt` (see below). Kie.ai reads the protagonist from that public URL, not from your local disk.

### Option B — You Don't Have the Protagonist Yet

Use the following prompt in ChatGPT (GPT-4o) or any image generation tool. The design language below matches the `protagonist_fixed_core` string the script generator locks into every scene, so keep it faithful:

```
Create a 2D cartoon character reference sheet for the silent protagonist of a
cinematic "POV documentary" YouTube channel.

OUTPUT FORMAT:
- 16:9 image (1920x1080 px)
- 9 frames arranged in a 3x3 grid
- Pure white background for every frame
- Each frame shows the SAME character from a DIFFERENT angle, distance, or expression
- No grid lines, no labels, no text

CHARACTER DESIGN (follow exactly — this is his fixed core):
- Large rounded oval head with warm light-tan skin
- COMPLETELY BALD, with absolutely NO hair anywhere — this is his defining feature
- A small rounded button nose, visible rounded ears, and thick dark eyebrows
- A full expressive mouth
- Two small solid black oval eyes set wide apart, no visible whites
- Short thick neck; head roughly 35% of body height; average adult build
- Appears to be a neutral middle age (around 40)
- Plain, unbranded neutral clothing (a simple grey work shirt and dark trousers) —
  costume changes per video, so keep it generic here

FRAME BREAKDOWN (9 frames — vary angles and expressions):
Frame 1 (top-left):     Close-up face, front facing, neutral empty expression
Frame 2 (top-center):   Close-up face, front facing, exhausted (flat slack mouth)
Frame 3 (top-right):    Close-up face, front facing, quiet grief (mouth turned down)
Frame 4 (mid-left):     Upper body, 3/4 angle left, determined (firm tight line mouth)
Frame 5 (mid-center):   Upper body, front facing, standing still, hands at sides
Frame 6 (mid-right):    Upper body, 3/4 angle right, fear/shock (mouth open in an O)
Frame 7 (bot-left):     Full body, front facing, standing at attention
Frame 8 (bot-center):   Full body, side profile right, walking
Frame 9 (bot-right):    Full body, slight from behind, looking back over shoulder

WHY THESE ANGLES MATTER:
These 9 frames give the AI a complete reference from all directions — front, side,
back, close-up, full body — so it can generate consistent scenes at any angle. The
range of expressions matters because this character never smiles by default; his
face carries cost, not cheer.

ART STYLE:
- Clean 2D cartoon illustration, bold thick black outlines
- Flat color fills with soft cel shading
- Muted naturalistic palette (dusty blues, warm browns, sage greens, cream, charcoal)
- Cinematic, grounded — NOT Family Guy, Simpsons, South Park, or any existing TV show
- No text, no background elements, white only
```

> **Important:** Do not give him hair, and do not remove his nose/ears/eyebrows. Bald + full face is the exact combination the whole pipeline depends on. A character with hair, or a "no-nose" character, will break consistency against the locked `protagonist_fixed_core` string.

After generating, download the image, name it `character_ref.png`, place it in `pov-automation/character/`, then upload it to ImgBB and put the direct link in `channel_config.txt` as `PROTAGONIST_URL`.

**This file is the foundation of your entire channel. Keep it. Never replace it.**

---

## Project Folder Structure

Create a folder called `pov-automation` on your Desktop (or wherever you prefer).

**Step 1 — Create these folders and files manually:**

| What | How |
|---|---|
| `pov-automation/` | Right-click Desktop → New → Folder |
| `character/` | Right-click inside `pov-automation/` → New → Folder |
| `.claude/` | Right-click inside `pov-automation/` → New → Folder |
| `.claude/commands/` | Right-click inside `.claude/` → New → Folder |
| `videos/` | Right-click inside `pov-automation/` → New → Folder |
| `.env` | Notepad → Save As → name: `.env` → type: **All Files** |
| `channel_config.txt` | Notepad → Save As normally |
| `generate_ideas_pov.md` | Notepad → Save As → name: `generate_ideas_pov.md` → type: **All Files** |
| `generate_script_pov.md` | Notepad → Save As → name: `generate_script_pov.md` → type: **All Files** |
| `generate_images_pov.md` | Notepad → Save As → name: `generate_images_pov.md` → type: **All Files** |
| `generate_audio_pov.md` | Notepad → Save As → name: `generate_audio_pov.md` → type: **All Files** |
| `generate_final_pov.md` | Notepad → Save As → name: `generate_final_pov.md` → type: **All Files** (optional automated MP4 flow) |
| `character_ref.png` | Copy your protagonist image into `character/` |

**Step 2 — Everything else is created automatically by Claude Code:**

| What | Created when |
|---|---|
| `videos/images/pending/` | `/generate_images_pov` runs |
| `videos/images/approved/` | `/generate_images_pov` runs |
| `videos/images/rejected.txt` | You reject an image |
| `videos/current_script.json` | `/generate_script_pov` runs |
| `videos/ideas_history.txt` | `/generate_script_pov` runs |
| `videos/worlds_covered.txt` | You maintain it (optional — lists broad world *categories* already used, so `/generate_ideas_pov` diversifies) |
| `videos/audio/` | `/generate_audio_pov` runs |
| `videos/final_output/` | `/generate_audio_pov` runs |
| `videos/final_output/subtitles.srt` | `/generate_audio_pov` completes |

**Final folder structure for reference:**

```
pov-automation/                     ← YOU CREATE
├── .env                                ← YOU CREATE
├── channel_config.txt                  ← YOU CREATE
├── character/                          ← YOU CREATE
│   └── character_ref.png               ← YOU ADD
├── .claude/                            ← YOU CREATE
│   └── commands/                       ← YOU CREATE
│       ├── generate_ideas_pov.md           ← YOU CREATE
│       ├── generate_script_pov.md          ← YOU CREATE
│       ├── generate_images_pov.md          ← YOU CREATE
│       ├── generate_audio_pov.md           ← YOU CREATE  (v38 default — CapCut manual assembly)
│       └── generate_final_pov.md           ← YOU CREATE  (v38 optional — automated FFmpeg MP4 output)
└── videos/                             ← YOU CREATE
    ├── ideas_history.txt               ← AUTO
    ├── worlds_covered.txt              ← YOU MAINTAIN (optional category log)
    ├── current_video_folder.txt        ← AUTO (tracks active video)
    └── 2026-07-11_submarine-crew/      ← AUTO (one folder per video)
        ├── current_script.json         ← AUTO → KEPT (needed for CapCut metadata)
        ├── images/
        │   ├── pending/                ← AUTO → deleted after /generate_audio_pov (optional)
        │   ├── approved/               ← AUTO → KEPT (you import these into CapCut)
        │   └── rejected.txt            ← AUTO
        ├── audio/                      ← AUTO → KEPT (you import these into CapCut)
        └── final_output/               ← AUTO
            └── subtitles.srt           ← AUTO → KEPT (you import this into CapCut)
```

> **How `ideas_history.txt` works:** Every time you run `/generate_script_pov`, the system appends a line to this file in the format `YYYY-MM-DD | topic title | thumbnail_line`. When you run `/generate_ideas_pov`, the system reads this file first and avoids generating topics OR thumbnail verdict words that repeat ones you've already used (a submarine crew already covered means an aircraft-carrier crew is rejected as a near-neighbor). Claude has no memory between sessions — this file is the memory.
>
> **Optional `worlds_covered.txt`:** a plain list of broad world *categories* you've already used (military, crime, medicine, monastic, aviation, etc.), one per line. `/generate_ideas_pov` reads it to keep each batch category-diverse instead of, say, five military topics in a row.

### Create the .env File

Create a file named `.env` inside `pov-automation/` and paste:

```
OPENAI_API_KEY=your_kie_ai_key_here
ELEVEN_LABS_API_KEY=your_elevenlabs_key_here
ELEVEN_LABS_VOICE_ID=your_voice_id_here
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

> **Note:** `OPENAI_API_KEY` here is your **Kie.ai API key** — Kie.ai uses OpenAI-compatible API format, so the variable name is the same.

> **Note:** `ANTHROPIC_API_KEY` is your key from the Anthropic API Console. This is separate from your Claude Pro subscription and has its own billing (~$0.50–0.80 per POV script — these run long, up to 64k tokens streaming).

Replace each placeholder with your actual keys.

### Create the channel_config.txt File

Create a file named `channel_config.txt` inside `pov-automation/` and fill in your details:

```
# Your channel identity — fill this in once, never change it
CHANNEL_NAME=Your Channel Name Here

# Protagonist — the silent bald figure the viewer inhabits.
# Upload your character_ref.png to https://imgbb.com/ then paste the Direct Link here
# (ends in .png or .jpg). This URL keeps the protagonist consistent across every scene.
PROTAGONIST_URL=https://i.ibb.co/xxxxx/your-protagonist.png

# Brand Colors — enter your hex codes (used for thumbnail text styling)
BRAND_COLOR_PRIMARY=#FFFFFF
BRAND_COLOR_SECONDARY=#1A1A1A
BRAND_COLOR_ACCENT=#E63946
```

> **Note:** This channel has no on-screen host or named narrator character, so there is no `NARRATOR_NAME`. The narration is anonymous second-person ("you"). The only recurring visual identity is the protagonist, set by `PROTAGONIST_URL`.
>
> **How to get your PROTAGONIST_URL:**
> 1. Go to https://imgbb.com/
> 2. Upload your `character_ref.png` (bald protagonist, white background, PNG)
> 3. After uploading, click the image → copy the **Direct Link** (ends in `.png`)
> 4. Paste that URL as `PROTAGONIST_URL` above
>
> The image script also accepts the legacy `CHARACTER_URL` key as a fallback if you reused a finance config — but `PROTAGONIST_URL` is the correct key for this channel.
>
> **Brand colors:** PRIMARY is background, SECONDARY is text/lines, ACCENT is highlights and thumbnail text.

---

## API Keys Setup

### Kie.ai (Image Generation)
1. Go to https://kie.ai/
2. Create an account
3. Go to API section → copy your API key
4. Paste as `OPENAI_API_KEY` in `.env`

### ElevenLabs (Voiceover)
1. Go to https://elevenlabs.io/
2. Create an account (free tier available)
3. Go to Profile → API Keys → copy your key
4. Paste as `ELEVEN_LABS_API_KEY` in `.env`
5. Go to Voices → pick your voice → copy the Voice ID
6. Paste as `ELEVEN_LABS_VOICE_ID` in `.env`

### Anthropic API (Script Generation)
1. Go to https://console.anthropic.com/
2. Sign in (create a new account — this is separate from your Claude Pro account, but you can use the same email)
3. Click **API Keys** in the left sidebar
4. Click **Create Key** → give it a name (e.g. "pov-automation") → copy the key
5. Go to **Billing** → add a payment method → load at least **$5 credit** to start
6. Paste your key as `ANTHROPIC_API_KEY` in `.env`

> **Cost:** Each `/generate_script_pov` call uses ~8,000–12,000 tokens total. At claude-opus-4-8 pricing (~$3 input / $15 output per million tokens), one script costs approximately **$0.10–0.15**. For 10 scripts/month that's ~$1–1.50 — effectively free.

> **Anthropic Python library** is auto-installed on first run — no manual pip install needed.
> The script checks for the package and installs it automatically if missing. After the first run it's cached permanently.

---

## Claude Code Configuration

### Windows & Mac (Same Process)

1. Open the Claude Desktop app
2. Click the **Code** button at the top
3. At the bottom, click the **folder icon**
4. Navigate to your `pov-automation` folder
5. Click **Select Folder**
6. In the bottom left, click the **Settings** gear icon
7. Go to **Claude Code** in the left sidebar
8. Turn on **Allow bypass permissions mode**
9. You're ready

---

## The 5 Command Files

Create these files inside `.claude/commands/`. These are the engine of the entire system.

> **Two final-stage workflows available — pick one:**
>
> **Option A — Manual (recommended, no drift):** Run `/generate_audio_pov` → import raw MP3s + PNGs into CapCut → assemble manually. Clean one-pass export. Full creative control.
>
> **Option B — Automated (optional):** Run `/generate_final_pov` → FFmpeg assembles everything into a ready-to-upload MP4. This is a **static hard-cut** edit — each scene is a still held for the length of its audio, no Ken Burns, no zoom, no burned-in subtitles (the SRT stays a separate YouTube caption track). Because every clip shares identical encode parameters, the concat step stream-copies instead of re-encoding, making it dramatically faster across ~250 scenes. Re-importing this MP4 into CapCut for further editing will still cause drift on export — only use it when you want a finished video with no manual editing.

---

### File 1: generate_ideas_pov.md

```markdown
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

"Pick a topic and run: /generate_script_pov \"your chosen title here\"

If you want a different angle on the same world, run: /generate_ideas_pov --expand \"world name\""
```

---

### File 2: generate_script_pov.md

> **Important:** The `generate_script_pov.md` command file calls the **Anthropic API** instead of having Claude Code write the script directly. This produces significantly better scripts because the writing is done by claude-opus-4-8 with full creative writing instructions, streamed and parsed back into JSON — not by Python string generation. A POV script runs long (a full ten-level, ~3,300-word, ~250-scene script), which is why streaming with a high token ceiling matters here.
>
> Key behaviors:
> 1. Reads `ANTHROPIC_API_KEY` from `.env` (auto-installs the `anthropic` Python package on first run if missing)
> 2. Sends the full Master POV writing system prompt to the Anthropic API using **streaming** (`max_tokens=64000`) so long scripts are never cut off mid-JSON
> 3. Receives the complete JSON script back and parses it — including the `character_bank` block (protagonist fixed core, ten age layers, ten costumes, and locked secondary-character appearances)
> 4. Saves it as `current_script.json` — UTF-8 without BOM (no manual BOM removal needed)
> 5. Creates the video folder structure, updates `current_video_folder.txt`, and appends `YYYY-MM-DD | title | thumbnail_line` to `ideas_history.txt`

```markdown
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
"the protagonist (large rounded oval head with warm light-tan skin, completely bald with NO hair; a small rounded button nose, visible rounded ears, thick dark eyebrows, and a full expressive mouth; two small solid black oval eyes set wide apart, no visible whites; short thick neck; head roughly 35% of body height; average adult build)"

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

CRITICAL DISTINCTION: the protagonist has a FULL face — nose, ears, eyebrows, mouth, eyes — exactly like a secondary character, EXCEPT that he is completely bald. Baldness (no hair) is his single visual signature and the only thing marking him as the viewer's avatar. Because that one feature carries the whole distinction, every recurring secondary character MUST be given hair — a bald secondary character will read as the protagonist. Never give the protagonist hair, and never make a recurring named secondary character bald.

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

The protagonist has a full face — a small rounded nose, ears, thick eyebrows, and a mouth. The ONLY thing separating him from a secondary character is that he is completely bald. Keep him hairless in every scene, and give every recurring secondary character hair.

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

NEVER give the protagonist hair. That is the HARD FAILURE. (A nose, ears, and eyebrows are correct — he has all three.)

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

16. BALDNESS CHECK: the protagonist is completely bald in every single scene, but DOES have a nose, ears, eyebrows, and a mouth. Every recurring secondary character has hair. The contrast is baldness alone. Scan for any image_prompt that gives the protagonist hair, or that makes a recurring secondary character bald — either is a HARD FAILURE.

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
    "protagonist_fixed_core": "the protagonist (large rounded oval head with warm light-tan skin, completely bald with NO hair; a small rounded button nose, visible rounded ears, thick dark eyebrows, and a full expressive mouth; two small solid black oval eyes set wide apart, no visible whites; short thick neck; head roughly 35% of body height; average adult build)",
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

FIXED_CORE = "completely bald with NO hair"
main_scenes = [s for s in scenes if s.get('character') == 'main']
missing_core = [s['scene_number'] for s in main_scenes if FIXED_CORE not in s.get('image_prompt', '')]
if missing_core:
    print(f"   [ERR] {len(missing_core)} main-character scenes missing the protagonist fixed core string.")
    print(f"         First offenders: {missing_core[:10]}")
else:
    print(f"   [OK ] Protagonist fixed core present in all {len(main_scenes)} main scenes.")

# Hair check. Baldness is the protagonist's ONLY signature, so the single additive
# failure to hunt for is HAIR. The fixed_core legitimately says "completely bald
# with NO hair" and also names a nose, ears, and eyebrows (all correct now), so
# strip the core and the standard "NO hair" negation before scanning — otherwise
# every correct scene reports a false positive. A nose is no longer a failure.
bad_hair = []
for s in main_scenes:
    ip = s.get('image_prompt', '')
    # cost_scene prompts legitimately describe a SECONDARY character (who HAS hair)
    # after the "WITH:" marker. Scan only the protagonist's own segment, otherwise
    # every cost_scene reports a false positive.
    segment = re.split(r'\bWITH:', ip, maxsplit=1, flags=re.I)[0]
    # remove the locked core and the standard hair negation before scanning
    residue = segment.replace(FIXED_CORE, '')
    residue = re.sub(r'NO hair[^,;.]*', '', residue, flags=re.I)
    low = residue.lower()
    if re.search(r'\b(hair|haircut|bangs|ponytail|balding|hairline|combover)\b', low):
        bad_hair.append(s['scene_number'])
if bad_hair:
    print(f"   [ERR] Protagonist given hair in {len(bad_hair)} scenes: {bad_hair[:8]}")
else:
    print(f"   [OK ] Protagonist is bald (no hair) in every scene.")

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
    print(f"   /generate_images_pov will not be able to lock the protagonist's face without it.")

print(f"\nNext step: review {script_path}, then run /generate_images_pov")
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

`MAIN_CHARACTER_REF` points to a single reference image of the protagonist (completely bald, with a small rounded nose, ears, thick eyebrows, and a mouth). `/generate_images_pov` passes it to the image model on every `pov_scene` and `cost_scene` so the head shape, nose shape, eye spacing, and proportions stay locked across all ~250 scenes. The age layer and costume in the prompt handle the aging; the reference image handles the identity.

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
> Then run: `/generate_images_pov`
```

### File 3: generate_images_pov.md

```markdown
CRITICAL: Use EXACTLY the Python code in STEP 2.
The URL parsing logic below reads the image URL from `resultJson` (a JSON-encoded
string inside the poll response), with a fallback to a top-level `resultUrls`
field. This was fixed on 2026-06-18 — Kie.ai's actual response nests the URL
inside `resultJson`, not `resultUrls` directly; the previous version of this
script read the wrong field and silently failed every scene (Kie.ai reported
`success` and consumed credits, but no image was ever downloaded). Do not
revert this parsing logic back to reading only `resultUrls`.

**IMPORTANT — Execution method:** Run this Python script using the Bash tool with `run_in_background=true`. Do NOT use PowerShell's `Start-Process` to launch it. `run_in_background=true` triggers an automatic notification the moment the script finishes — without it, the script can complete successfully in the background with zero indication, and Claude Code will sit idle waiting for the user to ask "is it done?" instead of reporting results immediately.

**Windows execution note:** `run_in_background=true` via the Bash tool is preferred because it auto-notifies when the script finishes. If Bash fails due to Windows path parsing and you fall back to PowerShell's `Start-Process`, there is no automatic notification on completion. In that case, do not wait silently. After launching, periodically read the script's log/output file (e.g. every 60-90 seconds) to check progress, and proactively report completion to the user as soon as the final summary line appears in the output — never wait for the user to ask "is it done?"

**SCALE WARNING — read before running.** A POV video is ~250 scenes, not ~120. At roughly 40 seconds per scene, sequential generation would take about **2.8 hours**. This script therefore generates scenes **in parallel with a bounded worker pool** (default 3 concurrent scenes, ~55 minutes). Raise `MAX_CONCURRENT_SCENES` only if Kie.ai is not rate-limiting you; lower it to 1 to reproduce the old sequential behavior.

# Generate Images for Current POV Script

## STEP 0 — What Is Different About POV

Three things break if you reuse the finance `/generate_images_pov` script unchanged:

**1. There is a third `character` value.** Finance scripts only ever emit `"main"` or `"supporting"`. POV scripts also emit `"none"` for `object_scene` — a close-up of a folded flag, a medal in a shadow box, an empty locker. These scenes contain **no people at all**. If a character reference image is attached to them, the model will hallucinate a person into the frame. Object scenes must go through `text-to-image` with **no `image_input`**, exactly like supporting scenes, and the script below routes them that way.

**2. The protagonist's appearance is not constant.** In the finance pipeline, Jeff looks identical in every scene, so one global `CHARACTER_DESCRIPTION` string works. In POV the protagonist ages 40–55 years across ten levels. The script must therefore assemble the character description **per scene**, from three parts stored in the script JSON's `character_bank` block:

```
fixed_core  +  age_layer[scene.level]  +  costume[scene.level]
```

The `fixed_core` never changes and is what keeps the character recognizable. The age layer and costume are what make him age. This assembly happens at runtime, below.

**3. The reference image anchors a head shape, not a whole person.** The protagonist has a **full face** — a small rounded nose, visible ears, thick eyebrows, and a mouth — but he is **completely bald**. Baldness (no hair) is the *only* thing separating him from a secondary character, so keep the reference head clean and hairless and make sure every recurring secondary character has hair. `MAIN_CHARACTER_REF` should be a clean, front-facing, neutral image of exactly that head on a plain body. It anchors head shape, nose shape, eye spacing, and proportion. It does **not** anchor age or clothing — those come from the text.

## STEP 1 — Read Channel Config and Active Video Folder

Read `channel_config.txt` and extract:
- `PROTAGONIST_URL` — the ImgBB **public URL** of the protagonist reference image (completely bald, with a small rounded nose, ears, thick eyebrows, and a mouth)

> Note: `.env` may also hold `MAIN_CHARACTER_REF` as a **local file path** for your own reference. Kie.ai cannot read local paths. Upload that image to ImgBB once, and put the resulting public URL in `channel_config.txt` as `PROTAGONIST_URL`. The script below reads `PROTAGONIST_URL` and will fall back to `CHARACTER_URL` if you reused the finance config key.

Read `videos/current_video_folder.txt` to get the active video folder path.

Read `[video_folder]/current_script.json` and get the full scene list **and the `character_bank` block**.

Check if `[video_folder]/images/rejected.txt` exists:
- If YES: only regenerate the scene numbers listed in that file
- If NO: generate all scenes that don't already have an image in `[video_folder]/images/approved/` or `[video_folder]/images/pending/`

## STEP 2 — Generate Images with GPT Image 2 via Kie.ai

For each scene that needs an image, call the Kie.ai API.
All images must be saved as **PNG** — PNG is lossless and produces sharper colors for flat 2D cartoon style.

**Character consistency strategy — three routes:**

| `scene.character` | scene styles | Route | Reference image | Text anchor |
|---|---|---|---|---|
| `"main"` | `pov_scene`, `cost_scene` | image-to-image | `PROTAGONIST_URL` | assembled per-level string |
| `"supporting"` | `world_scene` | text-to-image | none | full locked string already in prompt |
| `"none"` | `object_scene` | text-to-image | none | none — no people in frame |

- For `"main"` → pass `PROTAGONIST_URL` via `image_input` AND append the assembled protagonist description. Two anchors beat one.
- For `"supporting"` → **never** pass `PROTAGONIST_URL`. That URL is the protagonist's bald head; attaching it to a `world_scene` will make a secondary character go bald (or wear the protagonist's exact face) when they are supposed to have their own hair and features. The prompt already carries each recurring character's full locked appearance string from the CHARACTER BANK — do not strip or shorten it. For recurring secondary characters the text description IS the only consistency anchor. "Generate freely" applies only to one-off background figures.
- For `"none"` → no reference, no character text. The prompt describes an object and a surface. Any person appearing in an object scene is a failure.

```python
import requests, os, json, sys, time, threading
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FutureTimeoutError
sys.stdout.reconfigure(line_buffering=True, encoding='utf-8')

# ---- Tuning -------------------------------------------------------------
# 250 scenes sequentially is ~2.8 hours. 3 concurrent scenes brings that to
# ~55 minutes. Raise only if Kie.ai is not rate-limiting; set to 1 to get the
# old strictly-sequential behavior back.
MAX_CONCURRENT_SCENES = 3
# -------------------------------------------------------------------------

# Load .env file directly — do not rely on os.getenv
env = {}
try:
    with open('.env', 'r', encoding='utf-8-sig') as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
except:
    pass

KIE_API_KEY = env.get('OPENAI_API_KEY') or env.get('KIE_AI_API_KEY') or os.getenv('OPENAI_API_KEY') or os.getenv('KIE_AI_API_KEY')

# Read channel config
with open('channel_config.txt', 'r', encoding='utf-8-sig') as f:
    config = {}
    for line in f:
        line = line.strip()
        if line and not line.startswith('#') and '=' in line:
            key, value = line.split('=', 1)
            config[key.strip()] = value.strip()

# PROTAGONIST_URL is the POV key; CHARACTER_URL is the legacy finance key.
PROTAGONIST_URL = config.get('PROTAGONIST_URL', '') or config.get('CHARACTER_URL', '')
if not PROTAGONIST_URL:
    print("[WARNING] No PROTAGONIST_URL in channel_config.txt.", flush=True)
    print("          Main-character scenes will fall back to text-to-image.", flush=True)
    print("          The protagonist's head shape will drift across scenes.", flush=True)

with open('videos/current_video_folder.txt', 'r', encoding='utf-8-sig') as f:
    video_folder = f.read().strip()

with open(f'{video_folder}/current_script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)

os.makedirs(f'{video_folder}/images/pending', exist_ok=True)
os.makedirs(f'{video_folder}/images/approved', exist_ok=True)

# ---- Assemble the protagonist description per level ---------------------
# The protagonist ages across ten levels. A single global description string
# (as used in the finance pipeline for Jeff) would freeze him at one age.
# Instead we rebuild the description for each scene from the script's own
# character_bank: fixed_core + age_layer[level] + costume[level].
bank = script.get('character_bank', {})
FIXED_CORE = bank.get('protagonist_fixed_core', '')
AGE_LAYERS = bank.get('protagonist_age_layers', {})
COSTUMES = bank.get('protagonist_costumes', {})

if not FIXED_CORE:
    print("[ERROR] character_bank.protagonist_fixed_core is missing from current_script.json.", flush=True)
    print("        Regenerate the script with /generate_script_pov before generating images.", flush=True)
    sys.exit(1)

# age_layers are keyed by span: "level_1_2", "level_3_4", "level_5_6",
# "level_7_8", "level_9", "level_10". Map each level number to its key.
def age_layer_key(level):
    if level in (1, 2):   return 'level_1_2'
    if level in (3, 4):   return 'level_3_4'
    if level in (5, 6):   return 'level_5_6'
    if level in (7, 8):   return 'level_7_8'
    if level == 9:        return 'level_9'
    return 'level_10'

_warned_levels = set()
_warn_lock = threading.Lock()

def protagonist_description(level):
    """Rebuild the full protagonist string for this scene's level."""
    age = AGE_LAYERS.get(age_layer_key(level), '')
    costume = COSTUMES.get(f'level_{level}', '')
    with _warn_lock:
        if (not age or not costume) and level not in _warned_levels:
            _warned_levels.add(level)
            missing = []
            if not age: missing.append('age_layer')
            if not costume: missing.append('costume')
            print(f"  [WARN] Level {level}: missing {', '.join(missing)} in character_bank.", flush=True)
    parts = [p for p in (FIXED_CORE, age, costume) if p]
    body = ' '.join(parts)
    return (
        "\n\nMAIN CHARACTER — must appear EXACTLY as described:\n"
        f"- {body}\n"
        "- The protagonist is completely bald (NO hair). Baldness is his ONLY signature — he otherwise has a full face.\n"
        "- He HAS a small rounded nose, visible ears, thick dark eyebrows, and a full expressive mouth.\n"
        "- Skin is a warm light tan. The head is large and rounded on a short thick neck.\n"
        "- Eyes are two small solid black ovals set wide apart, with no visible whites.\n"
        "- Expression comes from the mouth shape, the eyebrow line, and the eye shape together. He may smile, but only on genuinely positive beats.\n"
        "- Bold clean outlines, flat 2D cartoon style, soft cel shading, muted naturalistic palette.\n"
        "- Same character across all scenes — only age, costume, pose, and mouth/eyebrow/eye expression change.\n"
    )

headers = {
    "Authorization": f"Bearer {KIE_API_KEY}",
    "Content-Type": "application/json"
}

# Track results
failed_scenes = []
success_count = 0
_stats_lock = threading.Lock()
total_scenes = script.get('total_scenes') or len(script['scenes'])

# --- HARD TIMEOUT WRAPPER ---
# requests' own timeout= parameter only covers connect/read at the HTTP layer.
# If the TCP socket itself hangs (network drop, proxy stall, DNS stuck), that
# timeout never fires and the call blocks forever. Wrapping every network call
# in its own thread with a hard deadline guarantees the call is abandoned no
# matter where it gets stuck — the thread is left to die in the background,
# and execution moves on immediately.
#
# This pool sizing matters: each in-flight scene may hold one timeout thread,
# so the pool must be at least as large as MAX_CONCURRENT_SCENES or scenes will
# deadlock waiting for a free timeout worker.
_executor = ThreadPoolExecutor(max_workers=max(4, MAX_CONCURRENT_SCENES * 2))

def call_with_hard_timeout(func, timeout_seconds, *args, **kwargs):
    """Run func in a separate thread. If it doesn't return within
    timeout_seconds, give up and move on — the underlying thread is
    abandoned (daemon-like) rather than waited on."""
    future = _executor.submit(func, *args, **kwargs)
    try:
        return future.result(timeout=timeout_seconds)
    except FutureTimeoutError:
        return None
    except Exception as e:
        print(f"  (call error: {e})", flush=True)
        return None

def build_payload(scene):
    prompt = scene['image_prompt']
    role = scene.get('character', 'supporting')
    level = scene.get('level', 1)

    # ROUTE 1 — main: protagonist appears. Reference image + assembled text.
    if role == 'main' and PROTAGONIST_URL:
        return {
            "model": "gpt-image-2-image-to-image",
            "callBackUrl": "",
            "input": {
                "prompt": f"{prompt}{protagonist_description(level)}",
                "image_input": [PROTAGONIST_URL],
                "aspect_ratio": "16:9",
                "resolution": "1K",
                "output_format": "png"
            }
        }

    # ROUTE 1b — main but no reference URL configured. Text-only fallback.
    if role == 'main':
        return {
            "model": "gpt-image-2-text-to-image",
            "callBackUrl": "",
            "input": {
                "prompt": f"{prompt}{protagonist_description(level)}",
                "image_input": [],
                "aspect_ratio": "16:9",
                "resolution": "1K",
                "output_format": "png"
            }
        }

    # ROUTE 2 — supporting (world_scene): full faces, NO protagonist reference.
    # ROUTE 3 — none (object_scene): no people at all, no character text.
    # Both are plain text-to-image. The prompt already carries everything needed.
    return {
        "model": "gpt-image-2-text-to-image",
        "callBackUrl": "",
        "input": {
            "prompt": prompt,
            "image_input": [],
            "aspect_ratio": "16:9",
            "resolution": "1K",
            "output_format": "png"
        }
    }

def _do_create_task(payload):
    response = requests.post(
        "https://api.kie.ai/api/v1/jobs/createTask",
        json=payload,
        headers=headers,
        timeout=20
    )
    return response.json()

def _do_poll(task_id):
    poll = requests.get(
        f"https://api.kie.ai/api/v1/jobs/recordInfo?taskId={task_id}",
        headers=headers,
        timeout=15
    )
    return poll.json()

def _do_download(img_url):
    return requests.get(img_url, timeout=20).content

def poll_and_save(n, task_id):
    """Poll Kie.ai for result. Each individual poll call is wrapped in a
    hard 20s thread timeout — if a single poll hangs at the TCP layer, it
    is abandoned and the loop moves to the next attempt instead of freezing.
    Overall budget: 40 attempts x 4s sleep = ~2m40s before giving up on this
    scene entirely."""
    POLL_INTERVAL = 4
    MAX_POLLS = 40
    PROGRESS_EVERY = 8

    for attempt in range(MAX_POLLS):
        time.sleep(POLL_INTERVAL)

        if attempt > 0 and attempt % PROGRESS_EVERY == 0:
            elapsed = attempt * POLL_INTERVAL
            print(f"  Scene {n}: still waiting... ({elapsed}s)", flush=True)

        result = call_with_hard_timeout(_do_poll, 20, task_id)
        if result is None:
            print(f"  Scene {n}: poll hung or errored, skipping this attempt...", flush=True)
            continue

        data = result.get('data', {})
        state = data.get('state', '')

        if state == 'success':
            # Kie.ai returns the image URL(s) inside `resultJson`, a JSON-encoded
            # string like {"resultUrls": ["https://..."]} — NOT in a top-level
            # `resultUrls` field. Parse resultJson first; fall back to a
            # top-level resultUrls field only in case the API response shape
            # ever changes, so a single bad field name can't silently fail
            # every scene again.
            img_url = None
            try:
                rj = data.get('resultJson')
                if rj:
                    if isinstance(rj, str):
                        rj = json.loads(rj)
                    if isinstance(rj, dict):
                        urls = rj.get('resultUrls') or rj.get('urls')
                        if urls:
                            img_url = urls[0]
                    elif isinstance(rj, list) and rj:
                        img_url = rj[0]
            except Exception as e:
                print(f"  Scene {n}: failed to parse resultJson: {e}", flush=True)

            if not img_url:
                try:
                    legacy = data.get('resultUrls')
                    if isinstance(legacy, str):
                        legacy = json.loads(legacy)
                    if legacy:
                        img_url = legacy[0]
                except Exception:
                    pass

            if not img_url:
                print(f"  Scene {n}: success but no image URL found in response, skipping. (raw resultJson={data.get('resultJson')!r})", flush=True)
                return False

            img_data = call_with_hard_timeout(_do_download, 25, img_url)
            if img_data is None:
                print(f"  Scene {n}: image download hung, will retry scene.", flush=True)
                return False

            filepath = f"{video_folder}/images/pending/scene_{n:03d}.png"
            with open(filepath, 'wb') as f:
                f.write(img_data)
            print(f"[OK] Scene {n} saved.", flush=True)
            return True

        elif state == 'failed':
            print(f"[FAIL] Scene {n}: Kie.ai returned failed.", flush=True)
            return False

    print(f"[TIMEOUT] Scene {n}: timed out after {MAX_POLLS * POLL_INTERVAL}s.", flush=True)
    return False

def generate_image(scene):
    n = scene['scene_number']
    payload = build_payload(scene)

    # Create task — hard timeout 20s
    result = call_with_hard_timeout(_do_create_task, 20, payload)
    if result is None:
        print(f"[FAIL] Scene {n}: task creation hung or errored.", flush=True)
        return False

    if result.get('code') != 200:
        print(f"[FAIL] Scene {n}: task creation failed: {result}", flush=True)
        return False

    task_id = result['data']['taskId']
    role = scene.get('character', '?')
    style = scene.get('scene_style', '?')
    print(f"Scene {n}/{total_scenes} [L{scene.get('level','?')} {style}/{role}]: task created, polling...", flush=True)

    success = poll_and_save(n, task_id)
    if success:
        return True

    # Auto-retry once on fail/timeout
    print(f"[RETRY] Scene {n}: retrying once...", flush=True)
    result2 = call_with_hard_timeout(_do_create_task, 20, payload)
    if result2 is not None and result2.get('code') == 200:
        task_id2 = result2['data']['taskId']
        success2 = poll_and_save(n, task_id2)
        if success2:
            return True
    else:
        print(f"[FAIL] Scene {n}: retry task creation hung or errored.", flush=True)

    return False

def process_scene(scene):
    """Skip-if-exists, then generate. Thread-safe stat updates."""
    global success_count
    n = scene['scene_number']

    # Skip if already saved to approved/ (user has reviewed it) OR pending/ (downloaded but not yet reviewed).
    # Without checking pending/, every restart goes back to scene 1 even though images are already on disk —
    # causing unnecessary API calls, credit waste, and the script never getting past the stuck scene.
    if os.path.exists(f"{video_folder}/images/approved/scene_{n:03d}.png"):
        print(f"[SKIP] Scene {n}: already in approved/", flush=True)
        with _stats_lock:
            success_count += 1
        return
    if os.path.exists(f"{video_folder}/images/pending/scene_{n:03d}.png"):
        print(f"[SKIP] Scene {n}: already in pending/", flush=True)
        with _stats_lock:
            success_count += 1
        return

    ok = generate_image(scene)
    with _stats_lock:
        if ok:
            success_count += 1
        else:
            failed_scenes.append(n)

# --- Optional: restrict to rejected scenes only --------------------------
scenes_to_do = script['scenes']
rejected_path = f"{video_folder}/images/rejected.txt"
if os.path.exists(rejected_path):
    with open(rejected_path, 'r', encoding='utf-8') as f:
        retry_nums = {int(line.strip()) for line in f if line.strip().isdigit()}
    if retry_nums:
        scenes_to_do = [s for s in script['scenes'] if s['scene_number'] in retry_nums]
        # Delete stale pending images so skip-if-exists doesn't block the retry.
        for s in scenes_to_do:
            p = f"{video_folder}/images/pending/scene_{s['scene_number']:03d}.png"
            if os.path.exists(p):
                os.remove(p)
        total_scenes = len(scenes_to_do)
        print(f"Retry mode: regenerating {total_scenes} rejected scenes only.\n", flush=True)

# --- Run, bounded-parallel ----------------------------------------------
print(f"Generating {len(scenes_to_do)} scenes with {MAX_CONCURRENT_SCENES} concurrent workers...\n", flush=True)
start_ts = time.time()

with ThreadPoolExecutor(max_workers=MAX_CONCURRENT_SCENES) as pool:
    list(pool.map(process_scene, scenes_to_do))

elapsed_min = round((time.time() - start_ts) / 60, 1)

# Auto-write rejected.txt for failed scenes
failed_scenes.sort()
if failed_scenes:
    with open(rejected_path, 'w', encoding='utf-8') as f:
        for n in failed_scenes:
            f.write(f"{n}\n")
    print(f"\n[WARN] {len(failed_scenes)} scenes failed — written to rejected.txt: {failed_scenes}", flush=True)
    print(f"       Run /generate_images_pov again to retry only these scenes.", flush=True)
else:
    if os.path.exists(rejected_path):
        os.remove(rejected_path)

print(f"\n{'='*50}", flush=True)
print(f"Done: {success_count}/{total_scenes} scenes generated successfully in {elapsed_min} min.", flush=True)
if failed_scenes:
    print(f"Failed: {failed_scenes}", flush=True)
else:
    print(f"All scenes completed — no failures.", flush=True)

# --- Route summary — confirms object scenes never got a face reference ---
from collections import Counter
routes = Counter()
for s in scenes_to_do:
    role = s.get('character', '?')
    if role == 'main' and PROTAGONIST_URL:
        routes['image-to-image (protagonist ref)'] += 1
    elif role == 'main':
        routes['text-to-image (main, NO ref!)'] += 1
    elif role == 'supporting':
        routes['text-to-image (world/full faces)'] += 1
    elif role == 'none':
        routes['text-to-image (object, no people)'] += 1
    else:
        routes[f'UNKNOWN character value: {role!r}'] += 1

print(f"\n--- Route summary ---", flush=True)
for k, v in routes.items():
    print(f"   {v:>4}  {k}", flush=True)
bad = [k for k in routes if k.startswith('UNKNOWN')]
if bad:
    print(f"   [ERROR] Unexpected character values found: {bad}", flush=True)
    print(f"           Valid values are 'main', 'supporting', 'none'.", flush=True)
```

## STEP 3 — Report Results

After the script finishes, it will automatically print a summary. Tell the user:
- Total scenes generated successfully vs. total scenes, and elapsed minutes
- The **route summary** — confirm that `object_scene` count matches the `text-to-image (object, no people)` route, and that no `main` scene fell into the no-reference fallback
- If any failed: the scene numbers are already written to `rejected.txt` automatically — just run `/generate_images_pov` again to retry only those scenes, no manual editing needed
- Location of new images: `[video_folder]/images/pending/`
- Next step: review images in pending/, copy approved ones to approved/, then run `/generate_audio_pov` or `/generate_final_pov` when all images are in approved/

## STEP 4 — What to Look For When Reviewing 250 Images

Reviewing 250 images is the real bottleneck. Do not scrub them in order — scan for these five failure modes, which account for nearly everything that goes wrong:

1. **The protagonist grew hair.** Baldness is the one thing he must never lose — it is his only signature. The model adds hair by default because nearly every head it has ever seen has some — this is the single most common failure. Check every scene. A nose, ears, eyebrows, and a mouth are all correct now; only hair is the failure.

2. **The protagonist did not age.** Compare a Level 2 image against a Level 9 image side by side. If they look the same age, the `character_bank` age layers were too subtle. Fix the bank, then reject that level's scenes.

3. **A secondary character went bald, or wears the protagonist's exact face.** The inverse failure. The protagonist reference bled onto a `world_scene`. This means a `supporting` scene was routed to image-to-image — check the route summary.

4. **A person appeared in an object scene.** A hand holding the flag, a figure in the background. Object scenes must be empty of people.

5. **A smile on a grieving line.** He has a mouth, so the model will use it — and its default is a pleasant one. Scan every `cost_scene` for a protagonist smiling while someone dies. Reject those.

6. **Costume drift within a level.** The protagonist changes uniform between scene 88 and scene 91, both Level 5. The costume string for that level was too vague. Tighten it in `character_bank`.

Write failing scene numbers into `rejected.txt`, one per line, and re-run. The script deletes their stale pending images automatically so the retry actually regenerates them.
```

---

### File 4: generate_audio_pov.md

Save this as `.claude/commands/generate_audio_pov.md`. This is the **recommended** final stage — clean one-pass export, no drift. If you want a fully automated MP4 output instead, see the optional `generate_final_pov.md` file below.

```markdown
**IMPORTANT — Execution method:** Run the Python scripts in this file using the Bash tool with `run_in_background=true`. Do NOT use PowerShell's `Start-Process` to launch them. `run_in_background=true` triggers an automatic notification the moment a script finishes — without it, the script can complete successfully with zero indication, and Claude Code will sit idle waiting for the user to ask "is it done?" instead of reporting results immediately.

**Windows execution note:** `run_in_background=true` via the Bash tool is preferred because it auto-notifies when the script finishes. If Bash fails due to Windows path parsing and you fall back to PowerShell's `Start-Process`, there is no automatic notification on completion. In that case, do not wait silently. After launching, periodically read the script's log/output file (e.g. every 60-90 seconds) to check progress, and proactively report completion to the user as soon as the final summary line appears in the output — never wait for the user to ask "is it done?"

# Generate Audio Files for Each POV Scene

Generate one ElevenLabs voiceover audio file per scene, plus one subtitle file you can upload to YouTube. Stop there. The user assembles the final video manually in CapCut, or runs `/generate_final_pov` for an automated assembly.

**No FFmpeg video assembly here. No clip concatenation. No final MP4.** This command ends at audio + subtitles.

## Scale Warning — Read Before Running

A POV video is **~250 scenes**, not ~120. That means:

- **~250 ElevenLabs calls.** Sequentially at ~5s each, that is roughly **20–25 minutes**.
- **~3,300 words ≈ 18,000 characters.** Check your ElevenLabs quota before starting. A single POV video consumes roughly double what a finance video does.
- **Resumability is not optional.** The script skips any scene that already has an MP3 on disk. If it dies at scene 180, re-running it costs nothing for scenes 1–179.

The script below generates audio with a small worker pool (default 3 concurrent) to bring the wall time down to ~8 minutes. Set `MAX_CONCURRENT` to 1 if ElevenLabs rate-limits you.

## STEP 1 — Read Active Video Folder and Verify Images

Read `videos/current_video_folder.txt` to get the active video folder path.

Read `[video_folder]/current_script.json` and extract:
- `total_scenes` — the actual number of scenes (do NOT assume any fixed number)
- The full scene list with voiceover text for each scene

Check that `[video_folder]/images/approved/` contains an image for every scene number from 1 to `total_scenes`.
If any are missing → tell the user exactly which scene numbers are missing and stop. Do not proceed to audio generation if images are incomplete.

## STEP 2 — Voice Direction for the POV Format

This is not the finance channel. The narrator is not a friendly host explaining a concept. The narrator is telling the viewer what is happening to **them**, in the present tense, calmly, without warmth.

Recommended ElevenLabs voice settings for this format:

```
stability:        0.45    (higher than finance — the POV voice should not swing emotionally)
similarity_boost: 0.80
style:            0.15    (low — restraint carries the format; theatricality kills it)
speed:            0.92    (slower than finance — this is a documentary, not an explainer)
```

**Why higher stability.** The finance channel uses `0.3` because Jeff is expressive and reacts to the material. A POV narrator who dramatizes every cost beat sounds like a trailer voice and the immersion collapses. The lines are already devastating; the delivery must be flat enough to let them land. Grief is stated, not performed.

**Why slower.** At 130 wpm the script runs 25 minutes. The sentences are short — 5 to 11 words. Read at finance pace, the short sentences run together and the format's rhythm disappears. `speed: 0.92` gives each sentence its own beat.

**Do not add emotion tags.** No `[sad]`, no `[whispers]`. The script's power comes from a steady voice describing an unsteady life. If a line needs a pause, the script's sentence break already provides it.

Set these in `.env`:
```
ELEVEN_LABS_VOICE_ID=<your chosen voice>
ELEVEN_LABS_STABILITY=0.45
ELEVEN_LABS_SIMILARITY=0.80
ELEVEN_LABS_STYLE=0.15
ELEVEN_LABS_SPEED=0.92
```

The script reads these with the values above as defaults, so `.env` overrides are optional.

## STEP 3 — Generate Voiceover for Each Scene

**Naming convention:** `scene_001.mp3`, `scene_002.mp3`, … — three-digit zero-padded scene numbers matching the image filenames (`scene_001.png` in `images/approved/`). Alphabetical sort then puts matching audio and image side by side in CapCut.

**Save raw MP3 from ElevenLabs.** No re-encoding, no normalization, no trimming. The MP3 the API returns is what we keep.

**Hard timeout on every API call:** `requests`' own `timeout=` parameter only covers the HTTP layer. If the TCP socket itself hangs, that timeout never fires and the call blocks forever. Every ElevenLabs call below is wrapped in a thread with a hard deadline.

```python
import requests, os, json, sys, time, threading
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FutureTimeoutError
sys.stdout.reconfigure(line_buffering=True, encoding='utf-8')

# ---- Tuning -------------------------------------------------------------
# 250 sequential ElevenLabs calls is ~20-25 min. 3 concurrent brings it to
# ~8 min. Drop to 1 if you hit 429 rate limits.
MAX_CONCURRENT = 3
# -------------------------------------------------------------------------

# Load .env file directly — do not rely on os.getenv
env = {}
try:
    with open('.env', 'r', encoding='utf-8-sig') as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
except:
    pass

ELEVEN_LABS_API_KEY = env.get('ELEVEN_LABS_API_KEY') or os.getenv('ELEVEN_LABS_API_KEY')
VOICE_ID = env.get('ELEVEN_LABS_VOICE_ID') or os.getenv('ELEVEN_LABS_VOICE_ID')

if not ELEVEN_LABS_API_KEY or not VOICE_ID:
    print("ERROR: ELEVEN_LABS_API_KEY or ELEVEN_LABS_VOICE_ID missing from .env", flush=True)
    sys.exit(1)

# POV voice defaults — flatter and slower than the finance channel.
def _f(key, default):
    try:
        return float(env.get(key, default))
    except ValueError:
        return default

STABILITY  = _f('ELEVEN_LABS_STABILITY',  0.45)
SIMILARITY = _f('ELEVEN_LABS_SIMILARITY', 0.80)
STYLE      = _f('ELEVEN_LABS_STYLE',      0.15)
SPEED      = _f('ELEVEN_LABS_SPEED',      0.92)

with open('videos/current_video_folder.txt', 'r', encoding='utf-8-sig') as f:
    video_folder = f.read().strip()

with open(f'{video_folder}/current_script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)

scenes = script['scenes']
total_scenes = script.get('total_scenes') or len(scenes)
os.makedirs(f'{video_folder}/audio', exist_ok=True)

# --- Verify every approved image exists before spending money on audio ---
missing_images = [
    s['scene_number'] for s in scenes
    if not os.path.exists(f"{video_folder}/images/approved/scene_{s['scene_number']:03d}.png")
]
if missing_images:
    print(f"ERROR: {len(missing_images)} scenes have no approved image.", flush=True)
    print(f"       Missing: {missing_images[:20]}{' ...' if len(missing_images) > 20 else ''}", flush=True)
    print("       Approve all images before generating audio.", flush=True)
    sys.exit(1)

# --- HARD TIMEOUT WRAPPER ---
# The pool must be at least as large as MAX_CONCURRENT or scenes deadlock
# waiting for a free timeout worker.
_executor = ThreadPoolExecutor(max_workers=max(4, MAX_CONCURRENT * 2))

def call_with_hard_timeout(func, timeout_seconds, *args, **kwargs):
    """Run func in a separate thread. If it doesn't return within
    timeout_seconds, give up and move on — the underlying thread is
    abandoned rather than waited on, so a TCP-layer hang can never
    freeze the whole script."""
    future = _executor.submit(func, *args, **kwargs)
    try:
        return future.result(timeout=timeout_seconds)
    except FutureTimeoutError:
        return None
    except Exception as e:
        print(f"  (call error: {e})", flush=True)
        return None

def _do_generate_audio(text, scene_number):
    url = f"https://api.elevenlabs.io/v1/text-to-speech/{VOICE_ID}"
    headers = {
        "xi-api-key": ELEVEN_LABS_API_KEY,
        "Content-Type": "application/json"
    }
    data = {
        "text": text,
        "model_id": "eleven_v3",
        "voice_settings": {
            "stability": STABILITY,
            "similarity_boost": SIMILARITY,
            "style": STYLE,
            "speed": SPEED
        }
    }
    response = requests.post(url, json=data, headers=headers, timeout=30)
    if response.status_code == 429:
        raise Exception(f"rate limited (429) on scene {scene_number}")
    if response.status_code != 200:
        raise Exception(f"ElevenLabs error scene {scene_number}: {response.status_code} {response.text[:200]}")
    return response.content

def generate_audio(text, scene_number, max_retries=2):
    for attempt in range(max_retries + 1):
        content = call_with_hard_timeout(_do_generate_audio, 35, text, scene_number)
        if content:
            filename = f"{video_folder}/audio/scene_{scene_number:03d}.mp3"
            # Write to a temp file then rename, so a crash mid-write can never
            # leave a truncated MP3 that skip-if-exists would happily accept.
            tmp = filename + '.part'
            with open(tmp, 'wb') as f:
                f.write(content)
            os.replace(tmp, filename)
            return filename
        if attempt < max_retries:
            wait = 2 ** attempt
            print(f"  Scene {scene_number}: attempt {attempt+1} hung or errored, retrying in {wait}s...", flush=True)
            time.sleep(wait)
    print(f"[FAIL] Scene {scene_number}: failed after {max_retries+1} attempts.", flush=True)
    return None

failed_scenes = []
done_count = 0
_lock = threading.Lock()

def process(scene):
    global done_count
    n = scene['scene_number']
    path = f"{video_folder}/audio/scene_{n:03d}.mp3"
    if os.path.exists(path) and os.path.getsize(path) > 1024:
        print(f"[SKIP] Scene {n}: audio already exists.", flush=True)
        with _lock:
            done_count += 1
        return
    result = generate_audio(scene['voiceover'], n)
    with _lock:
        if result:
            done_count += 1
            if done_count % 25 == 0:
                print(f"  --- progress: {done_count}/{total_scenes} ---", flush=True)
        else:
            failed_scenes.append(n)

print(f"Generating audio for {total_scenes} scenes with {MAX_CONCURRENT} workers...\n", flush=True)
start = time.time()
with ThreadPoolExecutor(max_workers=MAX_CONCURRENT) as pool:
    list(pool.map(process, scenes))
elapsed = (time.time() - start) / 60

failed_scenes.sort()
if failed_scenes:
    print(f"\n[WARN] {len(failed_scenes)} scenes failed: {failed_scenes}", flush=True)
    print(f"       Re-run /generate_audio_pov to retry — it skips scenes that already have an MP3.", flush=True)
else:
    print(f"\n[OK] Generated {total_scenes} audio files in {video_folder}/audio/ ({elapsed:.1f} min)", flush=True)

# --- Character count, for quota accounting ---
chars = sum(len(s['voiceover']) for s in scenes)
print(f"     Characters sent to ElevenLabs: {chars:,}", flush=True)
```

This produces `total_scenes` MP3 files — one per scene, each with its own natural duration. Files are named `scene_001.mp3` through `scene_NNN.mp3`.

## STEP 4 — Generate the Subtitle File

Generate one `.srt` covering all scenes in sequence. This file is **not burned into any video**. It exists so the user can upload it to YouTube as a caption track, which improves accessibility and gives YouTube a clean transcript to index.

Use `ffprobe` to measure each MP3's real duration. This is the only ffmpeg-family tool used here, and it does not encode anything — it reads metadata.

**A note on MP3 durations.** ElevenLabs returns MP3s whose real duration is slightly longer than the "natural" read — MP3 frames are padded to a fixed size. A 4.30-second read comes back as a 4.336-second file. Across 250 scenes those fractions sum to several seconds. Always measure with `ffprobe`; never estimate duration from word count.

```python
import subprocess, json, os

def get_audio_duration(filepath):
    """Read duration from the container format, not the stream. Some MP3s
    report a missing or zero stream duration while the format-level duration
    is correct."""
    result = subprocess.run([
        'ffprobe', '-v', 'quiet',
        '-print_format', 'json',
        '-show_format',
        filepath
    ], capture_output=True, text=True)
    info = json.loads(result.stdout)
    return float(info['format']['duration'])

def seconds_to_srt_time(seconds):
    """Convert seconds to SRT timestamp.

    Round to whole milliseconds FIRST, then decompose. Rounding each field
    separately lets a value like 59.9999 render as "00:00:60,000" — an invalid
    SRT timestamp that YouTube rejects outright. Integer division after a single
    round makes every carry (ms -> s -> m -> h) happen correctly.
    """
    total_ms = int(round(seconds * 1000))
    h, rem = divmod(total_ms, 3_600_000)
    m, rem = divmod(rem, 60_000)
    s, ms  = divmod(rem, 1000)
    return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"

srt_lines = []
current_time = 0.0

for scene in scenes:
    n = scene['scene_number']
    audio_file = f"{video_folder}/audio/scene_{n:03d}.mp3"
    duration = get_audio_duration(audio_file)

    start = seconds_to_srt_time(current_time)
    end = seconds_to_srt_time(current_time + duration)

    srt_lines.append(str(n))
    srt_lines.append(f"{start} --> {end}")
    srt_lines.append(scene['voiceover'])
    srt_lines.append("")

    current_time += duration

os.makedirs(f'{video_folder}/final_output', exist_ok=True)
srt_path = f'{video_folder}/final_output/subtitles.srt'
with open(srt_path, 'w', encoding='utf-8') as f:
    f.write('\n'.join(srt_lines))

print(f"[OK] SRT subtitle file: {srt_path}")
print(f"     Total audio duration: {current_time:.1f} seconds ({current_time/60:.1f} minutes)")

if current_time < 1380:   # 23 minutes
    print(f"[WARN] Audio is only {current_time/60:.1f} min — target is ~25 min.")
    print(f"       The script may have come back short. Check the word count.")
```

**Important note about SRT timing.** This SRT assumes scenes are placed back-to-back with zero gap. That is exactly what `/generate_final_pov` does, so the timings will match its output.

If you instead assemble in CapCut and add transitions, pauses, or extra clips, these timings will drift. In that case either regenerate the SRT after the edit is locked, or let CapCut auto-caption the exported video.

## STEP 5 — No Cleanup. Keep Everything.

This command deletes nothing. The user needs everything on disk:

- `images/approved/` — all approved scene images
- `audio/` — all per-scene MP3 files **(KEPT)**
- `final_output/subtitles.srt` — subtitle file for YouTube upload
- `current_script.json` — original script

The `images/pending/` folder may be deleted if the user explicitly confirms all images are approved. That is the only optional cleanup.

## STEP 6 — Report Results

Output to the user, with concrete paths:

```
Audio generation complete.

Video folder: [video_folder]/

Files ready:
- Images:    [video_folder]/images/approved/   (N PNG files: scene_001.png -> scene_NNN.png)
- Audio:     [video_folder]/audio/             (N MP3 files: scene_001.mp3 -> scene_NNN.mp3)
- Subtitles: [video_folder]/final_output/subtitles.srt

Stats:
- Total scenes: [total_scenes]
- Combined audio duration: [min:sec]
- Characters sent to ElevenLabs: [count]

Next step — pick one:

  A) Automated assembly:  run /generate_final_pov
     Produces final_video.mp4 with no subtitles burned in, no zoom effects.
     Upload subtitles.srt to YouTube separately as a caption track.

  B) Manual edit in CapCut:
     1. New project, 1920x1080, 30fps
     2. Drag all PNGs from images/approved/ into the timeline (auto-sorts by filename)
     3. Drag all MP3s from audio/ into the audio track (also auto-sorts)
     4. Match each image's duration to its audio clip (scene_001.png = scene_001.mp3)
     5. Do NOT import subtitles.srt into CapCut — upload it to YouTube instead
     6. Add intro, B-roll, music as needed, then export

YouTube metadata (from current_script.json):
- Title:          [title]
- Description:    [description]
- Thumbnail Text: [thumbnail_text]
- Hashtags:       [hashtags]

Cost of this run:
- Audio (ElevenLabs): [character count] characters against your subscription
- ffprobe: free

If any scenes failed: they are listed above. Re-run /generate_audio_pov — it skips
scenes that already have an MP3 and retries only the missing ones.
```

Do not generate a video file here. The pipeline stops at audio and subtitles.
```


---

### File 5: generate_final_pov.md (Optional — Automated FFmpeg Assembly)

> **When to use this:** Run `/generate_final_pov` instead of `/generate_audio_pov` when you want a fully assembled, ready-to-upload MP4 without any manual CapCut editing. FFmpeg will combine all images + audio with Ken Burns zoom effects and burn in styled subtitles automatically.
>
> **When NOT to use this:** If you plan to re-import the output MP4 into CapCut to add B-roll, intro clips, or transitions — don't use this. Re-encoding a pre-rendered MP4 in CapCut causes audio/subtitle drift. In that case, use `/generate_audio_pov` and assemble in CapCut instead.

```markdown
**IMPORTANT — Execution method:** Run every Python and FFmpeg script in this file using the Bash tool with `run_in_background=true`. Do NOT use PowerShell's `Start-Process` to launch them. `run_in_background=true` triggers an automatic notification the moment a step finishes — without it, a step can complete successfully with zero indication, and Claude Code will sit idle waiting for the user to ask "is it done?" This matters more here than anywhere else, since this pipeline runs multiple long stages back-to-back on ~250 scenes.

**Windows execution note:** `run_in_background=true` via the Bash tool is preferred because it auto-notifies when the script finishes. If Bash fails due to Windows path parsing and you fall back to PowerShell's `Start-Process`, there is no automatic notification on completion. In that case, do not wait silently. After launching, periodically read the script's log/output file (e.g. every 60-90 seconds) to check progress, and proactively report completion to the user as soon as the final summary line appears in the output — never wait for the user to ask "is it done?"

# Assemble Final POV Video — Static Cut, No Burn-In

Combine all approved images + voiceover audio into one ready-to-upload MP4.

**Deliberately simple.** No Ken Burns. No zoom. No pan. No burned-in subtitles. Each scene is a still image held for exactly the length of its audio, cut hard to the next. That is the whole edit.

An SRT file is still produced — as a **separate file** you upload to YouTube as a caption track. It is never rendered into the picture.

> **When to use this:** you want a finished MP4 with zero manual editing.
>
> **When NOT to use this:** if you plan to re-import the MP4 into CapCut to add B-roll, an intro, or music. Re-encoding a finished MP4 causes drift. Use `/generate_audio_pov` and assemble raw assets in CapCut instead.

## Why No Zoom

The finance pipeline applies a slow Ken Burns push to every scene because a static cartoon on a white background feels dead. The POV format has the opposite problem: the images are detailed, cinematic, and full of environment. A slow zoom on a 5-second scene does not read as motion — it reads as drift, and across 250 scenes it becomes seasick.

Removing `zoompan` also buys a large technical win. Every clip becomes a still image encoded with identical parameters, which means the concatenation step can **stream-copy** instead of re-encoding.

Measured on this pipeline:

| Concat method | Time (3 clips) | A/V drift |
|---|---|---|
| `-c copy` (stream copy) | 0.08 s | +0.029 s |
| `-c:v libx264` (re-encode) | 8.68 s | +0.041 s |

Stream copy is **~112× faster and slightly more accurate**. Extrapolated to 250 clips, that is the difference between roughly 90 minutes and under a minute for the concat stage. The copy path is only available because every clip shares the same codec, resolution, frame rate, pixel format, and audio parameters — which the build step below enforces.

## The `-shortest` Trap — Do Not Reintroduce It

The finance pipeline ends each clip with `-shortest`. With a `zoompan` filter that is harmless. With a static looped image it is **not**: `-shortest` cuts the clip at the shorter of the two streams, and the looped video stream is quantized to the frame grid while the MP3 is not.

Measured:

```
clip 1: video=4.300s  audio=4.336s  ->  36ms of audio discarded
clip 2: video=6.100s  audio=6.139s  ->  39ms discarded
clip 3: video=5.200s  audio=5.251s  ->  51ms discarded
```

That is ~42 ms lost per scene. Across 250 scenes, **roughly 10 seconds of narration is silently cut** — usually the tail of the final word, on every single scene.

The fix, used below: measure the audio first with `ffprobe`, pass its exact duration as `-t`, and never use `-shortest`. Residual drift drops from 126 ms to 8 ms across three clips, and the leftover is pure frame quantization (1/30 s = 33 ms grain), which does not accumulate.

## STEP 1 — Read Active Video Folder and Verify Assets

Read `videos/current_video_folder.txt` to get the active video folder path.

Read `[video_folder]/current_script.json` and extract:
- `total_scenes` — the actual number of scenes (do NOT assume any fixed number)
- The full scene list with voiceover text

Verify `[video_folder]/images/approved/` contains an image for every scene from 1 to `total_scenes`.
If any are missing → list the exact scene numbers and stop.

The `camera_effect` field in the script JSON is **ignored by this command**. It exists for the CapCut path. Nothing here reads it.

## STEP 2 — Generate Voiceover for Each Scene

Identical to `/generate_audio_pov`. If you already ran that command, every MP3 is on disk and this step skips instantly.

See `/generate_audio_pov` STEP 2 for the voice-direction rationale (higher stability, slower speed). The same defaults are applied below.

```python
import requests, os, json, sys, time, threading, subprocess
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FutureTimeoutError
sys.stdout.reconfigure(line_buffering=True, encoding='utf-8')

MAX_CONCURRENT = 3

# Load .env file directly — do not rely on os.getenv
env = {}
try:
    with open('.env', 'r', encoding='utf-8-sig') as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
except:
    pass

ELEVEN_LABS_API_KEY = env.get('ELEVEN_LABS_API_KEY') or os.getenv('ELEVEN_LABS_API_KEY')
VOICE_ID = env.get('ELEVEN_LABS_VOICE_ID') or os.getenv('ELEVEN_LABS_VOICE_ID')

if not ELEVEN_LABS_API_KEY or not VOICE_ID:
    print("ERROR: ELEVEN_LABS_API_KEY or ELEVEN_LABS_VOICE_ID missing from .env", flush=True)
    sys.exit(1)

def _f(key, default):
    try:
        return float(env.get(key, default))
    except ValueError:
        return default

STABILITY  = _f('ELEVEN_LABS_STABILITY',  0.45)
SIMILARITY = _f('ELEVEN_LABS_SIMILARITY', 0.80)
STYLE      = _f('ELEVEN_LABS_STYLE',      0.15)
SPEED      = _f('ELEVEN_LABS_SPEED',      0.92)

with open('videos/current_video_folder.txt', 'r', encoding='utf-8-sig') as f:
    video_folder = f.read().strip()

with open(f'{video_folder}/current_script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)

scenes = script['scenes']
total_scenes = script.get('total_scenes') or len(scenes)

os.makedirs(f'{video_folder}/audio', exist_ok=True)
os.makedirs(f'{video_folder}/clips', exist_ok=True)
os.makedirs(f'{video_folder}/final_output', exist_ok=True)

missing_images = [
    s['scene_number'] for s in scenes
    if not os.path.exists(f"{video_folder}/images/approved/scene_{s['scene_number']:03d}.png")
]
if missing_images:
    print(f"ERROR: {len(missing_images)} scenes have no approved image.", flush=True)
    print(f"       Missing: {missing_images[:20]}{' ...' if len(missing_images) > 20 else ''}", flush=True)
    sys.exit(1)

_executor = ThreadPoolExecutor(max_workers=max(4, MAX_CONCURRENT * 2))

def call_with_hard_timeout(func, timeout_seconds, *args, **kwargs):
    future = _executor.submit(func, *args, **kwargs)
    try:
        return future.result(timeout=timeout_seconds)
    except FutureTimeoutError:
        return None
    except Exception as e:
        print(f"  (call error: {e})", flush=True)
        return None

def _do_generate_audio(text, scene_number):
    url = f"https://api.elevenlabs.io/v1/text-to-speech/{VOICE_ID}"
    headers = {"xi-api-key": ELEVEN_LABS_API_KEY, "Content-Type": "application/json"}
    data = {
        "text": text,
        "model_id": "eleven_v3",
        "voice_settings": {
            "stability": STABILITY,
            "similarity_boost": SIMILARITY,
            "style": STYLE,
            "speed": SPEED
        }
    }
    response = requests.post(url, json=data, headers=headers, timeout=30)
    if response.status_code != 200:
        raise Exception(f"ElevenLabs error scene {scene_number}: {response.status_code} {response.text[:200]}")
    return response.content

def generate_audio(text, scene_number, max_retries=2):
    for attempt in range(max_retries + 1):
        content = call_with_hard_timeout(_do_generate_audio, 35, text, scene_number)
        if content:
            filename = f"{video_folder}/audio/scene_{scene_number:03d}.mp3"
            tmp = filename + '.part'
            with open(tmp, 'wb') as f:
                f.write(content)
            os.replace(tmp, filename)   # atomic — never leave a truncated MP3
            return filename
        if attempt < max_retries:
            time.sleep(2 ** attempt)
    print(f"[FAIL] Scene {scene_number}: audio failed after {max_retries+1} attempts.", flush=True)
    return None

audio_failed = []
_lock = threading.Lock()

def do_audio(scene):
    n = scene['scene_number']
    existing = f"{video_folder}/audio/scene_{n:03d}.mp3"
    if os.path.exists(existing) and os.path.getsize(existing) > 1024:
        return
    if generate_audio(scene['voiceover'], n) is None:
        with _lock:
            audio_failed.append(n)

print(f"STEP 2: audio for {total_scenes} scenes ({MAX_CONCURRENT} workers)...", flush=True)
t0 = time.time()
with ThreadPoolExecutor(max_workers=MAX_CONCURRENT) as pool:
    list(pool.map(do_audio, scenes))

if audio_failed:
    audio_failed.sort()
    print(f"\n[FAIL] {len(audio_failed)} scenes failed audio: {audio_failed}", flush=True)
    print("Stopping — fix audio before building clips. Re-run /generate_final_pov to retry.", flush=True)
    sys.exit(1)

print(f"[OK] All {total_scenes} audio files present. ({(time.time()-t0)/60:.1f} min)\n", flush=True)
```

## STEP 3 — Build Per-Scene Clips (Static, Identical Params)

Every clip is encoded with the **exact same** codec, resolution, frame rate, pixel format, sample rate, and channel layout. That uniformity is what makes the stream-copy concat in STEP 4 legal. Change any parameter here and you must re-encode in STEP 4.

Note there is no `-shortest`. The audio duration is measured first and passed as `-t`.

```python
FPS = 30

def probe_duration(filepath):
    """Read duration from the container format, not the stream. Some MP3s
    report a missing or zero stream duration while format-level is correct."""
    r = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json', '-show_format', filepath],
        capture_output=True, text=True
    )
    return float(json.loads(r.stdout)['format']['duration'])

def build_clip(n):
    img   = f"{video_folder}/images/approved/scene_{n:03d}.png"
    audio = f"{video_folder}/audio/scene_{n:03d}.mp3"
    out   = f"{video_folder}/clips/scene_{n:03d}.mp4"

    if os.path.exists(out) and os.path.getsize(out) > 4096:
        return 'skip'

    dur = probe_duration(audio)

    # Write to a temp file then rename, so an interrupted encode can never leave
    # a half-written clip that the skip-if-exists check would accept.
    # The temp name MUST still end in .mp4 — ffmpeg infers the container format
    # from the extension, and a name ending in ".part" makes the muxer fail with
    # "Invalid argument".
    tmp = f"{video_folder}/clips/.tmp_scene_{n:03d}.mp4"

    r = subprocess.run([
        'ffmpeg', '-y',
        '-loop', '1', '-framerate', str(FPS), '-i', img,
        '-i', audio,
        # Explicit duration from the AUDIO. Never -shortest: it would truncate
        # the audio to the frame-quantized video length, losing ~42ms/scene.
        '-t', f'{dur:.6f}',
        '-vf', ('scale=1920:1080:force_original_aspect_ratio=decrease,'
                'pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1,format=yuv420p'),
        '-r', str(FPS), '-fps_mode', 'cfr',
        '-c:v', 'libx264', '-preset', 'veryfast', '-crf', '20',
        '-pix_fmt', 'yuv420p',
        '-c:a', 'aac', '-b:a', '192k', '-ar', '48000', '-ac', '2',
        '-movflags', '+faststart',
        tmp
    ], capture_output=True, text=True)

    if r.returncode != 0:
        if os.path.exists(tmp):
            os.remove(tmp)
        print(f"[FAIL] Clip {n}: {r.stderr[-300:]}", flush=True)
        return 'fail'

    os.replace(tmp, out)
    return 'built'

print(f"STEP 3: building {total_scenes} static clips...", flush=True)
t0 = time.time()
built = skipped = 0
clip_failed = []

for scene in scenes:
    n = scene['scene_number']
    status = build_clip(n)
    if status == 'built':
        built += 1
    elif status == 'skip':
        skipped += 1
    else:
        clip_failed.append(n)
    if (built + skipped) % 25 == 0 and built + skipped > 0:
        print(f"  --- {built + skipped}/{total_scenes} clips ---", flush=True)

if clip_failed:
    print(f"\n[FAIL] {len(clip_failed)} clips failed: {clip_failed}", flush=True)
    print("Stopping. Fix and re-run — completed clips are skipped.", flush=True)
    sys.exit(1)

print(f"[OK] {built} built, {skipped} skipped. ({(time.time()-t0)/60:.1f} min)\n", flush=True)
```

## STEP 4 — Concatenate with Stream Copy

Build the concat list dynamically from the scene list — never hardcode a scene count.

```python
concat_list = f'{video_folder}/clips/concat_list.txt'
clips_dir = os.path.abspath(f'{video_folder}/clips')

with open(concat_list, 'w', encoding='utf-8') as f:
    for scene in scenes:
        i = scene['scene_number']
        # IMPORTANT: write the ABSOLUTE path of each clip, with forward slashes.
        # FFmpeg's concat demuxer resolves bare/relative filenames against the
        # ffmpeg PROCESS's working directory, not against the folder the list
        # file lives in — so a bare "scene_001.mp4" silently fails whenever the
        # script runs from a different cwd (the normal case on Windows).
        # Forward slashes avoid backslash-escaping inside the "file '...'" directive.
        clip_path = os.path.join(clips_dir, f'scene_{i:03d}.mp4').replace('\\', '/')
        f.write(f"file '{clip_path}'\n")

final_video = os.path.abspath(f'{video_folder}/final_output/final_video.mp4')

print("STEP 4: concatenating (stream copy — no re-encode)...", flush=True)
t0 = time.time()

r = subprocess.run([
    'ffmpeg', '-y',
    '-f', 'concat', '-safe', '0', '-i', os.path.abspath(concat_list),
    '-c', 'copy',
    '-movflags', '+faststart',
    final_video
], capture_output=True, text=True)

if r.returncode != 0:
    # Fallback: if any clip drifted from the uniform parameter set, copy fails.
    # Re-encode once rather than dying. Slow, but correct.
    print("[WARN] Stream copy failed — clips are not parameter-identical.", flush=True)
    print("       Falling back to a full re-encode. This will take much longer.", flush=True)
    print(f"       ffmpeg said: {r.stderr[-400:]}", flush=True)
    r2 = subprocess.run([
        'ffmpeg', '-y',
        '-f', 'concat', '-safe', '0', '-i', os.path.abspath(concat_list),
        '-c:v', 'libx264', '-preset', 'veryfast', '-crf', '20', '-pix_fmt', 'yuv420p',
        '-c:a', 'aac', '-b:a', '192k', '-ar', '48000', '-ac', '2',
        '-r', str(FPS), '-fps_mode', 'cfr',
        '-movflags', '+faststart',
        final_video
    ], capture_output=True, text=True)
    if r2.returncode != 0:
        print(f"[FAIL] Re-encode concat also failed: {r2.stderr[-400:]}", flush=True)
        sys.exit(1)

print(f"[OK] Concatenated. ({time.time()-t0:.1f}s)\n", flush=True)
```

## STEP 5 — Verify A/V Sync Before Declaring Success

Never trust a concat you have not measured. This step compares the video stream length, the audio stream length, and the sum of the source MP3s.

```python
def stream_duration(path, selector):
    r = subprocess.run([
        'ffprobe', '-v', 'error', '-select_streams', selector,
        '-show_entries', 'stream=duration', '-of', 'csv=p=0', path
    ], capture_output=True, text=True)
    try:
        return float(r.stdout.strip())
    except ValueError:
        return 0.0

v_dur = stream_duration(final_video, 'v:0')
a_dur = stream_duration(final_video, 'a:0')
mp3_sum = sum(probe_duration(f"{video_folder}/audio/scene_{s['scene_number']:03d}.mp3") for s in scenes)
skew = a_dur - v_dur

print("STEP 5: sync verification", flush=True)
print(f"   video stream : {v_dur:8.3f}s", flush=True)
print(f"   audio stream : {a_dur:8.3f}s", flush=True)
print(f"   sum of MP3s  : {mp3_sum:8.3f}s", flush=True)
print(f"   A/V skew     : {skew:+8.3f}s", flush=True)
print(f"   vs source    : {v_dur - mp3_sum:+8.3f}s", flush=True)

# Frame quantization gives ~1/30s = 33ms of unavoidable grain per boundary,
# but it does NOT accumulate. Anything past 0.5s means something is wrong.
if abs(skew) > 0.5:
    print(f"\n[ERROR] A/V skew of {skew:+.3f}s is audible. Do not upload this file.", flush=True)
    print("        Likely cause: -shortest was reintroduced, or clips have mixed", flush=True)
    print("        frame rates / sample rates. Delete clips/ and rebuild.", flush=True)
elif abs(skew) > 0.15:
    print(f"\n[WARN] A/V skew of {skew:+.3f}s is larger than expected but probably inaudible.", flush=True)
else:
    print(f"\n[OK] Sync is clean.", flush=True)

if abs(v_dur - mp3_sum) > 2.0:
    print(f"[WARN] Final video is {v_dur - mp3_sum:+.1f}s off the sum of source audio.", flush=True)
    print("       Check whether any clip was truncated.", flush=True)
```

## STEP 6 — Generate the SRT (Separate File, Never Burned In)

The subtitle file is built from the **clip** durations, not the MP3 durations.

This distinction matters. Each clip's length is `-t` quantized onto the 30 fps frame grid, so it differs from its source MP3 by up to 33 ms. An SRT built from MP3 durations drifts against the video it is supposed to caption:

```
scene 1: mp3  0.000- 4.336   clip  0.000- 4.333   start drift +0.000
scene 2: mp3  4.336-10.475   clip  4.333-10.467   start drift -0.003
scene 3: mp3 10.475-15.726   clip 10.467-15.733   start drift -0.008
```

Small over three scenes; meaningful over 250. Measure the clips.

```python
def seconds_to_srt_time(seconds):
    """Convert seconds to SRT timestamp.

    Round to whole milliseconds FIRST, then decompose. Rounding each field
    separately lets a value like 59.9999 render as "00:00:60,000" — an invalid
    SRT timestamp that YouTube rejects outright. Integer division after a single
    round makes every carry (ms -> s -> m -> h) happen correctly.
    """
    total_ms = int(round(seconds * 1000))
    h, rem = divmod(total_ms, 3_600_000)
    m, rem = divmod(rem, 60_000)
    s, ms  = divmod(rem, 1000)
    return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"

srt_lines = []
current_time = 0.0

for scene in scenes:
    n = scene['scene_number']
    # Measure the CLIP, not the MP3 — the clip is what the video actually plays.
    duration = probe_duration(f"{video_folder}/clips/scene_{n:03d}.mp4")

    start = seconds_to_srt_time(current_time)
    end   = seconds_to_srt_time(current_time + duration)

    srt_lines.append(str(n))
    srt_lines.append(f"{start} --> {end}")
    srt_lines.append(scene['voiceover'])
    srt_lines.append("")

    current_time += duration

srt_path = f'{video_folder}/final_output/subtitles.srt'
with open(srt_path, 'w', encoding='utf-8') as f:
    f.write('\n'.join(srt_lines))

print(f"\n[OK] Subtitle file: {srt_path}", flush=True)
print(f"     SRT total: {current_time:.3f}s   video: {v_dur:.3f}s   delta: {current_time - v_dur:+.3f}s", flush=True)
print(f"\n[OK] Final video: {final_video}", flush=True)
print(f"     Duration: {v_dur/60:.1f} minutes", flush=True)
print(f"\n     Subtitles are NOT burned into the video.", flush=True)
print(f"     Upload subtitles.srt to YouTube separately as a caption track.", flush=True)
```

## STEP 7 — No Automatic Cleanup

Delete nothing. Leave everything on disk; the user cleans up manually.

Files present after `/generate_final_pov` completes:

| Path | Keep? |
|---|---|
| `images/approved/` | **KEEP** — needed to rebuild if anything changes |
| `audio/` | safe to delete manually to reclaim space |
| `clips/` | safe to delete manually — this is the largest folder (~250 MP4s) |
| `final_output/final_video.mp4` | **KEEP** |
| `final_output/subtitles.srt` | **KEEP** |
| `current_script.json` | **KEEP** |

There is no `raw_concat.mp4`. Because nothing is burned in, the concat output *is* the final video — one less intermediate file and one less encode pass than the finance pipeline.

## STEP 8 — Report Results

Output to the user:

```
Video assembly complete.

Output files:
- Final video: [video_folder]/final_output/final_video.mp4
- Subtitles:   [video_folder]/final_output/subtitles.srt   (NOT burned in)

Stats:
- Total scenes:  [total_scenes]
- Duration:      [minutes] min
- A/V skew:      [skew] s        (anything under 0.15s is clean)
- Edit style:    static cut, no zoom, no pan, no burned-in text

Uploading to YouTube:
1. Upload final_video.mp4
2. Go to Subtitles -> Add Language -> English -> Upload File -> select subtitles.srt
   This gives YouTube a clean transcript for indexing and accessibility.

YouTube metadata (from current_script.json):
- Title:          [title]
- Description:    [description]
- Thumbnail Text: [thumbnail_text]
- Hashtags:       [hashtags]

Cost of this run:
- Audio (ElevenLabs): [character count] characters against your subscription
- FFmpeg: free
- Total additional cost: ~$0

Storage:
- Kept: everything. Nothing deleted automatically.
- To reclaim space: delete clips/ (largest), then audio/
- Never delete images/approved/ until you are sure you will not rebuild.

Warning: do NOT re-import final_video.mp4 into CapCut to add B-roll or music.
Re-encoding a finished MP4 causes drift. Run /generate_audio_pov instead and
assemble the raw assets on CapCut's timeline.

If audio or clip building failed partway: the failing scene numbers are listed
above. Re-run /generate_final_pov — it skips every scene that already has an
MP3 and a clip, and resumes from the failure.
```
```


---

## Running Your First Video

### Stage 0 — Generate a Topic (~1 minute, fully automatic)

Type this in Claude Code:
```
/generate_ideas_pov
```

Claude Code will read your `ideas_history.txt` to see what you've already covered, then generate 20 fresh "Your Life at Every Rank in X" topic ideas — each with a thumbnail line and a world type. Pick one from the list.

**→ Pick a topic, then run:**

### Stage 1 — Write the Script (~5–10 minutes, fully automatic)

```
/generate_script_pov "Your Life as Every Rank in a Nuclear Submarine Crew"
```

Claude Code will write the full ten-level POV script, split it into ~250 scenes, build the `character_bank` (protagonist age layers + costumes + locked secondary characters), automatically log the topic to `ideas_history.txt`, and output your YouTube title and thumbnail text.

**→ Open `videos/[video_folder]/current_script.json` and read through the script. Check the voiceover text for each scene. If anything needs changing, edit the JSON directly before proceeding. When you're happy, run:**

### Stage 2 — Generate Images (~55 minutes for ~250 scenes, automatic + your review)

```
/generate_images_pov
```

Claude Code will call GPT Image 2 for each scene in a bounded parallel worker pool. It attaches the protagonist reference (from `PROTAGONIST_URL`) only to scenes where the protagonist appears (`pov_scene` and `cost_scene`), and routes `world_scene` / `object_scene` through plain text-to-image with **no** reference so no protagonist bleeds into them. All images save to `videos/images/pending/`.

**→ Open the `pending/` folder. Review EVERY image before moving on:**
- Like it → copy to `approved/`
- Don't like it → write the scene number in `rejected.txt`, then run `/generate_images_pov` again to regenerate only those scenes
- Only proceed to the next stage when ALL images are in `approved/`

### Stage 3 — Final Stage: Choose Your Workflow

**Option A — Manual assembly in CapCut (recommended):**
```
/generate_audio_pov
```
Generates one ElevenLabs MP3 per scene + one combined `subtitles.srt`. No MP4 produced. You import the raw audio + images into CapCut and assemble manually. Clean one-pass export, no drift, full creative control.

**Option B — Automated FFmpeg assembly (optional):**
```
/generate_final_pov
```
Generates audio, builds one static clip per scene (a still held for the length of its audio — no Ken Burns, no zoom), hard-cuts them together with a fast stream-copy concat, and produces a ready-to-upload `final_video.mp4` plus a **separate** `subtitles.srt` (uploaded to YouTube as a caption track, never burned into the picture). Use this when you want a finished video without manual editing. ⚠️ Do NOT re-import this MP4 into CapCut — it will cause drift.

### Stage 4A — Manual Assembly in CapCut (~15–30 minutes — only if you chose Option A)

Open CapCut and create a new project at 1920×1080 / 30fps. Then:

1. Drag all PNG files from `images/approved/` into the video track. They sort by filename, so `scene_001.png` is first.
2. Drag all MP3 files from `audio/` into the audio track. Same alphabetical sort.
3. For each scene, set the image clip duration to match the audio clip duration (CapCut shows audio length on each clip — just drag the image's right edge to match).
4. Import `final_output/subtitles.srt` as the subtitle track.
5. Add any intro clips, B-roll, transitions, or background music you want.
6. Export at 1920×1080, H.264, AAC.

Because CapCut works with the raw audio + raw images directly (rather than re-encoding a pre-rendered MP4), the export will not suffer the audio/subtitle desync that v37's FFmpeg pipeline caused.

---

## Handling Images You Don't Like

After Stage 1, if some images need to be redone:

**Step 1:** Open `videos/images/rejected.txt`

**Step 2:** Write the scene numbers you want regenerated, one per line:
```
3
7
15
42
```

**Step 3:** Run:
```
/generate_images_pov
```

Claude Code reads `rejected.txt` and only regenerates those specific scenes. Everything already in `approved/` stays untouched. Repeat until all images are approved.

---

## Cost Breakdown

### Per ~25-Minute POV Video (~250 Images)

| Component | Tool | Calculation | Cost |
|---|---|---|---|
| ~250 images (1K, 16:9) | GPT Image 2 via Kie.ai | ~250 × $0.03 | ~$7.50 |
| Voiceover (~25 min, ~18k chars) | ElevenLabs | Existing account | ~$1.70 |
| Audio + SRT generation (Python) | Built-in | Free | $0 |
| Manual edit in CapCut | CapCut Free or Pro | Free / your existing license | $0 |
| Script writing | Anthropic API (claude-opus-4-8) | ~$0.50–0.80 per script (64k tokens max, streaming) | ~$0.65 |
| **Total** | | | **~$9.85/video** |

Note: a POV video is roughly double the scene count of a finance video (~250 vs ~130), so both image and voiceover costs are correspondingly higher. Image generation dominates the cost and the wall-clock time. If you produce shorter POV cuts the image cost scales down proportionally.

**`/generate_final_pov` adds no extra API cost** — FFmpeg runs locally on your machine for free. Because the POV final cut is a static hard-cut with stream-copy concat (no re-encode, no zoom), the concat stage is near-instant; the only real CPU time is building the ~250 per-scene clips.

### Why GPT Image 2?

| | GPT Image 2 | Nano Banana 2 |
|---|---|---|
| Price (1K) | **$0.03/image** | $0.04/image |
| Character consistency | ✅ image-to-image mode | ⚠️ Limited |
| Aging a locked character across levels | ✅ per-scene description holds | ⚠️ drifts |

GPT Image 2 is cheaper AND better at keeping the protagonist recognizable across ~250 scenes while aging him level by level.

### Monthly Cost Estimate

| Videos per month | Monthly cost |
|---|---|
| 4 videos | ~$39 |
| 8 videos | ~$79 |
| 12 videos | ~$118 |

(Based on ~$9.85/video for ~25-min POV content with ~250 scenes.)

---

## File Checklist

- [ ] Python 3.12+ installed
- [ ] Git installed
- [ ] CapCut installed (Free or Pro — only needed if using `/generate_audio_pov` + manual assembly)
- [ ] FFmpeg installed and in PATH (required for `/generate_final_pov` — also provides `ffprobe` used by `/generate_audio_pov`)
- [ ] Poppins or Inter font installed (optional — for `/generate_final_pov` subtitle styling; Arial is the fallback if not installed)
- [ ] Claude Desktop App installed, signed into Claude Pro
- [ ] Kie.ai API key created, credits added to account
- [ ] ElevenLabs API key + Voice ID saved
- [ ] Protagonist image (bald, full face) uploaded to ImgBB and `PROTAGONIST_URL` saved in `channel_config.txt`
- [ ] `.env` file filled in with all 4 keys (OPENAI_API_KEY, ELEVEN_LABS_API_KEY, ELEVEN_LABS_VOICE_ID, ANTHROPIC_API_KEY)
- [ ] `channel_config.txt` filled in with channel name, `PROTAGONIST_URL`, and brand colors (no narrator name — this channel has no on-screen host)
- [ ] `pov-automation/` folder created with correct structure (separate from any finance-automation folder)
- [ ] 5 command files created in `.claude/commands/`: `generate_ideas_pov.md`, `generate_script_pov.md`, `generate_images_pov.md`, `generate_audio_pov.md`, and (optional) `generate_final_pov.md`
- [ ] Claude Code opened and pointing to the `pov-automation/` folder
- [ ] Bypass permissions mode turned on

---

## Troubleshooting

### "Unknown skill: generate_script_pov"
- Confirm the file is at `.claude/commands/generate_script_pov.md`
- Close Claude Code completely, wait 10 seconds, reopen
- Re-select your `pov-automation` folder

### generate_ideas_pov keeps repeating topics I've already done
- Open `videos/ideas_history.txt` and confirm past topics are logged correctly
- If the file is empty or missing, Claude has no history to reference — add past topics manually in the format `YYYY-MM-DD | topic title | thumbnail_line`
- Remember near-neighbors are also rejected: if you've done a submarine crew, an aircraft-carrier crew won't be offered. Keep `worlds_covered.txt` updated so whole categories rotate.

### "OPENAI_API_KEY is missing"
- Open `.env` and verify all keys are filled in with no extra spaces
- Confirm your Kie.ai account has credits — go to https://kie.ai/ → Billing

### The protagonist is not consistent across scenes
- Confirm `PROTAGONIST_URL` in `channel_config.txt` points to a live ImgBB direct link (ends in `.png`), and that `character_ref.png` exists locally in `character/`.
- Every `pov_scene` and `cost_scene` image_prompt must contain the **full** protagonist string — `protagonist_fixed_core` + that level's age layer + that level's costume — verbatim, not a bare pronoun or name. Open `current_script.json` and scan; if any are missing it, patch those prompts.
- **Protagonist has hair in some scenes** — this is the hard failure. He must be completely bald in every scene (he keeps his nose, ears, eyebrows, mouth). Reject those scene numbers and regenerate.
- **A secondary character went bald or wears the protagonist's face** — the inverse failure: a `world_scene` was routed to image-to-image, or a secondary's locked string is missing hair. Give every recurring secondary character hair in their `character_bank` string.
- **The protagonist didn't age** — compare a Level 2 image to a Level 9 image side by side. If they look the same age, the `character_bank` age layers are too subtle. Deepen them, then reject that level's scenes.

### A person appeared in an object scene (folded flag, empty locker, medal)
- `object_scene` images must contain **no people**. They must route through plain text-to-image with **no** protagonist reference attached. If a figure hallucinated in, check the route summary in the image script output — that scene was wrongly given a reference image. Reject and regenerate; confirm the scene's `character` field is `none`.

### The protagonist is smiling on a grief/death line
- He has a full mouth, so the model defaults to a pleasant one. Scan every `cost_scene` for a smile on a line about loss. The image_prompt's Expression field should specify the correct mouth/eyebrow/eye shape (e.g. grief: mouth turned down, eyebrows angled up at inner corners). Fix the expression and regenerate those scenes.

### "ffprobe not found" or "FFmpeg not found"
v38 uses `ffprobe` (which ships with the ffmpeg package) only to measure audio durations for the SRT file. No video encoding happens.
- Windows: Verify `C:\ffmpeg\bin` is in your PATH
- Mac: Run `brew install ffmpeg` in Terminal
- Verify by running `ffprobe -version` in a new terminal — if you see a version number, you're good

### ElevenLabs returns an error
- Double-check the API key and Voice ID in your `.env` file
- Verify your ElevenLabs account is active and has remaining character quota

### Final video exported from CapCut has audio/subtitle desync in the second half
This was the v37 → v38 reason for redesign. In v38 you should not see this anymore as long as you import the raw audio + raw images directly into CapCut (rather than importing a pre-rendered MP4). If desync still happens:
- Check that your CapCut project frame rate matches the export frame rate (set both to 30fps or both to 60fps — don't mix)
- Avoid setting individual image clip durations using CapCut's "Set Duration" with fractional values like 5.137s — use the audio clip as the master and align image right-edge to audio right-edge by dragging
- If using SRT subtitles: re-import `subtitles.srt` after the final timeline is laid out (any intro clips or transitions you added will shift the timing)

### Audio file missing for one scene after /generate_audio_pov runs
- Check terminal output for the ElevenLabs error message — usually rate limit or empty voiceover text
- Re-run `/generate_audio_pov` — it regenerates all audio files; existing ones get overwritten cleanly
- For partial re-runs of just one scene, you can manually re-run the Python snippet in STEP 2 with just that scene number

### Image generation takes longer than an hour
- This is expected at scale — a POV video is ~250 scenes. At default concurrency (3 workers) image generation runs ~55 minutes; the pipeline is the bottleneck, not a bug.
- GPT Image 2 API may be under heavy load — try again in a few minutes.
- If Kie.ai is not rate-limiting you, raise `MAX_CONCURRENT_SCENES` in the image script to speed things up; lower it to 1 if you hit rate limits.
- The script is resumable — it skips any scene that already has an image on disk, so a re-run after a failure only regenerates what's missing.

### On Mac: "Python command not found"
- Use `python3` instead of `python`
- Confirm Python 3.12+ is installed

---

## Your First Command

Open Claude Code with your `pov-automation` folder selected and type:

```
/generate_ideas_pov
```

Pick a topic from the list, then run:

```
/generate_script_pov "your chosen topic here"
```
