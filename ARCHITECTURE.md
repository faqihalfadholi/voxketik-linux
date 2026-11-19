# VoxKetik (Linux) — Architecture

> **Agent: do not skip §1.** Every library choice below depends on which display
> session is running. Choosing wrong produces an app that starts cleanly, logs
> nothing, and never responds to the hotkey.

---

## 1. The Wayland decision tree (start here)

### 1.1 Detect the session — run this first

```bash
echo $XDG_SESSION_TYPE
```

Output is `wayland` or `x11`. Zorin OS 17/18 default to **`wayland`**.

Do not assume. Do not infer it from the distro name. Run the command, then
follow the matching branch.

### 1.2 Why this matters

Wayland deliberately forbids what the Windows implementation relied on. A
client cannot grab global keyboard input, and it cannot synthesise keystrokes
into another window. `pynput` therefore **silently does nothing** for both
hotkey capture and `Ctrl+V` injection. There is no exception and no error.

| Capability | X11 | Wayland (GNOME/Mutter) |
|---|---|---|
| Global hotkey via `pynput` | works | **blocked** |
| Synthetic keystroke via `pynput` | works | **blocked** |
| `wtype` keystroke injection | n/a | **unsupported on GNOME** — Mutter does not implement the virtual-keyboard protocol. Do not suggest it. |
| `ydotool` keystroke injection | works | works, but needs `/dev/uinput` access |
| Clipboard | `xclip` / `xsel` | `wl-clipboard` |

### 1.3 Hotkey — recommended solution for Wayland

**Do not try to grab the hotkey from inside the app.** Let the desktop do it.

GNOME can bind any key combination to a shell command. Bind it to a tiny
"toggle" command that signals the already-running daemon over a Unix socket.

This is the cleanest available answer: it is native, needs **zero special
permissions**, cannot be blocked by the compositor, and works identically on
X11 and Wayland.

```bash
KEY=/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/voxketik/
SCHEMA=org.gnome.settings-daemon.plugins.media-keys

gsettings set $SCHEMA custom-keybindings "['$KEY']"
gsettings set $SCHEMA.custom-keybinding:$KEY name 'VoxKetik Toggle'
gsettings set $SCHEMA.custom-keybinding:$KEY command '/full/path/to/.venv/bin/python -m voxketik.toggle'
gsettings set $SCHEMA.custom-keybinding:$KEY binding '<Control><Shift>space'
```

> If other custom keybindings already exist, read the current list first and
> append to it — the `set` above replaces the whole array.

**The app therefore ships two entry points:**

| Entry point | Role |
|---|---|
| `python -m voxketik` | The long-running daemon: audio, transcription, delivery, indicator |
| `python -m voxketik.toggle` | A ~20-line client: connect to the socket, send one byte, exit |

### 1.4 Socket design — solves two problems at once

Use a Unix domain socket at `$XDG_RUNTIME_DIR/voxketik.sock`.

Binding it **also serves as the single-instance lock**: if `bind()` fails
because the address is in use, another daemon is already running, so exit
quietly. One mechanism, two requirements, no separate lock file.

Clean up a stale socket file on startup (after confirming nothing is listening)
so a crash doesn't permanently block future launches.

### 1.5 Text delivery — pick one mode

**Mode A — clipboard only (default, recommended).**
Put the transcript on the clipboard and stop there. The user presses paste
themselves. This is not a degraded fallback; it is the robust path:

- works on Wayland and X11 with no permissions
- sidesteps the terminal problem entirely — terminals paste with
  `Ctrl+Shift+V`, other apps with `Ctrl+V`, and on Wayland you cannot reliably
  detect which kind of window is focused
- the transcript is always recoverable (see PRD §9, Lesson 2)

**Mode B — auto-paste via `ydotool` (opt-in).**
Requires the `ydotoold` daemon and access to `/dev/uinput`:

```bash
sudo apt install ydotool
sudo usermod -aG input "$USER"      # log out and back in afterwards
# keycodes: ctrl=29, shift=42, v=47
ydotool key 29:1 42:1 47:1 47:0 42:0 29:0   # Ctrl+Shift+V
```

Make the key combination configurable; do not hardcode `Ctrl+V`.

**Mode C — X11 session.** If `$XDG_SESSION_TYPE` is `x11`, `pynput` works for
both hotkey and paste exactly as on Windows. Simpler, but Zorin 19 is expected
to remove the Xorg session, so do not build *only* for this.

### 1.6 Wayland clipboard gotcha

On Wayland the clipboard is owned by a live client. **If the process that set
the clipboard exits, the content disappears.** `wl-copy` handles this by
forking a background process that stays alive to serve the data — so call
`wl-copy` as a detached subprocess and let it do its job. Do not use
`--foreground`, and do not kill it.

---

## 2. Components

