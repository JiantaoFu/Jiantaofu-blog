title: obs视频采集源码分析
comments: true
date: 2015-07-12 12:00:06
tags:
keywords:
description:
thumbnailImage:
coverImage:
photos:
---
#obs视频采集源码分析

obs采用的plugin的方式来进行视频采集，下面以OSX下的视频采集插件为示例。

相关的文件:

  - plugins/av-capture.m
  - plugins/AVCaptureInputPort+PreMavericksCompat.h
  - libobs/obs.c
  - libobs/media-io/video-io.c
  - libobs/media-io/video-frame.c 
  - libobs/obs-output.c
  - obs/window-basic-main.cpp
  - obs/window-basic-main-outputs.cpp

这里定义了插件接口的实现:

```c
struct obs_source_info av_capture_info = {
	.id             = "av_capture_input",
	.type           = OBS_SOURCE_TYPE_INPUT,
	.output_flags   = OBS_SOURCE_ASYNC_VIDEO,
	.get_name       = av_capture_getname,
	.create         = av_capture_create,
	.destroy        = av_capture_destroy,
	.get_defaults   = av_capture_defaults,
	.get_properties = av_capture_properties,
	.update         = av_capture_update,
};
```

av\_capture\_create的流程基本上演示了AVFoundation的VideoCapture的基本api调用: 

	  av_capture_create -> av_capture_init -> init_session -> dispatch_queue_create
	                                                       -> addOutput
	                                                       -> setSampleBufferDelegate
	                                       -> AVCaptureDeviceWasDisconnectedNotification
	                                       -> AVCaptureDeviceWasConnectedNotification
	                                       -> deviceWithUniqueID
	                                       -> capture_device -> init_device_input(add AVCaptureDeviceInput)
	                                                         -> init_format(kCVPixelBufferPixelFormatTypeKey)
	                                                         -> startRunning
	                    -> av_capture_enable_buffering(set/unset OBS_SOURCE_FLAG_UNBUFFERED)
	                    
其中包括了创建AVCaptureSession用来协调输入输出，创建AVCaptureVideoDataOutput来获取输出， OBSAVCaptureDelegate来处理每一帧的数据，创建AVCaptureDeviceInput用来从AVCaptureDevice中采集符合Preset设置的格式的数据。

下面实现了采集Frame的回调函数，其中调用了update\_frame将输出转化为obs内部的frame数据结构，然后调用obs\_source\_output\_video将数据存储在cache中(cache\_video)：

```c
- (void)captureOutput:(AVCaptureOutput *)captureOutput
        didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
        fromConnection:(AVCaptureConnection *)connection
{
	UNUSED_PARAMETER(captureOutput);
	UNUSED_PARAMETER(connection);

	CMItemCount count = CMSampleBufferGetNumSamples(sampleBuffer);
	if (count < 1 || !capture)
		return;

	struct obs_source_frame *frame = &capture->frame;

	CMTime target_pts =
		CMSampleBufferGetOutputPresentationTimeStamp(sampleBuffer);
	CMTime target_pts_nano = CMTimeConvertScale(target_pts, NANO_TIMESCALE,
			kCMTimeRoundingMethod_Default);
	frame->timestamp = target_pts_nano.value;

	if (!update_frame(capture, frame, sampleBuffer))
		return;

	obs_source_output_video(capture->source, frame);

	CVImageBufferRef img = CMSampleBufferGetImageBuffer(sampleBuffer);
	CVPixelBufferUnlockBaseAddress(img, kCVPixelBufferLock_ReadOnly);
}
```

当通过UI操作添加视频源(OBSBasicSourceSelect::on\_buttonBox\_accepted)时，会调用obs\_source\_create，obs\_source\_create会调用av\_capture\_info.create。

```c
        if (info)
                source->context.data = info->create(source->context.settings,
                                source);
```

回过头在看一下Video相关的一些初始化参数:

```c
bool OBSBasic::InitBasicConfigDefaults()
{
...
        config_set_default_uint  (basicConfig, "Video", "BaseCX",   cx);
        config_set_default_uint  (basicConfig, "Video", "BaseCY",   cy);

        cx = cx * 10 / 15;
        cy = cy * 10 / 15;
        config_set_default_uint  (basicConfig, "Video", "OutputCX", cx);
        config_set_default_uint  (basicConfig, "Video", "OutputCY", cy);

        config_set_default_uint  (basicConfig, "Video", "FPSType", 0);
        config_set_default_string(basicConfig, "Video", "FPSCommon", "30");
        config_set_default_uint  (basicConfig, "Video", "FPSInt", 30);
        config_set_default_uint  (basicConfig, "Video", "FPSNum", 30);
        config_set_default_uint  (basicConfig, "Video", "FPSDen", 1);
        config_set_default_string(basicConfig, "Video", "ScaleType", "bicubic");
        config_set_default_string(basicConfig, "Video", "ColorFormat", "NV12");
        config_set_default_string(basicConfig, "Video", "ColorSpace", "709");
        config_set_default_string(basicConfig, "Video", "ColorRange",
                        "Partial");
...
}
```

