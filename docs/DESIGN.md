# Neon Swarm — Engineering Design

## 1. Deterministic sim core

- **Fixed 60 Hz logical ticks**, decoupled from rendering (rAF only interpolates).
- **State is one serializable plain struct.** Entities live in a fixed-size slot array —
  never a map: iteration order is itself a nondeterminism source.
- **Fixed-point math**: positions/velocities are int32 (1/1024 unit scale). Damage,
  cooldowns, XP — all integer arithmetic. Trig via a precomputed 16.16 lookup table.
  IEEE-754 `+ - * /` is identical across JS engines; `Math.sin`/`Math.atan` are not —
  banned from `src/sim` by lint.
- **Seeded `mulberry32` RNG** injected into the sim. `Math.random` / `Date.now` banned
  by lint under `src/sim`. The RNG's internal state is itself hashed.

## 2. Replay verification (hash chain)

- Every 60 ticks = 1 checkpoint. Canonical serialization (sorted keys, fixed-point as
  integers) → FNV-1a → rolling hash `h = H(h_prev ‖ tick ‖ state)`.
- Checkpoints + rolling hashes land in a **golden transcript** committed to the repo.
- Replay stores only `seed + per-tick quantized inputs` (move vector, level-up pick
  index). Replaying recomputes the hash chain; mismatch = desync or nondeterminism.
- Diagnosis: binary-search the first divergent checkpoint, dump a field-level state
  diff — the diverging field names the bug.

## 3. Bot balance loop (observe → diagnose → search → PR-with-evidence)

1. **Bench harness** (`tools/bench`): headless batch sim, N=1000 seeded runs with a
   scripted policy bot (kiting movement + priority-table upgrade picks). Emits
   `balance_report.json`: survival rate, time-to-death distribution, per-weapon DPS,
   pick rate, overkill %.
2. **Health bands** in `data/balance/targets.json`: survival 35–55% @10min, weapon
   pick-rate 5–40%, per-wave TTK bands.
3. **Nightly CI** runs bench → renders SVG→PNG charts (`tools/render_report`) → opens
   an issue when a metric leaves its band ("weapon X pick rate 3%, DPS −12% vs median").
4. **Agent retunes**: grid search over that weapon's damage/cooldown/projectiles;
   200-run bench per candidate; pick the smallest diff that lands inside the band.
5. **PR**: touches `data/**` JSON only. Body carries before/after metrics, win-rate and
   pick-rate charts, and a transcript note (unchanged = content-only; changed =
   separately labeled regen commit).
6. **Guardrails**: agent write scope is `data/**` only; `targets.json` carries parameter
   bounds; every PR links the bench artifact + seed set, reproducible via
   `npm run bench -- --seed-set <id>`.

## 4. Week-1 CI acceptance

- `vitest` green across spawn / collide / xp / level-up / death branches
- Replay gate: 50 seeds × scripted tapes → hash chain == golden transcript
- Playwright smoke: boot → move → survive 60s → draft-3 appears → die → restart
- Budgets: sim tick p99 < 2 ms @500 entities, bundle < 500 KB gz
- Pages deploy on `main`, preview build per PR

## 5. Roadmap

1. MVP loop: move / spawn / collide / xp / draft / death, tests + smoke + Pages
2. Content engine: data-driven weapons/enemies/waves — 10 weapons, 3 evolutions,
   20 enemies, 2 maps; balance simulator online
3. Meta: currency unlocks, talent tree, daily-seed challenge + leaderboard API
4. Stretch: WebTransport 2-player co-op — client prediction, server-authoritative
   replay verification (the hash chain doubles as the anti-cheat)
