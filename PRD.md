# VoxKetik (Linux) — Product Requirements

## 1. Overview

A personal, always-running dictation tool for Zorin OS. The user presses a
global hotkey, speaks, presses it again, and the transcribed text is delivered
to wherever their cursor is.

Speech is **Bahasa Indonesia mixed with English technical vocabulary** —
"code-switching". This is the single most important characteristic of the input,
and the reason off-the-shelf dictation fails.

Transcription is cloud-only, via **Groq's Whisper** (`whisper-large-v3-turbo`).

## 2. Problem

Native Linux dictation options handle Indonesian poorly and handle
Indonesian-with-English even worse. They mis-hear technical terms, punctuate
badly, and often require speaking formatting commands aloud.

The user types prompts into terminals and AI coding tools all day. Speaking is
roughly 3× faster than typing for this, but only if the transcript is accurate
enough that correcting it doesn't erase the speed advantage.

## 3. Target user

Exactly one person: the owner. Technical, comfortable editing a JSON config and
running shell commands. This is a **personal utility, not a product** — there
are no other users, no onboarding funnel, no analytics, no growth goals.

Design implication: when forced to choose between "nice for strangers" and
"reliable for the owner", choose reliable.

## 4. Goals

- Accurate transcription of **Indonesian + Indonesian/English code-switching**
- **Insert anywhere**: terminals, editors, browsers, chat boxes
- **Toggle hotkey** UX with an unmistakable "I am recording" signal
- **Never lose a transcript**, even when insertion fails
- **No GPU**, **near-zero cost** (Groq free tier)
- Starts with the session and stays out of the way

## 5. Non-goals (deliberate)

Do not build these. Each was considered and rejected:

| Not building | Why |
|---|---|
| Local/offline model (faster-whisper) | Was built into the Windows version and used ~0.1% of the time. Cost ~335 MB RAM resident and ~460 MB disk for nothing. |
| Realtime streaming / partial captions | Large complexity increase; the toggle flow is already fast enough. |
| Audio-reactive waveform animation | Looks impressive, adds no information. A static "busy" indicator conveys the same thing. |
| Spoken formatting commands ("koma", "titik") | Breaks natural speech flow. |
| Multi-user, accounts, telemetry | Single-user tool. |
| Windows/macOS support | Out of scope. A separate Windows implementation already exists. |

## 6. Platform constraints

- **OS**: Zorin OS (Ubuntu base, GNOME desktop)
- **Session**: **Wayland by default** on Zorin 17 and 18. Xorg is available at
  the login screen; Zorin 19 is expected to drop the Xorg session.
  → This is the dominant technical constraint. See ARCHITECTURE.md §1.
- **Hardware**: CPU-only, no GPU assumed
- **Network**: required (cloud transcription). Must fail gracefully offline.
- **Secrets**: the Groq API key must never be committed. Environment variable
  takes precedence; otherwise a `0600` file outside the repo.

## 7. Features

### Must have

- **Toggle hotkey** — press to start, press again to stop and transcribe
- **Audio capture** at 16 kHz mono int16
- **Groq transcription** with `prompt` biasing and `language` hint
- **Clipboard delivery** — transcript is placed on the clipboard and **left
  there** (see §9, Lesson 2)
- **Recording indicator** that is visible without hunting for it
- **Config file** — hotkey, language, prompt, input device, dictionary
- **Explicitly pinned input device** (see §9, Lesson 4)
- **Single-instance lock** (see §9, Lesson 3)
- **Hallucination filter** — discard known empty-audio artifacts
- **Graceful failure** — offline, rate-limited, no key, no mic

### Should have

- First-run setup for the API key (no manual env-var editing required)
- Tray icon with state colour + quit
- Custom dictionary (find/replace for terms consistently misheard)
- Pre-roll buffer so the first word isn't clipped
- Autostart on login
- `doctor` command that checks session type, clipboard tool, audio, and key

### Could have (later)

- Transcript history
- Per-application behaviour profiles
- Local LLM cleanup pass

## 8. Core flow

```
IDLE ──(hotkey)──▶ RECORDING ──(hotkey)──▶ TRANSCRIBING ──▶ DELIVERING ──▶ IDLE
 │                      │                        │                │
 │                 indicator ON             indicator ON      clipboard +
 │                   (red)                    (amber)         paste attempt
 │                                                                 │
 └──────────────────── on error: notify, return to IDLE ◀──────────┘
```

**Rules:**
- A hotkey press during `TRANSCRIBING` or `DELIVERING` is ignored, not queued.
- The indicator must remain visible for the **entire** `TRANSCRIBING` phase.
  Long recordings can take tens of seconds; with no indicator the user assumes
  the app is dead and re-records.
