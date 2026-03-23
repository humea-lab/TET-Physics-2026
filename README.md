# TET-harjoittelu 25.–26.3.2026

During this TET internship, students will work with an articulated robotic arm system. The internship focuses on imitation learning using the **SO-101 robotic arm** and the [SmolVLA](https://arxiv.org/pdf/2506.01844) (Vision-Language-Action) model to perform a pick-and-place task. Students will participate in system setup, data collection, model training, and evaluation of autonomous robot behavior.

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
- You can follow instructions and copy-paste commands carefully
- Basic [Linux command-line](https://ubuntu.com/tutorials/command-line-for-beginners#1-overview) knowledge is helpful.

# Beginner Notes (Read First)

- Open a terminal and run commands one by one.
- Copy commands exactly. A small typo can break a step.
- Wait for one command to finish before running the next command.
- If a command shows an error, read the message and ask the instructor.
- Keep this README open while working.

How to read command placeholders:

- Text like `<your_hugging_face_user_name>` means you must replace it with your own value.
- Text like `${HUGGINGFACE_TOKEN}` means paste your own token there.
- Do not type `<` or `>` when replacing values.

# Tasks

0. [Setup & Installation](#task-0-setup--installation)
1. [3D printing the cube](#task-1-3d-printing-the-cube)
2. [Calibrate the robot](#task-2-calibrate-the-robot)
3. [Teleoperate](#task-3-teleoperate)
4. [Record a dataset](#task-4-record-a-dataset)
5. [Train the model](#task-5-train-the-model)
6. [Run the model on the robot real-time](#task-6-run-the-model-on-the-robot-real-time)

## Task 0. Setup & Installation

### Hardware setup

> [!WARNING]
> Safety first:
> Keep fingers away from moving robot joints.
> Do not force robot joints by hand while the robot is powered.
> Keep cables tidy so they do not get pulled during movement.

1. Power on the SO-101 robot
2. Firstly, Connect the SO-101 **leader** robot to the PC
3. Then Connect the SO-101 **follower** robot to the PC
4. Lastly, Connect the front and side Intel RealSense cameras to the PC

### Dependencies

**Step 1: Install Miniforge**

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

After installation, close and reopen the terminal.

Quick check:

```bash
conda --version
```

If you see a version number, Miniforge is ready.

<details>
  <summary>What is Miniforge? Why we install it?</summary>

Miniforge is a small tool that installs Conda. Conda helps us create a clean Python environment for our project.

We install Miniforge so you can use the same Python version and the same packages. This makes the setup easier and helps avoid errors.

In short, Miniforge keeps the project organized and makes the commands work more reliably on different computers.

</details>

**Step 2: Create and activate a Conda environment**

```bash
conda create -y -n lerobot python=3.10
conda activate lerobot
conda install ffmpeg -c conda-forge
```

Quick check:

```bash
conda info --envs
```

You should see an environment called `lerobot`.

<details>
  <summary>What is a Conda environment? Why do we create it?</summary>

A Conda environment is a separate workspace for one project.

We create it so this robot project has its own Python (version 3.10) and its own packages.

This helps beginners because commands work the same way for everyone, and it prevents conflicts with other projects on your computer.

In this step:
`conda create` makes the environment,
`conda activate` turns it on,
and `conda install ffmpeg` adds a tool needed for video and data processing.

</details>

**Step 3: Install LeRobot**

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e .

```

<details>
  <summary>What is LeRobot framework? Why we install it?</summary>

LeRobot is a framework made by Hugging Face to teach robots with data.

In this project, we use LeRobot to control the robot, record demonstrations, train a model, and test if the robot can do the task by itself.

We install LeRobot because it gives us ready-made tools for every step, so beginners can follow one clear workflow.

</details>

**Step 4: Install SmolVLA**

```bash
pip install -e ".[smolvla]"
```

<details>
  <summary>What is SmolVLA? Why do we install it?</summary>

SmolVLA is the AI model we use to teach the robot from examples.

After we install LeRobot, we install SmolVLA so training and robot prediction commands can work. In simple words: LeRobot is the toolbox, and SmolVLA is the learning brain inside this project.

</details>

<details>
  <summary>Troubleshooting</summary>

If you see build errors, install the following dependencies:

```bash
sudo apt-get install cmake build-essential python3-dev pkg-config libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev libswresample-dev libavfilter-dev
```

</details>

## Task 1. 3D printing the cube

Download the [3D cube](/3D_cube.stl) and print it. We will use it to train the robot for pick-and-place. The [3D container box](/3D_container_box.stl) is also available.

Tip: try printing the cube in a different color than the box. It often makes training easier.

## Task 2. Calibrate the robot

In our setup, the follower robot is connected to `/dev/ttyACM1` and the leader is connected to `/dev/ttyACM0`.

Before calibration, verify device ports:

```bash
ls /dev/ttyACM*
```

If the port numbers are different on your PC, replace them in the commands below.

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

[How to calibrate the robot (video instruction)](https://huggingface.co/docs/lerobot/en/so101#calibration-video)

Nice! Calibration is done. Next, we will play with the robots using teleoperation.

## Task 3. Teleoperate

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

Great work! This is the core skill you’ll need for collecting a good dataset.

## Task 4. Record a dataset

We will use the Hugging Face Hub to upload the dataset. Log in to Hugging Face and generate a token from [Hugging Face settings](https://huggingface.co/settings/tokens).

Run the following command and **replace** `${HUGGINGFACE_TOKEN}` with your generated token.

```bash
huggingface-cli login --token ${HUGGINGFACE_TOKEN} --add-to-git-credential
```

You can create a token here: [https://huggingface.co/settings/tokens](https://huggingface.co/settings/tokens).

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

Run the following command to record your dataset. **Note:** Replace `sajibpra` with `<your_hugging_face_user_name>` and `cube-pick-place` with your repository name.

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

## Task 5. Train the model

Now that you have a dataset, it’s time to train the model. We will use [SmolVLA](https://huggingface.co/lerobot/smolvla_base) as the base model.

Before we start training, go to [Weights & Biases (wandb)](https://wandb.ai/home), log in, and generate an API key. Copy the API key in a note, you will require it in the next step.

In the terminal, run:

```bash
wandb login
```

It will ask for the API key—paste the one you generated.

Now run the following command to start training. **Note:** Replace `sajibpra` with your `<your_hugging_face_user_name>` and `cube-pick-place` with your repository name.

```bash
cd ~/lerobot/

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

- **GPU:** NVIDIA GeForce RTX 4070 Ti SUPER
- **VRAM:** 16GB
- **RAM:** 32GB

If your PC is slower, training can take much longer. That is normal.

After completing training, upload your trained model to Hugging Face. **Note:** Replace `sajibpra` with your `<your_hugging_face_user_name>`.

```bash
huggingface-cli upload sajibpra/my_smolvla_local \
  outputs/train/my_smolvla_local/checkpoints/last/pretrained_model
```

Awesome! You have trained the model and uploaded it to the Hugging Face Hub.

## Task 6. Run the model on the robot real-time

You have reached the final task: running the model on the robot in real time.

The [hardware setup](#hardware-setup) is the same as when you recorded the data. This time, you do not need the leader robot - only the follower will perform the task autonomously.

Run the following command. **Note:** Replace `sajibpra` with `<your_hugging_face_user_name>`.

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

Place the cube in front of the robot. If everything worked perfectly, the robot should pick it up and place it into the box. Now you can evaluate the model. Watch the video below.

https://github.com/user-attachments/assets/ec329be0-bca1-4899-8f53-4b51be944744

Congrats 🎉! You have successfully completed all 6 tasks.

## References

1. Hugging Face, [LeRobot: Getting Started with a Real-World Robot](https://huggingface.co/docs/lerobot/main/en/getting_started_real_world_robot)

2. Hugging Face, [SmolVLA Base (Model Card)](https://huggingface.co/lerobot/smolvla_base)