| Component | Responsibility |
|---|---|
| `Daemon` | Owns the socket, the state machine, and the main loop |
| `ToggleClient` | Separate entry point; one socket write, then exit |
| `AudioRecorder` | Start/stop capture; 16 kHz mono int16; pre-roll ring buffer; returns WAV bytes |
| `Transcriber` | Interface: `transcribe(wav, *, prompt, language) -> str` |
| `GroqTranscriber` | The only implementation needed at launch |
| `PostProcessor` | Hallucination filter + dictionary find/replace |
| `TextDelivery` | Clipboard write (+ optional auto-paste) |
| `Indicator` | Conveys state: idle / recording / transcribing |
| `Config` | JSON at `~/.config/voxketik/config.json` |
| `Secrets` | API key: env var first, else `0600` file |
| `Doctor` | Preflight diagnostics |

### State machine

```
IDLE ──▶ RECORDING ──▶ TRANSCRIBING ──▶ DELIVERING ──▶ IDLE
```

Transcription and delivery run on a worker thread. The socket-reading loop must
never block, or the toggle will appear frozen.

Toggle presses received while not `IDLE` or `RECORDING` are **ignored, not
queued**.

---

## 3. Tech stack

### System packages

```bash
sudo apt install libportaudio2 wl-clipboard
# X11 session instead of Wayland:
sudo apt install xclip
# only if you want auto-paste (Mode B):
sudo apt install ydotool
```

### Python packages

```
groq            # transcription client
sounddevice     # audio capture (PortAudio)
numpy           # audio buffers
pillow          # indicator icon rendering
pystray         # tray icon (optional — see §3.1)
```

Note there is no `pynput` in the default (Wayland) build, and **no
`faster-whisper`** — dropping the local model removes roughly 460 MB of disk
and 335 MB of resident memory.

Use `uv` for the environment:

```bash
uv venv --python 3.12
uv pip install groq sounddevice numpy pillow pystray
```

### 3.1 Indicator — read this before choosing

Modern GNOME does not display legacy tray icons without the *AppIndicator and
KStatusNotifierItem Support* extension. Zorin OS ships its own panel and
generally supports indicators, but **verify rather than assume** — have
`doctor` check and report.

Recommended layering, most to least reliable:

1. **Desktop notification** (`notify-send` / libnotify) on completion and on
   every error — always works, no extension required
2. **Tray icon** via `pystray` + AppIndicator, if available — best for
   persistent state (idle / recording / busy)
3. **Floating overlay window** — on GNOME Wayland a client **cannot position
   its own window**, so a bottom-centre pill like the Windows version is not
   reliably reproducible. Do not spend effort here. Use the tray icon and
   notifications instead.

Whatever is chosen, the indicator **must stay visible for the whole
transcribing phase**, not just while recording (PRD §8).

---

## 4. Groq API

Verified against a working implementation, not copied from documentation.

```
POST https://api.groq.com/openai/v1/audio/transcriptions
Authorization: Bearer $GROQ_API_KEY
Content-Type: multipart/form-data
```

| Field | Value |
|---|---|
| `file` | WAV bytes, 16 kHz mono int16, filename e.g. `audio.wav` |
| `model` | `whisper-large-v3-turbo` |
| `prompt` | short vocabulary hint (see §4.2) |
| `language` | `id`, or omit for auto-detect |
| `temperature` | `0.0` — reduces hallucination |

Using the official SDK:

```python
from groq import Groq

client = Groq()   # reads GROQ_API_KEY from the environment

result = client.audio.transcriptions.create(
    file=("audio.wav", wav_bytes),
    model="whisper-large-v3-turbo",
    prompt=prompt or None,
    language=language or None,
    temperature=0.0,
)
text = (result.text or "").strip()
```

### 4.1 Errors to handle explicitly

| Exception | Meaning | Response |
|---|---|---|
| `groq.APIConnectionError` | offline / DNS failure | notify "offline"; keep audio |
| `groq.RateLimitError` | HTTP 429 | notify "rate limited, retry shortly" |
| `groq.AuthenticationError` | bad or missing key | open the key setup flow |
| `groq.APIStatusError` | other HTTP error | surface the status code |

Never let an exception reach the user as a silent failure. Every failure path
ends with a visible notification **and** a return to `IDLE`.

### 4.2 Prompt biasing — handle with care

`prompt` is a *context hint*, not an instruction. Keep it a **short vocabulary
list**, not a natural sentence.

```
Transkrip Bahasa Indonesia dengan istilah teknis Inggris.
Istilah: Claude Code, Groq, prompt, API, deploy, repo.
```

A long, natural-sounding prompt measurably caused the **beginning of
transcripts to be dropped** in the Windows build. If transcripts start losing
their first words, suspect the prompt before suspecting the audio.

### 4.3 Language

The speech is ~80% Indonesian with English technical terms. Setting
`language="id"` is more reliable than auto-detect, which can mis-detect short
clips or heavily code-switched speech as Malay or English. Keep it
configurable.

---

## 5. Files and layout

```
~/.config/voxketik/
├── config.json        # user settings
└── secrets.json       # chmod 0600, API key — NEVER in the repo

$XDG_RUNTIME_DIR/voxketik.sock    # IPC + single-instance lock
~/.config/autostart/voxketik.desktop
```

