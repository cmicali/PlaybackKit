# PlaybackKit: the plan

**Status: planned 2026-10-09, not started.**

PlaybackKit is a toolbox for fast, optimized audio playback on macOS and iOS: the engine, its DSP, its analysis, and its waveform UI. It is extracted from [Vibe](https://github.com/cmicali/vibe), where all of it lives today, so that Vibe and other apps can share it.

**For now its users are Vibe and one other app, not the public.** Both consume it as a git submodule through a shared XcodeGen spec. A Swift package for outside users comes later (the last section).

## Decided

- **The name is PlaybackKit, and the prefix is `PBK`** for every public class and C symbol.
- **Audio code and UI code are separate products.** FX is its own product, starting with the varispeed.
- **One repo.** It is this one.
- **Consumers use a git submodule and a shared XcodeGen spec**, not a Swift package, until outside users matter.

Still open: the license. Apache 2.0 is recommended, like Vibe, so code moves between them freely. Until a license is chosen, nothing here is licensed for reuse.

## The products

| Product | Holds | Depends on | Third-party code |
| --- | --- | --- | --- |
| **PlaybackKit** | The engine: the file handle and decoders, formats, errors, loading, metadata, art, the player, the voice bus, the resampler, levels, devices, and the waveform loader and cache | PlaybackKitFX, PlaybackKitAnalysis, the waveform data | TagLib, PINCache, r8brain, dr_mp3, dr_flac, and dr_wav |
| **PlaybackKitFX** | Realtime DSP stages a render can host. It starts with the varispeed. The DJ chain joins later. | nothing | none |
| **PlaybackKitAnalysis** | The BPM and key analyzers. They take PCM and return a number. | nothing | none |
| **PlaybackKitUI** | The waveform renderers, the morph engine, the theme, the loading indicator, and the macOS and iOS views | the waveform data | none |

Plus one internal target, **PlaybackKitWaveformData**. It holds the waveform type (`AudioWaveform` and its codable form) and includes only system headers.

```
PlaybackKitFX    PlaybackKitAnalysis    PlaybackKitWaveformData
      ^                 ^                  ^              ^
      └──────────── PlaybackKit ───────────┘        PlaybackKitUI
```

**What each split buys:**

- **PlaybackKit without the UI.** A command-line tool, a background analyzer, or an app with its own interface links no view code.
- **PlaybackKitUI without the engine.** An app on AVFoundation can draw these waveforms from its own samples, with no TagLib, r8brain, or player.
- **PlaybackKitFX without the engine.** Any render callback can host the varispeed: an `AVAudioSourceNode`, an Audio Unit, or someone else's player.
- **PlaybackKitAnalysis without anything.** BPM and key from a buffer of PCM.

Each split follows a line Vibe's code already draws. The waveform views reach into the engine only through `AudioWaveform.h`. The analyzers import only `MusicalKey.h`, which moves with them. The varispeed imports only system headers.

**One cycle stays inside PlaybackKit.** `AudioTrack.m` imports `AudioTrackMetadata.h`, because a track carries its tags. The metadata code imports `AudioTrack.h` and the file handle. So metadata and the player stay in one product. To split them, `AudioTrack` would become two types: a core track (URL, cue window, cache and source keys) and the app's row model (tags, art, detected BPM and key, display names). Do that when someone needs the player without TagLib, not before.

**Artwork is the one place the engine touches UI types.** `AudioTrack` and the metadata code hand out `NSImage` or `UIImage`. That is tolerable, since AppKit and UIKit are always present. A UI-free engine would return a `CGImage` or the encoded bytes instead. Leave it until someone asks.

## PlaybackKitFX

**It starts with `AudioVarispeed`, the pitch fader's stage from [Vibe PR #190](https://github.com/cmicali/vibe/pull/190).** That is Vibe's own windowed-sinc converter. Its distortion and false tones sit at the float32 floor, near −150 dB, against −83 to −135 dB for Apple's Varispeed unit. It stays flat to 20 kHz. At zero pitch it is a bit-perfect pass-through, and a drag glides instead of stepping. It costs about 0.2% of one core at 48 kHz.

**It is already a library in all but location:**

- **Its API is plain C.** Create a stage, set the pitch from the queue, and render slices on the audio thread through an input callback. A caller needs no Vibe type.
- **It imports only system headers**: Foundation, CoreAudioTypes, Accelerate, and simd. Its one hidden dependency is Vibe's prefix header's realtime-check macros (`VIBE_REALTIME_CHECKED_BEGIN`, `VIBE_REALTIME_END`).
- **It has one caller in Vibe**: `AudioPlayer+Pipeline.m` hosts it. `AudioFX.m` and the pipeline also use the header's `VibeStereoBufferList`, a small stack-built buffer list. That type becomes PlaybackKitFX's shared buffer type, since every future stage needs one.
- **Its tests and benchmark stand alone.** `AudioVarispeedTests` drives the stage on its own, and the `pitch` benchmark group measures it beside Apple's unit. Both move with it. The player-level tests (`testPitchQuality`, the edge and position tests in `AudioPlayerRenderTests.m`) stay with the engine, since they render through the player.

**It is renamed on the way in.** The C symbols become `PBKVarispeed…` (`PBKVarispeedCreate`, `PBKVarispeedRender`), and the file becomes `PBKVarispeed.h`. `VibeStereoBufferList` becomes `PBKStereoBufferList`.

**The DJ chain joins later.** `AudioFX` is Vibe's master-bus segment: the low kill, the reverb send, and two BPM-synced delays, hosting ten of Apple's units. It needs `FadeMath.h` from the engine, which moves with it. It also follows the player's lifecycle rules: connect with the output stopped, and dispose only after the render leaves. Those rules must be written as its contract before another host can use it.

**What PlaybackKitFX grows into.** Any stage that runs in place on a render's buffers, under the same realtime discipline: no allocation, locking, logging, or messaging on the audio thread, checked by `-Wfunction-effects`. Each stage is a C API with a queue half and an audio-thread half, like the varispeed.

## How an app uses it

**A git submodule, plus one shared XcodeGen spec.**

- The app's repo has this repo as a submodule at `PlaybackKit/`, pinned to a commit.
- This repo has `PlaybackKit.yml` at its root. It defines one static framework target per product, built for macOS and iOS, with every build setting the engine needs.
- The app's `project.yml` adds `include: [PlaybackKit/PlaybackKit.yml]` and depends on the targets it uses. XcodeGen resolves the included file's paths relative to it.
- This repo also has its own `project.yml`. It includes the same `PlaybackKit.yml` and adds the test and benchmark targets, so CI builds and tests PlaybackKit on its own.

**Why this and not a Swift package, for now.** The engine's build carries things only XcodeGen expresses today:

- per-file flags: r8brain and PFFFT at `-O3` with NEON defines, and dr_* at `-O3 -w`
- a prefix header
- a verbose-logging define per configuration
- DEBUG-only seams, such as the manual render pump the tests drive
- warnings as errors, and C++17

A shared spec keeps all of it, unchanged, in one place. Both apps build exactly what Vibe builds today. A Swift package would force each of these to change before the first file moves.

**Settings the app provides.** `PlaybackKit.yml` reads one project setting, `PBK_VERBOSE_LOGGING` (0 or 1), from the app's project. Vibe maps its `VIBE_VERBOSE_LOGGING` to it. Debug and Release follow the app's configurations.

**Imports.** An app imports `<PlaybackKit/PBKAudioPlayer.h>` and so on, framework style. That is also what a later Swift package's module would look like, so no import changes when it comes.

**History.** `git filter-repo` carries each moved file's history from Vibe into this repo. This repo's first commit is this plan, so each filtered history arrives as one merge with `--allow-unrelated-histories`. `git log --follow` and blame still work.

## The prefix

**Every public class and C symbol takes `PBK`.** Objective-C has one namespace for classes, and C has one for functions. `AudioPlayer`, `AudioTrack`, `AudioFX`, and `AudioFileHandle` are generic names. Two classes with one name in one process is undefined behavior at run time.

- `AudioPlayer` → `PBKAudioPlayer`, `AudioTrack` → `PBKAudioTrack`, `AudioWaveformView` → `PBKWaveformView`, `AudioBPMAnalyzer` → `PBKBPMAnalyzer`.
- The engine's C symbols move from `Vibe` to `PBK`: `VibeVarispeedRender` → `PBKVarispeedRender`, `VibeMasterBusRender` → `PBKMasterBusRender`.
- Vibe's own names keep `Vibe`. After the move, `Vibe` in Vibe's code means app code, and `PBK` means library code.

**Rename as each product moves, not later.** The rename is mechanical, and each file is being touched anyway. Later, every app that has adopted the old names pays for it again.

## The work list

**Decoupling happens in place, inside the Vibe repo first.** Every item is checked by Vibe's existing gates before anything moves. Then each move is mechanical. The coupling was measured on 2026-10-08 at Vibe's `da438267`: about 78 imports leave `Vibe/Audio/` and `Vibe/WaveformUI/`. None go to `Vibe/Playlist/` or `Vibe/iOS/`. The playback core reads no settings; the app pushes them in.

1. **A prefix header per product.** 33 engine files use the log macros, `clampRange`, or `run_on_main_thread` without importing them. Each PlaybackKit target gets its own small prefix header with what it uses: `HelperMacros.h`, the log macros, the realtime macros, and the signposts. `VibeLog()` becomes `PBKLog()`, and its subsystem comes from the main bundle's identifier, so each app logs under its own name.
2. **No user-facing text in the engine.** 14 files use Vibe's `STR_*` macros, which read the app's string catalog. Every app would have to carry the engine's keys.
   - `AudioError` carries a code. The code-to-message table (`AudioErrorRules.h`) moves to the app.
   - The waveform style names move to the settings UI that lists them.
   - `AudioTrack.displayTitle` and `.displayArtist` move to the app.
   - The accessibility label and the `Formatters` calls in the waveform views become properties the app sets.
3. **No settings reads.** The playback core already reads none. The waveform views read `AppSettings` directly. They get the player's shape: the app pushes style, theme, density, bar width, centering, normalize, gain, and drag behavior. `+[WaveformTheme themeForAppTheme:isDark:]` moves to the app's theme code. `FolderArtResolver`'s default `-init` is deleted, since its caller already injects the setting and the folder access check.
4. **The loading road comes along.** `CloudFileMaterializer`, `DownloadProgressMonitor`, `NSURL+Hash`, and the placeholder functions of `NSURLUtil` join the engine. `NSURLUtil` splits, and its playlist-file parts stay in Vibe.
5. **Debug seams.** `VibeManualRenderPump`, `AudioLoadTiming`, and `VibeWorkTally` join the engine as `PBK…`. Each stays DEBUG-only where it is today. The debug command channel stays in each app.
6. **`Controls/LoadingIndicator`** joins PlaybackKitUI. The waveform views draw it. Vibe's rows import it from there.
7. **`MusicalKey.h`** joins PlaybackKitAnalysis. **`AudioWaveform`** becomes PlaybackKitWaveformData. **`FadeMath.h`** joins PlaybackKitFX with the DJ chain.
8. **`Audio/Mac/Convert/`** stays in Vibe. It is a Vibe feature, not engine.
9. **Tests move with the code.** Each product's tests move with it. The engine's render-pump tests and the fixture generator move with the engine. This repo's CI runs them on GitHub Actions, which is free for a public repo, macOS runners included. Vibe's `VibeTests` and `VibeAudioTests` stop listing about 85 and 40 engine sources and link the frameworks instead. `perf.py` and the component benchmarks move too, since they measure the engine.
10. **Docs.** The `AGENTS.md` files move with their directories. Engine-only cross-directory guarantees (the handle-open ceiling, the meter's place in the render) move to this repo's root `AGENTS.md`. Vibe's root doc keeps only the guarantees that span app and engine, each naming its PlaybackKit side. Vibe's `docs/audio-quality.md`, which measures the resampler, the decoders, and the varispeed, moves too.
11. **Layout checks.** Here: no file imports anything outside its product's declared dependencies. In Vibe: no file under `Vibe/` reaches into `PlaybackKit/` by relative path.

**What it removes from Vibe.** About 39,000 lines of engine and its tests. The two hand-kept source lists in the test targets. The engine's lookups into the app's string catalog. The settings reads inside the waveform views. The playlist code the tests compile only because `NSURLUtil` drags it in.

**What it adds.** This repo with its CI, `PlaybackKit.yml`, one prefix header per product, and a submodule in each app. No new types, except the `AudioTrack` split if it is ever done.

**What does not change.** The engine's behavior and its build settings. Every Vibe check passes before and after. `perf.py compare` shows no change.

## Costs

- **Since 2026-04-01, 232 commits changed Vibe's engine, and 167 of them (72%) also changed Vibe's app code.** After the move, each of those is a commit here, then a submodule bump in the app. One editing session still covers both, through the submodule.
- **Since then, 90 of those commits (39%) changed more than one product.** That is why this is one repo, not one per product: with one repo per product, each of those would span several repos and need releases in dependency order.
- **Submodules and worktrees.** Vibe's workflow leans on worktrees, and each worktree needs `git submodule update --init`. Vibe's `make project` should run it, so no one has to remember.
- **Agent docs split across two repos.** An agent in Vibe must read this repo's `AGENTS.md` before changing the engine. Vibe's root doc says so, with the submodule path.
- **The stress, perf, and audio-quality tooling** lives here, with the engine. Vibe keeps its app-level stress runs.

## Phases

### Phase 0: setup

- Choose the license.
- This plan as the first commit. Done.

### Phase 1: the pilot, PlaybackKitFX with the varispeed

The smallest real piece goes first. It proves every step the big move will take:

- `git filter-repo` of the varispeed, its tests, and its benchmark from Vibe, merged in
- the `PBK` rename, for `PBKVarispeed`
- `PlaybackKit.yml` with the PlaybackKitFX target, and this repo's own `project.yml`, Makefile, and CI
- PlaybackKitFX's prefix header, with the realtime-check macros
- Vibe consuming it: the submodule, the `include:`, `make project` running the submodule update, worktrees, Vibe's CI, and `perf.py`

Done when Vibe's checks pass with the varispeed coming from here, the `pitch` benchmark shows no change, and this repo's CI runs `AudioVarispeedTests`.

### Phase 2: PlaybackKitAnalysis

The analyzers and `MusicalKey.h`, the next piece with no dependencies. Done when Vibe's tempo and key results are unchanged (`perf.py --analyze` proves it exact).

### Phase 3: decouple the rest, in Vibe

Work list items 1 to 8, inside the Vibe repo, under Vibe's gates. Done when every check passes and `perf.py compare main` shows no regression.

### Phase 4: PlaybackKit, PlaybackKitWaveformData, and PlaybackKitUI

The engine, the waveform data, and the UI move here, renamed with `PBK`, with their tests, tooling, and docs. Done when Vibe's checks pass against this repo and this repo's CI passes on its own.

### Phase 5: the DJ chain joins PlaybackKitFX

`AudioFX` and `FadeMath.h`, with the hosting rules written as its contract.

### Later: the HTTP road

Streaming over HTTP into the engine, planned in Vibe ([share links](https://github.com/cmicali/vibe/blob/main/docs/future/share-links.md) Phase 2 and [streaming from any source](https://github.com/cmicali/vibe/blob/main/docs/future/streaming-any-source.md) Phase 1). Once the engine lives here, that work lands here.

## Later: a Swift package, for outside users

When people outside these apps want PlaybackKit, a `Package.swift` makes it addable by URL in Xcode. That is how AudioKit is consumed. The apps can keep the XcodeGen spec. What the package needs first:

- **No unsafe flags.** SwiftPM refuses `unsafeFlags` in any package fetched by version, so `-O3` for r8brain and PFFFT goes. Measure their cost at the package's release default first, with `perf.py`. If it matters, the apps keep building from `PlaybackKit.yml`, and the package serves everyone else.
- **No prefix headers.** Each file imports what it uses.
- **The verbose-logging define** becomes a package trait (SwiftPM 6.1), or a runtime switch.
- **Swift users.** Nullability on every public header. No C++ in a public header: `AudioWaveform.h` declares a C++ class, so its codable form becomes the public face.
- **A public API is a promise.** Tag `0.x` until the API settles.
