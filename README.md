# Waymo Open Dataset

This is a compensate document for how to access different features of the dataset.


## What the official doc gives us

- How to install all the utils created by themselves.
- Some text description of the dataset and the data inside.
- We use **"the doc"** to refer to the [official documentation](https://waymo.com/open/data/).

## What's new

-  How to really access the dataset. Using code, not human language description.
- This is some kind of explanation of the tutorial provided in the doc.
- Maybe some troubleshooting works. Just maybe.


## Data format and how to load
This dataset is stored in `.tfrecord` files. We first create a `TFRecordDataset`, whose first argument is a list of all paths of the `.tfrecord` files.
```
train_set = tf.data.TFRecordDataset(train_files, compression_type='')
```
Then we can iterate over the `train_set`. Each sample in the dataset is a frame carrying all kinds of information.
```
from waymo_open_dataset import dataset_pb2 as open_dataset
for i, data in enumerate(train_set):
    frame = open_dataset.Frame()
    frame.ParseFromString(bytearray(data.numpy()))
```
Here, `waymo_open_dataset` is the pre-configured package, which is already instructed in the doc. In the `for` loop, the `frame` loads all the information from `data`. We now focus on this single `frame` and see what's inside.

## Tree structure of the dataset classes
- If a node has ⇒ specifying a list of XXX, then the sub-branch (if any) is for each element in the list.
- Capitalized nodes are class names in the source code of the dataset. You can simply convert them to lowercase when writing codes.
	- For 1xample: `frame.context.stats.location`
	- For 2xample: `frame.camera_labels[0].labels[0].box.length`
```
open_dataset
|-- LaserName
|   |-- UNKNOWN
|   |-- TOP
|   |-- FRONT
|   |-- SIDE_LEFT
|   |-- SIDE_RIGHT
|   `-- REAR
|-- CameraName
|   |-- UNKNOWN
|   |-- FRONT
|   |-- FRONT_LEFT
|   |-- FRONT_RIGHT
|   |-- SIDE_LEFT
|   `-- SIDE_RIGHT
|-- RollingShutterReadOutDirection
|   |-- UNKNOWN
|   |-- TOP_TO_BOTTOM
|   |-- LEFT_TO_RIGHT
|   |-- BOTTOM_TO_TOP
|   |-- RIGHT_TO_LEFT
|   `-- GLOBAL_SHUTTER
|-- Frame
|   |-- images ⇒ list of CameraImage
|   |   |-- name (CameraName)
|   |   |-- image
|   |   |-- pose
|   |   |-- velocity (v_x, v_y, v_z, w_x, w_y, w_z)
|   |   |-- pose_timestamp
|   |   |-- shutter
|   |   |-- camera_trigger_time
|   |   `-- camera_readout_done_time
|   |-- Context
|   |   |-- name
|   |   |-- camera_calibrations ⇒ list of CameraCalibration
|   |   |   |-- name
|   |   |   |-- intrinsic
|   |   |   |-- extrinsic
|   |   |   |-- width
|   |   |   |-- height
|   |   |   `-- rolling_shutter_direction (RollingShutterReadOutDirection)
|   |   |-- laser_calibrations ⇒ list of LaserCalibration
|   |   |   |-- name
|   |   |   |-- beam_inclinations
|   |   |   |-- beam_inclination_min
|   |   |   |-- beam_inclination_max
|   |   |   `-- extrinsic
|   |   `-- Stats
|   |       |-- laser_object_counts
|   |       |-- camera_object_counts
|   |       |-- time_of_day
|   |       |-- location
|   |       `-- weather
|   |-- timestamp_micros
|   |-- pose
|   |-- lasers ⇒ list of Laser
|   |   |-- name (LaserName)
|   |   |-- ri_return1 (RangeImage class)
|   |   |   |-- range_image_compressed
|   |   |   |-- camera_projection_compressed
|   |   |   |-- range_image_pose_compressed
|   |   |   `-- range_image
|   |   `-- ri_return2 (same as ri_return1)
|   |-- laser_labels ⇒ list of Label
|   |-- projected_lidar_labels (same as camera_labels)
|   |-- camera_labels ⇒ list of CameraLabels
|   |   |-- name (CameraName)
|   |   `-- labels ⇒ list of Label
|   `-- no_label_zones (Refer to the doc)
`-- Label
    |-- Box
    |   |-- center_x
    |   |-- center_y
    |   |-- center_z
    |   |-- length
    |   |-- width
    |   |-- height
    |   `-- heading
    |-- Metadata
    |   |-- speed_x
    |   |-- speed_y
    |   |-- accel_x
    |   `-- accel_y
    |-- type
    |-- id
    |-- detection_difficulty_level
    `-- tracking_difficulty_level

```
We will only explore the `frame` in detail. Others please refer to this tree. Also for the 6 attributes in `velocity`, I still dunno what they are.

## Attributes of `frame`

### LiDAR data

Lidar data are given by `frame.lasers`. It's an iterable of raw laser data, indicating different lidar positions: top, front, side left, side right, rear. Each element has an attribute `.name` telling the name of its lidar. The lidar names are stored as constants in the `open_dataset` package (similar to macro):
```
open_dataset.LaserName.UNKNOWN    = 0
open_dataset.LaserName.TOP        = 1
open_dataset.LaserName.FRONT      = 2
open_dataset.LaserName.SIDE_LEFT  = 3
open_dataset.LaserName.SIDE_RIGHT = 4
open_dataset.LaserName.REAR       = 5
```
We may want to use point cloud data. In the tutorial given by the doc, they provide a function `parse_range_image_and_camera_projection`:
```
(range_images, camera_projections, range_image_top_pose) = 
parse_range_image_and_camera_projection(frame)
```

The `range_images` here has two dimensions, e.g., `range_images[2][0]`. 
- The first dimension `[2]` indicates which lidar
	> 2 here is front lidar. Refer to `LaserName`
	
- The second dimension `[0]` is for the strongest two intensity returns 
	> 0 is for the strongest, 1 is for the second strongest. 
	There might be some data that has only one return. I haven't tested yet, so I put "might be".

- `range_images[2][0]` is a tuple with length of 4, consistent with the "**4 channels**" listed in the **Lidar Data** section in the doc. 
	> `range_images[2][0].shape` returns `(..., ..., 4)`

After that,  there is `convert_range_image_to_point_cloud`:
```
points, cp_points = convert_range_image_to_point_cloud(frame,
                                                       range_images,
                                                       camera_projections,
                                                       range_image_top_pose)
```
- It converts `range_images` into point cloud, `points`.
- It's a `{[N, 3]}` list of 3D lidar points of length 5 (number of lidars). 
	> `len(points)` returns 5.
	> `points[2].shape` returns `(..., 3)`.
- The returned 3D lidar data is in cartesian coordinate system.
	> Each element in `points[2]` is (x,y,z) coordinate.

### Camera images
Camera images are given by `frame.images`. It's an iterable of images, indicating different camera positions: front, front left, front right, side left, side right. Each element has an attribute `.name` telling the name of its camera. The camera names are stored as constants in the `open_dataset` package (similar to macro):
```
open_dataset.CameraName.UNKNOWN     = 0
open_dataset.CameraName.FRONT       = 1
open_dataset.CameraName.FRONT_LEFT  = 2
open_dataset.CameraName.FRONT_RIGHT = 3
open_dataset.CameraName.SIDE_LEFT   = 4
open_dataset.CameraName.SIDE_RIGHT  = 5
```
Each element also has an attribute `.image` that stores the real image. The camera images are of `JPEG` format. To save them to local storage: 
```
from PIL import Image
import io
image = Image.open(io.BytesIO(frame.images[0].image))
image.save(filename, 'JPEG')
```

### Camera projection of lidar data
TODO: The image together with depth information provided by point cloud projection. There's also projected labels. For current stage, I don't think we are using this part of information.

### Bounding box labels
#### 2D labels
```
front_label = None
for lbl in frame.camera_labels:
    if lbl.name == open_dataset.CameraName.FRONT:
        front_label = lbl
        break

for label in front_label.labels:
    if label.type == 1:
        pass
```
-  2D bbox for camera images
- Only first 3 batches of the training set and the first batch of validation set have 2D labels.
- Stored in `frame.camera_labels`, which has 5 elements indicating 5 different cameras.
	- Each element `lbl` has attribute `.name`, same as `CameraName`, corresponding to one specific camera.
	- Each element contains all bbox information in the image taken by that specific camera.
- In the example, we get the `front_label` for the image taken by the front camera. In `front_label.labels`, it stores all bbox labels in that single image. 
	- Each `label` has an attribute `.type`:
		> For the constants of different types please refer to their [github repo](https://github.com/waymo-research/waymo-open-dataset/blob/master/waymo_open_dataset/label.proto#L58).
		> Here 1 means vehicle.
	- Bbox is given by `box = label.box`
- In a nutshell, the access chain for bbox is `box = frame.camera_labels[0].labels[0].box`.
- Now we look into the `box`:
 	- `box.width` is the top-bottom distance of the image.
	- `box.length` is the left-right distance of the image.
	- `box.center_x` x-coordinate of the image is along the `length` direction.
	- `box.center_y` y-coordinate of the image is along the `width` direction.

#### 3D labels
```
for label in frame.laser_labels:
    if label.type == 1:
        pass
```
- This is different from camera labels. It does not distinguish which lidar it's using. `frame.laser_labels` simply stores all bboxes in all directions in that frame.
	> The doc provides no information on how these labels are related to different lidars. 
	
- The other attributes of `label` are pretty similar. Except that the `box` is 3D. Here are the correspondence:
	- `box.length` <=> `box.center_x` along the "front/forward" direction. 
	- `box.width` <=> `box.center_y` along the "left" direction.
	- `box.height` <=> `box.center_z` along the "upward/pointing-to-the-sky" direction.
	- Please refer to the doc about **Vehicle frame** in the **Coordinate systems** section.

### Misc
This subsubsection lists all other features that might potentially be useful.
- For `waymo.open_dataset.Label` class, aka the `label` variable as whose instance in previous section, they have metadata for speed and acceleration. 
	- `label.metadata.speed_x`
	- `label.metadata.speed_y`
	- `label.metadata.accel_x`
	- `label.metadata.accel_y`
	- The unit of the data still unknown.
- However, only 3D labels have such metadata. (Note that these are for bboxes, not for the ego car itself). The data for the car itself are stored in the `image.velocity` (refer to the tree structure), but what does `w_x` stand for? Angular velocity?

## Troubleshooting
- `tf.enable_eager_execution()` is required for tf1.x to load the dataset. tf2.x has eager execution by default.
- Built dataset using pip3 install in the [tutorial](https://github.com/waymo-research/waymo-open-dataset/blob/master/tutorial/tutorial.ipynb), but still cannot import `dataset_pb2`:
	- Solution: use protoc to manually generate python files like in [this issue](https://github.com/waymo-research/waymo-open-dataset/issues/35#issuecomment-536451837)
- Using code based on Kitti dataset? See [this repo](https://github.com/caizhongang/waymo_kitti_converter) for a convertion tool. It fixed the difference in the coordination systems between Waymo and Kitti dataset, so you can visualize the dataset with [kitti_object_vis](https://github.com/kuixu/kitti_object_vis) with no trouble.
- How to read `gt.bin` file: [this issue](https://github.com/waymo-research/waymo-open-dataset/issues/142#issuecomment-619359939).

## Differentiable LiDAR-to-depth
- Get gradients of LiDAR point coordinates:
	- In the tutorial, they already showed us how to project points onto the image with `points_all` and `cp_points_all`.
	- `cp_points_all` stores the image pixel location, and the norm of `points_all` is the depth value of the corresponding pixel.
- Modifying the original point cloud:
	- After modifying the coordinates, we need to re-generate `cp_points_all`.
	- They provided a [third party tool](https://github.com/waymo-research/waymo-open-dataset/tree/master/third_party/camera) for this, please refer to [this issue](https://github.com/waymo-research/waymo-open-dataset/issues/127).
	- Note that the generation of pixel location (`cp_points_all`) doesn't have to be differentiable, so we can just use the third party tool.

---
Reference:
@misc{waymo_open_dataset,
  title = {Waymo Open Dataset: An autonomous driving dataset},
  website = {\url{https://www.waymo.com/open}},
  year = {2019}
}

