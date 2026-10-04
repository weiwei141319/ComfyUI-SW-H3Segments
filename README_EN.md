# ComfyUI-SW-H3Segments

English | [简体中文](README.md)

Multi-segment long-video nodes for **MiniMax H3 (海螺 H3)** in ComfyUI: generate a long clip as several prompt segments, **stitch them in latent space**, and run **one single VAE decode** at the end.

H3 tops out at roughly 7 seconds per shot. This pack pushes that to **15 – 35 seconds**, and the output **already has audio** — H3 is an audio+video joint model, so speech is generated straight from the prompt. No lip-sync pass, no manual audio alignment.

---

## Which module should I use?

| Module | Nodes | Use when |
|---|---|---|
| **All-in-one** (recommended) | `SW_H3MultiPrompt` | You want **a different prompt per segment**. Fill 3–5 prompt slots, get a 20–35 s clip. **No loop nodes needed.** |
| **Segment trio** | `SW_H3_SegPlan` / `SW_H3_SegBridge` / `SW_H3_SegConcat` | You want to hand-build the loop with official `StartLoop` / `EndLoop`, or inject custom nodes inside the loop. |

---

## Install

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/weiwei141319/ComfyUI-SW-H3Segments.git
```

Restart ComfyUI. No pip packages, no extra downloads.

**Requirements**

- A ComfyUI build that supports `comfy_api.latest` (`io.ComfyNode`, `io.Autogrow.Input`)
- Models: H3 UNET + Qwen3-VL CLIP + H3 **video** VAE + H3 **audio** VAE
- Optional: `ComfyUI-SW-Sampler` (SW companion nodes such as `SW_loadvideo`)

Nodes are filed under `SW/H3Segments`.

---

## Quick start

```
UNETLoader(H3)        ──────► model
CLIPLoader(Qwen3-VL)  ──────► clip
VAELoader(H3 video)   ──────► vae
VAELoader(H3 audio)   ──────► audio_vae     ★ required for sound
LoadImage ×N          ──────► ref_images

