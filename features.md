## AI Companion – Complete Feature List & Usage Guide

All features are available in the **AI Companion** desktop app (Tauri + React). The app is fully local, privacy‑friendly, and optimised for CPU/GPU, RAM, and disk speed.

---

## 🧠 Core AI Features

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Streaming Chat** | Real‑time token‑by‑token responses from DeepSeek API (or local LLM). | Type in the chat input and press Enter. Responses appear as they are generated. |
| **Vector Memory** | Stores conversations as 128‑dim embeddings, compressed via Product Quantization (16 subvectors, 256 centroids). | Automatic. Every user‑AI exchange is stored. |
| **Memory Retrieval** | Uses LSH (Locality‑Sensitive Hashing) + SDC (Symmetric Distance Computation) to fetch relevant past memories. | Triggered automatically when you send a message. Top‑30 most similar memories are injected into the prompt. |
| **Importance Decay** | Memories lose importance exponentially (half‑life = 1 day). Low‑importance memories are pruned. | Automatic. No user action needed. |
| **Incremental Pruning** | Deletes 5% of low‑importance memories every 10 minutes to keep total memory ≤50,000 items. | Automatic. Prevents storage bloat and performance degradation. |

---

## 🧬 Personality System

| Feature | Description | How to Use |
|---------|-------------|------------|
| **4‑Dimensional Personality** | Four independent traits: **Formality**, **Creativity**, **Empathy**, **Verbosity**, each ranging 0–15. | The AI’s responses adapt automatically. |
| **Personality Hex Code** | Personality represented as a 4‑character hex string (e.g., `3A7F`). | View it in the **Settings** panel or via `/personality` command. |
| **Personality Update** | Change personality using sliders or chat commands. | In **Settings**, adjust each dimension with +/- buttons. Or type `/personality +0.1` (adds 0.1 to creativity). |
| **Stochastic Weight Averaging (SWA)** | Smooths personality changes over 7 days to avoid sudden shifts. | Automatic. The hex code updates gradually. |
| **Weight Decay** | Personality slowly drifts back toward neutral (0.5) if not updated. | Automatic. Prevents permanent extreme behaviour. |

---

## 🚫 Anti‑Memory (Unlearning)

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Anti‑Memory Creation** | Stores a correction: “Do NOT say X, instead say Y”. | After a mistake, type `/anti "wrong phrase" -> "correct phrase"` in chat. |
| **Stretched Exponential Decay** | Anti‑memories lose strength over time (half‑life 30 days). | Automatic. Old, unused corrections fade away. |
| **Reinforcement** | If the same mistake occurs again, reinforce the anti‑memory (strength +0.5, up to 1.6). | Automatic when you add another correction for the same mistake. |
| **Thompson Sampling** | Only anti‑memories with high probability of effectiveness are included in prompts. | Automatic. Warnings appear in the chat as “Do NOT say …”. |

---

## 🛠️ Performance & Hardware Tuning

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Auto‑Detection** | Detects CPU cores, RAM, VRAM, disk speed, memory bandwidth at startup. | Automatic. No setup. |
| **Performance Modes** | Choose between **Speed**, **Balanced**, or **Memory** (low RAM). | Go to **Performance Tuner** panel and select a mode. |
| **Manual Overrides** | Adjust LSH candidates, PQ subvectors, prune batch size, cache size, GPU usage. | In **Performance Tuner**, use sliders and checkboxes. |
| **Platform Defaults** | Optimised defaults for Windows (GPU, 300 candidates), macOS (250), Linux (150, CPU). | Automatic. Can be reset via “Reset to Platform Defaults” button. |

---

## 🧩 Component System & UI Builder

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Component Library** | Store and enable/disable sandboxed HTML/CSS/JS components. | In **Components** panel, toggle components on/off. |
| **Add from URL** | Paste a raw GitHub URL (or any HTML code) to add a new component. | In **Components**, enter URL, click Fetch, then Add. |
| **UI Builder (Natural Language)** | Describe a UI widget in plain English – the AI generates a complete component. | In **UI Builder** tab, type a description (e.g., “A button that shows current weather”), click Generate, preview, then Add. |
| **Sandbox** | Components run in an iframe with `sandbox="allow-scripts"` – no access to main app or network. | Automatic. You can test any component safely. |
| **Component Preloading** | After adding, the component is preloaded in a hidden iframe for instant activation. | Automatic. |

