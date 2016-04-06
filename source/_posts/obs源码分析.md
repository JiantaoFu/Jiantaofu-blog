title: obs源码分析
comments: true
date: 2015-08-13 21:32:20
tags:
keywords:
description:
thumbnailImage:
coverImage:
photos:
---
#obs源码分析

##编译源码

分析源码的第一步是编译源码，这样才能边调试边学习。obs studio支持cmake，在mac osx下可以生成Xcode的工程文件，这样方便调试。obs依赖的库有ffmpeg, qt5等，用brew可以很方便的安装。

```
brew install qt5
```

需要注意的是用cmake生成工程文件前需要设置环境变量:

```
export CMAKE_PREFIX_PATH=/usr/local/opt/qt5/
```

生成好了工程文件后，设置一下工作目录，否则会找不到locale文件报错。

```
$(SRCROOT)/macbuild/rundir/Debug/bin/
```

##线程分析

接下来可以把程序跑起来了，可以调试运行，让程序暂停，然后分析一下程序有哪些主要的线程。

程序起来开始streaming前可以看到有三个线程:

  - "audio-io: audio thread": 音频采集
  - "video-io: video thread": 视频采集
  - "libobs: graphics thread": 视频预览

##基本数据结构分析

###视频部分

```c
struct video_data {
        uint8_t           *data[MAX_AV_PLANES];
        uint32_t          linesize[MAX_AV_PLANES];
        uint64_t          timestamp;
};
```

```c
struct video_input {
        struct video_scale_info   conversion;
        video_scaler_t            *scaler;
        struct video_frame        frame[MAX_CONVERT_BUFFERS];
        int                       cur_frame;

        void (*callback)(void *param, struct video_data *frame);
        void *param;
};
```

```c
struct video_output {
        struct video_output_info   info;

        pthread_t                  thread;
        pthread_mutex_t            data_mutex;
        bool                       stop;

        os_sem_t                   *update_semaphore;
        uint64_t                   frame_time;
        uint32_t                   skipped_frames;
        uint32_t                   total_frames;

        bool                       initialized;

        pthread_mutex_t            input_mutex;
        DARRAY(struct video_input) inputs;

        size_t                     available_frames;
        size_t                     first_added;
        size_t                     last_added;
        struct cached_frame_info   cache[MAX_CACHE_SIZE];
};

typedef struct video_output video_t;
```