Video初始化过程:

	OBSBasic::OBSInit() -> OBSBasic::ResetVideo() -> AttemptToResetVideo -> obs_reset_video -> obs_init_video
	
这里obs\_init\_video创建了两个线程，video\_thread用于输出(streaming/recording)，obs\_video\_thread用于预览。

	obs_init_video -> video_output_open -> video_thread -> video_output_cur_frame -> callback
	               -> obs_video_thread -> output_frame -> render_video

下面是rtmp streaming的初始化，在这里会创建一个obs\_video\_thread线程，并开始video encoder，并设置采集到video frame后的callback为receive\_video，在receive\_video里面会做视频编码。
	
	StartStreaming -> obs_output_start -> rtmp_stream_start(connect_thread)
	connect_thread -> init_send -> obs_output_begin_data_capture -> hook_data_capture -> obs_encoder_start -> add_connection -> video_output_connect(receive_video)
	receive_video -> do_encode
	
接下来再看看视频采集出来的format，前面的初始化的配置里面ColorFormat设置为NV12，不同的Format在内存中分配的数据结构不一样，从video\_frame\_init里面可以看到，现在支持的Format包括：

  - VIDEO\_FORMAT\_I420
  - VIDEO\_FORMAT\_NV12
  - VIDEO\_FORMAT\_YVYU
  - VIDEO\_FORMAT\_YUY2
  - VIDEO\_FORMAT\_UYVY
  - VIDEO\_FORMAT\_RGBA
  - VIDEO\_FORMAT\_BGRA
  - VIDEO\_FORMAT\_BGRX
  - VIDEO\_FORMAT\_I444

其中VIDEO\_FORMAT\_YVYU，VIDEO\_FORMAT\_YUY2，VIDEO\_FORMAT\_UYVY的数据结构一样；VIDEO\_FORMAT\_RGBA，VIDEO\_FORMAT\_BGRA，VIDEO\_FORMAT\_BGRX的数据结构一样。NV12的数据帧内存结构如下:

![Single Frame YUV420: NV12](http://p.blog.csdn.net/images/p_blog_csdn_net/hhygcy/EntryImages/20081125/nv12.png)

更多格式可以参考[这里](http://www.cnblogs.com/azraelly/archive/2013/01/01/2841269.html)

对应video\_frame\_init中NV12的代码如下:

```c
        case VIDEO_FORMAT_NV12:
                size = width * height;
                ALIGN_SIZE(size, alignment);
                offsets[0] = size;
                size += (width/2) * (height/2) * 2;
                ALIGN_SIZE(size, alignment);
                frame->data[0] = bmalloc(size);
                frame->data[1] = (uint8_t*)frame->data[0] + offsets[0];
                frame->linesize[0] = width;
                frame->linesize[1] = width;
                break;
```
首先存储Y需要width * height字节，因为每4个Y分别对应一个U和V，所以再加上width/2 * height/2 * 2。这里的data[0]指向的就是frame开始的存储地址，data[1]是UV分量的起始地址。NV12是一种two-plane模式，即Y和UV分为两个Plane，但是UV（CbCr）为交错存储。linesize[0]是Y Plane跳到下一个行的字节数，linesize[1]是UV Plane跳到下一个行的字节数。更多请参考[这里](http://stackoverflow.com/questions/13286022/can-anyone-help-in-understanding-avframe-linesize)

在mac下面，摄像头采集出来的ColorFormat是UYVY，每两个相邻的Y分别对应一个U和V。YUY2和YVYU的内存大小一样，唯一的区别是YUV分量的顺序有些区别。下面是UYVY的结构：

![UYVY](http://img1.51cto.com/attachment/201104/202455202.png)

对应video\_frame\_init中UYVY的代码如下:

```c
        case VIDEO_FORMAT_YVYU:
        case VIDEO_FORMAT_YUY2:
        case VIDEO_FORMAT_UYVY:
                size = width * height * 2;
                ALIGN_SIZE(size, alignment);
                frame->data[0] = bmalloc(size);
                frame->linesize[0] = width*2;
                break;
```
从上面的内存结构可以看出存储需要的字节数时width * height * 2，data[0]指向的就是frame开始的存储地址，linesize[0]是Plane跳到下一个行的字节数，从上图可以看出这个是width个字节。