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
