

![ESPHome + Home Assistant Voice Assistant on the Guition JC3636K718C](assets/header.jpg)

# ESPHome Voice Assistant for the Guition JC3636K718C (round knob display)

A full-featured **Home Assistant Voice Assistant** running on the **Guition
JC3636K718C** - a 1.8" round 360×360 touch display with a rotary knob, speaker,
microphone and an **addressable LED ring**, all driven by a single ESPHome YAML
(no custom C firmware).

It started as "my kid needs a physical timer" and turned into a whole puck. 🙂

## Demo



https://github.com/user-attachments/assets/33c2a708-5f8b-4bab-a165-285cbfd919d9



<div align="center">
  <video src="YOUR_VIDEO_URL" controls width="400"></video>
</div>

[Watch the demo video (full quality, with sound)](assets/demo.mp4)

## What it does

- **Voice assistant** - on-device wake word ("Alexa") via `micro_wake_word`, full
  Home Assistant Assist pipeline (STT / LLM / TTS), wake beep + music ducking.
- **Music player** - `speaker` media player visible in HA / Music Assistant, with
  album art, title/artist, transport buttons and a progress bar.
- **Timers** - set by knob or by voice; big countdown with a depleting ring,
  pause/stop, and an alarm (sound + on-screen + LED) when it finishes.
- **Device control** - a tiles screen toggling your lights/switches.
- **Home screen** - clock, date, battery, weather + room temp/humidity.
- **LED ring** - controllable from HA *and* reactive: assistant (comet/spinner/wave),
  timer countdown, alarm flash, volume bar - each reaction toggleable in Settings.
- **Two built-in arcade games** (a lane racer and a vertical shooter) for the kid.

Everything is navigated with **swipes + taps on the screen** and the **rotary knob**.

## Documentation

Full docs live in the **[wiki](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki)**:

| Page | What's inside |
|---|---|
| [Hardware](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Hardware) | Board specs, full pinout, what to buy on AliExpress |
| [Installation](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Installation) | Requirements, first flash (USB), OTA, the bundled sounds |
| [Usage](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Usage) | Gestures, screens, the settings menu, the LED ring |
| [Configuration](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Configuration) | Change entities, tiles, wake word, run without Music Assistant |
| [Troubleshooting](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Troubleshooting) | Known issues (battery %, GPIO0 strapping, performance, the knob) |

## Quick start

1. Copy `secrets.example.yaml` → `secrets.yaml` and fill in your Wi-Fi.
2. Edit the `substitutions:` at the top of `guition-va.yaml` (HA URL + your entity IDs).
3. Images and sounds are **fetched from GitHub at compile time** (like the fonts), so you do not have to copy the `assets/` folder anywhere - only `guition-va.yaml` and `partitions.csv` need to sit together.
4. **First flash over USB** - easiest via the **ESPHome dashboard** (GUI) or the CLI; the 16 MB partition table can't be set over OTA, so the first flash is USB, then updates go wireless.
5. In Home Assistant: open the new ESPHome device → assign an Assist pipeline.

Full details on the [Installation](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Installation) wiki page.

## Repository layout

```
guition-va.yaml            # the whole device config (English UI)
partitions.csv             # 16 MB partition table (MAX app slots)
secrets.example.yaml       # copy to secrets.yaml
assets/                    # fetched from GitHub at compile time (no need to copy locally)
  header.jpg               # banner
  sounds/                  # wake.wav + alarm.wav
  sprites/cool-cars/       # "Cool Cars" game graphics
  sprites/space-wars/      # "Space Wars" game graphics
scripts/
  make_sounds.py           # (re)generate the wav sounds
  esplog.py                # stream device logs over the native API
skill/                     # Claude Code skill: hardware spec + gotchas
```

## Claude Code skill

This repo ships a [Claude Code](https://claude.com/claude-code) skill at
[`skill/guition-jc3636k718c/`](skill/guition-jc3636k718c/SKILL.md). It gives the
assistant the correct pinout, ESPHome component choices, and the hard-won gotchas
(the knob isn't quadrature, GPIO0 ring strapping, 16 MB partitions need a USB flash,
LVGL performance limits, lambda/string pitfalls, the battery heuristic).

### Install it

So Claude can use it on any project:

- **User-wide** - copy the folder into `~/.claude/skills/`:
  ```bash
  cp -r skill/guition-jc3636k718c ~/.claude/skills/
  ```
- **Per-project** - copy it into that project's `.claude/skills/`.

Start a new Claude Code session and ask anything about this board; the skill loads
automatically. See the [wiki](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Claude-Code-Skill) for details.

## Credits / notes

- Pinout and the display `init_sequence` come from the **official manufacturer demo**
  (`JC3636K718_knob_EN`) - this is the correct pinout for the **K718C** board, which
  differs from the otherwise-similar **JC3636W518**.
- Built with [ESPHome](https://esphome.io/) + Home Assistant.
