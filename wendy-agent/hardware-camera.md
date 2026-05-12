# Camera Access

Camera access is used for webcam streaming, computer vision, image capture, and ML inference pipelines. Wendy apps cannot open `/dev/video*` unless the app declares a camera entitlement.

Add the entitlement from the project directory:

```bash
wendy project entitlements add camera
```

The older `video` entitlement is still accepted for compatibility, but new projects should use `camera`.

## Device Commands

Camera device inspection and streaming currently live under `wendy device video`:

```bash
wendy device video list
wendy device video stream --id 0
wendy device video stream --id 0 --width 1280 --height 720 --fps 30
wendy device video stream --id 0 --stdout
```

For generic capability inspection:

```bash
wendy device hardware list --category camera
```

## GStreamer Support

WendyOS includes a full GStreamer stack for camera streaming pipelines. The following packages are installed on all images:

| Package | Contents |
|---|---|
| `gstreamer1.0-plugins-base` | Core elements: `v4l2src`, `videoconvert`, `capsfilter`, etc. |
| `gstreamer1.0-plugins-good` | `vp8enc`, `webmmux`, `rtpvp8pay`, and other good-quality plugins |
| `gstreamer1.0-plugins-bad` | Extended elements (Vulkan sink is disabled — no windowing system) |
| `gstreamer1.0-plugins-ugly` | `x264enc` (H.264 software encoder) |
| `gstreamer1.0-libav` | FFmpeg-backed codec elements (`avenc_h264`, etc.) |

On Jetson targets, `gstreamer1.0-plugins-nvvideo4linux2` is also installed, providing the `nvv4l2h264enc` hardware H.264 encoder.

### GStreamer invocation

The agent invokes `gst-launch` with the `-q` (quiet) flag to suppress status messages such as `"Setting pipeline to PLAYING"` from being written to stdout. Without this flag, those messages corrupt the binary H.264 or VP8 stream read from stdout.

### Available encoders

| Encoder | GStreamer element | Hardware |
|---|---|---|
| H.264 (NVIDIA hardware) | `nvv4l2h264enc` | Jetson only |
| H.264 (software, libx264) | `x264enc` | All targets |
| H.264 (software, FFmpeg) | `avenc_h264` | All targets |
| H.264 (software, OpenH264) | `openh264enc` | All targets |
| VP8 | `vp8enc` | All targets |

### H.264 colour format and profile handling

All H.264 encoder pipelines insert a `video/x-raw,format=I420` caps filter before the encoder element. This forces 4:2:0 chroma subsampling into the encoder, preventing encoders such as `x264enc` from selecting H.264 High 4:4:4 Predictive (profile 244) when the upstream colour format is 4:4:4. Profile 244 is rejected by VideoToolbox and most hardware decoders.

Forcing I420 input does not by itself enforce a specific H.264 output profile; explicit profile caps are added only where needed:

| Encoder | Pipeline segment |
|---|---|
| `v4l2h264enc` | `videoconvert ! video/x-raw,format=I420 ! v4l2h264enc ! video/x-h264,profile=baseline` |
| `x264enc` | `videoconvert ! video/x-raw,format=I420 ! x264enc tune=zerolatency profile=high` |
| `openh264enc` | `videoconvert ! video/x-raw,format=I420 ! openh264enc` |
| `avenc_h264` | `videoconvert ! video/x-raw,format=I420 ! avenc_h264` |
| all others | `videoconvert ! video/x-raw,format=I420 ! <encoder>` |

`v4l2h264enc` is capped to `baseline` because some V4L2 hardware encoders report H.264 support but fail to deliver frames; if no frames are received before the first DQBUF error, the agent falls back to the GStreamer software encoder path automatically.

## App Configuration

```json
{
  "appId": "com.example.camera",
  "version": "1.0.0",
  "entitlements": [
    { "type": "camera" }
  ]
}
```

To restrict the app to one camera device:

```json
{ "type": "camera", "allowlist": ["/dev/video0"] }
```
