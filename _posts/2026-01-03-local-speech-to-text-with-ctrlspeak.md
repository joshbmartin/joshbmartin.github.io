---
title: Local Speech-to-Text with ctrlspeak
date: 2026-03-01 09:00:00 -0500
categories: [utilities]
tags: [python, macos, speech-to-text]
img_path: /assets/img/
published: false
---

I type a lot. Whether it's writing code, responding to messages, or drafting docs, my hands are on the keyboard for most of the day. I've always been curious about voice dictation as a way to get thoughts out faster, but every solution I looked at came with tradeoffs.

### The problem with cloud-based dictation

An option like Wispr Flow has a nice UI but, ultimately only offers so much for free. And you sacrafice everything you say going back through someone elses infra.

1. **Privacy** — Your raw audio is being sent to someone else's servers. Wispr Flow ($144/year) sends recordings to OpenAI and Meta for processing. Some tools have even been caught capturing screenshots of your active window alongside the audio.
2. **Cost** — These are subscription services wrapping freely available models. You're paying $10-15/month for something your laptop can do locally.
3. **Connectivity** — No internet, no dictation. That rules out airplanes, spotty wifi, or just not wanting to depend on a connection.
4. **Integration** — Most dictation apps are their own little world. What I actually wanted was something that stays invisible until I need it, activates with a hotkey, and pastes text wherever my cursor happens to be.

### Why not just use Apple Dictation?

Apple's built-in dictation on Apple Silicon actually runs on-device, which is great for privacy. But it lacks any kind of customization — you can't swap models, there's no API, and the accuracy doesn't hold up to dedicated models like Parakeet. It's fine for a quick text message, but not for serious daily use.

### Enter ctrlspeak

