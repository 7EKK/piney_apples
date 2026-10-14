---
number: 408
title: Issue #65's panic is a lost GPU device: the GS stops drawing until the window makes another, and a frame's vertices go in buffers within the device's size limit
date: 2026-10-14
area: render, engine, test
files: crates/piney-gs/src/lib.rs, crates/piney-gs/src/shadow.rs, crates/piney-game/src/main.rs
---

# 408. Issue #65's panic is a lost GPU device: the GS stops drawing until the window makes another, and a frame's vertices go in buffers within the device's size limit

Issue #65 (ThreePendant, build 85f79e4, Windows): standing in Carmina
Gadelica for a long time ended in a panic in wgpu's `create_buffer_init`:
"Buffer with 'gs vertices' label is invalid". The message says the GPU
device was lost. It does not say the frame had grown too big.

## What the message means

- wgpu 30 sorts every error by kind (`webgpu_error_type`). A validation
  or out-of-memory error goes to the device's error sink, and the default
  handler panics with "wgpu error: ...". A `DeviceError::Lost` goes
  nowhere: `handle_error_inner` returns for `ErrorType::DeviceLost`
  (wgpu 30.0.1 `backend/wgpu_core.rs`).
- So `create_buffer` fails quietly only when the device is lost. Then
  `create_buffer_init` maps the invalid buffer, and its `expect` panics
  with the player's text (`util/device.rs:72`).
- A buffer over `max_buffer_size` (256 MiB by default) panics earlier,
  inside `create_buffer`: "Buffer size N is greater than the maximum
  buffer size". A probe on wgpu 30.0.1 gave exactly these two messages:
  a 256 MiB + 56 byte buffer gives the second, and a destroyed device
  gives the player's.
- The frame was small. Carmina Gadelica draws 60,000 to 79,000 vertices
  a frame (3.4 to 4.4 MB); a buffer would need 4.8 million. The next
  entry checks that nothing grows while Kite idles.
- On Windows, a device is lost when D3D12 reports
  `DXGI_ERROR_DEVICE_REMOVED` or `_RESET`, or Vulkan reports
  `VK_ERROR_DEVICE_LOST`. A driver reset (TDR), a driver update, or the
  display or system going to sleep can each cause one. Nothing in the
  port raised it.

## Cause

`piney-gs` built its vertex buffer with `create_buffer_init` each frame,
and nothing in the port watched for the device's loss. The first frame
after a loss panicked. The shadow packets' buffers were made the same way.
A write to an invalid buffer (`Queue::write_buffer`) is a validation
error, so it would also have panicked under the default handler.

## Fix

- **`watch_loss`** (piney-gs) sets a flag from the device's lost callback.
  It also installs an error handler. Once the flag is set, errors are
  logged, not fatal. Out of memory counts as a loss. Any other error
  still panics, as wgpu's own handler does, so a real validation bug
  still shows in the tests.
- **`Gs::render`** draws nothing while `Gs::lost()`. **`Gs::read_back`**
  returns black.
- **`Gs::renewed(device, queue)`** puts the same GS on a new device. It
  keeps the assets and the scale; everything on the GPU is made again.
- **The window** (`App::renew_lost_device`, piney-game main.rs) checks at
  each redraw. On a loss it drops the surface and makes the surface, the
  adapter, the device and the presenter again (`open_display`, the
  window's first set-up moved into a method). Then it renews the GS, and
  the game carries on. If no device can be made, the window logs why and
  exits.
- **Buffers.** No buffer is mapped at creation any more. The frame's
  vertices go into buffers kept from frame to frame (`vertex_buffers`)
  and are written through the queue. Each buffer holds at most `chunk`
  vertices: the whole triangles that fit the device's `max_buffer_size`.
  Draws are cut at the seams (`draw_chunked`). Every draw starts on a
  triangle, so a cut never splits one. A frame of any size fits the
  device's limits. The adapter's larger limit is not requested, because
  the chunks make it unnecessary and a very large buffer risks an
  out-of-memory error. The shadow packets' buffers are written through
  the queue too (`filled`).

## Checked

- **`a_frame_past_the_buffer_limit_draws_whole`** (piney-gs) draws a
  27,000-vertex frame on a device whose buffers hold 1 MiB (18,723
  vertices). The frame is 3000 sprites and a 3000-triangle prim that
  crosses the seam. Two buffers are made, the picture is the
  single-buffer device's byte for byte, and triangles 240 and 241 on
  either side of the seam are their colours. With the chunk unlimited,
  the test fails.
- **`a_lost_device_stops_the_drawing_and_a_new_one_resumes_it`**
  (piney-gs) destroys the device and polls it.
  - `Gs::lost()` is set.
  - wgpu's `create_buffer_init` on the destroyed device panics with the
    player's message, which reproduces the panic.
  - The GS's own buffer writes and a render go through without a panic,
    and the read-back is black.
  - Renewed on a fresh device, the GS keeps its scale of 2 and draws the
    frame again.
- piney-gs's tests, piney-game's suite (four threads), clippy, fmt,
  `cairns check` and `tools/docs.py check` pass.

**Still unknown:** two questions need the player's machine, not the repo.
What lost the device: Windows' event log at the crash time (Event Viewer,
System, `Display` / `nvlddmkm` / `amdkmdag` / `dxgkrnl` events) would
say, and this build's log line "the GPU device was lost" gives wgpu's
reason. And that the window's renewal works on a real loss: tested
headless by destroying the device, but no device can be lost on demand
here in a window. A retest on the player's Windows machine would show it.
