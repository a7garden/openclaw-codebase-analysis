# Vendor Code, Python Skills & Special Systems

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Vendor: Google A2UI (`vendor/a2ui/`)

### 1.1 What is A2UI?

**A2UI** = Agent-to-User Interface — Google's **open-source** (Apache 2.0) protocol for agents to generate **rich declarative UIs** via a JSON specification.

- Spec versions: **v0.8** and **v0.9**
- OpenClaw vendors the **Lit web components renderer** and **Angular renderer**
- Flutter renderer is in a separate repo (`github.com/flutter/genui`)

### 1.2 Why Vendored?

OpenClaw uses A2UI for the **canvas/control UI rendering system** in `src/canvas-host/a2ui/`. Vendoring avoids npm dependency issues and allows modifications.

### 1.3 Structure

```
vendor/a2ui/
├── renderers/
│   ├── lit/               # Lit web components renderer (v0.8)
│   └── angular/           # Angular renderer
├── specification/
│   ├── 0.8/json/         # JSON Schema for v0.8
│   └── 0.9/json/         # JSON Schema for v0.9
├── .gemini/              # Gemini agent guide for the A2UI repo
└── .github/workflows/    # CI from upstream
```

### 1.4 No Modifications

The vendored A2UI code is **unchanged from upstream**. OpenClaw references it directly.

---

## 2. Python Skills (`skills/`)

### 2.1 Overview

OpenClaw includes **Python-based skill packages** for various agent capabilities. These are separate from the TypeScript core and are linted/tested via ruff + pytest.

### 2.2 Python Tooling

```toml
# pyproject.toml
[tool.ruff]
target-version = "py310"
line-length = 100
[tool.ruff.lint]
select = ["E9", "F63", "F7", "F82", "I"]
[tool.pytest.ini_options]
testpaths = ["skills"]
python_files = ["test_*.py"]
```

### 2.3 Skill Categories

| Category | Skills |
|----------|--------|
| Productivity | 1password, apple-notes, bear-notes, himalaya (email), notion, obsidian, things-mac |
| Communication | discord, slack, imessage, bluebubbles |
| Media | camsnap, gifgrep, video-frames, songsee, spotify-player |
| Dev | coding-agent, github, gh-issues, node-connect, tmux |
| AI | gemini, model-usage |
| IoT | openhue, sonoscli, gog |
| Utilities | blogwatcher, healthcheck, ordercli, sag, sherpa-onnx-tts, summarize, weather, xurl, canvas |

### 2.4 Each Skill Contains

- `SKILL.md` — Skill definition and usage
- Supporting Python scripts
- Tests: `test_*.py` via pytest

### 2.5 Skill Execution

Skills are executed via the Python interpreter as tools by the agent. The ruff linter catches issues; pytest runs the tests.

---

## 3. Patches (`patches/`)

### 3.1 Single Patch File

`patches/@whiskeysockets__baileys@7.0.0-rc.9.patch`

This patches the WhatsApp library (used by the `whatsapp` extension).

### 3.2 Changes

**1. File Write Race Condition Fix:**
```javascript
// Before: finish promises created AFTER write streams ended
// After: finish promises created BEFORE calling end()
```

Prevents `ENOENT` when readers open a file before the OS finishes writing.

**2. Dispatcher Type Guard for fetch:**
```javascript
// Only passes dispatcher option when fetchAgent implements dispatch
if (fetchAgent.dispatch) {
  fetch({ ..., dispatcher: fetchAgent })
}
```

Fixes undici compatibility when Baileys passes a generic agent.

---

## 4. WhatsApp Extension (Baileys)

The `extensions/whatsapp/` uses `@whiskeysockets/baileys` library with this patch applied via `pnpm.patchedDependencies`.

---

## 5. Vendor Summary

| Vendor | Version | Purpose | Modification |
|--------|---------|---------|-------------|
| `a2ui/renderers/lit/` | v0.8 | Canvas UI rendering | None |
| `a2ui/renderers/angular/` | v0.8 | Canvas UI rendering | None |
| `a2ui/specification/0.8/json/` | v0.8 | JSON Schema | None |
| `a2ui/specification/0.9/json/` | v0.9 | JSON Schema | None |
| `baileys` (via patch) | 7.0.0-rc.9 | WhatsApp protocol | Race condition + undici fix |