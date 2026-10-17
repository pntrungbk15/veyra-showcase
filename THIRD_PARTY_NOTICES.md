# Third-party components

Veyra ships no third-party code, models, images or datasets in its source tree. The sample parts are drawn by
`veyra/simulation.py`, and the sample anomaly model is trained on them at run time.

## Required

| Component | Licence | Use |
|---|---|---|
| FluentQt Community | GPL-3.0 or the FluentQt commercial licence | UI framework and persistence (`fluentqt.storage`). |
| FluentQt Pro | Proprietary (see FluentQt Pro's `LICENSE`) | Image viewer, property inspector, data grid, charts, pipeline view, segmented progress, command palette. |
| [PyQt5](https://www.riverbankcomputing.com/software/pyqt/) / Qt 5 | GPL-3.0 (PyQt5); LGPL-3.0 (Qt) | Qt bindings (studio only; the runtime does not import Qt). |
| [NumPy](https://numpy.org/) | BSD-3-Clause | Image arrays everywhere. |
| [OpenCV](https://opencv.org/) (`opencv-python-headless`) | Apache-2.0 (OpenCV 4.5+); the wheels bundle further components under their own permissive licences | Vision tools, image decoding, the simulator. |
| [ONNX Runtime](https://onnxruntime.ai/) | MIT | Model inference in the runtime process. |
| [ONNX](https://onnx.ai/) (`onnx`) | Apache-2.0 | Exporting models trained in Veyra (the built-in anomaly model). |
| [openpyxl](https://foss.heptapod.net/openpyxl/openpyxl) with [et-xmlfile](https://foss.heptapod.net/openpyxl/et_xmlfile) | MIT | Writing Excel exports in the runtime process. |
| [lxml](https://lxml.de/) | BSD-3-Clause (bundles libxml2 and libxslt, MIT) | Lets openpyxl write large exports faster. |
| [PyYAML](https://pyyaml.org/) | MIT | Reading the detector configurations of the model catalogue. |
| [zxing-cpp](https://github.com/zxing-cpp/zxing-cpp) | Apache-2.0 | Reading Data Matrix, QR, PDF417, Aztec and linear codes. |

Because the application links GPL components, it is distributed under GPL-3.0-or-later.

## Optional

| Component | Licence | Use |
|---|---|---|
| [PyTorch](https://pytorch.org/), torchvision, einops (`[training]`) | BSD-3-Clause, BSD-3-Clause, MIT | Training deep models in the trainer process; never imported by the studio or the runtime. Not in the standalone build. |
| [pypylon](https://github.com/basler/pypylon) (`[basler]`) | BSD-3-Clause for pypylon; it carries Basler's pylon runtime under Basler's own licence | Basler cameras. Not in the standalone build unless built with `--pylon`, after reviewing Basler's licence. |
| [harvesters](https://github.com/genicam/harvesters) with [genicam](https://pypi.org/project/genicam/) (`[genicam]`) | Apache-2.0 (harvesters); the GenICam licence of EMVA (genicam, which includes a simulated GenTL producer) | GenICam cameras through their vendor's GenTL producer. Not in the standalone build unless built with `--genicam`. |

## Used through the AI environment (never bundled, installed by the user)

| Component | Licence | Use |
|---|---|---|
| [Ultralytics](https://github.com/ultralytics/ultralytics) | AGPL-3.0, or Ultralytics' Enterprise licence | Training, fine-tuning and exporting YOLO, RT-DETR, YOLO-World and YOLOE models in a separate Python environment. Veyra runs the exported ONNX files with its own code. The pretrained weights are AGPL-3.0 too and inherit their training data's terms (COCO, ImageNet, DOTA, Open Images, Cityscapes). |
| [Anomalib](https://github.com/open-edge-platform/anomalib) with PyTorch Lightning and timm | Apache-2.0 | Training and exporting anomaly models in a separate Python environment. Pretrained backbones come from torchvision, timm and Hugging Face under their own licences. |
| [PyTorch](https://pytorch.org/), ONNX Script, onnxslim | BSD-3-Clause, MIT, MIT | The frameworks' runtime and ONNX exporters, in the same environment. |

## Downloaded on request

| Component | Licence | Use |
|---|---|---|
| PP-OCRv4 text detection and PP-OCRv3 English recognition ([PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR)), ONNX conversion by [RapidOCR](https://huggingface.co/SWHL/RapidOCR) | Apache-2.0 | The *Read text* tool. Downloaded from a pinned revision into the data folder and verified against their SHA-256. |

## Studied, not copied

[PyFlow](https://github.com/pedroCabrera/PyFlow) (Apache-2.0), [PyFlowOpenCv](https://github.com/pedroCabrera/PyFlowOpenCv)
(GPL-3.0) and [ComfyUI](https://github.com/comfy-org/comfyui) (GPL-3.0) were studied for their node catalogues and
editing ideas; no code, text or assets were copied (see [docs/vision-tools.md](docs/vision-tools.md)). Ultralytics'
and Anomalib's export formats and post-processing (letterboxing, the decoding of each head, probabilistic-IoU
suppression, score normalisation) are reimplemented from their documented behaviour and verified against them.

## Not bundled

GenTL producers, PLC libraries, TensorRT and third-party model weights are not part of Veyra. Modbus TCP is
implemented in Veyra itself; its tests use [pymodbus](https://github.com/pymodbus-dev/pymodbus) (BSD-3-Clause) as an
independent server and client, in the maintainers' test environment only.
