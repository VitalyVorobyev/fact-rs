# Parallel linearization in factrs: detailed report

This report backs a short upstream proposal for [rpl-cmu/fact-rs](https://github.com/rpl-cmu/fact-rs): ‹issue link›, ‹PR link›.

- The code is on branch [`feat/parallel-linearize`](https://github.com/VitalyVorobyev/fact-rs/tree/feat/parallel-linearize) of this fork, at `d2d5a53`, on upstream `main` `33b07e4`.
- This branch, `notes/parallel-linearization`, holds only this file.

## Summary

- **`+ Sync`** next to the `Send` that #47 added, on `Variable`, `VariableSafe`, `Residual`, `NoiseModel` and `RobustCost`.
  - Every type in the crate already satisfies it, and `Graph`, `Factor` and `Values` become `Sync`.
  - Like #47, it breaks only a user type that is not `Sync`, such as one holding a `Cell`, `RefCell` or `Rc`.
- **With the existing `rayon` feature**, `Graph::linearize` and `Graph::error` map over the factors with `par_iter`.
  - The factor order is kept, and `error` sums the per-factor errors in factor order. Both results are therefore bit-identical to the serial ones for any number of threads.
  - Without the feature nothing changes.
- **On 12 threads:**
  - Linearizing a synthetic calibration gets 2.6–5.8× faster, and factrs' own LM solve 1.15–1.45×.
  - The bundled g2o pose graphs solve 1.2–1.3× faster with Gauss–Newton.
  - calibration-rs, which drives `Graph::linearize` from its own LM, solves 3.9× faster across 150 scenes (geometric mean; 2.6–5.1× by workflow), with bit-identical results.
- **The cost** is a fixed handoff to rayon's pool of roughly 7 µs per `linearize` or `error` call.
  - It makes `Graph::error` slower on graphs below about 2000 factors.
  - On one thread it adds +0.4 % over calibration-rs's 150 scenes, and about +5 % on its smallest ones.
  - A serial fallback for small graphs would remove it (see the follow-ups at the end).

## Why calibration needs it

[calibration-rs](https://github.com/VitalyVorobyev/calibration-rs) uses factrs as one of its two solver backends. It covers single cameras, multi-camera rigs, hand-eye and laser planes.

- **The graph is large.** A calibration problem has one reprojection factor per detected target corner: tens of views with about a hundred corners each, so thousands of factors.
- **Each factor is expensive.** It projects a target point through a pinhole camera with lens distortion (9 parameters) and the view's SE(3) pose. That is a 2×15 Jacobian per factor, much more work than a pose-graph factor.
- **Linearization dominates the solve.** calibration-rs calls `Graph::linearize` from its own Levenberg–Marquardt loop ([ADR 0025](https://github.com/VitalyVorobyev/calibration-rs/blob/main/docs/adrs/0025-factrs-backend-shared-lm.md)), so factor evaluation dominates the solve.
- **factrs is fastest per core, but loses on a desktop.** Its static-size duals make it the fastest Rust backend per core that calibration-rs has measured. Yet `Graph::linearize` and `Graph::error` run on one thread, and the `rayon` feature only switches on faer's parallel kernels. So on a desktop factrs loses to backends that evaluate factors in parallel.

## The change

Three commits on `feat/parallel-linearize`:

1. **`feat!: Require Sync …`**
   - Adds the bounds.
   - `tests/thread_safety.rs` replaces `tests/graph_and_factors_are_send.rs` and also asserts that `Graph`, `Factor` and `Values` are `Send` and `Sync`.
2. **`feat: Linearize and evaluate factors in parallel …`**
   - Sets `rayon = ["faer/rayon", "dep:rayon"]`.
   - `linearize` uses an order-preserving `par_iter().map().collect()`.
   - `error` collects the per-factor errors, then sums them in factor order.
   - A rayon task takes at least 32 factors to linearize, or 512 to evaluate. A factor's error costs tens of nanoseconds, and smaller tasks made small graphs slower than serial.
   - `tests/parallel.rs` checks on sphere2500 that the error, the residual and every Jacobian value match a factor-by-factor serial evaluation, bit for bit.
3. **`bench: Add a camera calibration benchmark`**
   - Adds `factrs-bench/benches/reprojection.rs`: views of a planar target, one reprojection factor per corner, with a 9-parameter camera and an SE(3) pose, 15 tangent dimensions per factor.
   - It times an LM solve, one linearization and one error evaluation at 540, 1760 and 4680 factors.
   - The commit is optional upstream.

**Still serial:**
- `LinearGraph::error`, which LM calls per damping trial;
- the assembly of the sparse Jacobian in `LinearGraph::residual_jacobian`;
- forming `JᵀJ` in LM;
- GNC's per-factor weight update (`gnc.rs:243`).

None of these is touched.

## Measurements

Apple M4 Pro (8 performance + 4 efficiency cores, 12 threads), macOS 26, rustc 1.99.0, 2026-10-10. factrs at `d2d5a53`. The values are divan medians of 100 samples. The benches keep faer sequential (`faer::Par::Seq`), as the upstream g2o bench does, so only factor evaluation runs in parallel.

### Reprojection bench (`factrs-bench/benches/reprojection.rs`)

| factors | | serial | `rayon` | speed-up | `rayon`, 1 thread |
|---:|---|---:|---:|---:|---:|
| 540 | `linearize` | 270 µs | 104 µs | 2.6× | 284 µs |
| 1760 | `linearize` | 907 µs | 238 µs | 3.8× | 902 µs |
| 4680 | `linearize` | 2.45 ms | 0.42 ms | 5.8× | 2.38 ms |
| 540 | `error` | 20.1 µs | 26.8 µs | 0.75× | 18.9 µs |
| 1760 | `error` | 62.6 µs | 70.1 µs | 0.89× | 71.3 µs |
| 4680 | `error` | 162 µs | 95 µs | 1.7× | 169 µs |
| 540 | LM solve | 2.94 ms | 2.55 ms | 1.15× | 3.09 ms |
| 1760 | LM solve | 11.6 ms | 8.9 ms | 1.30× | 11.8 ms |
| 4680 | LM solve | 31.1 ms | 21.5 ms | 1.45× | 31.2 ms |

- **Linearization scales well**, up to 5.8×.
- **`Graph::error` costs about 7 µs more on graphs below about 2000 factors.** That is the latency of handing the work to rayon's pool and waking its threads. A call evaluates only tens of µs of work, so it gains nothing until the graph is a few thousand factors.
  - It is a fixed cost per call, and the LM solves on the same graphs are still faster.
  - A serial fallback below a size threshold would remove it. The right threshold depends on the factor type and the machine, so the patch leaves it out.
- **factrs' own LM gains less than linearization.** Forming `JᵀJ` (a general sparse product), the λ trials (`LinearGraph::error`) and the sparse solve stay serial. In the 2026-10-06 profile of the same code (not re-run for this report), `JᵀJ` alone was 37 % of a serial solve.
- **On one thread**, `rayon` linearizations and solves stay within −3 % to +6 % of serial, which is the run-to-run spread of these benches. `error` on one thread ranges from −6 % to +14 %.

### g2o pose graphs (`factrs-bench/benches/g2o.rs`, Gauss–Newton)

| dataset | serial | `rayon` | speed-up | `rayon`, 1 thread |
|---|---:|---:|---:|---:|
| M3500 | 39.2 ms | 32.5 ms | 1.21× | 41.2 ms |
| sphere2500 | 189.3 ms | 154.5 ms | 1.23× | 191.0 ms |
| parking-garage | 108.2 ms | 81.0 ms | 1.34× | 114.4 ms |

- Pose graphs gain less, and that is expected.
- In the 2026-10-06 profile (not re-run), factor evaluation was 23–34 % of a serial run and the sparse solve 58–71 %. So by Amdahl's law, 12 threads could save at most 1.27–1.45×.

### calibration-rs end to end

This is `calib-bench solver run --preset full --backend factrs` at calibration-rs `00a8ab00`, on the same machine.

**Setup**
- The run covers 150 synthetic scenes across nine workflows: three sizes (8 to 36 views), two pixel-noise levels, and 5 % outliers under Huber or Cauchy losses.
- Each time is the median of three solves after a warm-up, covering all non-linear stages.
- The comparison is crates.io factrs 0.3.0 against this fork's [`calib-rs/parallel-on-0.3.0`](https://github.com/VitalyVorobyev/fact-rs/tree/calib-rs/parallel-on-0.3.0) (`c73d022`). That branch is the same `src/` change on 0.3.0 (plus #47), because calibration-rs is still on faer 0.23. `rayon` is enabled.
- faer's own `rayon` feature is already on in calibration-rs's baseline build, so the difference is the factor evaluation alone.

| workflow | scenes | serial (median scene) | speed-up (geo. mean) | range | `rayon` on 1 thread vs serial on 1 thread |
|---|---:|---:|---:|---:|---:|
| planar intrinsics | 18 | 32 ms | 2.56× | 1.37–3.32× | +4.6 % |
| rig extrinsics | 18 | 123 ms | 2.77× | 1.78–3.58× | −0.5 % |
| rig laser plane | 6 | 85 ms | 3.85× | 3.54–4.53× | −3.7 % |
| rig extrinsics (Scheimpflug cameras) | 18 | 787 ms | 4.03× | 3.03–4.79× | −3.4 % |
| Scheimpflug intrinsics | 18 | 411 ms | 4.27× | 3.62–4.84× | −1.0 % |
| rig hand-eye | 18 | 628 ms | 4.34× | 3.31–5.13× | −0.5 % |
| single-camera hand-eye | 18 | 252 ms | 4.40× | 2.83–5.19× | −0.1 % |
| laser plane | 18 | 152 ms | 4.92× | 4.02–5.55× | −1.1 % |
| rig hand-eye + laser | 18 | 1564 ms | 5.08× | 4.55–5.66× | +6.7 % (re-run: +4.0 %) |
| **all scenes** | **150** | | **3.94×** | 1.37–5.66× | +0.4 % |

- **Total solve time** over the 150 scenes goes from 95.4 s to 20.5 s.
- **Results are bit-identical across all four runs** (serial, `rayon`, and each on one thread):
  - the same final cost, to the bit, in all 144 scenes that report it;
  - the same iteration count in those 144 scenes;
  - the same objective, to the bit, in all 150 scenes.
- **The smallest scenes gain least.** Planar intrinsics on 8 views has about 500 factors.
- **On one thread**, the change costs +0.4 % over all scenes (geometric mean).
  - Small solves pay the fixed dispatch cost that the `error` row above shows. For example, a 5.4 ms planar Huber solve takes 5.8 ms, consistently across its samples.
  - The rig hand-eye + laser scenes were noisy in the first run, with one sample at 1.4× the next. A second run gave +4.0 % (−10 % to +27 % per scene), again with bit-identical costs.

## Reproduce

factrs, on `feat/parallel-linearize`:

```sh
cargo bench -p factrs-bench --no-default-features --features factrs --bench reprojection
cargo bench -p factrs-bench --no-default-features --features factrs,factrs/rayon --bench reprojection
RAYON_NUM_THREADS=1 cargo bench -p factrs-bench --no-default-features --features factrs,factrs/rayon --bench reprojection
# the same three with --bench g2o
```

calibration-rs, at `00a8ab00`:

```sh
# baseline: crates.io factrs 0.3.0
cargo run --release -p vision-calibration-bench --bin calib-bench -- \
  solver run --preset full --backend factrs --out serial.json
# parallel: in the workspace Cargo.toml, set
#   factrs = { version = "0.3", default-features = false, features = ["rayon"] }
# and add
#   [patch.crates-io]
#   factrs = { path = "<this fork, checked out at calib-rs/parallel-on-0.3.0>" }
cargo run --release -p vision-calibration-bench --bin calib-bench -- \
  solver run --preset full --backend factrs --out rayon.json
# one thread: the same two runs with RAYON_NUM_THREADS=1
```

## Possible follow-ups (not proposed)

- **A serial fallback for small graphs in `Graph::error`**, as discussed above.
- **Parallel `LinearGraph::error` and Jacobian assembly.** This would help factrs' own optimizers, which calibration-rs does not use.
