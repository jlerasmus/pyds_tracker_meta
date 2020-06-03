# pyds_tracker_meta

[pybind11](https://github.com/pybind/pybind11) wrapper to access Nvidia DeepStream tracker meta info (`NvDsPastFrame...` classes) from Python.

This library provides access to the `NvDsPastFrameObjBatch` type user metadata of DeepStream data streams. This metadata is generated by the object trackers of [nvtracker](https://docs.nvidia.com/metropolis/deepstream/dev-guide/#page/DeepStream%20Plugins%20Development%20Guide/deepstream_plugin_details.3.02.html#) plugin. The C API of these structures can be found in [nvds_tracker_meta.h](https://docs.nvidia.com/metropolis/deepstream/dev-guide/DeepStream_Development_Guide/baggage/nvds__tracker__meta_8h.html) header file of DeepStream SDK. The instances of these metadata structures contain information about the object tracking results in past frames. 

Some trackers do not reported the state of the tracked object if there are no current matching detection results. If in a later frame a detection result confirms the state of the tracked object, the past history of the tracking is reported retroactively in `NvDsPastFrame...` metadata. This library provides Python access to this metadata. For more information refer to the [nvtracker](https://docs.nvidia.com/metropolis/deepstream/dev-guide/#page/DeepStream%20Plugins%20Development%20Guide/deepstream_plugin_details.3.02.html#) plugin documentation.

## Installation

### Prerequisites

1. Install [pybind11](https://github.com/pybind/pybind11). The recommended way is to [build it from source](https://pybind11.readthedocs.io/en/stable/basics.html?highlight=install#compiling-the-test-cases). Alternatively you might try simply `pip3 install pybind11`.
2. You should have `gstreamer-1.0` and `gstreamer-video-1.0` packages installed in your system. If you are using DeepStream, you probably have these packages already.
3. You will need also the standard `c++` compiler that you usually find in Linux distribution. `c++11` standard is used.

### Compile the source

1. The source should be compiled on your target platform (Jetson or x86).
2. Set your DeepStream version and path in `build.sh`.
3. Launch `build.sh`
4. Copy the compiled library (eg. `pyds_tracker_meta.cpython-36m-aarch64-linux-gnu.so`) to your python `site-packages` folder, or alternatively to the folder where your python script will run.

## Usage

`pyds_tracker_meta` is meant to be used together with the standard [Python bindings for DeepStream](https://github.com/NVIDIA-AI-IOT/deepstream_python_apps). Make sure you have `pyds` available.

Ensure you have set `enable-past-frame` property of the `gst-nvtracker` plugin to `1`. (See [nvtracker](https://docs.nvidia.com/metropolis/deepstream/dev-guide/#page/DeepStream%20Plugins%20Development%20Guide/deepstream_plugin_details.3.02.html#) plugin documentation.)

Most likely you will use this library from the buffer probe callbacks of a gstreamer plugin pad, when the object tracking results are available. The [deepstream-test2](https://github.com/NVIDIA-AI-IOT/deepstream_python_apps/tree/master/apps/deepstream-test2) python app shows you how to set up such a callback. 

The example snippet provided bellow shows how to cast a user meta to a past frame object batch, and how to access all fields of the metadata. Add the following lines to the `osd_sink_pad_buffer_probe` method found int `deepstream-test2.py`, just after the [`batch_meta` was acquired](https://github.com/NVIDIA-AI-IOT/deepstream_python_apps/blob/2931f6b295b58aed15cb29074d13763c0f8d47be/apps/deepstream-test2/deepstream_test_2.py#L61):

```python
def osd_sink_pad_buffer_probe(pad,info,u_data):
    
    # ... code to acquire batch_meta ...
    
    user_meta_list = batch_meta.batch_user_meta_list
    while user_meta_list is not None:
        user_meta = pyds.NvDsUserMeta.cast(user_meta_list.data)
        
        print('user_meta:', user_meta)
        print('user_meta.user_meta_data:', user_meta.user_meta_data)
        print('user_meta.base_meta:', user_meta.base_meta)
        if not pyds_tracker_meta.NvDsPastFrameObjBatch.user_meta_is_past_frame_obj_batch(user_meta):
            continue
        past_frame_object_batch = pyds_tracker_meta.NvDsPastFrameObjBatch.from_user_meta(user_meta)
        print('past_frame_object_batch:', past_frame_object_batch)
        print('  list:')
        for past_frame_object_stream in past_frame_object_batch.list:
            print('    past_frame_object_stream:', past_frame_object_stream)
            print('      streamID:', past_frame_object_stream.streamID)
            print('      surfaceStreamID:', past_frame_object_stream.surfaceStreamID)
            print('      list:')
            for past_frame_object_list in past_frame_object_stream.list:
                print('        past_frame_object_list:', past_frame_object_list)
                print('          numObj:', past_frame_object_list.numObj)
                print('          uniqueId:', past_frame_object_list.uniqueId)
                print('          classId:', past_frame_object_list.classId)
                print('          objLabel:', past_frame_object_list.objLabel)
                print('          list:')
                for past_frame_object in past_frame_object_list:
                    print('            past_frame_object:', past_frame_object)
                    print('              frameNum:', past_frame_object.frameNum)
                    print('              tBbox.left:', past_frame_object.tBbox.left)
                    print('              tBbox.width:', past_frame_object.tBbox.width)
                    print('              tBbox.top:', past_frame_object.tBbox.top)
                    print('              tBbox.right:', past_frame_object.tBbox.height)
                    print('              confidence:', past_frame_object.confidence)
                    print('              age:', past_frame_object.age)
        
        try:
            user_meta_list = user_meta_list.next
        except StopIteration:
            break
```