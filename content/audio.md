---
title: "Sound"
weight: 29
description: "A container plays sound with wctr run --audio: the host module wctr_audio_out, the page's AudioWorklet, volume and mute per container and for the page, and Quake with sound. Engine v0.40.0, playground v0.10.0."
tags: [audio, playground, quake]
---

A container can play sound in the playground. It is opt-in, like the GPU:

```console
$ wctr run -d --audio tone            # a 440 Hz sine; click the page once
$ wctr run --rm --gpu --audio -m 512m quake --map e1m1
```

Engine v0.40.0 adds `run --audio` (the device capability `audio`, independent of
`--gpu`, with no option) and the host module **`wctr_audio_out`**. It is not a
`wasi_` module: no WASI proposal covers audio (the proposal list has no audio, PCM
or media entry in any phase), so the import is private, like `wasi_webgpu_batch`,
and is kept small so that it can be lowered onto a standard one if one appears.
The design, with its prior art and threat model, is
[`docs/audio-design.md`](https://github.com/go-wasmcontainers/engine/blob/main/docs/audio-design.md)
in the engine.

## What a guest sees

Four core-wasm functions, every parameter an `i32`, every result an errno. Output
only: there is no capture.

| Function | Effect |
|---|---|
| `stream_open(rate, channels, capacity, out)` | rate 8000 to 48000 Hz, 1 or 2 channels, capacity 256 to 16384 frames (a power of two). The handle is stored at `out`. The host never picks another rate: out of bounds is `InvalidArgument` |
| `stream_write(stream, ptr, frames, out)` | copies and queues up to `frames` interleaved little-endian `float32` frames, never blocking; stores `{accepted, queued}`. `frames = 0` is a poll. Samples are clamped to [-1, 1] and a non-finite one becomes 0 |
| `stream_stats(stream, out)` | stores `{played, underruns, refused, state}`; `state` is 0 waiting, 1 playing, 2 closed |
| `stream_close(stream)` | closes it; the process ending does too |

Errors: `Denied` (no `--audio`), `Unsupported` (no audio provider: **native `wctr`
has no audio output**), `TooLarge`, `QuotaExceeded` (2 streams per container, 8 per
page), `Fault` (a range outside the guest's memory, computed in 64 bits).

## The page

The engine worker allocates one `SharedArrayBuffer` ring per stream and the page
plays it through an **AudioWorklet** (the `audioring` module of
[wasmbox](https://github.com/wasmdesk/wasmbox), served as `audio-worklet.js`, byte
for byte), one `AudioContext` for the page. A browser lets a page play sound only
after a user gesture, so **click the page once** (or press a key in it). Until
then the stream is *waiting*: the host accepts what the container writes and
discards it, so sound made before the click is never played late. The host knows
the page plays from a heartbeat the worklet advances every render quantum.

**Volume and mute** work at two levels. Each container's **dashboard pane** (the
Surfaces view, dashboard v0.12.0) has a sound line with *Vol -*, *Vol +* (steps of
10) and *Mute* for the sound of its container, and a *Sound* strip at the bottom
left of the page has the page-wide *mute all*. They are gain nodes between the
worklet and the speakers, so a container cannot undo them. A container with no
surface has no pane, so no per-container control. The end-to-end test clicks the
pane's buttons against Quake's real sound: what reaches the speakers over what the
worklet played is 1.000 at full volume and 0.300 at 30 %, and 0 muted.

## `tone`

`wctr run -d --audio tone [--hz N] [--silent] [--seconds N]` plays a sine
(440 Hz, 11025 Hz stereo, amplitude 0.25), keeping 60 ms queued. It prints the
mean cost of a `stream_write` call at the end (one engine round trip each; about
0.2 ms in headless Chromium on an Apple-silicon Mac). Without `--audio` it prints that
the host answered *denied* and exits 2.

The end-to-end test (headless Chromium with
`--autoplay-policy=document-user-activation-required`) asserts the context is
suspended before any gesture, clicks, and over 2 s asserts that at least half of
the frames played, the analyser's peak is in the 440 Hz bin (plus or minus one), the
RMS is above -30 dBFS and fewer than 5 underruns happened. Four planted negatives
must each fail their own assertion: `tone --silent` (the RMS), `tone --hz 880` (the
frequency), no click (nothing plays, the context stays suspended) and no `--audio`
(*denied*, no stream).

## Quake with sound

The `quake` image does **not** run with `--audio` by default; add it
(`wctr run --rm --gpu --audio -m 512m quake`). The game's backend opens one stream
(11025 Hz stereo, 1024 frames), converts its mix to `float32` and writes once a tic;
the host's answer is the queue its mixer is paced by. Without the grant, on native
`wctr`, or in a build without the `wctraudio` tag, the game plays silent and raises
nothing.

The mixer used to paint a fixed 512 frames a tic, whatever the tic's length: 2.8
times real time at 60 tics a second. It is now paced like tyrquake's `S_Update`
(go-quake1/engine v0.22.0). Measured as mix frames offered per second of game time
over the 11025 an output plays, in headless Chromium on a real GPU (300 frames of
e1m1): **2.77 before, 1.01 after**.

### The mixer runs in the page's worklet

With go-quake1/engine v0.56.0 and go-wasmcontainers/engine v0.49.0 the game no
longer mixes on its own thread. It loads its samples once and sends *play*,
*stop* and *volume* commands (the voices of `wctr_audio_out`); the page's
AudioWorklet (wasmbox `audioring` v0.3.0, the `wasmbox-voices` worklet) paints the
voices **0.1 s ahead** on the audio thread. A garbage collection or a slow frame
on the game's thread then cannot starve the speakers: the game prints `QUAKE:
sound on (wctr_audio_out voices, mixed in the page)`, and `--fixed-mixer` forces
the old single stream.

### How a guest and its GPU host worker attach

A guest's GPU calls are served by a second worker (`role=host`), so a present or a
submit does not wait for the engine's thread. Since go-wasmcontainers/engine
v0.49.1 the guest's memory (a `SharedArrayBuffer`) and the control block do not
travel on the guest's own port: the guest posts its memory to the engine, the
engine relays it on its control channel to the host worker as `{t: "attach"}`,
and the host answers `{t: "attached"}` on the guest's port. The guest starts only
after that answer (or traps with "the gpu host worker did not answer" after 30
s). Firefox keeps a message body that carries a `SharedArrayBuffer` in a table
keyed by the sending port and forgets it when that port closes, so the old way
ended in `attach-refused` there (go-wasmcontainers/engine#84).
