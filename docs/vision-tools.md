# Vision tools

The tools a workflow is built from: classical vision and AI side by side, with typed inputs and outputs so that the
editor refuses a connection that cannot work, and with every result placed on the original image whatever crop, turn,
resize or alignment came before. The catalogue at the end is generated from the tool registry.

## How the tools fit together

- **Images keep their way back to the frame.** Every tool that changes geometry (region, resize, rotate, flip,
  perspective correction, alignment) composes the transform from its output to the original frame, so a blob found in
  a rotated, aligned crop is reported where it is on the camera image, and measurements are in the camera's calibrated
  unit.
- **Measuring and judging are separate.** A measurement is a fact with its unit (*Measure blob*, *Find line*, *Measure
  shape*, *Measure objects*, *Calculate*…); a check turns it into a verdict against limits (*Tolerance check*, *Range
  check*, *Condition check*, *Text check*…); the *Decision* fuses verdicts into OK, NG, Indeterminate or Error. A tool
  that fails makes the inspection Error, a skipped check makes it Indeterminate: neither can pass.
- **No pixel is measured in millimetres without a calibration.** The calibration belongs to the camera (scale, and the
  lens when it was calibrated with a checkerboard); a tolerance check that expects millimetres fails as an error on an
  uncalibrated camera instead of comparing pixels.
- **AI and classical vision mix freely.** An object detector's boxes, a segmentation model's outlines, a pose model's
  keypoints or a rotated-box model's angle become measurements like any blob (*Measure objects*), and an anomaly
  model can be given a region drawn on an aligned reference so it always sees the same part of the part.

## Alignment and comparison with a good part

Three ways to find a taught reference, from robust to precise, each giving an *aligned image* in the reference's pixels
and an *Alignment* (the motion, a match score, the number of agreeing features):

| Tool | Finds | Copes with | Needs |
|---|---|---|---|
| **Align to reference** | ORB, AKAZE or SIFT features matched to the reference, motion estimated with RANSAC | large shifts, any rotation, some scale, perspective (translation, rigid, similarity, affine or perspective motion) | texture or corners |
| **Shift alignment** | phase correlation over the reference region | shifts, sub-pixel | the part to look the same |
| **Refine alignment** | ECC (enhanced correlation coefficient) | a few pixels or degrees, sub-pixel | a rough alignment first |

A region with *Coordinates: this image* after an alignment is drawn on the reference and follows the part (a fixture).
**Compare with reference** then subtracts the aligned part from the good reference after matching brightness and
contrast, ignores differences within a few pixels of the reference's edges (misregistration) and specks below the
smallest defect, or compares local structure (SSIM); the differing pixels come out as a mask with their area. Tested
against a part moved by (32, 22) px and turned 12°: the motion is recovered within 2 px and 0.7°, and a 9 px spot is the
only difference found, placed within 1.5 px of where it was drawn (automated tests).

## Geometry and measurement

*Find line* fits a straight edge along parallel rays with sub-pixel edge points and outlier rejection, and reports its
angle and straightness; *Angle between lines* and *Point to line distance* build datums; *Caliper* measures widths
across a band; *Circle fit* and *Find circles* (Hough) find bores and holes; *Measure shape* gives the true length and
width of a blob's minimum rotated rectangle at any orientation, its angle, elongation, solidity, Feret diameters,
ellipse axes and deepest notch. A 160 × 40 px bar turned 30° measures 16.0 × 4.0 mm at 0.1 mm/px within 0.3 mm, and an
edge at 7° is found within 0.2° with a straightness below a pixel (automated tests).

