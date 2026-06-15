# Fixing Screen Recording Issue - Development Log

**Date:** January 20-23, 2026
**Issue:** Screen recording exports with frozen/black screen after a few seconds
**Resolution:** Successfully implemented after multiple iterations

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Investigation Phase](#investigation-phase)
3. [Root Causes Identified](#root-causes-identified)
4. [Fix Iterations](#fix-iterations)
5. [Final Solution](#final-solution)
6. [Key Learnings](#key-learnings)
7. [Files Modified](#files-modified)

---

## Problem Statement

The ASMR recorder app's screen recording feature was producing video files where:

- The screen appeared frozen after export
- Later tests showed recording worked for 3-8 seconds before turning black
- The issue occurred consistently regardless of recording duration

**Test Environment:**

- Screen resolution: 1080x1920 (portrait mode)
- Target frame rate: 30fps
- Output format: H.264/MP4 via FFmpeg

---

## Investigation Phase

### Files Analyzed

1. `src-tauri/src/screen_macos.rs` - macOS screen capture using ScreenCaptureKit
2. `src-tauri/src/encoder.rs` - FFmpeg-based video encoding
3. `src-tauri/src/compositor.rs` - Video frame compositing
4. `src-tauri/src/manager.rs` - Recording orchestration

### Key Technologies

- **ScreenCaptureKit (SCK)**: Apple's modern screen capture API
- **screencapturekit-rs**: Rust bindings (v1.5.0)
- **ffmpeg-next**: Rust FFmpeg bindings for H.264 encoding
- **crossbeam-channel**: For inter-thread frame communication

---

## Root Causes Identified

### 1. Missing Cursor Capture Configuration

**Problem:** SCStreamConfiguration was missing `.with_shows_cursor(true)`

**Impact:** ScreenCaptureKit only delivers frames when screen content changes. Without cursor tracking, static screens produced no frames.

### 2. Channel Blocking Causing Pixel Buffer Pool Exhaustion

**Problem:** The frame handler used `send_timeout(100ms)` which blocked SCK's callback thread.

**Impact:** ScreenCaptureKit maintains a limited pixel buffer pool. When callbacks are blocked:

- Buffers can't be recycled
- Pool becomes exhausted
- `image_buffer()` returns `None`
- Recording gets empty frames (black screen)

**Evidence from logs:**

```
Screen capture: 100 callbacks with no image buffer
FrameHandler dropped: 476 frames captured, 100 empty buffers (ratio: 17.4%)
```

### 3. Encoder Too Slow for Real-Time Capture

**Problem:** Default "medium" x264 preset couldn't encode 1080x1920 @ 30fps in real-time.

**Impact:** Frame queue filled up, causing backpressure throughout the pipeline.

### 4. Expensive BGRA to RGBA Conversion in Compositor

**Problem:** Every frame went through pixel-by-pixel color channel swapping:

```rust
// Old code - expensive per-pixel operation
fn to_rgba(&self) -> Vec<u8> {
    for each pixel {
        rgba.push(self.data[offset + 2]); // R (was B)
        rgba.push(self.data[offset + 1]); // G
        rgba.push(self.data[offset]);     // B (was R)
        rgba.push(self.data[offset + 3]); // A
    }
}
```

**Impact:** CPU saturation, especially at high resolutions.

---

## Fix Iterations

### Iteration 1: Basic Configuration Fixes

**Changes:**

- Added `.with_shows_cursor(true)` to SCStreamConfiguration
- Increased channel capacity from 5 to 30 frames
- Changed from `try_send` to `send_timeout`

**Result:** Recording started working but turned black after 3 seconds.

### Iteration 2: Reduce Callback Blocking

**Changes:**

- Reduced send timeout from 100ms to 5ms
- Increased channel capacity to 60 frames
- Added diagnostic logging for empty buffers

**Result:** Recording lasted longer but still failed. Logs showed frequent `image_buffer() returned None`.

### Iteration 3: Faster Encoder Settings

**Changes:**

```rust
// encoder.rs
video_options.set("preset", "ultrafast");  // Was "medium"
video_options.set("tune", "zerolatency");  // For real-time recording
```

**Result:** Encoder speed improved from ~11fps to ~24fps, but still not enough.

### Iteration 4: Adaptive Frame Skipping

**Changes in manager.rs:**

```rust
// Drain all frames, keep only latest
while let Ok(screen_frame) = receiver.try_recv() {
    if latest_screen_frame.is_some() {
        skipped_frames += 1;
    }
    latest_screen_frame = Some(screen_frame);
}

// Skip frames when queue pressure > 80%
let queue_pressure = queue_len as f32 / 60.0;
let should_skip = queue_pressure > 0.8;
```

**Result:** Recording reached 8 seconds before failing. Progress but not stable.

### Iteration 5: BGRA Fast Path (Major Optimization)

**Key Insight:** FFmpeg's scaler can convert BGRA to YUV420P directly. The BGRA to RGBA step was unnecessary.

**Changes:**

1. **Added `is_bgra` flag to CompositeFrame:**

```rust
pub struct CompositeFrame {
    pub data: Vec<u8>,
    pub width: u32,
    pub height: u32,
    pub timestamp: Duration,
    pub is_bgra: bool,  // NEW: indicates pixel format
}
```

2. **Added fast path in compositor:**

```rust
fn composite(&self, screen_frame: &ScreenFrame, webcam_frame: Option<&WebcamFrame>) -> CompositeFrame {
    // Fast path: skip BGRA to RGBA when no webcam and dimensions match
    if !self.config.include_webcam
        && screen_frame.width == self.config.output_width
        && screen_frame.height == self.config.output_height
    {
        return self.composite_fast_path(screen_frame);
    }
    // ... slow path with RGBA conversion for webcam overlay
}

fn composite_fast_path(&self, screen_frame: &ScreenFrame) -> CompositeFrame {
    CompositeFrame {
        data: screen_frame.to_packed_bgra(),  // Just handle stride, no color conversion
        is_bgra: true,
        // ...
    }
}
```

3. **Added `to_packed_bgra()` method in screen.rs:**

```rust
pub fn to_packed_bgra(&self) -> Vec<u8> {
    let row_bytes = (self.width * 4) as usize;

    // Fast path: already tightly packed
    if self.stride == row_bytes {
        return self.data[..row_bytes * self.height as usize].to_vec();
    }

    // Remove stride padding only, no color conversion
    let mut bgra = Vec::with_capacity(row_bytes * self.height as usize);
    for y in 0..self.height as usize {
        let row_start = y * self.stride;
        bgra.extend_from_slice(&self.data[row_start..row_start + row_bytes]);
    }
    bgra
}
```

4. **Updated encoder to handle both formats:**

```rust
// Create both scalers upfront
let mut bgra_scaler = Context::get(Pixel::BGRA, ...);
let mut rgba_scaler = Context::get(Pixel::RGBA, ...);

// Choose based on input format
let conversion_result = if composite_frame.is_bgra {
    bgra_scaler.run(&bgra_frame, &mut yuv_frame)
} else {
    rgba_scaler.run(&rgba_frame, &mut yuv_frame)
};
```

**Result:** Recording reached 15-19 seconds. Major improvement but empty buffers still accumulated.

### Iteration 6: Eliminate All Callback Blocking (Final Fix)

**Key Insight:** Even 5ms timeout was too long. Any blocking causes pixel buffer pool exhaustion.

**Changes:**

1. **Changed to `try_send()` - never blocks:**

```rust
// screen_macos.rs - CRITICAL FIX
// Old: self.sender.send_timeout(frame, Duration::from_millis(5))
// New:
if let Err(_) = self.sender.try_send(frame) {
    // Frame dropped but callback returns immediately
    // This prevents pixel buffer pool exhaustion
}
```

2. **Increased buffer capacity to 120 frames (4 seconds):**

```rust
const FRAME_CHANNEL_CAPACITY: usize = 120;
```

**Result:** Recording works for entire session duration.

---

## Final Solution

The working solution combines all optimizations:

### screen_macos.rs

- `try_send()` for non-blocking frame delivery
- 120 frame channel capacity
- `.with_shows_cursor(true)` for continuous frame delivery
- Diagnostic logging for empty buffer tracking

### compositor.rs

- BGRA fast path when webcam disabled
- `is_bgra` flag to indicate pixel format
- Falls back to RGBA only when webcam overlay needed

### encoder.rs

- Dual scalers (BGRA and RGBA to YUV420P)
- "ultrafast" preset with "zerolatency" tune
- Format-aware frame processing

### manager.rs

- Adaptive frame skipping with 80% queue pressure threshold
- 120 frame composite channel
- Latest-frame-only processing to reduce lag

---

## Key Learnings

### 1. ScreenCaptureKit Pixel Buffer Pool

SCK uses a finite pool of pixel buffers. The callback MUST return quickly to allow buffer recycling. Blocking causes `image_buffer()` to return `None`.

**Rule:** Never block in SCK callbacks. Use `try_send()`, not `send_timeout()`.

### 2. Color Space Conversions Are Expensive

BGRA to RGBA to YUV involves two conversions. BGRA to YUV directly eliminates one entire pass over all pixels.

**Rule:** Minimize color space conversions. Use the format closest to the final output.

### 3. Real-Time Encoding Requires Fast Presets

H.264 "medium" preset is designed for quality, not speed. For real-time capture:

- Use "ultrafast" or "superfast" preset
- Use "zerolatency" tune (disables B-frames, reduces latency)

### 4. Backpressure Management

When the encoder can't keep up:

- Skip frames rather than blocking
- Keep only the latest frame
- Monitor queue pressure and adapt

### 5. Buffer Sizing

Larger buffers absorb temporary slowdowns but increase latency and memory usage.

- 120 frames (4 seconds) provides good balance
- Must update all related queue pressure calculations

---

## Files Modified

| File                            | Changes                                                     |
| ------------------------------- | ----------------------------------------------------------- |
| `src-tauri/src/screen_macos.rs` | Non-blocking `try_send()`, increased buffer, cursor capture |
| `src-tauri/src/screen.rs`       | Added `to_packed_bgra()` method                             |
| `src-tauri/src/compositor.rs`   | BGRA fast path, `is_bgra` flag                              |
| `src-tauri/src/encoder.rs`      | Dual scalers, ultrafast preset                              |
| `src-tauri/src/manager.rs`      | Adaptive frame skipping, increased buffers                  |

---

## Performance Metrics

### Before Fixes

- Recording duration: 3-8 seconds before black screen
- Empty buffer ratio: 17-25%
- Effective FPS: 8-11 fps

### After Fixes

- Recording duration: Full session (tested 19+ seconds)
- Empty buffer ratio: <5%
- Effective FPS: 24+ fps
- 0 skipped frames in compositor

---

## Future Improvements

1. **Hardware Encoding**: Use VideoToolbox (macOS) for H.264 encoding instead of software libx264
2. **Adaptive Frame Rate**: Dynamically reduce capture FPS when system is under load
3. **Memory Optimization**: Reuse frame buffers instead of allocating new ones
4. **Resolution Scaling**: Offer lower resolution options for slower systems
