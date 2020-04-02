General Info
===================
Audio playback and capture library. Choice of public domain or MIT-0. See license statements at the end of this file.
miniaudio - v0.10.2 - 2020-03-22

David Reid - davidreidsoftware@gmail.com

Website: https://miniaud.io
GitHub:  https://github.com/dr-soft/miniaudio

RELEASE NOTES - VERSION 0.10
============================
Version 0.10 includes major API changes and refactoring, mostly concerned with the data conversion system. Data conversion is performed internally to convert
audio data between the format requested when initializing the `ma_device` object and the format of the internal device used by the backend. The same applies
to the `ma_decoder` object. The previous design has several design flaws and missing features which necessitated a complete redesign.


Changes to Data Conversion
--------------------------
The previous data conversion system used callbacks to deliver input data for conversion. This design works well in some specific situations, but in other
situations it has some major readability and maintenance issues. The decision was made to replace this with a more iterative approach where you just pass in a
pointer to the input data directly rather than dealing with a callback.

The following are the data conversion APIs that have been removed and their replacements:

  - ma_format_converter -> ma_convert_pcm_frames_format()
  - ma_channel_router   -> ma_channel_converter
  - ma_src              -> ma_resampler
  - ma_pcm_converter    -> ma_data_converter

The previous conversion APIs accepted a callback in their configs. There are no longer any callbacks to deal with. Instead you just pass the data into the
`*_process_pcm_frames()` function as a pointer to a buffer.

The simplest aspect of data conversion is sample format conversion. To convert between two formats, just call `ma_convert_pcm_frames_format()`. Channel
conversion is also simple which you can do with `ma_channel_converter` via `ma_channel_converter_process_pcm_frames()`.

Resampling is more complicated because the number of output frames that are processed is different to the number of input frames that are consumed. When you
call `ma_resampler_process_pcm_frames()` you need to pass in the number of input frames available for processing and the number of output frames you want to
output. Upon returning they will receive the number of input frames that were consumed and the number of output frames that were generated.

The `ma_data_converter` API is a wrapper around format, channel and sample rate conversion and handles all of the data conversion you'll need which probably
makes it the best option if you need to do data conversion.

In addition to changes to the API design, a few other changes have been made to the data conversion pipeline:

  - The sinc resampler has been removed. This was completely broken and never actually worked properly.
  - The linear resampler now uses low-pass filtering to remove aliasing. The quality of the low-pass filter can be controlled via the resampler config with the
    `lpfOrder` option, which has a maximum value of MA_MAX_FILTER_ORDER.
  - Data conversion now supports s16 natively which runs through a fixed point pipeline. Previously everything needed to be converted to floating point before
    processing, whereas now both s16 and f32 are natively supported. Other formats still require conversion to either s16 or f32 prior to processing, however
    `ma_data_converter` will handle this for you.


Custom Memory Allocators
------------------------
miniaudio has always supported macro level customization for memory allocation via MA_MALLOC, MA_REALLOC and MA_FREE, however some scenarios require more
flexibility by allowing a user data pointer to be passed to the custom allocation routines. Support for this has been added to version 0.10 via the
`ma_allocation_callbacks` structure. Anything making use of heap allocations has been updated to accept this new structure.

The `ma_context_config` structure has been updated with a new member called `allocationCallbacks`. Leaving this set to it's defaults returned by
`ma_context_config_init()` will cause it to use MA_MALLOC, MA_REALLOC and MA_FREE. Likewise, The `ma_decoder_config` structure has been updated in the same
way, and leaving everything as-is after `ma_decoder_config_init()` will cause it to use the same defaults.

The following APIs have been updated to take a pointer to a `ma_allocation_callbacks` object. Setting this parameter to NULL will cause it to use defaults.
Otherwise they will use the relevant callback in the structure.

  - ma_malloc()
  - ma_realloc()
  - ma_free()
  - ma_aligned_malloc()
  - ma_aligned_free()
  - ma_rb_init() / ma_rb_init_ex()
  - ma_pcm_rb_init() / ma_pcm_rb_init_ex()

Note that you can continue to use MA_MALLOC, MA_REALLOC and MA_FREE as per normal. These will continue to be used by default if you do not specify custom
allocation callbacks.


Buffer and Period Configuration Changes
---------------------------------------
The way in which the size of the internal buffer and periods are specified in the device configuration have changed. In previous versions, the config variables
`bufferSizeInFrames` and `bufferSizeInMilliseconds` defined the size of the entire buffer, with the size of a period being the size of this variable divided by
the period count. This became confusing because people would expect the value of `bufferSizeInFrames` or `bufferSizeInMilliseconds` to independantly determine
latency, when in fact it was that value divided by the period count that determined it. These variables have been removed and replaced with new ones called
`periodSizeInFrames` and `periodSizeInMilliseconds`.

These new configuration variables work in the same way as their predecessors in that if one is set to 0, the other will be used, but the main difference is
that you now set these to you desired latency rather than the size of the entire buffer. The benefit of this is that it's much easier and less confusing to
configure latency.

The following unused APIs have been removed:

    ma_get_default_buffer_size_in_milliseconds()
    ma_get_default_buffer_size_in_frames()

The following macros have been removed:

    MA_BASE_BUFFER_SIZE_IN_MILLISECONDS_LOW_LATENCY
    MA_BASE_BUFFER_SIZE_IN_MILLISECONDS_CONSERVATIVE


Other API Changes
-----------------
Other less major API changes have also been made in version 0.10.

`ma_device_set_stop_callback()` has been removed. If you require a stop callback, you must now set it via the device config just like the data callback.

The `ma_sine_wave` API has been replaced with a more general API called `ma_waveform`. This supports generation of different types of waveforms, including
sine, square, triangle and sawtooth. Use `ma_waveform_init()` in place of `ma_sine_wave_init()` to initialize the waveform object. This takes a configuration
object called `ma_waveform_config` which defines the properties of the waveform. Use `ma_waveform_config_init()` to initialize a `ma_waveform_config` object.
Use `ma_waveform_read_pcm_frames()` in place of `ma_sine_wave_read_f32()` and `ma_sine_wave_read_f32_ex()`.

`ma_convert_frames()` and `ma_convert_frames_ex()` have been changed. Both of these functions now take a new parameter called `frameCountOut` which specifies
the size of the output buffer in PCM frames. This has been added for safety. In addition to this, the parameters for `ma_convert_frames_ex()` have changed to
take a pointer to a `ma_data_converter_config` object to specify the input and output formats to convert between. This was done to make it more flexible, to
prevent the parameter list getting too long, and to prevent API breakage whenever a new conversion property is added.

`ma_calculate_frame_count_after_src()` has been renamed to `ma_calculate_frame_count_after_resampling()` for consistency with the new `ma_resampler` API.


Filters
-------
The following filters have been added:

    |-------------|-------------------------------------------------------------------|
    | API         | Description                                                       |
    |-------------|-------------------------------------------------------------------|
    | ma_biquad   | Biquad filter (transposed direct form 2)                          |
    | ma_lpf1     | First order low-pass filter                                       |
    | ma_lpf2     | Second order low-pass filter                                      |
    | ma_lpf      | High order low-pass filter (Butterworth)                          |
    | ma_hpf1     | First order high-pass filter                                      |
    | ma_hpf2     | Second order high-pass filter                                     |
    | ma_hpf      | High order high-pass filter (Butterworth)                         |
    | ma_bpf2     | Second order band-pass filter                                     |
    | ma_bpf      | High order band-pass filter                                       |
    | ma_peak2    | Second order peaking filter                                       |
    | ma_notch2   | Second order notching filter                                      |
    | ma_loshelf2 | Second order low shelf filter                                     |
    | ma_hishelf2 | Second order high shelf filter                                    |
    |-------------|-------------------------------------------------------------------|

These filters all support 32-bit floating point and 16-bit signed integer formats natively. Other formats need to be converted beforehand.


Sine, Square, Triangle and Sawtooth Waveforms
---------------------------------------------
Previously miniaudio supported only sine wave generation. This has now been generalized to support sine, square, triangle and sawtooth waveforms. The old
`ma_sine_wave` API has been removed and replaced with the `ma_waveform` API. Use `ma_waveform_config_init()` to initialize a config object, and then pass it
into `ma_waveform_init()`. Then use `ma_waveform_read_pcm_frames()` to read PCM data.


