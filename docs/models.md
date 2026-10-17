# AI models

Which models Veyra can train, import and run, what each needs, and how far each was verified on this installation's
reference machine. The tables are generated from the application's model inventory and its verification record; everything else
on this page is written by hand.

## Three sources of models

| Source | Trains where | Runs in production with | Licence of the code |
|---|---|---|---|
| **Veyra's own** (the Gaussian reference anomaly model; the YOLOv1–v11, YOLOX, FCOS and centre-point detectors; compact CNN, ResNet and ViT classifiers) | Veyra's trainer process (PyTorch, an optional component) | ONNX Runtime | GPL-3.0-or-later |
| **Ultralytics 8.4** (YOLOv3u to YOLO26, RT-DETR, YOLO-World, YOLOE) | the AI environment | ONNX Runtime, through Veyra's own decoder of the Ultralytics export | AGPL-3.0 (an Enterprise licence is sold by Ultralytics) |
| **Anomalib 2.7** (PatchCore, PaDiM, EfficientAD, Dinomaly and 26 more) | the AI environment | ONNX Runtime | Apache-2.0 |

Production never imports PyTorch, Ultralytics or Anomalib. A model trained or imported through a framework becomes an
ONNX file with a manifest; Veyra's adapters run it with the
framework's own pre- and post-processing reimplemented: letterboxing, decoding of every Ultralytics head (boxes, mask
prototypes, keypoints, rotated boxes with Ultralytics' probabilistic-IoU suppression, classification, class maps), and
Anomalib's normalised scores, maps and masks.

## The AI environment

Ultralytics and Anomalib live in a Python environment of their own, next to Veyra rather than inside it: PyTorch and its
CUDA build are large and version-sensitive, and Ultralytics' AGPL licence would otherwise extend to Veyra's build.
**Settings → AI environment** shows the interpreter in use and what it can do (Python, the frameworks, PyTorch, CUDA
devices and memory); **Set up…** creates one in Veyra's data folder with Python's own `venv` and `pip` (the commands are
shown before they run and can be copied), **Choose Python…** uses an existing one. The interpreter is chosen in this
order: the setting, `VEYRA_AI_PYTHON`, the environment Veyra created, and, when Veyra runs from source, its own Python.

```bash
python -m venv ai-env
ai-env/bin/python -m pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu   # or cu126, cu128…
ai-env/bin/python -m pip install ultralytics anomalib onnx onnxscript onnxslim onnxruntime
```

Each job runs a worker script in that interpreter; the worker imports nothing from Veyra,
talks JSON lines on its standard streams, honours *Cancel* at the next batch and reports out-of-memory kills in words.
Ultralytics' settings (and its analytics switch, turned off) are kept in a folder of Veyra's data, never the user's.

## Training, fine-tuning and importing

1. **Datasets**: label the images with the task's tools and freeze a version (see [datasets.md](datasets.md)).
2. **Training → Setup**: choose the version and a family (Ultralytics and Anomalib families carry their support level:
   ✓ verified, ◐ exported but not confirmed), set the framework's own options (input size, epochs, batch, patience,
   optimiser, learning rate, frozen layers, Ultralytics' augmentation, device), start.
3. The trainer writes the version in the framework's layout (YOLO detection, segmentation, pose, OBB or
   classification folders; Anomalib's `Folder` layout with masks rasterised from defect outlines), linking the images
   to Veyra's object store instead of copying them. Unlabelled images, a model's pending suggestions and images marked
   *needs a fix* are left out.
4. The worker trains (fine-tuning the published weights, downloaded once into the data folder, or from scratch),
   exports to ONNX as one self-contained file, and records the framework's own predictions on validation images.
