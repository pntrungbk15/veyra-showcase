# Veyra architecture

Veyra is a platform for designing, validating and operating machine-vision inspections. An engineer builds an
inspection as a typed workflow of vision tools (image processing, blob analysis, measurement, colour, AI models),
tests it against sample images, folders and simulated cameras, and runs the same workflow in production against
cameras and I/O. Every result is typed, located in the coordinates of the original image, and traceable to the exact
workflow, model and configuration that produced it.

This document records the design and the reasons for it. It describes the current release; where something is
planned rather than built, it says so. Section numbers are referred to from the other documents.

## 1. What shaped the design

**Principles, in priority order.** When two conflict, the earlier one wins.

1. **Results are trustworthy.** Original images are never modified. Inference never sees visualisation. An error
   never becomes GOOD. A result says which workflow, model and configuration produced it.
2. **No hardware is needed to do real work.** A sample image, a folder or the simulator exercises exactly the
   engine, nodes and decision logic that a camera would. This is not a separate test mode.
3. **The engine does not depend on the interface.** The engine and runtime never import Qt. The desktop
   application is one client of a runtime process that can also run headless.
4. **Costs are bounded and measured.** Queues are bounded, frames that are dropped are counted, every node is
   timed, and performance claims come from benchmarks.
5. **Few, meaningful boundaries.** A folder is added when a second real user or a real size problem appears,
   not in advance.

**What the workspace offered.** Veyra sits in a workspace of independent repositories:

- FluentQt Community supplies windowing, navigation, controls, theme and `fluentqt.storage`, which is SQLite
  with migrations and has no Qt dependency apart from its worker.
- FluentQt Pro supplies `ImageViewer`, `PropertyInspector`, `ChartView`, `ThumbnailStrip`, `FilterBar`,
  `CommandPalette` and the vision overlays.
- No industrial-vision code existed.
- The workspace's engineering rules carry over directly:
  - Engines and inference run in a separate process, never the GUI thread.
  - On Windows, ONNX Runtime must be loaded before PyQt5 in any process.
  - Worker updates are throttled and tagged with a request id.
  - Persistence goes through numbered migrations.

  The first two rules alone make a separate runtime process necessary, not optional.

## 2. Name

Data folders, workflow files, model manifests and databases written by earlier versions are read as they
are; the SQLite application id stays the one databases were created with, so existing databases keep opening.

## 3. Boundaries

```
veyra/                         the application package (distribution "veyra")
  imaging.py                     Frame (immutable original), Image (derived view + transform), Transform, Roi, Calibration
  results.py                     typed results, geometry, statuses, verdicts, InspectionResult — plain data, JSON round-trip
  workflow/                      the workflow model and engine (no Qt, no OpenCV)
    types.py                     pin types and their compatibility table
    nodes.py                     NodeSpec, ports, parameters, the node registry and the @node decorator
    document.py                  Workflow document, editing operations (undo/redo), validation
    serialize.py                 versioned JSON format and migrations
    engine.py                    compile → Plan; run with scheduler, cache, events, immutability checks
  tools/                         built-in node library, one module per category (acquisition, processing, alignment,
                                 blob, measure, geometry, color, pattern, identify, ai, logic, output)
  acquisition.py                 sources (image file, folder, dataset, simulator) and the bounded FrameQueue
  cameras.py                     live cameras (OpenCV/UVC and streams, Basler pylon, GenICam/GenTL, folder replay), discovery,
                                 camera tests
  calibration.py                 checkerboard calibration (lens and scale), known-length scale, undistortion maps
  simulation.py                  deterministic synthetic parts with ground truth, used by the simulator and samples
  models/                        model adapters (Veyra's own, Ultralytics and Anomalib exports, OCR), ONNX Runtime, the model
                                 registry, evaluation, the frameworks' inventory and its verification record
  engines/                       the AI environment: probing an interpreter, running worker scripts (workers/, which import
                                 nothing from Veyra) for Ultralytics and Anomalib jobs, cancellation
  training/                      training jobs: Veyra's own trainers, and external.py for the frameworks (layouts, parity,
                                 provenance)
  formats.py                     dataset layouts of the frameworks, and YOLO, COCO and DOTA import and export
  synthetic.py                   generated sample datasets for every task
  templates.py                   workflow templates for common inspections
  production.py                  station, production state machine, counters, inspection events, config snapshots
  devices.py                     I/O abstraction, the mock module and Modbus TCP
  releases.py                    versions of workflows, models and other artefacts: download, verify, stage,
                                 validate, approve, activate, roll back
  store.py                       data folder, model registry rows, inspection records, holds, settings, events, snapshots,
                                 audit log (fluentqt.storage); adopting a data folder from before the rename
  outcomes.py                    the vocabulary of results (OK, NG, Indeterminate, Error) and of the rates computed from them
  retention.py                   storage categories, retention policy, cleanup plan and journalled cleanup, usage, growth
  backups.py                     verified database backups, their rule, restore at the next start
  overview.py                    the production overview and the NG-rate alert
  reports.py, exports.py         period reports (CSV, JSON) and Excel workbooks and evidence packages
  logs.py                        log files per process and day, crash log
  security.py                    roles, permissions, password hashing
  samples.py                     the sample flange inspection, built in code for tests, samples and screenshots
  runtime/                       the runtime process: session, protocol, host (process entry), previews, and one command
                                 module per domain (design, station, data, lifecycle, training, storage, system,
                                 overview, admin) with the housekeeper
  overlay.py                     visualisation primitives built from results (never from pixels)
  ui/                            the FluentQt studio: window, pages, canvas, graph, panels, evidence viewer, camera and
                                 export dialogs, sign-in, runtime client (the only Qt code)
  cli.py                         headless commands: runtime, run a workflow on a file, folder or simulator, samples, train,
                                 users (local recovery)
```

