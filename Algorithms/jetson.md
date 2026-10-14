Jetson's default operating system (JetPack / L4T) includes **USB Device Mode**. When connected via USB to a computer, it automatically spins up a virtual network interface with a fixed IP address, as well as a serial COM port for terminal access.

1. Setup
https://www.reddit.com/r/JetsonNano/comments/x0fif4/how_to_network_jetson_nano_with_a_laptop_via/
https://jetsonhacks.com/2019/08/21/jetson-nano-headless-setup/ 
reconfiguring a jetson nano with oem - config

virtual USB ethernet over Jetson's USB automatically

```
sudo apt-get install screen

ls /dev/ttyACM*
sudo screen /dev/ttyACM0 115200

ifconfig

ssh szyfrowac@192.168.55.1

```

ubuntu login 
```
whoami

nmcli connection show --active
```

to scroll in screen : ctrl + A, "[", then j to scroll up and k to scroll down 

wifi --> network --> IPV4 --> share with other computers 

kill window: ctrl + A, k, y

kill : 
```
sudo killall screen
sudo fuser -k /dev/ttyACM0
```

----
1. Installing QGC with flatpak (WORKED)
https://flathub.org/en/apps/org.mavlink.qgroundcontrol
https://flathub.org/en/setup/Ubuntu

```
sudo apt install flatpak

#add repo
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak install --user -y flathub org.kde.Platform//5.15-24.08 org.kde.Sdk//5.15-24.08

git clone https://github.com/flathub/org.mavlink.qgroundcontrol.git 

cd org.mavlink.qgroundcontrol

flatpak-builder --user --install --force-clean build-dir org.mavlink.qgroundcontrol.yml

#cause i had some error
flatpak-builder --user --install --finish-only build-dir org.mavlink.qgroundcontrol.yml

#confirm install (ill jump from sbox second floor if this shit doesn't work)
flatpak list --app

#run QGC
flatpak run --user org.mavlink.qgroundcontrol
```

![[Pasted image 20260914214152.png]]

1. Building QGC from source with jetpack + installing Qt5 /Qt6
build QGC natively or with cross compilation 
- Qt6 with jetpack 6.2
- or with flatpak

```
sudo apt-cache show nvidia-jetpack
```

a. install all by hand 
```
#dependancies
sudo apt update && sudo apt install -y \
  build-essential git ccache cmake ninja-build \
  qt6-base-dev qt6-base-private-dev \
  qt6-declarative-dev qt6-declarative-private-dev \
  qt6-location-dev qt6-positioning-dev \
  libqt6serialport6-dev libqt6svg6-dev libqt6charts6-dev \
  libqt6texttospeech6-dev qt6-multimedia-dev \
  libqt6opengl6-dev \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
  libsdl2-dev
```

clone repo
```
# qt 6 : master
cd ~
git clone --branch v4.4.0 --recurse-submodules https://github.com/mavlink/qgroundcontrol.git

# qt 5 : stable
git clone --branch v4.3.0 --recursive https://github.com/mavlink/qgroundcontrol.git

cd qgroundcontrol
# verify submodules intact
git submodule update --init --recursive
```
b. clone repo, then

```
# better
cd ~/qgroundcontrol/tools/setup
sudo python3 -m install_dependencies
#or
python3 tools/setup/install-dependencies-debian.py

#dryrun to test
python3 tools/setup/install-dependencies-debian.py --dry-run

cd ~/tools/setup
python3 read_config.py

cmake --version
just --version
```

![[Pasted image 20260914024621.png]]

https://docs.qgroundcontrol.com/master/en/qgc-dev-guide/getting_started/
- https://github.com/mavlink/qgroundcontrol:
	- https://github.com/mavlink/qgroundcontrol/blob/master/tools/setup/install_dependencies/_debian.py : DEBIAN_PACKAGES installer code
	- https://github.com/mavlink/qgroundcontrol/blob/master/tools/setup/install_dependencies/_packages.py : list of DEBIAN_PACKAGES to install before installing QGC

cmake build QGC
- create swap file with 8 GB 

every time cmake fails
```
cd ~/qgroundcontrol
git checkout v4.3.0
git submodule sync --recursive
git submodule update --init --recursive --force
rm -rf build
```

3. Building QGC with app image

