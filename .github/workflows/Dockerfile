FROM ubuntu:20.04
MAINTAINER m-murooka@aist.go.jp

ARG REPOSITORY=mmurooka
ARG COMMIT_SHA=RAL2024
ENV REPOSITORY ${REPOSITORY}
ENV COMMIT_SHA ${COMMIT_SHA}

# https://qiita.com/haessal/items/0a83fe9fa1ac00ed5ee9
ENV DEBCONF_NOWARNINGS=yes
# https://qiita.com/yagince/items/deba267f789604643bab
ENV DEBIAN_FRONTEND=noninteractive
# https://qiita.com/jacob_327/items/e99ca1cf8167d4c1486d
ENV APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1

# https://stackoverflow.com/a/25423366
SHELL ["/bin/bash", "-c"]

# Install basic packages
RUN apt-get update -qq
RUN apt-get install -y sudo aptitude build-essential lsb-release wget gnupg2 curl emacs
RUN aptitude update -q

# Install ROS
ENV ROS_DISTRO noetic
RUN sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
RUN wget http://packages.ros.org/ros.key -O - | apt-key add -
RUN apt-get update -qq
RUN apt-get install -y ros-${ROS_DISTRO}-ros-base python3-catkin-tools python3-rosdep python3-wstool python3-rosinstall python3-rosinstall-generator doxygen graphviz

# Install mc_rtc
RUN curl -1sLf 'https://dl.cloudsmith.io/public/mc-rtc/stable/setup.deb.sh' | bash
RUN apt-get install -y libmc-rtc-dev mc-rtc-utils ros-${ROS_DISTRO}-mc-rtc-plugin ros-${ROS_DISTRO}-mc-rtc-rviz-panel libeigen-qld-dev
# https://github.com/jrl-umi3218/lipm_walking_controller/blob/b564d655388ae6a6725c504e5c74a62192e58c7c/.github/workflows/build.yml#L69
RUN apt-get install -y mc-state-observation xvfb ffmpeg mesa-utils fluxbox xserver-xorg xserver-xorg-core xserver-xorg-video-all libwayland-egl1-mesa

# Install mc_mujoco
RUN apt-get install -y libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libglew-dev
ENV HOME /root
RUN mkdir ${HOME}/mujoco
WORKDIR ${HOME}/mujoco
ENV MUJOCO_VERSION=2.3.7
RUN wget --quiet https://github.com/deepmind/mujoco/releases/download/${MUJOCO_VERSION}/mujoco-${MUJOCO_VERSION}-linux-x86_64.tar.gz
RUN tar xzf mujoco-${MUJOCO_VERSION}-linux-x86_64.tar.gz
RUN git clone --recursive https://github.com/rohanpsingh/mc_mujoco
WORKDIR ${HOME}/mujoco/mc_mujoco/robots/jvrc_mj_description
RUN git remote add mmurooka https://github.com/mmurooka/jvrc_mj_description && \
    git fetch mmurooka && git checkout -b RAL2024 -t mmurooka/RAL2024 && git branch -vv
RUN mkdir ${HOME}/mujoco/mc_mujoco/build
WORKDIR ${HOME}/mujoco/mc_mujoco/build
RUN cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -DMUJOCO_ROOT_DIR=${HOME}/mujoco/mujoco-${MUJOCO_VERSION}
RUN make & make install

# Enable nvidia
# https://qiita.com/eisoku9618/items/c2cca0f0bf764def2efd
ENV NVIDIA_VISIBLE_DEVICES ${NVIDIA_VISIBLE_DEVICES:-all}
ENV NVIDIA_DRIVER_CAPABILITIES ${NVIDIA_DRIVER_CAPABILITIES:+$NVIDIA_DRIVER_CAPABILITIES,}graphics

# Setup catkin workspace
ENV HOME /root
RUN mkdir -p ${HOME}/catkin_ws/src
WORKDIR ${HOME}/catkin_ws
RUN wstool init src
# To handle CI triggered by Pull Request, COMMIT_SHA must be specified in the git fetch argument,
# but since wstool does not support it, execute the git commands directly instead.
# RUN wstool set -t src MultiContactController https://github.com/${REPOSITORY}/MultiContactController --git -y -v ${COMMIT_SHA}
RUN mkdir -p src/isri-aist/MultiContactController && \
    cd src/isri-aist/MultiContactController && \
    git init && \
    git remote add origin https://github.com/${REPOSITORY}/MultiContactController && \
    git fetch origin ${COMMIT_SHA} && \
    git checkout ${COMMIT_SHA} && \
    git submodule update --init --recursive
RUN wstool merge -t src src/isri-aist/MultiContactController/depends.rosinstall
RUN wstool set -t src isri-aist/MujocoTactileSensorPlugin https://github.com/isri-aist/MujocoTactileSensorPlugin --git -y
RUN wstool update -t src

