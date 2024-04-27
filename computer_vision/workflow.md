# Computer Vision Workflow
This page explains how to train a [YOLO](https://doi.org/10.48550/arXiv.1506.02640) model for use in a DAI camera.

We'll use the following repositories in this guide:
- [cv-simulation](https://github.com/DukeRobotics/cv-simulation/tree/main): A Unity 3D underwater simulation to generate synthetic, labeled data for computer vision.
- [cv-training](https://github.com/DukeRobotics/cv-training): Scripts to train computer vision models.
- [robosub-ros](https://github.com/DukeRobotics/robosub-ros): ROS system to control a robot for the RoboSub Competition.

## Dataset Generation
First, use [cv-simulation](https://github.com/DukeRobotics/cv-simulation/tree/main) to generate a synthetic dataset using Unity Perception. Follow the repository README for detailed steps.

Make sure to set the game resolution to `416x416` (the resolution of the DAI cameras) and generate 10,000 images.

## Dataset Filtering
First, change to the `Datasets` directory.
```bash
cd Datasets
```

Convert the Unity SOLO dataset to the COCO format:
```bash
solo2coco solo coco
```
> [!NOTE]
> `solo2coco` creates the real `coco` dataset inside an extraneous `coco` directory. In the following steps, either specify `coco/coco` instead of just `coco` or remove the unnecessary parent folder.

Navigate to `coco/images` and manually delete any poor quality images. Unity Perception usually outputs some bad images at the beginning of generation.

Finally, remove bad annotations:
```bash
python bbox_filter.py coco
```

## Roboflow Upload
Create a new Roboflow project by duplicating a null images project. For example, for RoboSub 2024, we duplicated [null_images_base](https://universe.roboflow.com/duke-robotics-club-2024/null_images_base). The null images in `null_images_base` can be used in any underwater dataset to enhance model robustness.

Upload to this new Roboflow project:
```bash
python roboflow_upload.py coco
```

In Roboflow, generate a new dataset version. You can use the settings from previous years as a starting point.

## CV Training
> [!NOTE]
> For cv-training, only Ubuntu 22.04 LTS is officially supported. Also, ensure you have a CUDA-enabled GPU.

Use the [cv-training](https://github.com/DukeRobotics/cv-training) repository to download the Roboflow dataset and train a YOLOv7-tiny model.

After training, upload the `best.pt` weights file to [tools.luxonis.com](https://tools.luxonis.com). Choose `YoloV7` as the YOLO version and input `416` as the input image shape. Download the `.blob` file.

## DAI Camera Upload
Upload the `.blob` file to [robosub-ros](https://github.com/DukeRobotics/robosub-ros). See the `cv` package README for details. Ensure that the appropiate configuration files are updated.