**Calibration.** The scale (mm per pixel) can be entered, measured from a known length, or measured from a
checkerboard lying in the inspection plane, which also reports the tilt and the spread of the scale across the target
so a tilted setup is noticed. Twelve to twenty checkerboard images that cover the field of view calibrate the lens
(OpenCV's camera model, with the RMS reprojection error); *Remove lens distortion* then straightens the camera image
before regions and measurements. The scale holds in the plane where it was measured.

## Codes and text

*Read code* reads Data Matrix, QR and Micro QR, PDF417, Aztec and the linear symbologies (EAN, UPC, Code 128, Code 39,
ITF, GS1 DataBar) with zxing-cpp, and QR codes and barcodes with OpenCV as well (each finds codes the other misses).
*Read text* reads printed text with PP-OCR models on ONNX Runtime (installed on request, about 14 MB, Apache-2.0): in
*single line* mode the region frames one marking; in *lines* mode every line of the region is found. *Characters*
restricts the alphabet; *Field* (for example `AAA999999`) resolves look-alikes by position: O/0, I/1, S/5, B/8, Z/2,
G/6 are swapped only where the field asks for the other kind, and only in a text of the field's length. Measured on
generated samples (the OCR benchmark): 40 of 40 single-line lot and date markings in three fonts
read exactly; the serial number on 12 of 12 simulated carrier tags found in whole 1600 × 1200 frames. These are
synthetic prints; real markings (dot-peen, laser on metal, inkjet on curved surfaces) need their own samples and, often,
contrast and flat-field correction first.

## What PyFlow, PyFlowOpenCv and ComfyUI contributed

PyFlowOpenCv (GPL-3.0) wraps OpenCV functions as nodes of PyFlow (Apache-2.0), an editor for general visual programming;
ComfyUI is a node editor for diffusion pipelines. They were studied for their catalogues and interaction ideas; no code
was copied, for three reasons: Veyra's tools must state typed inputs and outputs, keep the transform back to the frame
and never modify the original image, which PyFlow's untyped pins and in-place image objects do not; the workspace's
rules forbid copying other projects' code; and most of PyFlowOpenCv's functions are thin calls of OpenCV, which Veyra
calls directly. What carried over is the catalogue: filtering (blur, bilateral, median, Gabor/Sobel/Scharr/Laplacian
gradients), morphology, thresholds and colour filters, contours and shapes, Hough lines and circles, feature detectors
and matchers (ORB, AKAZE, SIFT, BRISK in OpenCV) for alignment, image arithmetic for reference comparison, OCR,
and the idea of watching any intermediate image. Their face, gender and HED networks, optical flow and background
subtraction are not inspection tools and were left out; ComfyUI's grouping of nodes and its quick search inspired the
graph's groups and the command palette's *Add* entries.

## The catalogue

<!-- generated:tools -->
**72 tools in 18 categories.**

### Acquisition

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Frame input** `input.frame` | — → Frame (Frame) | The frame acquired for a camera role: from an image file, a folder, a dataset, the simulator or a camera. |

### Image processing

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Region of interest** `image.roi` | Image (Image) → Image (Image) | Restrict the following tools to a region. Results found inside it stay placed correctly on the original. |
| **Grayscale** `image.gray` | Image (Image) → Image (Image) | Convert to a single channel: luminance, one colour channel, or saturation / value in HSV. |
| **Blur** `image.blur` | Image (Image) → Image (Image) | Reduce noise before thresholding or edge detection. |
| **Threshold** `image.threshold` | Image (Image) → Mask (Mask) | Separate objects from the background by brightness. |
| **Morphology** `image.morphology` | Mask (Mask) → Mask (Mask) | Clean a mask: remove specks, fill gaps, grow or shrink regions. |
| **Color space** `image.convert` | Image (Image) → Image (Image) | Convert to another colour space, or extract one of its channels, before thresholding or measuring colour. |
| **Contrast** `image.contrast` | Image (Image) → Image (Image) | Make faint features visible with consistent contrast. Colour images are processed in their lightness only, so colours do not shift. |
| **Flatten lighting** `image.flatfield` | Image (Image) → Image (Image) | Remove uneven lighting (vignetting, a light from one side) by estimating the background with a large blur. |
| **Invert** `image.invert` | Image (Image) → Image (Image) | Swap dark and bright. |
| **Denoise** `image.denoise` | Image (Image) → Image (Image) | Non-local means denoising: removes sensor noise while keeping edges (slower than a blur). |
| **Sharpen** `image.sharpen` | Image (Image) → Image (Image) | Unsharp masking: strengthen edges and fine detail. |
| **Gradient** `image.gradient` | Image (Image) → Image (Image) | Edge strength of a grey image (scaled to 0–255): the input of thresholds that look for edges or scratches. |
| **Color threshold** `image.colorthreshold` | Image (Image) → Mask (Mask) | Select the pixels within a hue, saturation and brightness range: coloured parts, seals, labels, wires. |
| **Resize** `image.resize` | Image (Image) → Image (Image) | Scale the image (to speed up later tools, or to give a model its input size). |
| **Rotate** `image.rotate` | Image (Image) → Image (Image) | Turn the image; results found in it stay placed correctly on the original. |
| **Flip** `image.flip` | Image (Image) → Image (Image) | Mirror the image (a camera mounted the other way round). |
| **Perspective correction** `image.perspective` | Image (Image) → Image (Image) | Rectify a tilted view of a flat surface (a label, a panel seen at an angle) into a straight rectangle. Measurements inside are in the rectangle's pixels; results stay placed correctly on the original. |
| **Combine masks** `mask.combine` | Mask A (Mask), Mask B (Mask) → Mask (Mask) | Combine two masks of the same image pixel by pixel. |
| **Fill holes** `mask.fill` | Mask (Mask) → Mask (Mask) | Fill the enclosed holes of every region (a ring becomes a disc). |
| **Separate touching objects** `mask.separate` | Mask (Mask) → Mask (Mask) | Split objects that touch (pills, nuts, cells) along their necks with a distance-transform watershed, so blob analysis counts them one by one. |
| **Skeleton** `mask.skeleton` | Mask (Mask) → Mask (Mask) | Thin every region to its one-pixel centre line (wires, cracks, traces): its length is then its pixel count. |

### Alignment

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Align to reference** `align.features` | Image (Image) → Image (Image), Alignment (Alignment) | Find the taught reference by its features (any position and rotation, some scale and perspective) and resample the image onto it. Later regions drawn on the reference follow the part. |
| **Shift alignment** `align.phase` | Image (Image) → Image (Image), Alignment (Alignment) | Find how far the part moved since the reference was taught (no rotation) by phase correlation over the reference region: fast, sub-pixel, robust to noise. |
| **Refine alignment** `align.ecc` | Image (Image) → Image (Image), Alignment (Alignment) | Refine the alignment to sub-pixel precision by maximising the correlation with the reference. It converges only from a few pixels or degrees away: put feature or shift alignment before it for larger motions. |
| **Compare with reference** `compare.reference` | Image (Image) → Mask (Mask), Difference (Image), Area (Measurement) | Compare the aligned part with the good reference: missing or extra material, misprints, stains. Brightness is matched first, small misregistration along edges is tolerated, and specks below the smallest defect are ignored. |

### Blob analysis

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Blob analysis** `vision.blobs` | Mask (Mask) → Blobs (BlobSet) | Find the connected regions of a mask with their area, perimeter, centroid, box and circularity. |

### Measurement

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Measure objects** `measure.objects` | Detections (Detections) → Measurement (Measurement) | Turn what a model found into a number in calibrated units: how many, how large, at which angle, how far apart two keypoints are. The measurement of the largest object is taken where one object is needed. |
| **Measure blob** `measure.blob` | Blobs (BlobSet) → Measurement (Measurement) | Measure one feature of one blob, in the calibrated unit of the camera that took the image. |
| **Coverage** `measure.coverage` | Mask (Mask) → Coverage (Measurement) | The share of the mask that is set, in percent: presence and absence checks on a thresholded region. |
| **Intensity statistics** `measure.intensity` | Image (Image) → Value (Measurement) | A statistic of the grey levels of an image or region (colour images use their luminance). |
| **Measure shape** `measure.shape` | Blobs (BlobSet) → Measurement (Measurement) | Measure the shape of one blob from its outline (Blob analysis must keep outlines): its true length and width at any orientation, its angle, how convex and how elongated it is. |

### Edges and geometry

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Edge detection** `vision.edges` | Image (Image) → Mask (Mask) | Find edges with hysteresis thresholds (Canny): strong edges, and weak ones connected to strong ones. |
| **Caliper** `measure.caliper` | Image (Image) → Width (Measurement) | Measure the width of a part across a band: the profile is averaged across the band, and the distance between its opening and closing edges is converted to the camera's calibrated unit. |
| **Circle fit** `measure.circle` | Image (Image) → Circle (Measurement) | Find a circular edge along rays through a ring and fit a circle to it, rejecting outliers; the diameter is in the camera's calibrated unit and the circle is placed on the original image. |
| **Distance and angle** `measure.distance` | From (Point), To (Point) → Distance (Measurement), Angle (Measurement) | The distance between two points in the calibrated unit, and the direction from the first to the second. |
| **Find line** `measure.line` | Image (Image) → Line (Line), Angle (Measurement), Straightness (Measurement) | Find a straight edge in a band (along parallel rays, with sub-pixel edge points) and fit a line to it, rejecting outliers: a datum for distances and angles, or a check of straightness. |
| **Angle between lines** `measure.angle` | Line A (Line), Line B (Line) → Angle (Measurement), Intersection (Point) | The angle between two lines (squareness of a corner, parallelism of two edges) and where they meet. |
| **Point to line distance** `measure.pointline` | Point (Point), Line (Line) → Distance (Measurement) | The perpendicular distance of a point (a hole, a found pattern) from a line (a datum edge). |
| **Find circles** `vision.circles` | Image (Image) → Blobs (BlobSet) | Find circular features (holes, rivets, caps) with the Hough transform; each is returned as a blob, so blob measurements and count checks apply. |

### Color

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Color statistics** `color.stats` | Image (Image) → Color (ColorStats) | Measure the mean colour of a ring or rectangle: BGR, CIE L*a*b*, hue and saturation. |

### Pattern

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Pattern match** `vision.pattern` | Image (Image) → Match (PatternMatch) | Locate the taught pattern: where it is (on the original image), how it is rotated, and how well it matches. |

### Calibration

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Remove lens distortion** `image.undistort` | Image (Image) → Image (Image) | Remove the lens distortion measured when the camera was calibrated with a checkerboard, so straight edges are straight and distances are true across the whole field of view. Results stay placed on the original; they differ from it by the distortion itself. |

### Identification

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Read code** `id.code` | Image (Image) → Codes (Codes), Text (Text) | Find and decode Data Matrix, QR, PDF417 and Aztec codes and linear barcodes; outlines are placed on the original image. |
| **Read text** `id.ocr` | Image (Image) → Reading (Reading), Text (Text) | Read printed text (lot numbers, date codes, serials, labels) with the PP-OCR models; each line has its confidence and outline on the original image. Verify it with a Text check. |

### Classification

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Classification** `ai.classify` | Image (Image) → Classification (Classification) | Assign the image to one of the model's classes, with the probability of each. |

### Object detection

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Object detection** `ai.detect` | Image (Image) → Detections (Detections) | Find objects with a detection model; boxes are placed on the original image. Segmentation, pose and rotated-box models also detect, so they are accepted too (their boxes are used). |
| **Keypoints** `ai.pose` | Image (Image) → Detections (Detections) | Find objects and their named keypoints (landmarks); distances and angles between keypoints can be measured. |
| **Rotated objects** `ai.obb` | Image (Image) → Detections (Detections) | Find objects at any angle with rotated boxes: their orientation and true size. |

### Segmentation

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Instance segmentation** `ai.segment` | Image (Image) → Detections (Detections) | Find objects and their exact outlines; areas are measured from the outlines, not the boxes. |
| **Semantic segmentation** `ai.semantic` | Image (Image) → Segmentation (Segmentation) | Classify every pixel; the result has the area of each class. |

### Anomaly detection

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Anomaly detection** `ai.anomaly` | Image (Image) → Anomaly (Anomaly) | Score how unlike the good parts an image is, and where. The model is loaded once and kept while only the threshold changes. |

### Logic

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Calculate** `logic.math` | A (Number), B (Number) → Value (Measurement) | Arithmetic on measured values (a difference of two diameters, a ratio of areas, an offset from nominal). |
| **Compare** `logic.compare` | A (Number), B (Number) → Result (Bool) | Compare two values (or a value and a constant) into true or false, for conditions and combined checks. |
| **Logic gate** `logic.boolean` | Conditions (Bool) → Result (Bool) | Combine conditions. |
| **Choose value** `logic.select` | Condition (Bool), When true (Number), When false (Number) → Value (Measurement) | Use one value or the other depending on a condition (a limit for one product variant or another). |

### Checks

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Anomaly check** `check.anomaly` | Anomaly (Anomaly) → Verdict (Verdict) | Judge an anomaly score against a limit. |
| **Class area check** `check.segmentation` | Segmentation (Segmentation) → Verdict (Verdict) | Judge the area of one class of a semantic segmentation. |
| **Detection check** `check.detections` | Detections (Detections) → Verdict (Verdict) | Judge how many objects (of one class) were detected. |
| **Classification check** `check.classification` | Classification (Classification) → Verdict (Verdict) | Judge a class: good classes pass, others fail, and an uncertain answer is Unknown. |
| **Pattern check** `check.pattern` | Match (PatternMatch) → Verdict (Verdict) | Judge whether the pattern is present (score) and oriented as expected (rotation). |
| **Alignment check** `check.alignment` | Alignment (Alignment) → Verdict (Verdict) | Judge whether the part was found where and how it should be: present (the match), in place (the shift) and the right way round (the rotation). |
| **Color check** `check.color` | Color (ColorStats) → Verdict (Verdict) | Judge the colour difference (CIE76 ΔE*ab) between a region and a reference colour. |
| **Code check** `check.code` | Codes (Codes) → Verdict (Verdict) | Judge that the expected number of codes was read and that their text has the expected form. |
| **Condition check** `check.condition` | Condition (Bool) → Verdict (Verdict) | Turn a condition into a verdict. |
| **Text check** `check.text` | Text (Text) → Verdict (Verdict) | Verify printed or read text: a lot number's format, a product name, that a date code is present. |
| **Tolerance check** `check.measurement` | Measurement (Measurement) → Verdict (Verdict) | Judge a measurement against nominal ± tolerance. |
| **Range check** `check.range` | Value (Number) → Verdict (Verdict) | Judge any number against a minimum and a maximum. |
| **Count check** `check.count` | Blobs (BlobSet) → Verdict (Verdict) | Judge how many blobs were found: presence, absence, or an exact number of features. |

### Decision

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Decision** `logic.decision` | Verdicts (Verdict) → Inspection (Inspection) | Combine the verdicts of traditional and AI checks into Good, Not good, Unknown or Error. |

### Output

| Tool | Inputs → outputs | What it does |
|---|---|---|
| **Final result** `output.decision` | Inspection (Inspection), Product id (Text) → Inspection (Inspection) | Marks the inspection result of the workflow: what production counts, stores and signals, with the part's identity when the workflow reads it. |
| **Output signals** `output.signal` | Inspection (Inspection) → Signal (Signal) | Map the inspection status to device outputs (a PLC, I/O module or the simulator). |
<!-- /generated:tools -->
