# NAM Web Shaper

**Real-time guitar tone modelling in the browser: circuit pedals, NAM A2 profiles, cabinet IRs, harmonic analysis, offline rendering and in-browser training of NAM A2 profiles.**

NAM Web Shaper is a single self-contained HTML page. It runs entirely in the browser (Chrome or Edge), needs no installation, no plugin and no server-side processing. It is designed to be published on GitHub Pages or opened from any local web server.

Interface languages: English (default), Italiano, Français, Español, 简体中文, 日本語 — selectable with the flag buttons at the top right.

https://github.com/cnnflv/NAM_WEB_SHAPER

---

## Table of contents

1. [What it does](#what-it-does)
2. [Requirements](#requirements)
3. [Quick start](#quick-start)
4. [Publishing on GitHub Pages](#publishing-on-github-pages)
5. [The signal chain](#the-signal-chain)
6. [Blocks in detail](#blocks-in-detail)
7. [Analysis tools](#analysis-tools)
8. [Presets](#presets)
9. [Model export / import (JSON)](#model-export--import-json)
10. [Offline rendering (WAV → WAV)](#offline-rendering-wav--wav)
11. [Training a NAM A2-Lite profile in the browser](#training-a-nam-a2-lite-profile-in-the-browser)
12. [NAM engine notes](#nam-engine-notes)
13. [Performance](#performance)
14. [Repository layout](#repository-layout)
15. [Credits and licenses](#credits-and-licenses)

---

## What it does

NAM Web Shaper takes the guitar signal from an audio interface (for example a BOSS GX-100 connected over USB), a test tone or a WAV file, and shapes it through a configurable chain:

* a **circuit pedal** built from classic analog building blocks (pre-filter, octave rectifier, soft-clip overdrive with op-amp slew rate and dynamic bias shift, two cascaded diode clippers with Si/Ge/LED/MOSFET selection, hard clip / fold distortion, post-filter);
* up to **three NAM A2 profiles in cascade** (`.nam` files from [TONE3000](https://www.tone3000.com)), run by a JavaScript port of the official NeuralAmpModelerCore *a2_fast* engine, with the same Normalize, input calibration and calibrated hand-off rules as the TONE3000 plugin;
* a **cabinet impulse response** (`.wav`) through a zero-latency partitioned convolution, handled like the TONE3000 plugin does (energy normalisation, −18 dB cab pad, 10 s cap, trimming);
* noise gate, sustain/compressor, wah, 3-band EQ with high cut, delay and reverb.

Everything you hear is also measured: oscilloscope, transfer curve (static or dynamic), theoretical and measured harmonics with even/odd balance and THD, output spectrum. The current model can be exported as JSON (parameters plus the f(x) curve) — the starting point for designing a real circuit that reproduces the sound.

Finally, the whole chain can be rendered offline on a WAV file, and a NAM A2-Lite profile of the resulting tone can be trained directly in the browser.

## Requirements

* **Browser:** Google Chrome or Microsoft Edge (Web Audio API with AudioWorklet, `setSinkId` for output selection). Firefox and Safari load the page but may lack output-device selection or be slower.
* **Audio:** any audio interface recognised by the browser. If the interface exposes the guitar on channel 1 or 2, choose it in the *Channel* selector. Interfaces named "GX-100" are preselected.
* **Secure context:** the page must be served over `https://` (GitHub Pages) or from `http://localhost`. Opening it as `file://` works for the test tone and WAV sources, but Chrome may refuse microphone access.
* No network access is required after the page is loaded.

## Quick start

1. Open the page (GitHub Pages URL, or a local server: `python3 -m http.server 8000` in the folder, then `http://localhost:8000/`).
2. Pick a language with the flags (top right).
3. Choose the **Source**: live audio input, test tone (sine, frequency and amplitude adjustable) or a WAV file played in a loop.
4. Select input and output devices, then press **Start audio**. Grant the microphone permission when asked. On a modeller/pedalboard, send a clean DI to the computer (all blocks off) and mute its direct monitoring.
5. Turn blocks on/off and move the sliders; everything is applied in real time. Use **Bypass** to compare with the dry signal and the **Listening point** selector to monitor the signal after a specific block.
6. Load a `.nam` profile in one of the NAM blocks and an IR `.wav` in the IR block (an example profile is included in the repository).

## Publishing on GitHub Pages

1. Create a repository and copy the contents of this folder into it (`index.html`, `README.md`, `LICENSE`, `NOTICE`, `.nojekyll`, the example `.nam`).
2. In the repository settings open **Pages**, choose *Deploy from a branch*, branch `main`, folder `/ (root)`. (GitHub Pages on a *private* repository requires a paid GitHub plan; on a free account make the repository public first.)
3. After a minute the site is available at `https://<user>.github.io/<repo>/`. Everything runs client-side; GitHub only serves the static files.

## The signal chain

The control panels are laid out in the exact order in which the signal travels:

```
Input gain
  → Noise gate (input position)
  → Sustain (compressor)
  → Wah
  → Circuit pedal            (can be moved after the NAM blocks)
       1 pre-filter (high-pass)
       2 octave (full-wave rectifier)
       3 overdrive (soft clip, slew rate, bias shift)
       4 diode clipping, stage 1
       5 diode clipping, stage 2
       6 distortion (hard clip / fold)
       7 post-filter (low-pass)
       even/odd harmonic weights
  → NAM 1 → NAM 2 → NAM 3   (A2 profiles in cascade)
  → Equalizer (bass / mid / treble / high cut)
  → Noise gate (output position)
  → Delay
  → Reverb
  → IR (cabinet impulse response)
  → Output level
  → "Harmonics only" analysis filter (live only)
```

If the circuit pedal is set to *after the NAMs*, its panels physically move after the NAM blocks so that the page always mirrors the real chain.

## Blocks in detail

### Input
*Input gain* (0.1–20×) and the amplitude of the internal test tone. The input gain is applied before everything else, so it also drives the NAM profiles harder or softer.

### Noise gate
Three algorithms: *gate* (hard cut with soft hysteresis), *expander* (gentle 1:4 downward expansion), *hush* (progressively closes a low-pass filter instead of muting). Threshold and release are adjustable; the gate can be placed at the input (before the distortion, usually the most effective) or at the output. The current gate opening is shown in the oscilloscope status line.

### Sustain
A feed-forward 4:1 compressor with automatic make-up gain. The *Sustain* slider (0–100 %) lowers the threshold from −10 dB to −50 dB; release is adjustable.

### Wah
Resonant band-pass filter swept between 350 Hz and 2.2 kHz. *Manual* mode uses the pedal position slider; *auto-wah* follows the signal envelope (sensitivity and range, which may be negative for a reverse sweep). Resonance (Q) and mix are adjustable.

### Circuit pedal
The pedal is a static non-linearity surrounded by filters, processed with 4× oversampling to avoid aliasing. It can be *circuit* (stages), *harmonic* (Chebyshev synthesis) or *off*.

| Stage | Controls | Notes |
|---|---|---|
| 1 Pre-filter | cutoff 20–1500 Hz, 6/12 dB/oct | high-pass before clipping (Tube Screamer ≈ 720 Hz, Big Muff ≈ 100 Hz) |
| 2 Octave | amount 0–100 % | full-wave rectification (Octavia / Green Ringer): generates the upper octave and even harmonics |
| 3 Overdrive | type tanh / atan / cubic / tube, drive 0.5–100×, bias, op-amp slew rate, dynamic bias shift | the drive is a real gain into the soft clip; *tube* has more headroom on the negative half-wave; the slew-rate limiter reproduces the LM308 grain of a RAT; bias shift moves the operating point with the signal envelope (fuzz sputter / gating) |
| 4 Diode clipping 1 | diode type Si / Ge / LED / MOSFET / custom, gain, threshold Vf, symmetric ⟷ asymmetric, knee | selecting a diode type sets typical Vf and knee, which stay editable; asymmetry scales Vf+ and Vf− in opposite directions (1 vs 2 diodes) |
| 5 Diode clipping 2 | same as stage 1 | cascaded second clipper (Big Muff) |
| 6 Distortion | hard clip or fold-back, gain, threshold | |
| 7 Post-filter | cutoff 100 Hz–12 kHz, 6/12 dB/oct | low-pass after clipping (the RAT "Filter" knob) |

**Harmonic mode:** instead of circuit stages, you set the amplitudes of harmonics H2…H8 directly. The curve is synthesised with Chebyshev polynomials, so on a sine of amplitude 1 the output contains exactly those harmonics; drive > 1 adds clipping.

**Even / odd harmonics:** in both modes the resulting curve f(x) is split into its even and odd parts, each weighted by its own slider (0 removes it). Setting *even* to 0 removes all even harmonics; the asymmetry controls (bias, asymmetric diodes, octave) are what generates them.

### NAM 1–3
Each block loads a `.nam` file: A2 *SlimmableContainer* files from TONE3000 (two sizes, LITE = 3 channels and FULL = 8 channels) or plain WaveNet files (A1 / custom, without gating or FiLM). Controls per block:

* **Size** LITE/FULL (default LITE, as in the plugin);
* **Normalize** (on by default): output gain = target − profile loudness, clamped to ±12 dB, target adjustable (−18 dB default);
* **Calibrate input**: gain = user dBu − profile `input_level_dbu` (if the profile carries calibration data);
* **Level In / Level Out** ±24 dB.

Between two consecutive calibrated NAM blocks the output of the first is handed to the second using `output_level_dbu → input_level_dbu`, exactly as the TONE3000 plugin does; otherwise normalisation applies. The info line shows architecture, size, loudness, calibration data and whether the file has the A2 shape (fast engine). If a profile is tagged as a cabinet and the IR is on, a "two cabinets" warning is shown.

### Equalizer
Bass shelf (120 Hz), mid peak (800 Hz), treble shelf (3 kHz), ±12 dB with centre = neutral, plus a high cut (2–12 kHz) usable as a simple cabinet simulation.

### Delay and reverb
Feedback delay (20–1000 ms, repeats, mix, damped repeats) and a Freeverb-style reverb (8 combs + 4 all-passes: size, damping, mix).

### IR
Loads a cabinet impulse response (`.wav`). Following the TONE3000 plugin rules: channel 1 is used, the file is capped at 10 s, resampled to the audio-context rate, trimmed of leading/trailing silence, energy-normalised (`gain = clamp(1/‖h‖₂, −48 dB, 1)`, never a boost); short IRs (≤ 1 s, cabinets) get a −18 dB pad and 100 % mix by default, long IRs (reverbs) get no pad and 50 % mix. Controls: on/off, pad, mix, level ±24 dB. The convolution is a frequency-domain partitioned convolution with zero latency (128-sample partitions; long IRs use a 4096-sample head plus 4096-sample tail partitions).

### Output
Final level.

## Analysis tools

* **Oscilloscope:** input (grey) and output (green), triggered on the input zero crossing. The slider below (0–2000 ms, 100 ms steps) switches from real time (0) to coherent averaging of the triggered waveforms over the chosen period, refreshed once per period. *Freeze waves* holds the display. The status line shows peaks, the DSP load of the audio thread, the gate opening and the detected f0.
* **Transfer curve f(x):** with the circuit pedal alone, the static curve with its even (amber) and odd (green) parts and the identity line; the light band marks the test-tone amplitude. When a NAM block is active (or the pedal is off), a dynamic x→y curve of the whole chain on a 220 Hz sine is drawn — NAM profiles have memory, so the hysteresis is real.
* **Harmonics:** bars = theoretical harmonics H1…H10 of the whole chain on the test tone (computed from the DSP itself, including filters, oversampling and NAM), dots = harmonics measured from the actual output when the source is the test tone. Odd harmonics in green, even in amber; THD and the even/odd energy ratio are shown.
* **Spectrum:** FFT (16384 points) of the output. *Spectrum: real time / average* (averaged over the oscilloscope period) and *Freeze spectrum*.
* **HARMONICS ONLY:** a real-time comb filter on the output that lets only multiples of the fundamental through. f0 comes from the test tone, or from a YIN pitch detector on the clean input when playing guitar.
* **Listening point:** monitor the signal after the circuit pedal or after any NAM block.

## Presets

Genre presets (clean, blues, country/edge, classic rock, hard rock, heavy metal, thrash, modern hi-gain, liquid lead, fuzz), classic-pedal presets (Tube Screamer, Klon-style, RAT, Turbo RAT, Big Muff, Fuzz Face, Octavia, velcro fuzz, auto-wah + overdrive), stage-type presets and harmonic-synthesis presets. Preset levels assume a guitar peaking around −20 dBFS at the input.

## Model export / import (JSON)

*Export JSON* downloads the full model: chain description, all block parameters, the names of the NAM profiles and IR in use, the theoretical harmonics in dB and the static curve f(x) sampled at 201 points. *Import JSON* reloads a model pasted in the text area. The JSON is the hand-over document for the next step of the project: synthesising an electronic circuit that reproduces the modelled sound.

## Offline rendering (WAV → WAV)

In the *Offline render and NAM A2 training* panel:

1. Load an input WAV (a clean DI; the standard NAM capture file `v3_0_0.wav` is ideal, or your own recording). Any sample rate is accepted; the file is resampled to 48 kHz.
2. Press **Process → download in/out WAV**. The whole chain (pedal, NAM blocks, EQ, delay, reverb, IR — everything except the analysis comb filter) is applied with the current settings, and two 24-bit/48 kHz mono files are downloaded: `<name>_input_48k.wav` and `<name>_output.wav`.
3. Upload the pair to [tone3000.com/capture](https://www.tone3000.com/capture) to obtain an official **NAM A2 FULL** profile of your tone.

## Training a NAM A2-Lite profile in the browser

The same panel can train a profile locally, without any installation:

* choose how many seconds of the file to use and the number of epochs, then press **Train NAM A2-Lite (local)**;
* a Web Worker trains a WaveNet with the A2 topology (23 layers, kernels 6/15, LeakyReLU, 1×1 layers, 16-tap head, 3 channels) using manual back-propagation, Adam, MSE + pre-emphasis loss and the last seconds as validation set;
* the log shows train/validation ESR per epoch; **Stop** interrupts at any time; **Download output.nam** always gives the best model so far (lowest validation ESR).

The resulting `output.nam` is a *SlimmableContainer* that contains the Lite model in both size slots, so it loads in any A2-capable host (TONE3000 plugin, NAM Web Shaper itself, hardware with A2 support). Training time is roughly equal to the audio duration per epoch on a modern laptop (e.g. 120 s × 8 epochs ≈ 15 minutes). Delay, reverb and IR tails longer than the network's 132 ms memory cannot be captured by a NAM profile — keep the IR as a separate `.wav`.

## NAM engine notes

* The engine is a JavaScript port of the `a2_fast` path of [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore): frame-major buffers, per-layer history rings, tap-major convolution with the last tap fused with the mixin, activation, head sum and 1×1 layer; fully unrolled kernels generated at load time for 3 and 8 channels; prewarm over the receptive field (6347 samples). Output matches the reference implementation to ~5·10⁻⁷.
* Normalize, input calibration, calibrated hand-off and LITE/FULL selection follow the open-source [TONE3000 plugin](https://github.com/tone-3000/tone3000-plugin) (`Processor.cpp`).
* If the audio context does not run at the profile's sample rate (48 kHz), a cubic-interpolation rate converter wraps each NAM block.

## Performance

* The DSP load of the audio thread is displayed in the oscilloscope status line. A2-Lite profiles are light; A2-Full costs about 7× more. If audio drops out, switch intermediate blocks to LITE or reduce the number of cascaded profiles.
* Oversampling (4×) is applied only to the circuit pedal; NAM profiles run at their native rate.
* IR convolution cost grows with IR length; cabinet IRs (< 100 ms) are negligible.

## Repository layout

```
index.html                       the application (single file: engine, DSP chain, trainer, UI, translations)
esempio_marshall_jcm800_a2.nam   example NAM A2 profile (Marshall JCM800, TONE3000 demo model)
README.md                        this file
LICENSE                          freeware license (Flavio Canonico)
NOTICE                           third-party attributions (MIT)
.nojekyll                        tells GitHub Pages to serve the files as they are
```

## Credits and licenses

NAM Web Shaper © 2026 Flavio Canonico — freeware, see `LICENSE`.

It contains code derived from, and behaviour modelled on, the following open-source projects (MIT License, see `NOTICE`):

* [Neural Amp Modeler](https://github.com/sdatkinson/neural-amp-modeler) and [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore) by Steven Atkinson — `.nam` file format, WaveNet/A2 architecture and the `a2_fast` inference path;
* [TONE3000 plugin](https://github.com/tone-3000/tone3000-plugin) and [neural-amp-modeler-wasm](https://github.com/tone-3000/neural-amp-modeler-wasm) by TONE3000 — normalisation, calibration, IR handling rules and the example JCM800 profile.

NAM A2 and TONE3000 are trademarks of their respective owners; this project is not affiliated with them.
