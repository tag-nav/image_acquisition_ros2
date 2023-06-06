# Image Acquisition by FLIR/Spinnaker ROS2 driver

This is a ROS2 driver for the FLIR cameras using the [Spinnaker SDK](https://www.flir.com/products/spinnaker-sdk/?vertical=machine+vision&segment=iis).

NOTE: This code is forked from Bernd Pfrommer's repository which is not written or supported by FLIR.

## Supported platforms

Software:

- Ubuntu 20.04 LTS
- ROS2 Galactic and Foxy
- Spinnaker 2.6.0.157 (other versions may work as well but this is what the continuous integration builds are using)

The code compiles under Ubuntu 22.04 / Humble but has not been tested thoroughly with real hardware. Currently known to work with Blackfly S (USB3) using Spinnaker 2.6.0.157.


## Features

Basic features are supported like setting exposure, gain, and external triggering. It's straight forward to support new camera types and features by editing the camera definition (.cfg) files. Unless you need new pixel formats you may not have to modify any source code. The code is meant to be a thin wrapper for setting the features available in FLIR's SpinView program. The driver has following parameters, *in addition to the parameters defined in the .cfg files*:

- ``serial_number``: must have the serial number of the camera. If you don't know it, put in anything you like and the driver will croak with an error message, telling you what cameras serial numbers are available
- ``frame_id``: the ROS frame id to put in the header of the published
  image messages.
- ``camerainfo_url``: where to find the camera calibration yaml file.
- ``parameter_file``: location of the .cfg file defining the camera
  (blackfly_s.cfg etc)
- ``compute_brightness``: if true, compute image brightness and
  publish it in meta data message. This is useful for external
  exposure control but incurs extra CPU load. Default: false.
- ``buffer_queue_size``: max number of images to queue internally
  before shoving them into the ROS output queue. Decouples the
  Spinnaker SDK thread from the ROS publishing thread. Default: 4.
- ``image_queue_size``: ROS output queue size (quality of
  service). Default: 4
- ``dump_node_map``: set this to true to get a dump of the node map. This
  feature is helpful when hacking a new config file. Default: false.
  

## How to build

1) Install the FLIR spinnaker driver.
2) Prepare the ROS2 driver build:
Make sure you have your ROS2 environment sourced:
```
source /opt/ros/galactic/setup.bash
```

Create a workspace (``flir_spinnaker_ros2_ws``), clone this repo, and use ``wstool``
to pull in the remaining dependencies:

```
mkdir -p ~/flir_spinnaker_ros2_ws/src
cd ~/flir_spinnaker_ros2_ws
git clone https://github.com/berndpfrommer/flir_spinnaker_ros2 src/flir_spinnaker_ros2
wstool init src src/flir_spinnaker_ros2/flir_spinnaker_ros2.rosinstall

# or to update an existing space
# wstool merge -t src src/flir_spinnaker_ros2/flir_spinnaker_ros2.rosinstall
# wstool update -t src
```

To automatically install all packages that the ``flir_spinnaker_ros2``
depends upon, run this at the top of your workspace:
```
rosdep install --from-paths src --ignore-src
```

3) Build the driver and source the workspace:
```
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
. install/setup.bash
```

## Enable Jumbo packet 

Jumbo Packet is strongly recommended to be enabled for the network adapter and the camera, in order to greatly improve streaming results.

Run ifconfig and find the network adapter that the cameras are connected to (eg. enp15s0):
```
ifconfig
```
To temporarily update enable Jumbo Packet until the next reboot, for a specific network adapter, eg. enp15s0, run the following commands:
```
sudo ifconfig enp15s0 mtu 9000
```

## Example usage

How to launch the example file:
```
ros2 launch flir_spinnaker_ros2 blackfly_s_gige.launch.py camera_name:=blackfly_0 serial:="'20435008'"
```

## Topics

### Publisher:
`/image_raw` (type: `sensor_msgs/msg/Image`)
`/camera_info` (type: `sensor_msgs/msg/CameraInfo`)

[Note] By default, the values of the parameters in `/camera_info` are initialized to zero. The values need to be provided through [camera calibration](https://github.com/ros-perception/image_pipeline) and subsequently stored in `camerainfo_url` yaml file.

## Camera synchronization

In the ``launch`` folder you can find a working example for launching
drivers for two hardware synchronized Blackfly S cameras. The launch
file requires two more packages to be installed,
[cam_sync_ros2](https://github.com/berndpfrommer/cam_sync_ros2)(for
time stamp syncing) and
[exposure_control_ros2](https://github.com/berndpfrommer/exposure_control_ros2)
(for external exposure control). See below for more details on those packages.

### Time stamps

By default the driver will set the ROS header time stamp to be the
time when the image was delivered by the SDK. Such time stamps are not
very precise and may lag depending on host CPU load. However the
driver has a feature to use the much more accurate sensor-provided
camera time stamps. These are then converted to ROS time stamps by
estimating the offset between ROS and sensor time stamps via a simple
moving average. For the adjustment to work
*the camera must be configured to send time stamps*, and the
``adjust_timestamp`` flag must be set to true, and the relevant field
in the "chunk" must be populated by the camera. For the Blackfly S
the parameters look like this:

```
    'adjust_timestamp': True,
    'chunk_mode_active': True,
    'chunk_selector_timestamp': 'Timestamp',
    'chunk_enable_timestamp': True,
```

When running hardware synchronized cameras in a stereo configuration
two drivers will need to be run, one for each camera. This will mean
however that their published ROS header time stamps are *not*
identical which in turn may prevent down-stream ROS nodes from recognizing the
images as being hardware synchronized. You can use the
[cam_sync_ros2 node](https://github.com/berndpfrommer/cam_sync_ros2)
to force the time stamps to be aligned. In this scenario it is
mandatory to configure the driver to adjust the ROS time stamps as
described above.

