# TET-harjoittelu 25.–26.3.2026

During this TET week, students will get hands-on practice with a robotic arm. We will use the **SO-101 robot** and [SmolVLA](https://arxiv.org/pdf/2506.01844) for a **pick-and-place** imitation-learning task.

The goal is simple (and fun): you will see how a robot can learn from demonstrations, run the learned policy in real time, and understand the basic idea of a Vision–Language–Action (VLA) model.

| <p align="center"><img src="/attachment/front_camera_view.gif"/><br/></p> | <p align="center"><img src="/attachment/side_camera_view.gif"/><br/></p> |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------ |

# Learning Outcomes

After successfully completing the tasks, the student:

- Understands basic robot operation
- Can perform a pick-and-place task
- Gains beginner experience in training a model
- Improves logical thinking and problem-solving
- Gets hands-on experience with 3D printing

# Prerequisites

- Basic programming knowledge
- Basic Linux command-line knowledge
- No previous robot experience required

# Tasks

0. [Setup & Installation](#0-setup--installation)
1. [3D printing the cube](#1-3d-printing-the-cube)
2. [Calibrate the robot](#2-calibrate-the-robot)
3. [Teleoperate](#3-teleoperate)
4. [Record a dataset](#4-record-a-dataset)
5. [Train the model](#5-train-the-model)
6. [Run the model on the robot (real time)](#6-run-the-model-on-the-robot-real-time)

## 0. Setup & Installation

### Hardware setup

1. Power on the SO-101 robot
2. Connect the SO-101 **leader** robot to the laptop
3. Connect the SO-101 **follower** robot to the laptop
4. Connect the front and side Intel RealSense cameras to the laptop

### Dependencies

Step 1: Install Miniforge

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

Step 2: Create and activate a Conda environment

```bash
conda create -y -n lerobot python=3.10
conda activate lerobot
conda install ffmpeg -c conda-forge
```

Step 3: Install LeRobot

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e .
```

Step 4: Install SmolVLA

```bash
pip install -e ".[smolvla]"
```

<details>
  <summary>Troubleshooting</summary>

If you see build errors, install the following dependencies:

```bash
sudo apt-get install cmake build-essential python3-dev pkg-config libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev libswresample-dev libavfilter-dev
```

</details>

## 1. 3D printing the cube

Download the [3D cube](/3D_cube.stl) and print it. We will use it to train the robot for pick-and-place. The [3D container box](/3D_container_box.stl) is also available.

Tip: try printing the cube in a different color than the box. It often makes training easier.

## 2. Calibrate the robot

In our setup, the follower robot is connected to `/dev/ttyACM1` and the leader is connected to `/dev/ttyACM0`.

Follower

```bash
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_awesome_follower_arm
```

First, move the robot to a position where all joints are roughly in the middle of their ranges. After pressing Enter, move each joint through its full range of motion.

Leader

```bash
lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_awesome_leader_arm
```

First, move the robot to a position where all joints are roughly in the middle of their ranges. After pressing Enter, move each joint through its full range of motion.

[Calibration video](https://huggingface.co/docs/lerobot/en/so101#calibration-video)

Nice! Calibration is done. Next, we will play with the robots using teleoperation.

## 3. Teleoperate

We have two RealSense cameras: one in front, and one mounted on the robot (wrist). Run the following command to enable teleoperation and visualization in `rerun` viewer at the same time.

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{
    front: {
      type: intelrealsense,
      serial_number_or_name: 825312073829,
      width: 640,
      height: 480,
      fps: 30
    },
    side: {
      type: intelrealsense,
      serial_number_or_name: 419122271321,
      width: 640,
      height: 480,
      fps: 30
    }
  }" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true
```

Here the front camera is `serial_number_or_name: 825312073829` and the side camera is `serial_number_or_name: 419122271321`.

<details>
  <summary>Teleoperate without camera</summary>

```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_awesome_follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_awesome_leader_arm
```

</details>

Now you can move the leader, and the follower will mirror the motion. Check the `rerun` dashboard and observe the joint positions.

Great work—this is the core skill you’ll need for collecting a good dataset.

## 4. Record a dataset

We will use the Hugging Face Hub to upload the dataset. Log in to Hugging Face and generate a token from [Hugging Face settings](https://huggingface.co/settings/tokens).

Run the following command and **replace** `${HUGGINGFACE_TOKEN}` with your generated token.

```bash
huggingface-cli login --token ${HUGGINGFACE_TOKEN} --add-to-git-credential
```

Run the following command to store your Hugging Face username in a variable:

```bash
HF_USER=$(hf auth whoami | head -n 1)
echo $HF_USER
```

You will see something like:

**user:** `<your_hugging_face_user_name>`

Now we will record a dataset to train a policy that can run in real time at the end.

Before you start recording, take a quick look at this example dataset: [cube-pick-place](https://huggingface.co/spaces/lerobot/visualize_dataset?path=%2Fsajibpra%2Fcube-pick-place%2Fepisode_0). It helps you understand what “good demonstrations” look like.

Recommended starting point: **50 episodes** at **30 FPS**.

Run the following command to record your dataset.

Note: Replace `sajibpra` with your Hugging Face username and `cube-pick-place` with your repository name.

```bash
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{ front: {type: intelrealsense, serial_number_or_name: 825312073829, width: 640, height: 480, fps: 30}, side: {type: intelrealsense, serial_number_or_name: 419122271321, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_awesome_leader_arm \
    --display_data=true \
    --dataset.repo_id=sajibpra/cube-pick-place \
    --dataset.single_task="Put cube into the box" \
    --dataset.num_episodes=50 \
    --dataset.episode_time_s=30 \
    --dataset.reset_time_s=5
```

| Command                                      | Description                                                      |
| -------------------------------------------- | ---------------------------------------------------------------- |
| `--display_data=true`                        | rerun visualization open                                         |
| `--dataset.repo_id=sajibpra/cube-pick-place` | Hugging face username `sajibpra` and repo name `cube-pick-place` |
| `--dataset.num_episodes=50`                  | Total 50 episodes                                                |
| `--dataset.episode_time_s=30`                | Per episode time is 30 second to record                          |
| `--dataset.reset_time_s=5`                   | 5 second reset time between two episodes                         |

Locally your data will be stored in this folder: `~/.cache/huggingface/lerobot/{repo-id}`.

After successfully recording and uploading, you can [visualize the dataset online](https://huggingface.co/spaces/lerobot/visualize_dataset). Search for: `<your_hugging_face_user_name>/<repository_name>`.

## 5. Train the model

Now that you have a dataset, it’s time to train the model. We will use [SmolVLA](https://huggingface.co/lerobot/smolvla_base) as the base model.

Before we start training, go to [Weights & Biases (wandb)](https://wandb.ai/home), log in, and generate an API key.

In the terminal, run:

```bash
wandb login
```

It will ask for the API key—paste the one you generated.

Now run the following command to start training.

Note: Replace `sajibpra` with your Hugging Face username and `cube-pick-place` with your repository name.

```bash
python src/lerobot/scripts/lerobot_train.py \
  --policy.path=lerobot/smolvla_base \
  --policy.push_to_hub=false \
  --rename_map='{"observation.images.front": "observation.images.camera1", "observation.images.side": "observation.images.camera2"}' \
  --dataset.repo_id=sajibpra/cube-pick-place \
  --batch_size=20 \
  --steps=20000 \
  --output_dir=outputs/train/my_smolvla_local \
  --job_name=my_smolvla_training_local \
  --policy.device=cuda \
  --wandb.enable=true
```

For reference, training on the following PC took about ~2 hours:

**GPU:** NVIDIA GeForce RTX 4070 Ti SUPER
**VRAM:** 16GB
**RAM:** 32GB

After completing training, upload your trained model to Hugging Face.

Note: Replace `sajibpra` with your Hugging Face username.

```bash
huggingface-cli upload sajibpra/my_smolvla_local \
  outputs/train/my_smolvla_local/checkpoints/last/pretrained_model
```

Awesome - you have trained the model and uploaded it to the Hugging Face Hub.

## 6. Run the model on the robot real-time

You have reached the final task: running the model on the robot in real time.

The [hardware setup](#0-setup--installation) is the same as when you recorded the data. This time, you do not need the leader robot - only the follower will perform the task autonomously.

Run the following command.

Note: Replace `sajibpra` with your Hugging Face username.

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{ front: {type: intelrealsense, serial_number_or_name: 825312073829, width: 640, height: 480, fps: 30}, side: {type: intelrealsense, serial_number_or_name: 419122271321, width: 640, height: 480, fps: 30}}" \
  --policy.input_features='{"observation.images.front":  {"type":"VISUAL","shape":[3,256,256]}, "observation.images.side": {"type":"VISUAL","shape":[3,256,256]}}' \
  --policy.empty_cameras=1 \
  --display_data=false \
  --dataset.repo_id=sajibpra/eval_cube_pick_place \
  --dataset.single_task="Put cube into the box" \
  --dataset.num_episodes=10 \
  --dataset.episode_time_s=999 \
  --policy.path=sajibpra/my_smolvla_local \
  --policy.push_to_hub=false
```

Place the cube in front of the robot. If everything worked, the robot should pick it up and place it into the box. Watch the video below.

![video](</attachment/SO-101%20Train%20SmolVLA%20model%20on%20CPU%20(2x).mp4>)

## Key Takeaways
