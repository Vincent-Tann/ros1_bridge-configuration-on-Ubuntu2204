# ros1_bridge-configuration-on-Ubuntu2204
记录如何在Ubuntu22.04 jammy上安装ROS2、ROS1和ros1_bridge

### 关于ros1_bridge
ros1_bridge是一个ROS2 package，其作用是桥接ROS2和ROS1间的message和service。主要是为了满足在ROS2环境中开发时需要使用的包只有ROS1版本的问题。

### ros1_bridge安装难点和官方教程

在Ubuntu 22.04上配置ros1_bridge的主要难点在于两个ROS版本的依赖冲突。[官方教程](https://github.com/ros2/ros1_bridge#example-1-run-the-bridge-and-the-example-talker-and-listener)的做法是install ROS2 from source，但是对于后续安装新的ROS2 packages不太友好。install ROS2 from debian packages是最简单的做法，但是后续再安装ROS1会出现依赖冲突（主要是几个catkin相关的包），自己手动搞比较难（我最后没有成功）。

### 一个好用的非官方安装工具

一个叫Tommy的老哥提供了一个Dockerfile，在里面完成了在Ubuntu 22.04 jammy上 install ROS2 Humble and ROS1 Noetic from debian packages，并colcon build ros1_bridge的过程，非常丝滑，编译完成之后把Docker Image中编译好的ros1_bridge的库拿到本地的ros1_bridge的工作空间下，就可以使用了。这样避免了在Ubuntu22.04本地安装ROS1，可能污染ROS2依赖环境的问题。

Repo: [https://github.com/TommyChangUMD/ros-humble-ros1-bridge-builder](https://github.com/TommyChangUMD/ros-humble-ros1-bridge-builder)

跟随教程完成`How to create ros-humble-ros1-bridge package`这一步之后，ros1_bridge的工作空间位于`~/ros-humble-ros1-bridge/`。

Repo中给的示例是把ROS1环境放在了一个Container中，这也和使用场景吻合——希望使用一个包，其提供了Docker，但只有ROS1版本。示例中是用的rocker来启动Container，但直接用Docker也可以。主要是相关的配置要配好。

例如使用最基础的ROS1 Noetic环境：
```bash
docker pull ros:noetic
```

启动ROS1容器：
```bash
docker run -it \
   --net=host \
   --gpus all \
   --runtime nvidia \
   --env="DISPLAY=$DISPLAY" \
   --env="QT_X11_NO_MITSHM=1" \
   --env="NVIDIA_VISIBLE_DEVICES=all" \
   --env="NVIDIA_DRIVER_CAPABILITIES=all" \
   --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
   --volume="$HOME/.Xauthority:/root/.Xauthority:rw" \
   --volume "/dev/shm:/dev/shm" \
   --device /dev/dri \
   --name="ros_noetic" \
   ros:noetic \
   bash
```
比较关键的是`--net=host`（允许容器接入网络，否则宿主机发现不了容器中的rosmaster服务器。作者还强调了`--volume "/dev/shm:/dev/shm"`（作用是挂载宿主机的共享内存目录到容器，应该是为了加速数据传输）。

在ROS1容器中启动roscore：
```bash
# ros1 container shell 1
source /opt/ros/noetic/setup.bash
roscore
```

在宿主机（本地）source ROS2的环境，以及ros1_bridge的workspace
```bash
# local shell 1
source /opt/ros/humble/setup.bash
source ~/ros-humble-ros1-bridge/install/local_setup.bash
```
强调source ros1_bridge的workspace时，要source `local_setup.bash`，而不是`setup.bash`。因为编译ros1_bridge的docker container的底层环境可能和本地不同。

随后查看支持的ROS1-ROS2 消息对和服务对：
```bash
# local shell 1
ros2 run ros1_bridge dynamic_bridge --print-pairs
```

配置桥接所有话题：
```bash
# local shell 1
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

会出现提示例如：
```
created 2to1 bridge for topic '/joint_states' with ROS 2 type sensor_msgs/msg/JointState' and ROS 1 type 'sensor_msgs/JointState'
```

新开一个ROS1 container的shell（`docker exec -it ros_noetic bash`)，查看相应话题:
```bash
# ros1 container shell 2
source /opt/ros/noetic/setup.bash
rostopic list
rostopic echo /joint_states
```
看到消息流说明桥接成功。

1发2收也是一样的:
```bash
# local shell 2
source /opt/ros/humble/setup.bash
ros2 topic list
ros2 topic echo /xxx
```