# Verification

What has been verified, on what, and what is still open. Every claim below names the machine or device it ran on.
Nothing here is extrapolated: a capability tested on a CPU, an emulator or a simulated device is described as such,
not as verified on industrial hardware.

## The reference machine

Intel Core i5-1235U laptop (10 cores, 12 threads, Iris Xe graphics, **no NVIDIA GPU**), Windows 11 with WSL2
(Linux 6.18), about 7.6 GB of memory available to WSL2. Every result on this page comes from this machine.

## Automated tests

The test suite is part of the private source repository; it is not published here.

| Run | Where | Result |
|---|---|---|
| Whole suite with the AI environment (`VEYRA_AI_PYTHON`, Ultralytics 8.4.178, Anomalib 2.7.0, PyTorch 2.14.1 CPU) and the OCR models (`VEYRA_OCR_MODELS`) | Linux (WSL2), offscreen Qt, CPU | **197 passed, 2 skipped** in 5 min 23 s (2026-10-17). The 2 skipped tests are Windows-only. |
| The two Windows-only tests (standard-input pipe must not stall a DLL load in the runtime and in a worker) | native Windows Python | **2 passed** (2026-10-17) |
| `ruff check --select F,E9` | the whole application | passes (2026-10-17) |
| Command line: `--help`, `simulate` (44 labelled sample parts) | Linux | works (2026-10-17) |

The suite covers the workflow engine, the vision tools against generated ground truth, the detectors and their
reference decoders, the runtime protocol, storage, retention and backups, reports and exports, roles and sign-in,
cameras (simulator, replay, OpenCV, pylon's emulator, the simulated GenTL producer), Modbus TCP against an independent
implementation (pymodbus), every studio page, and, with the AI environment, real Ultralytics and Anomalib training,
export and parity checks.

## AI models (recorded 2026-10-16, CPU only)

The model verification script trained every framework model a few epochs (memory-bank models: one pass) on Veyra's
synthetic sample set of its task, exported it to ONNX and checked that Veyra's runtime reproduces the framework's own
predictions. The per-model table is in [models.md](models.md).

| Framework | Verified end to end | Not verified |
|---|---|---|
| Ultralytics 8.4 | 27 of 30 | YOLO26 semantic segmentation (the sample sets have no class masks); YOLOv3u and YOLOv6 (exported, but parity inconclusive: neither side found an object after short training) |
| Anomalib 2.7 | 28 of 30 | SuperADD (killed for lack of memory); AnomalyVFM (Anomalib does not export it to ONNX) |
| **Total** | **55 of 60** | 5 |

These runs prove that training, export and inference work through Veyra on a CPU. They do not measure the accuracy a
model reaches on real parts, and none of them ran on a GPU. The run was not repeated for this release; the code paths
it exercises are covered by the AI-environment tests above.

## Devices

| Device | Verified on | Status |
|---|---|---|
| Simulated camera, image-folder replay | automated tests | verified |
| USB camera (OpenCV) | the laptop's built-in UVC camera, Windows 11, DirectShow, 640 × 480 at 15 frames/s | verified on that camera |
| Basler cameras (pylon) | pylon's camera emulator | **emulator only**; no physical Basler camera yet |
| GenICam / GenTL cameras | the simulated devices of the `genicam` package's GenTL producer; Basler pylon 7's GenTL producers load on Windows | **simulated only**; no physical camera yet |
| Cognex, Keyence | not supported through their SDKs; their GigE Vision cameras would go through GenICam | not verified |
| Modbus TCP station I/O | pymodbus as an independent server and client | **software only**; no physical PLC yet |
| Modbus RTU, OPC UA | not implemented | — |
| GPU (CUDA) training in the AI environment | — | **not verified**: the reference machine has no NVIDIA GPU |
| GPU inference in production | not implemented (ONNX Runtime's CPU provider) | — |

## Packaging (recorded 2026-10-16)

The packaging script built the 0.3.0 standalone Windows folder (357 MB, 28 minutes with 8 jobs) and verified
it by running it: every studio page, the packaged runtime over its protocol, the command line, an empty crash log, and
a pretrained YOLO imported through the Windows AI environment by the packaged trainer and validated by the packaged
runtime. The build was not repeated for this release.

## Hardware checks still open

These need hardware the reference machine does not have. They are listed so that they can be run and recorded; none
of them is claimed.

1. **GPU training.** In an AI environment with a CUDA build of PyTorch on an NVIDIA GPU: Settings → AI environment
   shows the device; train YOLO26n detection and PatchCore on a sample dataset with device `cuda:0`; the export passes
   the parity check; record the time per epoch against the CPU figures in [models.md](models.md). Optionally run
   the model verification script with that environment and keep its record apart from the CPU one.
2. **A physical Basler camera** (the optional Basler component): discovery, Test (frames, frame rate, time to first
   frame), exposure and gain applied and read back, software trigger, hardware trigger on `Line1`, reconnection after
   the cable is pulled and plugged back in, a production run with the camera's pipeline.
3. **A physical GenICam camera** through its vendor's GenTL producer (the optional GenICam component): the same checks as
   for Basler; a Cognex or Keyence GigE Vision camera can be checked this way if one is available.
4. **A physical PLC over Modbus TCP**: the connection test (it reads the configured discrete inputs), the result
   signals written to the configured coils during a production run, and behaviour when the PLC restarts or the cable
   is pulled (the failed write is reported and the next request reconnects).
5. **Line conditions**: trigger-to-result latency and jitter with a real camera, dropped frames under sustained
   overload, and a soak run of several hours with memory staying flat.

Record each check with the device's model, firmware, driver or SDK version and the date, and update the tables above
and the backend's *verified against* text in the camera dialog only after it has passed.
