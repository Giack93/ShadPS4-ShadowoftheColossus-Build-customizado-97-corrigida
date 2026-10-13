# Background pipeline compilation (SotC build)

A graphics pipeline the game has not used before is created on low-priority worker threads
instead of on the GPU thread. While it is being built, the draws that need it are skipped, so
the frame does not wait for the driver compile: an object or effect can appear a few frames
late the first time it is seen, instead of the game freezing for tens of milliseconds.

The finished pipelines go into the same disk cache as before, so the next session loads them
at start-up and nothing is skipped for them.

Idea and queue design adapted from FelipePrado0/gow3_pc (GPL-2.0-or-later); this version only
moves the driver-side pipeline creation off the GPU thread. Shader translation, which reads
guest memory, stays where it was.

## What is built in the background

- Graphics pipelines when the driver supports dynamic vertex input state
  (VK_EXT_vertex_input_dynamic_state, current NVIDIA and AMD drivers).
- Not compute pipelines, and not pipelines with tessellation or geometry stages: those are
  still built on the GPU thread, as upstream does.

## Settings (environment variables)

| Variable | Default | Effect |
|---|---|---|
| `SOTC_ASYNC_PIPELINES` | `1` | `0` turns background compilation off (upstream behaviour) |
| `SOTC_ASYNC_PIPELINE_THREADS` | a quarter of the CPU threads, 1–4 | number of worker threads |

The log says at start-up whether it is on (`Background graphics pipeline compilation on`), and
each new pipeline is logged as `Compiling graphics pipeline ... in the background`.

## Known trade-off

A draw that happens only once (for example a texture rendered once during loading) is lost if
its pipeline is not ready yet. If something looks wrong only the first time, or stays wrong,
start the game with `SOTC_ASYNC_PIPELINES=0` once to fill the cache, then turn it back on.
