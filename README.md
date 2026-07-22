# visual-occlusion-reasoning

## Capture LIBERO-Occ samples

The capture script renders one initialized task from each of the four
LIBERO-Occ suites and saves the images to `outputs/libero_occ_samples/`.

```bash
git submodule update --init --recursive
uv sync
MUJOCO_GL=egl uv run python scripts/save_libero_occ_samples.py
```

The project uses Python 3.11 and PyTorch 2.7.1 to match the bundled OpenPI
checkout. LIBERO itself comes from OpenPI's pinned `third_party/libero`
submodule because upstream LIBERO's wheel does not include its Python package.

Use `MUJOCO_GL=osmesa` instead of `egl` on a CPU-only machine with OSMesa
installed. To render every task in all four suites (40 images), numbered
`1-` through `10-` within each suite, run:

```bash
MUJOCO_GL=egl uv run python scripts/save_libero_occ_samples.py --all-tasks \
  --output-dir outputs/libero_occ_tasks
```

Run `uv run python scripts/save_libero_occ_samples.py --help` for output,
resolution, camera, seed, and settling-step options.

Build a 2-row by 5-column composite for each suite from the 40 task images:

```bash
uv run python scripts/make_libero_occ_composites.py
```