```
cd ~
wget https://github.com/mavlink/qgroundcontrol/releases/latest/download/QGroundControl-aarch64.AppImage

ls -lh QGroundControl-aarch64.AppImage

chmod +x QGroundControl-aarch64.AppImage
sudo usermod -a -G dialout $USER
```

---
not sending packets
**on thinkpad

```
# 1. Allow the Linux kernel to route packets between interfaces
sudo sysctl -w net.ipv4.ip_forward=1

# 2. Allow forwarded traffic through the laptop's firewall
sudo iptables -P FORWARD ACCEPT
sudo iptables -F FORWARD

# 3. Masquerade outbound traffic leaving your active Wi-Fi interface
WIFI_IF=$(ip route get 8.8.8.8 | awk '{print $5; exit}')
sudo iptables -t nat -A POSTROUTING -o "$WIFI_IF" -j MASQUERADE

#ethernet interface is up and what ip
ip addr

#default gateway
ip route
#enp0s31f6 --> UP

#name of connections
nmcli connection show
#wired 3

sudo ip route add default via 192.168.55.1

sudo nmcli connection modify "Wired connection 3" ipv4.method shared
sudo nmcli connection up "Wired connection 3"
```

**on jetson

```
sudo ip route replace default via 10.42.3.1 dev enP8p1s0
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

ping -c 3 8.8.8.8 
#google public DNS


```

```
#jetson has no wifi module
ip link show wlan0

ncmli dev wifi list

#to search for ethernet 
ip -br link

#to see what IP gateway network is on (for my ethernet)
ip addr show enP8p1s0

#check routing table
ip route show

#UP or DOWN
ip link show

#host IP
hostname -I
```
![[Pasted image 20260915040700.png]]
---
RUNNING QGC ON LENOVO GUI
```
#on jetson
exit

ssh -Y szyfrowac@<jetson-ip>

#display fwding
echo $DISPLAY
flatpak run --user --socket=x11 org.mavlink.qgroundcontrol

```

---
MAVLINK ROUTER (WORKED)
```
sudo apt install -y git meson ninja-build
git clone https://github.com/mavlink-router/mavlink-router.git
cd mavlink-router
git submodule update --init --recursive
meson setup build
ninja -C build
sudo ninja -C build install
```

QGC on laptop 22.04 (WORKED)

```
cd ~
curl -L -o QGroundControl.AppImage https://github.com/mavlink/qgroundcontrol/releases/download/v4.4.3/QGroundControl.AppImage

chmod +x QGroundControl.AppImage
./QGroundControl.AppImage
```


---
CONNECT
laptop IP 
```
hostname -I | awk '{print $1}'
```

----
Install ROS2 HUMBLE
1. ISAAC ROS 2 for 22.04 Humble
https://nvidia-isaac-ros.github.io/v/release-3.2/getting_started/index.html
- install with python 3.10
- don't run on bare metal, run in docker (3.11 versus 3.10 package incompatibility)
```
cat /etc/nv_tegra_release

#gpu cpu clock to max
sudo /usr/bin/jetson_clocks
#power to max
sudo /usr/sbin/nvpmodel -m 0

```

- docker and nvidia toolkit
```
#install git lfs to pull larger files

#Install docker
sudo apt-get update
sudo apt-get install -y docker.io nvidia-container-toolkit git-lfs

sudo nvidia-ctk runtime configure --runtime=docker

#restard docker
sudo systemctl daemon-reload && sudo systemctl restart docker

#add user to docker grp
sudo usermod -aG docker $USER
newgrp docker

#verify 
exec su -l $USER

```

- ros 2 workspace (WORKED) : 
https://www.youtube.com/watch?v=rF0eilKFy40 (has all links in desc)
https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common
https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_common/index.html
```
#mkdir -p  /mnt/nova_ssd/workspaces/isaac_ros-dev/src (not run cause of root permissions)

#create directory
mkdir -p ~/workspaces/isaac_ros-dev/src 
cd ~/workspaces/isaac_ros-dev/src

echo "export ISAAC_ROS_WS=/mnt/nova_ssd/workspaces/isaac_ros-dev/" >> ~/.bashrc
source ~/.bashrc

#enter source directory and clone repo
git clone https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common.git

#verify that run_dev.sh is there
ls -l isaac_ros_common/scripts/run_dev.sh
```

