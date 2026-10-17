# Performance

What was measured, how, and what the numbers mean. Every figure here comes from
Veyra's benchmark suite. Nothing is estimated. Numbers describe one workflow on one machine; they are not
a promise of real-time behaviour for any other.

## Method

- **Machine:** Intel Core i5-1235U laptop (10 cores / 12 threads), 8 GB of memory available to WSL2 (Linux 6.18),
  CPU only. Python 3.12, OpenCV 5.0, NumPy 2.5, ONNX Runtime 1.30 (CPU execution provider).
- **Filesystem:** the source tree was copied to the Linux filesystem first, because file and import timings on a
  Windows drive mounted in WSL are several times slower.
- **Data:** the labelled flange sample set (44 images of 1600 × 1200, colour) and an anomaly model trained on 16 good
  parts, both created fresh by the benchmark in a temporary data folder.
- **Runs:** 100 per figure after 5 to 20 warm-up runs. Medians and percentiles are over those runs.

## Results

### Engine overhead

A chain of pass-through tools measures what the engine itself costs, apart from any vision work:

| Chain | Whole run | Per node |
|---|---|---|
| 10 nodes | 125 µs | 11.4 µs |
| 50 nodes | 541 µs | 10.6 µs |
| 200 nodes | 2.66 ms | 13.3 µs |

Each node costs input gathering, fingerprinting for the cache, output type checks and a record. For inspection
workflows of 5–50 tools this is well under a millisecond, small next to any real image operation.

### The sample inspection

17 tools on one 1600 × 1200 image: region, grayscale, blur, threshold, morphology, blob analysis, a measurement, colour
statistics, the ONNX anomaly model, four checks, the decision, the final result and the output signals.

| Case | p50 | p95 | p99 |
|---|---|---|---|
| Full run, 1 engine thread | 16.7 ms | 26.4 ms | 32.0 ms |
| Full run, 2 engine threads | 17.9 ms | 21.0 ms | 25.9 ms |
| Full run, 4 engine threads | 17.1 ms | 21.4 ms | 31.2 ms |
| Unchanged re-run (every tool from the cache) | 0.22 ms | 0.27 ms | 0.30 ms |
| One limit changed: recompile, incremental run, model kept loaded | 0.53 ms | 0.66 ms | 0.85 ms |

Slowest tools in a full run: blob analysis 6.8 ms (connected components plus outlines of each kept blob), the
anomaly model 3.7 ms (resize, normalise, ONNX inference, regions), colour statistics 1.8 ms.

What this shows:

- **More engine threads do not help this workflow.** Its branches are light, and OpenCV already uses several cores
  inside one operation. The median does not improve; only the tail narrows slightly. This is why one thread is the
  default. Parallel branches pay off when several heavy, GIL-releasing operations (two models, large filters) are
  independent. The scheduler supports that, bounded by its resource tokens.
- **Changing a parameter is interactive.** An engineer adjusting a limit waits about half a millisecond for the
  engine: the plan is recompiled, the loaded model is kept (its set-up parameters did not change), and only the
  changed check and the decision after it run again. In the studio, the rest of the round trip is the protocol and
  drawing.

### Batch throughput

The labelled folder through the same engine, decoding included: **15.6 images/s** (latency per image p50 13.7 ms,
p95 20.4 ms). Decoding a 1600 × 1200 PNG takes longer than inspecting it. A camera delivers raw frames and skips that
step.

### Simulator

Rendering a simulated part (compositing, noise and a defect when one is drawn) takes 56 ms at the median, so the
simulated camera sustains about 15 parts per second per camera on this machine. It is a test source, not a benchmark
of acquisition.

### Previews between the runtime and the studio

A full 1600 × 1200 colour preview (5.8 MB of raw pixels) from the runtime process to a client, request to last byte:
p50 9.2 ms, p95 13.1 ms. Previews are sent only for what the studio shows, at display rate. Production camera tiles use
reduced previews (at most 960 pixels on the long side) at up to four per second per camera.

### The production screenshot

The Production page in the README runs four simulated cameras at three parts per second each, in the same runtime
process as the inspections. Its latency figure (acquisition to result, p95 around 190 ms there) includes waiting for
the simulator's rendering, which competes with the inspections for the same interpreter. It says how this demo
behaves, not what a station with real cameras (which deliver frames without Python-side rendering) would reach.

## Storage at production scale

The storage benchmark fills a data folder (on the Linux filesystem) with 500,000 inspections over
365 days on four cameras (85 % OK, 12 % NG, 2 % indeterminate, 1 % error; most NG reviewed; two checks per
inspection) and 20,000 image files on the newest ones, then times what the studio and the housekeeper do. Medians of
five runs (three for the slower ones), same machine as above.

| What | Time |
|---|---|
| Newest page of the inspection list (120 rows) | 1.0 ms |
| One day's NG inspections, first page | 1.1 ms |
| Counting all inspections | 1.2 ms |
| Calendar of the whole year by day (366 days × result), no filter | 228 ms |
| The same calendar for NG only | 23 ms |
| Unreviewed rejects (the overview's review backlog) | 46 ms |
| Overview of today | 5.9 ms |
| Overview of the last 30 days (refreshed every 5 s while production runs) | 97 ms |
| Report of the last 30 days | 277 ms |
| Storage usage (database sizes and a walk of the other folders) | 368 ms |
| Cleanup preview of the policy | 152 ms |
| Cleanup of expired OK images (8,861 files) | 2,500 images/s |
| Excel export of the last 30 days (41,095 inspections, 82,190 checks, 8 MB) | 2,880 inspections/s (14 s) |

Two changes came from these measurements. The calendar first grouped every row by a local-date string (727 ms for the
year); it now counts exact local-midnight ranges (the same boundaries the list uses when a day is chosen) through a
covering index on `(at, status)`, which replaced the plain index on `at`: 228 ms. The Excel export first ran at 1,900
inspections/s; openpyxl writes about 1.7 times faster with lxml installed, and plain date values instead of styled
cells added the rest.

## What is not measured yet

- Behaviour with industrial cameras: acquisition latency and jitter, trigger-to-result time, and dropped frames under
  sustained overload. The OpenCV backend was run on a laptop's UVC camera (15 frames/s at 640 × 480) and the pylon
  backend on pylon's emulator; neither is a production camera measurement.
- GPU inference and larger models (none are implemented yet).
- Long soak runs (hours) to check that memory stays flat. The queues, histories and caches are bounded by
  construction, but that is not a measurement.
- Windows timings. The suite passes on Linux; the benchmark has not been run on native Windows.
