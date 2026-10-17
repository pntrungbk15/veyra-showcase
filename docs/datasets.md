# Datasets and labelling

## What a dataset is

A dataset has a **task**, a class list and samples. Each sample is one image stored once, as its original file bytes,
in a content-addressed object store (`datasets/objects/`), whichever dataset uses it and wherever it came from; a
dataset keeps working when the folder it was imported from moves or a production image is removed by retention. The
draft changes freely; **freezing** writes an immutable, checksummed version that training and validation name
(`dataset@version`), so every model can say exactly what it learned from. Splits (train, val, test) are assigned from a
hash of the image and a seed: deterministic, and stable when samples are added; a split set by hand is kept.

| Task | Annotations | Trains |
|---|---|---|
| Classification | a class per image | Veyra's classifiers, Ultralytics classification |
| Object detection | boxes | Veyra's detectors, Ultralytics YOLO and RT-DETR |
| Instance segmentation | outlines (polygons) | Ultralytics segmentation (YOLO-seg, YOLOE) |
| Rotated objects | rotated boxes (centre, size, angle) | Ultralytics OBB |
| Keypoints (pose) | the dataset's named keypoints per object, with visibility (0 not labelled, 1 hidden, 2 visible), and optionally the object's box | Ultralytics pose |
| Anomaly detection | good parts marked as references; defect classes with outlines or boxes | Veyra's Gaussian model, every Anomalib model |

A pose dataset has a **keypoint schema**: the names in order, the skeleton (which keypoints are joined) and, for
horizontal-flip augmentation, which keypoint each one becomes when mirrored.

## Labelling

**Datasets** is the labelling workspace: the datasets on the left with their status and class counts; the images in the
middle, filtered by status, class and split; the editor on the right (FluentQt Pro's `ShapeEditor`) with the tools of
the task. Shapes are kept in original-image pixels whatever the size of the copy shown. Every change is saved as it is
made; undo and redo work within an image.

| Key | Action |
|---|---|
| 1–9 | draw new shapes as that class, relabel the selected shapes (classification: the image's class; Shift+number: the image's class in other tasks) |
| B, P, R, K, G | box, outline, rotated box, keypoints, outline from a box (GrabCut) |
| V | select; drag to move, handles to resize or reshape, the round handle to turn a rotated box (Shift: 15° steps) |
| Delete, Ctrl+D, arrows, [ ] | remove, duplicate, nudge, rotate |
| Space (keypoints) | skip the next keypoint as not visible |
| A, X, Enter | accept the model's suggestions, discard them, accept only the selected suggestion |
| Ctrl+Enter | approve the image's annotations |
| PgUp / PgDn | previous / next image |

**Outline from a box** (GrabCut) turns a box drawn a little larger than a part into an outline, from colour models of
the box's inside and outside, to be corrected by hand.

**Status.** Every image is *unlabelled*, *suggestions to review*, *labelled*, *reviewed* (approved by a reviewer) or
*needs a fix* (a reviewer said what, and freezing is refused until it is fixed). An object-task image confirmed to
contain no object (class `good`) is labelled. Selected images can be accepted, reviewed, moved to a split or removed in
one step.

## Suggestions from a model

**Suggest with a model…** runs a registered model (any task that fits: a detection, segmentation, pose or rotated-box
model for the object tasks, a classifier or an anomaly model for image classes) on the selected images or on every
unlabelled one, in the runtime, and stores its results **beside** the annotations as suggestions. They are drawn
hatched with their score, never exported or trained on, and become annotations only when a person accepts them (all,
or one at a time). An accepted shape keeps where it came from (`origin: model`, the model version, its score, who
accepted it and when), so machine-made labels stay distinguishable in every version.

## Checking and freezing

**Check** lists what is wrong before a version is frozen: shapes outside the image, keypoint sets that do not match the
schema, classes without samples, strongly imbalanced classes, images labelled with a defect but without the task's
shapes, suggestions still waiting, images that need a fix, too few good references for anomaly training, missing
splits. Errors block freezing; warnings and notes do not.

## Exchange with other tools

| Format | Import | Export | Holds |
|---|---|---|---|
| Folders | `good/`, `ng/<defect>/` or `<class>/` | — | image classes |
| YOLO (`data.yaml`, `images/`, `labels/`) | ✓ | ✓ | boxes, outlines, keypoints, rotated boxes (the dataset's task) |
| COCO (`annotations.json`) | ✓ | ✓ | boxes, outlines, keypoints with the schema |
| DOTA (`images/`, `labelTxt/`) | ✓ | ✓ | rotated boxes |

The AI Annotation Workbench, CVAT, Label Studio and Roboflow read and write these, so heavier labelling (video,
interactive segmentation) can be done there and the result imported. Veyra keeps boxes by their pixel edges and points
at pixel centres; the formats use continuous coordinates, and a round trip is exact (automated tests).

## The sample data

Every image the samples, tests, benchmarks and screenshots use is **generated by Veyra**; no dataset is
redistributed.

- **The flange**: a machined flange on a conveyor belt with a centre bore of 9 mm at
  0.05 mm per pixel, six bolt holes, a green seal and a paper tag with the serial as a QR code and as text. Good parts
  vary in position, brightness and noise; defective parts carry one known defect:

  | Defect | What is drawn | The check meant to catch it |
  |---|---|---|
  | `scratch` | a thin bright scratch on the metal face | Surface (anomaly model) |
  | `stain` | a dark irregular stain | Surface (anomaly model) |
  | `missing_hole` | one bolt hole not drilled | Hole count (blob analysis) |
  | `oversize_bore` | a 10 mm bore instead of 9 mm | Bore tolerance |
  | `wrong_seal` | an orange seal instead of green | Seal colour (ΔE) |

- **Task sets** (*Add a sample dataset* in Datasets): a tray of bolts, washers and brackets at any
  position and angle, with boxes, outlines traced from each part's mask, minimum-area rotated boxes and, for brackets,
  four keypoints; crops of single parts for classification; crops of the flange face with scratches and stains (and
  their outlines) for anomaly detection.

They exercise every part of the platform without rights questions, but they are easier than real parts (fixed
lighting, no reflections or dust): the numbers they produce show that the system works, not how well a model would do
on a real line.

## Public datasets

When a public dataset is used, record its name and version, source URL, licence, intended use, local path and the
archive's SHA-256, and never commit its images to a repository whose licence differs. For example, MVTec AD
(https://www.mvtec.com/company/research/datasets/mvtec-ad) is CC BY-NC-SA 4.0: non-commercial use only. It is not
redistributed with Veyra and nothing depends on it; Anomalib's own data modules can read it in the AI environment.

## Production images as training data

Stored production images become training data only on purpose: a person reviews the inspection, someone with the
`candidate.approve` permission adds the reviewed image to a dataset (it keeps its production origin and review), and a
new model version from it goes through the same validation and approval as any other. Each step is in the audit log.
