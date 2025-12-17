# RealSense D435 Camera Integration

## Overview

The MPO-700 robot now includes an Intel RealSense D435 depth camera mounted on the front chassis for navigation and perception tasks.

## Camera Specifications

- **Model**: Intel RealSense D435
- **Location**: Front chassis, centered
- **Position**: X=0.292m, Y=0.0m, Z=0.716m (relative to base_link)
- **Orientation**: Facing forward (along robot's X-axis)

### Sensor Capabilities

- **RGB Camera**: 640×480 @ 30 Hz
- **Depth Camera**: 640×480 @ 30 Hz
- **Depth Range**: 0.2m to 10.0m
- **RGB Field of View**: 69.4° horizontal
- **Depth Field of View**: 58° horizontal
- **Point Cloud**: Generated from depth data

## Published Topics

The camera publishes the following ROS 2 topics:

| Topic | Type | Description |
|-------|------|-------------|
| `/camera/color/image_raw` | `sensor_msgs/Image` | RGB color image |
| `/camera/color/camera_info` | `sensor_msgs/CameraInfo` | RGB camera calibration |
| `/camera/depth/image_raw` | `sensor_msgs/Image` | Depth image (16-bit) |
| `/camera/depth/camera_info` | `sensor_msgs/CameraInfo` | Depth camera calibration |
| `/camera/depth/points` | `sensor_msgs/PointCloud2` | 3D point cloud |

## TF Frame Tree

```
base_link
  └─ camera_link (main camera body)
       ├─ camera_depth_frame
       │    └─ camera_depth_optical_frame (depth sensor optical frame)
       ├─ camera_color_frame
       │    └─ camera_color_optical_frame (RGB sensor optical frame)
       ├─ camera_infra1_frame
       │    └─ camera_infra1_optical_frame (left IR sensor)
       └─ camera_infra2_frame
            └─ camera_infra2_optical_frame (right IR sensor)
```

## Usage

### Launch Simulation with Camera

The camera is automatically included when launching the MPO-700 robot:

```bash
# Using the full bringup package
ros2 launch neo_full_bringup full_navigation.launch.py

# Or just simulation
ros2 launch neo_simulation2 simulation.launch.py my_robot:=mpo_700 arm_type:=ur10
```

### Verify Camera Topics

Check that camera topics are being published:

```bash
ros2 topic list | grep camera
```

Expected output:
```
/camera/color/camera_info
/camera/color/image_raw
/camera/depth/camera_info
/camera/depth/image_raw
/camera/depth/points
```

### View Camera Data

#### RGB Image

Using RQt Image View:
```bash
ros2 run rqt_image_view rqt_image_view /camera/color/image_raw
```

#### Depth Image

```bash
ros2 run rqt_image_view rqt_image_view /camera/depth/image_raw
```

**Note**: Depth images are 16-bit, so they may appear mostly black. Use encoding dropdown to select "16UC1" for better visualization.

#### Point Cloud

View the 3D point cloud in RViz:

1. Launch RViz:
```bash
ros2 run rviz2 rviz2
```

2. Add PointCloud2 display:
   - Click "Add" button
   - Select "PointCloud2"
   - Set Topic: `/camera/depth/points`
   - Set Fixed Frame: `base_link` or `camera_depth_optical_frame`

3. Adjust display:
   - Size: 0.01
   - Style: Points or Flat Squares
   - Color Transformer: RGB8 or Intensity

### Check Camera Frame Transforms

Verify all camera frames are properly connected:

```bash
# View TF tree
ros2 run tf2_tools view_frames

# This creates frames.pdf - open it to see the full TF tree
```

Or check specific transform:
```bash
ros2 run tf2_ros tf2_echo base_link camera_color_optical_frame
```

### Monitor Camera Data Rate

Check the publishing frequency:

```bash
# RGB camera (should be ~30 Hz)
ros2 topic hz /camera/color/image_raw

# Depth camera (should be ~30 Hz)
ros2 topic hz /camera/depth/image_raw

# Point cloud (should be ~30 Hz)
ros2 topic hz /camera/depth/points
```

## Integration with Navigation

The camera data can be used for:

### 1. Obstacle Detection

Add the point cloud to the costmap configuration:

Edit `navigation.yaml`:
```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      plugins: ["obstacle_layer", "inflation_layer"]
      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        observation_sources: scan scan2 camera_points
        camera_points:
          topic: /camera/depth/points
          sensor_frame: camera_depth_optical_frame
          data_type: "PointCloud2"
          min_obstacle_height: 0.1
          max_obstacle_height: 2.0
          obstacle_range: 5.0
          raytrace_range: 6.0
```

### 2. Visual Odometry

Use RGB-D data for visual odometry with RTAB-Map or similar:

```bash
ros2 launch rtabmap_ros rtabmap.launch.py \
  rgb_topic:=/camera/color/image_raw \
  depth_topic:=/camera/depth/image_raw \
  camera_info_topic:=/camera/color/camera_info \
  frame_id:=base_link \
  approx_sync:=true
```

### 3. 3D Mapping

Create 3D maps using OctoMap:

```bash
ros2 run octomap_server octomap_server_node \
  --ros-args \
  -r cloud_in:=/camera/depth/points \
  -p frame_id:=map
```

## Technical Details

### Camera Parameters

The camera simulation uses the following parameters:

**RGB Camera:**
- Resolution: 640×480
- Horizontal FOV: 1.211 radians (69.4°)
- Update Rate: 30 Hz
- Noise: Gaussian (mean=0.0, stddev=0.007)

**Depth Camera:**
- Resolution: 640×480
- Horizontal FOV: 1.012 radians (58°)
- Update Rate: 30 Hz
- Near Clip: 0.2m
- Far Clip: 10.0m
- Noise: Gaussian (mean=0.0, stddev=0.01)

### Gazebo Plugins Used

- **RGB Camera**: `libgazebo_ros_camera.so`
- **Depth Camera**: `libgazebo_ros_camera.so` with `type="depth"`

These are standard Gazebo ROS 2 plugins included in the `gazebo_ros_pkgs` package.

### Modifying Camera Position

To change the camera mounting position, edit:

`src/neo_simulation2/robots/mpo_700/urdf/mpo_700_body.urdf.xacro`

Find the camera instantiation and modify the origin:

```xml
<xacro:gazebo_d435_camera
    camera_name="camera"
    parent_link="base_link"
    update_rate="30">
    <origin xyz="0.292 0.0 0.716" rpy="0 0 0"/>  <!-- Modify this line -->
</xacro:gazebo_d435_camera>
```

**Position Parameters:**
- `xyz`: X (forward), Y (left), Z (up) in meters
- `rpy`: Roll, Pitch, Yaw in radians

After modifying, rebuild:
```bash
colcon build --packages-select neo_simulation2
source install/setup.bash
```

### Modifying Camera Parameters

To change camera resolution, FOV, or update rate:

Edit: `src/neo_simulation2/components/common_macro/gazebo_d435_camera_macro.xacro`

Key parameters to modify:
- `<width>` and `<height>`: Image resolution
- `<horizontal_fov>`: Field of view in radians
- `update_rate`: Frames per second
- `<near>` and `<far>`: Depth range (clip values)

## Troubleshooting

### Camera Topics Not Published

**Problem**: No camera topics appear when running `ros2 topic list`

**Solutions**:
1. Verify simulation is running completely (wait 20+ seconds)
2. Check Gazebo console for plugin errors
3. Rebuild package: `colcon build --packages-select neo_simulation2`
4. Source workspace: `source install/setup.bash`

### Camera Not Visible in Gazebo

**Problem**: Camera link not visible in Gazebo

**Solutions**:
1. The camera is small (90mm × 25mm) - zoom in on front of robot
2. In Gazebo, enable "View" → "Transparent" to see through robot body
3. Check TF tree to verify camera_link exists

### Point Cloud Is Empty

**Problem**: `/camera/depth/points` topic exists but no data

**Solutions**:
1. Check if robot is pointing at objects (point cloud only shows what's in view)
2. Verify depth range (0.2m-10m) - objects outside this range won't appear
3. Check in RViz with proper Fixed Frame (`camera_depth_optical_frame`)

### Depth Image Appears Black

**Problem**: Depth image is all black in rqt_image_view

**Solutions**:
1. This is normal - depth is 16-bit data
2. In rqt_image_view, change encoding to "16UC1"
3. Or use RViz DepthCloud display for better visualization

### Performance Issues

**Problem**: Simulation runs slowly with camera enabled

**Solutions**:
1. Reduce camera resolution in macro (try 320×240)
2. Reduce update rate (try 15 Hz instead of 30 Hz)
3. Disable point cloud generation if not needed
4. Use headless Gazebo: `LIBGL_ALWAYS_SOFTWARE=1 ros2 launch ...`

## Example Applications

### 1. Record Camera Data

Record RGB-D data for offline processing:

```bash
ros2 bag record \
  /camera/color/image_raw \
  /camera/color/camera_info \
  /camera/depth/image_raw \
  /camera/depth/camera_info \
  /camera/depth/points \
  /tf /tf_static
```

### 2. Object Detection

Use camera for object detection with your own node:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge

class CameraSubscriber(Node):
    def __init__(self):
        super().__init__('camera_subscriber')
        self.subscription = self.create_subscription(
            Image,
            '/camera/color/image_raw',
            self.image_callback,
            10)
        self.bridge = CvBridge()

    def image_callback(self, msg):
        cv_image = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        # Process image here
        self.get_logger().info(f'Received image: {cv_image.shape}')

def main():
    rclpy.init()
    node = CameraSubscriber()
    rclpy.spin(node)
```

### 3. Depth-Based Navigation

Use depth data for obstacle avoidance:

```bash
# Subscribe to depth image and point cloud
# Process to detect obstacles in path
# Adjust navigation goals accordingly
```

## Files Modified

This camera integration modified the following files:

### New Files:
- `src/neo_simulation2/components/common_macro/gazebo_d435_camera_macro.xacro`

### Modified Files:
- `src/neo_simulation2/robots/mpo_700/urdf/mpo_700_body.urdf.xacro`

## Further Information

- [Intel RealSense D435 Datasheet](https://www.intelrealsense.com/depth-camera-d435/)
- [Gazebo ROS Camera Plugin Documentation](https://classic.gazebosim.org/tutorials?tut=ros_gzplugins#Camera)
- [ROS 2 sensor_msgs Documentation](https://docs.ros2.org/latest/api/sensor_msgs/index.html)
- [RViz PointCloud2 Display](http://wiki.ros.org/rviz/DisplayTypes/PointCloud)

## License

Same as neo_simulation2 package (MIT License)
