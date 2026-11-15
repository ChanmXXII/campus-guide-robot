# Autonomous Multilingual Campus Guide Robot

A TurtleBot3 Burger that guides visitors around a cardboard model of the Inha University campus.
Users ask in Korean or English (by talking to the robot or through a phone web app opened by QR code)
where a building is. The robot drives there with Nav2 and explains the building out loud.

See `docs/Project_Idea.pdf` for the full concept.

## Status

| Part | Status |
|---|---|
| ROS 2 Humble on Pi 4, OpenCR firmware, motors + teleop | ✅ |
| LiDAR LDS-03 (`/scan`) | ✅ |
| SLAM setup (slam_toolbox + `scan_fixer.py`) | ✅ tested in a room |
| Pi Camera V2 + calibration (0.64 px) | ✅ |
| AprilTag detection, IDs 1-10, with distance | ✅ |
| 2.0" SPI display (ST7789) | ✅ |
| USB speaker + gTTS (EN/KR), USB mic detected | ✅ |
| Map of the cardboard campus | ⬜ |
| Building goals in `buildings.yaml` | ⬜ |
| Nav2 autonomous driving | ⬜ |
| AprilTag arrival logic | ⬜ |
| LLM intent parser | ⬜ |
| Phone web app (QR, KR/EN, E-stop) | ⬜ |
| Voice input (speech-to-text) | ⬜ |

## Hardware

- TurtleBot3 Burger, Raspberry Pi 4, OpenCR
- LiDAR: **LDS-03** (driver `coin_d4_driver`), not LDS-01
- Raspberry Pi Camera **V2** (IMX219). Camera Module 3 does not work on the Ubuntu 22.04 kernel.
- 2.0" SPI TFT, ST7789, 240x320
- USB speaker (`UACDemoV10`), USB mic (`Britz BE-STM10`)

## Software

- Ubuntu 22.04 + ROS 2 Humble
- SLAM: `slam_toolbox` with `scan_fixer.py`, because the LDS-03 sends a different number of points
  every scan (395-409) and slam_toolbox needs exactly the same number
- AprilTags: `apriltag_ros`, family `tag36h11`, size 0.08 m, IDs 1-10 = buildings
- TTS: `gTTS` + `mpg123`

## Folder structure

```
config/          system config for the Pi (udev rule, audio, ROS env vars, Wi-Fi template, camera boot lines)
docs/            project idea + printables (AprilTags, checkerboard)
maps/            saved SLAM maps (.pgm + .yaml)
robot_ws/src/    ROS 2 code
  campus_guide_bringup/config/   tags.yaml, buildings.yaml, slam_params.yaml, camera_calibration.yaml
  campus_guide_perception/       scan_fixer.py
  campus_guide_navigation/       (Nav2 goal logic - to do)
  campus_guide_interface/        (display, speaker, mic nodes - to do)
tools/           one-off utilities (calibration, tag test, display test, printable generators)
web_app/         phone web app (to do)
```

## Run: SLAM

```bash
ros2 launch turtlebot3_bringup robot.launch.py
python3 robot_ws/src/campus_guide_perception/scan_fixer.py
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=$PWD/robot_ws/src/campus_guide_bringup/config/slam_params.yaml use_sim_time:=false
ros2 run turtlebot3_teleop teleop_keyboard
ros2 run nav2_map_server map_saver_cli -f maps/campus
```

Create `slam_params.yaml` first with `bash tools/make_slam_params.sh`.

## Run: AprilTag detection

```bash
ros2 run v4l2_camera v4l2_camera_node --ros-args -p video_device:=/dev/video0 -p image_size:="[640,480]" -p time_per_frame:="[1,15]"
ros2 run image_proc rectify_node --ros-args -r image:=/image_raw -r camera_info:=/camera_info
ros2 run apriltag_ros apriltag_node --ros-args -r image_rect:=/image_rect -r camera_info:=/camera_info \
  --params-file robot_ws/src/campus_guide_bringup/config/tags.yaml
ros2 topic echo /detections | grep "id:"
ros2 run tf2_ros tf2_echo camera building_1
```

## Buildings

| ID | Building | KR | Key | Box |
|---|---|---|---|---|
| 1 | Main Building | 본관 | main_building | rectangle |
| 2 | Jungseok Library | 정석학술정보관 | library | rectangle |
| 3 | Hi-Tech Center | 하이테크센터 | hitech_center | rectangle |
| 4 | Building 2 | 2호관 | building_2 | rectangle |
| 5 | Building 5 | 5호관 | building_5 | rectangle |
| 6 | 60th Anniversary Hall | 60주년기념관 | hall_60th | rectangle |
| 7 | Student Union | 학생회관 | student_union | rectangle |
| 8 | Business School | 경영대학 | business_dept | rectangle |
| 9 | Cafe | 카페 | cafe | triangle |
| 10 | Restroom | 화장실 | restroom | triangle |

Tag 0 is a spare.

## Setup on a new Pi

See `docs/setup_guide.md`.
