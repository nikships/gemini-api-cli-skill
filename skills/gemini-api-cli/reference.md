# gemini-api reference (v0.4.4)

Source: https://github.com/google-gemini/gemini-api-cli. Verified against the installed binary's `--help` and `--schema`. Re-verify with `gemini-api <cmd> --help` and `gemini-api --usage` (full KDL tree) since the CLI is beta.

## Global flags

| Flag | Purpose |
|---|---|
| `--api-key`, `--access-token` | Credentials (`GEMINI_API_KEY`, OAuth bearer) |
| `--no-keyring` | Never touch OS keychain (`GEMINI_NO_KEYRING`) |
| `--api-version` (default `v1beta`), `--api-revision`, `--user-project` | API selection and quota project (`GEMINI_API_VERSION`, `GEMINI_API_REVISION`, `GEMINI_USER_PROJECT`) |
| `--server-url` | Override base URL |
| `-H, --header "K: V"` | Extra HTTP header (repeatable) |
| `--timeout 30s` | HTTP timeout |
| `-o, --output-format` | `pretty` (default), `json`, `yaml`, `table`, `toon` |
| `-q, --jq` | jq filter; `--raw-output` defaults true so strings print raw |
| `--include-headers` | Include response headers |
| `--color auto|always|never` | Respects `NO_COLOR`, `FORCE_COLOR` |
| `--interactive` / `--no-interactive` | Force or disable prompts and TUI |
| `--agent-mode` | JSON error envelope on stderr, TOON stdout default (`GEMINI_CLI_AGENT_MODE=1`) |
| `-d, --debug` | Request/response diagnostics on stderr |
| `--dry-run` | No network, no keychain; with `-o json`/`--jq` prints one JSON object per request |
| `--usage` | KDL schema of the whole CLI |

Config: `~/.config/gemini-api/config.yaml`, env prefix `GEMINI_`. `gemini-api configure --default-output-format json` stores a default format.

## Error envelope (`--agent-mode`)

```json
{"error_type":"validation_error|authentication_error|...","error_reason":"CLI_VALIDATION|API_KEY_INVALID|...","exit_code":2,"message":"...","hints":["..."],"status_code":400}
```

Hints can contain direct instructions (for example, to stop and ask the user for a key). Follow them.

## Interaction object shape (`--stream=false -o json`)

```
{ id, status, model|agent, object:"interaction", created, updated, service_tier,
  usage:{total_tokens,total_input_tokens,total_output_tokens,total_thought_tokens,...},
  steps:[ {type:"thought",signature}, {type:"model_output",content:[{type:"text",text}]}, ... ] }
```

Status values: `in_progress`, `requires_action`, `completed`, `failed`, `cancelled`, `incomplete`, `budget_exceeded`, `queued`. Other step types: `user_input`, `function_call`, `function_result`, `google_search_call/result`, `url_context_call/result`, `code_execution_call/result`, `mcp_server_tool_call/result`, `retrieval_call/result`, `file_search_call/result`, `google_maps_call/result`.

Final text: `[.steps[]|select(.type=="model_output")|.content[]|select(.type=="text")|.text]|join("")`.

`requires_action` means the model returned a `function_call` for your own declared function. Run it yourself and send a `function_result` step through `agent run --body` with `previous_interaction_id`.

## Request body fields (ModelInteraction / AgentInteraction)

`model` | `agent`, `input` (string or structured content), `system_instruction`, `tools[]`, `generation_config`, `response_format`, `response_mime_type`, `response_modalities`, `previous_interaction_id`, `background`, `stream`, `store`, `service_tier`, `safety_settings`, `cached_content`, `labels`, `webhook_config{uris,user_metadata}`, `environment`/`environment_id`, `agent_config`.

`generation_config`: `temperature`, `top_p`, `seed`, `max_output_tokens`, `stop_sequences`, `thinking_level` (`minimal|low|medium|high`), `thinking_summaries` (`auto|none`), `tool_choice{allowed_tools{mode,tools}}`, `image_config{aspect_ratio,image_size}`, `video_config{task: text_to_video|image_to_video|reference_to_video|edit|extend}`, `speech_config{voice,language,speaker}`, `transcription_config`.

`response_format` variants: `{"type":"text","mime_type":"application/json","schema":{...}}`, `{"type":"image","aspect_ratio":"16:9","image_size":"2K"}`, `{"type":"audio"}`, `{"type":"video"}`.

Tool objects (`type`): `google_search`, `url_context`, `code_execution`, `google_maps{latitude,longitude}`, `file_search{file_search_store_names,top_k,metadata_filter}`, `mcp_server{name,url,headers,allowed_tools}`, `function{name,description,parameters}`, `retrieval{...}`, `computer_use{...}`.

Deep Research `agent_config`: `{"type":"deep-research","collaborative_planning":false,"thinking_summaries":"auto","visualization":"auto|off","enable_bigquery_tool":false}`. Confirm exact keys with `gemini-api agent run --schema`.

Example `req.json` for a grounded answer:

```json
{"model":"gemini-3.8-flash","input":"Latest stable Go release and its headline features?",
 "tools":[{"type":"google_search"},{"type":"url_context"}],
 "generation_config":{"thinking_level":"low"},"store":false,"stream":false}
```

```bash
gemini-api agent run --body @req.json -q '[.steps[]|select(.type=="model_output")|.content[].text]|join("")'
```