SW_H3MultiPrompt.video ──► CreateVideo.images
SW_H3MultiPrompt.audio ──► CreateVideo.audio
CreateVideo ──► SaveVideo
```

1. Fill `prompt_1` … `prompt_5` — **one segment per filled slot**
2. Wire reference images into `ref_images` (slots `ref_image_0`, `ref_image_1`, … grow automatically)
3. Reference them in the prompt as `<Picture 1>`, `<Picture 2>`, …
4. Run

---

## SW_H3MultiPrompt

### How the segment count is decided

Node inputs are fixed at graph-load time, so the node cannot detect "which slot has a wire". The rule is therefore **"which prompt slot holds non-empty text"**.

| Case | Segments |
|---|---|
| `prompt_1` filled | 1 |
| `prompt_1..4` filled | 4 |
| First empty slot stops the count | Anything after it is ignored (warned in report) |
| `cond_N` receives external CONDITIONING | Counts as a segment, and **takes priority** over `prompt_N` |

`segment_count` can be locked manually (`"1"` … `"5"`), which ignores empty slots.

> ⚠️ **auto resolving to fewer than 3 segments raises an error.** This node targets 15 s+ clips; 2 segments = 14.4 s. Set `segment_count` to `"1"` or `"2"` manually if you just want a quick test.

### Parameters

**Models**

| Param | Type | Note |
|---|---|---|
| `model` | MODEL | H3 weights (post-LoRA is fine) |
| `clip` | CLIP | Qwen3-VL. **Auto-unloaded after conditioning** (~16.5 GB freed) |
| `vae` | VAE | H3 **video** VAE — the single final decode |
| `audio_vae` | VAE | H3 **audio** VAE. **No audio_vae = silent output** |
| `cond_1..5` | CONDITIONING | Optional external conditioning, overrides `prompt_N` |
| `prompt_1..5` | STRING | Segment prompts. `prompt_1` required |

**References** (all Autogrow — slots grow as you wire them, no code change needed)

| Param | Max | Prompt tag |
|---|---|---|
| `ref_images` | **9** | `<Picture 1>` … `<Picture 9>` |
| `ref_videos` | **3** | `<Video 1>` … `<Video 3>` |
| `ref_video_audios` | **3** | Soundtrack paired with `ref_video_N`; needs `audio_vae` |
| `ref_audios` | **3** | `<Audio 1>` … `<Audio 3>`; needs `audio_vae` |

> **Use reference video OR standalone reference audio, not both.** A reference video already carries its soundtrack; stacking `ref_audios` on top feeds two competing audio latents into the DiT and the timbre fights itself.

**Canvas & timing**

| Param | Default | Note |
|---|---|---|
| `width` / `height` | 1344 / 768 | Multiple of 32. Portrait: `768 × 1344` |
| `segment_seconds` | 7.0 | Snapped to the `5+17n` grid (7 s → 175 frames = 7.29 s) |
| `anchor_frames` | `"5"` | Keep at 5 — guarantees the stitched total stays on the grid |
| `裁剪到秒数` | 0.0 | `0` = no crop. See below |

`裁剪到秒数` is an **upper bound** (it never stretches a short clip). It makes the **last segment shorter** so it fills exactly the remaining time, rather than generating a full segment and chopping its tail off — that is what used to eat the last line of dialogue. Video and audio are cropped together, so they stay in sync.

**Sampling**

| Param | Default | Note |
|---|---|---|
| `seed` | 0 | Start seed |
| `递增种子` | `True` | Segment 2 = `seed+1`, etc. Different noise per segment → smoother transitions. **Keep on** |
| `steps` | 20 | See tuning below |
| `cfg` | 1.0 | Official (no CFG). **Not** the purple-cast root cause — see tuning below |
| `sampler_name` / `scheduler` | `euler` / **`beta`** | ★ `scheduler` must be `beta`; `simple` causes the purple cast |
| `denoise` | 1.0 | |

**Conditioning details**

| Param | Default | Note |
|---|---|---|
| `ref_image_size` | `match` | `match` scales refs to canvas area (fast); `max` uses a 2048 short edge (best identity, much slower) |
| `visual_strength` | 0.999 | Reference latent fidelity. `1.0` if your refs are colour-correct |
| `audio_strength` | 1.0 | Ref audio latent fidelity |
| `内置sigma_shift` | `True` | Internal sigma shift (= official `MiniMaxH3SigmaShift`). **Turn off if you wired that node externally** |
| `shift_video` / `shift_audio` | 12.0 / 3.0 | Official defaults |
| `内部解码` | `True` | `False` → emit LATENT only, decode yourself |

### Outputs

| # | Name | Type |
|---|---|---|
| 1 | `video` | IMAGE |
| 2 | `latent` | LATENT |
| 3 | `段数` | INT — segments run |
| 4 | `成片帧数` | INT |
| 5 | `成片秒数` | FLOAT |
| 6 | `每段帧数` | INT |
| 7 | `报告` | STRING — full run report |
| 8 | `audio` | AUDIO → `CreateVideo.audio` |

### Reference tags in the prompt

Numbering follows **slot order**, not filename order:

```
<subject_definitions>
  <Subject 1> shown in <Picture 1>
  <Subject 2> shown in <Picture 2>
</subject_definitions>
<background>
  <Picture 3>