[ctrlspeak](https://github.com/patelnav/ctrlspeak) by Nav Patel takes a fundamentally different approach. As he put it on Hacker News: *"I wanted something that ran locally, was open-source, and wasn't trying to squeeze me for a subscription."*

Here's how it works:

1. Launch `ctrlspeak` in a terminal
2. Double-tap the Ctrl key to start recording
3. Speak naturally — it uses Silero VAD to detect when you're actually talking
4. Double-tap Ctrl again to stop
5. Transcribed text is automatically pasted at your cursor position

That's it. No app switching, no UI to click through. It works system-wide — in VS Code, the terminal, Slack, a browser, wherever your cursor is.

### What makes it fast: Parakeet + MLX + Apple Silicon

The default model is NVIDIA's Parakeet TDT 0.6B, which holds first place on Hugging Face's Open ASR Leaderboard with a 6.05% word error rate. That's competitive with or better than Whisper Large v3, but in a much smaller package — only 0.6B parameters requiring about 2GB of memory.

The key ingredient is Apple's [MLX framework](https://github.com/ml-explore/mlx). The [parakeet-mlx](https://github.com/senstella/parakeet-mlx) port runs natively on Apple Silicon's GPU through Metal Performance Shaders. This isn't CPU fallback — it's actual GPU acceleration using the unified memory architecture on M1/M2/M3/M4 chips. On an M3 MacBook Pro, it can transcribe a 1-hour audio file in roughly 1 minute. For short dictation clips, you're looking at sub-second transcription.

ctrlspeak also supports multiple models you can swap at runtime (press `m`):

- **Parakeet 0.6B (MLX)** — Fastest, Apple Silicon optimized, the default
- **Canary 1B Flash** — Multilingual (English, German, French, Spanish)
- **Whisper Large v3** — Best punctuation/capitalization, but slower
- **Nemotron Streaming** — Experimental real-time transcription

### Making it repeatable: dotfiles + install scripts

ctrlspeak is available via Homebrew (`brew tap patelnav/ctrlspeak && brew install ctrlspeak`), but since I'm running a customized fork, I wanted a setup that runs directly from source and is fully automated through my dotfiles.

I maintain a [dotfiles repo](https://github.com/jbmartino/dotfiles) with an `install.sh` that handles everything from Homebrew packages to editor configs. The ctrlspeak setup has three parts: a launcher script, a Python venv, and the install script to tie them together.

**bin/ctrlspeak** — A launcher script that lives in my dotfiles and gets symlinked onto `$PATH`:

```bash
#!/bin/bash
# ctrlspeak - local speech-to-text using Parakeet AI (MLX)
# Double-tap Ctrl to start/stop recording, auto-pastes at cursor

CTRLSPEAK_SRC="$HOME/repos/ctrlspeak"
VENV_PYTHON="$HOME/.local/ctrlspeak/venv/bin/python"

export PYTHONPATH="$CTRLSPEAK_SRC:$PYTHONPATH"

# Set dynamic library paths for torch (detect Python version automatically)
SITE_PACKAGES="$("$VENV_PYTHON" -c 'import site; print(site.getsitepackages()[0])')"
TORCH_LIB_PATH="$SITE_PACKAGES/torch/lib"
TORCHAUDIO_LIB_PATH="$SITE_PACKAGES/torchaudio/lib"
export DYLD_LIBRARY_PATH="$TORCH_LIB_PATH:$TORCHAUDIO_LIB_PATH:$DYLD_LIBRARY_PATH"

exec "$VENV_PYTHON" "$CTRLSPEAK_SRC/ctrlspeak.py" "$@"
```

The launcher points at my fork cloned to `~/repos/ctrlspeak` and uses the venv's Python to run it. The torch library paths are detected dynamically so it doesn't break across Python version upgrades.

**install.sh** — Creating the venv and wiring up the launcher:

```bash
# Setup ctrlspeak venv
CTRLSPEAK_VENV="$HOME/.local/ctrlspeak/venv"
CTRLSPEAK_SRC="$HOME/repos/ctrlspeak"
if [ ! -d "$CTRLSPEAK_VENV" ]; then
  echo "Setting up ctrlspeak venv..."
  mkdir -p "$HOME/.local/ctrlspeak"
  python3 -m venv "$CTRLSPEAK_VENV"
  "$CTRLSPEAK_VENV/bin/pip" install --upgrade pip
  "$CTRLSPEAK_VENV/bin/pip" install -r "$CTRLSPEAK_SRC/requirements.txt"
  "$CTRLSPEAK_VENV/bin/pip" install -r "$CTRLSPEAK_SRC/requirements-mlx.txt"
  "$CTRLSPEAK_VENV/bin/pip" install huggingface_hub packaging
  echo "ctrlspeak venv created at $CTRLSPEAK_VENV"
else
  echo "ctrlspeak venv already exists, skipping..."
fi

# Symlink ctrlspeak launcher to PATH
mkdir -p "$HOME/.local/bin"
ln -sf "$HOME/dotfiles/bin/ctrlspeak" "$HOME/.local/bin/ctrlspeak"
```

The venv lives at `~/.local/ctrlspeak/venv` and installs the ML dependencies (torch, torchaudio, mlx, parakeet-mlx, etc.) from the requirements file. This keeps the heavy Python dependencies isolated from anything else on the system. The symlink drops the launcher into `~/.local/bin`, which is already on my `$PATH`.

### Forking for customization

The upstream ctrlspeak uses a triple-tap on Ctrl to toggle recording. I found that a double-tap felt more natural and responsive, so I [forked the repo](https://github.com/jbmartino/ctrlspeak) and made the change. It was a clean 4-file diff:

- `utils/keyboard_shortcuts.py` — Changed the tap count threshold from 3 to 2
- `ctrlspeak.py` — Updated the docstring and welcome message
- `ui/screens/help.py` — Updated help text
- `ui/screens/recording.py` — Updated status label

Having the fork means I can pull upstream improvements while keeping my keybinding preference. And since my dotfiles run directly from the cloned source, new machines get my customized version automatically — just clone the fork and run `./install.sh`.

### The bottom line

ctrlspeak solves a real problem without the usual tradeoffs. No subscriptions, no cloud dependency, no privacy concerns. Just a local model running on your GPU, triggered by a hotkey, pasting text wherever you need it. Combined with a dotfiles-driven install, it's one `./install.sh` away on any new Mac.

If you're on Apple Silicon and haven't tried local speech-to-text yet, give it a shot. The upstream is available via Homebrew:

```bash
brew tap patelnav/ctrlspeak
brew install ctrlspeak
ctrlspeak
```

Or clone [my fork](https://github.com/jbmartino/ctrlspeak) if you prefer the double-tap activation. Either way — fire it up, tap Ctrl, and start talking.