```
struct obs_source {
	struct obs_context_data         context;
	struct obs_source_info          info;
	struct obs_weak_source          *control;

	/* general exposed flags that can be set for the source */
	uint32_t                        flags;
	uint32_t                        default_flags;

	/* indicates ownership of the info.id buffer */
	bool                            owns_info_id;

	/* signals to call the source update in the video thread */
	bool                            defer_update;

	/* ensures show/hide are only called once */
	volatile long                   show_refs;

	/* ensures activate/deactivate are only called once */
	volatile long                   activate_refs;

	/* used to indicate that the source has been removed and all
	 * references to it should be released (not exactly how I would prefer
	 * to handle things but it's the best option) */
	bool                            removed;

	bool                            active;
	bool                            showing;

	/* used to temporarily disable sources if needed */
	bool                            enabled;

	/* timing (if video is present, is based upon video) */
	volatile bool                   timing_set;
	volatile uint64_t               timing_adjust;
	uint64_t                        next_audio_ts_min;
	uint64_t                        last_frame_ts;
	uint64_t                        last_sys_timestamp;
	bool                            async_rendered;

	/* audio */
	bool                            audio_failed;
	bool                            muted;
	struct resample_info            sample_info;
	audio_resampler_t               *resampler;
	audio_line_t                    *audio_line;
	pthread_mutex_t                 audio_mutex;
	struct obs_audio_data           audio_data;
	size_t                          audio_storage_size;
	float                           base_volume;
	float                           user_volume;
	float                           present_volume;
	int64_t                         sync_offset;

	/* async video data */
	gs_texture_t                    *async_texture;
	gs_texrender_t                  *async_convert_texrender;
	struct obs_source_frame         *cur_async_frame;
	bool                            async_gpu_conversion;
	enum video_format               async_format;
	enum video_format               async_cache_format;
	enum gs_color_format            async_texture_format;
	float                           async_color_matrix[16];
	bool                            async_full_range;
	float                           async_color_range_min[3];
	float                           async_color_range_max[3];
	int                             async_plane_offset[2];
	bool                            async_flip;
	bool                            async_active;
	DARRAY(struct async_frame)      async_cache;
	DARRAY(struct obs_source_frame*)async_frames;
	pthread_mutex_t                 async_mutex;
	uint32_t                        async_width;
	uint32_t                        async_height;
	uint32_t                        async_cache_width;
	uint32_t                        async_cache_height;
	uint32_t                        async_convert_width;
	uint32_t                        async_convert_height;

	/* filters */
	struct obs_source               *filter_parent;
	struct obs_source               *filter_target;
	DARRAY(struct obs_source*)      filters;
	pthread_mutex_t                 filter_mutex;
	gs_texrender_t                  *filter_texrender;
	enum obs_allow_direct_render    allow_direct;
	bool                            rendering_filter;

	/* sources specific hotkeys */
	obs_hotkey_pair_id              mute_unmute_key;
	obs_hotkey_id                   push_to_mute_key;
	obs_hotkey_id                   push_to_talk_key;
	bool                            push_to_mute_enabled : 1;
	bool                            push_to_mute_pressed : 1;
	bool                            push_to_talk_enabled : 1;
	bool                            push_to_talk_pressed : 1;
	uint64_t                        push_to_mute_delay;
	uint64_t                        push_to_mute_stop_time;
	uint64_t                        push_to_talk_delay;
	uint64_t                        push_to_talk_stop_time;
};

```

```
EXPORT bool video_output_connect(video_t *video,
                const struct video_scale_info *conversion,
                void (*callback)(void *param, struct video_data *frame),
                void *param);
```

在obs-encoder.c中:

```c
void obs_encoder_start(obs_encoder_t *encoder,
		void (*new_packet)(void *param, struct encoder_packet *packet),
		void *param)
{
	struct encoder_callback cb = {false, new_packet, param};
	bool first   = false;

	if (!encoder || !new_packet || !encoder->context.data) return;

	pthread_mutex_lock(&encoder->callbacks_mutex);

	first = (encoder->callbacks.num == 0);

	size_t idx = get_callback_idx(encoder, new_packet, param);
	if (idx == DARRAY_INVALID)
		da_push_back(encoder->callbacks, &cb);

	pthread_mutex_unlock(&encoder->callbacks_mutex);

	if (first) {
		encoder->cur_pts = 0;
		add_connection(encoder);
	}
}

static void add_connection(struct obs_encoder *encoder)
{
        if (encoder->info.type == OBS_ENCODER_AUDIO) {
                struct audio_convert_info audio_info = {0};
                get_audio_info(encoder, &audio_info);

                audio_output_connect(encoder->media, encoder->mixer_idx,
                                &audio_info, receive_audio, encoder);
        } else {
                struct video_scale_info info = {0};
                get_video_info(encoder, &info);

                video_output_connect(encoder->media, &info, receive_video,
                        encoder);
        }

        encoder->active = true;
}
```

###基本流程

```c
bool obs_output_begin_data_capture(obs_output_t *output, uint32_t flags)
{
	bool encoded, has_video, has_audio, has_service;

	if (!output) return false;
	if (output->active) return false;

	output->total_frames   = 0;

	convert_flags(output, flags, &encoded, &has_video, &has_audio,
			&has_service);

	if (!can_begin_data_capture(output, encoded, has_video, has_audio,
				has_service))
		return false;

	hook_data_capture(output, encoded, has_video, has_audio);

	if (has_service)
		obs_service_activate(output->service);

	output->active = true;

	if (output->reconnecting) {
		signal_reconnect_success(output);
		output->reconnecting = false;
	} else {
		signal_start(output);
	}

	return true;
}
```