ALWAYS REMEMBER TO RUN IN THE DOCKER CONTAINER (don't fuck up the host)
- launch
```
cd isaac_ros_common
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev
```

- error: THE FUCKING ERROR WAS THAT MY TERMINAL WAS NOT RENDERING GREEN WTFFF
```
cd ~/workspaces/isaac_ros-dev/src/isaac_ros_common
export TERM=xterm-256color
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev

echo $ROS_DISTRO
#humble
```

BUILDING FROM THERE
	system specs: Jetson orin nano : jetpack 6.2 : ubuntu 22.04, CUDA 12.6, VPI 3.2.4, Ros 2 Humble in Isaac Ros docker container, Isaac ros 3.2 (all repos are also 3.2), Cmake 3.28.4, rosdep clean, sequential colcon

```
cd ~/workspaces/isaac_ros-dev/src/isaac_ros_common
export TERM=xterm-256color
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev

source /opt/ros/humble/setup.bash

export PATH="$HOME/.local/bin:$PATH" cmake --version 
#3.28.4

nvcc --version 
#CUDA 12.6.x

echo $ROS_DISTRO 
#humble

grep -E 'NV_VPI_VERSION_(MAJOR|MINOR|PATCH)' \ /usr/include/vpi/VersionDef.h #VPI 3.2.4

rosdep update
rosdep check --from-paths src --ignore-src
rosdep install --from-paths src --ignore-src
# Expected: # All system dependencies have been satisfied

colcon build --symlink-install

#or better

colcon build \
    --symlink-install \
    --executor sequential \
    --parallel-workers 1

source install/setup.bash

colcon list
```

```
2. Librealsense for the isaac_ros_container
#USB 3 --> JETSON --> ISAAC ROS DOCKER CONTAINER --> REALSENSE2_CAMERA --> ROS TOPICS --> HOST RTAB MAP

cd ~/workspaces/isaac_ros-dev/src/isaac_ros_common
export TERM=xterm-256color
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev


export PATH=/opt/librealsense/bin:$PATH
export LD_LIBRARY_PATH=/opt/librealsense/lib:${LD_LIBRARY_PATH:-}


# Check the librealsense version visible to the container
grep realsense2_VERSION \
    /opt/librealsense/lib/cmake/realsense2/realsense2Config.cmake

# Expected:
# 2.58.4

#build realsense2_camera against librealsense 2.58.4

cd /workspaces/isaac_ros-dev

rm -rf build/realsense2_camera
rm -rf install/realsense2_camera

source /opt/ros/humble/setup.bash

colcon build \
    --packages-select realsense2_camera \
    --symlink-install \
    --cmake-clean-cache \
    --cmake-args \
        -Drealsense2_DIR=/opt/librealsense/lib/cmake/realsense2

source install/setup.bash

#verify ros wrapper uses realsense2_camera

ros2 pkg prefix realsense2_camera

ldd \
    /workspaces/isaac_ros-dev/install/realsense2_camera/lib/librealsense2_camera.so \
    | grep librealsense

# Expected:
# lib...librealsense2.so.2.58 => /opt/librealsense/lib/librealsense2.so.2.58

#verify i can D435 through USB 3.0 in the container

export PATH=/opt/librealsense/bin:$PATH
export LD_LIBRARY_PATH=/opt/librealsense/lib:${LD_LIBRARY_PATH:-}

#check for realsense sdk
echo "realsense SDK"
grep realsense2_VERSION \
    /opt/librealsense/lib/cmake/realsense2/realsense2Config.cmake

# Check camera
echo "Camera"
/opt/librealsense/bin/rs-enumerate-devices | head -15

#START REALSENSE ROS NODE
#RGB + DEPTH ONLY

source /opt/ros/humble/setup.bash
source /workspaces/isaac_ros-dev/install/setup.bash

#syntax : export LD_LIBRARY_PATH=/path (no spaces)

export PATH=/opt/librealsense/bin:$PATH
export LD_LIBRARY_PATH=/opt/librealsense/lib:${LD_LIBRARY_PATH:-}

ros2 launch realsense2_camera rs_launch.py \
    enable_color:=true \
    enable_depth:=true \
    enable_gyro:=false \
    enable_accel:=false

```

```
#TO RUN IT
cd ~/workspaces/isaac_ros-dev/src/isaac_ros_common
export TERM=xterm-256color
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev

source /opt/ros/humble/setup.bash
source /workspaces/isaac_ros-dev/install/setup.bash

export PATH=/opt/librealsense/bin:$PATH
export LD_LIBRARY_PATH=/opt/librealsense/lib:${LD_LIBRARY_PATH:-}

ros2 launch realsense2_camera rs_launch.py \
    enable_color:=true \
    enable_depth:=true \
    rgb_camera.color_profile:=640x480x30 \
    depth_module.depth_profile:=640x480x30 \
    enable_infra1:=false \
    enable_infra2:=false \
    enable_gyro:=false \
    enable_accel:=false


#in diff terminal
source /opt/ros/humble/setup.bash

ros2 topic list | grep camera


#host
ros2 launch rtabmap_launch rtabmap.launch.py \
    rgb_topic:=/camera/camera/color/image_raw \
    depth_topic:=/camera/camera/depth/image_rect_raw \
    camera_info_topic:=/camera/camera/color/camera_info \
    frame_id:=camera_link \
    approx_sync:=true \
    rtabmap_viz:=true
    
source /opt/ros/humble/setup.bash
rviz2
```

```
3. apt installation
lite shit
----
RUN ROS ON WEBCAM

- in this use case: gstreamer on lenovo --> gscam on jetson
- in actual arducam IMX219 stream : isaac_ros_argus_camera

```

```
ros2 --help
echo $ROS_DISTRO
nvcc --version
nvidia-smi

- repo im installing : https://github.com/ros-drivers/usb_cam
  
#check webcam
ls -l /dev/video*
```

---
# See the camera hardware details and bus connection
```
v4l2-ctl --list-devices

#for my thinkpad rgb cam
v4l2-ctl -d /dev/video0 --list-formats-ext
#should be video0

sudo usermod -aG video $USER

#OPTION 1
#on lenovo after installing ros
sudo apt install ros-humble-usb-cam
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=1
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:="/dev/video0" \
  -p image_width:=640 \
  -p image_height:=480 \
  -p pixel_format:="mjpeg"
  
#on jetson
export ROS_DOMAIN_ID=1
ros2 topic list

#OPTION 2 (raw UDP feed over gstreamer) 
#preferred cause the video appears inside jetson as native ROS2 container so it is like the actual depth cam feed

#lenovo
gst-launch-1.0 -v v4l2src device=/dev/video0 ! \
  video/x-raw,width=640,height=480,framerate=30/1 ! \
  videoconvert ! \
  x264enc tune=zerolatency speed-preset=ultrafast bitrate=2000 ! \
  rtph264pay ! \
  udpsink host=192.168.10.2 port=5000
  

- gstreamer install
  
#straight from GPT


sudo apt update && sudo apt install -y gstreamer1.0-tools gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly

gst-launch-1.0 -v v4l2src device=/dev/video0 ! \
  video/x-raw,width=640,height=480,framerate=30/1 ! \
  videoconvert ! \
  x264enc tune=zerolatency speed-preset=ultrafast bitrate=2000 ! \
  rtph264pay ! \
  udpsink host=192.168.10.2 port=5000
```

- receive feed on jetson

gscam : https://index.ros.org/p/gscam/

ROS : https://docs.ros.org/en/humble/Tutorials.html

gstreamer params : https://gstreamer.freedesktop.org/documentation/tools/gst-launch.html?gi-language=c#pipeline-examples

```
sudo apt install ros-humble-gscam   
```

```
source /opt/ros/humble/setup.bash  

ros2 run gscam gscam_node --ros-args \
  -p gstreamer_config:="udpsrc port=5000 caps=\"application/x-rtp, media=video, clock-rate=90000, encoding-name=H264, payload=96\" ! rtpjitterbuffer ! rtph264depay ! h264parse ! avdec_h264 ! videoconvert" \
  -p camera_name:="laptop_cam" \
  
```

- second ros terminal
```
cd ~/workspaces/isaac_ros-dev/src
cd isaac_ros_common
./scripts/run_dev.sh -b -d ~/workspaces/isaac_ros-dev

#rate at which published
ros2 topic list
ros2 topic hz /camera/image_raw

ros2 run rqt_graph rqt_graph
```

![[Pasted image 20260915054408.png|541]]
![[Pasted image 20260915054433.png|502]]

https://github.com/RobotWebTools/web_video_server
https://www.youtube.com/watch?v=m7qoRbQYX2E : integrate pycam with jetson 
https://jetsonhacks.com/2022/02/02/in-practice-usb-cameras-on-jetson/ : arducam IMX219
https://software-dl.ti.com/jacinto7/esd/robotics-sdk/latest/docs/source/ros2/drivers/gscam2/README_TI.html : has IMX219 cam
- isaac_ros_argus_camera

---
CONNECTING JETSON TO DP
```
sudo systemctl status gdm
#active --> gnome display manager is active

cat /sys/class/drm/card*/status
#connected

#check override config
sudo cat /etc/systemd/system/gdm.service.d/override.conf

#modeset config
sudo cat /sys/module/nvidia_drm/parameters/modeset
#Y

```
----
FLASHING DESKTOP IMAGE ON JETSON
![[Pasted image 20260917031513.png|406]]
![[Pasted image 20260917031644.png|381]]

https://www.youtube.com/watch?v=KXQEhidMDfc

----
CONNECTING TO THE VINO FROM REMMINI VNC
- follow the L4T readme 
edit gdm config 
```
sudo nano /etc/gdm3/custom.conf

[daemon]
WaylandEnable=false
AutomaticLoginEnable=true
AutomaticLogin=szyfrowac

sudo reboot

#x11vnc
sudo x11vnc -auth /run/user/1000/gdm/Xauthority -display :0 -rfbauth /home/szyfrowac/.vnc/passwd -rfbport 5900 -forever -shared -noxdamage

#vino 
just run, password = s
```

Prob don't have to run this again after the first time

---
SLAM 
- RTAB map 
- for OAK-D: depth AI v3, but use for this also (rtabmap_ros is not so good)
https://www.youtube.com/watch?v=tcJHnHpwCXk
- VIO 
	- google cartographer
	- Hector SLAM
	- RTAB Map
	- ORB slam
	- Rovio
- VIO as a ROS package: rtabmap + imu_filter_madgwick
![[Pasted image 20260918215334.png|363]]

- Playlist: https://www.youtube.com/watch?v=j8OYlSZcqMc&list=PLX9ajn_s_Vz-YplG70SMWvstcKPuv0jdM&index=1
- rtabmap_ros package : https://index.ros.org/p/rtabmap_ros/ : nt for isaac ros
- rviz
- camera D435i : sdk version 1 - ros doesn't work on that ; use version 2 + libuvc + downgrade camera firmware
- or build pi realsense 2 from source

point realsense-viewer to the binary we just installed

```
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install libusb-1.0-0-dev
sudo apt-get install libudev-dev
sudo apt-get install libssl-dev pkg-config libgtk-3-dev
sudo apt-get install git wget cmake build-essential
sudo apt-get install libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev at


git clone https://github.com/realsenseai/librealsense.git
cd librealsense
./scripts/setup_udev_rules.sh


mkdir build && cd build
cmake ../ -DFORCE_LIBUVC=true -DCMAKE_BUILD_TYPE=release -DBUILD_WITH_CUDA=ON
make -j4
sudo make install

#run from /librealsense/build/Releases/ with sudo
sudo ./realsense-viewer


cd ~/workspaces/isaac_ros-dev

colcon build --packages-select realsense2_camera isaac_ros_realsense \
  --symlink-install \
  --cmake-clean-cache \
  --cmake-args -Drealsense2_DIR=/librealsense/build/Release
```

```
tegrastats
#check CPU and GPU usage when launching realsense-viewer

```

![[Pasted image 20260919023039.png]]
![[Pasted image 20260919110057.png]]

---

Path planning 
- A* + frontier expansion
- Generating cost map in RTAB slam
- navFN nav2 ros 

```
NAV2 NAVFN A* 
YAML 

planner_server:
  ros__parameters:
    allow_partial_planning: true 
    expected_planner_frequency: 20.0
    costmap_update_timeout: 1.0
    introspection_mode: "disabled"
    planner_plugins: ['GridBased']
    GridBased:
      plugin: 'nav2_navfn_planner::NavfnPlanner'
      use_astar: True
      allow_unknown: True
      tolerance: 1.0
      max_cycles_factor: 4
      
controller_server:
  ros__parameters:
    controller_frequency: 20.0
    costmap_update_timeout: 0.3
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    failure_tolerance: 0.3
    publish_zero_velocity: true #added now
    odom_topic: "odom"
    odom_duration: 0.3
    progress_checker_plugins: ["progress_checker"]
    goal_checker_plugins: ["goal_checker"]
    path_handler_plugins: ["PathHandler"]
    controller_plugins: ["FollowPath"]
    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.5
      movement_time_allowance: 10.0
    goal_checker:
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.25
      yaw_goal_tolerance: 0.25
      path_length_tolerance: 1.0
      stateful: True
    PathHandler:
      plugin: "nav2_controller::FeasiblePathHandler"
      prune_distance: 2.0
      enforce_path_inversion: True
      enforce_path_rotation: False
      inversion_xy_tolerance: 0.2
      inversion_yaw_tolerance: 0.4
      minimum_rotation_angle: 0.785
      reject_unit_path: False
    FollowPath:
      plugin: "dwb_core::DWBLocalPlanner"
	
      
```

- NavFN: https://docs.ros.org/en/noetic/api/navfn/html/classnavfn_1_1NavFn.html
- Planner server: https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/core_servers/configuring_planner_server/
- controller server : https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/core_servers/controller_server/
- NavigateToPose
https://github.com/ros-navigation/navigation2
https://learnbydoing.dev/mastering-nav2-costmaps-in-ros-2/

```
Python launch file : keeper_filter.launch.py

import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, GroupAction, SetEnvironmentVariable
from launch.conditions import IfCondition
from launch.substitutions import LaunchConfiguration, PythonExpression
from launch_ros.actions import LoadComposableNodes, Node, PushROSNamespace, SetParameter
from launch_ros.descriptions import ComposableNode, ParameterFile
from nav2_common.launch import LaunchConfigAsBool, RewrittenYaml


def generate_launch_description() -> LaunchDescription:
    bringup_dir = get_package_share_directory('nav2_bringup')

    namespace = LaunchConfiguration('namespace')
    use_sim_time = LaunchConfigAsBool('use_sim_time')
    autostart = LaunchConfigAsBool('autostart')
    params_file = LaunchConfiguration('params_file')
    use_composition = LaunchConfigAsBool('use_composition')
    container_name = LaunchConfiguration('container_name')
    container_name_full = (namespace, '/', container_name)
    use_respawn = LaunchConfigAsBool('use_respawn')
    use_keepout_zones = LaunchConfigAsBool('use_keepout_zones')
    log_level = LaunchConfiguration('log_level')

    lifecycle_nodes = [
        'keepout_costmap_filter_info_server'
    ]

    remappings = [
        ('/tf', 'tf'),
        ('/tf_static', 'tf_static')
    ]

    yaml_substitutions = {
        'KEEPOUT_ZONE_ENABLED': use_keepout_zones,
    }

    configured_params = ParameterFile(
        RewrittenYaml(
            source_file=params_file,
            root_key=namespace,
            param_rewrites={},
            value_rewrites=yaml_substitutions,
            convert_types=True,
        ),
        allow_substs=True,
    )

    stdout_linebuf_envvar = SetEnvironmentVariable(
        'RCUTILS_LOGGING_BUFFERED_STREAM', '1'
    )

    declare_namespace_cmd = DeclareLaunchArgument(
        'namespace',
        default_value='',
        description='Top-level namespace'
    )

    declare_use_sim_time_cmd = DeclareLaunchArgument(
        'use_sim_time',
        default_value='false',
        description='Use simulation (Gazebo) clock if true',
    )

    declare_params_file_cmd = DeclareLaunchArgument(
        'params_file',
        default_value=os.path.join(
            bringup_dir,
            'params',
            'nav2_params.yaml'
        ),
        description='Full path to the ROS2 parameters file to use for all launched nodes',
    )

    declare_use_composition_cmd = DeclareLaunchArgument(
        'use_composition',
        default_value='False',
        description='Use composed bringup if True',
    )

    declare_container_name_cmd = DeclareLaunchArgument(
        'container_name',
        default_value='nav2_container',
        description='the name of container that nodes will load in if use composition',
    )

    declare_use_respawn_cmd = DeclareLaunchArgument(
        'use_respawn',
        default_value='False',
        description='Whether to respawn if a node crashes. Applied when composition is disabled.',
    )

    declare_use_keepout_zones_cmd = DeclareLaunchArgument(
        'use_keepout_zones',
        default_value='True',
        description='Whether to enable keepout zones or not'
    )

    declare_log_level_cmd = DeclareLaunchArgument(
        'log_level',
        default_value='info',
        description='log level'
    )

    load_nodes = GroupAction(
        condition=IfCondition(
            PythonExpression(['not ', use_composition])
        ),
        actions=[
            PushROSNamespace(namespace),

            SetParameter(
                'use_sim_time',
                use_sim_time
            ),

            Node(
                condition=IfCondition(use_keepout_zones),
                package='nav2_map_server',
                executable='costmap_filter_info_server',
                name='keepout_costmap_filter_info_server',
                output='screen',
                respawn=use_respawn,
                respawn_delay=2.0,
                parameters=[
                    configured_params
                ],
                arguments=[
                    '--ros-args',
                    '--log-level',
                    log_level
                ],
                remappings=remappings,
            ),

            Node(
                package='nav2_lifecycle_manager',
                executable='lifecycle_manager',
                name='lifecycle_manager_keepout_zone',
                output='screen',
                arguments=[
                    '--ros-args',
                    '--log-level',
                    log_level
                ],
                parameters=[
                    {'autostart': autostart},
                    {'node_names': lifecycle_nodes}
                ],
            ),
        ],
    )

    load_composable_nodes = GroupAction(
        condition=IfCondition(use_composition),
        actions=[
            PushROSNamespace(namespace),

            SetParameter(
                'use_sim_time',
                use_sim_time
            ),

            LoadComposableNodes(
                target_container=container_name_full,
                condition=IfCondition(use_keepout_zones),
                composable_node_descriptions=[
                    ComposableNode(
                        package='nav2_map_server',
                        plugin='nav2_map_server::CostmapFilterInfoServer',
                        name='keepout_costmap_filter_info_server',
                        parameters=[
                            configured_params
                        ],
                        remappings=remappings,
                    ),
                ],
            ),

            LoadComposableNodes(
                target_container=container_name_full,
                composable_node_descriptions=[
                    ComposableNode(
                        package='nav2_lifecycle_manager',
                        plugin='nav2_lifecycle_manager::LifecycleManager',
                        name='lifecycle_manager_keepout_zone',
                        parameters=[
                            {
                                'autostart': autostart,
                                'node_names': lifecycle_nodes
                            }
                        ],
                    ),
                ],
            ),
        ],
    )

    ld = LaunchDescription()

    ld.add_action(stdout_linebuf_envvar)
    ld.add_action(declare_namespace_cmd)
    ld.add_action(declare_use_sim_time_cmd)
    ld.add_action(declare_params_file_cmd)
    ld.add_action(declare_use_composition_cmd)
    ld.add_action(declare_container_name_cmd)
    ld.add_action(declare_use_respawn_cmd)
    ld.add_action(declare_use_keepout_zones_cmd)
    ld.add_action(declare_log_level_cmd)
    ld.add_action(load_nodes)
    ld.add_action(load_composable_nodes)

    return ld
```

```
NAV2 KEEPOUT 
YAML

keepout_filter_mask_server:
  ros__parameters:
    topic_name: "keepout_filter_mask"

keepout_costmap_filter_info_server:
  ros__parameters:
    type: 0
    filter_info_topic: "keepout_costmap_filter_info"
    mask_topic: "keepout_filter_mask"
    base: 0.0
    multiplier: 1.0


global_costmap:
  global_costmap:
    ros__parameters:
      ...
      plugins: ["static_layer", "obstacle_layer", "inflation_layer"]
      filters: ["keepout_filter"]

      keepout_filter:
        plugin: "nav2_costmap_2d::KeepoutFilter"
        enabled: True
        filter_info_topic: "keepout_costmap_filter_info"
        transform_tolerance: 0.1
        override_lethal_cost: True
        lethal_override_cost: 254

local_costmap:
  local_costmap:
    ros__parameters:
      ...
      plugins: ["voxel_layer", "inflation_layer"]
      filters: ["keepout_filter"]

      keepout_filter:
        plugin: "nav2_costmap_2d::KeepoutFilter"
        enabled: True
        filter_info_topic: "keepout_costmap_filter_info"
        transform_tolerance: 0.1
        override_lethal_cost: True
        lethal_override_cost: 254


map_server:
  ros__parameters:
    yaml_filename: "map.yaml"
    topic_name: "map"
    frame_id: "map"
    introspection_mode: "disabled"
    
```


ros2 launch nav2_bringup tb4_simulation_launch.py

- nav2 collision monitor : https://github.com/ros-navigation/navigation2/tree/main/nav2_collision_monitor
----
INTEGRATION
- realsense-viewer with CUDA --> isaac ros real_sense node --> RTAB Map ros2 
- navFN 
  
![[Pasted image 20260920105347.png]]


- host detects USB from realsense -> librealsense (camera SDK, drivers + API) on host with CUDA and sudo detects camera  -> we see on realsense-viewer -> pass cam USB into Isaac ROS container -> isaac ROS realsense2_camera (ros 2 wrapper and node) uses correct librealsense -> ROS topics -> host on RTAB map
- ARDUCAM --> vision node: QR Code bounding box + red zone mask/bounding box --> QR pixel coordinates + red zone pixel coordinates --> camera calibration --> map on ground --> navigation/goal or planner node : QR is goal, red zone is no go --> pose stamp --> nav2 A* Frontier --> cmd_vel --> SITL
- red zone : image segmentation map, qr code : bounding box for localization
- https://thegradient.pub/semantic-segmentation/
- https://docs.ultralytics.com/tasks/segment
- A* current position to QR Code while avoiding red zones --> NavFN
- Frontier : next area to explore
- dynamically update costmap

frontier manager -> post stamped -> subscribes to exploration map and drone pose 

custom red zone costmap : red_zone --> lethal costmap cells

navigation :
- perception
	- qr_detection
	- red_zone_detection
- mapping
	- exploration_map_node
- exploration
	- frontier
- mission
	- mission_manager
- alignment
	- qr_alignment
- nav2_plugins
	- red_zone_layer: https://docs.nav2.org/rolling/tutorials/general_tutorials/navigation2_with_keepout_filter/navigation2_with_keepout_filter/
- sitl
	- nav2_to_sitl_bridge

- simple HSV segmentation versus training a YOLO model
---
GAZEBO

```
aerothon_env/
├── worlds/
│   └── aerothon_mission2.sdf
├── models/
│   └── aerothon_mission2/
│       ├── model.config
│       └── model.sdf
├── materials/
│   └── textures/
│       └── README.txt
├── README.md
└── run_gazebo.sh
```

- launch with ros: https://ros-learnings.hashnode.dev/step-by-step-guide-to-creating-and-launching-custom-worlds-in-gazebo-ros

```
#export GZ_SIM_RESOURCE_PATH=/path/to/your/models:/path/to/your/worlds   

export GZ_SIM_RESOURCE_PATH="/home/lenovo/Desktop/Aeolus/gazebo aerothon/aerothon_mission2_gazebo_env/aerothon_env/models:/home/lenovo/Desktop/Aeolus/gazebo aerothon/aerothon_mission2_gazebo_env/aerothon_env/worlds"

#check with echo "$GZ_SIM_RESOURCE_PATH"

gz sim "/home/lenovo/Desktop/Aeolus/gazebo aerothon/aerothon_mission2_gazebo_env/aerothon_env/worlds/aerothon_mission2_updated.sdf"
```
- Put pngs in materials/textures
- entity pose : position + orientation (for the geofence of corridors)
---
# Flow

- ~~qgc + mavlink router install~~
- ~~install docker + nvidia toolkit + isaac ros2 humble~~ 
- laptop cam (ros topic) --> RTAB Map SLAM to calc 3D position and orientation --> NAV2 + custom path planner for obstacle avoidance path and waypoints --> Mavlink bridge (MAVROS + pymavlink) to send odometry --> FC (SITL) --> mavlink tel (UDP/SERIAL) --> QGC (mission update)

- stream laptop using Gstreamer/ opencv 
- install pymavlink / mavros
- send mavlink script for heartbeat

- install ROS2 Humble
- camera bridge
- RTAB Map
- NAV2 + algos : frontier expansions algo + A*
- Micro XRCE DDS agent for PX4 or MAVROS 


QR Code via gstreamer pipeline : OpenCV + pyZbar with pymavlink

