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