</background>
```

> There is **no `negative_prompt` input**. H3 negation goes in the `negative_prompt:` block of the prompt text (the 7th key of the H3 prompt format) — that is model-level textual negation, and it does work.

---

## Tuning

### Purple / magenta colour cast → three lines of defence

The most common H3 complaint.

> ⚠️ **This section was revised.** Earlier revisions blamed `cfg` and the VAE temporal chunk
> boundaries; both were disproven on real runs. Verified A/B: **switching `scheduler` from
> `simple` to the official `beta` eliminated the purple cast completely.**

H3's official sampling chain is `RandomNoise → BasicGuider → KSamplerSelect → BasicScheduler → SamplerCustomAdvanced`:

- `BasicGuider` → `Guider_Basic`, which only takes `model + conditioning` — it is
  **structurally unable to carry a negative**, and `CFGGuider.__init__` hard-codes
  `self.cfg = 1.0`. Officially, **CFG is never enabled**.
- `BasicScheduler` uses **`beta`**.

`simple` takes much larger strides across the high-noise region. Sigmas for 6 steps against the
H3 flow-matching curve:

| step | `simple` | `beta` (official) | `simple` deviation |
|---|---|---|---|
| 1 | 0.167 | 0.092 | **82% more noise** |
| 2 | 0.334 | 0.276 | 21% more |
| 3 | 0.501 | 0.500 | 0 |
| 4 | 0.667 | 0.725 | 8% less |
| 5 | 0.834 | 0.909 | 8% less |

A distilled LoRA (e.g. a 4-step turbo) already sits outside the original distribution; making the
first step jump 82% of the noise range pushes the latent out of distribution → chroma collapses
toward zero in **low-saturation areas (skin, sky, asphalt)** → purple cast. This is why it shows up
most in smooth, low-texture regions.

**Fix: keep `scheduler = beta`** (this node's default). That is the actual cure.

#### What about `cfg`

`cfg` is **not** the root cause, though it amplifies the symptom. This node's negative branch is
`_zero_out(cond)` — an **empty condition, not a negative prompt**. Substituting into
`comfy/samplers.py`'s blend formula:

```
cfg_result = uncond + (cond − uncond) × cfg
            = 0+ (cond − 0)      × cfg
            = cond × cfg
