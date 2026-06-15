This plan outlines the steps to replace the slow pure-Rust decompression with the SIMD-accelerated **libjpeg-turbo** library via the `turbojpeg` crate. This shift is designed to drop backend decompression time from ~80ms to <10ms, enabling smooth 1080p recording at 15–30fps.

---

## 🛠 Prerequisites

### 1. System Dependencies

`libjpeg-turbo` must be installed on the host machine for the Rust bindings to link against:

- **macOS**: `brew install libjpeg-turbo`
- **Ubuntu/Debian**: `sudo apt-get install libturbojpeg0-dev`
- **Windows**: Download and install the [libjpeg-turbo SDK](https://libjpeg-turbo.org/Downloads/DigitalSignatures). Ensure the `bin` folder is in your `PATH`.

### 2. Cargo Dependencies

Update your `src-tauri/Cargo.toml` with the following:

```toml
[dependencies]
# Use turbojpeg for SIMD-accelerated decoding
turbojpeg = "1.4"
# Ensure base64 is available for decoding the frontend payload
base64 = "0.21"

```

---

## 🦀 Backend Implementation (`lib.rs`)

Replace your current `receive_video_frame_jpeg` command with this optimized version.

```rust
use turbojpeg::{Decompressor, PixelFormat};
use base64::{engine::general_purpose, Engine as _};

#[tauri::command]
pub fn receive_video_frame_jpeg(payload: String) -> Result<(), String> {
    let start_time = std::time::Instant::now();

    // 1. Decode Base64 payload to raw JPEG bytes
    let jpeg_bytes = general_purpose::STANDARD
        .decode(payload)
        .map_err(|e| format!("Base64 decode error: {}", e))?;
    let b64_time = start_time.elapsed().as_millis();

    // 2. Initialize TurboJPEG Decompressor
    let mut decompressor = Decompressor::new()
        .map_err(|e| format!("TurboJPEG init error: {}", e))?;

    // Read header to determine image dimensions
    let header = decompressor.read_header(&jpeg_bytes)
        .map_err(|e| format!("Header read error: {}", e))?;

    // 3. Prepare RGBA buffer
    // Width * Height * 4 (RGBA)
    let mut rgba_buffer = vec![0u8; header.width * header.height * 4];

    // 4. SIMD-Accelerated Decompression
    // Directly decompressing into our RGBA buffer
    decompressor.decompress_to_slice(
        &jpeg_bytes,
        &mut rgba_buffer,
        PixelFormat::RGBA
    ).map_err(|e| format!("Decompression error: {}", e))?;

    let decompress_time = start_time.elapsed().as_millis() - b64_time;

    // 5. Pass to your existing pipeline
    // Example: external_recorder.push_frame(rgba_buffer);

    // Profiling (Matches your current dev log style)
    if is_profile_frame() {
        println!(
            "[Backend-Turbo] Frame: b64_dec={}ms, turbo_dec={}ms, total={}ms ({}x{} @ {}KB)",
            b64_time,
            decompress_time,
            start_time.elapsed().as_millis(),
            header.width,
            header.height,
            jpeg_bytes.len() / 1024
        );
    }

    Ok(())
}

```

---

## ⚡ Performance Optimization Checklist

### Backend Bottlenecks

- **Avoid Vec Reallocation**: If your resolution is constant (e.g., 1080p), reuse the `rgba_buffer` by keeping it in a `Mutex` or `State` instead of allocating a new `vec!` every frame.
- **Scaling Factor**: If you ever need to downscale for previews, `turbojpeg` can do this **during** decompression (e.g., 1/2 or 1/4 scale) much faster than scaling after.

### Frontend Bottlenecks (Optional)

- **Web Workers**: If the browser's `toBlob()` is causing UI lag, move the canvas capture and base64 encoding into a Web Worker using `OffscreenCanvas`.
- **Binary Transfer**: To save another ~33% overhead, consider sending the frame as a `Uint8Array` (binary) instead of a Base64 string. Tauri's `invoke` supports binary arguments, which avoids the CPU-heavy Base64 encoding/decoding step.

---

## 🎯 Success Criteria

- [ ] **Decompress Time**: `< 15ms` for 1080p (down from 80ms+).
- [ ] **Frame Drops**: `< 5%` at 960x540 or higher.
- [ ] **CPU Usage**: Significant drop in backend process load.

Would you like me to provide the **Web Worker** script to handle the frontend capture logic asynchronously?
