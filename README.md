<div align="center">

# AV Perception

**Fine-tuning Faster R-CNN on KITTI, and finding out where it breaks.**

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)
![torchvision](https://img.shields.io/badge/torchvision-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)
![Faster_R--CNN](https://img.shields.io/badge/Faster_R--CNN-333333?style=flat-square)
![ResNet--50--FPN](https://img.shields.io/badge/ResNet--50--FPN-333333?style=flat-square)
![KITTI](https://img.shields.io/badge/KITTI-Object%20Detection-0B6E99?style=flat-square)
![mAP@50](https://img.shields.io/badge/mAP%4050-0.879-2EA043?style=flat-square)

</div>

A Faster R-CNN detector with a ResNet-50-FPN backbone, fine-tuned on the KITTI
benchmark to detect and classify vehicles, pedestrians, and cyclists.

It reaches **0.879 mAP@50** on a held-out validation split. The more interesting
half of this project is what happens when you point it at footage KITTI never
covered. See [Limitations](#limitations).

![KITTI validation sample](results/val_dataset/val_detection_3905.png)

---

## Dataset

[KITTI Object Detection Benchmark](https://www.cvlibs.net/datasets/kitti/eval_object.php):
7,481 annotated images captured from a vehicle around Karlsruhe, Germany, in
clear daytime conditions.

---

## Results

Evaluated with `torchmetrics` `MeanAveragePrecision` (bbox, xyxy) on a held-out
20% validation split, 1,497 of the 7,481 images, partitioned with a fixed seed so
the split is reproducible.

<table align="center">
<tr>
  <th>Metric</th>
  <th>Before fine-tuning</th>
  <th>After fine-tuning</th>
</tr>
<tr>
  <td>mAP@50</td>
  <td align="right">0.020</td>
  <td align="right"><b>0.879</b></td>
</tr>
<tr>
  <td>mAP@0.5:0.95</td>
  <td align="right">0.004</td>
  <td align="right"><b>0.572</b></td>
</tr>
</table>

The "before" column is the COCO-pretrained network with its box predictor head
replaced but not yet trained. The backbone already knows how to see, but the new
head is randomly initialized, so its class predictions are noise. The gap between
the columns is what 10 epochs of fine-tuning actually bought.

### Training setup

| | |
|---|---|
| Model | `fasterrcnn_resnet50_fpn_v2`, COCO-pretrained |
| Transfer method | Replaced the box predictor head (4 classes: background, Vehicle, Pedestrian, Cyclist) |
| Split | 80/20 train/val, `manual_seed(42)` |
| Epochs | 10 |
| Batch size | 8 |
| Optimizer | SGD, lr 0.005, momentum 0.9, weight decay 0.0005 |
| Augmentation | Random horizontal flip (p=0.5), training split only |

KITTI's `Car`, `Van`, and `Truck` labels are collapsed into a single **Vehicle**
class; `Pedestrian` and `Cyclist` are kept as-is. `Person_sitting`, `Tram`,
`Misc`, and `DontCare` are excluded rather than forced into a class they don't
belong in.

### Sample detections

| | |
|---|---|
| ![](results/val_dataset/val_detection_3905.png) | ![](results/val_dataset/val_detection_6817.png) |
| ![](results/val_dataset/val_detection_3656.png) | ![](results/val_dataset/val_detection_6938.png) |

---

## Limitations

The validation numbers above are flattering, because validation images come from
the same distribution as training: clear daytime footage, one city, one camera
rig. To find out what the model had actually learned, I ran it on dashcam footage
it had never seen, from different cities, different cameras, and crucially,
different times of day.

**It holds up in conditions resembling KITTI.** Daytime footage with clear
separation between vehicles produces confident, well-placed boxes:

![Daytime, Los Angeles](results/yt/day_la.png)

**It degrades sharply at night.** Recall drops, and vehicles that a human reads
instantly from their headlights alone go undetected:

![Night, Tokyo, missed detections](results/yt/night_tokyo_fail.png)

**It also struggles with dense urban scenes** it has no analogue for in training:
heavy occlusion, unfamiliar vehicle types, and crowded frames.

![Daytime, Nashville, failure case](results/yt/day_nashville_fail.png)

### Why this happens

This is a **domain gap**, not a modeling bug. KITTI contains essentially no night
scenes, no rain or snow, and no dense city traffic, so the detector never had the
chance to learn what a car looks like when it's a pair of headlights and a
silhouette. A higher mAP on KITTI's own validation split would not fix it.

It's also the reason production autonomous-vehicle stacks fuse infrared and radar
with visible-light cameras rather than relying on RGB alone. The failure mode
above is exactly what the extra sensors exist to cover.

### What would close the gap

- Train on a dataset with night and adverse-weather coverage (BDD100K, nuScenes), or combine one with KITTI
- Photometric augmentation (brightness, contrast, and gamma jitter) to simulate low light
- Report recall separately by lighting condition rather than reading a single aggregate mAP

All failure cases are kept in `results/yt/` rather than pruned, since they're the
more informative half of the evaluation.

---

## Installation

1. **Clone the repository**

    ```bash
    git clone https://github.com/beck-bargas/av-perception.git
    cd av-perception
    ```

2. **Create and activate a virtual environment**

    **Windows PowerShell:**
    ```powershell
    py -m venv .venv
    .\.venv\Scripts\Activate.ps1
    ```

    **macOS/Linux:**
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate
    ```

3. **Install dependencies**

    ```bash
    pip install -r requirements.txt
    ```

    A CUDA-capable GPU is strongly recommended for training. Inference on a
    handful of images runs fine on CPU.

---

## Usage

### Run detection with pretrained weights

Download `checkpoint_e10.pth` from the
[latest release](https://github.com/beck-bargas/av-perception/releases) into the
repo root. In the notebook, run the setup cells (model creation and the `detect`
function), then:

```python
model.load_state_dict(torch.load("checkpoint_e10.pth"))
detect("path/to/your/image.jpg")
```

### Train from scratch

Run `fine_tune.ipynb` top to bottom. KITTI (~12 GB) downloads automatically on
the first run. Training is 10 epochs over roughly 5,984 images in batches of 8.

---

## Repository Structure

```
av-perception/
├── fine_tune.ipynb          # training, evaluation, and detection
├── requirements.txt
└── results/
    ├── val_dataset/         # detections on held-out KITTI validation images
    └── yt/                  # out-of-distribution dashcam footage, incl. failures
```