Why this shape:

- **`imaging` and `results` are single modules at the root.** Every other part uses them. They are small, stable
  contracts, and a `core/` package would add a level without adding meaning.
- **`workflow/` is a package.** It is the largest pure-logic area, with five clearly separate concerns, and it
  must stay free of OpenCV so it can be reasoned about and tested on its own.
- **`tools/` holds the node implementations.** It is the part that grows fastest. One module per category keeps
  "where is the Threshold tool" obvious. A category becomes a sub-package only when it outgrows one file.
- **`acquisition`, `devices`, `production`, `releases` and `store` start as single modules.** Each will grow
  (camera backends, PLC protocols), and each becomes a package when its first second backend arrives.
- **`runtime/` is the process boundary.** It owns threads, queues and the protocol. The engine inside it is the
  same one the tests and the CLI call directly.
- **`ui/` is the only Qt-dependent code.** A test enforces that nothing else imports PyQt5.

Rejected: separate `domain/`, `application/` and `infrastructure/` layers, because the dependency direction below
already gives their benefit without three levels of folders; a plugin SDK, because nodes are registered with a
decorator today and a manifest-based module system comes when a node is shipped separately (section 16); and
microservices, because one runtime process per station is the whole distributed system until a requirement says
otherwise.

**Dependency direction** (enforced by tests):

```
ui ──► runtime.client (protocol only) ──► [ runtime process ]
                                            runtime ─► production, releases, store, retention, exports, cameras, …
                                            tools ─► workflow, imaging, results, models (contracts)
                                            workflow ─► imaging, results
```

`workflow` never imports `tools`. Nodes register themselves, and the engine knows only `NodeSpec`s. So the engine has
no knowledge of OpenCV, of any model, or of cameras.

## 4. Image lifecycle and coordinates

This is the design's central invariant, so it is built into the types rather than left to convention.

- **`Frame`** is the original image as acquired, from a camera, a file, a dataset or the simulator. Its pixel
  array is marked read-only at creation. NumPy refuses in-place writes, and so does OpenCV, including drawing
  functions. A `Frame` carries `FrameMeta`: source kind and id, camera role, frame id, trigger id, timestamp,
  line and lane, and file path.
- **`Image`** is anything derived from a frame: a crop, grey conversion, threshold output, mask or model input.
  It holds a read-only array and `toFrame`, a 3×3 projective `Transform` that maps its pixel coordinates into its
  frame's coordinates. A `Frame` becomes an `Image` with the identity transform without copying (`frame.image`).
- **Every node that changes geometry composes the transform.** Crop composes a translation, resize a scale,
  rotation or perspective their matrices. Nodes that only change values (blur, threshold) keep it unchanged.
- **All result geometry is stored in frame coordinates.** A node maps its local coordinates through
  `image.toFrame` once, when it builds the result. Downstream fusion, rules, overlays and storage never convert
  coordinates, because everything already agrees.
- **Model input is one more `Image`.** An adapter's `prepare()` returns the tensor and the transform from tensor
  coordinates to the image it was given (letterbox or resize). Detections at 640 × 640 map to the original through
  `image.toFrame ∘ inputToImage`. That one composition is the only coordinate formula in the AI path.

The coordinate regression test runs the whole workflow, not just the helper. It feeds a 1920 × 1080 frame through a
ROI at (300, 200, 1200 × 600) and a model input of 640 × 640. It then checks that detections land in original
coordinates, that ROI offset and scale are both applied, that overlay primitives align, that the frame is
bit-identical afterwards, and that no downstream node received an overlay.

**Visualisation can never reach inference, and the pin types enforce it.** No node outputs a drawn image. Overlays
are primitives built from results in `overlay.py` and drawn by the interface (or by the runtime when saving
annotated images). The one node that renders an annotated image, for storage, outputs the pin type `Rendering`,
which no processing or model input accepts. A connection that tries fails validation.

