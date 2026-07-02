# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JJazzLab is a cross-platform desktop application that generates MIDI backing tracks: the user types chord symbols, selects a rhythm (music style), and the app generates a full backing track (drums, bass, piano, etc.). It is a Java application built on the **Apache NetBeans Platform** (NetBeans modules, not JPMS modules) using **Maven** (`nbm-maven-plugin`). License: LGPL v2.1.

## Build & Run

Requires JDK 23+ (compiler release is 25) and Maven 3. `mvn` is on the PATH on this machine.

```bash
# Full build (from repo root)
mvn clean install

# Run the application (after install)
cd app/application && mvn nbm:cluster-app nbm:run-platform

# Build a single module (e.g. model/Harmony)
mvn install -pl model/Harmony

# Run all tests / a single test class / a single test method
mvn test
mvn test -pl model/Harmony -Dtest=ChordTypeTest
mvn test -pl model/Harmony -Dtest=ChordTypeTest#someMethod

# Skip tests
mvn install -Djjazzlab.surefire.skipTests=true
```

One-time preparation: a soundfont file too big for git must be downloaded from https://archive.org/download/jjazz-lab-sound-font/JJazzLab-SoundFont.sf2 and placed in `plugins/FluidSynthEmbeddedSynth/src/main/soundfont`. On Linux/macOS the native `fluidsynth` package must also be installed.

Only a few modules have tests (`model/Harmony`, `model/Phrase`, `model/SongImpl`, `core/Utilities`).

## Architecture

The parent POM (`pom.xml` at root) lists all modules, grouped into four directories:

- **`model/`** — core data models: `Harmony` (ChordSymbol, Degree, TimeSignature...), `Midi`, `Phrase` (NoteEvent sequences), `Rhythm`, `Song`/`SongImpl`.
- **`core/`** — services with no or minimal UI: music generation (`RhythmMusicGeneration`), playback (`MusicControl`), `RhythmDatabase`, `OutputSynth`, `UndoManager`, utilities, repackaged Guava/XStream.
- **`app/`** — the NetBeans application (`app/application` is the app assembly, `app/branding` the branding), plus UI modules: editors (`CL_Editor`/`CL_EditorImpl` = chord leadsheet, `SS_Editor`/`SS_EditorImpl` = song structure, `PianoRoll`, `MixConsole`), actions, dialogs, options.
- **`plugins/`** — rhythm generation engines (`YamJJazz`) and FluidSynth integration (`FluidSynthEmbeddedSynth`).

### Key data model

- **`Song`** = `ChordLeadSheet` (sections + chord symbols; model of the CL editor) + `SongStructure` (list of `SongPart`s, each linked to a parent section and holding `RhythmParameter` values; model of the SS editor). Combining the two yields the "unrolled" lead sheet from which music is generated.
- **`MidiMix`** stores a song's MIDI configuration and maps each `RhythmVoice` (track: drums, bass, ...) to a MIDI channel.

### Music generation pipeline

Framework → `RhythmProvider` (SPI, registered via `@ServiceProvider`, discovered by `RhythmDatabase` at startup) → `Rhythm` (defines `RhythmVoice`s and `RhythmParameter`s) → `MusicGenerator` (receives a song context, returns one `Phrase` per instrument) → framework post-processing (mute, velocity shift, fade-out, tempo factor). Engines don't handle real-time scheduling. Simple reference implementations: `core/RhythmStubs`, and `DummyGenerator` in `core/RhythmMusicGeneration`.

### NetBeans Platform conventions

- **api/spi package split**: each module only exposes packages named `**/api/**` or `**/spi/**` (declared as `publicPackages` in each module's pom). Everything else is module-private — the build fails (`verifyRuntime=fail`) if a module uses a non-public package of another module without a proper Maven dependency.
- **Lookup**: services are found via the NetBeans global Lookup (`@ServiceProvider` annotation / `Lookup.getDefault()`); selection-driven UI actions use context Lookup (e.g. `CL_ContextActionSupport`).
- **Actions** are declared with `@ActionID` / `@ActionRegistration` / `@ActionReference` annotations; display names starting with `#` resolve to `Bundle.properties` keys. Localized bundles (`Bundle_xx_XX.properties`, managed via Crowdin) sit alongside; use `ResUtil.getString(...)` from Java code.
- **Undo**: model edits must be wrapped in `JJazzUndoManagerFinder.getDefault().get(model).startCEdit(name)` / `endCEdit(name)`.
- UI forms are NetBeans-generated `.form` files with matching `.java` — avoid hand-editing the generated code sections.

## Code Style (from .github/copilot-instructions.md)

- Favor immutability: `final` classes/fields where possible, `List.of()`/`Map.of()`, `Stream.toList()`.
- Use Java Records for data-carrier classes instead of traditional classes.
- Use pattern matching for `instanceof` and `switch` expressions.
- Use `var` for locals only when the type is clear from the right-hand side.
- Javadoc required except for trivial self-explanatory methods; keep it simple, limit HTML tags to `<p>` and `<br>`.
- Preconditions: `Objects.requireNonNull(var)`; Guava `Preconditions` for other checks.
