---
name: gemini-api-cli
description: "TTS, image generation or conversational image editing, background music or audio track generation, video generation or editing, audio/video transcription and SRT captions, media/document analysis (PDFs, images, charts, audio, video, YouTube URLs), Deep Research. Use whenever you or the user needs TTS, image gen, music, video, transcription, multimodal analysis."
---

# gemini-api CLI

`gemini-api` is a stateless Go CLI over the Gemini Interactions API (v1beta). Each call is independent. You hold the IDs and file paths you want to reuse. Tested against v0.4.4 (beta; flags can change between releases, so check `--help` when something errors).

## Preflight (do this first, once)

```bash
command -v gemini-api && gemini-api version
gemini-api whoami            # shows credential source, masked
```

- Not installed: `curl -fsSL https://raw.githubusercontent.com/google-gemini/gemini-api-cli/main/scripts/install.sh | bash`, or `go install github.com/google-gemini/gemini-api-cli/cmd/gemini-api@latest`.
- Auth precedence: flags > env (`GEMINI_API_KEY`) > OS keychain > `~/.config/gemini-api/config.yaml`. Pass `--api-key` or set `GEMINI_API_KEY` for a one-off.
- Exit code 3 means the key is bad or missing. A replacement key must be created by the user in Google AI Studio. Do not run `configure` or loop retries yourself. Ask the user for a valid key.
- Never print or log the API key. `--dry-run` redacts it.

## Golden rules for agents

1. **Prompt goes first, flags after.** `gemini-api <cmd> "prompt" [flags]`. Quote prompts. For long or multi-line prompts use `--stdin` or `-f file`.
2. **Real result on stdout, progress on stderr.** Lines like `Analyzing 1 input(s)...` are stderr. Capture stdout only (`2>/dev/null` if noisy).
3. **Media commands print only the written file path** (`image`, `tts`, `music`, `video`, `transcribe`). Capture it: `P=$(gemini-api image "..." --out ./a.jpg)`.
4. **Always pass `--out`** with an explicit path inside the project, otherwise files land in the cwd as `gemini-<kind>-<ts>-<rand>.<ext>`.
5. **For parseable output use `--stream=false -o json`** (or `-q <jq>`). Streaming is the default for `generate` and `agent run`; with `-o json` it gives raw NDJSON events, not one object.
6. **Preview first when unsure:** `--dry-run -o json` validates flags and prints the request with no network call.
7. **Exit codes:** 0 ok, 1 runtime/API error, 2 usage/validation, 3 auth. Add `--agent-mode` (or `GEMINI_CLI_AGENT_MODE=1`) to get a JSON error envelope on stderr (`error_type`, `error_reason`, `exit_code`, `hints`) and TOON stdout by default. Combine with `-o json` if you need JSON stdout.
8. **Missing prompt with no TTY** exits 2 after dumping help. Use `--no-interactive` in automation so nothing prompts.
9. Costs real money and quota. Do not loop generation calls, and do not run Deep Research or video without a reason.

## Pick a command

| Goal | Command | Default model |
|---|---|---|
| Text answer, reasoning, code, JSON | `generate` | `gemini-3.8-flash` |
| Make or edit an image | `image` | `gemini-nano-banana-2.1` |
| Speech from text | `tts` | `gemini-3.8-flash-tts` (voice Kore) |
| Music | `music` | `lyria-3.5` |
| Video (text, image, edit, extend) | `video` | `gemini-omni-1.1-flash` |
| Questions about PDF/image/audio/video/CSV/YouTube | `analyze` | `gemini-3.8-flash` |
| Transcript or captions | `transcribe` | `gemini-3.8-flash` |
| Big files (>~14 MB) | `files upload` then reference `files/<id>` | n/a |
| Long autonomous research or coding task | `agent run --agent ... --background` | n/a |
| Anything the flags do not cover | `agent run --body @req.json` | n/a |

List models: `gemini-api models` (offline catalog), `models list --all` (live, what your key can use), `models get <id>`. Aliases `gemini-flash-latest`, `gemini-flash-lite-latest`, `gemini-pro-latest` track newest releases. Cheaper/faster text: `gemini-flash-lite-latest`. Hardest reasoning: `gemini-pro-latest` or `gemini-4-argon`.

## Recipes (all verified live)

### Text

```bash
gemini-api generate "Explain CRDTs in 2 sentences"                          # streams to stdout
gemini-api generate "Review this diff" --system "You are a strict reviewer" --stdin < change.diff
gemini-api generate "Summarize" -i report.pdf -m gemini-pro-latest

# One complete object, extract just the text
gemini-api generate "Say hi" --stream=false \
  -q '[.steps[]|select(.type=="model_output")|.content[].text]|join("")'
```

