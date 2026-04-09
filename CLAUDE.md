# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ComfyUI_Video_Utilities is a ComfyUI custom node extension for video processing and AI-driven subtitle generation. It is installed under `ComfyUI/custom_nodes/` and loaded automatically by ComfyUI at startup. There is no standalone build process — the plugin runs inside the ComfyUI Python environment.

## Development Setup

Install dependencies into the ComfyUI Python environment:

```bash
pip install -r requirements.txt
# Optional subtitle features:
pip install -r requirements_subtitle.txt
# Optional face restoration:
pip install -r requirements_face_restore.txt
```

FFmpeg must be installed and available on PATH (used by `server.py` and subtitle renderers). The plugin will also auto-detect FFmpeg from the VideoHelperSuite (VHS) extension if present.

Models are auto-downloaded from Hugging Face on first use (controlled by `config.py:AUTO_DOWNLOAD_MODELS`). For users in China, set `USE_HF_MIRROR = True` in `config.py` to use `hf-mirror.com`.

## Testing

There is no test suite. Test by loading the plugin in a running ComfyUI instance and executing workflows. Check ComfyUI's terminal output for `[Video Utilities]` prefixed log messages.

## Architecture

### Entry Point and Node Registration

`__init__.py` is the ComfyUI entry point. It calls `load_modules()` which:
1. Registers server HTTP routes from `server.py`
2. Loads legacy nodes from `Video_Utilities.py` into `NODE_CLASS_MAPPINGS`
3. Loads new modular nodes from `nodes/` into `NODE_CLASS_MAPPINGS`

ComfyUI reads `NODE_CLASS_MAPPINGS`, `NODE_DISPLAY_NAME_MAPPINGS`, and `WEB_DIRECTORY` from `__init__.py`.

### Two Codebases in One

**Legacy (monolithic):** `Video_Utilities.py` (~6400 lines) contains all original nodes: `Video_Stitching`, `Video_To_GIF`, `Get_First/Last_Frame`, `Video_Preview`, `Upload_Live_Video`, `Load_AF_Video`, `Live_Video_Monitor`, `Preview_GIF`, `Prompt_Text_Node`, `RGB_Empty_Image`, `Get VHS File Path`, and the old `Audio_To_Subtitle`.

**New (modular):** `nodes/` contains the v2 nodes with shared utility engines in `utils/`:
- `nodes/audio_to_text.py` → `Audio_To_Text` node (ASR via faster-whisper or transformers)
- `nodes/text_to_video_static.py` → `Text_To_Video_Static` (whole-sentence subtitles)
- `nodes/text_to_video_dynamic.py` → `Text_To_Video_Dynamic` (word-by-word, TikTok-style)
- `nodes/text_to_video_scrolling.py` → `Text_To_Video_Scrolling` (rolling credits)
- `nodes/color_picker.py` → `Color_Picker`

### Utility Engines (`utils/`)

| Module | Purpose |
|--------|---------|
| `asr_engine.py` | Wraps faster-whisper; handles model download, chunked audio, word timestamps |
| `subtitle_renderer.py` | Core PIL-based frame renderer; handles positioning, stroke, vertical text |
| `scrolling_renderer.py` | Specialized renderer for scrolling text effects |
| `animation.py` | 19 animation effects (fade, slide, zoom, typewriter, wave, etc.) applied per-frame |
| `text_wrapper.py` | Smart line-breaking for Chinese/English mixed text |

### ComfyUI Node Pattern

Every node class must define:
- `INPUT_TYPES()` — classmethod returning parameter schema
- `RETURN_TYPES` / `RETURN_NAMES` — output tuple types and names
- `FUNCTION` — name of the method ComfyUI calls
- `CATEGORY` — UI grouping string

### Server Routes (`server.py`)

Registers `aiohttp` routes on ComfyUI's server for:
- Video transcoding to H.264 for browser preview (handles Topaz and other exotic codecs)
- Video file serving with range request support
- Path safety validation (prevents directory traversal)

FFmpeg is located by checking VideoHelperSuite's bundled binary first, then falling back to system PATH.

### Frontend (`js/`)

JavaScript files in `js/` are served as ComfyUI web extensions via `WEB_DIRECTORY = "./js"`. Each `.js` file typically registers a custom widget or UI behavior for a specific node.

### Configuration (`config.py`)

Central defaults for: model auto-download, HF mirror, ASR engine selection, subtitle rendering (font, color, animation), video encoding (codec, CRF, audio bitrate). Modify here to change defaults without touching node code.

### Fonts

Place `.ttf`, `.ttc`, or `.otf` font files in `Fonts/`. The default is `SourceHanSansCN-Bold.otf`. Fonts are loaded by `subtitle_renderer.py` using PIL's `ImageFont`.

## Typical Data Flow

```
Video input (Upload_Live_Video / Load_AF_Video)
  → Audio extraction (FFmpeg in asr_engine.py)
  → ASR transcription (faster-whisper → word timestamps)
  → Subtitle node (Static / Dynamic / Scrolling)
      → subtitle_renderer.py + animation.py (per-frame PIL rendering)
      → OpenCV VideoWriter → output .mp4
  → Video_Preview (server.py transcodes if needed → browser)
```