5. **The parity check**: Veyra opens the export with its adapter and compares. Ultralytics: every object the framework
   finds at confidence 0.05 must be found by Veyra with the same class at IoU ≥ 0.9 and nothing else (keypoints within
   1 px, classification probabilities within 0.05). Boxes are compared clipped to the image, as Veyra reports them
   (Ultralytics clips YOLO boxes but not RT-DETR's). Anomalib: the ONNX scores must match the PyTorch model's on the
   same resized images within 0.02 with the same decision. An export that does not reproduce the framework is **not
   registered**; when neither side finds an object, the check is recorded as inconclusive.
6. The model is registered as a **DRAFT** with its provenance: dataset version, the full configuration, framework and
   PyTorch versions, device, the pretrained weights' file and SHA-256, the framework's validation metrics and the parity
   record. It reaches production only after validation, approval and an activated release; a newer checkpoint never
   replaces a deployed model by itself, and rolling back is a release operation.

**Import pretrained** (Ultralytics only) registers the published weights as they are, with the same export and parity
check, for inference on classes they were trained on (COCO, ImageNet, DOTA…) or as a starting point.

## Evaluation in Veyra

**Models → Validate** runs the registered ONNX model in Veyra's runtime on the test split of a frozen version and
reports what it did there, whatever framework trained it: AUROC and the confusion matrix (anomaly), accuracy and
per-class recall (classification), and for the object tasks precision, recall, AP per class, mAP@0.5 and mAP@0.5:0.95
with the task's own matching: box IoU (detection), mask IoU of the outlines (segmentation), rotated-box IoU (OBB) and
object keypoint similarity (pose; COCO's spreads for 17 person keypoints, one spread otherwise). Latency is measured per
image. The report states facts; approval is a person's decision.

## Hardware

Training and export run on the CPU and on CUDA GPUs the AI environment's PyTorch sees (`auto` picks the first). The
probe says when PyTorch is built for CUDA but sees no device. Production inference runs on ONNX Runtime's CPU provider.
**Nothing here was verified on a GPU**: the reference machine has none.

## Support levels

`framework` (the framework has it) → `registered` (Veyra lists it with its requirements) → `trainable` (a job trained
it through Veyra) → `exported` (its ONNX export opened) → `runtime` (the export reproduced the framework's predictions
in Veyra) → `hardware` (verified on target hardware: none yet). The verification script trains every model a few epochs
on Veyra's synthetic sample set of its task, which proves the path, not the accuracy a model reaches on real parts. The record of the run is summarised in
[verification.md](verification.md).

<!-- generated:models -->
**Ultralytics: 27 of 30 verified; Anomalib: 28 of 30 verified.**

Verified with anomalib 2.7.0, lightning 2.6.6, onnx 1.23.2, onnxruntime 1.31.0, torch 2.14.1+cpu, ultralytics 8.4.178 on 12th Gen Intel(R) Core(TM) i5-1235U (12 threads, 7.6 GB), Linux-6.18.54.1-microsoft-standard-WSL2-x86_64-with-glibc2.39, CPU only. Each model trained a few epochs (memory-bank models: one pass) on Veyra's synthetic sample set of its task (32 images), on CPU (x86_64, 12 threads). The metrics show that training and export work; they do not say how well a model does on real parts.

### Ultralytics

| Model | Task | Sizes | Pretrained on | Level | Verification run |
|---|---|---|---|---|---|
| YOLO26 | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 31 s; parity on 8 images, 2 objects, max. deviation 0.0e+00; mAP50 0.30 |
| YOLO26 | segmentation | n, s, m, l, x | coco | ✓ verified | 2 epochs, 50 s; parity on 8 images, 27 objects, max. deviation 0.0e+00; mAP50 0.28 |
| YOLO26 | pose | n, s, m, l, x | coco-pose | ✓ verified | 2 epochs, 34 s; parity on 4 images, 1 object, max. deviation 0.0e+00; mAP50 0.00 |
| YOLO26 | obb | n, s, m, l, x | dota | ✓ verified | 2 epochs, 88 s; parity on 8 images, 14 objects, max. deviation 8.9e-07; mAP50 0.04 |
| YOLO26 | classification | n, s, m, l, x | imagenet | ✓ verified | 2 epochs, 14 s; parity on 8 images; top1 0.25 |
| YOLO26 | semantic | n, s, m, l, x | cityscapes | not verified | semantic segmentation needs class masks, which the sample sets do not provide |
| YOLO11 | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 20 s; parity on 8 images, 23 objects, max. deviation 0.0e+00; mAP50 0.11 |
| YOLO11 | segmentation | n, s, m, l, x | coco | ✓ verified | 2 epochs, 44 s; parity on 8 images, 27 objects, max. deviation 0.0e+00; mAP50 0.58 |
| YOLO11 | pose | n, s, m, l, x | coco-pose | ✓ verified | 2 epochs, 30 s; parity on 4 images, 2 objects, max. deviation 0.0e+00; mAP50 0.00 |
| YOLO11 | obb | n, s, m, l, x | dota | ✓ verified | 2 epochs, 112 s; parity on 8 images, 42 objects, max. deviation 3.1e-06; mAP50 0.10 |
| YOLO11 | classification | n, s, m, l, x | imagenet | ✓ verified | 2 epochs, 16 s; parity on 8 images; top1 0.25 |
| YOLO12 | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 71 s; parity on 8 images, 295 objects, max. deviation 0.0e+00; mAP50 0.26 |
| YOLOv10 | detection | n, s, m, b, l, x | coco | ✓ verified | 2 epochs, 58 s; parity on 8 images, 108 objects, max. deviation 0.0e+00; mAP50 0.13 |
| YOLOv9 | detection | t, s, m, c, e | coco | ✓ verified | 2 epochs, 47 s; parity on 8 images, 748 objects, max. deviation 0.0e+00; mAP50 0.08 |
| YOLOv9 | segmentation | c, e | coco | ✓ verified | 2 epochs, 114 s; parity on 8 images, 10 objects, max. deviation 0.0e+00; mAP50 0.00 |
| YOLOv8 | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 79 s; parity on 8 images, 33 objects, max. deviation 0.0e+00; mAP50 0.12 |
| YOLOv8 Open Images | detection | n, s, m, l, x | oiv7 | ✓ verified | 2 epochs, 61 s; parity on 8 images, 180 objects, max. deviation 0.0e+00; mAP50 0.03 |
| YOLOv8 | segmentation | n, s, m, l, x | coco | ✓ verified | 2 epochs, 59 s; parity on 8 images, 110 objects, max. deviation 0.0e+00; mAP50 0.22 |
| YOLOv8 | pose | n, s, m, l, x | coco-pose | ✓ verified | 2 epochs, 30 s; parity on 4 images, 2 objects, max. deviation 0.0e+00; mAP50 0.00 |
| YOLOv8 | obb | n, s, m, l, x | dota | ✓ verified | 2 epochs, 63 s; parity on 8 images, 3 objects, max. deviation 6.4e-07; mAP50 0.13 |
| YOLOv8 | classification | n, s, m, l, x | imagenet | ✓ verified | 2 epochs, 11 s; parity on 8 images, max. deviation 4.5e-03; top1 0.62 |
| YOLOv6 (Ultralytics) | detection | n, s, m, l, x | none (from scratch) | ◐ exported, parity inconclusive | 40 epochs, 226 s; parity inconclusive: no object found on either side; mAP50 0.00 |
| YOLOv5u | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 31 s; parity on 8 images, 3 objects, max. deviation 0.0e+00; mAP50 0.19 |
| YOLOv5u P6 | detection | n, s, m, l, x | coco | ✓ verified | 2 epochs, 169 s; parity on 8 images, 279 objects, max. deviation 0.0e+00; mAP50 0.17 |
| YOLOv3u | detection | one | coco | ◐ exported, parity inconclusive | 15 epochs, 1896 s; parity inconclusive: no object found on either side; mAP50 0.02 |
| YOLOv3u tiny | detection | one | coco | ✓ verified | 2 epochs, 56 s; parity on 8 images, 77 objects, max. deviation 0.0e+00; mAP50 0.52 |
| RT-DETR | detection | l, x | coco | ✓ verified | 2 epochs, 143 s; parity on 8 images, 2400 objects, max. deviation 2.2e-05; mAP50 0.14 |
| YOLO-World | detection | s, m, l, x | coco | ✓ verified | 12 epochs, 207 s; parity on 8 images, 8 objects, max. deviation 0.0e+00; mAP50 0.89 |
| YOLOE-26 | segmentation | n, s, m, l, x | coco | ✓ verified | 2 epochs, 80 s; parity on 8 images, 5 objects, max. deviation 0.0e+00; mAP50 0.09 |
| YOLOE-11 | segmentation | s, m, l | coco | ✓ verified | 2 epochs, 172 s; parity on 8 images, 1 object, max. deviation 0.0e+00; mAP50 0.36 |

### Anomalib

| Model | Approach | Learns from | Level | Verification run |
|---|---|---|---|---|
| PatchCore (2022) | memory bank | one-class | ✓ verified | 1 epoch, 42 s; parity on 8 images, max. deviation 2.7e-06; image_AUROC 1.00 |
| PaDiM (2020) | feature distribution | one-class | ✓ verified | 1 epoch, 22 s; parity on 8 images, max. deviation 1.8e-07; image_AUROC 1.00 |
| MH-PatchCore (2025) | memory bank | one-class | ✓ verified | 1 epoch, 67 s; parity on 8 images, max. deviation 7.2e-07; image_AUROC 1.00 |
| CFA (2022) | memory bank | one-class | ✓ verified | 2 epochs, 165 s; parity on 8 images, max. deviation 4.7e-05; image_AUROC 0.70 |
| DFKDE (2022) | feature density | one-class | ✓ verified | 1 epoch, 23 s; parity on 8 images, max. deviation 7.9e-06; image_AUROC 1.00 |
| DFM (2019) | feature density | one-class | ✓ verified | 1 epoch, 46 s; parity on 8 images, max. deviation 0.0e+00; image_AUROC 1.00 |
| FRE (2023) | reconstruction | one-class | ✓ verified | 2 epochs, 34 s; parity on 8 images, max. deviation 2.6e-05; image_AUROC 0.20 |
| C-Flow (2021) | normalising flow | one-class | ✓ verified | 2 epochs, 374 s; parity on 8 images, max. deviation 6.0e-06; image_AUROC 1.00 |
| CS-Flow (2021) | normalising flow | one-class | ✓ verified | 2 epochs, 419 s; parity on 8 images, max. deviation 0.0e+00; image_AUROC 0.60 |
| FastFlow (2021) | normalising flow | one-class | ✓ verified | 2 epochs, 70 s; parity on 8 images, max. deviation 3.5e-04; image_AUROC 0.40 |
| U-Flow (2022) | normalising flow | one-class | ✓ verified | 2 epochs, 1342 s; parity on 8 images, max. deviation 0.0e+00; image_AUROC 0.25 |
| PatchFlow (2025) | normalising flow | one-class | ✓ verified | 2 epochs, 223 s; parity on 8 images, max. deviation 0.0e+00; image_AUROC 0.60 |
| STFPM (2021) | knowledge distillation | one-class | ✓ verified | 2 epochs, 23 s; parity on 8 images, max. deviation 5.5e-06; image_AUROC 0.65 |
| Reverse Distillation (2022) | knowledge distillation | one-class | ✓ verified | 2 epochs, 89 s; parity on 8 images, max. deviation 1.5e-05; image_AUROC 0.60 |
| EfficientAD (2023) | knowledge distillation | one-class | ✓ verified | 2 epochs, 2750 s; parity on 8 images, max. deviation 5.2e-06; image_AUROC 0.80 |
| UniNet (2025) | knowledge distillation | one-class | ✓ verified | 2 epochs, 247 s; parity on 8 images, max. deviation 0.0e+00; image_AUROC 0.55 |
| DRÆM (2021) | reconstruction | one-class | ✓ verified | 2 epochs, 1270 s; parity on 8 images, max. deviation 1.4e-04; image_AUROC 0.90 |
| DSR (2022) | reconstruction | one-class | ✓ verified | 2 epochs, 258 s; parity on 8 images, max. deviation 3.9e-05; image_AUROC 0.10 |
| GANomaly (2018) | reconstruction | one-class | ✓ verified | 2 epochs, 139 s; parity on 8 images, max. deviation 3.3e-05; image_AUROC 0.45 |
| GLASS (2024) | synthesis | one-class | ✓ verified | 2 epochs, 203 s; parity on 8 images, max. deviation 2.4e-05; image_AUROC 0.55 |
| SuperSimpleNet (2024) | synthesis | one-class | ✓ verified | 2 epochs, 148 s; parity on 8 images, max. deviation 3.7e-05; image_AUROC 0.20 |
| Dinomaly (2025) | reconstruction (ViT) | one-class | ✓ verified | 2 epochs, 300 s; parity on 8 images, max. deviation 1.7e-05; image_AUROC 0.90 |
| INP-Former (2025) | reconstruction (ViT) | one-class | ✓ verified | 2 epochs, 239 s; parity on 8 images, max. deviation 8.9e-06; image_AUROC 1.00 |
| FoundAD (2025) | foundation features | one-class | ✓ verified | 2 epochs, 292 s; parity on 8 images, max. deviation 7.2e-05; image_AUROC 0.70 |
| GeneralAD (2024) | foundation features | one-class | ✓ verified | 2 epochs, 1242 s; parity on 8 images, max. deviation 9.3e-05; image_AUROC 0.30 |
| AnomalyDINO (2025) | memory bank (ViT) | few-shot | ✓ verified | 1 epoch, 70 s; parity on 8 images, max. deviation 3.1e-06; image_AUROC 1.00 |
| RAD (2025) | memory bank (ViT) | few-shot | ✓ verified | 1 epoch, 237 s; parity on 8 images, max. deviation 8.3e-07; image_AUROC 1.00 |
| SuperADD (2025) | memory bank (ViT) | few-shot | not verified | killed for lack of memory on the verification machine |
| WinCLIP (2023) | vision-language (zero-shot) | zero-shot | ✓ verified | no training (zero-shot), 710 s; parity on 8 images, max. deviation 1.2e-05; image_AUROC 1.00 |
| AnomalyVFM (2025) | vision foundation model (zero-shot) | zero-shot | not verified | Anomalib does not export AnomalyVFM to ONNX (its export returned no file), so it cannot run in Veyra's runtime |

### Not integrated

| Model | Framework | Why |
|---|---|---|
| SAM, SAM 2, SAM 2.1, MobileSAM | Ultralytics | Promptable segmentation (click or box → mask). Ultralytics does not export these to ONNX, so they cannot run in the inspection runtime; they are suited to assisted labelling, which Veyra does not run in the AI environment yet. |
| FastSAM | Ultralytics | Segments everything with a YOLOv8 segmentation network and then filters by prompt; the prompt step is not part of the export. |
| YOLO-NAS | Ultralytics | Needs Deci's super-gradients package, and its pretrained weights are licensed for non-commercial use only. |
| YOLO26 depth | Ultralytics | Monocular relative depth; exportable, but no inspection tool consumes relative depth yet. |
| AI-VAD | Anomalib | Video anomaly detection (object attributes across frames); Veyra inspects single images. |
| FUVAS | Anomalib | Video anomaly detection; Veyra inspects single images. |
| CFM | Anomalib | Multimodal (RGB with 3-D point clouds, as in MVTec 3D-AD); Veyra's cameras deliver 2-D images. |
| L2BT | Anomalib | Multimodal (RGB with 3-D point clouds); Veyra's cameras deliver 2-D images. |
| VLM-AD | Anomalib | Asks a vision-language model (Ollama, OpenAI or Hugging Face) whether an image is anomalous; it has no export and depends on an external model service per image. |
<!-- /generated:models -->

## Licences to check before shipping a model

- **Ultralytics** code and weights are AGPL-3.0: a product that distributes or serves them must meet the AGPL or buy
  Ultralytics' Enterprise licence. Veyra does not bundle them; the AI environment is installed by its user.
- **Pretrained weights** inherit their training data's terms: COCO (CC BY 4.0 annotations), ImageNet (non-commercial
  research), DOTA (academic use only), Cityscapes (non-commercial). Fine-tuning does not remove them.
- **Anomalib** is Apache-2.0; its backbones come from torchvision, timm and Hugging Face under their own licences (DINOv3
  weights are gated and need accepted terms).
