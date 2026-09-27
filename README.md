# Gym Exercise Rep Counter using Pose Estimation

A computer vision project that detects a person performing an exercise in a video, tracks their joint angles frame by frame using pose estimation, counts completed repetitions automatically, and displays an on-screen alert once a target rep count is reached.

## Overview

Manually counting reps during a workout, or reviewing training footage afterward, is repetitive and easy to get wrong. This project automates that process using human pose estimation. Instead of relying on full body segmentation or manual labeling, it tracks specific body landmarks (such as the shoulder, elbow, and wrist for an arm exercise) and calculates the angle at the relevant joint on every frame. A simple state machine then interprets the changing angle as a completed rep every time the joint moves from a bent position to a fully straight position, or vice versa depending on the exercise.

This approach is lightweight, does not require any custom model training, and works directly on a standard video file.

## Features

- Automatic rep counting from a video file, no manual frame counting needed
- Joint angle calculated mathematically from three body landmarks using vector geometry
- Configurable per exercise, with separate landmark sets and angle thresholds for each
- On-screen visual alert that appears once the target rep count is reached
- Skeleton overlay drawn on every frame, showing exactly what the model is tracking
- Per-rep logging with the timestamp (in video time) at which each rep was completed
- Diagnostic reporting (frames processed, frames where a person was detected, and the actual angle range observed) to help tune thresholds for a new video

## How It Works

### 1. Pose estimation
Each video frame is passed through Google's Mediapipe Pose model, which returns 33 body landmarks (shoulders, elbows, wrists, hips, knees, ankles, and more), each with normalized x and y coordinates.

### 2. Angle calculation
For any joint, three landmarks are used: the point before it, the joint itself, and the point after it. For example, an elbow angle uses the shoulder, elbow, and wrist. Two vectors are built outward from the middle point, and the angle between them is calculated using the standard vector dot product formula:

```
angle = arccos( (vector1 . vector2) / (|vector1| * |vector2|) )
```

This produces a single angle in degrees, for example close to 180 when a limb is fully extended, and a much smaller number when it is fully bent.

### 3. Exercise configuration
Each supported exercise defines which three landmarks to track and two angle thresholds: one representing the fully straight position, and one representing the fully bent position. These thresholds vary by exercise since different movements have different natural ranges of motion.

| Exercise | Landmarks used | Straight angle threshold | Bent angle threshold |
|---|---|---|---|
| Bicep curl | Shoulder, elbow, wrist | Above 160 degrees | Below 40 degrees |
| Squat | Hip, knee, ankle | Above 165 degrees | Below 100 degrees |
| Push up | Shoulder, elbow, wrist | Above 160 degrees | Below 90 degrees |
| Pull up | Shoulder, elbow, wrist | Above 160 degrees | Below 70 degrees |

### 4. Rep counting logic
A state variable tracks whether the joint is currently in the "up" (straight) or "down" (bent) position. A rep is only counted when the joint transitions from bent back to straight, completing one full cycle. This prevents counting the same rep multiple times, or counting partial, incomplete movements.

### 5. Alert system
Once the rep counter reaches the target count set for the session, a green on-screen text overlay appears on every subsequent frame of the output video, confirming the target has been reached.

### 6. Logging
Every time a rep is counted, its rep number and the corresponding timestamp within the video are recorded and saved to a plain text log file after processing finishes.

## Tech Stack

- **Python**
- **OpenCV** — video reading, frame processing, drawing overlays, and writing the output video
- **Mediapipe** — pose estimation model providing body landmark detection
- **NumPy** — vector and angle calculations

## Project Structure

```
.
├── gym-count-project.ipynb     Main notebook containing all project code
├── output_video_fixed.mp4      Annotated output video with skeleton overlay, rep counter, and alert
├── rep_log.txt                 Text log of each completed rep with its video timestamp
└── README.md
```

## How to Run

This project was developed and tested in a Kaggle Notebook environment.

1. Upload a video file of the exercise you want to analyze as a Kaggle dataset input
2. Install Mediapipe (a specific version is required for compatibility, see note below)
```
pip install mediapipe==0.10.14
```
3. Run the notebook cells in order:
   - Import libraries
   - Locate the uploaded video's file path
   - Define the angle calculation function
   - Define the exercise configuration dictionary
   - Run the main processing loop on the video
   - Save and view the rep log
   - Preview the annotated output video

**Compatibility note:** Newer versions of Mediapipe (1.x) restructured the library and removed the `solutions` interface this project relies on. Version 0.10.14 is confirmed to work correctly alongside a compatible protobuf version (5.28.3) on Kaggle's environment.

## Sample Result

Using a sample video of a person performing pull-ups:

- Total frames processed: 566
- Frames where a person was successfully detected: 476
- Observed angle range: 1.2 degrees (fully bent) to 179.9 degrees (fully straight)
- Total reps counted: 3, matching the number of reps visibly performed in the source video
- Target reps set: 3, with the on-screen alert correctly triggering at the moment the third rep was completed
