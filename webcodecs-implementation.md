# WebCodecs Implementation Summary

## Overview

Replaced the failed JPEG compression approach with **WebCodecs API** for hardware-accelerated video encoding directly in the browser. This eliminates the CPU-intensive decompression bottleneck and enables high-resolution, high-framerate recording.

---

## Architecture

### Frontend (Browser)
1. **Canvas Compositing**: Composite 4 video/canvas sources into 2x2 grid
2. **VideoFrame Creation**: Convert canvas to `VideoFrame` object
3. **Hardware Encoding**: `VideoEncoder` encodes to H.264 (GPU-accelerated)
4. **MP4 Muxing**: `mp4-muxer` library muxes encoded chunks into MP4
5. **File Transfer**: Send complete MP4 file to backend via base64

### Backend (Rust)
1. **Receive MP4**: Decode base64 MP4 data
2. **Save File**: Write MP4 to disk (test-results/ in dev mode)

---

## Key Components

### Frontend: `recording-canvas.tsx`

```typescript
// Initialize VideoEncoder + MP4 Muxer
const muxer = new Muxer({
  target: new ArrayBufferTarget(),
  video: { codec: 'avc', width, height },
  fastStart: 'in-memory',
});

const encoder = new VideoEncoder({
  output: (chunk, metadata) => {
    muxer.addVideoChunk(chunk, metadata); // Send to muxer
  },
  error: (error) => console.error(error),
});

encoder.configure({
  codec: 'avc1.42001f', // H.264 Baseline
  width: 1920,
  height: 1080,
  framerate: 60,
  bitrate: 5_000_000, // 5 Mbps
  hardwareAcceleration: 'prefer-hardware',
  latencyMode: 'realtime',
});
```

**Per-Frame Capture:**
```typescript
// Create VideoFrame from canvas
const videoFrame = new VideoFrame(canvas, { timestamp: timestampUs });

// Encode (hardware-accelerated, async)
encoder.encode(videoFrame, { keyFrame: frameCount % 30 === 0 });
videoFrame.close();
```

**On Stop:**
```typescript
// Finalize muxer to get complete MP4
muxer.finalize();
const mp4Buffer = muxer.target.buffer;

// Send to backend
await invoke('save_webcodecs_recording', {
  mp4Data: base64(mp4Buffer),
  width,
  height,
});
```

### Backend: `lib.rs`

```rust
#[tauri::command]
fn save_webcodecs_recording(
    mp4_data: String,
    width: u32,
    height: u32,
) -> Result<String, String> {
    // Decode base64 MP4 data
    let mp4_bytes = STANDARD.decode(&mp4_data)?;

    // Generate output path
    let output_path = test_results_dir.join("webcodecs_recording_{timestamp}.mp4");

    // Write MP4 file
    std::fs::write(&output_path, &mp4_bytes)?;

    Ok(path_str)
}
```

---

## Performance Advantages

### vs. JPEG Compression Approach

| Metric | JPEG Approach | WebCodecs Approach |
|--------|---------------|-------------------|
| **Frontend** | 10ms JPEG compress + 2ms base64 | <1ms VideoFrame + <1ms encode call (async) |
| **IPC Transfer** | 600KB JPEG per frame | 5MB total MP4 file (at end) |
| **Backend** | 270ms JPEG decode + frame processing | 0ms (just file write) |
| **Total per frame** | ~282ms | ~1-2ms |
| **Max FPS** | ~3-4 fps | 60+ fps |

### Actual Expected Performance

- **1080p @ 15fps**: ✅ Trivial (~0.5ms per frame budget: 66ms)
- **1080p @ 30fps**: ✅ Easy (~1ms per frame budget: 33ms)
- **1080p @ 60fps**: ✅ Achievable (~2ms per frame budget: 16ms)
- **4K @ 30fps**: ✅ Possible with hardware encoder

---

## Dependencies

### Frontend
```json
{
  "mp4-muxer": "^5.2.2"
}
```

**Note**: `mp4-muxer` is deprecated in favor of `mediabunny`, but still works perfectly for WebCodecs use case and has a simpler API.

### Backend
- No new dependencies (removed `jpeg-decoder`)

---

## Browser Support

### WebCodecs API Support (2026)
- ✅ Chrome 94+
- ✅ Edge 94+
- ✅ Opera 80+
- ⚠️ Firefox: Behind flag (needs `dom.media.webcodecs.enabled`)
- ❌ Safari: Not yet supported

**Workaround for unsupported browsers**: Fall back to raw RGBA transfer at lower resolution.

---

## File Locations

### Frontend Changes
- `frontend/src/components/asmr-recorder/recording-canvas.tsx`
  - Removed JPEG compression
  - Added WebCodecs VideoEncoder
  - Added mp4-muxer integration

- `frontend/src/contexts/recording-context.tsx`
  - Updated comments to reflect WebCodecs

### Backend Changes
- `src-tauri/src/lib.rs`
  - Removed `receive_video_frame_jpeg` command
  - Added `save_webcodecs_recording` command

- `src-tauri/Cargo.toml`
  - Removed `jpeg-decoder` dependency

---

## Testing Plan

1. **Start app**: `npm run tauri:dev`
2. **Configure**: Set resolution to 1920x1080, 60fps
3. **Record**: 10-30 seconds of multi-source recording
4. **Verify**:
   - Check console for encoding performance logs
   - Verify no frame drops
   - Check output file in `test-results/webcodecs_recording_*.mp4`
   - Play video to verify quality and smoothness

---

## Success Criteria

- ✅ **No JPEG decompression** - eliminated 270ms bottleneck
- ✅ **Hardware acceleration** - GPU encoding instead of CPU
- ✅ **Minimal IPC overhead** - single MP4 transfer instead of per-frame
- ⏳ **60fps @ 1080p** - to be verified by testing
- ⏳ **<5% frame drops** - to be verified by testing

---

## Known Limitations

1. **Browser support**: Not available in Safari/Firefox yet
2. **Single MP4 transfer**: Entire file sent at end (could be large for long recordings)
3. **No audio yet**: This implementation is video-only (audio can be added via `AudioEncoder`)

---

## Next Steps

1. Test WebCodecs recording at 1080p 60fps
2. Add audio encoding support (WebCodecs AudioEncoder + mp4-muxer audio track)
3. Add fallback for unsupported browsers
4. Consider streaming MP4 chunks during recording for very long recordings
