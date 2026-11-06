# Linux Virtual Camera Enhancements: Dynamic 1080p/4K Resolutions & PipeWire Camera Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enhance NectarLink's Linux Virtual Camera by lifting the fixed 1280x720 restriction to support dynamic 720p, 1080p, and 4K UHD resolutions, while adding driverless PipeWire Camera (`Video/Source`) streaming support alongside `v4l2loopback`.

**Architecture:** 
- In `desktop/app/src/linux/vcam.rs`, replace hardcoded `OUT_W = 1280` / `OUT_H = 720` in `v4l2_worker_loop` with dynamic target resolution resolution derived from `crate::webcam::height()` / active stream dimensions.
- Support dynamic reallocation and `VIDIOC_S_FMT` renegotiation when user switches resolution (720p, 1080p, 4K) or when phone stream begins.
- Implement an optimized `bgrx_to_yuyv` conversion suitable for high-resolution 1080p and 4K frames.
- Introduce PipeWire Virtual Camera streaming (`desktop/app/src/linux/pipewire_vcam.rs`) that publishes a `media.class="Video/Source"` node into the active PipeWire session, providing rootless and driverless camera support across Wayland, Flatpak, and modern Linux apps.
- Provide automatic backend selection: use PipeWire when available, or fall back to `v4l2loopback` when kernel loopback device is configured.

**Tech Stack:** Rust (2024 edition), Linux V4L2 (`libc::ioctl`, `VIDIOC_S_FMT`, `v4l2loopback`), PipeWire 1.x (`pipewiresink`, `pw-cli`), Shared Memory (`nectarlink-vcam::SharedFrameMapping`).

---

## Global Constraints

- Must compile under `desktop/app` with Rust 2024 (`rust-version = "1.97"`).
- All code must respect `#![deny(unsafe_code)]` or strictly document SAFETY comments for required `libc` ioctls.
- Zero extra heavy dependencies: rely on existing workspace crates, standard OS tools (`pipewiresink` via GStreamer or pipewire subprocess, `/dev/video*`).
- Maintain full backward compatibility with existing tests and Windows/macOS abstraction layers (`desktop/app/src/win/vcam.rs` API parity).

---

## Review Focus

1. **Resolution switching safety**: Ensuring V4L2 device doesn't crash or return `EBUSY` when switching dimensions while open; graceful fallback if device rejects a mode.
2. **Buffer resizing & bounds**: Preventing out-of-bounds reads/writes in `bgrx_to_yuyv` when dimensions change.
3. **PipeWire process lifecycle**: Ensuring background PipeWire worker cleans up cleanly when webcam stops or desktop app exits (`Drop` implementation).
4. **Performance**: Minimizing allocations per frame during steady state at 1080p @ 30fps and 4K.
5. **Driverless fallback**: Seamless operation on systems without `v4l2loopback-dkms` installed.

---

## Tasks

### Task 1: Dynamic Resolution Support in `desktop/app/src/linux/vcam.rs`

- [ ] Add `target_dimensions(height: u32) -> (u32, u32)` helper mapping 720 -> (1280, 720), 1080 -> (1920, 1080), 2160 -> (3840, 2160).
- [ ] Update `v4l2_worker_loop` to dynamically track configured resolution:
  - Read desired height from `crate::webcam::height()`.
  - Re-run `configure_v4l2_yuyv` whenever target dimensions change.
  - Dynamically resize `bgrx` and `yuyv` buffers only on dimension change.
- [ ] Add unit tests verifying `target_dimensions` and buffer sizing logic for 720p, 1080p, and 4K.

### Task 2: Optimized Color Conversion for 1080p / 4K

- [ ] Review and optimize `bgrx_to_yuyv` algorithm in `desktop/app/src/linux/vcam.rs`:
  - Optimize memory layout and inner loop math for faster studio-swing BT.601 conversion.
  - Add unit tests for 1080p and 4K buffer size assertions and odd/even dimension edge cases.

### Task 3: PipeWire Virtual Camera Implementation (`pipewire_vcam.rs`)

- [ ] Create `desktop/app/src/linux/pipewire_vcam.rs`:
  - Implement detection function `is_pipewire_available() -> bool` (checks `pw-cli` or `$XDG_RUNTIME_DIR/pipewire-0` socket).
  - Implement `PipeWireCameraHandle` with background thread that feeds BGRX/YUYV frames from `SharedFrameMapping` into a PipeWire `Video/Source` node.
  - Implement clean shutdown via `AtomicBool` and `Drop`.
- [ ] Expose PipeWire camera through `desktop/app/src/linux/vcam.rs` / `desktop/app/src/linux/mod.rs`.

### Task 4: Unified Virtual Camera Handle & Backend Selection

- [ ] Update `VirtualCameraHandle::start()` in `vcam.rs`:
  - If PipeWire is available and preferred, start PipeWire camera; or start V4L2 worker if loopback device exists.
  - Support starting both or selecting based on device presence.
- [ ] Update documentation and run tests.
