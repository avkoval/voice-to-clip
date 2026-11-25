# voice-to-clip

Press a hotkey, speak, press `q` — the transcript is in your clipboard. Speech recognition runs **locally** with
[whisper.cpp](https://github.com/ggml-org/whisper.cpp); no audio leaves your machine.

```
Model: ggml-large-v3-q5_0.bin | lang=uk | input=pulse/alsa_input.usb-…mono-fallback | vad=on | live=on

🎙  Recording → ~/.local/state/voice-to-clip/voice-20261125-164923.wav
   Press  q  (or Ctrl+C) to stop.
   Text appears after each pause; after q only the last piece is left.

  › So, for example, in the requirements it said which platform we should use.
  › But the constraint probably will be in the case like if…

⏳ Finishing the last piece…
✓ Copied to clipboard
```

## Features

- **Live mode** (`--live`): the text is transcribed piece by piece while you are still speaking, cut at your pauses.
  What you see is the final text, and after `q` only the last few seconds are left to process — typically 2–7 s of
  waiting instead of a full pass over the whole recording.
- **Any language Whisper knows**, including mixed-language speech with `-l auto`. A `--large` switch uses the full
  `large-v3` model, noticeably better for languages other than English (tested with Ukrainian).
- **Pinned microphone**: name your mic in the config, and a webcam that becomes the system default won't silently
  take over. Falls back to the default input when the mic is unplugged.
- **Clipboard that survives**: with [CopyQ](https://hluk.github.io/CopyQ/) the text is also added to the clipboard
  history, so copying something else before pasting does not lose a long dictation.
- **Nothing is thrown away**: every recording (`.wav`) and transcript (`.txt`) is kept in the output directory.
- **GPU when available**: whisper.cpp uses Vulkan/CUDA if it was built with them; works on an Intel laptop iGPU.

## How it works

The hotkey starts the script, which opens a small [kitty](https://sw.kovidgoyal.net/kitty/) window and records the
microphone with `ffmpeg` until you press `q`. Then:

- **Without `--live`**: `whisper-cli` transcribes the whole recording.
- **With `--live`**: a `whisper-server` with the model already loaded runs next to the recorder. The script watches
  the audio level, cuts the recording at each pause (≥ 0.4 s, pieces of 3–28 s) and sends each new piece to the server,
  together with the end of the previous text for continuity. While the server is busy, the next piece simply grows, so
  transcription never falls behind. If anything in live mode fails, the script falls back to transcribing the whole
  file — you still get your text.

The result goes to the clipboard and is shown in a desktop notification.

## Requirements

Linux with PipeWire or PulseAudio. Developed and used daily on Arch Linux with X11 and i3; the clipboard part also
supports Wayland (`wl-copy`), but that path is less tested.

| Tool                     | Why                                   | Arch package                                       |
|--------------------------|---------------------------------------|----------------------------------------------------|
| [babashka](https://babashka.org) | runs the script               | `babashka-bin` (AUR)                               |
| whisper.cpp              | `whisper-cli`, `whisper-server`       | `whisper.cpp` (AUR; pulls `libggml` with Vulkan)   |
| ffmpeg                   | recording                             | `ffmpeg`                                           |
| kitty                    | the recording window                  | `kitty`                                            |
| curl                     | talks to `whisper-server` in live mode | `curl`                                            |
| pactl                    | finding the microphone                | `libpulse`                                         |
| notify-send              | notification with the result          | `libnotify`                                        |
| CopyQ (recommended), or wl-clipboard, or xclip | clipboard       | `copyq` / `wl-clipboard` / `xclip`                 |

On other distributions install the same tools from your package manager; whisper.cpp can be
[built from source](https://github.com/ggml-org/whisper.cpp#quick-start) (enable `-DGGML_VULKAN=1` or
`-DGGML_CUDA=1` for GPU).

## Installation

```sh
git clone https://github.com/avkoval/voice-to-clip.git ~/src/voice-to-clip
ln -s ~/src/voice-to-clip/voice-to-clip ~/bin/voice-to-clip     # any directory on your PATH
```

Download the models (≈ 0.5 GB for turbo, 1.1 GB for large-v3, 1 MB for the voice activity detector):

```sh
mkdir -p ~/Models/whisper.cpp && cd ~/Models/whisper.cpp
curl -L -O https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo-q5_0.bin
curl -L -O https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-q5_0.bin   # for --large
curl -L -O https://huggingface.co/ggml-org/whisper-vad/resolve/main/ggml-silero-v5.1.2.bin   # for --vad
```

Try it in a terminal first:

```sh
voice-to-clip --inline --live -l en
```

## Configuration

Optional: `~/.config/voice-to-clip/config.edn` (see [examples/config.edn](examples/config.edn)). Every key may be
omitted; command-line options override it.

```clojure
{:device     "alsa_input.usb-Your_Microphone-00.mono-fallback"   ; from `pactl list short sources`
 :language   "en"                                                ; or "uk", "de", "auto", ...
 :models-dir "~/Models/whisper.cpp"
 :output-dir "~/.local/state/voice-to-clip"
 :threads    4}
```

To find your microphone's name:

```sh
pactl list short sources | grep -v monitor
```

## Usage

```
voice-to-clip [OPTIONS]

  -l, --language LANG     Whisper language (uk, en, de, auto, ...)
  -m, --model PATH        Path to a ggml-*.bin model (default: turbo, or large-v3 with --large)
      --large             Use the full large-v3 model (slower, better for non-English)
      --live              Transcribe piece by piece while recording
  -t, --threads N         Whisper CPU threads (= physical cores)
  -d, --device DEV        Audio input source name; default: :device from config, else system default
      --input-format FMT  ffmpeg input format (pulse|jack|alsa)
      --vad               Voice activity detection (skip silence)
      --vad-model PATH    Path to the Silero VAD model
      --inline            Run in the current terminal instead of a new kitty window
      --hold              Keep the window open after success
      --no-keep-wav       Delete the recording after transcription
  -h, --help              Show help
```

Recommended combinations:

| Command                               | When                                                        |
|---------------------------------------|-------------------------------------------------------------|
| `voice-to-clip --vad --live -l en`    | English: the turbo model is fast and accurate               |
| `voice-to-clip --vad --live --large`  | Other languages (set `:language` in the config)             |
| `voice-to-clip --vad --live -l auto`  | Mixed languages; Whisper detects the language per piece     |

## Hotkeys

### i3

A mode keeps the hotkeys short: `$mod+n`, then `u`, `e` or `l`. Full snippet in
[examples/i3-config](examples/i3-config):

```
mode "voice" {
        bindsym u      exec --no-startup-id voice-to-clip --vad --live --large ; mode "default"
        bindsym e      exec --no-startup-id voice-to-clip --vad --live -l en ; mode "default"
        bindsym l      exec --no-startup-id voice-to-clip --vad --live -l auto ; mode "default"
        bindsym Return mode "default"
        bindsym Escape mode "default"
}
bindsym $mod+n mode "voice"

for_window [class="voice-to-clip"] floating enable, resize set 900 400, move position center
```

Or a single key: `bindsym $mod+v exec --no-startup-id voice-to-clip --vad --live`.

### Sway

The same `mode` block works; for the window rule use `app_id` instead of `class`:

```
for_window [app_id="voice-to-clip"] floating enable, resize set 900 400, move position center
```

### GNOME, KDE, XFCE and others

Add a custom keyboard shortcut (Settings → Keyboard → Custom Shortcuts in GNOME, System Settings → Shortcuts in KDE,
Keyboard → Application Shortcuts in XFCE) with the command:

```
/full/path/to/voice-to-clip --vad --live
```

Use the full path: desktop shortcuts often don't see `~/bin` on the PATH. With
[sxhkd](https://github.com/baskerville/sxhkd):

```
super + v
    voice-to-clip --vad --live
```

## Tips

- **Microphone level matters more than the model.** Quiet input is the most common cause of missing or invented
  words. Record a test and check the peaks reach roughly −6…−3 dB:
  `ffmpeg -i recording.wav -af volumedetect -f null - 2>&1 | grep max_volume`.
  Raise the input with `pactl set-source-volume <source> 85%`.
- **Set the language** when you know it. A wrong forced language makes Whisper translate or garble the speech; use
  `-l auto` for mixed speech.
- **Pause briefly between thoughts** in live mode: pauses are where the pieces are cut, so the text appears sooner.
- Whisper sometimes invents phrases on silence ("Thank you.", "Дякую за перегляд!"); live mode drops them when they
  come from short near-silent pieces, and `--vad` reduces them further.

## Troubleshooting

- **The window flashes and closes**: run `voice-to-clip --inline --live` in a terminal to see the error.
- **Wrong microphone**: set `:device` in the config or pass `-d`; the first line of the window shows which input is
  used.
- **Text disappears from the clipboard**: install CopyQ — `xclip`/`wl-copy` hold the clipboard only while their
  process lives, and the text is not kept in any history. Transcripts are always saved as `.txt` in the output
  directory.
- **Slow transcription**: check that whisper.cpp uses the GPU (`ggml_vulkan: Found 1 Vulkan devices` in the
  output); otherwise use the turbo model (drop `--large`).

## License

[MIT](LICENSE)
