# Models

Model binaries are **not committed** — they are large and rebuildable.

Place here after export:

| File | Mode | Notes |
|---|---|---|
| `fire_smoke.tflite` | `cpu` | int8 quantised, for accuracy testing on x86 |
| `fire_smoke_edgetpu.tflite` | `edgetpu` | compiled for Coral, production on ARM64 |
| `labels.txt` | both | committed — `0 fire` / `1 smoke` |

Export instructions: `scripts/export_colab.md`.
The Edge TPU compiler does not run on ARM — export and compile on x86 or Colab.
