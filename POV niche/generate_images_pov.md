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

Three things break if you reuse the finance `/generate_images` script unchanged:

**1. There is a third `character` value.** Finance scripts only ever emit `"main"` or `"supporting"`. POV scripts also emit `"none"` for `object_scene` — a close-up of a folded flag, a medal in a shadow box, an empty locker. These scenes contain **no people at all**. If a character reference image is attached to them, the model will hallucinate a person into the frame. Object scenes must go through `text-to-image` with **no `image_input`**, exactly like supporting scenes, and the script below routes them that way.

**2. The protagonist's appearance is not constant.** In the finance pipeline, Jeff looks identical in every scene, so one global `CHARACTER_DESCRIPTION` string works. In POV the protagonist ages 40–55 years across ten levels. The script must therefore assemble the character description **per scene**, from three parts stored in the script JSON's `character_bank` block:

```
fixed_core  +  age_layer[scene.level]  +  costume[scene.level]
```

The `fixed_core` never changes and is what keeps the character recognizable. The age layer and costume are what make him age. This assembly happens at runtime, below.

**3. The reference image anchors a head shape, not a whole person.** The protagonist is bald and has **no nose** — but he does have a mouth, ears, and eyebrows. Those two absences, no nose and no hair, are the *only* things separating him from a secondary character. `MAIN_CHARACTER_REF` should be a clean, front-facing, neutral image of exactly that head on a plain body. It anchors head shape, eye spacing, and proportion. It does **not** anchor age or clothing — those come from the text.

## STEP 1 — Read Channel Config and Active Video Folder

Read `channel_config.txt` and extract:
- `PROTAGONIST_URL` — the ImgBB **public URL** of the protagonist reference image (bald, noseless, but with a mouth)

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
- For `"supporting"` → **never** pass `PROTAGONIST_URL`. That URL is the protagonist's bald noseless head; attaching it to a `world_scene` will strip the nose and hair off a secondary character who is supposed to have both. The prompt already carries each recurring character's full locked appearance string from the CHARACTER BANK — do not strip or shorten it. For recurring secondary characters the text description IS the only consistency anchor. "Generate freely" applies only to one-off background figures.
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
        "- The protagonist is completely bald (NO hair) and has NO nose at all.\n"
        "- He DOES have a full expressive mouth, visible ears, and thin dark eyebrows.\n"
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
- Next step: review images in pending/, copy approved ones to approved/, then run `/generate_audio` or `/generate_final` when all images are in approved/

## STEP 4 — What to Look For When Reviewing 250 Images

Reviewing 250 images is the real bottleneck. Do not scrub them in order — scan for these five failure modes, which account for nearly everything that goes wrong:

1. **The protagonist grew a nose or hair.** These are the two things he must never have. The model adds a nose by default because nearly every face it has ever seen has one — this is the single most common failure. Check every scene. He should still have a mouth, ears, and eyebrows; those are correct.

2. **The protagonist did not age.** Compare a Level 2 image against a Level 9 image side by side. If they look the same age, the `character_bank` age layers were too subtle. Fix the bank, then reject that level's scenes.

3. **A secondary character lost their nose or hair.** The inverse failure. The protagonist reference bled onto a `world_scene`. This means a `supporting` scene was routed to image-to-image — check the route summary.

4. **A person appeared in an object scene.** A hand holding the flag, a figure in the background. Object scenes must be empty of people.

5. **A smile on a grieving line.** He has a mouth, so the model will use it — and its default is a pleasant one. Scan every `cost_scene` for a protagonist smiling while someone dies. Reject those.

6. **Costume drift within a level.** The protagonist changes uniform between scene 88 and scene 91, both Level 5. The costume string for that level was too vague. Tighten it in `character_bank`.

Write failing scene numbers into `rejected.txt`, one per line, and re-run. The script deletes their stale pending images automatically so the retry actually regenerates them.