## Per-command flags

**generate**: `-m`, `-i` (repeatable; path, `files/<id>`, YouTube URL), `--mime-type` (single input), `--stdin`, `--system`, `--stream[=false]`, `--previous-interaction-id`, `--body`, `--schema`.

**image**: `-m`, `-i`, `--mime-type`, `--stdin`, `--aspect-ratio`, `--image-size`, `--output-mime-type`, `--out`, `--previous-interaction-id`, `--raw-response`, `--body`, `--schema`. Models: `gemini-nano-banana-2.1` (default), `gemini-3.1-flash-image`, `gemini-3.1-flash-lite-image`, `gemini-3-pro-image`, `nano-banana-pro-preview`, `gemini-2.5-flash-image`. JSON result: `{kind,mime_type,path,response_id,size_bytes,status}`.

**tts**: `-f`, `--stdin`, `-v/--voice`, `--multi-speaker "A=Kore,B=Puck"` (turns must begin `A: ...`), `--language`, `-m`, `--out`. Models: `gemini-3.8-flash-tts`, `gemini-3.8-flash-lite-tts`, `gemini-3.1-flash-tts-preview`. JSON result: `{mime_type,model,path,size_bytes,usage}`. Expressive audio tags and director-style prompts are supported by 3.x TTS models; put style direction in the text.

**music**: `-m` (`lyria-3.5`, `lyria-3-pro-preview`, `lyria-3-clip-preview`), `-f`, `--stdin`, `--out`, `--raw-response`, `--body`.

**video**: `-m` (`gemini-omni-1.1-flash`, `gemini-omni-flash-preview`), `-i`, `--aspect-ratio`, `--resolution`, `--async`, `--resume <id>`, `--poll-interval`, `--poll-timeout`, `--previous-interaction-id`, `--out`, `--raw-response`, `--body`.

**analyze**: `-i` (required, repeatable), `--stdin`, `--system`, `-m`, `--mime-type`. Optional positional question.

**transcribe**: `-i` (required, repeatable, local or `files/<id>`), `--format md|text|json|srt`, `--no-speakers`, `--no-timestamps`, `--out`, `-m`, `--mime-type`. Prints one path per input.

**agent**: `run [input]` (`--agent` | `-m`, `--background`, `--stream`, `-i`, `--previous-interaction-id`, `--body`), `status [id]` (`--stream`, `-l/--last-event-id`, `--include-input`), `cancel [id]`, `delete-interaction|di [id]`, `create` (`--base-agent`, `-i/--id`, `-s/--system-instruction`, `--description`, `-t/--tools` JSON array, `-a/--agent-config` JSON, `--base-environment` JSON), `list` (`-a`, `--page-size`, `--page-token`, `--max-pages`), `get [id]`, `delete [id]`.

**files**: `upload <path>` (`--wait`, `--wait-timeout 5m`, `--display-name`, `--mime-type`), `list` (`-a`, `--page-size` max 100), `get [file]`, `delete [file]`, `generated-files-list|gfl` (`--page-size` max 50), `register --uris '["gs://..."]'` (OAuth token required).

**models**: bare = offline catalog; `list [-a]` live; `get [model]` shows token limits and supported methods.

**environments**: `create` (`-f/--from-environment`, `-n/--network` JSON, `-s/--sources` JSON array), `list`, `get`, `delete`, `files list -e <env> [-p path] [-r]`, `files upload <env> <local> [--path --extract --overwrite --mime-type]`, `files download <env> <path> [--out]`, `connect` (self-hosted worker; `--environment-key` or `GEMINI_ENVIRONMENT_KEY`, `--root-dir`, `--workdir`, `--max-concurrent 4`, `--disable-bash`, `--command-timeout 2m`), `poll`, `serve`. `execute_bash` in `connect` runs on the host unsandboxed. Run only in a container or VM, or pass `--disable-bash`.

**credentials**: `create|list|get|update|delete`. Bodies:
- `{"id":"github-token","type":"bearer_token","token":"..."}` (optional `header_name`, `prefix`)
- `{"id":"search-key","type":"environment_variable","value":"...","injection_location":"header"}`
- `{"id":"x","type":"oauth2","client_id":"...","client_secret":"...","refresh_token":"..."}`

Tokens are write-only. Never echo them back or commit them.

**webhooks**: `create -u <uri> -s '["interaction.completed","video.generated"]' [-n name]`, `list`, `get`, `update [id] (--uri --state enabled|disabled --subscribed-events --update-mask)`, `ping`, `rotate-signing-secret`, `delete`.

**triggers**: `list`, `get`, `update [id] (--display-name, -s/--status active|paused|error)`, `run [id]`, `list-executions`, `delete`. The CLI has no `triggers create`; use `--schema` on `update` to see editable fields.

## Pagination

`list` commands take `--all` (auto-paginate; NDJSON with `-o json`), `--max-pages N`, `--page-size N`, `--page-token T`.

## Safe-use checklist

- `--dry-run -o json` before any destructive call (`delete`, `cancel`, `rotate-signing-secret`).
- Do not delete resources you did not create in this session without being asked.
- Uploaded files auto-expire after 48 h, but delete large or sensitive ones when finished.
- `store:false` in `--body` stops server-side retention of the interaction when that matters.
- Pin the CLI version in CI (`go install ...@v0.4.4`); breaking changes happen between releases.
