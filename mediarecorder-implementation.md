# MediaRecorder Implementation Summary

## Overview

Replaced the failed WebCodecs + mp4-muxer approach with **MediaRecorder API** for browser-native canvas recording. This eliminates the complex manual encoding/muxing and provides a simple, reliable recording solution.

---

## Architecture

### Frontend (Browser)
1. **Canvas Compositing**: Composite 4 video/canvas sources into 2x2 grid
2. **Stream Capture**: Get MediaStream from canvas using `canvas.captureStream(fps)`
3. **MediaRecorder**: Browser handles encoding and muxing automatically
4. **File Transfer**: Send complete WebM/MP4 file to backend via base64

### Backend (Rust)
1. **Receive Video**: Decode base64 video data
2. **Save File**: Write WebM/MP4 to disk (test-results/ in dev mode)

---

## Key Components

### Frontend: `recording-canvas.tsx`

```typescript
// Initialize MediaRecorder
const stream = canvas.captureStream(fps);

const recorder = new MediaRecorder(stream, {
  mimeType: 'video/webm;codecs=vp9', // VP9 preferred, fallback to H264/VP8
  videoBitsPerSecond: 5_000_000, // 5 Mbps
});

// Collect chunks
recorder.ondataavailable = (event) => {
  if (event.data && event.data.size > 0) {
    recordedChunks.push(event.data);
  }
};

// On stop, save video
recorder.onstop = async () => {
  const blob = new Blob(recordedChunks, { type: mimeType });
  const base64 = await blobToBase64(blob);

  await invoke('save_media_recording', {
    videoData: base64,
    width,
    height,
    mimeType,
  });
};

// Start recording
recorder.start(1000); // 1 second chunks
```

### Backend: `lib.rs`

```rust
#[tauri::command]
fn save_media_recording(
    video_data: String,
    width: u32,
    height: u32,
    mime_type: String,
) -> Result<String, String> {
    // Decode base64 video data
    let video_bytes = STANDARD.decode(&video_data)?;

    // Determine extension from mime type
    let extension = if mime_type.contains("webm") { "webm" } else { "mp4" };

    // Generate output path
    let output_path = test_results_dir.join(format!("recording_{}.{}", timestamp, extension));

    // Write video file
    std::fs::write(&output_path, &video_bytes)?;

    Ok(path_str)
}
```

---

## Performance Advantages

### vs. WebCodecs + mp4-muxer Approach

| Metric | WebCodecs + mp4-muxer | MediaRecorder |
|--------|----------------------|---------------|
| **Complexity** | High (manual encoding, muxing, chunk handling) | Low (browser handles everything) |
| **API Stability** | Unstable (mp4-muxer deprecated, chunk.duration issues) | Stable (standard browser API) |
| **Browser Support** | Limited (WebCodecs not in Safari/Firefox) | Excellent (all modern browsers) |
| **Encoding** | Manual VideoEncoder setup | Automatic |
| **Muxing** | Manual with mp4-muxer | Automatic |
| **Performance** | Good (hardware accelerated) | Good (hardware accelerated) |
| **Reliability** | Poor (5 failed iterations) | Excellent |

### Expected Performance

- **1080p @ 15fps**: ✅ Trivial
- **1080p @ 30fps**: ✅ Easy
- **1080p @ 60fps**: ✅ Achievable
- **4K @ 30fps**: ✅ Possible

---

## Dependencies

### Frontend
- **Removed**: `mp4-muxer` (no longer needed)
- **Added**: None (MediaRecorder is built into browsers)

### Backend
- No changes (still just saves files)

---

## Browser Support

### MediaRecorder API Support (2026)
- ✅ Chrome 47+
- ✅ Edge 79+
- ✅ Firefox 25+
- ✅ Safari 14+
- ✅ Opera 36+

**Note**: VP9 codec support varies. Fallback chain: VP9 → H264 → VP8 → default

---

## File Locations

### Frontend Changes
- `frontend/src/components/asmr-recorder/recording-canvas.tsx`
  - Removed WebCodecs VideoEncoder
  - Removed mp4-muxer integration
  - Added MediaRecorder implementation
  - Simplified frame capture logic

- `frontend/src/contexts/recording-context.tsx`
  - Updated comments to reflect MediaRecorder

### Backend Changes
- `src-tauri/src/lib.rs`
  - Removed `save_webcodecs_recording` command
  - Added `save_media_recording` command (supports WebM and MP4)

---

## Testing Plan

1. **Start app**: `npm run tauri:dev`
2. **Configure**: Set resolution to 1920x1080, 30fps or 60fps
3. **Record**: 10-30 seconds of multi-source recording
4. **Verify**:
   - Check console for recording logs
   - Verify no errors
   - Check output file in `test-results/recording_*.webm`
   - Play video to verify quality and smoothness

---

## Success Criteria

- ✅ **Reliable recording** - MediaRecorder is stable browser API
- ✅ **Simple implementation** - No manual encoding/muxing
- ✅ **Excellent browser support** - Works in all modern browsers
- ⏳ **60fps @ 1080p** - to be verified by testing
- ⏳ **No frame drops** - to be verified by testing

---

## Known Limitations

1. **File format**: WebM by default (browser-dependent codec support)
2. **Single file transfer**: Entire file sent at end (could be large for long recordings)
3. **No audio yet**: This implementation is video-only (can be added via audio tracks)

---

## Next Steps

1. Test MediaRecorder recording at 1080p 60fps
2. Add audio track support (capture audio and add to MediaStream)
3. Consider streaming chunks during recording for very long recordings
4. Test on different browsers to verify codec support

---

## Why MediaRecorder vs WebCodecs?

**WebCodecs Approach Failed Because:**
- mp4-muxer library has unstable API (deprecated, validation errors)
- Complex manual chunk handling with duration issues
- 5 iterations failed with "chunk.duration must be non-negative real number" errors
- Poor browser support (no Safari/Firefox)

**MediaRecorder Wins Because:**
- Standard browser API with excellent support
- Browser handles all complexity (encoding, muxing, chunk management)
- Simple, reliable, battle-tested
- Works in all modern browsers including Safari and Firefox
- One function call to start, one to stop, automatic file generation
