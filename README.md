# TET-harjoitteluhakemus – Viikko 13 (23.–29.3.2026)

In this TET week, students practice using an Arm robot. We will use the **SO-101 robot** and [SmolVLA](https://arxiv.org/pdf/2506.01844) for a _pick and place_ Imitation training task. The goal is to learn how a robot learns and perform the task dynamically, and how Vision Language Action (VLA) model works.

# Learning Outcomes

After sucessfully completing all the tasks, the student:

- Understands basic robot operation
- Can perform a pick and place task
- Gains basic experience in machine training
- Improves logical thinking and problem-solving skills
- Hands-on experiement with 3D printing

# Prerequisites

- Basic knowledge in programming
- Basic knowledge in Linux commands
- No previous robot experience required

# Tasks

0. [Setup & Installation](#0-setup--installation)
1. [3D printing the cube](#1-3d-printing-the-cube)
2. [Calibration the robot](#2-calibration-the-robot)
3. [Teleoperate](#3-teleoperate)
4. [Record a dataset](#4-record-a-dataset)
5. [Train the model](#5-train-the-model)
6. [Run the model on the robot real-time](#6-run-the-model-on-the-robot-real-time)

## 0. Setup & Installation

### Hardware setup

1. Power on the SO-101 robot
2. Connect the SO-101 leader robot to Laptop
3. Connect the SO-101 follwer robot to Laptop
4. Connect the Front and Side Realsense cameras to the Laptop

### Dependencis

Step 1: Install miniforge

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

Step 2: Environment Setup and Active conda

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

Step 4. Install SmolVLA

```bash
pip install -e ".[smolvla]"
```

<details>
  <summary>Troubleshooting</summary>

If there is any building error apears then install the following dependencis on your device.

```bash
sudo apt-get install cmake build-essential python3-dev pkg-config libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev libswresample-dev libavfilter-dev
```

</details>

## 1. 3D printing the cube

Download the [3D cube](/3D_cube.stl) and print it. It will be used to train the robot pick and place. The [3D container box](/3D_container_box.stl) is also available to download. **Tips:** print the the cube using different color of the materials.

## 2. Calibration the robot

Our follower robot is connected on the port _/dev/ttyACM1_ and leader is on _/dev/ttyACM0_

Follower

```bash
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_awesome_follower_arm
```

First you need to move the robot to the position where all joints are in the middle of their ranges. Then after pressing enter you have to move each joint through its full range of motion.
Leader.

```bash
lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_awesome_leader_arm
```

First you need to move the robot to the position where all joints are in the middle of their ranges. Then after pressing enter you have to move each joint through its full range of motion.
Leader.

[Calibration video](https://huggingface.co/docs/lerobot/en/so101#calibration-video)

Now all set, next we will play with robots using Teleoperate!

## 3. Teleoperate

We have two Realsense cameras, one is in front and another one is mounted on the robot wrist roll. Run the following command which will active teleopration and visualization on `rerun` simultaneously.

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

Here front realsense camera `serial_number_or_name: 825312073829` and wrist roll camera `serial_number_or_name: 419122271321`.

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

Now you can use the Leader and follwer will do the same things. Check the `rerun` dashboard and Observe the robot joints positions.

Great! Now we know how the robot joints positions moves and how to teleoperate.

## 4. Record a dataset

We will record the dataset to train our policy which will run in real-time at the end. Before you record the dataset please review this dataset [cube-pick-place](https://huggingface.co/spaces/lerobot/visualize_dataset?path=%2Fsajibpra%2Fcube-pick-place%2Fepisode_0), to understand how to collect a good data to train a model.

At least **Episodes:** 50, with**FPS:** 30 would be good be receommend.

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

## 5. Train the model

## 6. Run the model on the robot real-time

## Key Takeaways

- Robots move based on programs and coordinates
- Safety is always the most important rule
- Programming requires precision
- Small changes can affect the result
- Robotics combines math, physics, and engineering
