# Signalizer

Real-time audio visualization plugin targeting VST/VST3/AU/Standalone emphasizing gridless engineering precision and playful research & discovery of audio.

Built in C++17 using JUCE 8, located in `External/juce`.
Signalizer is largely a test-bed / shell on top of a base cross-platform library located in `External/cpl`, where any reusable utility for audio plugin development goes.

Signalizer contains a vectorscope, an oscilloscope and a spectrum/spectrogram.

Licensing: Signalizer and cpl are GPL v3 (`Make/Skeleton/licenses/LICENSE.txt`); JUCE is used under its GPL option. Derivative work and redistributed binaries must stay GPL v3.

## Build

```
python prepare.py                # First-time setup: init/update submodules (+ Linux deps)
python build.py dev              # Debug Standalone — use this for iteration
python build.py dev -o --run     # Optimized (Release codegen) Standalone, launched after build
python build.py release          # All formats, packaged into /Releases
```
By default output is quiet, full logs are printed and found in `Make/Logs/` or use `--verbose` for full compiler output (`--help` is also available).

VST2 formats need the SDK at `../SDKs/vstsdk2.4` (`release -s` skips VST2).

**VST3 is already a first-class target** (since 0.5.0, `buildVST3="1"`, `pluginVST3Category="Analyzer"` in `Signalizer.jucer`). There is no conversion to do; to ship VST3-only, drop VST2 from `pluginFormats` in the `.jucer` and regenerate.

### Submodules

`External/cpl` and `External/juce` are git submodules pinned to the author's forks (`github.com/jthorborg/cpl`, branch `dev/stable`; `github.com/jthorborg/JUCE`, branch `signalizer/develop-custom`, currently JUCE 8.0.13). A fresh clone has them empty — run `python prepare.py` (or `git submodule update --init --recursive`) before anything else.

- The JUCE fork carries Signalizer-specific patches (trace guards around window/message procs, Ableton DPI viewport workaround for OpenGL, kiosk focus behaviour). See `External/cpl/Juce_Changes.txt` and `git log` in `External/juce`. **Do not swap in stock JUCE**; upgrading means merging upstream into the fork.
- Edits to cpl/JUCE are commits in those submodules, followed by a submodule pointer bump in this repo. Prefer putting reusable code in cpl, Signalizer-specific code in `Source/`.

### Projucer / generated files

`Signalizer.jucer` is the source of truth for project settings, file lists, exporters and JUCE options. `Builds/` and `JuceLibraryCode/` are regenerated from it — never hand-edit them except as a deliberate stopgap mirrored in the `.jucer`. Use the Projucer built from the fork so exporter output matches:

```
# once: build Projucer (VS2022 developer environment)
msbuild External/juce/extras/Projucer/Builds/VisualStudio2022/Projucer.sln /p:Configuration=Release /p:Platform=x64
# after editing Signalizer.jucer
External/juce/extras/Projucer/Builds/VisualStudio2022/x64/Release/App/Projucer.exe --resave Signalizer.jucer
```

**A raw resave clobbers hand-maintained generated files.** After resaving, revert anything you didn't intend to change — in particular:

- `JuceLibraryCode/JucePluginDefines.h` — `JucePlugin_Version*` are hand-pointed at `SIGNALIZER_VERSION`, `SIGNALIZER_VST_VERSION_HEX`, `SIGNALIZER_VERSION_STRING` (from `Source/version.h`); Projucer writes literals.
- `Builds/MacOSX/Info-*.plist`, `Builds/VisualStudio2022/resources.rc` — rewritten by the release scripts; Projucer reformats them.
- Line endings: some files are LF, some CRLF; Projucer writes CRLF. Match the original (`git diff --ignore-cr-at-eol` shows the real change).

Review `git diff` after every resave; only the exporter output you meant to change should remain.

### Windows build notes