**Enforcement in depth:** read-only arrays stop accidental writes; in strict mode, used by the test suite, the
engine checksums every image input before and after each node; the type table stops structural mistakes.

## 5. Acquisition

```
Source ──► Frame ──► FrameQueue (bounded, policy) ──► Plan.run({role: Frame})
```

- A **workflow** starts at one or more **Frame input** nodes. Each names a *camera role* such as `top` or
  `side`, not a device.
- A **binding** maps each role to a source when the workflow is run. In engineering mode the source is usually an
  image file or a folder; in batch mode a folder or dataset; in production a camera or the simulator. The
  workflow document does not contain the binding. That is why the same file runs everywhere.
- **Sources** implement one small contract: `open()`, `grab(timeout) -> Frame`, `trigger()`, `close()`, `info`.
  Built in: `ImageFileSource`, `FolderSource` (sorted, looping or finite), `DatasetSource` (a folder with a
  labels manifest or `good/` and `ng/` subfolders, which yields ground truth), and `SimulatedCamera`
  (synthetic parts with configurable defect rates and timing).
- **Live cameras** (`cameras.py`) implement the same contract on a reader thread that keeps only the newest frame,
  counts the frames nobody took, retries failed reads and reconnects after a silence. Only when reconnecting fails
  does `grab` raise, which faults that camera's job (never an OK result, never the other cameras). Backends and what
  each was verified against:

  | Backend | Covers | Verified against |
  |---|---|---|
  | `opencv` | UVC cameras and webcams (DirectShow, Media Foundation, V4L2), RTSP/HTTP streams | a laptop's built-in UVC camera on Windows |
  | `pylon` | Basler USB3 Vision and GigE Vision cameras through `pypylon` | pylon's camera emulator (automated tests); not yet a physical Basler camera |
  | `genicam` | cameras of most makers through their vendor's GenTL producer (`harvesters`) | the simulated devices of the `genicam` package's GenTL producer; Basler's producers load; not yet a physical camera |
  | `replay` | a folder of images played back at a frame rate | automated tests |

  Cognex and Keyence (their own SDKs) are listed in the studio as not implemented, with the reason. Device
  controls are those the device reports; every value set is read back, and a value the device did not take is
  reported as such (see [cameras.md](cameras.md)).
- **Validation understands bindings.** A workflow is never "missing a camera" in design or batch mode. Readiness
  for production checks that every role is bound to a real, configured source (section 13).

**Queues.** `FrameQueue` has a fixed capacity and one of three policies:

| Policy | Behaviour when full | Used for |
|---|---|---|
| `latest` | the oldest frame is replaced, and the drop is counted | live view, free-run |
| `dropNewest` | the new frame is discarded, and the drop is counted | bursts where order matters more than recency |
| `block` | the producer waits, with a timeout | triggered inspection, where every part matters; a timeout is reported as an overrun |

Nothing accumulates without bound. Queue depth and drop counts are part of the station metrics.

## 6. Workflow model

- A **workflow** is a list of node instances (`id`, `type`, `typeVersion`, `params`, `title`, `enabled`)
  and a list of links (`from node.port`, `to node.port`), plus metadata. Editor layout (positions) is stored
  beside the graph in the same file but never read by the engine.
- **Pins are typed.** The types are `Frame`, `Image`, `Mask`, `Number`, `Integer`, `Bool`, `Text`, `Point`,
  `Box`, `BlobSet`, `Measurement`, `Detections`, `Anomaly`, `Classification`, `ColorStats`, `Verdict`,
  `Inspection`, `Rendering` and `Signal`. Compatibility is an explicit table with a few widenings:
  `Frame → Image`, `Mask → Image`, `Integer → Number`, and `Measurement → Number` (its value). There is no `Any`.
  An input may accept several links (a *multi* input, for example the verdicts of a Decision node).
- **Nodes** are declared with `@node(...)` and ports, parameters and resources:
  - Parameters are typed (int, float, bool, choice, text, roi, model, colour) with ranges, units and
    descriptions. The property inspector is generated from them.
  - A function implements a stateless node.
  - A class with `setup` / `run` / `close` implements a node that holds resources, such as a loaded model.
  - Each node type has a version and can migrate older parameters.
- **Graph view and step view are two views of one document.** The step view lists the nodes in topological order
  like a tool list, the graph view draws them as a node graph, and both edit the same `Workflow` through the same
  editing operations.
- **Undo and redo** are command objects on the document (add, remove, connect, disconnect, set parameter,
  move), so they need no snapshots and work the same from either view.
- **Validation** returns every problem at once as diagnostics, each with node, port, severity, message and
  suggestion. It checks unknown node types, type mismatches, missing required inputs, cycles, parameters out of
  range, invalid ROIs, missing models, several decision outputs, and (for production) unbound camera roles.
