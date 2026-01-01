## Mistral Vibe CLI + Ollama (Devstral Small 2 24B) on Ubuntu 24.04 + RTX 5090

This guide assumes you’re on a **Linux box meant to run an agentic coding CLI locally**:

* **OS:** Ubuntu 24.04 LTS
* **GPU:** NVIDIA **RTX 5090** (i.e., plenty of compute; VRAM is still the real limiter for long context)
* **Ollama:** already installed as **`ollama-linux-amd64_v0.13.4`** and already running as a user service using your script (so we **do not** redo Ollama setup here).
* **Important networking assumption:** your Ollama server is reachable at the Docker bridge IP (example: `172.17.0.1:11434`), and you can do:

  ```bash
  curl -s http://172.17.0.1:11434/v1/models | head
  ```

  (That endpoint is part of Ollama’s OpenAI-compatible API.)

---

# 0) What you have now (and what you’re missing)

You already proved Ollama is working and you already have the model pulled:

* The model (exact tag you asked for):
  **`devstral-small-2:24b-instruct-2512-q4_K_M`**

We assume Ollama was set up using https://github.com/BigBIueWhale/personal_server/blob/master/install_ollama_user_service.sh

What you’re missing is: **making Vibe use the same “good” inference settings you discovered in OpenWebUI**, specifically:

```json
{
  "min_p": 0.01,
  "num_ctx": 104000,
  "max_tokens": -1
}
```

The catch: **Vibe talks “OpenAI API.”** And the OpenAI API **does not provide a way to set context size (`num_ctx`)**. Ollama explicitly calls this out: if you need a different context size, you must create a new model via a **Modelfile**.

So the correct move is:

✅ **Wrap the base Devstral model in an Ollama Modelfile** that bakes in your `num_ctx`, `min_p`, and infinite generation.

---

# 1) Point your shell at your running Ollama (one-liner)

Because your Ollama is listening on the Docker bridge IP (not `127.0.0.1`), export this before running Ollama CLI commands:

```bash
export OLLAMA_HOST=172.17.0.1:11434
```

(That matches what you already did successfully.)

---

# 2) Create a “Vibe-tuned” Devstral model in Ollama (this is the key step)

### Why this is necessary

* **`num_ctx`**: Vibe cannot set it over OpenAI-style requests, so you must bake it into the model.
* **`min_p`**: OpenAI chat-completions doesn’t standardize it; Ollama supports it as a model parameter. ([Ollama Documentation][2])
* **`max_tokens=-1`**: in Ollama terms this is **`num_predict -1`** (“infinite generation”). ([Ollama Documentation][2])

### Create a Modelfile

Make a folder anywhere (example uses `~/models/devstral-vibe`):

```bash
mkdir -p ~/models/devstral-vibe
cd ~/models/devstral-vibe
nano Modelfile
```

Put this inside (edit nothing except the model name if you want):

```text
FROM devstral-small-2:24b-instruct-2512-q4_K_M

# Agentic coding: keep tool plans stable.
# Mistral recommends temperature 0.2 for Vibe/Devstral.
PARAMETER temperature 0.2

# Your measured "max that reliably fits VRAM"
PARAMETER num_ctx 104000

# Your preferred sampling floor
PARAMETER min_p 0.01

# Infinite generation (OpenWebUI's max_tokens = -1)
PARAMETER num_predict -1
```

Now create the derived model:

```bash
ollama create devstral-vibe -f ./Modelfile
```

Verify it:

```bash
ollama show --modelfile devstral-vibe
curl -s http://172.17.0.1:11434/v1/models | grep -n devstral-vibe
```

### Why these exact values (opinionated, for “Claude Code”-like reliability)

* **temperature = 0.2**
  Mistral explicitly recommends **0.2** for optimal Vibe/Devstral agent behavior. Low temperature reduces “wandering edits” and makes multi-step tool use more consistent. ([Mistral AI][3])

* **num_ctx = 104000**
  Context length is literally “how many tokens the model can hold in memory,” and increasing it increases memory usage. ([Ollama Documentation][4])
  You tested 104k as the practical ceiling on your VRAM, so we treat that as your “safe max.” This is exactly how you get “big-repo awareness” without random CUDA OOMs.

* **min_p = 0.01**
  Ollama describes `min_p` as a probability floor relative to the most likely token; it filters ultra-low-probability junk. ([Ollama Documentation][2])
  For coding agents, a tiny `min_p` is a nice compromise: it still keeps outputs conservative, but helps avoid occasional bizarre token choices that can derail tool calls or code edits.

