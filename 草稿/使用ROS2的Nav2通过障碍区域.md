## 前置知识
### 理解比赛目标

> [!quote] 比赛地图
> ![[Pasted image 20260518120004.png|418]]

我们的狗从启停区出发，要经过避障区，并且不能碰到旁边的板。我们肯定需要使用到雷达，但是官方的SDK仅仅提供雷达的数据，所以具体要怎么用，还得看我们自己。

### 理解 ROS2 和 纯C++调用SDK的关系

用纯C++没有办法使用 ROS2 里的东西，但是如果你用 ROS2，SDK的东西和ROS2的东西都可以使用。
> 这里说的"东西"是指函数、库之类的

### 理解什么是 Nav2

简单来说，这是一个可以给机器狗导航的模块。

你可以使用狗身上的雷达扫描现实世界的障碍或房间的墙壁，把他们的雷达点云保存成地图，当实际在狗身上运行Nav2的时候，Nav2可以使用实时的雷达点云去和地图对比，然后**计算出自己的坐标**。

有了坐标，Nav2就能根据两点的坐标进行导航，**他有一套导航算法，用来在避开障碍物的同时到达目标点。**

## 环境配置
假设已经拥有了ros的环境，如果没有，参考
https://support.unitree.com/home/zh/developer/ROS2_service

安装 Nav2 及其配套的建图工具，默认ROS2的版本为foxy
```shell
sudo apt update
sudo apt install ros-${ROS_DISTRO}-navigation2 ros-${ROS_DISTRO}-nav2-bringup ros-${ROS_DISTRO}-slam-toolbox
```
**或者**运行
```shell
sudo apt update
sudo apt install ros-foxy-navigation2 ros-foxy-nav2-bringup ros-foxy-slam-toolbox
```

创建我们的项目文件夹
```bash
cd ~/my_ros2_ws/src
ros2 pkg create --build-type ament_cmake go2_nav_project --dependencies rclcpp nav2_msgs geometry_msgs
```

创建完后，**大概**是这样子：
```
my_ros2_ws/                     # 你的 ROS 2 工作空间根目录
├── build/                      # 编译生成的文件（自动生成，不要手动改）
├── install/                    # 编译后的可执行文件和环境配置（自动生成）
├── log/                        # 运行日志（自动生成）
└── src/                        # 🌟 你的所有源码都放在这里
    └── go2_nav_project/        # 你的具体项目（功能包名称）
        ├── CMakeLists.txt      # 编译配置文件（极其重要）
        ├── package.xml         # 包信息和依赖项声明
        ├── include/            # 存放 C++ 头文件
        │   └── go2_nav_project/# 头文件最好再嵌套一层包名，防止命名冲突
        │       ├── my_node.hpp
        │       └── utils.hpp
        ├── src/                # 存放 C++ 源文件
        │   ├── my_node.cpp
        │   └── utils.cpp
        ├── launch/             # 存放启动文件 (.py 或 .xml)
        │   └── start_nav.launch.py
        ├── config/             # 存放配置文件
        │   └── nav2_params.yaml# 比如针对 Go2 修改的 Nav2 参数
        ├── maps/               # 存放建好的地图文件
        │   ├── my_go2_map.yaml
        │   └── my_go2_map.pgm
        └── rviz/               # 存放 RViz2 的界面配置文件
            └── default_view.rviz
```


## 建图

```bash

```