Streamed plain output may print an `Interaction ID: "..."` line after the reply. Use `--stream=false -q .id` to get the ID cleanly.

**Structured JSON output** (schema-constrained):

```bash
gemini-api generate "List 3 primary colors" --stream=false \
  --body '{"response_format":{"type":"text","mime_type":"application/json","schema":{"type":"object","properties":{"colors":{"type":"array","items":{"type":"string"}}},"required":["colors"]}}}' \
  -q '.steps[-1].content[0].text' | jq .
```

**Multi-turn**: capture `.id`, pass `--previous-interaction-id`:

```bash
T1=$(gemini-api generate "Remember 42" --stream=false -q .id)
gemini-api generate "What number?" --previous-interaction-id "$T1" --stream=false -q '.steps[-1].content[0].text'
```

Grounding and tools go through `--body` (see reference.md): `--body '{"tools":[{"type":"google_search"}]}'`. Other tool types: `url_context`, `code_execution`, `google_maps`, `file_search`, `mcp_server`, `function`, `retrieval`, `computer_use`. Thinking: `{"generation_config":{"thinking_level":"minimal|low|medium|high"}}`.

### Images

```bash
P=$(gemini-api image "product shot on white background" --aspect-ratio 16:9 --image-size 2K --out ./assets/hero.jpg)
gemini-api image "..." --out ./a.jpg -o json     # {kind, mime_type, path, response_id, size_bytes, status}
```

- `--image-size`: `512`, `1K`, `2K`, `4K`. `--aspect-ratio`: `1:1 2:3 3:2 3:4 4:3 4:5 5:4 9:16 16:9 21:9 1:8 8:1 1:4 4:1`.
- **Edit the image you just made** (conversational, preferred): get the ID with `--jq .response_id` on the first call, then:
  ```bash
  ID=$(gemini-api image "a blue square" --out ./sq.jpg --jq .response_id)
  gemini-api image "make it green" --previous-interaction-id "$ID" --out ./sq2.jpg
  ```
- **Edit or reference an existing file:** `-i ./photo.png` (repeatable) plus the instruction as the prompt.
- Better quality: `-m gemini-3-pro-image` or `nano-banana-pro-preview`. Output type: `--output-mime-type image/png`.
- `--out` with a directory (trailing `/`) auto-names the file with the right extension.

### Speech

```bash
gemini-api tts "Welcome." --voice Puck --out ./a.wav
gemini-api tts -f script.txt --out ./a.wav
gemini-api tts "Alice: Hi! Bob: Hello." --multi-speaker "Alice=Kore,Bob=Puck" --out ./d.wav
```

Output is always WAV (24 kHz mono); the extension is forced to `.wav` even if you ask for `.mp3`. Voices include Kore, Puck, Zephyr, Charon, Fenrir, Aoede. Hint language with `--language en-US`. `-m gemini-3.8-flash-lite-tts` is the fast/cheap option. Convert with ffmpeg if mp3 is needed.

### Music and video

```bash
gemini-api music "warm ambient pads, 90 bpm" --out ./bg/                 # lyria-3.5 full song; -m lyria-3-clip-preview for short clips
gemini-api video "ocean waves at dawn" --aspect-ratio 9:16 --resolution 1080p --out ./clips/waves.mp4
```

`video` blocks and polls until done. For long jobs, do not hold the shell open:

```bash
ID=$(gemini-api video "city timelapse" --async --jq .id)       # returns immediately
gemini-api video --resume "$ID" --out ./clips/city.mp4         # poll to completion and write file
```

Tune with `--poll-interval 2s --poll-timeout 15m`. Extend or edit a clip: `--previous-interaction-id <id>` (or `-i clip.mp4`) with an instruction like "now pan left". Resolutions: `360p 720p(default) 1080p 4k`.

### Analyze and transcribe

```bash
gemini-api analyze -i report.pdf "Key risks?"
gemini-api analyze -i a.png -i b.png "What changed?"
gemini-api analyze -i https://youtu.be/ID "Summarize, with timestamps"
cat notes.txt | gemini-api analyze -i chart.png --stdin "Does the chart support the notes?"
```

`analyze` requires at least one `-i` (exit 2 otherwise). It prints only the answer; the default question is "Describe this file in detail."

```bash
gemini-api transcribe -i call.mp3                                    # md with speakers + timestamps, prints path
gemini-api transcribe -i call.mp3 --format srt --out call.srt
gemini-api transcribe -i a.mp3 -i b.mp3 --out ./transcripts/         # multiple inputs => --out is a directory
gemini-api transcribe -i call.mp3 --format json --no-speakers --no-timestamps
```

