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
| `gstreamer1.0-libav` | FFmpeg-backed codec elements |

On Jetson targets, `gstreamer1.0-plugins-nvvideo4linux2` is also installed, providing the `nvv4l2h264enc` hardware H.264 encoder.

### Available encoders

| Encoder | GStreamer element | Hardware |
|---|---|---|
| H.264 (NVIDIA hardware) | `nvv4l2h264enc` | Jetson only |
| H.264 (software) | `x264enc` | All targets |
| VP8 | `vp8enc` | All targets |

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
