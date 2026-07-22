---
layout: project
title: "Car Inspect AI: Teaching YOLO to Read a Car the Way an Inspector Does"
date: 2026-06-22 14:00:00 +0000
lang: en
tags: [Computer Vision, Machine Learning, Python, MLOps]
type: project
github_repo: "https://github.com/sanayasfp/car-inspect-ai"
---

Anyone who's rented a car or filed an insurance claim knows the ritual: someone walks around the vehicle, snaps a few photos, and a human decides later whether that mark on the bumper was already there. It's slow, it's subjective, and disputes between renters, owners, and insurers over "pre-existing damage" are extremely common precisely because the whole process runs on someone's word against a handful of photos. **Car Inspect AI** is my attempt to chip away at that problem: a computer-vision system that automatically detects and labels the individual parts of a vehicle from a photo, as a first building block toward more objective, automated inspection.

Code's here: [github.com/sanayasfp/car-inspect-ai](https://github.com/sanayasfp/car-inspect-ai).

## Starting with the actual constraint: data

Before touching a model, the real constraint on a project like this is data. I trained on the public **Car Parts Segmentation** dataset (Kitsuchart Pasupa et al.), 500 annotated images of sedans, pickups, and SUVs in COCO format, covering 18 distinct vehicle parts — bumpers, doors, lights, mirrors, hood, trunk, wheels, and so on, shot from front, back, and angled views, with plates and faces blurred for privacy. 500 images is not a lot by deep-learning standards, and that constraint shaped almost every other decision in the project.

## Why YOLO11n specifically

I picked **YOLO11n** — the nano variant — deliberately, not just because "YOLO is what people use for object detection." Two things mattered:

- **It's small enough to run on modest hardware.** A tool meant for garages, small rental agencies, or independent inspectors is useless if it needs a beefy GPU to run inference. YOLO11n trades some raw accuracy for a footprint that runs comfortably on a CPU or an entry-level GPU.
- **With only 500 images, model capacity is a liability, not an asset.** A larger model has more room to overfit a small dataset. A lightweight architecture, combined with aggressive data augmentation, was the more honest choice given what I actually had to train on.

To stretch that small dataset further, preprocessing included resizing to YOLO's expected input dimensions and augmenting with rotation (simulating different shooting angles), horizontal flips, and Gaussian noise (to make the model less sensitive to lighting variation — a real problem when photos come from random phone cameras in a parking lot, not a studio).

Training ran for 50 epochs with an 80/20 train/validation split, batch size 16, and early stopping if validation performance stalled for 10 consecutive epochs. The result: **87% mAP** on the validation set, with the strongest performance on well-defined, geometrically simple parts like wheels and doors — exactly where I'd expect a detector to do best, and exactly the kind of result that tells you where to focus next (smaller/ambiguous parts like mirrors are the harder cases).

## Rolling my own tiny ORM instead of reaching for SQLAlchemy

This is the part of the project I'd guess most people skip past, but it's the part I learned the most from. The app needs to track training runs — which model, how many epochs, whether it completed, where the checkpoint lives, and whether it resumes from a previous run. Instead of pulling in SQLAlchemy for what's fundamentally a handful of tables, I wrote a small dataclass-based model layer myself:

```python
@dataclasses.dataclass
class TrainLogsModel(BaseModel):
    _table_name = "train_logs"

    name: str
    epochs: int
    model: str
    path: str
    completed: bool = Field(type=bool, default=False).set()
    id: Optional[int] = Field(type=int, primary_key=True, autoincrement=True).set()
    created_at: Optional[float] = Field(type=int, default=lambda: dt.now().timestamp()).set()
    resumed_from: Optional[int] = Field(type=int, foreign_key="id", foreign_table=_table_name).set()
```

`BaseModel` reads the dataclass's type annotations and field metadata and turns them into SQL column definitions (`INTEGER`, `TEXT`, `REAL`, with primary keys, autoincrement, and foreign keys handled explicitly), and derives table names from class names in snake_case, camelCase, or PascalCase depending on what's asked for. It's a fraction of what a real ORM does — no query builder, no migrations system — but building even that fraction by hand forced me to actually understand what an ORM is automating away, rather than just importing one and trusting the magic. That's the trade I'd make again: for a project this size, writing 100 lines to understand the mechanism beat 10 lines that hide it.

That model backs a genuinely useful feature: the training page lets me pick either a fresh YOLO11n base or any previous checkpoint, kick off training, and log it — including a `resumed_from` foreign key back to the run it continued, so I have an actual lineage of experiments instead of a folder full of `best_v2_final_FINAL.pt` files.

## The interface: Streamlit, on purpose

The whole thing is wrapped in a small multi-page Streamlit app — a home page, a "register a car" page (upload front/back/left/right photos plus color and plate number), the training page described above, and a scratch page for in-progress experiments. Streamlit was the right call here specifically because it isn't the point of the project: I wanted to spend my time on the detection model and the training/versioning story, not on hand-rolling a frontend, and Streamlit gets out of the way for that.

## Where it actually stands right now

I want to be precise about this rather than round it up: the part-detection model and the training/versioning pipeline are working and measured (that 87% mAP figure is real, from an actual run, not an estimate). The full inspection pipeline — fusing all four angles of a vehicle into a single confident report, and moving from "these are the detected parts" to an actual damage or fraud verdict — is still active, in-progress work, not a finished product. The README lists automated damage severity scoring and vehicle-history integration as future directions, and that's accurate: those are the roadmap, not something I'm claiming already works end-to-end.

If I were pitching what's genuinely done today: a lightweight, honestly-benchmarked vehicle part detector, trained reproducibly, with its own minimal experiment-tracking layer built from first principles. That's a smaller claim than "automated fraud detection system," but it's the true one — and it's a better foundation to build the rest on top of.

Repo's public if you want to see the training code, the mini-ORM, or argue that I should've just used SQLAlchemy: [github.com/sanayasfp/car-inspect-ai](https://github.com/sanayasfp/car-inspect-ai).
