# Changelog

All notable changes to this fork will be documented here.
Format loosely based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] — 2026-04-23

Initial release of **SD Model Toolkit Reloaded**, a fork of
[arenasys/stable-diffusion-webui-model-toolkit](https://github.com/arenasys/stable-diffusion-webui-model-toolkit)
(unmaintained since 2022).

This release collects compatibility fixes for SDXL and Forge / Forge Neo
alongside a new safetensors metadata editor.

### Added

- **Safetensors metadata editor.** A new *Safetensors metadata* accordion
  with `Author`, `Homepage`, and `Description` fields that appears once a
  model is loaded.
  - On load, existing `__metadata__` is read from the source file and shown
    in the fields (falling back to configurable defaults when absent).
  - On **Save** and **Export**, the current field values are embedded in the
    output `.safetensors` `__metadata__` header.
  - For `.ckpt` / `.pt` output the fields are ignored, preserving the
    previous pytorch-lightning training metadata for legacy loaders.
- SDXL AUX CLIP (`CLIP-XL-AUX-SD`) is now recognized by the
  `position_ids` fix-up path, so `Fix broken CLIP position IDs` correctly
  handles SDXL checkpoints.
- Documentation rewritten to cover SDXL, Forge / Forge Neo, and the new
  metadata workflow.

### Changed

- Forge / Forge Neo compatibility confirmed and documented.
- `save()` now passes user metadata through to
  `safetensors.torch.save_file(..., metadata=...)` instead of silently
  discarding it. Values are normalized to strings as required by the
  safetensors format.
- `load()` now reads the `.safetensors` `__metadata__` header via
  `safetensors.safe_open().metadata()` and returns it alongside the tensor
  dict. Previously the header was dropped on load.
- Settings section is now labelled **Model Toolkit Reloaded**. Option keys
  (`model_toolkit_fix_clip`, `model_toolkit_autoprune`) are unchanged, so
  upgrading from the original extension preserves existing settings.

### Fixed

- **Architecture detection on SDXL and many merged checkpoints.** The
  original toolkit treated `embeddings.position_ids` as a required CLIP
  key. When that buffer was missing — common for SDXL — detection fell
  through to a standalone `VAE-*-BROKEN` classification and then recommended
  catastrophic pruning (e.g. reducing a full SDXL checkpoint to a VAE-only
  file). These non-essential buffers are now treated as optional during
  component matching.
- **Fix broken CLIP position IDs** no longer injects SD 1.x CLIP keys into
  SDXL checkpoints (or vice versa). The fix-up only runs when the
  corresponding CLIP module is actually present in the model.
- Standalone `*-BROKEN` architectures no longer produce misleading junk-key
  estimates in the basic report, with a warning explaining that prune
  estimates are unreliable for partial checkpoints.

### Attribution

All credit for the original design, architecture fingerprinting, and
component key templates goes to the original author, arenasys.
