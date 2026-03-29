# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm start          # Dev server (port 3005)
pnpm build          # Production build (CI=false to suppress warnings-as-errors)
pnpm test           # Run tests (react-scripts / Jest)
pnpm test -- --testPathPattern=<file>  # Run a single test file
```

ESLint runs automatically via `react-scripts` — there is no standalone lint command.

## What this app is

Genscapes is a browser-based drone music generator. Users build tracks, each consisting of a **PolySynth source** chained through optional **audio effects** (AutoFilter, Reverb, Delay, Tremolo), then into a master output. A Tone.js `Pattern` drives each track, triggering random notes from a chosen musical scale at randomised intervals. Compositions are shareable via URL (state serialised to the `p` query param).

## Architecture

### Dual Redux state slices

State is split into two parallel slices that mirror each other per track:

- **`params`** (`src/reducers/params.ts`) — user configuration: synth options, effect settings, scale, intervals, randomisation amounts
- **`audio`** (`src/reducers/audio.ts`) — live Tone.js nodes: `PolySynth`, `AutoFilter`, `Pattern`, etc.

When a user changes a param, a component dispatches both a `params` action (to persist the setting) and an `audio` action (to update the live Tone.js node). These two slices must be kept in sync.

### Signal chain model

Each track's state is an ordered `signalChain[]` array of modules. Index 0 is always the source (`polySynth`); subsequent entries are effects. The `audio` slice holds the same-indexed array of Tone.js nodes. Adding/removing an effect updates both slices and re-chains the nodes.

### Module mapping pattern

`src/lib/constants.ts` contains four parallel maps keyed by module name (`'autoFilter'`, `'reverb'`, etc.):
- `mapEffectNameToToneComponent()` — Tone.js class
- `mapEffectNameToUiComponent()` — full settings UI component
- `mapEffectNameToUiMinComponent()` — compact inline UI component
- `mapEffectNameToInitialState()` — default params

Adding a new effect type requires an entry in all four maps plus the corresponding types in `src/types/params.ts`.

### Selectors

All selectors in `src/selectors/index.ts` are **factory functions** (`makeSelect*`) that return a new memoised selector per call. Components create them with `useMemo(() => makeSelectX(trackId), [trackId])` to avoid creating fresh selector instances on every render.

### Param change callbacks

UI components receive two callbacks:
- `onTrackParamChange: UpdateTrackParamHelper` — for composition/sequencer params (scale, notes, intervals)
- `onModuleParamChange: UpdateModuleParamHelper` — for synth/effect params

Both follow the lodash `_.set` path convention: `onModuleParamChange('options.frequency', 440)`.

### Audio helpers

`src/helpers/tone.ts` contains all Tone.js wiring logic:
- `createPolySynth` — builds synth + LFO tremolo
- `setPattern` — creates a `Tone.Pattern` with the randomised note-triggering callback
- `updateAudioState` — hydrates a full audio state from a params snapshot (used on URL load)
- `triggerPatternNote` — the pattern callback; calculates randomised pitch, detune, and duration per note

## Architecture Review

See [`docs/architecture-review.md`](docs/architecture-review.md) for the full v2 refactoring plan — prioritised architectural issues and suggested execution order.

## Key type locations

| Concern | File |
|---------|------|
| All param state shapes | `src/types/params.ts` |
| All audio node state shapes | `src/types/audio.ts` |
| `ModuleName`, `ModuleType` unions | `src/types/params.ts` |
| `RootState`, `AppDispatch` | `src/store.ts` |
| Typed hooks | `src/hooks.ts` |