* **num_predict = -1** (infinite)
  Ollama’s Modelfile docs define `num_predict` and note the default `-1` means infinite generation. ([Ollama Documentation][2])
  This prevents Vibe from being silently cut off mid-refactor.

---

# 3) Install Mistral Vibe CLI on Ubuntu

Official options (pick one):

### Option A (simplest)

```bash
curl -LsSf https://mistral.ai/vibe/install.sh | bash
```

([Mistral AI Documentation][5])

### Option B (cleanest / reproducible, recommended)

If you use `uv`:

```bash
uv tool install mistral-vibe
```

([Mistral AI Documentation][5])

---

# 4) Configure Vibe to use Ollama (OpenAI-compatible)

Vibe reads config from `./.vibe/config.toml` first, then `~/.vibe/config.toml`. ([Mistral AI Documentation][6])
For a “works everywhere” setup, use the global file:

```bash
mkdir -p ~/.vibe
nano ~/.vibe/config.toml
```

Paste:

```toml
# Trigger Vibe's automatic summarization ("compaction") before we hit the real limit.
# We set it below 104k to leave headroom for:
# - Vibe's system prompt + tool traces
# - the model's in-progress output tokens
auto_compact_threshold = 95000

active_model = "devstral-local"

[[providers]]
name = "ollama-docker0"
api_base = "http://172.17.0.1:11434/v1"
api_key_env_var = "OLLAMA_API_KEY"
api_style = "openai"
backend = "generic"

[[models]]
name = "devstral-vibe"          # the Ollama model we created
provider = "ollama-docker0"
alias = "devstral-local"
temperature = 0.2
input_price = 0.0
output_price = 0.0
```

Why this works:

* Vibe supports custom providers with `api_style = "openai"` and a `generic` backend. ([Mistral AI Documentation][6])
* Ollama provides OpenAI-compatible endpoints and even notes the API key is “required but ignored.”

Create the `.env` Vibe expects:

```bash
nano ~/.vibe/.env
```

Put:

```env
OLLAMA_API_KEY=ollama
```

(Any string is fine; Ollama ignores it in OpenAI-compat mode.)

---

# 5) How Vibe “knows” your context length (and what happens when you exceed it)

### How Vibe knows (practically)

It **doesn’t truly know** your model’s real limit.

* The OpenAI-style API **does not let clients set context size**, and Ollama tells you to use a Modelfile if you need a larger context. ([Ollama Documentation][1])
* So Vibe can only use a **client-side heuristic**: `auto_compact_threshold`.

That’s why we set `auto_compact_threshold = 95000` manually: it forces Vibe to summarize earlier conversation *before* you hit the hard ceiling of `num_ctx=104000`.

### What happens if the limit is exceeded

Context length is the maximum tokens the model can hold “in memory.” ([Ollama Documentation][4])
If your conversation + tool traces + requested output push beyond the context window, one of two things typically happens in practice:

* **The server rejects the request** with a “maximum context length” style error (common in OpenAI-compatible servers), or
* **Earlier parts of the conversation stop being available** (effectively “forgotten”), which breaks agent continuity.

So: **compaction is not optional** if you want long-running “Claude Code”-style sessions on big repos.

---

# 6) Run it

Go to a repo and start:

```bash
cd /path/to/your/repo
vibe
```

Inside Vibe:

* Use `/config` to confirm the active model and provider (Vibe supports switching via `/config`). ([Mistral AI Documentation][6])
* Then give it a real agent task, e.g. “scan repo, propose plan, then implement.”

---

# 7) Two small, high-impact “Claude Code rival” tweaks (optional but worth it)

### A) Prefer “project-local” config for serious repos

Because Vibe checks `./.vibe/config.toml` first, you can put a repo-specific config there (especially useful if different repos need different compaction thresholds). ([Mistral AI Documentation][6])

### B) Keep temperature low for tool stability

If you ever feel it getting “creative” with commands or edits, don’t raise temperature first—raise **process discipline** (more planning, smaller diffs). Mistral’s own guidance is that **0.2** is the sweet spot for Vibe/Devstral. ([Mistral AI][3])

---

## Quick “sanity checklist”

* `curl http://172.17.0.1:11434/v1/models` shows **devstral-vibe**
* `ollama show --modelfile devstral-vibe` shows `num_ctx 104000`, `min_p 0.01`, `num_predict -1`
* `~/.vibe/config.toml` points `api_base` to `http://172.17.0.1:11434/v1`
* `vibe` starts without prompting endlessly for keys (because `~/.vibe/.env` exists)

If you want, paste your `~/.vibe/config.toml` (redact nothing—there are no real secrets for Ollama here), and I’ll tell you if anything will trip Vibe up (especially around model name/alias mismatches).
