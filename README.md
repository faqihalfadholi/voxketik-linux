# VoxKetik for Linux — Build Specification

> **This repository contains no application code.** It is a complete, buildable
> specification for a personal voice-dictation tool targeting **Zorin OS**
> (Ubuntu-based, GNOME). It is written to be executed by an AI coding agent.

## What is being built

Press a global hotkey, speak (Bahasa Indonesia, often mixed with English), press
again, and the transcribed text appears at your cursor — in a terminal, an
editor, a browser, anywhere a cursor blinks.

Transcription runs on **Groq's hosted Whisper** (free tier, no GPU needed).
There is no local model, no realtime waveform, no account system. It is
deliberately small.

## Why this spec exists

A Windows version of this tool was built first and used daily for weeks. That
version works, but its core input mechanism (`pynput` for global hotkeys and
synthetic `Ctrl+V`) **does not work on Wayland**, which is Zorin OS's default
session. A straight port would run without errors and silently never respond to
the hotkey.

So rather than port the code, this repository captures what actually matters:

- the **architecture that worked**,
- the **exact API calls**, verified against a running implementation,
- the **bugs that only appeared after weeks of real use**, and
- a **Wayland decision tree** that a fresh implementation would otherwise get
  wrong.

The hard part of this project is not the transcription. It is the platform
integration and a handful of non-obvious failure modes.

## How to use this repository

Read in this order:

| # | Document | What it gives you |
|---|---|---|
| 1 | **[PRD.md](PRD.md)** | What to build and what to deliberately *not* build. Scope, user flow, state machine, cost, risks. |
| 2 | **[ARCHITECTURE.md](ARCHITECTURE.md)** | How to build it. Component design, the Wayland decision tree, exact Groq API spec, Linux package requirements, and the hard-won lessons. |

> **Agent: start with the Wayland decision tree in ARCHITECTURE.md §1 and run
> the detection command before choosing any library.** Every later decision
> depends on its result.

## Status

Specification only — not yet implemented.

## Prerequisites for whoever builds this

- Zorin OS (or any Ubuntu/GNOME-based distribution)
- Python 3.11+
- A free Groq API key from [console.groq.com](https://console.groq.com) (no credit card)
- A working microphone

## License

[MIT](LICENSE) — do whatever you want with it.
