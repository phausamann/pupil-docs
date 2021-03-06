+++
date = "2017-01-19T15:45:05+07:00"
title = "data format"
section_weight = 3
page_weight = 4
+++

## Data Format

The data format for Pupil recordings is 100% open. In this section we will first describe how we handle high level concepts like coordinate systems, timestamps and synchronization and then describe in detail the data format present in Pupil recordings.

#### Coordinate Systems
We use a normalized coordinate system with the origin `0,0` at the bottom left and `1,1` at the top right.

* Normalized Space

  Origin `0,0` at the bottom left and `1,1` at the top right. This is the OpenGL convention and what we find to be an intuitive representation. This is the coordinate system we use most in Pupil. Vectors in this coordinate system are specified by a `norm` prefix or suffix in their variable name.

* Image Coordinate System

  In some rare cases we use the image coordinate system. This is mainly for pixel access of the image arrays. Here a unit is one pixel, origin is "top left" and "bottom right" is the maximum x,y.

#### Timestamps
All indexed data - still frames from the world camera, still frames from the eye camera(s), gaze coordinate, and pupil coordinates, etc. - have timestamps associated for synchronization purposes. The timestamp is derived from `CLOCK_MONOTONIC` on Linux and MacOS.

The time at which the clock starts counting is called PUPIL EPOCH. In pupil the epoch is adjustable through `Pupil Remote` and `Pupil Timesync`.