Resolve the config directory via `$XDG_CONFIG_HOME`, defaulting to
`~/.config`. Do not hardcode `~/.config`.

### config.json

```json
{
  "hotkey_hint": "<Control><Shift>space",
  "language": "id",
  "prompt": "Transkrip Bahasa Indonesia dengan istilah teknis Inggris.",
  "input_device": null,
  "pre_roll_seconds": 0.5,
  "delivery_mode": "clipboard",
  "paste_combo": "ctrl+shift+v",
  "dictionary": {}
}
```

`hotkey_hint` is documentation only — GNOME owns the real binding (§1.3).
`input_device` stores a device **name**, never an index (PRD §9, Lesson 4).

### Secrets resolution order

1. `GROQ_API_KEY` environment variable
2. `~/.config/voxketik/secrets.json`, created with mode `0600`
3. Neither → run the first-run setup flow

Keeping secrets in a separate file from `config.json` means the config can be
shared or pasted into a bug report without leaking the key.

### Autostart

```ini
[Desktop Entry]
Type=Application
Name=VoxKetik
Comment=Voice dictation daemon
Exec=/full/path/to/.venv/bin/python -m voxketik
Icon=/full/path/to/assets/voxketik.png
Terminal=false
X-GNOME-Autostart-enabled=true
```

Save as `~/.config/autostart/voxketik.desktop`. Startup must stay under one
second — lazy-import anything heavy.

---

## 6. Implementation notes for the non-obvious parts

### 6.1 Pre-roll buffer

The audio stream does not deliver samples instantly after the toggle, which
clips the first word. Keep the input stream open continuously and maintain a
rolling buffer of the last ~0.5 s; on toggle, seed the recording with it.

Trade-off: a continuously open stream means the OS shows the microphone as
in use at all times. Make the duration configurable, where `0` means "open the
microphone only while recording".

### 6.2 Hallucination filter

Normalise the transcript (lowercase, strip punctuation, collapse whitespace)
and discard it **only if the entire transcript** matches a known artifact:

```
terima kasih · terima kasih banyak · terima kasih telah menonton
sampai jumpa · thank you · thanks for watching · please subscribe
```

Never strip these phrases from inside a longer transcript — "makasih ya udah
bantu" is real speech and must survive.

### 6.3 Icon generation

The Windows build drew its icon programmatically with Pillow rather than
shipping a binary asset — a rounded tile plus a white microphone glyph,
rendered at several sizes, recoloured per state (blue idle / red recording /
amber busy). Reproduce the same approach: it keeps the repository free of
binary blobs and makes state colours trivial.

### 6.4 `doctor` command

`python -m voxketik.doctor` should check and print, with a fix for each failure:

- `$XDG_SESSION_TYPE` → wayland or x11, and which delivery mode applies
- clipboard tool present (`wl-copy` or `xclip`)
- PortAudio present; list input devices; flag the configured one as present/missing
- API key found (report **presence and length only — never print the value**)
- GNOME custom keybinding registered
- AppIndicator availability for the tray
- `ydotool` + `/dev/uinput` access, if `delivery_mode` is `auto_paste`

This is the first thing to run when something misbehaves, and the first thing
to ask for in a bug report.

---

## 7. Build order

Each step ends with something testable. Do not proceed on an unverified step.

1. **Daemon skeleton + socket** — `voxketik.toggle` flips a state and prints it.
   Verify the GNOME keybinding actually reaches the daemon. *This de-risks the
   single largest unknown first.*
2. **Audio capture** — toggle records and writes a WAV; play it back and confirm
   it contains real speech from the intended microphone.
3. **Groq transcription** — WAV in, text out, errors handled.
4. **Clipboard delivery** — transcript lands on the clipboard and survives the
   worker thread exiting.
5. **Indicator** — notification first, tray icon if available.
6. **Config, secrets, first-run setup.**
7. **Post-processing** — hallucination filter, dictionary.
8. **Pre-roll buffer.**
9. **`doctor` + autostart.**

---

## 8. Verification checklist

Prove each of these by running it, not by reading the code:

- [ ] `echo $XDG_SESSION_TYPE` recorded, and the matching delivery mode selected
- [ ] Hotkey triggers the daemon from any focused application
- [ ] Recorded WAV contains speech from the **intended** microphone (play it)
- [ ] Transcript of Indonesian-with-English speech is accurate
- [ ] **Transcript remains on the clipboard after delivery** (paste it somewhere new)
- [ ] **Focus a window with no text field, dictate — the transcript is still recoverable**
- [ ] Launching the daemon twice: the second exits quietly, and dictation is **not** pasted twice
- [ ] Unplugging/changing the audio device does not silently switch the input
- [ ] Silence produces **no** output, not `"Terima kasih."`
- [ ] Indicator stays visible for the entire transcription, including a 60-second clip
- [ ] Airplane mode produces a clear notification, not a silent failure
- [ ] `doctor` passes on a clean machine
- [ ] Autostart survives a reboot
- [ ] No API key appears in any file in the repository