Noise Generation
----------------
A noise generation API has been added. This is used via the `ma_noise` API. Currently white, pink and Brownian noise is supported. The `ma_noise` API is
similar to the waveform API. Use `ma_noise_config_init()` to initialize a config object, and then pass it into `ma_noise_init()` to initialize a `ma_noise`
object. Then use `ma_noise_read_pcm_frames()` to read PCM data.


Miscellaneous Changes
---------------------
Internal functions have all been made static where possible. If you get warnings about unused functions, please submit a bug report.

The `ma_device` structure is no longer defined as being aligned to MA_SIMD_ALIGNMENT. This resulted in a possible crash when allocating a `ma_device` object on
the heap, but not aligning it to MA_SIMD_ALIGNMENT. This crash would happen due to the compiler seeing the alignment specified on the structure and assuming it
was always aligned as such and thinking it was safe to emit alignment-dependant SIMD instructions. Since miniaudio's philosophy is for things to just work,
this has been removed from all structures.

Results codes have been overhauled. Unnecessary result codes have been removed, and some have been renumbered for organisation purposes. If you are are binding
maintainer you will need to update your result codes. Support has also been added for retrieving a human readable description of a given result code via the
`ma_result_description()` API.
*/


/*
Introduction
============
miniaudio is a single file library for audio playback and capture. To use it, do the following in one .c file:

    ```c
    #define MINIAUDIO_IMPLEMENTATION
    #include "miniaudio.h
    ```

You can #include miniaudio.h in other parts of the program just like any other header.

miniaudio uses the concept of a "device" as the abstraction for physical devices. The idea is that you choose a physical device to emit or capture audio from,
and then move data to/from the device when miniaudio tells you to. Data is delivered to and from devices asynchronously via a callback which you specify when
initializing the device.

When initializing the device you first need to configure it. The device configuration allows you to specify things like the format of the data delivered via
the callback, the size of the internal buffer and the ID of the device you want to emit or capture audio from.

Once you have the device configuration set up you can initialize the device. When initializing a device you need to allocate memory for the device object
beforehand. This gives the application complete control over how the memory is allocated. In the example below we initialize a playback device on the stack,
but you could allocate it on the heap if that suits your situation better.

    ```c
    void data_callback(ma_device* pDevice, void* pOutput, const void* pInput, ma_uint32 frameCount)
    {
        // In playback mode copy data to pOutput. In capture mode read data from pInput. In full-duplex mode, both pOutput and pInput will be valid and you can
        // move data from pInput into pOutput. Never process more than frameCount frames.
    }

    ...

    ma_device_config config = ma_device_config_init(ma_device_type_playback);
    config.playback.format   = MY_FORMAT;
    config.playback.channels = MY_CHANNEL_COUNT;
    config.sampleRate        = MY_SAMPLE_RATE;
    config.dataCallback      = data_callback;
    config.pUserData         = pMyCustomData;   // Can be accessed from the device object (device.pUserData).

    ma_device device;
    if (ma_device_init(NULL, &config, &device) != MA_SUCCESS) {
        ... An error occurred ...
    }

    ma_device_start(&device);     // The device is sleeping by default so you'll need to start it manually.

    ...

    ma_device_uninit(&device);    // This will stop the device so no need to do that manually.
    ```

In the example above, `data_callback()` is where audio data is written and read from the device. The idea is in playback mode you cause sound to be emitted
from the speakers by writing audio data to the output buffer (`pOutput` in the example). In capture mode you read data from the input buffer (`pInput`) to
extract sound captured by the microphone. The `frameCount` parameter tells you how many frames can be written to the output buffer and read from the input
buffer. A "frame" is one sample for each channel. For example, in a stereo stream (2 channels), one frame is 2 samples: one for the left, one for the right.
The channel count is defined by the device config. The size in bytes of an individual sample is defined by the sample format which is also specified in the
device config. Multi-channel audio data is always interleaved, which means the samples for each frame are stored next to each other in memory. For example, in
a stereo stream the first pair of samples will be the left and right samples for the first frame, the second pair of samples will be the left and right samples
for the second frame, etc.

The configuration of the device is defined by the `ma_device_config` structure. The config object is always initialized with `ma_device_config_init()`. It's
important to always initialize the config with this function as it initializes it with logical defaults and ensures your program doesn't break when new members
are added to the `ma_device_config` structure. The example above uses a fairly simple and standard device configuration. The call to `ma_device_config_init()`
takes a single parameter, which is whether or not the device is a playback, capture, duplex or loopback device (loopback devices are not supported on all
backends). The `config.playback.format` member sets the sample format which can be one of the following (all formats are native-endian):

    |---------------|----------------------------------------|---------------------------|
    | Symbol        | Description                            | Range                     |
    |---------------|----------------------------------------|---------------------------|
    | ma_format_f32 | 32-bit floating point                  | [-1, 1]                   |
    | ma_format_s16 | 16-bit signed integer                  | [-32768, 32767]           |
    | ma_format_s24 | 24-bit signed integer (tightly packed) | [-8388608, 8388607]       |
    | ma_format_s32 | 32-bit signed integer                  | [-2147483648, 2147483647] |
    | ma_format_u8  | 8-bit unsigned integer                 | [0, 255]                  |
    |---------------|----------------------------------------|---------------------------|

The `config.playback.channels` member sets the number of channels to use with the device. The channel count cannot exceed MA_MAX_CHANNELS. The
`config.sampleRate` member sets the sample rate (which must be the same for both playback and capture in full-duplex configurations). This is usually set to
44100 or 48000, but can be set to anything. It's recommended to keep this between 8000 and 384000, however.

Note that leaving the format, channel count and/or sample rate at their default values will result in the internal device's native configuration being used
which is useful if you want to avoid the overhead of miniaudio's automatic data conversion.

In addition to the sample format, channel count and sample rate, the data callback and user data pointer are also set via the config. The user data pointer is
not passed into the callback as a parameter, but is instead set to the `pUserData` member of `ma_device` which you can access directly since all miniaudio
structures are transparent.

Initializing the device is done with `ma_device_init()`. This will return a result code telling you what went wrong, if anything. On success it will return
`MA_SUCCESS`. After initialization is complete the device will be in a stopped state. To start it, use `ma_device_start()`. Uninitializing the device will stop
it, which is what the example above does, but you can also stop the device with `ma_device_stop()`. To resume the device simply call `ma_device_start()` again.
Note that it's important to never stop or start the device from inside the callback. This will result in a deadlock. Instead you set a variable or signal an
event indicating that the device needs to stop and handle it in a different thread. The following APIs must never be called inside the callback:

    ma_device_init()
    ma_device_init_ex()
    ma_device_uninit()
    ma_device_start()
    ma_device_stop()

You must never try uninitializing and reinitializing a device inside the callback. You must also never try to stop and start it from inside the callback. There
are a few other things you shouldn't do in the callback depending on your requirements, however this isn't so much a thread-safety thing, but rather a real-
time processing thing which is beyond the scope of this introduction.

The example above demonstrates the initialization of a playback device, but it works exactly the same for capture. All you need to do is change the device type
from `ma_device_type_playback` to `ma_device_type_capture` when setting up the config, like so:

    ```c
    ma_device_config config = ma_device_config_init(ma_device_type_capture);
    config.capture.format   = MY_FORMAT;
    config.capture.channels = MY_CHANNEL_COUNT;
    ```

In the data callback you just read from the input buffer (`pInput` in the example above) and leave the output buffer alone (it will be set to NULL when the
device type is set to `ma_device_type_capture`).

These are the available device types and how you should handle the buffers in the callback:

    |-------------------------|--------------------------------------------------------|
    | Device Type             | Callback Behavior                                      |
    |-------------------------|--------------------------------------------------------|
    | ma_device_type_playback | Write to output buffer, leave input buffer untouched.  |
    | ma_device_type_capture  | Read from input buffer, leave output buffer untouched. |
    | ma_device_type_duplex   | Read from input buffer, write to output buffer.        |
    | ma_device_type_loopback | Read from input buffer, leave output buffer untouched. |
    |-------------------------|--------------------------------------------------------|

You will notice in the example above that the sample format and channel count is specified separately for playback and capture. This is to support different
data formats between the playback and capture devices in a full-duplex system. An example may be that you want to capture audio data as a monaural stream (one
channel), but output sound to a stereo speaker system. Note that if you use different formats between playback and capture in a full-duplex configuration you
will need to convert the data yourself. There are functions available to help you do this which will be explained later.

The example above did not specify a physical device to connect to which means it will use the operating system's default device. If you have multiple physical
devices connected and you want to use a specific one you will need to specify the device ID in the configuration, like so:

    ```
    config.playback.pDeviceID = pMyPlaybackDeviceID;    // Only if requesting a playback or duplex device.
    config.capture.pDeviceID = pMyCaptureDeviceID;      // Only if requesting a capture, duplex or loopback device.
    ```

To retrieve the device ID you will need to perform device enumeration, however this requires the use of a new concept called the "context". Conceptually
speaking the context sits above the device. There is one context to many devices. The purpose of the context is to represent the backend at a more global level
and to perform operations outside the scope of an individual device. Mainly it is used for performing run-time linking against backend libraries, initializing
backends and enumerating devices. The example below shows how to enumerate devices.

    ```c
    ma_context context;
    if (ma_context_init(NULL, 0, NULL, &context) != MA_SUCCESS) {
        // Error.
    }

    ma_device_info* pPlaybackDeviceInfos;
    ma_uint32 playbackDeviceCount;
    ma_device_info* pCaptureDeviceInfos;
    ma_uint32 captureDeviceCount;
    if (ma_context_get_devices(&context, &pPlaybackDeviceInfos, &playbackDeviceCount, &pCaptureDeviceInfos, &captureDeviceCount) != MA_SUCCESS) {
        // Error.
    }

    // Loop over each device info and do something with it. Here we just print the name with their index. You may want to give the user the
    // opportunity to choose which device they'd prefer.
    for (ma_uint32 iDevice = 0; iDevice < playbackDeviceCount; iDevice += 1) {
        printf("%d - %s\n", iDevice, pPlaybackDeviceInfos[iDevice].name);
    }

    ma_device_config config = ma_device_config_init(ma_device_type_playback);
    config.playback.pDeviceID = &pPlaybackDeviceInfos[chosenPlaybackDeviceIndex].id;
    config.playback.format    = MY_FORMAT;
    config.playback.channels  = MY_CHANNEL_COUNT;
    config.sampleRate         = MY_SAMPLE_RATE;
    config.dataCallback       = data_callback;
    config.pUserData          = pMyCustomData;

    ma_device device;
    if (ma_device_init(&context, &config, &device) != MA_SUCCESS) {
        // Error
    }

    ...

    ma_device_uninit(&device);
    ma_context_uninit(&context);
    ```

The first thing we do in this example is initialize a `ma_context` object with `ma_context_init()`. The first parameter is a pointer to a list of `ma_backend`
values which are used to override the default backend priorities. When this is NULL, as in this example, miniaudio's default priorities are used. The second
parameter is the number of backends listed in the array pointed to by the first parameter. The third parameter is a pointer to a `ma_context_config` object
which can be NULL, in which case defaults are used. The context configuration is used for setting the logging callback, custom memory allocation callbacks,
user-defined data and some backend-specific configurations.

Once the context has been initialized you can enumerate devices. In the example above we use the simpler `ma_context_get_devices()`, however you can also use a
callback for handling devices by using `ma_context_enumerate_devices()`. When using `ma_context_get_devices()` you provide a pointer to a pointer that will,
upon output, be set to a pointer to a buffer containing a list of `ma_device_info` structures. You also provide a pointer to an unsigned integer that will
receive the number of items in the returned buffer. Do not free the returned buffers as their memory is managed internally by miniaudio.

The `ma_device_info` structure contains an `id` member which is the ID you pass to the device config. It also contains the name of the device which is useful
for presenting a list of devices to the user via the UI.

When creating your own context you will want to pass it to `ma_device_init()` when initializing the device. Passing in NULL, like we do in the first example,
will result in miniaudio creating the context for you, which you don't want to do since you've already created a context. Note that internally the context is
only tracked by it's pointer which means you must not change the location of the `ma_context` object. If this is an issue, consider using `malloc()` to
allocate memory for the context.



Building
========
miniaudio should work cleanly out of the box without the need to download or install any dependencies. See below for platform-specific details.


Windows
-------
The Windows build should compile cleanly on all popular compilers without the need to configure any include paths nor link to any libraries.

macOS and iOS
-------------
The macOS build should compile cleanly without the need to download any dependencies nor link to any libraries or frameworks. The iOS build needs to be
compiled as Objective-C (sorry) and will need to link the relevant frameworks but should Just Work with Xcode. Compiling through the command line requires
linking to -lpthread and -lm.

Linux
-----
The Linux build only requires linking to -ldl, -lpthread and -lm. You do not need any development packages.

BSD
---
The BSD build only requires linking to -lpthread and -lm. NetBSD uses audio(4), OpenBSD uses sndio and FreeBSD uses OSS.

Android
-------
AAudio is the highest priority backend on Android. This should work out of the box without needing any kind of compiler configuration. Support for AAudio
starts with Android 8 which means older versions will fall back to OpenSL|ES which requires API level 16+.

Emscripten
----------
The Emscripten build emits Web Audio JavaScript directly and should Just Work without any configuration. You cannot use -std=c* compiler flags, nor -ansi.


Build Options
-------------
#define these options before including miniaudio.h.

#define MA_NO_WASAPI
  Disables the WASAPI backend.

#define MA_NO_DSOUND
  Disables the DirectSound backend.

#define MA_NO_WINMM
  Disables the WinMM backend.

#define MA_NO_ALSA
  Disables the ALSA backend.

#define MA_NO_PULSEAUDIO
  Disables the PulseAudio backend.

#define MA_NO_JACK
  Disables the JACK backend.

#define MA_NO_COREAUDIO
  Disables the Core Audio backend.

#define MA_NO_SNDIO
  Disables the sndio backend.

#define MA_NO_AUDIO4
  Disables the audio(4) backend.

#define MA_NO_OSS
  Disables the OSS backend.

#define MA_NO_AAUDIO
  Disables the AAudio backend.

#define MA_NO_OPENSL
  Disables the OpenSL|ES backend.

#define MA_NO_WEBAUDIO
  Disables the Web Audio backend.

#define MA_NO_NULL
  Disables the null backend.

#define MA_NO_DECODING
  Disables the decoding APIs.

#define MA_NO_DEVICE_IO
  Disables playback and recording. This will disable ma_context and ma_device APIs. This is useful if you only want to use miniaudio's data conversion and/or
  decoding APIs. 

#define MA_NO_STDIO
  Disables file IO APIs.

#define MA_NO_SSE2
  Disables SSE2 optimizations.

#define MA_NO_AVX2
  Disables AVX2 optimizations.

#define MA_NO_AVX512
  Disables AVX-512 optimizations.

#define MA_NO_NEON
  Disables NEON optimizations.

#define MA_LOG_LEVEL <Level>
  Sets the logging level. Set level to one of the following:
    MA_LOG_LEVEL_VERBOSE
    MA_LOG_LEVEL_INFO
    MA_LOG_LEVEL_WARNING
    MA_LOG_LEVEL_ERROR

#define MA_DEBUG_OUTPUT
  Enable printf() debug output.

#define MA_COINIT_VALUE
  Windows only. The value to pass to internal calls to CoInitializeEx(). Defaults to COINIT_MULTITHREADED.

#define MA_API
  Controls how public APIs should be decorated. Defaults to `extern`.

#define MA_DLL
  If set, configures MA_API to either import or export APIs depending on whether or not the implementation is being defined. If defining the implementation,
  MA_API will be configured to export. Otherwise it will be configured to import. This has no effect if MA_API is defined externally.
    



Definitions
===========
This section defines common terms used throughout miniaudio. Unfortunately there is often ambiguity in the use of terms throughout the audio space, so this
section is intended to clarify how miniaudio uses each term.

Sample
------
A sample is a single unit of audio data. If the sample format is f32, then one sample is one 32-bit floating point number.

Frame / PCM Frame
-----------------
A frame is a group of samples equal to the number of channels. For a stereo stream a frame is 2 samples, a mono frame is 1 sample, a 5.1 surround sound frame
is 6 samples, etc. The terms "frame" and "PCM frame" are the same thing in miniaudio. Note that this is different to a compressed frame. If ever miniaudio
needs to refer to a compressed frame, such as a FLAC frame, it will always clarify what it's referring to with something like "FLAC frame".

Channel
-------
A stream of monaural audio that is emitted from an individual speaker in a speaker system, or received from an individual microphone in a microphone system. A
stereo stream has two channels (a left channel, and a right channel), a 5.1 surround sound system has 6 channels, etc. Some audio systems refer to a channel as
a complex audio stream that's mixed with other channels to produce the final mix - this is completely different to miniaudio's use of the term "channel" and
should not be confused.

Sample Rate
-----------
The sample rate in miniaudio is always expressed in Hz, such as 44100, 48000, etc. It's the number of PCM frames that are processed per second.

Formats
-------
Throughout miniaudio you will see references to different sample formats:

    |---------------|----------------------------------------|---------------------------|
    | Symbol        | Description                            | Range                     |
    |---------------|----------------------------------------|---------------------------|
    | ma_format_f32 | 32-bit floating point                  | [-1, 1]                   |
    | ma_format_s16 | 16-bit signed integer                  | [-32768, 32767]           |
    | ma_format_s24 | 24-bit signed integer (tightly packed) | [-8388608, 8388607]       |
    | ma_format_s32 | 32-bit signed integer                  | [-2147483648, 2147483647] |
    | ma_format_u8  | 8-bit unsigned integer                 | [0, 255]                  |
    |---------------|----------------------------------------|---------------------------|

All formats are native-endian.



Decoding
========
The `ma_decoder` API is used for reading audio files. To enable a decoder you must #include the header of the relevant backend library before the
implementation of miniaudio. You can find copies of these in the "extras" folder in the miniaudio repository (https://github.com/dr-soft/miniaudio).

The table below are the supported decoding backends:

    |--------|-----------------|
    | Type   | Backend Library |
    |--------|-----------------|
    | WAV    | dr_wav.h        |
    | FLAC   | dr_flac.h       |
    | MP3    | dr_mp3.h        |
    | Vorbis | stb_vorbis.c    |
    |--------|-----------------|

The code below is an example of how to enable decoding backends:

    ```c
    #include "dr_flac.h"    // Enables FLAC decoding.
    #include "dr_mp3.h"     // Enables MP3 decoding.
    #include "dr_wav.h"     // Enables WAV decoding.

    #define MINIAUDIO_IMPLEMENTATION
    #include "miniaudio.h"
    ```

A decoder can be initialized from a file with `ma_decoder_init_file()`, a block of memory with `ma_decoder_init_memory()`, or from data delivered via callbacks
with `ma_decoder_init()`. Here is an example for loading a decoder from a file:

    ```c
    ma_decoder decoder;
    ma_result result = ma_decoder_init_file("MySong.mp3", NULL, &decoder);
    if (result != MA_SUCCESS) {
        return false;   // An error occurred.
    }

    ...

    ma_decoder_uninit(&decoder);
    ```

When initializing a decoder, you can optionally pass in a pointer to a ma_decoder_config object (the NULL argument in the example above) which allows you to
configure the output format, channel count, sample rate and channel map:

    ```c
    ma_decoder_config config = ma_decoder_config_init(ma_format_f32, 2, 48000);
    ```

When passing in NULL for decoder config in `ma_decoder_init*()`, the output format will be the same as that defined by the decoding backend.

Data is read from the decoder as PCM frames:

    ```c
    ma_uint64 framesRead = ma_decoder_read_pcm_frames(pDecoder, pFrames, framesToRead);
    ```

You can also seek to a specific frame like so:

    ```c
    ma_result result = ma_decoder_seek_to_pcm_frame(pDecoder, targetFrame);
    if (result != MA_SUCCESS) {
        return false;   // An error occurred.
    }
    ```

When loading a decoder, miniaudio uses a trial and error technique to find the appropriate decoding backend. This can be unnecessarily inefficient if the type
is already known. In this case you can use the `_wav`, `_mp3`, etc. varients of the aforementioned initialization APIs:

    ```c
    ma_decoder_init_wav()
    ma_decoder_init_mp3()
    ma_decoder_init_memory_wav()
    ma_decoder_init_memory_mp3()
    ma_decoder_init_file_wav()
    ma_decoder_init_file_mp3()
    etc.
    ```

The `ma_decoder_init_file()` API will try using the file extension to determine which decoding backend to prefer.



Encoding
========
The `ma_encoding` API is used for writing audio files. To enable an encoder you must #include the header of the relevant backend library before the
implementation of miniaudio. You can find copies of these in the "extras" folder in the miniaudio repository (https://github.com/dr-soft/miniaudio).

The table below are the supported encoding backends:

    |--------|-----------------|
    | Type   | Backend Library |
    |--------|-----------------|
    | WAV    | dr_wav.h        |
    |--------|-----------------|

The code below is an example of how to enable encoding backends:

    ```c
    #include "dr_wav.h"     // Enables WAV decoding.

    #define MINIAUDIO_IMPLEMENTATION
    #include "miniaudio.h"
    ```

An encoder can be initialized to write to a file with `ma_encoder_init_file()` or from data delivered via callbacks with `ma_encoder_init()`. Below is an
example for initializing an encoder to output to a file.

    ```c
    ma_encoder_config config = ma_encoder_config_init(ma_resource_format_wav, FORMAT, CHANNELS, SAMPLE_RATE);
    ma_encoder encoder;
    ma_result result = ma_encoder_init_file("my_file.wav", &config, &encoder);
    if (result != MA_SUCCESS) {
        // Error
    }

    ...

    ma_encoder_uninit(&encoder);
    ```

When initializing an encoder you must specify a config which is initialized with `ma_encoder_config_init()`. Here you must specify the file type, the output
sample format, output channel count and output sample rate. The following file types are supported:

    |------------------------|-------------|
    | Enum                   | Description |
    |------------------------|-------------|
    | ma_resource_format_wav | WAV         |
    |------------------------|-------------|

If the format, channel count or sample rate is not supported by the output file type an error will be returned. The encoder will not perform data conversion so
you will need to convert it before outputting any audio data. To output audio data, use `ma_encoder_write_pcm_frames()`, like in the example below:

    ```c
    framesWritten = ma_encoder_write_pcm_frames(&encoder, pPCMFramesToWrite, framesToWrite);
    ```

Encoders must be uninitialized with `ma_encoder_uninit()`.



Sample Format Conversion
========================
Conversion between sample formats is achieved with the `ma_pcm_*_to_*()`, `ma_pcm_convert()` and `ma_convert_pcm_frames_format()` APIs. Use `ma_pcm_*_to_*()`
to convert between two specific formats. Use `ma_pcm_convert()` to convert based on a `ma_format` variable. Use `ma_convert_pcm_frames_format()` to convert
PCM frames where you want to specify the frame count and channel count as a variable instead of the total sample count.

Dithering
---------
Dithering can be set using the ditherMode parameter.

The different dithering modes include the following, in order of efficiency:

    |-----------|--------------------------|
    | Type      | Enum Token               |
    |-----------|--------------------------|
    | None      | ma_dither_mode_none      |
    | Rectangle | ma_dither_mode_rectangle |
    | Triangle  | ma_dither_mode_triangle  |
    |-----------|--------------------------|

Note that even if the dither mode is set to something other than `ma_dither_mode_none`, it will be ignored for conversions where dithering is not needed.
Dithering is available for the following conversions:

    s16 -> u8
    s24 -> u8
    s32 -> u8
    f32 -> u8
    s24 -> s16
    s32 -> s16
    f32 -> s16

Note that it is not an error to pass something other than ma_dither_mode_none for conversions where dither is not used. It will just be ignored.



Channel Conversion
==================
Channel conversion is used for channel rearrangement and conversion from one channel count to another. The `ma_channel_converter` API is used for channel
conversion. Below is an example of initializing a simple channel converter which converts from mono to stereo.

    ```c
    ma_channel_converter_config config = ma_channel_converter_config_init(ma_format, 1, NULL, 2, NULL, ma_channel_mix_mode_default, NULL);
    result = ma_channel_converter_init(&config, &converter);
    if (result != MA_SUCCESS) {
        // Error.
    }
    ```

To perform the conversion simply call `ma_channel_converter_process_pcm_frames()` like so:

    ```c
    ma_result result = ma_channel_converter_process_pcm_frames(&converter, pFramesOut, pFramesIn, frameCount);
    if (result != MA_SUCCESS) {
        // Error.
    }
    ```

It is up to the caller to ensure the output buffer is large enough to accomodate the new PCM frames.

The only formats supported are `ma_format_s16` and `ma_format_f32`. If you need another format you need to convert your data manually which you can do with
`ma_pcm_convert()`, etc.

Input and output PCM frames are always interleaved. Deinterleaved layouts are not supported.


Channel Mapping
---------------
In addition to converting from one channel count to another, like the example above, The channel converter can also be used to rearrange channels. When
initializing the channel converter, you can optionally pass in channel maps for both the input and output frames. If the channel counts are the same, and each
channel map contains the same channel positions with the exception that they're in a different order, a simple shuffling of the channels will be performed. If,
however, there is not a 1:1 mapping of channel positions, or the channel counts differ, the input channels will be mixed based on a mixing mode which is
specified when initializing the `ma_channel_converter_config` object.

When converting from mono to multi-channel, the mono channel is simply copied to each output channel. When going the other way around, the audio of each output
channel is simply averaged and copied to the mono channel.

In more complicated cases blending is used. The `ma_channel_mix_mode_simple` mode will drop excess channels and silence extra channels. For example, converting
from 4 to 2 channels, the 3rd and 4th channels will be dropped, whereas converting from 2 to 4 channels will put silence into the 3rd and 4th channels.

The `ma_channel_mix_mode_rectangle` mode uses spacial locality based on a rectangle to compute a simple distribution between input and output. Imagine sitting
in the middle of a room, with speakers on the walls representing channel positions. The MA_CHANNEL_FRONT_LEFT position can be thought of as being in the corner
of the front and left walls.

Finally, the `ma_channel_mix_mode_custom_weights` mode can be used to use custom user-defined weights. Custom weights can be passed in as the last parameter of
`ma_channel_converter_config_init()`.

Predefined channel maps can be retrieved with `ma_get_standard_channel_map()`. This takes a `ma_standard_channel_map` enum as it's first parameter, which can
be one of the following:

    |-----------------------------------|-----------------------------------------------------------|
    | Name                              | Description                                               |
    |-----------------------------------|-----------------------------------------------------------|
    | ma_standard_channel_map_default   | Default channel map used by miniaudio. See below.         |
    | ma_standard_channel_map_microsoft | Channel map used by Microsoft's bitfield channel maps.    |
    | ma_standard_channel_map_alsa      | Default ALSA channel map.                                 |
    | ma_standard_channel_map_rfc3551   | RFC 3551. Based on AIFF.                                  |
    | ma_standard_channel_map_flac      | FLAC channel map.                                         |
    | ma_standard_channel_map_vorbis    | Vorbis channel map.                                       |
    | ma_standard_channel_map_sound4    | FreeBSD's sound(4).                                       |
    | ma_standard_channel_map_sndio     | sndio channel map. www.sndio.org/tips.html                |
    | ma_standard_channel_map_webaudio  | https://webaudio.github.io/web-audio-api/#ChannelOrdering | 
    |-----------------------------------|-----------------------------------------------------------|

Below are the channel maps used by default in miniaudio (ma_standard_channel_map_default):

    |---------------|------------------------------|
    | Channel Count | Mapping                      |
    |---------------|------------------------------|
    | 1 (Mono)      | 0: MA_CHANNEL_MONO           |
    |---------------|------------------------------|
    | 2 (Stereo)    | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |---------------|------------------------------|
    | 3             | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |---------------|------------------------------|
    | 4 (Surround)  | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |               | 3: MA_CHANNEL_BACK_CENTER    |
    |---------------|------------------------------|
    | 5             | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |               | 3: MA_CHANNEL_BACK_LEFT      |
    |               | 4: MA_CHANNEL_BACK_RIGHT     |
    |---------------|------------------------------|
    | 6 (5.1)       | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |               | 3: MA_CHANNEL_LFE            |
    |               | 4: MA_CHANNEL_SIDE_LEFT      |
    |               | 5: MA_CHANNEL_SIDE_RIGHT     |
    |---------------|------------------------------|
    | 7             | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |               | 3: MA_CHANNEL_LFE            |
    |               | 4: MA_CHANNEL_BACK_CENTER    |
    |               | 4: MA_CHANNEL_SIDE_LEFT      |
    |               | 5: MA_CHANNEL_SIDE_RIGHT     |
    |---------------|------------------------------|
    | 8 (7.1)       | 0: MA_CHANNEL_FRONT_LEFT     |
    |               | 1: MA_CHANNEL_FRONT_RIGHT    |
    |               | 2: MA_CHANNEL_FRONT_CENTER   |
    |               | 3: MA_CHANNEL_LFE            |
    |               | 4: MA_CHANNEL_BACK_LEFT      |
    |               | 5: MA_CHANNEL_BACK_RIGHT     |
    |               | 6: MA_CHANNEL_SIDE_LEFT      |
    |               | 7: MA_CHANNEL_SIDE_RIGHT     |
    |---------------|------------------------------|
    | Other         | All channels set to 0. This  |
    |               | is equivalent to the same    |
    |               | mapping as the device.       |
    |---------------|------------------------------|



Resampling
==========
Resampling is achieved with the `ma_resampler` object. To create a resampler object, do something like the following:

    ```c
    ma_resampler_config config = ma_resampler_config_init(ma_format_s16, channels, sampleRateIn, sampleRateOut, ma_resample_algorithm_linear);
    ma_resampler resampler;
    ma_result result = ma_resampler_init(&config, &resampler);
    if (result != MA_SUCCESS) {
        // An error occurred...
    }
    ```

Do the following to uninitialize the resampler:

    ```c
    ma_resampler_uninit(&resampler);
    ```

The following example shows how data can be processed

    ```c
    ma_uint64 frameCountIn  = 1000;
    ma_uint64 frameCountOut = 2000;
    ma_result result = ma_resampler_process_pcm_frames(&resampler, pFramesIn, &frameCountIn, pFramesOut, &frameCountOut);
    if (result != MA_SUCCESS) {
        // An error occurred...
    }

    // At this point, frameCountIn contains the number of input frames that were consumed and frameCountOut contains the number of output frames written.
    ```

To initialize the resampler you first need to set up a config (`ma_resampler_config`) with `ma_resampler_config_init()`. You need to specify the sample format
you want to use, the number of channels, the input and output sample rate, and the algorithm.

The sample format can be either `ma_format_s16` or `ma_format_f32`. If you need a different format you will need to perform pre- and post-conversions yourself
where necessary. Note that the format is the same for both input and output. The format cannot be changed after initialization.

The resampler supports multiple channels and is always interleaved (both input and output). The channel count cannot be changed after initialization.

The sample rates can be anything other than zero, and are always specified in hertz. They should be set to something like 44100, etc. The sample rate is the
only configuration property that can be changed after initialization.

The miniaudio resampler supports multiple algorithms:

    |-----------|------------------------------|
    | Algorithm | Enum Token                   |
    |-----------|------------------------------|
    | Linear    | ma_resample_algorithm_linear |
    | Speex     | ma_resample_algorithm_speex  |
    |-----------|------------------------------|

Because Speex is not public domain it is strictly opt-in and the code is stored in separate files. if you opt-in to the Speex backend you will need to consider
it's license, the text of which can be found in it's source files in "extras/speex_resampler". Details on how to opt-in to the Speex resampler is explained in
the Speex Resampler section below.

The algorithm cannot be changed after initialization.

Processing always happens on a per PCM frame basis and always assumes interleaved input and output. De-interleaved processing is not supported. To process
frames, use `ma_resampler_process_pcm_frames()`. On input, this function takes the number of output frames you can fit in the output buffer and the number of
input frames contained in the input buffer. On output these variables contain the number of output frames that were written to the output buffer and the
number of input frames that were consumed in the process. You can pass in NULL for the input buffer in which case it will be treated as an infinitely large
buffer of zeros. The output buffer can also be NULL, in which case the processing will be treated as seek.

The sample rate can be changed dynamically on the fly. You can change this with explicit sample rates with `ma_resampler_set_rate()` and also with a decimal
ratio with `ma_resampler_set_rate_ratio()`. The ratio is in/out.

Sometimes it's useful to know exactly how many input frames will be required to output a specific number of frames. You can calculate this with
`ma_resampler_get_required_input_frame_count()`. Likewise, it's sometimes useful to know exactly how many frames would be output given a certain number of
input frames. You can do this with `ma_resampler_get_expected_output_frame_count()`.

Due to the nature of how resampling works, the resampler introduces some latency. This can be retrieved in terms of both the input rate and the output rate
with `ma_resampler_get_input_latency()` and `ma_resampler_get_output_latency()`.


Resampling Algorithms
---------------------
The choice of resampling algorithm depends on your situation and requirements. The linear resampler is the most efficient and has the least amount of latency,
but at the expense of poorer quality. The Speex resampler is higher quality, but slower with more latency. It also performs several heap allocations internally
for memory management.


Linear Resampling
-----------------
The linear resampler is the fastest, but comes at the expense of poorer quality. There is, however, some control over the quality of the linear resampler which
may make it a suitable option depending on your requirements.

The linear resampler performs low-pass filtering before or after downsampling or upsampling, depending on the sample rates you're converting between. When
decreasing the sample rate, the low-pass filter will be applied before downsampling. When increasing the rate it will be performed after upsampling. By default
a fourth order low-pass filter will be applied. This can be configured via the `lpfOrder` configuration variable. Setting this to 0 will disable filtering.

The low-pass filter has a cutoff frequency which defaults to half the sample rate of the lowest of the input and output sample rates (Nyquist Frequency). This
can be controlled with the `lpfNyquistFactor` config variable. This defaults to 1, and should be in the range of 0..1, although a value of 0 does not make
sense and should be avoided. A value of 1 will use the Nyquist Frequency as the cutoff. A value of 0.5 will use half the Nyquist Frequency as the cutoff, etc.
Values less than 1 will result in more washed out sound due to more of the higher frequencies being removed. This config variable has no impact on performance
and is a purely perceptual configuration.

The API for the linear resampler is the same as the main resampler API, only it's called `ma_linear_resampler`.


Speex Resampling
----------------
The Speex resampler is made up of third party code which is released under the BSD license. Because it is licensed differently to miniaudio, which is public
domain, it is strictly opt-in and all of it's code is stored in separate files. If you opt-in to the Speex resampler you must consider the license text in it's
source files. To opt-in, you must first #include the following file before the implementation of miniaudio.h:

    #include "extras/speex_resampler/ma_speex_resampler.h"

Both the header and implementation is contained within the same file. The implementation can be included in your program like so:

    #define MINIAUDIO_SPEEX_RESAMPLER_IMPLEMENTATION
    #include "extras/speex_resampler/ma_speex_resampler.h"

Note that even if you opt-in to the Speex backend, miniaudio won't use it unless you explicitly ask for it in the respective config of the object you are
initializing. If you try to use the Speex resampler without opting in, initialization of the `ma_resampler` object will fail with `MA_NO_BACKEND`.

The only configuration option to consider with the Speex resampler is the `speex.quality` config variable. This is a value between 0 and 10, with 0 being
the fastest with the poorest quality and 10 being the slowest with the highest quality. The default value is 3.



General Data Conversion
=======================
The `ma_data_converter` API can be used to wrap sample format conversion, channel conversion and resampling into one operation. This is what miniaudio uses
internally to convert between the format requested when the device was initialized and the format of the backend's native device. The API for general data
conversion is very similar to the resampling API. Create a `ma_data_converter` object like this:

    ```c
    ma_data_converter_config config = ma_data_converter_config_init(inputFormat, outputFormat, inputChannels, outputChannels, inputSampleRate, outputSampleRate);
    ma_data_converter converter;
    ma_result result = ma_data_converter_init(&config, &converter);
    if (result != MA_SUCCESS) {
        // An error occurred...
    }
    ```

In the example above we use `ma_data_converter_config_init()` to initialize the config, however there's many more properties that can be configured, such as
channel maps and resampling quality. Something like the following may be more suitable depending on your requirements:

    ```c
    ma_data_converter_config config = ma_data_converter_config_init_default();
    config.formatIn = inputFormat;
    config.formatOut = outputFormat;
    config.channelsIn = inputChannels;
    config.channelsOut = outputChannels;
    config.sampleRateIn = inputSampleRate;
    config.sampleRateOut = outputSampleRate;
    ma_get_standard_channel_map(ma_standard_channel_map_flac, config.channelCountIn, config.channelMapIn);
    config.resampling.linear.lpfOrder = MA_MAX_FILTER_ORDER;
    ```

Do the following to uninitialize the data converter:

    ```c
    ma_data_converter_uninit(&converter);
    ```

The following example shows how data can be processed

    ```c
    ma_uint64 frameCountIn  = 1000;
    ma_uint64 frameCountOut = 2000;
    ma_result result = ma_data_converter_process_pcm_frames(&converter, pFramesIn, &frameCountIn, pFramesOut, &frameCountOut);
    if (result != MA_SUCCESS) {
        // An error occurred...
    }

    // At this point, frameCountIn contains the number of input frames that were consumed and frameCountOut contains the number of output frames written.
    ```

The data converter supports multiple channels and is always interleaved (both input and output). The channel count cannot be changed after initialization.

Sample rates can be anything other than zero, and are always specified in hertz. They should be set to something like 44100, etc. The sample rate is the only
configuration property that can be changed after initialization, but only if the `resampling.allowDynamicSampleRate` member of `ma_data_converter_config` is
set to MA_TRUE. To change the sample rate, use `ma_data_converter_set_rate()` or `ma_data_converter_set_rate_ratio()`. The ratio must be in/out. The resampling
algorithm cannot be changed after initialization.

Processing always happens on a per PCM frame basis and always assumes interleaved input and output. De-interleaved processing is not supported. To process
frames, use `ma_data_converter_process_pcm_frames()`. On input, this function takes the number of output frames you can fit in the output buffer and the number
of input frames contained in the input buffer. On output these variables contain the number of output frames that were written to the output buffer and the
number of input frames that were consumed in the process. You can pass in NULL for the input buffer in which case it will be treated as an infinitely large
buffer of zeros. The output buffer can also be NULL, in which case the processing will be treated as seek.

Sometimes it's useful to know exactly how many input frames will be required to output a specific number of frames. You can calculate this with
`ma_data_converter_get_required_input_frame_count()`. Likewise, it's sometimes useful to know exactly how many frames would be output given a certain number of
input frames. You can do this with `ma_data_converter_get_expected_output_frame_count()`.

Due to the nature of how resampling works, the data converter introduces some latency if resampling is required. This can be retrieved in terms of both the
input rate and the output rate with `ma_data_converter_get_input_latency()` and `ma_data_converter_get_output_latency()`.



Filtering
=========

Biquad Filtering
----------------
Biquad filtering is achieved with the `ma_biquad` API. Example:

    ```c
    ma_biquad_config config = ma_biquad_config_init(ma_format_f32, channels, b0, b1, b2, a0, a1, a2);
    ma_result result = ma_biquad_init(&config, &biquad);
    if (result != MA_SUCCESS) {
        // Error.
    }

    ...

    ma_biquad_process_pcm_frames(&biquad, pFramesOut, pFramesIn, frameCount);
    ```

Biquad filtering is implemented using transposed direct form 2. The numerator coefficients are b0, b1 and b2, and the denominator coefficients are a0, a1 and
a2. The a0 coefficient is required and coefficients must not be pre-normalized.

Supported formats are `ma_format_s16` and `ma_format_f32`. If you need to use a different format you need to convert it yourself beforehand. When using
`ma_format_s16` the biquad filter will use fixed point arithmetic. When using `ma_format_f32`, floating point arithmetic will be used.

Input and output frames are always interleaved.

Filtering can be applied in-place by passing in the same pointer for both the input and output buffers, like so:

    ```c
    ma_biquad_process_pcm_frames(&biquad, pMyData, pMyData, frameCount);
    ```

If you need to change the values of the coefficients, but maintain the values in the registers you can do so with `ma_biquad_reinit()`. This is useful if you
need to change the properties of the filter while keeping the values of registers valid to avoid glitching. Do not use `ma_biquad_init()` for this as it will
do a full initialization which involves clearing the registers to 0. Note that changing the format or channel count after initialization is invalid and will
result in an error.


Low-Pass Filtering
------------------
Low-pass filtering is achieved with the following APIs:

    |---------|------------------------------------------|
    | API     | Description                              |
    |---------|------------------------------------------|
    | ma_lpf1 | First order low-pass filter              |
    | ma_lpf2 | Second order low-pass filter             |
    | ma_lpf  | High order low-pass filter (Butterworth) |
    |---------|------------------------------------------|

Low-pass filter example:

     ```c
    ma_lpf_config config = ma_lpf_config_init(ma_format_f32, channels, sampleRate, cutoffFrequency, order);
    ma_result result = ma_lpf_init(&config, &lpf);
    if (result != MA_SUCCESS) {
        // Error.
    }

    ...

    ma_lpf_process_pcm_frames(&lpf, pFramesOut, pFramesIn, frameCount);
    ```

Supported formats are `ma_format_s16` and` ma_format_f32`. If you need to use a different format you need to convert it yourself beforehand. Input and output
frames are always interleaved.

Filtering can be applied in-place by passing in the same pointer for both the input and output buffers, like so:

    ```c
    ma_lpf_process_pcm_frames(&lpf, pMyData, pMyData, frameCount);
    ```

The maximum filter order is limited to MA_MAX_FILTER_ORDER which is set to 8. If you need more, you can chain first and second order filters together.

    ```c
    for (iFilter = 0; iFilter < filterCount; iFilter += 1) {
        ma_lpf2_process_pcm_frames(&lpf2[iFilter], pMyData, pMyData, frameCount);
    }
    ```

If you need to change the configuration of the filter, but need to maintain the state of internal registers you can do so with `ma_lpf_reinit()`. This may be
useful if you need to change the sample rate and/or cutoff frequency dynamically while maintaing smooth transitions. Note that changing the format or channel
count after initialization is invalid and will result in an error.

The `ma_lpf` object supports a configurable order, but if you only need a first order filter you may want to consider using `ma_lpf1`. Likewise, if you only
need a second order filter you can use `ma_lpf2`. The advantage of this is that they're lighter weight and a bit more efficient.

If an even filter order is specified, a series of second order filters will be processed in a chain. If an odd filter order is specified, a first order filter
will be applied, followed by a series of second order filters in a chain.


High-Pass Filtering
-------------------
High-pass filtering is achieved with the following APIs:

    |---------|-------------------------------------------|
    | API     | Description                               |
    |---------|-------------------------------------------|
    | ma_hpf1 | First order high-pass filter              |
    | ma_hpf2 | Second order high-pass filter             |
    | ma_hpf  | High order high-pass filter (Butterworth) |
    |---------|-------------------------------------------|

High-pass filters work exactly the same as low-pass filters, only the APIs are called `ma_hpf1`, `ma_hpf2` and `ma_hpf`. See example code for low-pass filters
for example usage.


Band-Pass Filtering
-------------------
Band-pass filtering is achieved with the following APIs:

    |---------|-------------------------------|
    | API     | Description                   |
    |---------|-------------------------------|
    | ma_bpf2 | Second order band-pass filter |
    | ma_bpf  | High order band-pass filter   |
    |---------|-------------------------------|

Band-pass filters work exactly the same as low-pass filters, only the APIs are called `ma_bpf2` and `ma_hpf`. See example code for low-pass filters for example
usage. Note that the order for band-pass filters must be an even number which means there is no first order band-pass filter, unlike low-pass and high-pass
filters.


Notch Filtering
---------------
Notch filtering is achieved with the following APIs:

    |-----------|------------------------------------------|
    | API       | Description                              |
    |-----------|------------------------------------------|
    | ma_notch2 | Second order notching filter             |
    |-----------|------------------------------------------|


Peaking EQ Filtering
--------------------
Peaking filtering is achieved with the following APIs:

    |----------|------------------------------------------|
    | API      | Description                              |
    |----------|------------------------------------------|
    | ma_peak2 | Second order peaking filter              |
    |----------|------------------------------------------|


Low Shelf Filtering
-------------------
Low shelf filtering is achieved with the following APIs:

    |-------------|------------------------------------------|
    | API         | Description                              |
    |-------------|------------------------------------------|
    | ma_loshelf2 | Second order low shelf filter            |
    |-------------|------------------------------------------|

Where a high-pass filter is used to eliminate lower frequencies, a low shelf filter can be used to just turn them down rather than eliminate them entirely.


High Shelf Filtering
--------------------
High shelf filtering is achieved with the following APIs:

    |-------------|------------------------------------------|
    | API         | Description                              |
    |-------------|------------------------------------------|
    | ma_hishelf2 | Second order high shelf filter           |
    |-------------|------------------------------------------|

The high shelf filter has the same API as the low shelf filter, only you would use `ma_hishelf` instead of `ma_loshelf`. Where a low shelf filter is used to
adjust the volume of low frequencies, the high shelf filter does the same thing for high frequencies.




Waveform and Noise Generation
=============================

Waveforms
---------
miniaudio supports generation of sine, square, triangle and sawtooth waveforms. This is achieved with the `ma_waveform` API. Example:

    ```c
    ma_waveform_config config = ma_waveform_config_init(FORMAT, CHANNELS, SAMPLE_RATE, ma_waveform_type_sine, amplitude, frequency);

    ma_waveform waveform;
    ma_result result = ma_waveform_init(&config, &waveform);
    if (result != MA_SUCCESS) {
        // Error.
    }

    ...

    ma_waveform_read_pcm_frames(&waveform, pOutput, frameCount);
    ```

The amplitude, frequency and sample rate can be changed dynamically with `ma_waveform_set_amplitude()`, `ma_waveform_set_frequency()` and
`ma_waveform_set_sample_rate()` respectively.

You can reverse the waveform by setting the amplitude to a negative value. You can use this to control whether or not a sawtooth has a positive or negative
ramp, for example.

Below are the supported waveform types:

    |---------------------------|
    | Enum Name                 |
    |---------------------------|
    | ma_waveform_type_sine     |
    | ma_waveform_type_square   |
    | ma_waveform_type_triangle |
    | ma_waveform_type_sawtooth |
    |---------------------------|



Noise
-----
miniaudio supports generation of white, pink and brownian noise via the `ma_noise` API. Example:

    ```c
    ma_noise_config config = ma_noise_config_init(FORMAT, CHANNELS, ma_noise_type_white, SEED, amplitude);

    ma_noise noise;
    ma_result result = ma_noise_init(&config, &noise);
    if (result != MA_SUCCESS) {
        // Error.
    }

    ...

    ma_noise_read_pcm_frames(&noise, pOutput, frameCount);
    ```

The noise API uses simple LCG random number generation. It supports a custom seed which is useful for things like automated testing requiring reproducibility.
Setting the seed to zero will default to MA_DEFAULT_LCG_SEED.

By default, the noise API will use different values for different channels. So, for example, the left side in a stereo stream will be different to the right
side. To instead have each channel use the same random value, set the `duplicateChannels` member of the noise config to true, like so:

    ```c
    config.duplicateChannels = MA_TRUE;
    ```

Below are the supported noise types.

    |------------------------|
    | Enum Name              |
    |------------------------|
    | ma_noise_type_white    |
    | ma_noise_type_pink     |
    | ma_noise_type_brownian |
    |------------------------|



Ring Buffers
============
miniaudio supports lock free (single producer, single consumer) ring buffers which are exposed via the `ma_rb` and `ma_pcm_rb` APIs. The `ma_rb` API operates
on bytes, whereas the `ma_pcm_rb` operates on PCM frames. They are otherwise identical as `ma_pcm_rb` is just a wrapper around `ma_rb`.

Unlike most other APIs in miniaudio, ring buffers support both interleaved and deinterleaved streams. The caller can also allocate their own backing memory for
the ring buffer to use internally for added flexibility. Otherwise the ring buffer will manage it's internal memory for you.

The examples below use the PCM frame variant of the ring buffer since that's most likely the one you will want to use. To initialize a ring buffer, do
something like the following:

    ```c
    ma_pcm_rb rb;
    ma_result result = ma_pcm_rb_init(FORMAT, CHANNELS, BUFFER_SIZE_IN_FRAMES, NULL, NULL, &rb);
    if (result != MA_SUCCESS) {
        // Error
    }
    ```

The `ma_pcm_rb_init()` function takes the sample format and channel count as parameters because it's the PCM varient of the ring buffer API. For the regular
ring buffer that operates on bytes you would call `ma_rb_init()` which leaves these out and just takes the size of the buffer in bytes instead of frames. The
fourth parameter is an optional pre-allocated buffer and the fifth parameter is a pointer to a `ma_allocation_callbacks` structure for custom memory allocation
routines. Passing in NULL for this results in MA_MALLOC() and MA_FREE() being used.

Use `ma_pcm_rb_init_ex()` if you need a deinterleaved buffer. The data for each sub-buffer is offset from each other based on the stride. To manage your sub-
buffers you can use `ma_pcm_rb_get_subbuffer_stride()`, `ma_pcm_rb_get_subbuffer_offset()` and `ma_pcm_rb_get_subbuffer_ptr()`.

Use 'ma_pcm_rb_acquire_read()` and `ma_pcm_rb_acquire_write()` to retrieve a pointer to a section of the ring buffer. You specify the number of frames you
need, and on output it will set to what was actually acquired. If the read or write pointer is positioned such that the number of frames requested will require
a loop, it will be clamped to the end of the buffer. Therefore, the number of frames you're given may be less than the number you requested.

After calling `ma_pcm_rb_acquire_read()` or `ma_pcm_rb_acquire_write()`, you do your work on the buffer and then "commit" it with `ma_pcm_rb_commit_read()` or
`ma_pcm_rb_commit_write()`. This is where the read/write pointers are updated. When you commit you need to pass in the buffer that was returned by the earlier
call to `ma_pcm_rb_acquire_read()` or `ma_pcm_rb_acquire_write()` and is only used for validation. The number of frames passed to `ma_pcm_rb_commit_read()` and
`ma_pcm_rb_commit_write()` is what's used to increment the pointers.

If you want to correct for drift between the write pointer and the read pointer you can use a combination of `ma_pcm_rb_pointer_distance()`,
`ma_pcm_rb_seek_read()` and `ma_pcm_rb_seek_write()`. Note that you can only move the pointers forward, and you should only move the read pointer forward via
the consumer thread, and the write pointer forward by the producer thread. If there is too much space between the pointers, move the read pointer forward. If
there is too little space between the pointers, move the write pointer forward.

You can use a ring buffer at the byte level instead of the PCM frame level by using the `ma_rb` API. This is exactly the sample, only you will use the `ma_rb`
functions instead of `ma_pcm_rb` and instead of frame counts you'll pass around byte counts.

The maximum size of the buffer in bytes is 0x7FFFFFFF-(MA_SIMD_ALIGNMENT-1) due to the most significant bit being used to encode a flag and the internally
managed buffers always being aligned to MA_SIMD_ALIGNMENT.

Note that the ring buffer is only thread safe when used by a single consumer thread and single producer thread.



Backends
========
The following backends are supported by miniaudio.

    |-------------|-----------------------|--------------------------------------------------------|
    | Name        | Enum Name             | Supported Operating Systems                            |
    |-------------|-----------------------|--------------------------------------------------------|
    | WASAPI      | ma_backend_wasapi     | Windows Vista+                                         |
    | DirectSound | ma_backend_dsound     | Windows XP+                                            |
    | WinMM       | ma_backend_winmm      | Windows XP+ (may work on older versions, but untested) |
    | Core Audio  | ma_backend_coreaudio  | macOS, iOS                                             |
    | ALSA        | ma_backend_alsa       | Linux                                                  |
    | PulseAudio  | ma_backend_pulseaudio | Cross Platform (disabled on Windows, BSD and Android)  |
    | JACK        | ma_backend_jack       | Cross Platform (disabled on BSD and Android)           |
    | sndio       | ma_backend_sndio      | OpenBSD                                                |
    | audio(4)    | ma_backend_audio4     | NetBSD, OpenBSD                                        |
    | OSS         | ma_backend_oss        | FreeBSD                                                |
    | AAudio      | ma_backend_aaudio     | Android 8+                                             |
    | OpenSL|ES   | ma_backend_opensl     | Android (API level 16+)                                |
    | Web Audio   | ma_backend_webaudio   | Web (via Emscripten)                                   |
    | Null        | ma_backend_null       | Cross Platform (not used on Web)                       |
    |-------------|-----------------------|--------------------------------------------------------|

Some backends have some nuance details you may want to be aware of.

WASAPI
------
- Low-latency shared mode will be disabled when using an application-defined sample rate which is different to the device's native sample rate. To work around
  this, set wasapi.noAutoConvertSRC to true in the device config. This is due to IAudioClient3_InitializeSharedAudioStream() failing when the
  AUDCLNT_STREAMFLAGS_AUTOCONVERTPCM flag is specified. Setting wasapi.noAutoConvertSRC will result in miniaudio's lower quality internal resampler being used
  instead which will in turn enable the use of low-latency shared mode.

PulseAudio
----------
- If you experience bad glitching/noise on Arch Linux, consider this fix from the Arch wiki:
    https://wiki.archlinux.org/index.php/PulseAudio/Troubleshooting#Glitches,_skips_or_crackling
  Alternatively, consider using a different backend such as ALSA.

Android
-------
- To capture audio on Android, remember to add the RECORD_AUDIO permission to your manifest:
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
- With OpenSL|ES, only a single ma_context can be active at any given time. This is due to a limitation with OpenSL|ES.
- With AAudio, only default devices are enumerated. This is due to AAudio not having an enumeration API (devices are enumerated through Java). You can however
  perform your own device enumeration through Java and then set the ID in the ma_device_id structure (ma_device_id.aaudio) and pass it to ma_device_init().
- The backend API will perform resampling where possible. The reason for this as opposed to using miniaudio's built-in resampler is to take advantage of any
  potential device-specific optimizations the driver may implement.

UWP
---
- UWP only supports default playback and capture devices.
- UWP requires the Microphone capability to be enabled in the application's manifest (Package.appxmanifest):
      <Package ...>
          ...
          <Capabilities>
              <DeviceCapability Name="microphone" />
          </Capabilities>
      </Package>

Web Audio / Emscripten
----------------------
- You cannot use -std=c* compiler flags, nor -ansi. This only applies to the Emscripten build.
- The first time a context is initialized it will create a global object called "miniaudio" whose primary purpose is to act as a factory for device objects.
- Currently the Web Audio backend uses ScriptProcessorNode's, but this may need to change later as they've been deprecated.
- Google has implemented a policy in their browsers that prevent automatic media output without first receiving some kind of user input. The following web page
  has additional details: https://developers.google.com/web/updates/2017/09/autoplay-policy-changes. Starting the device may fail if you try to start playback
  without first handling some kind of user input.



Miscellaneous Notes
===================
- Automatic stream routing is enabled on a per-backend basis. Support is explicitly enabled for WASAPI and Core Audio, however other backends such as
  PulseAudio may naturally support it, though not all have been tested.
- The contents of the output buffer passed into the data callback will always be pre-initialized to zero unless the noPreZeroedOutputBuffer config variable in
  ma_device_config is set to true, in which case it'll be undefined which will require you to write something to the entire buffer.
- By default miniaudio will automatically clip samples. This only applies when the playback sample format is configured as ma_format_f32. If you are doing
  clipping yourself, you can disable this overhead by setting noClip to true in the device config.
- The sndio backend is currently only enabled on OpenBSD builds.
- The audio(4) backend is supported on OpenBSD, but you may need to disable sndiod before you can use it.
- Note that GCC and Clang requires "-msse2", "-mavx2", etc. for SIMD optimizations.