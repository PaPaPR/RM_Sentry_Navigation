# ROS distribution to use
ARG ROS_DISTRO=noetic

##############
# Base Image #
##############
FROM osrf/ros:noetic-desktop-full-focal as base
ENV ROS_DISTRO=${ROS_DISTRO}
SHELL ["/bin/bash", "-c"]

# Install basic apt packages
RUN apt-get update && apt-get install -y --no-install-recommends \
  git libeigen3-dev ros-${ROS_DISTRO}-foxglove-bridge \
  python-is-python3 \
  && rm -rf /var/lib/apt/lists/*

# Create workspace with external dependencies
RUN mkdir -p /underlay_ws/src
WORKDIR /underlay_ws/src

# Reduce "TF_REPEATED_DATA" warnings
WORKDIR /underlay_ws/src
RUN git clone --branch throttle-tf-repeated-data-error --depth 1 https://github.com/BadgerTechnologies/geometry2.git
WORKDIR /underlay_ws/src/geometry2/tf2/src
RUN sed -i '/if (error_str && new_data.stamp_ > last_repeated_warn_time_ + ros::Duration(10.0))/s/(10.0)/(1000.0)/' ./cache.cpp

# TF Transform Republisher
WORKDIR /underlay_ws/src
RUN git clone https://github.com/firephinx/tf_transform_republisher --depth 1
WORKDIR /underlay_ws/src/tf_transform_republisher/src
RUN sed -i '/tb_.sendTransform(tf::StampedTransform(identity_transform_/s/^/\/\//' ./tf_transform_republisher.cpp

# Build the base workspace, installing dependencies first.
WORKDIR /underlay_ws
RUN source /opt/ros/${ROS_DISTRO}/setup.bash \
 && catkin_make

# Remove display warnings
RUN mkdir /tmp/runtime-root
ENV XDG_RUNTIME_DIR "/tmp/runtime-root"
RUN chmod -R 0700 /tmp/runtime-root
ENV NO_AT_BRIDGE 1

# Set up the entrypoint
WORKDIR /underlay_ws
COPY ./.devcontainer/entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT [ "/entrypoint.sh" ]

#################
# Overlay Image #
#################
FROM base as overlay

# Install addictional apt packages
RUN apt-get update && apt-get install -y --no-install-recommends \
  git libeigen3-dev ros-${ROS_DISTRO}-pcl-conversions \
  libgoogle-glog-dev libignition-math4-dev \
  libusb-dev \
  && rm -rf /var/lib/apt/lists/*

# Install addictional ROS packages
WORKDIR /underlay_ws/src

# Livox-ROS-Driver
RUN git clone https://github.com/Livox-SDK/livox_ros_driver.git --depth 1

# Point-LIO
RUN git clone https://github.com/hku-mars/Point-LIO.git --depth 1 --branch point-lio-with-grid-map
WORKDIR /underlay_ws/src/Point-LIO
RUN git submodule update --init --recursive

# Livox Laser Simulator
RUN git clone https://github.com/engcang/livox_laser_simulation.git --depth 1
RUN sed -i '/<build_depend>tf<\/build_depend>/a <build_depend>livox_ros_driver<\/build_depend>' livox_laser_simulation/package.xml
RUN sed -i '/tfBroadcaster->sendTransform(/s/^/\/\//' livox_laser_simulation/src/livox_points_plugin.cpp
RUN sed -i '/tf::StampedTransform/s/^/\/\//' livox_laser_simulation/src/livox_points_plugin.cpp

# RMUA2022 Simulator
RUN git clone https://github.com/XDUZero2Hero/zero2hero_rmua2022_simulation.git --depth 1

# Autonomous Exploration Development Environment
RUN git clone https://github.com/HongbiaoZ/autonomous_exploration_development_environment.git --depth 1

# Far Planner
RUN git clone https://github.com/MichaelFYang/far_planner.git --depth 1

WORKDIR /underlay_ws
RUN source /opt/ros/${ROS_DISTRO}/setup.bash \
 && catkin_make

# Create an overlay workspace
RUN mkdir -p /overlay_ws/src
WORKDIR /overlay_ws
COPY ./sentry_bringup/ ./src/sentry_bringup/
COPY ./sentry_common/ ./src/sentry_common/
COPY ./sentry_description/ ./src/sentry_description/
RUN source /opt/ros/${ROS_DISTRO}/setup.bash \
 && catkin_make

# Set up the entrypoint
# COPY ./entrypoint.sh /
ENTRYPOINT [ "/entrypoint.sh" ]

#####################
# Development Image #
#####################
FROM overlay as dev

# Dev container arguments
ARG USERNAME=devuser
ARG UID=1000
ARG GID=${UID}

# Install extra tools for development
RUN apt-get update && apt-get install -y --no-install-recommends \
 gdb gdbserver nano

# Create new user and home directory
RUN groupadd --gid $GID $USERNAME \
 && useradd --uid ${GID} --gid ${UID} --create-home ${USERNAME} \
 && echo ${USERNAME} ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/${USERNAME} \
 && chmod 0440 /etc/sudoers.d/${USERNAME} \
 && mkdir -p /home/${USERNAME} \
 && chown -R ${UID}:${GID} /home/${USERNAME}

# Set the ownership of the overlay workspace to the new user
RUN chown -R ${UID}:${GID} /overlay_ws/

# Set the user and source entrypoint in the user's .bashrc file
USER ${USERNAME}
# RUN echo "source /entrypoint.sh" >> /home/${USERNAME}/.bashrc
RUN echo -e "source /opt/ros/${ROS_DISTRO}/setup.bash\n \
  source /underlay_ws/devel/setup.bash\n \
  source /overlay_ws/devel/setup.bash" >> /home/${USERNAME}/.bashrc
ENTRYPOINT [ "/entrypoint.sh" ]