# Karaoke Autotune

Sing into a USB mic and hear yourself autotuned to whatever key the song on your PC is in. It follows the music from song to song, and through key changes.

```
YouTube Music / Apple Music ─────────────────────────────────────────► speakers
     ├─ Windows loopback ──► key detector ───────┐
     └─ Windows "now playing" ──► GetSongBPM ────┴─► key decision
                                                         │ sets the scale notes
USB mic ──────────────────────► Graillon 3 autotune ──► reverb ─────► speakers
```

Two key sources check each other. The audio detector hears the actual recording and catches key changes. The GetSongBPM lookup knows the published key the instant a song starts. Details in "How the key gets picked" below.

## Setup, once

1. **Python 3.12 or newer** from python.org. Tick "Add python.exe to PATH" in the installer.
2. **Graillon 3 Free** from https://www.auburnsounds.com/products/Graillon.html. Run the Windows installer, keep VST3 checked.
3. Double-click **setup.bat**. It makes a private Python environment in `.venv` and installs everything.
4. In Windows Sound settings, open your mic's and your output's Properties > Advanced and set both to **48000 Hz**. Mismatched rates are the #1 cause of "couldn't open the audio devices".
5. **Optional, for the lookup:** get a free key at https://getsongbpm.com/api. They require a backlink to getsongbpm.com, so push this folder to a public GitHub repo (this README already links them) and use the repo URL for "Website URL" and "Backlink URL". Paste the key into `config.toml` under `[lookup]`. Without a key it runs audio only, which works fine too.

`config.toml` is in `.gitignore` so your API key doesn't get pushed.

## First run

1. `run.bat devices` lists your mics and outputs. Put part of each name into `config.toml`, like `mic = "USB"` and `output = "HDMI"`.
2. `run.bat probe` loads Graillon and checks the app can flip its scale notes. You want `Key control: toggles` and readback lines that match. If it says NOT FOUND, send the printout to Claude or map the parameters by hand at the bottom of `config.toml`.
3. `run.bat keytest` with music playing. No mic needed. You'll see lines like:
   ```
   20:14:03  song      Olivia Rodrigo - good 4 u  (Chrome)
   20:14:03  autotune  -> chromatic (all notes)  [none] new song, all notes allowed until the key is known
   20:14:04  lookup    GetSongBPM says F# minor (good 4 u by Olivia Rodrigo, 166 BPM)
   20:14:04  autotune  -> F# minor  [lookup] published key from GetSongBPM
   20:14:12  music     sounds like A major (fit 0.79, margin 0.14, 9s)
   20:14:12  autotune  -> F# minor  [audio+lookup] audio confirms the published key
   ```
   A major and F# minor share the same 7 notes, so that counts as agreeing.
4. `run.bat` to sing. Graillon's window opens and you should see its keyboard change notes when the key changes. Close the window or hit Ctrl+C to stop.

`run.bat run --key "A minor"` forces a key and skips detection. `run.bat keytest --file song.mp3` runs the detector over a file and prints a timeline.

## Making it sound right

All in Graillon's window, and your tweaks are saved when you quit:

- **Smooth** all the way down is the hard T-Pain sound. Turn it up for natural correction.
- **Correction Amount** 100% for full effect.
- **Gate** cuts room noise and speaker bleed between lines.
- Reverb and vocal level are in `config.toml` under `[audio]`.

## Delay and speakers

Use wired speakers or HDMI. AirPlay, Cast and Bluetooth add way too much delay to sing over yourself.

On startup it prints the buffer and plugin delay, like `buffer 256 (5.3 ms), plugin 2.0 ms`. Windows adds some on top. If you hear crackles, raise `buffer_size` to 512. On a TV over HDMI, turn on Game Mode. Keep the mic pointed away from the speakers so it doesn't feed back.

## How the key gets picked

**Audio.** Every 2 seconds it takes the last 20 seconds of what Windows is playing, FFTs it, folds the peaks onto the 12 notes, and correlates that against major and minor key profiles for all 12 roots. It also measures how far the recording sits from A440 and undoes the pull every note's 3rd harmonic has toward the key a fifth up. It reports a **fit** and a **margin** over the next-best scale.

**Autotune math.** Graillon detects your pitch f, turns it into a note number n = 69 + 12·log2(f/440), snaps n to the nearest allowed note, and shifts your voice by 2^((target − n)/12). This app's job is setting which 7 notes are allowed.

**Rules**, in `ktune/brain.py`:

- New song: all 12 notes allowed until something is known.
- The GetSongBPM key is used the moment it arrives, and whenever the audio isn't sure.
- Audio agreeing with the lookup locks it in.
- Audio can overrule the lookup only after it's strongly sure 3 windows in a row, which covers key changes and live versions.
- With no lookup, audio has to say the same thing twice before the key switches.

All the thresholds are in `config.toml` under `[keys]`.

## Troubleshooting

| Problem | Fix |
|---|---|
| `No mic device matches` | `run.bat devices`, then fix `mic` in config.toml |
| `Couldn't open the audio devices` | Both devices to 48000 Hz in Windows Sound settings, or `sample_rate = 0` |
| Crackles or dropouts | `buffer_size = 512` |
| `can't read what's playing from Windows` | Needs Windows 10 1809 or newer. Detection still works from the audio |
| Music plays on a different output than the default | Set `music_source` to part of that output's name |
| Audio never gets "sure" | Run `keytest`, look at the fit and margin numbers for your music, lower `min_fit` a bit |
| Key control NOT FOUND | Send Claude the `run.bat probe` printout |

One quirk: key detection listens to the default output, which also carries your autotuned voice. Since your voice is already snapped to the key it mostly agrees with the music, but it can slow down catching a key change by a few seconds.

## For tinkering

```
karaoke.py          command line: devices, probe, keytest, run
ktune/keydetect.py  chroma + key profiles, the DSP part
ktune/brain.py      the rules for audio vs lookup
ktune/lookup.py     GetSongBPM client with title cleanup and a cache
ktune/nowplaying.py Windows media session reader
ktune/loopback.py   WASAPI loopback capture into a ring buffer
ktune/plugin_ctl.py finds Graillon and flips its note switches
tests/              pytest suite with a fake-song synthesizer
```

Tests: `.venv\Scripts\python -m pip install pytest`, then `.venv\Scripts\python -m pytest`.

## Credits

Song keys from [GetSongBPM](https://getsongbpm.com). Autotune by [Graillon 3](https://www.auburnsounds.com/products/Graillon.html) from Auburn Sounds, installed separately. Live audio through Spotify's [pedalboard](https://github.com/spotify/pedalboard).