- **No arbitrary Python.** A workflow contains data, never code. Code runs only from nodes installed with the
  application. Developer scripting, if it ever exists, will have its own trust, sandbox and audit boundary.

### Serialisation

A workflow file is JSON:

```json
{"format": "veyra-workflow", "schema": 1, "id": "…", "name": "Flange inspection", "revision": 7,
 "nodes": [{"id": "roi", "type": "image.roi", "version": 1, "params": {"roi": [300, 200, 1200, 600]}}],
 "links": [{"from": ["input", "frame"], "to": ["roi", "image"]}],
 "layout": {"roi": [240, 80]}, "metadata": {"createdWith": "0.1.0"}}
```

Values are native JSON, ports are named, links are stored once, and Python objects are never serialised. `schema`
changes only when the layout of the file changes, and migrations run in order. A node's `version` lets the node
type migrate its own parameters.

## 7. Engine

`compile(workflow) → Plan`:

1. Validate. A workflow with errors does not compile, and the diagnostics say why.
2. Prune disabled nodes. Their dependants become `SKIPPED` (unless the input is optional).
3. Order topologically and assign each node's inputs to the outputs that feed them.
4. Instantiate node implementations. Stateful nodes run `setup` once per plan, so a model loads once.

`Plan.run(frames, context) → RunRecord`:

- **Scheduling** uses one bounded thread pool per plan. A node becomes ready when its inputs are resolved.
  Ready nodes start when the resources they declare are available (`cpu` tokens by default, `gpu`/`model`
  tokens for model nodes), so independent branches run in parallel without unlimited concurrency. With one
  worker the same code runs sequentially, which is the fastest choice for small graphs because OpenCV
  already parallelises inside an operation. It is not one thread per node.
- **Node states** are `WAITING`, `RUNNING`, `COMPLETED`, `SKIPPED`, `WARNING`, `ERROR`, `TIMEOUT` and
  `CANCELLED`.
  - An error in one node skips its dependants but not independent branches.
  - A node running past its time limit becomes `TIMEOUT` and its late outputs are discarded. A thread cannot
    be killed, which is one reason heavy stacks will be able to run in their own worker process.
  - Cancellation is cooperative, between nodes.
- **Records.** Each node gets its state, duration (`perf_counter_ns`), outputs, warnings and error (message and
  traceback). The run gets its frames' metadata, the workflow revision and the model versions used.
- **Incremental re-run.** Each node output is cached under a fingerprint: node type and version, parameters,
  and the fingerprints of its inputs, with frames identified by content. Changing a threshold re-runs only that
  node and its dependants. Changing a parameter a stateful node's `setup` does not depend on keeps the loaded
  model. So adjusting a limit never reacquires the image or reloads a model.
- **Stepping** runs one ready node at a time and reports it. That gives Step, Run to end and Stop. Pause and
  resume apply between frames in batch and production loops.
- **Events** are plain dataclasses: `NodeStarted`, `NodeFinished`, `RunFinished`. The runtime converts them into
  throttled protocol messages. The engine never talks to the interface.

## 8. Traditional vision

Deterministic vision is as important as AI. The tools (72 in 0.3, catalogued with measured accuracy in
[vision-tools.md](vision-tools.md)) cover image conversion and contrast (colour spaces, CLAHE, flat-field correction,
gamma, denoising, sharpening, gradients), thresholds by brightness and by colour, morphology including top-hat, hole
filling, watershed separation of touching objects and skeletons, geometry (resize, rotate, flip, perspective
rectification), blob analysis and shape measurement (rotated-rectangle length, width and angle, Feret diameters,
solidity), edges, calipers, line and circle finding, angles and point-to-line distances, pattern matching, alignment
to a taught reference (features with RANSAC, phase correlation, ECC) with fixture-relative regions, golden-template and
structural comparison, colour statistics and ΔE, Data Matrix, QR, PDF417, Aztec and linear codes, OCR, lens
undistortion, arithmetic and logic on values, and checks of every kind.

- **Calibration** (`mm`, `µm`, `inch` per pixel) belongs to the camera binding, because it is a property of the
  optics. A measurement node reports value and unit. The same workflow therefore measures correctly on two
  stations with different lenses.
- **Alignment is one more transform.** An aligned image is the input resampled onto the reference's grid, with
  `toFrame` = the image's transform composed with the found motion; a region in *this image* coordinates after it is a
  fixture, and every result still lands on the original frame.
- **The lens, when calibrated, belongs to the camera too.** A checkerboard calibration stores the camera matrix and
  distortion with the scale; *Remove lens distortion* uses them, and refuses to run on a crop (the maps are for the
  whole frame).

## 9. AI models

