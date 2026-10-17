# Release notes

Changes in the current releases of Veyra's software. Known limitations and the checks that are still
open are in [docs/verification.md](docs/verification.md).

## 0.3.0

### Added

- **Ultralytics and Anomalib** through an **AI environment**, a Python environment of their own that Veyra probes
  (frameworks, PyTorch, CUDA devices, memory), sets up on request, and runs worker scripts in: training and fine-tuning
  from frozen dataset versions, importing pretrained Ultralytics weights, ONNX export, and a **parity check** of every
  export against the framework's own predictions before it is registered, with full provenance (framework and PyTorch
  versions, device, weights' SHA-256).
- ONNX Runtime adapters for every Ultralytics task (detection, instance segmentation, pose, rotated boxes,
  classification, semantic segmentation; YOLOv3u to YOLO26, RT-DETR, YOLO-World, YOLOE) and for Anomalib exports.
- An **inventory** of both frameworks' models (30 Ultralytics entries, 30 Anomalib models, 9 not integrated with the
  reason) with support levels and a verification record per model ([models.md](docs/models.md)).
- Evaluation of segmentation (mask IoU), rotated boxes (rotated IoU) and pose (OKS) models, and mAP@0.5:0.95.
- **Labelling workspace** on FluentQt Pro's `ShapeEditor`: boxes, outlines, rotated boxes, keypoints with a schema,
  outline from a box (GrabCut), image classes, statuses, **model suggestions** kept apart until accepted (accepted shapes
  keep their origin), reviews (approved, needs a fix), batch actions, statistics; YOLO, COCO and DOTA import and export;
  pose and rotated-box datasets; sample datasets for every task.
- **Vision tools** (72 in all): colour spaces, contrast and CLAHE, flat-field correction, denoising, sharpening,
  gradients, colour thresholds, resize, rotate, flip, perspective correction, mask logic, hole filling, watershed
  separation, skeletons, feature, phase-correlation and ECC **alignment** with fixture-relative regions,
  **comparison with a golden reference** (difference and SSIM), line and circle finding, angles, point-to-line
  distances, shape measurement, lens **undistortion** with checkerboard calibration, **Data Matrix**, PDF417, Aztec and
  Micro QR with zxing-cpp, **OCR** (PP-OCR on ONNX Runtime, installed on request, with fielding), calculations, logic,
  condition and text checks, measurements of AI results (counts, areas, angles, keypoint distances).
- **Workflow templates** for ten common inspections; in the graph, copy and paste with links, duplication, named
  groups, select all, arrange and fit.
- **GenICam / GenTL cameras** of most makers through `harvesters` (an optional component).
- **Modbus TCP** station I/O with a connection test.

### Changed

- Rotating an image repeats its edge in the uncovered corners instead of filling them black.
- Anomalib exports are made with a batch of one image.

### Fixed

- The class buttons of the labelling page were given the page as their icon.
- Catalogue rows did not carry their task, which the Training page needed.
- The production-run, dataset, AI-environment, station I/O and template dialogs put their fields in a layout that
  does not exist, so they failed to open.
- WinCLIP had no class name for its prompts with Veyra's folder datasets; EfficientAD and DRÆM downloaded their
  auxiliary datasets into every job instead of once.
- On Windows, probing and training through an AI environment hung. Windows serialises the I/O of a synchronous pipe,
  so while the runtime, the trainer or a worker waited to read its standard input, loading a DLL (NumPy, PyTorch) or
  starting a child interpreter that inherited the pipe waited too. Standard input is now read only once data has
  arrived, the probe inherits none, and the probe waits up to 300 s for a first import of PyTorch.
- RT-DETR exports were decoded as YOLO heads; they are now read as RT-DETR queries (fractions of the input, no
  suppression) after the image is stretched to the input size, as Ultralytics does, and reproduce its predictions.
- The Training page kept the first suggested model name when another model was chosen, and listed the images of
  object datasets as unlabelled.
- Captions of the labelling page's thumbnails were cut short.
- The studio could crash on exit when its window outlived the application object.

## 0.2.0

### Added

- **Overview** as the landing page: production figures for a chosen period with their definitions, the comparison with
  the previous period, results over time, what needs attention (including an NG-rate alert that is raised only when
  the samples support it), live cameras, the newest NG evidence, what is in production, training and storage, with a
  drill-down from every figure. A new installation sees three ways in instead.
- **Inspections** rebuilt for investigation: Year → Month → Day → result calendar with counts, filters for hold and
  evidence state, painted rows with thumbnails, the evidence viewer (overlay, original, split, comparison with the
  nearest OK part), *why* per check with highlighting, traceability with links, append-only review and hold history,
  quality holds (single and for a whole selection).
- **Excel export** of any selection or report: summary with definitions, breakdowns, every inspection, every check,
  every review; optional evidence package with the images.
- **Storage**: usage and growth forecast, retention rules per category (days, weeks, months, years), protections,
  cleanup with preview and confirmation, a journal that keeps the database and files consistent across interruptions,
  disk-pressure cleanup, a consistency check, verified database backups with restore at the next start, and a
  housekeeper in the runtime.
- **Cameras**: OpenCV (UVC cameras, streams), Basler pylon and folder replay backends with discovery, a camera test with
  a preview, reconnection and controls with read-back; a camera dialog on the Stations page.
- **Pre-start checklist** on the Stations page.
- **Sign-in** (optional): scrypt-hashed passwords, lockout, idle sign-out, a runtime that answers nothing before sign-in,
  local recovery on the command line; a quality-engineer role; a user menu in the title bar.
- **System** page: health checks with actions, system events, the audit log, versions, a diagnostic bundle.
- Log files per process and day, a crash log, and camera and station faults recorded as events.
- A standalone Nuitka build verified by a smoke test of the packaged executable.
- A shared vocabulary of outcomes (OK, NG, Indeterminate, Error) with colour, icon and words, and rates over completed
  inspections everywhere.
- The window remembers its size and position, and asks before closing while production or training runs.

### Changed

- Navigation grouped by work: Production, Engineering and AI, with Storage, System and Settings at the bottom.
- Reports compute the NG rate and yield over completed inspections; indeterminate and error results are reported on
  their own.
- Evidence images are written atomically under `images/YYYY/MM/DD/` with relative references and their sizes.
- The pipeline title is recorded with every inspection.

### Fixed

- A command argument named `size` (thumbnails) was read as the payload length and desynchronised the runtime protocol,
  stopping the runtime; the payload length now has a reserved key, and an unreadable message is reported.
- Thumbnails of one page were dropped when another page had requested more pages of thumbnails.
- The inspection list counted rows twice when two queries overlapped; answers to outdated queries are now dropped.
- PyYAML, needed by the model catalogue in the runtime, was declared only for training.
