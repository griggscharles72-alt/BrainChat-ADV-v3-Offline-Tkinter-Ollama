````md
# BrainChat ADV v3 (Offline) — Tkinter + Ollama

A fast, local-only chat UI for **Ollama** models using **Tkinter**. Built to be *snappy*, *stable*, and *hard to break* when streaming.

It keeps chat sessions on disk, supports model discovery, has clean stop/cancel, and avoids the classic “Tkinter thread explosion” problem by using **one UI queue + one pump**.

---

## What this is

- **Offline GUI** (Tkinter) for chatting with a local Ollama model
- Uses Ollama’s **`/api/chat`** streaming endpoint
- Stores sessions as **JSONL** on disk
- Designed for **low error rate** under heavy streaming

---

## Requirements

### System
- Linux (Ubuntu works great)
- Ollama running locally (default): `http://localhost:11434`

### Python
- Python 3.10+ recommended (3.12 is fine)

### Python dependency
- `requests`

### Tkinter
Tkinter is usually included, but on Ubuntu you may need:

```bash
sudo apt-get update
sudo apt-get install -y python3-tk
````

---

## Install

### 1) Create a virtualenv (recommended)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install requests
```

### 2) Run Ollama

Verify Ollama is up:

```bash
curl http://127.0.0.1:11434/api/tags
```

If that returns JSON, you're good.

---

## Run

```bash
python3 brainchat_adv_v3.py
```

(Whatever you named the file.)

A Tkinter window should appear.

---

## UI Overview

### Top Bar

* **Ollama URL**: Defaults to `http://localhost:11434`
* **Ping/Models**: Fetches available models via `/api/tags`
* **Model**: Dropdown populated from `/api/tags`
* **Stable Mode** (default ON):

  * Blocks non-local URLs
  * Prevents accidental remote connections
* **Fast Stream UI** (default ON):

  * Batches streaming updates to reduce Tkinter redraw overhead
* **New Session / Load / Export / Copy Last Answer**
* **Session label** shows title + backing JSONL filename

### Right Panel

* **System Prompt**: Editable “personality / instructions”
* Generation controls:

  * Temperature
  * Context size (num_ctx)
  * Timeout
* Tools:

  * Search (find highlights matches)
  * Context Guard (rough token estimate)

### Bottom Entry

* Type your message and send
* **Ctrl+Enter**: Send (streaming)
* (Ctrl+Shift+Enter is wired to a non-stream call in the code pattern, but here it currently calls `send(False)` only if you implement it—your code uses the same send handler; keep as-is if you want.)

---

## Where files are saved

Sessions and index live here:

* `~/.brainchat/`

  * `sessions.json` (catalog)
  * `session_YYYYMMDD_HHMMSS.jsonl` (session log files)

### JSONL format (session files)

Events are line-delimited JSON objects:

* `session_start`
* `title`
* `msg`
* `clear`

This format is easy to grep, parse, or replay.

---

## How stopping works (important)

The Stop button (and **Esc**) does **real cancel**:

* Sets `stop_event`
* Closes the active streaming response (`resp.close()`)

That prevents “stream keeps running in background” and avoids hanging threads.

---

## “Fast Stream UI” (why it exists)

Tkinter gets slow if you update the Text widget for every tiny token.
With Fast Stream UI enabled, the app:

* buffers chunks
* flushes every ~60ms

Result: smoother UI, higher throughput, fewer random Tk render hiccups.

---

## Stable Mode (why it exists)

When Stable Mode is ON:

* Ollama URL must be localhost/127.0.0.1/[::1]
* prevents “oops I pointed at some weird remote host” situations

Disable it only if you know what you’re doing.

---

## Troubleshooting

### 1) Window doesn’t open at all

Most common: Tkinter missing.

```bash
python3 -c "import tkinter; print('tk ok')"
```

If that fails:

```bash
sudo apt-get install -y python3-tk
```

---

### 2) “Could not fetch /api/tags”

* Ollama isn’t running or URL is wrong.

Check:

```bash
curl http://127.0.0.1:11434/api/tags
```

If you’re using Stable Mode, make sure the URL is localhost/127.0.0.1.

---

### 3) Streaming stalls or feels slow

Try these in order:

1. Turn **Fast Stream UI ON**
2. Increase Timeout (right panel)
3. Lower `num_ctx` (context) if your machine is struggling
4. Use a smaller model

---

### 4) “Stop” doesn’t stop instantly

It should stop quickly, but not always instant because:

* a chunk may be mid-flight
* network stack can delay the close slightly

If it gets stuck repeatedly, it’s usually an Ollama-side stall.
Check Ollama logs or restart Ollama.

---

### 5) Session not saving / missing messages

This app buffers writes to reduce disk I/O.
It flushes:

* periodically (~1.5s)
* force flush on “not busy” / end of generation

If you hard-kill the app mid-stream, the last couple messages might not flush.
Normal close should be fine.

---

## Performance Notes (what makes v3 faster)

* Persistent `requests.Session()` (connection pooling)
* One UI queue + one UI pump (thread-safe Tk updates)
* Stop closes the response (no zombie streams)
* Chunk batching (“Fast Stream UI”)
* Buffered session writes (less disk churn)

---

## Safety / Scope Lock

* Localhost-first workflow
* Stable Mode blocks remote URLs by default
* No WebUI, no Docker required

---

## Quick sanity test

1. Verify Ollama responds:

```bash
curl http://127.0.0.1:11434/api/tags | head
```

2. Run app:

```bash
python3 brainchat_adv_v3.py
```

3. Click **Ping/Models**

* Should populate Model dropdown

4. Send a prompt

* Streaming response appears
* Status bar shows time-to-first-token and approx tok/s

If you want, I can also generate a tiny `requirements.txt` + a `scripts/run.sh` + a `.desktop` launcher entry that *actually* opens reliably on Ubuntu (with `xdg-user-dir` + “Allow Launching” handling).
```