- **A `ModelAdapter` is the only thing that knows a model.** It has:
  - task (`anomaly`, `detection`, `segmentation`, `classification`, `ocr`), input specification and output type;
  - `load(artifact, device)`, `prepare(image) → (tensor, inputToImage)`, `predict(image) → typed result`,
    `benchmark()` and `close()`.

  AI nodes call adapters. The engine and editor see only pins and parameters, so no model is hard-coded
  anywhere else.
- **Runtimes.** ONNX Runtime (CPU) is implemented and tested. It runs exported models from most frameworks,
  releases the GIL, and is light enough to live in the runtime process. TensorRT and CUDA/DirectML providers are not
  claimed.
- **Ultralytics and Anomalib** train and export in the *AI environment*, a Python environment of their own that Veyra
  runs worker scripts in (`veyra/engines`): PyTorch with its CUDA build is too large and version-sensitive to bundle, and
  Ultralytics' AGPL licence would extend to Veyra's build. Their exports run in production through Veyra's own adapters
  (one for every Ultralytics task, one for Anomalib) with the frameworks' pre- and post-processing reimplemented,
  including Ultralytics' probabilistic-IoU suppression of rotated boxes. Every export passes a **parity check** against
  the framework's own predictions before it is registered ([models.md](models.md)); the inventory of both frameworks,
  with what was verified for each model and on which machine, is data (`models/frameworks.py`,
  `models/verification.json`), so the studio never presents a model as supported because the framework has it.
- **The built-in anomaly model** is a per-pixel Gaussian reference model:
  1. It learns the mean and spread of good parts from a folder of good images.
  2. It is exported as an ONNX graph (normalise, z-score, smooth, max), so production inference goes through
     ONNX Runtime like any other model.
  3. It is trained in seconds on a CPU and needs no downloaded weights.

  It proves the registry, lifecycle and runtime path end to end, and it is genuinely useful for aligned parts.
  Deep models such as PatchCore or detectors are adapters to add. Their outputs already have typed results and
  coordinate handling.
