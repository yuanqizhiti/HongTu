# HongTu ROS Docker From Zero

本文从零开始记录：Jetson Orin 上如何创建固定 Docker 容器、安装 ROS 导航依赖、编译项目、保存镜像，以及以后如何多终端进入同一个容器。

不要使用 `docker run --rm`。`--rm` 会在退出后删除容器，导致安装过的环境丢失。

## 0. 项目路径约定

Jetson 宿主机项目路径：

```bash
~/rpf/HongTu
```

容器内项目路径：

```bash
/root/HongTu
```

ROS 工作区路径：

```bash
/root/HongTu/G1Nav2D
```

## 1. 拉取基础 ROS 镜像

```bash
sudo docker pull ros:noetic-ros-base
```

## 2. 删除旧的同名容器

如果之前创建过失败的 `hongtu-nav`，先删除。

查看：

```bash
sudo docker ps -a
```

如果存在 `hongtu-nav`，删除：

```bash
sudo docker rm hongtu-nav
```

## 3. 创建固定容器

第一次创建容器时执行。注意最后使用的是基础镜像 `ros:noetic-ros-base`。

```bash
sudo docker run -it \
  --name hongtu-nav \
  --net=host \
  --privileged \
  -v ~/rpf/HongTu:/root/HongTu \
  -w /root/HongTu/G1Nav2D \
  ros:noetic-ros-base \
  bash
```

进入后，提示符类似：

```text
root@ubuntu:~/HongTu/G1Nav2D#
```

下面所有安装和编译命令都在容器内执行。

## 4. 安装基础工具

```bash
apt update
apt install -y \
  build-essential \
  cmake \
  git \
  python3-catkin-tools \
  python3-rosdep \
  python3-pip \
  iproute2 \
  net-tools \
  iputils-ping \
  libeigen3-dev \
  libpcl-dev \
  libgoogle-glog-dev \
  libgflags-dev \
  libboost-all-dev \
  libopencv-dev \
  libapr1-dev
```

## 5. 安装必备 ROS 包

```bash
apt install -y \
  ros-noetic-pcl-ros \
  ros-noetic-pcl-conversions \
  ros-noetic-tf2 \
  ros-noetic-tf2-ros \
  ros-noetic-tf2-sensor-msgs \
  ros-noetic-eigen-conversions \
  ros-noetic-map-server \
  ros-noetic-move-base \
  ros-noetic-navigation \
  ros-noetic-amcl \
  ros-noetic-teb-local-planner \
  ros-noetic-global-planner \
  ros-noetic-costmap-converter \
  ros-noetic-octomap-server \
  ros-noetic-gazebo-msgs \
  ros-noetic-roslint \
  ros-noetic-rviz
```

安装 GTSAM：

```bash
apt install -y ros-noetic-gtsam || apt install -y libgtsam-dev
```

## 6. 安装 Python 依赖

目标点脚本用到 `playsound`。

```bash
pip3 install playsound
```

## 7. 初始化 rosdep

先手动安装项目里 `rosdep` 不容易识别的依赖：

```bash
apt install -y ros-noetic-gtsam || apt install -y libgtsam-dev
apt install -y libopencv-dev
```

```bash
rosdep init || true
rosdep update
```

在工作区根目录执行：

```bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
rosdep install --from-paths src --ignore-src -r -y
```

如果出现下面这些，可以先忽略：

```text
gtsam
opencv2
livox_ros_driver2
```

原因：这些是项目 `package.xml` 里的依赖 key 写法，`rosdep` 不一定能解析；上面已经手动安装了 GTSAM 和 OpenCV，`livox_ros_driver2` 则是源码包。

## 8. 安装 Livox-SDK2

```bash
cd /root/HongTu/Livox-SDK2
mkdir -p build
cd build
cmake ..
make -j4
make install
```

检查：

```bash
ls /usr/local/lib | grep livox
ls /usr/local/include | grep livox
```