- `Make/build_win.py` pins `vswhere` to VS2022 (`-version [17.0,18.0)`) because the projects use the **v143 toolset**; a newer side-by-side Visual Studio (e.g. VS 2026) without the v143 component would otherwise be picked and fail with MSB8020.
- Plugin binary copy steps are disabled (`enablePluginBinaryCopyStep="0"`) — JUCE's VST3 copy step targets `C:\Program Files\Common Files\VST3` and fails without admin. The post-build step copies `Make/Skeleton/*` into each target's `$(OutDir)` (uses Git's `cp`, which must be on PATH).
- The Skeleton is **not** copied inside the VST3 bundle (`Signalizer.vst3/Contents/x86_64-win/`) by dev builds; the release script does that. For DAW testing of a dev VST3, copy it in manually.
- On Windows, `build.py release` builds the whole `Signalizer.sln`, including the VST2 project — `-s/--skipvst2` is only honoured on macOS. Without the VST2 SDK, release builds fail on Windows until VST2 is removed from the `.jucer` or the script builds per-project.
- `build.py dev` builds only `Signalizer_SharedCode` + `Signalizer_StandalonePlugin`; output is `Builds/VisualStudio2022/x64/<Debug|Release>/Standalone Plugin/Signalizer.exe`.
- To build the VST3 alone, build the projects directly (solution-level `/t:` targets don't work here): `Signalizer_VST3ManifestHelper.vcxproj` (`/p:Platform=Win32`), then `Signalizer_SharedCode.vcxproj` and `Signalizer_VST3.vcxproj` (`/p:Platform=x64`), each with `/p:SolutionDir=<abs path to Builds\VisualStudio2022\>`.
- Plugin identity (`pluginCode="Sign"`, `pluginManufacturerCode="LbJt"`, company "Lightbridge") is still upstream's, so a fork's VST3 has the same class ID as official Signalizer. Hosts treat them as the same plugin; changing the codes makes it a distinct plugin but old sessions won't find it.

### Other build notes

- `fastMath="1"` is on for every config. NaN/Inf checks can be optimized away and denormal behaviour is unspecified — sanitize DSP inputs explicitly rather than relying on `std::isnan`.
- Linux builds require GCC >= 12 and target `-march=ivybridge` (AVX required at runtime).
- `CPL_PROFILING=1` and `CPL_TRACEGUARD_ENTRYPOINTS` are defined on all exporters.
- `python build.py release -j/-n/-b` bumps major/minor/build in `Make/config.ini` and rewrites `Source/version.h`.

## Structure

- `Source/` — plugin source (see layout below)
- `Builds/` — IDE project files (per platform, generated)
- `Make/` — build scripts (`common.py`, `build_win/osx/linux.py`), `config.ini` (version + product info), `Skeleton/` (presets, icons, licenses shipped next to every binary)
- `External/` — submodules (`juce`, `cpl`)
- `JuceLibraryCode/` — generated JUCE glue (do not edit)
- `Signalizer.jucer` — Projucer project file; regenerates `Builds/` and `JuceLibraryCode/`
- `Source/Notes/Bugs.txt` — author's known-bug notes
- `CHANGELOG.md` — Keep a Changelog format; add entries for user-visible changes

### `Source/` layout

- `Common/` — shared infra: `HostGraph`, `MixGraphListener`, `SentientViewState`, `SharedBehaviour` (global view options), `ConcurrentConfig` (sample rate/history/bpm atomics), `SignalizerDesign` (editor layout building blocks), and the MVC base classes in `CommonSignalizer.h` (`ProcessorState`, `ProcessorStreamState`, `SystemView`, `StateEditor`)
- `Processor/` — `AudioProcessor` (the JUCE `AudioProcessor`, in `PluginProcessor.cpp`)
- `Editor/` — `MainEditor`, `GraphEditor`, `ProfilerWindow`
- `Config/` — `SignalizerConfiguration.h`
- `Oscilloscope/`, `Spectrum/`, `Vectorscope/` — one folder per visualization
- `Unity/SignalizerSource.cpp` — unity-include build for embedding the visualizations elsewhere

Each visualization folder follows the same five-way split (mapping onto the MVC description below):

- `*Parameters.h` — the `…Content` persistent state class
- `*.cpp` / `*.h` — the visualization class
- `*Controller.cpp` — the editor UI
- `*Rendering.cpp` — OpenGL / 2D paint
- `*DSP.cpp` or `*DSP.inl` — signal processing

### Where things live in cpl

- `infrastructure/parameters/` — `ParameterGroup`/`ParameterSystem.h` (registration, handles, RT listeners, version cohorts), `JuceAudioParameterBridge.h` (exposes groups to the host)
- `gui/` — `DesignBase.{h,cpp}` (colour scheme `ColourEntry`, `CLookAndFeel_CPL`, font), `controls/` (`CValueKnobSlider`, `CButton`, `CValueComboBox`, `CColourControl`, …), `widgets/` (`CPresetWidget`, `CSignalGeneratorWidget`, `CDSPWindowWidget`, `CTransformWidget`)
- `AudioStream.{h,inl,cpp}` — lock-free audio stream with async listener threads
- `simd/` — `dynamic_isa_dispatch` and ISA traits
- `dsp/`, `ffts/` — windows, transforms, resonators, signal generator, FFT backends
- `rendering/` — OpenGL helpers (`COpenGLStack`, display setup)
- `Protected.{h,cpp}`, `Exceptions.cpp` — trace guards, runtime assertions, crash handling

## Signal Flow and Architecture

1. `PluginProcessor` receives the signal from device or DAW
2. The signal is sent into one of two `cpl::AudioStream`s (`realtimeInput`)
3. The `HostGraph` connects to the `realtimeOutput` of the stream, publishing the data (timestamped) to later be available to any other Signalizer (including itself)
4. The `HostGraph` discovers and remembers other Signalizers (persistent IDs) and receives mapped input data (configured in `GraphEditor`) through `MixGraphListener` to the visualization audio stream (`presentationInput`), enabling sidechain/mixing of any Signalizer's output into another for comparison and overlay
5. Each visualization class is split into something similar to "model-view-controller", eg.
	- The persistent State (built from simple parameters exposed to the DAW): `OscilloscopeContent`
	- The Visualization: `Oscilloscope`
	- The Editor UI: `OscilloscopeController`
6. MainEditor orchestrates the tabbing system of editor UIs, global settings UI, and view instantiation by forwarding `presentationOutput` through `SentientViewState`, which defers view instantiation and manages serialization on creation/destruction.
7. Each Visualization has a `ProcessorShell` (an `AudioStream::Listener`) that processes audio asynchronously into `StreamState`, held as a separate `shared_ptr` that the `AudioStream` safely releases to avoid deadlocks during plugin instance destruction.
8. In an OpenGL thread, each Visualization's `onOpenGLRendering` ingests parameter updates via `handleFlagUpdates()`, renders audio data from the stream/`StreamState`, and paints 2D grids and text overlay.
9. Each Visualization typically does runtime CPU feature detection to dispatch to various template instantiations taking advantage of SIMD/FMA etc. for graphics and DSP.

## Threading rules

| Thread | Entry point | Rules |
| --- | --- | --- |
| Audio | `AudioProcessor::processBlock` | Lock-free only. No allocation, no `streamState` lock, no mutexes. Hard-capped at 2 channels (`supportedChannels`). Early-returns if channel count / block size differs from `prepareToPlay`. |
| Host param | host `setValue` → `updateFromHostNormalized` → RT listeners | Can be the audio thread. `parameterChangedRT()` handlers must only set atomic flags (`cpl::ABoolFlag`), never do work. |
| AudioStream async | `ProcessorShell::onStreamAudio`, `MixGraphListener` | DSP runs here, holding `streamState` (a `CriticalSection<T, std::mutex>`) — this blocks the GL frame, keep it short. Spectrum fans out further via `cpl::jobs::parallel_for`. |
| OpenGL | `onOpenGLRendering` → `handleFlagUpdates()` | Takes `streamState`, snapshots parameters into the view's non-atomic `StateOptions`. Buffer resizes / reconfig belong here, under the lock. |
| Message | editor, `HostGraph` model, (de)serialization | Owns UI components and `getWidth()/getHeight()`. Nothing in `Source/` uses `MessageManagerLock`; don't start. |

- `HostGraph` uses a **process-global** static mutex and instance set shared by every Signalizer instance in the host.
- `NONTERMINAL_ASSUMPTION(cond)` logs once per site and surfaces a message box later from `MainEditor::timerCallback` — use it for recoverable invariants. `CPL_RUNTIME_ASSERTION` is fatal: it ends in `CrashIfUserDoesntDebug`, which shows a **modal** message box from whatever thread it fired on (deadlock risk if the message thread is waiting on that thread). Prefer non-terminal assumptions off the message thread.
- Exceptions are logged to the exception log (button in global settings reveals it).

### SIMD

New DSP/rendering kernels follow the existing pattern: a `struct XDispatcher { template<typename ISA> static void dispatch(...); }` invoked through `cpl::simd::dynamic_isa_dispatch<float, XDispatcher>(...)`, written against `typename ISA::V` and `ISA` helpers (see `OscilloscopeDSP.inl`). No raw intrinsics outside the template. Verify scalar, 4-wide and 8-wide paths.

## Parameters, IDs and state compatibility

**Read this before touching any `*Parameters.h`.** Saved DAW sessions, automation and presets depend on it.

- Each view's `…Content` owns a `ParameterSet` (cpl `ParameterGroup`). Parameters are registered in the constructor via `registerSingleParameter(x.generateUpdateRegistrator())` / `registerParameterBundle(...)`, then `seal()` (registering after seal throws).
- Groups are created in `ContentCreationList` order (`Source/Editor/MainEditor.cpp`: Vectorscope, Oscilloscope, Spectrum). Each group's offset is the running parameter count, so a parameter's host index = group offset + registration position.
- `JUCE_FORCE_LEGACY_PARAMETER_AUTOMATION_TYPE=1` and `JUCE_FORCE_USE_LEGACY_PARAM_IDS=1` mean **host-visible IDs are these flat indices** (VST2, VST3 and AU). Consequently:
	- Never reorder, remove or insert parameters mid-group. Never reorder `ContentCreationList`.
	- Appending a parameter to Vectorscope or Oscilloscope shifts every index in the groups after it — that breaks saved automation for those groups. Only appends to the **last** group are index-safe. Weigh this before adding parameters to earlier views; a new view must be appended at the end of `ContentCreationList`.
	- `ParameterGroup::setVersionCohort(N)` exists for AU index stability but is currently unused (everything is cohort 1). New parameters should get a new, never-reused cohort literal.
	- Do not flip the legacy flags without a migration plan; doing so changes every ID.
- Serialization is an order-dependent binary stream (`archive << param` / `builder >> param`), stored by **normalized** value. Changing a parameter's range silently remaps old saved values.
	- Writes are unconditional; reads of fields added later are gated: `if (version >= cpl::Version(a, b, c)) builder >> newParam;` using the version being shipped. Always append, always gate.
	- Controllers also serialize editor settings (`serializeEditorSettings` / `deserializeEditorSettings`) with the same append-and-gate rule.
	- Host state: `AudioProcessor::getStateInformation` / `setStateInformation` → `serialize`/`deserialize` writing `"Editor"`, `"Parameters"` (one block per set name), `"Engine"`, signal generator, plus `"host-graph"`.
- Factory presets are `Make/Skeleton/presets/*.sgn` (`<name>.<view>.sgn`, `default.main.sgn` loaded at construction). Old presets lacking a new field keep its default via the version gate.

### Recipe: add a parameter to an existing view

1. `*Parameters.h`: declare the `cpl::ParameterValue<ParameterSet::ParameterView>` (or `ChoiceParameter`) member and initialize it in the ctor list.
2. Register it **after** all existing registrations, before `seal()` (set a new version cohort first).
3. Append to `serialize`; append a version-gated read in `deserialize`.
4. `*Controller.cpp`: add a control member bound to the parameter, set title/description in `initControls()`, place it with `section->addControl(&k, row)` in `initUI()`, append (gated) to editor-settings serialization.
5. View: copy into `StateOptions` in `handleFlagUpdates()`; for expensive reactions set a flag in `parameterChangedRT()` and consume it in `handleFlagUpdates()`.
6. Use it from `*Rendering.cpp` / `*DSP`.
7. Add a CHANGELOG entry.

### Recipe: add a new visualization

1. `Source/Foo/` with the five-way split, modelled on Vectorscope (`FooContent : ProcessorStreamState<FooContent>`, `enable_shared_from_this`; view derives `GraphicsWindow` + `ParameterSet::RTListener`; controller derives `StateEditor`/`CContentPage`).
2. Append `{ FooContent::name, &FooContent::create }` to the **end** of `ContentCreationList` and include the header in `MainEditor.cpp`. `PluginProcessor` and the tab system iterate that list.
3. Add the `.cpp` files to `Source/Unity/SignalizerSource.cpp`.
4. Add a `<GROUP>` with `<FILE>` entries to `Signalizer.jucer` and regenerate `Builds/`.
5. Optional: `*.foo.sgn` presets in `Make/Skeleton/presets`.

## UI

- `MainEditor` owns a single `juce::OpenGLContext`; the active view attaches to it. Default size 700x480. Top bar: wrench (`ksettings`), view tabs, then freeze / help / kiosk / graph buttons. Editor panel sits under the tabs; view fills the rest.
- Editor panels are a LIFO `editorStack` of `StateEditor`s. Global settings (`MainEditor::createEditor`) are a `CContentPage` with Settings / Colours / Presets pages built from `MatrixSection`s.
- Layout primitives in `Common/SignalizerDesign.h`: `elementSize = 26`, `elementBorder = 1`, `CContentPage::addPage(name, svgIcon)`, `COrderedTabPage::addSection`, `MatrixSection::addControl(ctrl, row)` (auto-sizing rows).
- UI chrome colours/fonts/look-and-feel live in **cpl** (`gui/DesignBase.{h,cpp}`: `ColourEntry`, `defaultColours`, `CLookAndFeel_CPL` singleton; optionally loads `<binary dir>/resources/fonts/Verdana.ttf`, which the Skeleton does not currently ship). The look-and-feel is a process-wide singleton: changing colours affects every instance in the host.
- Visualization colours are per-view **parameters** (`cpl::ParameterColourValue` in each `*Parameters.h`), snapshotted in `handleFlagUpdates`. Data + grid are raw GL (`COpenGLStack`); text, legend and labels go through `renderGraphics(...)` → `paint2DGraphics` (JUCE 2D rasterized in the GL pass). Software engine path: `onGraphicsRendering`.
- Icons are SVGs under `Make/Skeleton/resources/icons/svg/`. Tooltips come from `bSetDescription`.
- To add a global setting: member in `MainEditor.h`, wire in `initUI` (`bAddChangeListener`, title, description), place in `createEditor`, handle in `valueChanged`, append (gated) to `MainEditor::serialize`/`deserialize`; if views need it, add a relaxed atomic to `SharedBehaviour`.

### UI hazards

- `views` (vector of `SentientViewState`) must never reallocate — moving a view state corrupts it (`views.reserve` guards this).
- Kiosk/fullscreen enter/exit and size restore are deferred ~10 ms via `FutureMainEvent` to dodge Ableton Live DPI/resize bugs; keep that when touching it. Windows DPI is handled via `ScopedThreadDPIAwarenessSetter` in kiosk and the fork's Ableton OpenGL viewport patch; views use `getRenderingScale()`.
- Antialiasing levels are a hard-coded list (not queried from GL); changing AA detaches/reattaches the context.
- `MainEditor::valueChanged` is one long if-chain; `resized()` skips layout while the resize corner is dragged.
- `GraphEditor` and `ProfilerWindow` are `DocumentWindow`s that `delete this` when closed or when `mainEditorDied()` is called; respect that lifetime when adding windows.
- **Any new or changed user-facing surface: propose the interaction design and get sign-off before building.**

## Known bugs and risk hotspots

Start here when fixing bugs (see also `Source/Notes/Bugs.txt`):

1. `HostGraph::getNumChannels()` returns a hard-coded 2 → mono instances advertise 2 channels, mix graph connects out-of-range channel indices (Bugs.txt #2). Only a `NONTERMINAL_ASSUMPTION` in `MixGraphListener::onStreamAudio` skips the bad index.
2. `MixGraphListener::updateTopologyCommands`: `CPL_RUNTIME_ASSERTION(command.stream.get() != self->source.lock().get())` has fired in the wild (self-connect, e.g. alias serialized then reloaded, Bugs.txt #1). Fatal assertion on an async thread → modal box → possible deadlock while the editor tears down the mix graph (Bugs.txt #3).
3. `CrashIfUserDoesntDebug` (cpl `Exceptions.cpp`) shows a modal box from any thread.
4. 2-channel hard cap in `PluginProcessor` (`supportedChannels`); surround input is truncated, and a changed input count silently drops audio until `prepareToPlay`.
5. Spectrum: `axisPoints == 0` (zero-size/minimized view) trips an assertion in `TransformConstant::remapFrequencies`; `resume()` writes `windowSize` off-thread; `setStorage`/`audioMemoryResize` vs. the separate audio-buffer lock; `StateOptions` plain bools shared across threads.
6. `MixGraphListener`: `containedSamples = 0` on connect stalls every channel from that source.
7. `CommonSignalizer.h`: lifetime TODO in `AudioHistoryTransformatter` ("can leave scope before us").
8. `GraphicsWindow` mouse handlers in `CommonSignalizer.h` have `beginChangeGesture`/`endChangeGesture` TODOs — in-view mouse interaction (zoom/drag) is not wrapped as host change gestures.

## Testing

There is no automated test suite for Signalizer (cpl has `CPLTests.cpp`). Verify changes by:

- `python build.py dev --run` and exercising the changed view in the Standalone (the built-in signal generator under the wrench is useful input).
- Loading the VST3 in a DAW for host-facing changes (automation, state save/restore, multiple instances + GraphEditor sidechaining). Test that a project saved with the previous version still restores correctly.
- The profiler window (wrench → profiler, `CPL_PROFILING`) for performance work.

## Notes

- `CLAUDE.md` just includes this file; keep project guidance here.
- Comment density and style: follow the surrounding code (tabs for indentation, `camelCase` members, `k`-prefixed controls, `C`-prefixed cpl classes).