- **Detection adapters.** Veyra's own detectors export rows of corner boxes; Ultralytics' adapter reads the export's own
  metadata (task, classes, input size, keypoint shape, end to end) and reproduces Ultralytics' results exactly on its
  exports (IoU 1.0 against Ultralytics' predictor on the same file).

## 10. Model registry and lifecycle

A model version is a record:

- identity: id, version, task, architecture;
- training: dataset reference, training configuration, preprocessing and postprocessing versions; for framework
  models also the framework and PyTorch versions, the device and the pretrained weights' SHA-256;
- export: the parity check (method, images, objects compared, largest deviation);
- runtime and runtime version;
- evaluation: metrics, validation report, benchmark;
- provenance: created at, created by, status, SHA-256.

The record lives in the SQLite store, and its artefact in `models/<id>/<version>/`. The status follows a checked
state machine:

```
DRAFT → TRAINING → VALIDATING → VALIDATED → APPROVED → DEPLOYED → ARCHIVED
                          └──► REJECTED (from VALIDATING or VALIDATED)
```

- Approval requires a validation report and a named approver, and is audited.
- A model reaches `DEPLOYED` only when a release that pins it is activated.
- Workflows pin models as `id@version`, and loading verifies the checksum.
- Nothing deploys an unvalidated model, and there is no path that skips approval.
- Validation shows facts side by side (metrics, confusion matrix, latency, memory) and never picks a "winner".

## 11. Versions, releases and updates

**Download is not deployment. Staging is not activation. Validation is not approval.**

`releases.py` manages *slots*. A slot holds one kind of artefact for one target, for example "the workflow of station
A", "model `flange-anomaly`" or "the platform". Each slot has at most one active version and remembers the previous
one. A candidate version moves through:

```
AVAILABLE → DOWNLOADED → VERIFIED → STAGED → VALIDATED → BENCHMARKED → APPROVED → ACTIVE
                  └────────── FAILED / REJECTED at any step ──────────┘            (previous: SUPERSEDED)
```

- **Feeds** list versions with a URL and SHA-256. A feed can be a local folder or HTTP(S). Signatures are
  planned; the checksum is verified today.
- **Downloading** writes to the slot's staging area, never over the active version.
- **Verifying** checks size and checksum, and the artefact's own manifest.
- **Staging** unpacks into `versions/<version>/`.
- **Validating and benchmarking** run the candidate beside the active version, for example over a labelled
  folder.
- **Approval** records who approved it and when.
- **Activation** atomically replaces a small state file (temp file plus rename) that points at the new version.
  The old version stays on disk, so **rollback** is the same pointer switch in the other direction.

Categories differ in what activation means. A workflow or model is activated in a running station at a frame
boundary. A platform update is only staged, and applied at the next restart by the launcher. Camera and PLC backends
will follow the model path. The state machine is shared; the activation step is per category.

**Workflow versions.** A released workflow is immutable: its file is stored under its checksum. Editing a released
workflow clones it into a draft, and a draft must be validated, approved and activated as a new revision. The active
production workflow is never edited in place.

**Configuration snapshot.** Every inspection record links to a snapshot recording:

- workflow revision and checksum;
- model versions;
- camera binding and calibration;
- I/O configuration;
- runtime and platform versions;
- device, line and lane.

Identical snapshots are stored once. That makes every production result reproducible.

## 12. Runtime process and concurrency

```
Veyra (GUI process)                          veyra runtime (child process, or headless on an edge PC)
  FluentQt UI ──► RuntimeClient (QProcess) ═══►  protocol reader thread
     ▲   document owner, draws snapshots  ◄═══   session: plan, cache, sources, batch loop, station
     │                                            engine thread pool (bounded)   acquisition threads
     └─ frames/previews only for what is shown    queues (bounded)               device I/O threads
```

- **The GUI never runs a vision tool, loads a model or blocks on a device.** It owns the workflow document,
  sends it to the runtime on change (debounced), and draws the results it receives.
- **Protocol.** Each message is a JSON header, optionally followed by a binary payload, over the child's
  standard streams. Pipes keep messages ordered, need no clean-up, and move a 6 MB preview in a few
  milliseconds. The payload carries only images the user is looking at, scaled to the display, at display rate.
- **Stale results are dropped.** Every run carries the request id and document revision, and the client drops
  results for an outdated revision.
- **Crash isolation.** A crash in a camera SDK or a model runtime ends the runtime process, not the interface.
  The interface reports it and restarts the runtime.
- **Headless operation.** The same runtime runs without the GUI (`veyra runtime`, `veyra run`). The
  production station lives in the runtime, so inspection continues if the engineering UI disconnects. For a
  station on another machine, the transport becomes a socket carrying the same messages. It is not built yet.

## 13. Production

- **The station** is the production configuration: cameras (role → source, calibration, trigger), workflow
  release, I/O backend and signal map, and storage policy.
- **The state machine** is `STOPPED → STARTING → READY → RUNNING ⇄ PAUSED`, with `→ STOPPING → STOPPED` and
  `→ FAULT` from any running state. Transitions are validated in one place. The UI can only request a
  transition, which may be refused with a reason.
- **Readiness** is checked before `STARTING`, and as a checklist people can run before they start
  (`station.checklist`):
  - every camera has an assignment, and every assignment runs an active or pinned *approved* release;
  - the workflows validate for production and every model they use is approved;
  - every camera opens and delivers frames (a real test, skipped only while production holds the cameras);
  - cameras that measure in physical units are calibrated;
  - the disk is above the cleanup level and the evidence folder is writable;
  - the I/O backend connects (the mock backend is flagged: no real reject gate is driven);
  - the current user may run production.

  Each failed check is reported, and none can be bypassed.
- **Per camera:** an acquisition thread feeds a bounded queue, and an inspection worker runs the plan. With
  several cameras that share a trigger, frames are grouped by trigger id (with a timeout) and fed to a
  workflow with several Frame inputs.
- **Results.** Each inspection produces an `InspectionEvent`: inspection id, timestamp, camera, frame, product,
  snapshot id, workflow revision, model versions, typed results, final status, latency, warnings and errors.
  Events update the counters (GOOD, NOT_GOOD, UNKNOWN, ERROR per line, lane, batch and shift), drive I/O
  through the signal map, and are stored according to the policy, for example images of NG and UNKNOWN parts.
- **ERROR and UNKNOWN are never GOOD.** A missing verdict is UNKNOWN, a failed node ERROR. Only an explicit,
  audited rule could ever map them otherwise, and none exists.
- **Human review.** Reviews are appended to a history (`reviews`); the machine's decision columns are written once and
  never updated. Reviewed images become training data only when someone adds them to a dataset (with the
  `candidate.approve` permission). Nothing flows into training on its own.
- **Faults are events.** A camera or station fault is sent to the studio, logged, and kept as an open system event
  until production starts again, so it appears on the overview and in System even if nobody was watching.

## 14. Devices

`IoBackend` is a small contract: connect, read inputs, write outputs, status, all in the runtime and never on the GUI
thread. The mock backend simulates I/O latency, a trigger input and failures for development and tests. **Modbus TCP**
(`ModbusTcpIo`) writes outputs as coils and reads inputs as discrete inputs at configured addresses, reconnects after a
failure and reports device exceptions in words; it is implemented from the specification and tested against an
independent implementation (pymodbus's server and client), not yet against a physical PLC, and the studio says so.
Modbus RTU, OPC UA and vendor protocols are not implemented. Signal maps (which output means NG) are station
configuration and never hard-coded.

## 15. Roles and sign-in

Roles are named sets of permissions (`security.py`), not a ladder:

| Role | Can |
|---|---|
| Operator | watch stations, run production, review rejected parts, read reports |
| Quality engineer | investigate history, review, place and release quality holds, export reports, read the audit log |
| Vision engineer | configure cameras and stations, design and test workflows, validate releases, label data |
| AI engineer | manage datasets, train, register and validate models |
| Administrator | everything, including approvals, activation, rollback, users, storage, backups and settings |

The runtime checks the permission of every command before it runs; the studio only hides what the role cannot do.
A user may act in a lesser role than the one assigned, never a greater one.

**Sign-in** is off by default (a single-user engineering workstation), and the studio then says that choosing a
user is not a login. An administrator turns it on with their own password. From then on the runtime answers nothing
but the handshake, the sign-in itself and the commands that stop work, until someone signs in. Passwords are scrypt
hashes with a random salt; there is no default password; five wrong attempts lock an account for five minutes; the
same answer is given for an unknown user and a wrong password; an idle studio signs out after the time the
administrator chose, while production keeps running. Every sign-in, failure, password change and policy change is in
the audit log. A lost last administrator password is reset on the station itself (`Veyra.exe users
reset-password NAME`): whoever can do that can read the data folder anyway, which is the real boundary of a local
installation. Directory services (LDAP, Active Directory) are not integrated; `security.allowed` remains the seam.
## 16. Extensibility

What is extensible today:

- A new vision tool is one decorated function or class in `tools/`, and it appears in the library, the
  inspector and the graph without editor changes.
- A new model is an adapter.
- A new source or I/O backend implements its contract.

All three are proved by the built-ins. A manifest-based module system (separately installed, versioned, switchable
back through `releases.py`) is designed but deliberately not built until a node actually ships separately.

## 17. Interface

A `FluentWindow` whose navigation follows the work people do, with the pages a role cannot use hidden (and a group
header hidden with its pages):

- **Overview**: production at a glance for a chosen period: inspections, NG rate over completed inspections (with its
  denominator), indeterminate and error counts kept apart, throughput and latency, compared with the period before;
  results over time; what needs attention (unreviewed rejects, an NG rate rising beyond chance, camera faults, disk,
  failed training, models awaiting approval, open system events); each camera; the newest NG evidence; what is in
  production; training; storage. Every figure opens the inspections behind it. A new installation sees three ways in
  instead (the sample line, an image of its own part, a camera).
- **Production**: *Stations* (cameras with their pipelines, live tiles, the pre-start checklist, camera assignments
  with the camera dialog), *Inspections* (below) and *Reports* (period summaries, CSV, JSON, PDF and Excel).
- **Engineering**: *Design* (image canvas and graph of one workflow, tool catalogue, inspector, node watch, results;
  templates for common inspections; in the graph, copy and paste with links, duplication, named groups drawn as
  frames, select all, arrange and fit), *Validate*, *Releases*. Groups live in the workflow's metadata and do not change
  its revision: they are layout, not behaviour.
- **AI**: *Datasets* (the labelling workspace: the task's tools on Pro's `ShapeEditor`, statuses, model suggestions
  kept apart until accepted, reviews, YOLO/COCO/DOTA exchange), *Training* (Veyra's trainers and the frameworks', with
  the AI environment's readiness and each model's support level), *Models* (by task and framework, with the models not
  integrated and why).
- **Storage**, **System** and **Settings** at the bottom.

**Inspections** is the investigation workspace: a Year → Month → Day → result calendar with counts (exact local days,
the same boundaries the list uses), the inspections that match it and the filters as painted rows with thumbnails
(paged by keyset; answers to outdated queries are dropped), and the evidence of the selected one. The evidence viewer
(`ui/evidence.py`) zooms and pans the original image with the overlay drawn as vector shapes, and has a draggable
divider to compare the original with the overlay, or the part with the nearest OK part in time from the same camera
and pipeline. *Why* lists every check with its value and limits (failing first); selecting a check highlights its
marks. *Traceability* names the pipeline version, models, run, snapshot and times. *Review and holds* shows the
append-only history, records reviews (a disagreement with the machine asks for a reason) and quality holds.

Outcomes look the same everywhere (`ui/status.py`): OK green with a check mark, NG red with a cross, Indeterminate
amber with a question mark, Error violet with a warning sign, always with the word, never colour alone. The image
canvas and the evidence viewer extend Pro's `ImageViewer`; the node graph is built in the application. The evidence
viewer's split comparison is a candidate for Pro once a second product needs it.
## 18. Testing and performance

- **Contracts:** transforms, immutability, pin-type compatibility, serialisation round trips and migrations,
  validation diagnostics.
- **Engine:** ordering, parallel and sequential equivalence, error propagation, caching and incremental re-runs,
  stepping, bounded concurrency.
- **Invariants:**
  - the coordinate test (section 4);
  - overlays never reach inference;
  - the same workflow on an image file and on the simulated camera gives equal downstream results for equal
    pixels;
  - repeated runs do not accumulate state, and batch runs do not leak state between images;
  - the workflow has no dependency on camera APIs.
- **Lifecycle:** registry transitions, and releases that never activate without approval, with rollback.
- **Runtime:** the real child process, driven through the protocol.
- **UI:** behaviour tests offscreen with pytest-qt; real screenshots captured from the application by a script.
- **Benchmarks**: engine overhead per node, workflow latency P50/P95/P99,
  sequential against parallel, throughput from a folder, queue behaviour under overload, and protocol and preview
  cost. Measured from a native filesystem, and published in [performance.md](performance.md) with the machine described.

## 19. Evidence, storage and retention

Evidence images are written atomically (a temporary file renamed when complete) under `images/YYYY/MM/DD/`, referenced
relative to the data folder, with their size recorded so storage is measured from the database, not by walking the
disk. `retention.py` holds the policy and the cleanup; `backups.py` the database backups; the runtime's
*housekeeper* applies both on its own (`runtime/storage.py`).

- **Categories and rules.** OK, NG and indeterminate/error images, inspection records, training artefacts, exports,
  logs and caches each keep their own period (days, weeks, calendar months or years) or are kept forever, which is
  the default for everything that matters. Datasets, models, releases, backups and the audit log are never removed by
  retention.
- **Protection.** Quality holds, anything newer than the minimum age, rejected parts nobody has reviewed, and under
  disk pressure the recent NG evidence are never removed. The selection runs inside the write transaction, so a hold
  placed after the preview is honoured.
- **Preview, then run.** Every cleanup is planned first: counts, space and what is protected and why. The run uses the
  confirmed scope and reference time.
- **Consistency.** Database references are detached and journalled (`cleanup_items`) batch by batch before any file is
  removed; an interrupted cleanup leaves only unreferenced files, which the next start removes. Removing an image keeps
  the inspection record, which says when and by which cleanup it was removed.
- **Disk pressure.** Below the cleanup level: expired data first, then (oldest first) only the categories the policy
  allows early, until the target. If that is not enough, an error event says what an administrator can do; protected
  data is not sacrificed.
- **Checks and backups.** *Check consistency* finds records whose image is missing (they can be marked so) and image
  files no record refers to (reported, never removed). Backups are SQLite online backups, integrity-checked, with their
  SHA-256; only the backup rule removes them; a restore happens at the next start and keeps the replaced database as a
  backup.

## 20. Exports, diagnostics and logs

- **Excel** (`exports.py`): the runtime streams the selected inspections into a workbook (Summary with definitions,
  breakdowns by station, camera, day and defect, every inspection, every check, every review) or an evidence package
  (the workbook and the stored images in a zip). Figures are computed from the exported rows only; rates use completed
  inspections; times are local with the offset stated; the file appears only when complete.
- **System health** (`runtime/system.py`): runtime, inference providers, database integrity, disk, retention,
  backups, station and cameras, missing evidence and open events, each with a level, a sentence and where to act. A
  diagnostic bundle (health, versions, events, configuration, today's logs) can be saved for support; it holds no images
  and no inspection records.
- **Logs** (`logs.py`): one file per process and day (`studio`, `runtime`, `trainer`) with local timestamps and the
  event's fields as JSON, and `crash.log` for native crashes and Qt's critical messages. Command failures are logged
  with their traceback; the person sees the message.

## 21. Packaging

`deploy/build.py` builds a standalone folder with Nuitka: one executable that is the studio, or the runtime with
`runtime`, or the trainer with `trainer`. It includes ONNX Runtime, OpenCV, ONNX, openpyxl, lxml and zxing-cpp, and the
AI environment's worker scripts as plain files (another Python runs them); PyTorch, Ultralytics, Anomalib, camera SDKs
and model weights stay out (`--pylon` adds pypylon, `--genicam` harvesters and genicam, each under its own licence).
With `--ai PYTHON` the smoke test also imports a pretrained Ultralytics model through that AI environment with the
packaged trainer and validates it with the packaged runtime. The build is verified by a smoke test
of the packaged executable: every studio page, the runtime driven over its protocol (sample preparation with model
training and ONNX export, health, camera backends, an Excel export), the command line running a workflow with ONNX
Runtime, and an empty crash log.

## 22. Decisions left open deliberately

- **Distributed stations** (socket transport, several runtimes per UI): the protocol is transport-neutral, but
  nothing is built until a deployment needs it.
- **Module packaging** for third-party nodes and camera SDKs (section 16).
- **GPU inference** in production (ONNX Runtime's CUDA, DirectML or OpenVINO providers): the adapters are provider-
  neutral, but nothing is claimed until it runs on such hardware.
- **Interactive segmentation (SAM) in labelling**: it needs a persistent AI-environment worker, since SAM has no ONNX
  path in Ultralytics.
- **Signed releases** (Ed25519 over the feed) once there is a publisher key to protect.
