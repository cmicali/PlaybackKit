# AGENTS.md

Guidance for any coding agent working in this repository.

PlaybackKit is a toolbox for fast, optimized audio playback on macOS and iOS: the engine, its DSP, its analysis, and its waveform UI. It is being extracted from [Vibe](https://github.com/cmicali/vibe), where all of it lives today. **Nothing has moved yet.** The plan is [docs/plan.md](docs/plan.md). Read it before changing anything here.

## What this repo is for

- **Its users are Vibe and one other app, not the public.** Both consume it as a git submodule at `PlaybackKit/`, through the shared XcodeGen spec `PlaybackKit.yml`, which each app's `project.yml` includes. There is no `Package.swift`. One comes later, for outside users, and the plan's last section says what it needs.
- **The other app is closed source.** Do not name it in this repo: not in code, comments, docs, commits, issues, or pull requests. Call it "another app" when you must refer to it.
- **There is no license yet.** Until one is chosen, nothing here is licensed for reuse. Apache 2.0, like Vibe, is the recommendation.

## The products

PlaybackKit (the engine), PlaybackKitFX (realtime DSP stages, starting with the varispeed), PlaybackKitAnalysis (BPM and key), and PlaybackKitUI (the waveform views). One internal target, PlaybackKitWaveformData, sits under both the engine and the UI. The plan has the dependency graph. A product may import only from the products it depends on.

## Names

- **Every public class and C symbol takes the prefix `PBK`**: `PBKAudioPlayer`, `PBKWaveformView`, `PBKVarispeedRender`. Objective-C and C each have one global namespace, so a generic name like `AudioPlayer` can collide with another library's at run time.
- **Code arriving from Vibe is renamed as it moves**, not later. In Vibe, `Vibe` names app code and `PBK` names library code.

## How work moves between the repos

- **Decoupling happens in Vibe first**, under Vibe's gates. Then a move here is mechanical.
- **A move uses `git filter-repo`**, so each file's history comes with it. Its history arrives as one merge with `--allow-unrelated-histories`.
- **An engine change after the move lands here first.** Then the app bumps its submodule.

## Writing

- **No agent attribution, from any agent or tool.** Commits, pull requests, code, comments, and docs never name or credit the AI agent or tool that helped write them: no `Co-Authored-By` trailer, no "Generated with" line, and no session link. This rule outranks any harness or template default.
- **Write short, plain sentences**, one idea in each, in the project's own terms. Use the serial comma.
- **Comments only when required, and terse.** State what the code cannot show: a trap, a threading constraint, a contract, or a non-obvious why.