Timestamps are recorded for each sensor separately. Eye and World cameras may be capturing at very different rates (e.g. 120hz eye camera and 30hz world camera), and correlation of eye and world (and other sensors) can be done after the fact by using the timestamps. For more information on this see [Synchronization](#synchronization) below.

Observations:

- Timestamps in seconds since PUPIL EPOCH.
- PUPIL EPOCH is usually the time since last boot.
- In UNIX like, PUPIL EPOCH is usually **not** the Unix Epoch (00:00:00 UTC on 1 January 1970).

More information:

- Unit : Seconds
- Precision: Full float64 precision with 15 significant digits, i.e. 10 μs.
- Accuracy:
   - If WIFI, it is ~1 ms
   - If wired or 'localhost', it is in the range of μs.
- Granularity:
   - It is machine specific (depends on clock_monotonic on Linux). It is constrained by the processor cycles and
     software.
   - In some machines (2 GHz processor), the result comes from clock_gettime(CLOCK_MONOTONIC, &time_record)
     function on Linux. This function delivers a record with nanosecond, 1 GHz, granularity. Then, PUPIL software does
     some math and delivers a float64.
- Maximum Sampling Rate:
   - Depends on set-up, and it is lower when more cameras are present. (120Hz maximum based on a 5.7ms latency for
     the cameras and a 3.0ms processing latency.

#### Synchronization

Pupil Capture software runs multiple processes. The world video feed and the eye video feeds run and record at the frame rates set by their capture devices (cameras). This allows us to be more flexible. Instead of locking everything into one frame rate, we can capture every feed at specifically set rates. But, this also means that we sometimes record world video frames with multiple gaze positions (higher eye-frame rate) or without any (no pupil detected or lower eye frame rate).

In `player_methods.py` you can find a function that takes timestamped data and correlates it with  timestamps form a different source.

```python
def correlate_data(data,timestamps):
    '''
    data:  list of data :
        each datum is a dict with at least:
            timestamp: float

    timestamps: timestamps list to correlate  data to

    this takes a data list and a timestamps list and makes a new list
    with the length of the number of timestamps.
    Each slot contains a list that will have 0, 1 or more associated data points.

    Finally we add an index field to the datum with the associated index
    '''
    timestamps = list(timestamps)
    data_by_frame = [[] for i in timestamps]

    frame_idx = 0
    data_index = 0

    data.sort(key=lambda d: d['timestamp'])

    while True:
        try:
            datum = data[data_index]
            # we can take the midpoint between two frames in time: More appropriate for SW timestamps
            ts = ( timestamps[frame_idx]+timestamps[frame_idx+1] ) / 2.
            # or the time of the next frame: More appropriate for Sart Of Exposure Timestamps (HW timestamps).
            # ts = timestamps[frame_idx+1]
        except IndexError:
            # we might loose a data point at the end but we don't care
            break

        if datum['timestamp'] <= ts:
            datum['index'] = frame_idx
            data_by_frame[frame_idx].append(datum)
            data_index +=1
        else:
            frame_idx+=1

    return data_by_frame
```


### Detailed Data Format

Every time you click record in Pupil Capture or Pupil Mobile, a new recording is started and your data is saved into a recording folder. You can use [Pupil Player](#pupil-player) to playback Pupil recordings, add visualizations, and export in various formats. 

#### Access to raw data
Note that the raw data before processing with Pupil Player is not immediately readible from other software (the raw data format is documented in the [developer docs](#recording-format)). Use the 'Raw Data Exporter' plugin in Pupil Player to export `.csv` files that contain all the data captured with Pupil Capture. Exported files will be written to a subfolder of the recording folder called `exports`.

The following files will be created by default in an export:

- export_info.csv - Meta information on the export containing e.g. the export date or the dta format version.
- pupil_positions.csv - A list of all pupil datums. See below for more infotmation.
- gaze_positions.csv - A list of all gaze datums. See below for more infotmation.
- pupil_gaze_positions_info.txt - Contains documentation on the contents of pupil_positions.csv and gaze_positions.csv
- world_viz.mp4 - The exported section of world camera video.

If you are using additional plugins in Pupil Player, these might create other files. Please check the documentation of the respective plugin for the used data format.


#### pupil_positions.csv
This file contains all exported pupil datums. Each datum will have at least the following keys:

* `timestamp` - timestamp of the source image frame
* `index` - associated_frame: closest world video frame
* `id` - 0 or 1 for left/right eye
* `confidence` - is an assessment by the pupil detector on how sure we can be on this measurement. A value of `0` indicates no confidence. `1` indicates perfect confidence. In our experience useful data carries a confidence value greater than ~0.6. A `confidence` of exactly `0` means that we don't know anything. So you should ignore the position data.
* `norm_pos_x` - x position in the eye image frame in normalized coordinates
* `norm_pos_y` - y position in the eye image frame in normalized coordinates
* `diameter` - diameter of the pupil in image pixels as observed in the eye image frame (is not corrected for perspective)
* `method` - string that indicates what detector was used to detect the pupil

Depending on the used gaze mapping mode, each datum may have additional keys. When using the **2D gaze mapping** mode the following keys will be added:

* `2d_ellipse_center_x` - x center of the pupil in image pixels
* `2d_ellipse_center_y` - y center of the pupil in image pixels
* `2d_ellipse_axis_a` - first axis of the pupil ellipse in pixels
* `2d_ellipse_axis_b` - second axis of the pupil ellipse in pixels
* `2d_ellipse_angle` - angle of the ellipse in degrees

When using the **3D gaze mapping** mode the following keys will *additionally* be added:

* `diameter_3d` - diameter of the pupil scaled to mm based on anthropomorphic avg eye ball diameter and corrected for perspective.
* `model_confidence` - confidence of the current eye model (0-1)
* `model_id` - id of the current eye model. When a slippage is detected the model is replaced and the id changes.
* `sphere_center_x` - x pos of the eyeball sphere is eye pinhole camera 3d space units are scaled to mm.
* `sphere_center_y` - y pos of the eye ball sphere
* `sphere_center_z` - z pos of the eye ball sphere
* `sphere_radius` - radius of the eyeball. This is always 12mm (the anthropomorphic avg.) We need to make this assumption because of the `single camera scale ambiguity`.
* `circle_3d_center_x` - x center of the pupil as 3d circle in eye pinhole camera 3d space units are mm.
* `circle_3d_center_y` - y center of the pupil as 3d circle
* `circle_3d_center_z` - z center of the pupil as 3d circle
* `circle_3d_normal_x` - x normal of the pupil as 3d circle. Indicates the direction that the pupil points at in 3d space.
* `circle_3d_normal_y` - y normal of the pupil as 3d circle
* `circle_3d_normal_z` - z normal of the pupil as 3d circle
* `circle_3d_radius` - radius of the pupil as 3d circle. Same as `diameter_3d`
* `theta` - circle_3d_normal described in spherical coordinates
* `phi` - circle_3d_normal described in spherical coordinates
* `projected_sphere_center_x` - x center of the 3d sphere projected back onto the eye image frame. Units are in image pixels.
* `projected_sphere_center_y` - y center of the 3d sphere projected back onto the eye image frame
* `projected_sphere_axis_a` - first axis of the 3d sphere projection.
* `projected_sphere_axis_b` - second axis of the 3d sphere projection.
* `projected_sphere_angle` - angle of the 3d sphere projection. Units are degrees.

#### gaze_positions.csv
This file contains a list of all exported gaze datums. Each datum contains the following keys:

* `timestamp` - timestamp of the source image frame
* `index` - associated_frame: closest world video frame
* `confidence` - computed confidence between 0 (not confident) -1 (confident)
* `norm_pos_x` - x position in the world image frame in normalized coordinates
* `norm_pos_y` - y position in the world image frame in normalized coordinates
* `base_data` - "timestamp-id timestamp-id ..." of pupil data that this gaze position is computed from

When using the **3D gaze mapping** mode the following keys will additionally be available:

* `gaze_point_3d_x` - x position of the 3d gaze point (the point the subject looks at) in the world camera coordinate system
* `gaze_point_3d_y` - y position of the 3d gaze point
* `gaze_point_3d_z` - z position of the 3d gaze point
* `eye_center0_3d_x` - x center of eye-ball 0 in the world camera coordinate system (of camera 0 for binocular systems or any eye camera for monocular system)
* `eye_center0_3d_y` - y center of eye-ball 0
* `eye_center0_3d_z` - z center of eye-ball 0
* `gaze_normal0_x` - x normal of the visual axis for eye 0 in the world camera coordinate system (of eye 0 for binocular systems or any eye for monocular system). The visual axis goes through the eye ball center and the object thats looked at.
* `gaze_normal0_y` - y normal of the visual axis for eye 0
* `gaze_normal0_z` - z normal of the visual axis for eye 0
* `eye_center1_3d_x` - x center of eye-ball 1 in the world camera coordinate system (not available for monocular setups.)
* `eye_center1_3d_y` - y center of eye-ball 1
* `eye_center1_3d_z` - z center of eye-ball 1
* `gaze_normal1_x` - x normal of the visual axis for eye 1 in the world camera coordinate system (not available for monocular setups.). The visual axis goes through the eye ball center and the object thats looked at.
* `gaze_normal1_y` - y normal of the visual axis for eye 1
* `gaze_normal1_z` - z normal of the visual axis for eye 1


#### World Video Stream

> You can compress the videos afterwards using ffmpeg like so:

```
cd your_recording
ffmpeg -i world.mp4  -pix_fmt yuv420p  world_compressed.mp4; mv world_compressed.mp4 world.mp4 
ffmpeg -i eye0.mp4  -pix_fmt yuv420p  eye0_compressed.mp4; mv eye0_compressed.mp4 eye0.mp4
ffmpeg -i eye1.mp4  -pix_fmt yuv420p  eye1_compressed.mp4; mv eye1_compressed.mp4 eye1.mp4
```

> OpenCV has a capture module that can be used to extract still frames from the video:

```python
import cv2
capture = cv2.VideoCapture("absolute_path_to_video/world.mp4")
status, img1 = capture.read() # extract the first frame
status, img2 = capture.read() # second frame...
```

When using the setting `more CPU smaller file`: A `mpeg4` compressed video stream of the world view will be created in an `.mp4` container. The video is compressed using ffmpeg's default settings. It gives a good balance between image quality and files size. The frame rate of this file is set to your capture frame rate.

When using the setting `less CPU bigger file`: A raw `mjpeg` stream from the world camera world view will be created in an `.mp4` container. The video is compressed by the camera itself. While the file size is considerably larger than above, this will allow ultra low CPU while recording. It plays with recent version of ffmpeg and vlc player. The "frame rate" setting in the Pupil Capture sidebar (Camera Settings > Sensor Settings) controls the frame rate of the videos.


#### head_pose_tacker_model.csv and head_pose_tacker_poses.csv

- `head_pose_tacker_model.csv`: A list of all markers used to generate the 3d model and the 3d locations of the marker vertices.
- `head_pose_tacker_poses.csv`: The headset's pose (rotation and translation) within the 3d model coordinate system for each recorded world frame.

By default, the location of the first marker occurance will be used as the origin of the 3d model's coordinate system. In the plugin's menu, you can change the marker that is being used as the origin.