```

In other words `cfg < 1.0` **scales the whole denoised signal down every step** — attenuation, not
guidance.

| cfg | behaviour | advice |
|---|---|---|
| **1.0** | ComfyUI drops the negative branch, forwards cond only, **saves half the compute** | ✅ **official, use this** |
| 0.9 / 0.8 | attenuates the signal 10% / 20% per step; hides the symptom | emergency only, not a cure |
| < 0.7 | grey image, weaker likeness, sluggish camera | ❌ |

> Earlier revisions recommended "drop to 0.8" — that was **symptom suppression** before the root
> cause was found, not a fix. With `beta` you can go back to `cfg = 1.0`.

#### Ruled out

All of these were tested and eliminated — don't waste time on them:

| Suspect | Verdict | Evidence |
|---|---|---|
| **Bit depth / colour space** | unrelated | `auto + sRGB` *is* 8-bit in source; ffprobe shows identical `pix_fmt`/`color_space` on purple vs clean runs |
| **Resolution** | unrelated | at the same 544×960 some runs are purple and some clean; the purple peak lands on the same frames across 640×1152 and 768×1344 |
| **Fixed seed** | unrelated | single-pass workflows using `randomize` also go purple |
| **VAE temporal chunk boundaries** | not the main cause | the official decode already blends 5 frames; stretching to 13 introduces temporal jitter (reverted) |
| **Per-segment decode / concat strategy** | unrelated | the purple peak lands on VAE chunk boundaries, 32 frames away from segment boundaries (frames 158/311/464) |
| **int8 VAE** | no help | measured, no improvement |
| **`ColorTransfer` node** | backfires | mkl_lab drags the hue toward the pitch field; the whole clip turns green (node removed) |

### `steps` — match your LoRA

The default `20` targets the full model. **With a turbo/acceleration LoRA, 20 steps is over-sampling.**

| Setup | Steps |
|---|---|
| Full model | 15 – 25 |
| **4-step turbo LoRA** | **6 – 8** |
| 8-step acceleration LoRA | 10 – 12 |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| **No sound** | `audio_vae` not wired | Wire the H3 **audio** VAE. Speech is generated from the prompt; no reference audio needed |
| **Last line of dialogue cut off** | Tail chopping | Use `裁剪到秒数`; check the report for the `砍尾回退` fallback warning |
| **Purple cast** | `scheduler` was set to `simple` (oversized strides in the high-noise region push the latent out of distribution under a distilled LoRA) | Set `scheduler` to **`beta`** (the node default) and return `cfg` to 1.0 |
| **Giant faces in the first seconds** | Portrait head-shot refs + landscape canvas | Use full-body front-view refs, match aspect ratio, add a medium-shot constraint to the prompt |
| `No input node found for id [x] slot [y] ref_images.ref_image_3` | Stale workflow JSON vs. new Autogrow slots | **Regenerate the workflow.** Never hand-edit the JSON after changing the reference count |
| **OOM from the 16.5 GB CLIP** | Encoder not unloaded | The node unloads it automatically — check the report's 显存 line. Any external node on `CLIPLoader` must finish first |
| **Reference audio has no effect** | `audio_vae` missing | Fails **silently, no error** — easy to miss |
| `cfg` shows `NaN` / "value type error" | Hand-written `widgets_values` missing the `control_after_generate` slot | `KSampler` seed needs **7** values: `[seed, "fixed", steps, cfg, sampler, scheduler, denoise]` |

---

## Segment trio (manual loop)

| Node | Placement | Role |
|---|---|---|
| `SW_H3_SegPlan` | outside loop | Compiles "7 s × N segments" into parameters |
| `SW_H3_SegBridge` | **inside loop** | Injects the previous segment's tail tokens as keyframes (latent, not pixels) |
| `SW_H3_SegConcat` | outside loop | Concatenates N latents along time → official `VAE Decode` |

Wiring notes:

1. `SW_H3_SegPlan` 单段帧数 → `MiniMaxH3ReferenceToVideo.length`; wire width/height too (single source of truth)
2. Keep `MiniMaxH3ReferenceToVideo` **outside** the loop — `SegBridge` derives a copy per iteration, so every segment still carries the references
3. `EndLoop` needs **`accumulate` on**, with both `output_value` and `next_iteration_value` fed by the `KSampler` latent
4. `StartLoop.max_iteration` must equal `SW_H3_SegPlan.segment_count`

---

## How it works

Official constants:

```
comfy/ldm/minimax/model.py:30        FRAME_PER_TOKEN = (1, 4, 4, 4, 4)   5 tokens = 17 frames
comfy_extras/nodes_minimax_h3.py:37  legal frame count = 5 + 17n
comfy_extras/nodes_minimax_h3.py:43  video_latent_t = ((f-5)//17)*5 + 2
```

Stitching math: segment 1 keeps all 52 tokens (175 frames); each later segment drops its first 2 anchor tokens (170 frames). Total tokens = `50N + 2`, total frames = `170N + 5`, always on the `5+17n` grid.

| Segments | Frames | Duration | Tokens |
|---|---|---|---|
| 1 | 175 | 7.29 s | 52 |
| 2 | 345 | 14.38 s | 102 |
| 3 | 515 | 21.46 s | 152 |
| 4 | 685 | 28.54 s | 202 |
| 5 | 855 | 35.62 s | 252 |

Three things differ from the naive approach:

1. **Reference images encoded once**, shared by all segments (naive = N redundant encodes)
2. **CLIP unloaded once**, after all conditioning is built — frees ~16.5 GB
3. **Continuation happens in latent space, no VAE round-trip** — the last 2 tokens of the previous segment are injected as `minimax_keyframes` at `frame_idx=0`, equivalent to `MiniMaxH3AddGuide` but consuming latents instead of pixels

The H3 video VAE decodes chunked along time (`comfy_has_chunked_io=True`, 5-token chunks with 2-token overlap and linear blending), so any length decodes in one pass.

> **On "chunk boundaries lack cross-chunk context"**: the VAE already blends 5 frames
> (`blend()`), and measurements show the chunk boundary is not the main cause of the purple cast.
> Stretching the blend (`token_drop` 3→1) introduces temporal jitter, so the official chunking
> parameters are kept. **The root cause lives in the sampling layer's `scheduler`** — see tuning below.

---

## Known limits

- Decoding 855 frames at once buffers on CPU: 1344×768 fp32 ≈ 10.6 GB, 960×544 ≈ 5.4 GB. Test at low resolution first
- Reference video needs **at least 5 frames**; frame count is snapped to `n % 17 == 5`
- Segment-to-segment consistency relies on latent tail injection + reference conditioning; later segments may still drift slightly. Shorten segments or add references if it shows
- Audio latent is corrected against the exact frame count (1 frame = 5/3 audio latents), so there is no cumulative drift

---

## License

All rights reserved. No copying, redistribution, or commercial use without the author's written permission.