Formats: `md` (default), `text`, `json`, `srt` (alias `captions`). Transcribe takes local paths and `files/<id>` only (not YouTube URLs).

### Large files (Files API)

Local inputs are sent inline up to ~14 MB raw (20 MB request cap). Above that, upload (48 h retention, 2 GB max, 50 MB for PDFs):

```bash
F=$(gemini-api files upload talk.mp4 --wait --jq .name)     # files/abc123, --wait blocks until ACTIVE
gemini-api analyze -i "$F" "Action items with timestamps"
gemini-api files get "$F" -q .state ; gemini-api files delete "$F"
```

Clean up uploads you no longer need. `files list --all`, `files generated-files-list` (alias `gfl`) exist too. `files register --uris '["gs://..."]'` needs an OAuth `--access-token`, not an API key.

### Managed agents and background runs

```bash
gemini-api agent list -q '.agents[].id'
# Deep Research (slow, many minutes, costly). Return immediately with an ID:
ID=$(gemini-api agent run "Compare battery recycling methods, cite sources" \
      --agent deep-research-preview-04-2026 --background --stream=false -q .id)
gemini-api agent status "$ID" -q .status        # in_progress | completed | failed | cancelled | requires_action ...
gemini-api agent status "$ID" -q '[.steps[]|select(.type=="model_output")|.content[].text]|join("")'
gemini-api agent status "$ID" --stream          # replay or follow events
gemini-api agent cancel "$ID"                   # only for still-running background runs
gemini-api agent delete-interaction "$ID"       # alias: di
```

Poll with `sleep` and backoff (10-30 s for research), stop on terminal status. Known `--agent` values: `deep-research-preview-04-2026`, `deep-research-max-preview-04-2026`, `deep-research-pro-preview-12-2025`, `antigravity-preview-05-2026`. `--agent` and `--model` cannot be combined. Custom agents: `agent create --base-agent <id> --id <name> -s "system instruction"`, plus `agent get|delete`.

## Full control with --body

Short flags merge with `--body`; duplicate keys are rejected. Discover the exact request shape instead of guessing:

```bash
gemini-api agent run --schema > schema.json     # JSON Schema (2020-12), also on generate/image/tts/music/video
gemini-api image --body @req.json --out ./x.jpg --dry-run -o json
```

`--body` accepts inline JSON, `@file.json`, or `@-` (stdin). Common fields: `system_instruction`, `tools`, `generation_config` (`temperature`, `top_p`, `seed`, `max_output_tokens`, `stop_sequences`, `thinking_level`), `response_format`, `previous_interaction_id`, `store` (false = do not retain server-side), `service_tier` (`flex|standard|priority|deferred`), `webhook_config`, `labels`.

## Output handling cheat sheet

| Need | Flags |
|---|---|
| Machine JSON | `-o json` (add `--stream=false` on `generate`/`agent run`) |
| Field extraction | `-q '.id'`; strings print raw (like `jq -r`) |
| Compact for LLM context | `-o toon` or `--agent-mode` |
| See HTTP request only | `--dry-run -o json` |
| Debug | `-d` (stderr diagnostics), `--include-headers` |
| Timeout | `--timeout 5m` (raise it for slow requests) |
| Self-describing | `--usage` (KDL tree of every command/flag), `--help-global` |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Exit 3 / `API_KEY_INVALID` | Key bad or missing. Run `whoami`; ask the user for a key. Do not retry. |
| Exit 2 `--input is required` | `analyze`/`transcribe` need `-i`. |
| `missing required argument <prompt>` | Pass a prompt, `--stdin`, or `--body`. |
| Request too large | File over ~14 MB: `files upload --wait`, then `-i files/<id>`. |
| `.mp3` requested from `tts` but got `.wav` | Expected; convert with ffmpeg. |
| JSON from `generate` is a stream of events | Add `--stream=false`. |
| 429 / quota | Back off, switch to a `-lite` model, or lower request volume. |
| Unknown flag after upgrade | CLI is beta; re-read `gemini-api <cmd> --help`. |
| Model name rejected | `gemini-api models list --all` for what your key can access. |

## Other command groups (rarely needed)

`environments` (sandbox environments, `files upload|download|list`, `connect`/`poll`/`serve` workers; `connect` runs `execute_bash` unsandboxed, so use `--disable-bash` or a container), `credentials` (server-side bearer/OAuth/env-var secrets injected by agents), `webhooks` (events: `batch.succeeded|expired|failed`, `interaction.requires_action|completed|failed`, `video.generated`), `triggers` (cron schedules for managed agents: `list|get|update --status active|paused|run|list-executions|delete`). Details, flags and sample bodies are in `reference.md` next to this file.
