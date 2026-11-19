# Neon Swarm

A browser-native survivors-like built to be **developed and live-ops'd by an AI agent team**.

Move-only controls, auto-firing weapons, XP-gem economy, draft-3 level-ups, timed waves.
The differentiator is not the genre — it is the engineering harness around it:

- **Deterministic simulation core** — pure TypeScript, fixed 60 Hz logical ticks,
  fixed-point math, seeded `mulberry32` RNG. Same seed + same input tape → same outcome,
  on every engine.
- **Checkpoint hash-chain replay verification** — canonical state serialization hashed
  every second of sim time; a golden transcript committed to the repo is the acceptance
  gate for every PR. See [docs/DESIGN.md](docs/DESIGN.md).
- **Bot-driven balance live-ops** — a headless batch runner plays thousands of seeded
  games, emits `balance_report.json`, and the agent files data-only PRs (JSON stats in
  `data/**`) with before/after win-rate charts attached.

Status: scaffold. The deterministic sim core lands first; content engine, meta
progression, and WebTransport co-op follow the roadmap in [docs/DESIGN.md](docs/DESIGN.md).

## Layout

```
src/sim/      pure fixed-tick simulation core (no DOM, no Math.random)
src/render/   PixiJS renderer, decoupled from sim
data/         weapons / enemies / waves / balance targets (JSON, agent-editable)
docs/         design docs
tools/        bench harness, transcript renderer, report generator
```

## Verifying determinism

`npm run bench -- --seed-set ci` replays scripted input tapes across N seeds and asserts
the checkpoint hash chain equals the committed golden transcript. Any PR that changes
sim behavior must regenerate the transcript in a separately labeled commit.
