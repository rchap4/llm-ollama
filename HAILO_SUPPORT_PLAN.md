# Plan: Support the HailoAI build of Ollama (hailo-ollama)

This plan describes what it takes to make `llm-ollama` work against the
**Hailo Ollama server** (`hailo-ollama`, shipped in the Hailo Model Zoo GenAI
package for the Raspberry Pi AI HAT+ 2 / Hailo-10H NPU).

Sources used for this analysis:

- Hailo server implementation:
  [`hailo-ai/hailo_model_zoo_genai`](https://github.com/hailo-ai/hailo_model_zoo_genai),
  specifically `src/library/controller/controller.cpp`, `src/library/dto/DTOs.hpp`
  and `src/library/controller/llm_generation_callback.cpp` (read at the current
  `main` branch).
- Hailo/Raspberry Pi setup docs:
  [Raspberry Pi AI software — "Run LLMs on Raspberry Pi 5"](https://www.raspberrypi.com/documentation/computers/ai.html#step2-llm).
- Reference Ollama REST API: [docs.ollama.com/api/chat](https://docs.ollama.com/api/chat).

## 1. What hailo-ollama is

`hailo-ollama` is **not** a fork of the Ollama Go server. It is a
C++ (oatpp) HTTP server on top of HailoRT that **imitates a subset of the
Ollama REST API**. It listens on **port 8000** by default (not 11434) and runs
models compiled to Hailo HEF format from the Hailo Gen-AI Model Zoo. It has no
CLI; models are pulled via the REST API.

Endpoints it implements (from `controller.cpp`):

| Endpoint              | Present | Notes                                             |
|-----------------------|---------|---------------------------------------------------|
| `GET /api/version`    | yes     | Always reports `"0.5.1"` (hard-coded)             |
| `GET /api/tags`       | yes     | **No `digest` value** (see below)                 |
| `GET /hailo/v1/list`  | yes     | Hailo-specific catalog of downloadable models     |
| `GET /api/ps`         | yes     | Only ever one loaded model (single-model server)  |
| `POST /api/show`      | yes     | **No `capabilities` value** (see below)           |
| `POST /api/pull`      | yes     | HEF download, streaming progress                  |
| `DELETE /api/delete`  | yes     |                                                   |
| `POST /api/generate`  | yes     |                                                   |
| `POST /api/chat`      | yes     | Tools supported if the model template allows them |
| `POST /v1/chat/completions` | yes | Non-streaming only                             |
| `POST /api/embed`     | **no**  | Embeddings are not available at all               |

## 2. Behavioral differences that affect llm-ollama

### 2.1 `/api/tags` returns no `digest`

`ModelInfoShort` (DTOs.hpp) declares a `digest` field, but
`MyController::get_model_info()` never sets it. A typical response is:

```json
{"models": [{
    "name": "qwen2:1.5b", "model": "qwen2:1.5b",
    "size": 1234567890, "modified_at": "2024-01-15T10:30:00Z",
    "details": {"parameter_size": "1.5B", "quantization_level": "A8W4"}
}]}
```

The `ollama` Python client parses this fine (`Model.digest: Optional[str] = None`
becomes `None`). The breakage is entirely in our plugin:

- `llm_ollama/commands.py:42` — `model["digest"][:12]` raises
  `TypeError: 'NoneType' object is not subscriptable`. This is the crash seen
  when following the README ("about line 35 of command.py").
- `llm_ollama/registry.py:82-89` — `_distinct_models()` groups names by
  `digest`; with `None` every model collapses into one bucket, and the
  capability cache is keyed on the string `"None"`, so all models would share
  one cache entry even if capabilities worked.
- `llm_ollama/registry.py:135-139` — the `@cache(key="digest")` decorator
  caches `get_ollama_model_capabilities` by digest. On Hailo that key is
  meaningless.

### 2.2 `/api/show` returns no `capabilities`

`ShowResponse` (DTOs.hpp) declares `capabilities`, but `MyController::show()`
never sets it (it returns `license`, `modelfile`, `parameters`, `details`,
`model_info`, `modified_at` — and `parameters`/`modelfile` are empty strings).
There is also no `chat_template` field in the current DTO.

Consequence for `llm_ollama/registry.py:151`
(`get_client().show(name).capabilities or []`): **every model reports zero
capabilities**, so `register_models()` registers nothing and
`register_embedding_models()` registers nothing. Even after fixing the digest
crash, `llm ollama models` would print an empty table and `llm -m ollama/...`
would fail with "unknown model". This is the cascade the digest crash hides.

### 2.3 Generation semantics

From `controller.cpp` and `llm_generation_callback.cpp`:

- **Options**: only `temperature`, `seed`, `top_k`, `top_p`,
  `frequency_penalty`, `num_predict` are read (`apply_user_options`).
  `temperature == 0` disables sampling. Everything else the plugin can send
  (`num_ctx`, `stop`, `repeat_penalty`, `repeat_last_n`, `min_p`) is silently
  ignored.
- **Usage**: the final chat chunk carries `total_duration` and `eval_count`
  only. There is **no `prompt_eval_count`**, so input token counts are
  unavailable.
- **Thinking**: there is no `thinking` field anywhere in the request or
  response DTOs, and no `think` request parameter. Reasoning-style models
  (e.g. `deepseek-r1:1.5b`) emit `<think>...</think>` inline in `content`.
- **Structured output**: `ChatParams` has a `format` field but `chat()` never
  reads it — `format`/JSON schema requests are silently ignored.
- **Tools**: supported on `/api/chat` when the model's chat template declares
  tool support; otherwise the server returns HTTP 400
  `"<model>" does not support tools`. There is no way to *query* tool support.
- **Vision/audio**: the `ChatMessage` DTO has only `role`, `content`,
  `tool_calls`, `tool_name` — **no `images` field** — so attachments cannot be
  sent through `/api/chat`, even for VLM models.
- **Embeddings**: no endpoint at all.
- **Decision models**: no such concept.
- **Single-model server**: one model is loaded at a time
  (`SyncGenerationContext::lock()`); requests serialize behind the lock.
- **`keep_alive`** is an integer number of seconds, not Ollama's `"5m"` string
  (the plugin does not currently send it, so this only matters for future work).

## 3. Fix plan

The guiding principle: **tolerate missing fields everywhere, detect the server
flavor once, and degrade features gracefully instead of crashing or silently
registering nothing.**

### Phase 1 — Make discovery digest-optional (fixes the reported crash)

Small, safe changes that also keep working against real Ollama:

1. `llm_ollama/commands.py` — replace both `model["digest"]` uses with
   `model.get("digest")` and render a placeholder:

   ```python
   capabilities = get_ollama_model_capabilities(
       model.get("digest"), model["model"]
   )
   ...
   {
       "model": model["model"],
       "digest": (model.get("digest") or "-")[:12],
       "capabilities": ", ".join(sorted(capabilities)),
   }
   ```

2. `llm_ollama/registry.py` — in `_distinct_models()`, fall back to the model
   name as the grouping key when `digest` is missing, so distinct names stay
   distinct models:

   ```python
   for model in get_ollama_models():
       name = model["model"]
       digest = model.get("digest") or name   # Hailo: no digest; name is unique
       names_by_digest[digest].append(name)
   ```

3. Same fallback for the cache key: `get_ollama_model_capabilities()` already
   receives the digest argument; with step 2 it simply receives the model name
   on Hailo, which is a correct cache key (a Hailo name maps to exactly one
   HEF). No change to `cache.py` is required beyond documenting that the key
   may be a name. Bump `Cache.CACHE_VERSION` if stale `"None"`-keyed entries
   from the buggy version need invalidating.

### Phase 2 — Detect the server flavor

Add a small, cached probe so the rest of the code can branch cleanly:

```python
# llm_ollama/registry.py
@cache("server_flavor", key="host")
def is_hailo_server() -> bool:
    """True if the server exposes the Hailo-specific /hailo/v1/list endpoint."""
    try:
        get_client()._request("GET", "hailo/v1/list")  # or httpx GET f"{host}/hailo/v1/list"
        return True
    except Exception:
        return False
```

- Keyed by `OLLAMA_HOST` so a user can talks to both servers.
- `/api/version` cannot be used: hailo-ollama hard-codes `"0.5.1"`.
- The Hailo-specific `/hailo/v1/list` (returns `{"models": ["qwen2:1.5b", ...]}`)
  is an unambiguous fingerprint, as is the root `GET /` body
  `"hailo-ollama is running"`.

### Phase 3 — Capability strategy on Hailo (the real cascade fix)

Since `/api/show` reports no capabilities, the plugin must decide what each
model can do on its own:

- **completion**: assume every installed model is a chat model. This is true by
  construction — the Hailo Model Zoo only ships generative LLM/VLM models, and
  `/api/chat` is the primary endpoint.
- **tools**: cannot be queried. Two options, in order of preference:
  1. **Probe once per model (recommended)**: send a minimal `/api/chat`
     request carrying a trivial tool with `stream: false`, `num_predict: 1`.
     HTTP 400 `"does not support tools"` → no tools; HTTP 200 → tools. Cache
     the result per model name (the existing `Cache` works — key on name).
  2. **Opt-in override**: an environment variable such as
     `OLLAMA_HAILO_TOOLS=1` (or a per-model list) to force tools on, for users
     who do not want the probe. The probe should be the default because the
     plugin's registration is lazy and the probe is one tiny request per model
     per cache lifetime.
- **vision / audio**: never advertise — the wire format cannot carry images
  (no `images` field in the DTO). If a future Hailo release adds an images
  field, this can be re-probed the same way as tools.
- **thinking**: never advertise. See Phase 4 for `<think>` handling.
- **embedding**: skip registration entirely on Hailo, and make
  `llm embed -m ollama/...` fail with a clear message. Optionally probe
  `POST /api/embed` once and warn loudly if a server that claims to be Ollama
  lacks it.
- **decision models**: nothing to do — the `decision` capability can never be
  reported, so `OllamaDecisionModel` is naturally skipped.
- **structured output**: when `is_hailo_server()`, set
  `supports_schema = False` on the chat model and drop `format`/`json_object`
  from the request in `_prepare_chat_kwargs()`, since the server ignores them
  silently. Keeping them would make `llm --schema` appear to work while
  returning unstructured text.

### Phase 4 — Chat request/response hardening

In `llm_ollama/models/chat.py`:

1. **Usage**: `finalize()` currently does
   `set_usage(input=self._final_chunk.prompt_eval_count, ...)`. On Hailo
   `prompt_eval_count` is absent (`None`), which `llm.Response.set_usage`
   accepts (`input: int | None`), so input tokens are simply not reported.
   Verify this end-to-end and leave as-is; optionally omit the `input` key
   when `None` for cleaner `response_json`.
2. **Options filtering**: when on Hailo, either (a) restrict `_ChatOptions` to
   the six supported parameters so `llm models --options` tells the truth, or
   (b) keep the full set and document that unsupported ones are ignored.
   Option (a) is better UX; implement by selecting an options class in
   `ChatMixin.__init__` based on the flavor, mirroring how the thinking
   variant is already swapped per instance.
3. **Thinking tags**: `deepseek-r1` on Hailo inlines `<think>...</think>` in
   `content`. Since `thinking` is never in the capability set, the plugin
   should (behind a small helper) split leading `<think>` content out of the
   first streamed chunks into a reasoning part — or, as a smaller first step,
   strip the tags from content. Decide based on how much fidelity we want;
   the minimal correct behavior is to not crash and not leak the tags into the
   answer.
4. **Tool errors**: the server's 400 `"does not support tools"` surfaces as
   `ollama.ResponseError`. Catch it in `execute()` and re-raise as a
   `ValueError` with a Hailo-aware message ("model X on this Hailo server does
   not support tools"), matching how the plugin already reports the
   think-capability error.
5. **`keep_alive`**: nothing to change today; noted so nobody passes Ollama's
   duration strings in future work.

### Phase 5 — Docs, config, and tests

1. **README**: add a "HailoAI Ollama (Hailo-10H)" section under "Connecting to
   Ollama server":
   - `export OLLAMA_HOST=http://localhost:8000` (the Hailo server's default
     port; plain `hailo-ollama` has no 11434 listener).
   - Feature matrix: chat + streaming + tools (model-dependent) work;
     embeddings, vision/audio attachments, structured output, thinking
     control, and decision models do not.
   - Pull models first via `curl http://localhost:8000/api/pull` (or Open
     WebUI); `llm ollama models` only lists already-installed models, matching
     `/api/tags`.
2. **Unit tests** (`tests/unit/`): fixtures that mimic Hailo responses —
   `/api/tags` entries without `digest`, `show()` with `capabilities=None` —
   asserting that:
   - `llm ollama models` prints the table with `-` digests and does not raise;
   - `_distinct_models()` yields one entry per model name;
   - the capability probe registers chat models (and tools models only after
     a successful probe);
   - `finalize()` tolerates a missing `prompt_eval_count`.
3. **Integration tests**: reuse the existing kind machinery
   (`tests/conftest.py`); a Hailo server will simply skip the kinds its models
   cannot support, which is exactly the designed behavior.

## 4. Suggested implementation order

1. Phase 1 (digest-optional discovery) — unblocks `llm ollama models` today.
2. Phase 2 (flavor detection) + Phase 3 (capability defaults + tools probe) —
   makes models actually register and usable via `llm -m ollama/qwen2:1.5b`.
3. Phase 4 hardening — usage, options truthfulness, tool error messages.
4. Phase 5 — README + tests alongside each phase rather than at the end.

## 5. Open questions

- Do we want the tools probe on by default (one extra tiny request per model),
  or opt-in via environment variable? (Plan assumes default-on, cached.)
- Should `<think>` inline content be surfaced as a reasoning part (like real
  Ollama thinking models) or stripped? Surfacing is nicer but adds parsing
  logic to streaming; stripping is the minimal fix.
- If Hailo later adds `capabilities` to `/api/show`, Phase 3 should prefer
  server-reported values; the probe design should check for their presence
  first and only fall back to inference.
