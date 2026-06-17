# sherec_semantic_slam (docker)

Docker image for [**DPS-SLAM**](https://github.com/perezsaura-david/dps_slam) (Dual Pose-graph
Semantic SLAM for AeroStack2). The image is **self-contained**: the dockerfile clones and builds the
whole colcon workspace — `dps_slam`, `dps_slam_msgs`, and `struct_slam` — so there's nothing to build
by hand. You just bring it up and run an experiment.

The image is ROS 2 Humble + Gazebo Harmonic and installs g2o/ceres from apt
(`ros-humble-libg2o`, `libceres-dev`, `libsuitesparse-dev`, `libgoogle-glog-dev`).

## Contents

| File | Purpose |
|------|---------|
| `dockerfile` | The image: base `aerostack2/nightly-humble`, Gazebo Harmonic, g2o/ceres deps; clones + `colcon build`s the workspace into `/root/workspace`. |
| `docker-compose.yml` | Project `sherec_semantic_slam`; runs the container, bind-mounts `launchers/` → `/root/launchers` and `rosbags/` → `/root/rosbags`. |
| `launchers/` | Run config: tmuxinator session (`session.yml`), `config.yaml`, `rviz.rviz`, `launcher.sh`. |

## Prerequisites (host)

- Docker + Docker Compose v2
- [NVIDIA Container Toolkit][nvidia-toolkit] (the compose file uses `runtime: nvidia` — drop that
  line if you have no NVIDIA GPU)
- A running **SSH agent** with a key that can clone `struct_slam`
  (`ssh-add -l` should list a key). The build forwards it via `build.ssh`.
- An X server (`$DISPLAY` / `$XAUTHORITY` set) for RViz.

## Usage (Docker)

### 1. Build the image

The workspace is cloned and compiled during the build, so this step needs your SSH agent (for the
`struct_slam` clone) — compose forwards it automatically via `build.ssh: [default]`.

```bash
cd docker
ssh-add -l            # confirm a key is loaded
docker compose build
```

### 2. Run an experiment

```bash
# allow the container to talk to your X server (once per login)
xhost +local:root

# drop your rosbag in ./rosbags/ (mounted at /root/rosbags) and point ROSBAG_FILE
# at it in launchers/session.yml (it must be a specific bag, not the directory), then:
docker compose up -d
docker exec -it sherec_semantic_slam bash

# inside the container — launch the SLAM + detections + rosbag tmux session
cd /root/launchers
./launcher.sh session.yml
```

This starts the tmuxinator session `sherec_semantic_slam`: the `dps_slam` node + static TF, the
detections launch, and `ros2 bag play` alongside RViz.

### 3. Stop

```bash
docker exec -it sherec_semantic_slam tmux kill-session -t sherec_semantic_slam   # stop the experiment
docker compose down                                                              # stop/remove the container
```

[nvidia-toolkit]: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