---

## 🛡️ Self‑Recovery & Health

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Automatic Snapshots** | Takes a snapshot of memory, anti‑memories, personality, settings every hour. | Automatic. Snapshots are stored in `~/.ai-companion/snapshots/`. |
| **Health Monitor** | Checks LSH consistency, personality bounds, component safety every 30 seconds. | Automatic. Errors are logged to console. |
| **Auto‑Heal** | After 3 consecutive errors, restores the last good snapshot. | Automatic. No user intervention needed. |
| **Manual Recovery Commands** | List snapshots, restore a specific snapshot, reset a component. | In chat, type: `/recover list`, `/recover restore <snapshot-id>`, `/recover reset memory` (or `anti` / `personality` / `settings`). |

---

## 🎨 Adaptive UI & “No Gimmicks” Mode

| Feature | Description | How to Use |
|---------|-------------|------------|
| **Dark Theme** | Clean dark theme by default. | Always on. No bright mode. |
| **Adaptive UI (optional)** | Border radius, spacing, font size, colour saturation adapt to personality. | Enable **Gimmicks** in Settings. Then the UI changes as you adjust personality. |
| **Mood Detection** | Detects sentiment of your messages (positive/negative/neutral) and adjusts brightness and animation. | Automatic when Gimmicks are enabled. Negative mood = muted, positive = subtle bounce. |
| **No Gimmicks Mode** | Disables all sounds, vibrations, animations, and adaptive styling – pure static dark theme. | In Settings, uncheck “Enable sounds, vibrations, and adaptive animations”. |

---

## ⌨️ Chat Commands

Type these directly in the chat input (they are not sent to the AI):

| Command | Action |
|---------|--------|
| `/help` | Show all commands. |
| `/personality` | Show current hex code. |
| `/personality +0.1` | Increase creativity by 0.1 (clamped to 0–15). |
| `/personality -0.1` | Decrease creativity by 0.1. |
| `/anti wrong -> correct` | Add an anti‑memory. |
| `/memory` | Show total memory count. |
| `/recover list` | List available snapshots. |
| `/recover restore <id>` | Restore a snapshot. |
| `/recover reset memory` | Clear all memories. |
| `/recover reset anti` | Clear all anti‑memories. |
| `/recover reset personality` | Reset personality to neutral (0000). |
| `/recover reset settings` | Reset all settings to defaults. |
| `/quit` | Exit the app. |

---

## 🔧 Settings Panel Options

| Section | Settings |
|---------|----------|
| **General** | Enable/disable Gimmicks (sounds, animations, adaptive UI). |
| **Performance** | Mode (Speed / Balanced / Memory), LSH candidates, prune batch size, cache size, GPU acceleration. |
| **API** | DeepSeek API key (stored locally). |
| **Storage** | Data directory, max memories (default 50,000). |

---

## 🧪 First‑Time Setup

1. Download the app from the GitHub releases or build from source.
2. Run the executable.
3. On first launch, the app will auto‑detect your hardware and set optimal defaults.
4. Enter your DeepSeek API key in **Settings → API** (or set `DEEPSEEK_API_KEY` environment variable).
5. Start chatting! The AI will remember everything and adapt to you.

---

## 📂 Data Location

- **Windows**: `%APPDATA%\ai-companion\data`
- **macOS**: `~/Library/Application Support/ai-companion`
- **Linux**: `~/.local/share/ai-companion`

Inside, you’ll find:
- `memories.bin` – compressed memory codes (bincode)
- `anti.json` – anti‑memories
- `personality.json` – personality state
- `settings.json` – user settings
- `snapshots/` – recovery snapshots

---

## ❓ Troubleshooting

- **No responses?** Check your DeepSeek API key in Settings.
- **High RAM usage?** Switch to **Memory** performance mode or reduce `max_mem` in Settings.
- **Slow retrieval?** Increase LSH candidates (Speed mode) or enable GPU acceleration if available.
- **UI glitches?** Disable Gimmicks (pure dark mode).
- **Corrupted data?** Use `/recover list` and `/recover restore <latest-id>` to roll back.

---

The AI Companion is now ready to use. Enjoy a private, adaptive, and powerful AI that learns your personality and never forgets. 🧠
