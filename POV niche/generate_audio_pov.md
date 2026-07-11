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