# Rosdep install
RUN rosdep init
RUN rosdep update
RUN source /opt/ros/${ROS_DISTRO}/setup.bash && rosdep install -y -r --from-paths src --ignore-src

# Catkin build
RUN source /opt/ros/${ROS_DISTRO}/setup.bash && catkin build multi_contact_controller mujoco_tactile_sensor_plugin \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_QLD=ON -DENABLE_CNOID=OFF -DENABLE_MUJOCO=ON -DMUJOCO_ROOT_DIR=${HOME}/mujoco/mujoco-${MUJOCO_VERSION}
RUN echo "source ${HOME}/catkin_ws/devel/setup.bash" >> ${HOME}/.bashrc

# Install jvrc_description
RUN mkdir ${HOME}/src
WORKDIR ${HOME}/src
RUN git clone https://github.com/mmurooka/mc_rtc_data -b wholebody-contact
RUN ln -sf ${HOME}/src/mc_rtc_data/jvrc_description ${HOME}/catkin_ws/src/jvrc_description
WORKDIR ${HOME}/catkin_ws/src
RUN source ${HOME}/catkin_ws/devel/setup.bash && catkin build jvrc_description --no-deps

# Install McRtcTactileSensorPlugin
WORKDIR ${HOME}/src
RUN git clone --recursive https://github.com/isri-aist/McRtcTactileSensorPlugin
RUN mkdir ${HOME}/src/McRtcTactileSensorPlugin/build
WORKDIR ${HOME}/src/McRtcTactileSensorPlugin/build
RUN source ${HOME}/catkin_ws/devel/setup.bash && cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_ESKIN=OFF
RUN source ${HOME}/catkin_ws/devel/setup.bash && make & make install

# Setup simulation
WORKDIR ${HOME}
RUN mkdir -p ${HOME}/.config/mc_rtc/controllers/MultiContactController/plugins
RUN cp ${HOME}/catkin_ws/src/isri-aist/MultiContactController/etc/mc_rtc.yaml ${HOME}/.config/mc_rtc/mc_rtc.yaml
RUN mkdir -p ${HOME}/.config/mc_rtc/mc_mujoco
RUN echo PluginPaths: [\"${HOME}/catkin_ws/devel/.private/mujoco_tactile_sensor_plugin/lib\"] > ${HOME}/.config/mc_rtc/mc_mujoco/mc_mujoco.yaml
RUN echo $'#!/usr/bin/env bash \n\
source ${HOME}/catkin_ws/devel/setup.bash \n\
ln -sf ${HOME}/catkin_ws/src/isri-aist/MultiContactController/config/MotionElbowContactWalk.yaml ${HOME}/.config/mc_rtc/controllers/MultiContactController.yaml \n\
ln -sf ${HOME}/catkin_ws/src/isri-aist/MultiContactController/config/plugins/TactileSensor/MotionElbowContact.yaml ${HOME}/.config/mc_rtc/controllers/MultiContactController/plugins/TactileSensor.yaml \n\
roscore > /dev/null 2>&1 < /dev/null & \n\
ROSCORE_PID=$! \n\
sleep 1s \n\
roslaunch multi_contact_controller display_motion_elbow_contact_walk.launch > /dev/null 2>&1 < /dev/null & \n\
RVIZ_PID=$! \n\
sleep 1s \n\
mc_mujoco --without-mc-rtc-gui --sync \n\
kill -2 ${ROSCORE_PID} \n\
kill -2 ${RVIZ_PID}' > ${HOME}/MotionElbowContactWalk.bash
RUN chmod 755 ${HOME}/MotionElbowContactWalk.bash
RUN echo $'#!/usr/bin/env bash \n\
source ${HOME}/catkin_ws/devel/setup.bash \n\
ln -sf ${HOME}/catkin_ws/src/isri-aist/MultiContactController/config/MotionKneeContact.yaml ${HOME}/.config/mc_rtc/controllers/MultiContactController.yaml \n\
ln -sf ${HOME}/catkin_ws/src/isri-aist/MultiContactController/config/plugins/TactileSensor/MotionKneeContact.yaml ${HOME}/.config/mc_rtc/controllers/MultiContactController/plugins/TactileSensor.yaml \n\
roscore > /dev/null 2>&1 < /dev/null & \n\
ROSCORE_PID=$! \n\
sleep 1s \n\
roslaunch multi_contact_controller display_motion_knee_contact.launch > /dev/null 2>&1 < /dev/null & \n\
RVIZ_PID=$! \n\
sleep 1s \n\
mc_mujoco --without-mc-rtc-gui --sync \n\
kill -2 ${ROSCORE_PID} \n\
kill -2 ${RVIZ_PID}' > ${HOME}/MotionKneeContact.bash
RUN chmod 755 ${HOME}/MotionKneeContact.bash
