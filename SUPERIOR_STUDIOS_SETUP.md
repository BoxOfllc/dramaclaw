# DramaClaw — Superior Studios Wiring Notes

Local setup notes for running this fork against our own infrastructure instead of RelayClaw. Not part of upstream DramaClaw — ours only. No secrets live in this file; fill those into `.env` and the DramaClaw web UI locally, never commit them.

**Status as of 2026-09-15:** Repo cloned from `https://github.com/BoxOfllc/dramaclaw.git` to `E:\superior-studios\tools\dramaclaw`. `.env` created from `.env.example` with the media relay switched from Aliyun OSS (China-only) to Cloudinary. Everything below this point requires Docker actually running on a real machine — this session's sandbox has no Docker, so none of it has been executed or tested yet. Treat this as a checklist, not a confirmation that it works.

---

## Why Custom mode, not Official/RelayClaw

Confirmed by reading the repo's own docs (`docs/en/getting-started/configuring-models.md`): DramaClaw needs an "OpenAI-compatible gateway" for every model call. Official mode routes that through RelayClaw (paid, pricing not published anywhere I could find). **Custom mode** routes it through a bundled local NewAPI instance that you point at your own providers instead — no RelayClaw account needed at all. That's the mode we're using.

Custom mode needs two separate things wired up:
1. **Text/vision models** (script generation, scene/character extraction, prompt optimization) — needs a real LLM provider.
2. **Image/video models** — this is where local ComfyUI plugs in directly.

## What's already done

- Repo cloned.
- `.env` created, Cloudinary set as the reference-media relay provider (swap for Aliyun OSS only if we ever have a reason to — see the note left in `.env`).
- Confirmed this repo already bundles three working WAN 2.2 (LightX2V) ComfyUI workflow exports at `src/novelvideo/generators/`:
  - `wan2-2-I2V-LightX2V.json` — image-to-video, full-precision UNETs
  - `wan2-2-I2V-GGUF-LightX2V.json` — same, GGUF-quantized (lower VRAM)
  - `wan2-2-FLF-LightX2V.json` — first-last-frame video
  
  These match what our AI Dev Team pipeline (`ip-director` / `wan-video-pipeline` agents) already standardized on for HISTORYFRIENDS, which is good — one model family across both pipelines instead of two.
- Inspected `wan2-2-I2V-LightX2V.json` directly to pull the exact custom-node and model-file manifest ComfyUI needs (see the HTML setup guide for the full list) — this is what the "matching environments" guide for Will's machine is built from.

## What's still needed — action items for Jon

1. **Cloudinary account** (free tier is fine) — fill the four `CLOUDINARY_RELAY_*` values in `.env` from the dashboard.
2. **An LLM provider key for text/vision.** DramaClaw's Custom-mode "Recommended" profile defaults to OpenRouter, which is the simplest option — one key, access to most major models, pay-as-you-go. Get a key at openrouter.ai and paste it into the DramaClaw Settings UI (not `.env` — text/vision provider keys live in the web UI, per the docs, stored in the local NewAPI SQLite DB, not environment variables).
3. **Docker Desktop running**, then from `E:\superior-studios\tools\dramaclaw`:
   ```
   docker compose up -d
   ```
   This starts the DramaClaw API, the web frontend (`http://localhost:8080`), and the bundled NewAPI gateway.
4. **First-run setup in the browser** at `http://localhost:8080` → Settings → Models & Channels:
   - Open **Custom**, set a NewAPI root password (8+ chars), click **Initialize Local NewAPI**.
   - Apply the **Recommended** profile, paste the OpenRouter key when prompted, **Save & Apply All**.
   - Click **Manage Channels**, add **ComfyUI**. Set:
     - Model name: `WAN2.2-LightX2V-local` (or whatever we want it labeled as in the DramaClaw UI)
     - **ComfyUI service URL — this is the one gotcha to get right:** DramaClaw's backend runs inside a Docker container. `127.0.0.1` from inside that container means the container itself, not the Windows machine running ComfyUI. On Docker Desktop for Windows, use **`http://host.docker.internal:8188`** instead of `http://127.0.0.1:8188`. This is confirmed directly in the repo's own troubleshooting table, which flags exactly this failure mode.
   - Under that channel's **ComfyUI Workflows** area, add the three bundled workflow JSONs from `src/novelvideo/generators/` (upload each as its own Workflow ID — they're already in ComfyUI API format, which is what this UI requires; browser-exported workflow JSON is explicitly not accepted).
   - Enable **Custom** mode.
5. **Run one test episode through the full path before committing production time to it** — this validates the cost estimate given to Jon earlier (working range was $150-$400/episode, extrapolated from a comparable commercial pipeline's ~$1,000/episode benchmark, not yet verified against our own setup) and confirms vertical/portrait output actually works, since nothing in DramaClaw's own docs mentions aspect ratio — that's a parameter on the underlying WAN workflow, not DramaClaw itself, and needs a live check.

## Open items / things I couldn't verify without live infrastructure

- Whether the bundled WAN workflows output portrait (9:16) by default or need the resolution/aspect fields edited in DramaClaw's media-model capability settings after import.
- Actual VRAM headroom needed for the WAN2.2 A14B model at fp8 with the block-swap node in play — the HTML setup guide gives the honest minimum/recommended split, but real usage should be confirmed on Jon's actual card before assuming it'll run the same way on Will's.
