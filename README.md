# SD Model Toolkit Reloaded

A multipurpose toolkit for managing, editing and creating Stable Diffusion models.
Works with Automatic1111 WebUI, Forge, and Forge Neo.

## Install

Extensions tab → Install from URL → paste this repo's URL → Install.

## Features

- Cleaning / pruning models (FP32 → FP16, strip junk keys, strip EMA).
- Converting between `.ckpt` / `.pt` / `.safetensors`.
- Extracting and replacing model components (UNET, VAE, CLIP, ControlNet, Depth).
- Identifying and debugging model architectures — SD 1.x, SD 2.x (incl. Depth / Inpainting),
  SD-XL (base + Refiner), Pix2Pix, LoRA, ControlNet.
- Editing the safetensors `__metadata__` header (author / homepage / description) directly
  when saving or exporting.

## Typical workflow — replace a broken VAE

Many checkpoints ship with a merged/broken VAE and a separate `*.vae.pt`
sidecar that you're supposed to load on top. You can fold it back in once
and keep a single clean file.

1. Select the checkpoint from the **Source** dropdown, press **Load**.
2. Switch to the **Advanced** tab.
3. Pick the **VAE-v1** class (leave component on `auto`).
4. Pick the external VAE file in the **Import** dropdown, press **Import**.
5. (Optional) Open the *Safetensors metadata* accordion and fill in author / homepage /
   description.
6. Change the output name if needed (defaults to `.safetensors`), press **Save**.

## Typical workflow — build a model from components

You can also rip a checkpoint apart and rebuild it, useful for swapping
multiple components at once or for debugging.

1. Load a source checkpoint.
2. On the **Advanced** tab, select a class (e.g. `UNET-v1`) and press **Export** — this
   writes `model-name.unet.pt` into `models/Components/`. Repeat for `CLIP-v1`.
3. Press **Clear**.
4. Pick **NEW SD-v1** (or `NEW SD-XL`, etc.) from the **Source** dropdown and press
   **Load**. This starts an empty shell for the chosen architecture.
5. For each class (`CLIP-v1`, `UNET-v1`, `VAE-v1`), pick the exported file and press
   **Import**.
6. Name the model, press **Save**.

## Safetensors metadata

When a model is loaded, an accordion labelled *Safetensors metadata* appears
with **Author**, **Homepage**, and **Description** fields.

- On load, the existing `__metadata__` header is read from the source file
  and shown in the fields (falling back to sensible defaults if the header
  is empty, which is the case for most merged models).
- On **Save** or **Export**, whatever is in the fields at that moment is
  embedded in the output file's `__metadata__` header. This matches what
  tools like Civitai and kohya-ss read when they scan a model.
- For `.ckpt` / `.pt` output the metadata fields are ignored — those
  formats don't have a standard header.

Leave a field blank to omit it from the header.

## Advanced tab

Shows a detailed architecture report, including:

- All matched architectures, and for each one the detected classes and components.
- Rejected architectures with the specific reason (e.g. missing keys — useful for
  figuring out why a model won't load).
- A full list of unknown keys that don't match any known component.
- The model's **metric** string — see below.

You can also `Export` a single component (CLIP, VAE, UNET, EMA UNET, Depth, ControlNet)
from any loaded model, or `Import` a component from any other file to replace it in
the currently loaded model.

## Autopruning

Toggle **Enable Autopruning** in Settings → Model Toolkit.

On WebUI startup, every model in `models/Autoprune/` is pruned to FP16
`.safetensors` (VAEs go to `.pt`) and moved into its proper folder (`Stable-diffusion/`,
`VAE/`, or `Components/`). Broken or unknown models are moved to `Autoprune/Failed/`.
Clicking **Reload UI** re-triggers the pass. Existing filenames are preserved; conflicts
get suffixed (`NAI.safetensors` → `NAI(1).safetensors`).

## Metric

During analysis the toolkit computes a three-part fingerprint per model:
`UNET / VAE / CLIP`, e.g. `2020/2982/0130`.

The toolkit knows the metrics of a handful of common components and calls
them out in the report — "Uses the NAI VAE" for `2982`, for example. A lot
of VAEs being distributed are just the NAI VAE under another name; the
metric makes that obvious. It's not foolproof — collisions happen — but
it's useful for sanity-checking what you're working with.

## Notes

### Components

Stable Diffusion needs three things to run: a **VAE**, a **UNET**, and a **CLIP** text
encoder. Full checkpoints contain all three. The toolkit will still happily recognize
incomplete checkpoints and flag them as e.g. `UNET-v1-BROKEN` + `VAE-v1-BROKEN`, which
you can then export normally to salvage the pieces.

The WebUI expects a checkpoint to be complete. If a component is missing the WebUI will
either keep using whatever was last loaded, or — if this is the first model — leave it
uninitialized and produce NaN errors.

Rule of thumb on size (FP16): a complete SD 1.5 checkpoint is ~2 GB, SD-XL is ~6.5 GB,
SD-XL Refiner is ~6 GB. Anything significantly smaller is probably missing a component.

### Precision

The WebUI casts all loaded models to FP16 by default. Without `--no-half` / `--no-half-vae`,
FP16 and FP32 checkpoints produce identical outputs — so saving as FP16 halves disk size
for free. FP32 only matters if you explicitly disable the cast.

### EMA

EMA data is stored so trainers can pause and resume fine-tuning without losing momentum.
If you're distributing a checkpoint intended for further training, keep the EMA in.

After any **merge**, the EMA no longer reflects the actual UNET and becomes dead weight
(and actively harmful for further training). The toolkit flags EMA data and strips it by
default when saving.

The EMA is itself a functional UNET — if you like it better than the main UNET in a
checkpoint, export the `UNET-v1-EMA` component and import it back as the regular UNET.

### CLIP position IDs

Merging often corrupts a CLIP tensor called `embeddings.position_ids`. This is a fixed
int64 buffer of `[0, 1, 2, …, 76]` — not something that should ever change. After a
merge it often ends up as `75.9975` etc., which then casts back to `75` at load time.

The toolkit reports how many positions are affected. In practice this is cosmetic for
SDXL (the CLIP loaders regenerate the buffer at runtime) and only slightly changes
output for SD 1.x. If you want to clean it up anyway, enable **Fix broken CLIP
position IDs** in settings, reload the model, and save. The fix replaces the buffer
with the correct `[0..76]` tensor.

**Don't panic if 40–50 positions are flagged on a merged SDXL checkpoint** — that's
the normal pattern, not a sign of a ruined model.

### VAE

The UNET tolerates merging reasonably well. The VAE does not — any merge between
different VAEs tends to produce a broken latent space. This is why merged models
often ship with a separate "good" VAE and instructions to swap it in.

Note that swapping the VAE isn't a perfect fix either. The UNET was trained against
a specific latent space, and there's no reason that space interpolates linearly with
weights. In practice, VAEs happen to be similar enough (by design) that using one of
the original reference VAEs works well enough.

The effect of merging on CLIP is less well understood but evidently less destructive.
