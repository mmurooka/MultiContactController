# [MultiContactController](https://github.com/isri-aist/MultiContactController)
Humanoid controller for whole-body multi-contact motion using tactile sensors

**This is a branch dedicated to reproduce the simulation experiments of the paper on whole-body multi-contact motion using tactile sensors (to be published in the future). The features in this branch will be merged into [the origin/master branch](https://github.com/isri-aist/MultiContactController) in the future.**

**For general information about MultiContactController, please refer to [the README in the origin/master branch](https://github.com/isri-aist/MultiContactController). The following describes only the information specific to whole-body multi-contact motion using tactile sensors.**

[![CI](https://github.com/mmurooka/MultiContactController/actions/workflows/docker.yaml/badge.svg)](https://github.com/mmurooka/MultiContactController/actions/workflows/docker.yaml)
[![Docker](https://img.shields.io/badge/Docker%20image-ready-blue)](https://github.com/mmurooka/MultiContactController/pkgs/container/multi_contact_controller)

## Procedure for reproducing simulation experiments

Please refer to [the README in the origin/master branch](https://github.com/isri-aist/MultiContactController) for instructions on how to install from source.
Currently, Ubuntu 20.04 / ROS1 environment is assumed, but Ubuntu 22.04 / ROS2 environment will be supported soon.

In the following, how to reproduce simulation experiments using Docker is explained.
This allows simulation experiments to be reproduced in a variety of environments.

1. (Skip if Docker is already installed.) Install Docker. See [here](https://docs.docker.com/engine/install) for details.
```console
$ sudo apt-get install ca-certificates curl gnupg lsb-release
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

To enable GPUs in Docker (i.e., enable the `--gpus` option in the `docker` command), install `nvidia-docker2`.
See [here](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.10.0/install-guide.html) for details.

2. Execute the following commands to start MuJoCo and Rviz, and the robot will start motion; close the MuJoCo window to exit.
```console
$ docker pull ghcr.io/mmurooka/multi_contact_controller:RAL2024
$ xhost +local:
```

- Walking with the right elbow in contact with the environment.
```console
$ docker run --gpus all --rm -it \
  --env="DISPLAY" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  ghcr.io/mmurooka/multi_contact_controller:RAL2024 ./MotionElbowContactWalk.bash
```
https://github.com/user-attachments/assets/0bd3fa6f-5992-43cc-be2d-aee5a5c6934e

- Standing with the left knee in contact with the environment.
```console
$ docker run --gpus all --rm -it \
  --env="DISPLAY" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  ghcr.io/mmurooka/multi_contact_controller:RAL2024 ./MotionKneeContact.bash
```
https://github.com/user-attachments/assets/97c369f5-e0ea-4199-bc4a-2f76f5b15046

The Docker image is automatically updated on [CI](https://github.com/mmurooka/MultiContactController/actions/workflows/docker.yaml) from [this Dockerfile](https://github.com/mmurooka/MultiContactController/blob/RAL2024/.github/workflows/Dockerfile).