```c
static void hook_data_capture(struct obs_output *output, bool encoded,
		bool has_video, bool has_audio)
{
	encoded_callback_t encoded_callback;

	if (encoded) {
		output->received_audio   = false;
		output->received_video   = false;
		output->highest_audio_ts = 0;
		output->highest_video_ts = 0;
		output->video_offset     = 0;

		for (size_t i = 0; i < MAX_AUDIO_MIXES; i++)
			output->audio_offsets[0] = 0;

		free_packets(output);

		encoded_callback = (has_video && has_audio) ?
			interleave_packets : default_encoded_callback;

		if (has_video)
			obs_encoder_start(output->video_encoder,
					encoded_callback, output);
		if (has_audio)
			start_audio_encoders(output, encoded_callback);
	} else {
		if (has_video)
			video_output_connect(output->video,
					get_video_conversion(output),
					default_raw_video_callback, output);
		if (has_audio)
			audio_output_connect(output->audio, output->mixer_idx,
					get_audio_conversion(output),
					default_raw_audio_callback, output);
	}
}

static inline void do_encode(struct obs_encoder *encoder,
		struct encoder_frame *frame)
{
	struct encoder_packet pkt = {0};
	bool received = false;
	bool success;

	pkt.timebase_num = encoder->timebase_num;
	pkt.timebase_den = encoder->timebase_den;
	pkt.encoder = encoder;

	success = encoder->info.encode(encoder->context.data, frame, &pkt,
			&received);
	if (!success) {
		full_stop(encoder);
		blog(LOG_ERROR, "Error encoding with encoder '%s'",
				encoder->context.name);
		return;
	}

	if (received) {
		/* we use system time here to ensure sync with other encoders,
		 * you do not want to use relative timestamps here */
		pkt.dts_usec = encoder->start_ts / 1000 + packet_dts_usec(&pkt);

		pthread_mutex_lock(&encoder->callbacks_mutex);

		for (size_t i = 0; i < encoder->callbacks.num; i++) {
			struct encoder_callback *cb;
			cb = encoder->callbacks.array+i;
			send_packet(encoder, cb, &pkt);
		}

		pthread_mutex_unlock(&encoder->callbacks_mutex);
	}
}
```

最重要的全局变量:

```
struct obs_core *obs = NULL;

/* internal initialization */
bool obs_source_init(struct obs_source *source,
		const struct obs_source_info *info)
{
	pthread_mutexattr_t attr;

	source->user_volume = 1.0f;
	source->present_volume = 1.0f;
	source->base_volume = 0.0f;
	source->sync_offset = 0;
	pthread_mutex_init_value(&source->filter_mutex);
	pthread_mutex_init_value(&source->async_mutex);
	pthread_mutex_init_value(&source->audio_mutex);

	if (pthread_mutexattr_init(&attr) != 0)
		return false;
	if (pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE) != 0)
		return false;
	if (pthread_mutex_init(&source->filter_mutex, &attr) != 0)
		return false;
	if (pthread_mutex_init(&source->audio_mutex, NULL) != 0)
		return false;
	if (pthread_mutex_init(&source->async_mutex, NULL) != 0)
		return false;

	if (info && info->output_flags & OBS_SOURCE_AUDIO) {
		source->audio_line = audio_output_create_line(obs->audio.audio,
				source->context.name, 0xF);
		if (!source->audio_line) {
			blog(LOG_ERROR, "Failed to create audio line for "
			                "source '%s'", source->context.name);
			return false;
		}
	}

	source->control = bzalloc(sizeof(obs_weak_source_t));
	source->control->source = source;

	obs_context_data_insert(&source->context,
			&obs->data.sources_mutex,
			&obs->data.first_source);
	return true;
}

obs_source_t *obs_source_create(enum obs_source_type type, const char *id,
		const char *name, obs_data_t *settings, obs_data_t *hotkey_data)
{
	struct obs_source *source = bzalloc(sizeof(struct obs_source));

	const struct obs_source_info *info = get_source_info(type, id);
	if (!info) {
		blog(LOG_ERROR, "Source ID '%s' not found", id);

		source->info.id      = bstrdup(id);
		source->info.type    = type;
		source->owns_info_id = true;
	} else {
		source->info = *info;
	}

	source->mute_unmute_key  = OBS_INVALID_HOTKEY_PAIR_ID;
	source->push_to_mute_key = OBS_INVALID_HOTKEY_ID;
	source->push_to_talk_key = OBS_INVALID_HOTKEY_ID;

	if (!obs_source_init_context(source, settings, name, hotkey_data))
		goto fail;

	if (info && info->get_defaults)
		info->get_defaults(source->context.settings);

	if (!obs_source_init(source, info))
		goto fail;

	obs_source_init_audio_hotkeys(source);

	/* allow the source to be created even if creation fails so that the
	 * user's data doesn't become lost */
	if (info)
		source->context.data = info->create(source->context.settings,
				source);
	if (!source->context.data)
		blog(LOG_ERROR, "Failed to create source '%s'!", name);

	blog(LOG_INFO, "source '%s' (%s) created", name, id);
	obs_source_dosignal(source, "source_create", NULL);

	source->flags = source->default_flags;
	source->enabled = true;

	if (info && info->type == OBS_SOURCE_TYPE_TRANSITION)
		os_atomic_inc_long(&obs->data.active_transitions);
	return source;

fail:
	blog(LOG_ERROR, "obs_source_create failed");
	obs_source_destroy(source);
	return NULL;
}
```


```c
static uint64_t tick_sources(uint64_t cur_time, uint64_t last_time)
{
	struct obs_core_data *data = &obs->data;
	struct obs_view      *view = &data->main_view;
	struct obs_source    *source;
	uint64_t             delta_time;
	float                seconds;

	if (!last_time)
		last_time = cur_time -
			video_output_get_frame_time(obs->video.video);

	delta_time = cur_time - last_time;
	seconds = (float)((double)delta_time / 1000000000.0);

	pthread_mutex_lock(&data->sources_mutex);

	/* call the tick function of each source */
	source = data->first_source;
	while (source) {
		obs_source_video_tick(source, seconds);
		source = (struct obs_source*)source->context.next;
	}

	/* calculate source volumes */
	pthread_mutex_lock(&view->channels_mutex);

	source = data->first_source;
	while (source) {
		calculate_base_volume(data, view, source);
		source = (struct obs_source*)source->context.next;
	}

	pthread_mutex_unlock(&view->channels_mutex);

	pthread_mutex_unlock(&data->sources_mutex);

	return cur_time;
}
```

###Video数据流

rtmp\_stream\_start -> connect\_thread -> try\_connect -> init\_send -> obs\_output\_begin\_data\_capture -> hook\_data\_capture -> obs\_encoder\_start(interleave\_packets) -> add\_connection -> video\_output\_connect(receive\_video)

obs\_source\_create -> av\_capture\_info.create[av\_capture\_create]

captureOutput didOutputSampleBuffer fromConnection -> obs\_source\_output\_video -> cache\_video

```
obs_init_video -> pthread_create(obs_video_thread) -> tick_sources -> obs_source_video_tick -> get_closest_frame
                                                   -> render_displays
                                                   -> output_frame -> output_video_data
                                                   
video_thread -> video_output_cur_frame -> input->callback -> receive_video -> do_encode -> encoder->info.encode -> send_packet -> interleave_packets

obs_init_video -> video_output_open -> pthread_create(video_thread)
```