应能看到 Livox SDK 的库和头文件。

## 9. 编译 ROS 工作区

```bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
```

准备 Livox ROS1 包文件：

```bash
cp src/livox_ros_driver2-master/package_ROS1.xml src/livox_ros_driver2-master/package.xml
```

清理旧构建：

```bash
rm -rf build devel
```

先编译 Livox driver：

```bash
catkin_make -DROS_EDITION=ROS1 --pkg livox_ros_driver2 -j1
```

编译 fastlio：

```bash
source devel/setup.bash
catkin_make -DROS_EDITION=ROS1 --only-pkg-with-deps fastlio -j1
```

编译 movebase 包：

```bash
source devel/setup.bash
catkin_make -DROS_EDITION=ROS1 --only-pkg-with-deps xju_pnc -j1
```

全局编译：

```bash
source devel/setup.bash
catkin_make -DROS_EDITION=ROS1 -j1
```

## 10. 编译后检查

```bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
```

检查 ROS 包：

```bash
rospack find fastlio
rospack find livox_ros_driver2
rospack find xju_pnc
rospack find map_server
rospack find move_base
```

检查节点：

```bash
ls devel/lib/fastlio
ls devel/lib/livox_ros_driver2
ls devel/lib/xju_pnc
ls devel/lib/tool
```

## 11. 保存当前容器为镜像

安装和编译都完成后，不要删除容器。

另开一个 Jetson 宿主机终端，执行：

```bash
sudo docker commit hongtu-nav hongtu-ros:noetic
```

检查镜像：

```bash
sudo docker images | grep hongtu
```

以后可以用 `hongtu-ros:noetic` 作为已安装好依赖的环境模板。

## 12. 退出容器

容器内执行：

```bash
exit
```

容器不会被删除，因为创建时没有使用 `--rm`。

## 13. 以后重新进入同一个容器

如果容器已退出：

```bash
sudo docker start -ai hongtu-nav
```

如果容器已经运行，另开终端进入：

```bash
sudo docker exec -it hongtu-nav bash
```

进入容器后加载 ROS 环境：

```bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
```

## 14. 日常启动顺序

终端 1：

```bash
sudo docker start -ai hongtu-nav
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
roscore
```

终端 2：

```bash
sudo docker exec -it hongtu-nav bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
rosrun map_server map_server src/ros_map_edit/maps/map.yaml
```

终端 3：

```bash
sudo docker exec -it hongtu-nav bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
roslaunch xju_pnc move_base.launch
```

终端 4：

```bash
sudo docker exec -it hongtu-nav bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
rostopic list
```

## 15. 目标点脚本

监听目标点：

```bash
sudo docker exec -it hongtu-nav bash
cd /root/HongTu/G1Nav2D
source /opt/ros/noetic/setup.bash
source devel/setup.bash
rostopic echo /move_base/goal
```

运行目标点：

```bash
sudo docker exec -it hongtu-nav bash
cd /root/HongTu/PythonProject/point_nav
source /opt/ros/noetic/setup.bash
source /root/HongTu/G1Nav2D/devel/setup.bash
python3 point1.py
```

## 16. 常见错误

### `hongtu-ros:noetic` not found

说明还没有执行：

```bash
sudo docker commit hongtu-nav hongtu-ros:noetic
```

从零开始时，应先用：

```bash
ros:noetic-ros-base
```

创建容器。

### `map_server` not found

说明当前容器没有安装：

```bash
apt install -y ros-noetic-map-server
```

如果你是新开了一个 `docker run --rm` 临时容器，也会出现这个问题。

### `move_base` not found

安装：

```bash
apt install -y ros-noetic-move-base ros-noetic-navigation
```

### `ament_cmake_auto` not found

说明 Livox driver 误走 ROS2 分支。编译时必须加：

```bash
-DROS_EDITION=ROS1
```