- Transcription runs off the main thread. The hotkey path must never block.

### User flow A — normal dictation
1. Focus a text field.
2. Press hotkey → indicator appears → speak.
3. Press hotkey → indicator turns "busy".
4. Text is on the clipboard and (if auto-paste is available) at the cursor.

### User flow B — insertion failed
Same as A, but the user had focused something with no text field. **The
transcript is still on the clipboard.** They press Ctrl+V where they actually
meant to. Nothing is lost. This path must work by design, not by luck.

## 9. Lessons that must shape the build

These come from weeks of real daily use of the Windows implementation. Each one
was a real bug, not a hypothetical.

1. **Clipboard delivery beats synthetic typing.** Per-character typing is slow,
   drops characters under load, and mangles non-ASCII.
2. **Never restore the previous clipboard.** The Windows version politely
   restored the old clipboard after pasting. When a paste didn't land, the
   transcript was destroyed and a 2-minute dictation was gone. Leaving the
   transcript on the clipboard is strictly better than preserving whatever was
   there before.
3. **A single-instance lock is mandatory.** Launching twice (easy to do from a
   search menu) produces two hotkey listeners, and every dictation gets pasted
   twice. This is invisible until it happens.
4. **Pin the microphone explicitly.** The OS default input device can change
   silently when a device is plugged or unplugged. The symptom is garbage
   transcripts with **no error at all** — it looks like a transcription bug but
   is an OS state bug. Store a device name (not an index — indices shift).
5. **Pre-roll buffer.** There is a delay between the keypress and the audio
   stream delivering samples, which clips the first word. Keep a short rolling
   buffer and prepend it.
6. **Whisper hallucinates on silence.** Near-silent audio reliably produces
   `"Terima kasih."` or `"Thank you."` — verified directly. Discard a transcript
   that consists *entirely* of a known artifact, but never strip the phrase from
   inside a longer real sentence.
7. **Prompt biasing is a loaded gun.** A long natural-sentence `prompt` caused
   the beginning of transcripts to be dropped. Keep it a short vocabulary hint.
8. **Lazy-import anything heavy** so startup stays well under a second.

## 10. Success metrics

Weighted for a personal tool — adoption and cost are what matter:

- **Adoption**: the owner reaches for the hotkey instead of the keyboard
- **Accuracy**: Indonesian/mixed output needs little correction
- **Latency**: short clips feel near-instant
- **Reliability**: a transcript is **never** lost, even on failed insertion
- **Cost**: stays inside the Groq free tier (effectively $0)

## 11. Cost

**Groq `whisper-large-v3-turbo`** — recommended.

| | Value |
|---|---|
| Free tier | ~20 req/min · ~2,000 req/day |
| Free audio allowance | ~7,200 audio-sec/hour · ~28,800 audio-sec/day (≈8 h audio/day) |
| Paid rate | ~$0.04 per hour of audio |
| Billing quirk | **10-second minimum per request** — a 2-second clip bills as 10 |

Personal dictation is nowhere near these limits. Realistic heavy use is minutes
of audio per day against an ~8-hour daily allowance.

> These figures come from third-party trackers and vary slightly between
> sources. **Verify before relying on them:**
> <https://console.groq.com/docs/rate-limits> and <https://groq.com/pricing>

**Gemini** is a possible alternative but is *not* an equivalent swap:
- It is a multimodal LLM, not a transcription model. It sometimes paraphrases,
  summarises, or adds commentary instead of transcribing verbatim.
- Its free-tier limits and audio rates have been unstable, and Google stopped
  publishing per-model RPM/RPD figures. Verify at
  <https://ai.google.dev/gemini-api/docs/pricing>.

Recommendation: **ship Groq only.** Keep the transcriber behind an interface so
another provider can be added later without touching the rest of the app.

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **Wayland blocks global hotkeys and key injection** | **Critical** — app appears to work but never responds | Follow the decision tree in ARCHITECTURE.md §1. Detect the session type *first*. |
| Auto-paste unavailable on Wayland without `ydotool` | Medium | Clipboard-only mode is a fully supported first-class path, not a degraded fallback |
| Groq outage / rate limit | Medium | Clear notification; transcript is never lost; retry is cheap |
| Audio sent to a third party | Medium | Accepted trade-off, documented. No local fallback by choice. |
| Whisper hallucination on silence | Low | Artifact filter (§9.6) |
| Clipboard tool missing | Low | `doctor` command checks for it and prints the exact install command |
