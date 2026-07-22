# visual-occlusion-reasoning

## Capture LIBERO-Occ samples

The capture script renders one initialized task from each of the four
LIBERO-Occ suites and saves the images to `outputs/libero_occ_samples/`.

```bash
git submodule update --init --recursive
uv sync
MUJOCO_GL=egl uv run python scripts/capture/save_libero_occ_samples.py
```

The project uses Python 3.11 and PyTorch 2.7.1 to match the bundled OpenPI
checkout. LIBERO itself comes from OpenPI's pinned `third_party/libero`
submodule because upstream LIBERO's wheel does not include its Python package.

Use `MUJOCO_GL=osmesa` instead of `egl` on a CPU-only machine with OSMesa
installed. To render every task in all four suites (40 images), numbered
`1-` through `10-` within each suite, run:

```bash
MUJOCO_GL=egl uv run python scripts/capture/save_libero_occ_samples.py --all-tasks \
  --output-dir outputs/libero_occ_tasks
```

Run `uv run python scripts/capture/save_libero_occ_samples.py --help` for output,
resolution, camera, seed, and settling-step options.

Build a 2-row by 5-column composite for each suite from the 40 task images:

```bash
uv run python scripts/figures/make_libero_occ_composites.py
```

## Evaluate the default OpenPI π0.5 checkpoint

The evaluator covers all 40 tasks in the four released LIBERO-Occ suites,
uses every released initial state by default (50 trials per task), saves
synchronized 128×128 agent-view and wrist-camera MP4s, and continually updates
episode, task, suite, and overall statistics. Start the stock Physical
Intelligence policy server in one terminal:

```bash
cd submodules/openpi
uv sync
uv run scripts/serve_policy.py --env LIBERO
```

The default `LIBERO` server configuration downloads and caches
`gs://openpi-assets/checkpoints/pi05_libero` automatically. In a second
terminal, from this repository's root, run:

```bash
uv sync
MUJOCO_GL=egl uv run python scripts/evaluation/eval_pi05_libero.py
```

The full benchmark is 2,000 episodes and can take a long time. A one-trial
smoke run over all 40 environments is:

```bash
MUJOCO_GL=egl uv run python scripts/evaluation/eval_pi05_libero.py \
  --num-trials-per-task 1 \
  --output-dir outputs/pi05_libero_occ_smoke
```

To run all 2,000 episodes in a detached tmux session from the repository root:

```bash
ROOT="$PWD"; mkdir -p "$ROOT/outputs/pi05_libero_occ"; \
tmux new-session -d -s pi05-libero-occ -n server -c "$ROOT/submodules/openpi" \
  "set -o pipefail; XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 uv run scripts/serve_policy.py --env LIBERO 2>&1 | tee '$ROOT/outputs/pi05_libero_occ/server.log'" \; \
  new-window -t pi05-libero-occ -n eval -c "$ROOT" \
  "set -o pipefail; until nc -z 127.0.0.1 8000; do sleep 5; done; MUJOCO_GL=egl uv run python scripts/evaluation/eval_pi05_libero.py --num-trials-per-task 50 2>&1 | tee '$ROOT/outputs/pi05_libero_occ/eval.log'"
```

Attach with `tmux attach -t pi05-libero-occ`. If the run is interrupted, launch
the same command again after removing the old tmux session; `episodes.csv`
makes the evaluator skip completed dual-camera episodes.

### Matched standard-LIBERO baseline

Every released LIBERO-Occ task maps one-to-one by filename to a standard
LIBERO task, with 50 initial states on both sides. Evaluate the standard scenes
at the same task and initial-state indices by adding `--scene-variant normal`:

```bash
MUJOCO_GL=egl uv run python scripts/evaluation/eval_pi05_libero.py \
  --scene-variant normal \
  --num-trials-per-task 50
```

This writes to `outputs/pi05_libero_matched_normal/` by default. Its episode
rows can be paired with `outputs/pi05_libero_occ/episodes.csv` using suite
(after removing `_occluded`), task, and `init_state_index`.

### Compare matched normal and occluded episodes

After both full runs finish, generate a human-readable list of episodes that
succeeded normally but failed with occlusion:

```bash
uv run python scripts/evaluation/compare_pi05_libero_runs.py
```

The command strictly pairs rows by canonical suite, task, and
`init_state_index`. It writes:

- `outputs/pi05_libero_comparison/occlusion_failures.md`, grouped by suite and
  task with direct agent/wrist links for both scene variants.
- `occlusion_failures.csv`, for further analysis.
- `comparison_summary.json`, with overall and per-suite outcome counts.

## Script layout

- `scripts/evaluation/`: policy evaluation and matched-run comparisons.
- `scripts/capture/`: simulator observation capture.
- `scripts/figures/`: composite and presentation-figure generation.

Outputs are written below `outputs/pi05_libero_occ/`:

- `videos/<suite>/<task>/episode_NNN.mp4`: low-resolution agent-view rollouts.
- `videos/<suite>/<task>/episode_NNN_wrist.mp4`: synchronized wrist-camera
  rollouts.
- `episodes.csv`: one durable row per rollout, including success, control
  frames, total simulator frames, inference calls, runtime, and errors.
- `task_summary.csv` and `suite_summary.csv`: grouped success and frame stats.
- `summary.json`: the aggregate summary plus suite/task breakdowns.
- `run_config.json`: arguments, server metadata, and checkpoint identity.

Review videos use H.264, `yuv420p`, and fast-start MP4 metadata so they play in
VS Code's embedded video preview as well as desktop players. The evaluator
therefore requires the `ffmpeg` command to be available on `PATH`.

The runner resumes by default using `episodes.csv`, so an interrupted full run
can be restarted with the same command. `control_frames` excludes the standard
10 simulator stabilization steps; `sim_frames` includes them. Use `--help` to
select suites or change trial count, replanning interval, video size, and
rendering options.
