# TurtleBot 4 SLAM và autonomous navigation

Workspace ROS 2 Jazzy cho bài tập TurtleBot 4 mô phỏng bằng Gazebo Sim. Package `tb4_assignment` cung cấp launch cho mapping, localization và Nav2; map `final_map` cùng tham số SLAM/Nav2 được lưu trong repository.

## Nội dung repository

- `src/tb4_assignment/launch/`: launch mapping, localization, navigation và Gazebo Ogre2.
- `src/tb4_assignment/config/`: tham số SLAM Toolbox và Nav2/AMCL.
- `src/tb4_assignment/maps/final_map.yaml` và `final_map.pgm`: bản đồ warehouse đã lưu (605 × 975 pixel, độ phân giải 0.05 m/pixel).

## Yêu cầu

- Ubuntu 24.04, ROS 2 Jazzy và Gazebo Sim/Harmonic.
- TurtleBot 4 ROS 2 packages và các dependency của chúng.
- SLAM Toolbox, Nav2, `teleop_twist_keyboard`, `colcon`.

Cài các package ROS cần dùng nếu chưa có:

```bash
sudo apt update
sudo apt install \
  ros-jazzy-turtlebot4-gz-bringup \
  ros-jazzy-turtlebot4-navigation \
  ros-jazzy-slam-toolbox \
  ros-jazzy-navigation2 \
  ros-jazzy-nav2-bringup \
  ros-jazzy-teleop-twist-keyboard \
  python3-colcon-common-extensions
```

## Clone và build

Repository này là ROS 2 workspace hoàn chỉnh; clone repository vào thư mục workspace:

```bash
git clone <URL_REPOSITORY_PUBLIC> ~/tb4_slam_ws
cd ~/tb4_slam_ws
source /opt/ros/jazzy/setup.bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

Mỗi terminal ROS mới cần source môi trường:

```bash
source /opt/ros/jazzy/setup.bash
source ~/tb4_slam_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

## Mapping bằng SLAM Toolbox

Khởi chạy Gazebo warehouse, TurtleBot 4, RViz2 và SLAM:

```bash
ros2 launch tb4_assignment sim_mapping.launch.py \
  world:=warehouse model:=standard rviz:=true gui:=true
```

Launch dùng renderer Ogre2; Gazebo server và GUI chạy tách riêng để lỗi render GUI không nhất thiết làm dừng physics/sensor server. Robot được spawn tại pose mặc định và launch mapping tự gửi yêu cầu undock; chờ undock hoàn tất trước khi lái.

Kiểm tra LiDAR/odometry và điều khiển teleop trong terminal khác:

```bash
ros2 topic hz /scan
ros2 topic hz /odom
ros2 run teleop_twist_keyboard teleop_twist_keyboard \
  --ros-args -p stamped:=true -p use_sim_time:=true -r cmd_vel:=/cmd_vel
```

Khảo sát các hành lang/khu vực có thể đi qua ở tốc độ chậm; nên quay lại gần các vùng đã quét để SLAM Toolbox đóng loop. Theo dõi `/map`, `/scan` và TF trên RViz2. Khi hoàn tất, lưu map vào thư mục package:

```bash
ros2 run nav2_map_server map_saver_cli \
  -f "$HOME/tb4_slam_ws/src/tb4_assignment/maps/final_map"
```

Lệnh này cập nhật `final_map.yaml` và `final_map.pgm`; kiểm tra map trước khi commit/push. Muốn giữ bản hiện tại, lưu dưới tên khác thay vì ghi đè.

## Localization

Dừng phiên mapping, khởi chạy lại trong cùng world với map đã lưu:

```bash
ros2 launch tb4_assignment localization.launch.py \
  world:=warehouse model:=standard rviz:=true \
  map:="$(ros2 pkg prefix tb4_assignment)/share/tb4_assignment/maps/final_map.yaml"
```

Trong RViz2, dùng **2D Pose Estimate** để chỉ initial pose nếu AMCL chưa định vị được. Xác nhận robot hiển thị đúng vị trí so với Gazebo và TF `map -> odom` được publish. Localization launch cũng tự gửi yêu cầu undock sau khi khởi động.

## Autonomous navigation với Nav2

```bash
ros2 launch tb4_assignment navigation.launch.py \
  world:=warehouse model:=standard rviz:=true gui:=true \
  map:="$(ros2 pkg prefix tb4_assignment)/share/tb4_assignment/maps/final_map.yaml"
```

`auto_undock` mặc định là `false`. Nếu robot còn trên dock và cần rời dock, bật tùy chọn này và chờ undock hoàn tất trước khi đặt goal:

```bash
ros2 launch tb4_assignment navigation.launch.py \
  world:=warehouse model:=standard rviz:=true gui:=true auto_undock:=true \
  map:="$(ros2 pkg prefix tb4_assignment)/share/tb4_assignment/maps/final_map.yaml"
```

Sau khi AMCL hội tụ, dùng **Nav2 Goal** trên RViz2. Chọn điểm nằm trong vùng tự do và cách tường/vật cản; kéo mũi tên theo hướng đích mong muốn. Không chạy teleop đồng thời. Nav2 tự lập kế hoạch, tránh vật cản và dừng theo goal tolerance trong `config/nav2_params.yaml`.

## Kiểm tra nhanh / xử lý sự cố

```bash
ros2 topic list | grep -E '^/(scan|odom|map|cmd_vel)$'
ros2 topic echo /amcl_pose --once
ros2 action list | grep navigate
ros2 run tf2_ros tf2_echo map base_link
```

- Không có `/scan`: kiểm tra Gazebo server còn chạy và đúng ROS environment/DDS.
- RobotModel báo thiếu TF: kiểm tra `/tf`, `/tf_static` và frame `base_link`; không khởi chạy nhiều phiên mô phỏng cũ cùng lúc.
- RViz không hiện map: kiểm tra `map:=...` và YAML có tham chiếu đúng `final_map.pgm`.
- Nếu GUI Gazebo gặp lỗi OpenGL, thử `gui:=false`; server mô phỏng vẫn có thể chạy không cửa sổ. Tránh bật software rendering nếu nó làm Gazebo crash trên máy.

## Đưa lên GitHub Public

Tạo repository Public trên GitHub, sau đó chạy từ thư mục workspace:

```bash
git init
git add README.md .gitignore src/tb4_assignment
git commit -m "Add TurtleBot 4 SLAM and Nav2 assignment"
git branch -M main
git remote add origin <URL_REPOSITORY_PUBLIC>
git push -u origin main
```

Không đưa `build/`, `install/`, `log/` hoặc cache IDE lên repository.
