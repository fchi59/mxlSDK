<!--
SPDX-FileCopyrightText: 2025 Contributors to the Media eXchange Layer project.
SPDX-License-Identifier: Apache-2.0
-->

# MXL Sink (mxlsink) Internals & Clock Management

This document provides a detailed, technical explanation of the internal architecture and clock management mechanisms of the `mxlsink` GStreamer plugin.

## 1. Internal Architecture & Data Flow

The `mxlsink` plugin acts as a bridge between the GStreamer pipeline and the Media eXchange Layer (MXL). It receives GStreamer buffers (audio or video), translates their presentation times to MXL's absolute clock, and writes the media payloads into MXL's shared memory ringbuffers (domains).

### 1.1 Structural Overview

```mermaid
graph TD
    A[GStreamer Upstream Pipeline] -->|GstBuffer| B(mxlsink `render()` or `video()`)
    B --> C{Clock Synchronization}
    C -->|Calculate MXL Index| D[MXL State & Writer]
    D --> E[(MXL Shared Memory Ringbuffer)]

    subgraph mxlsink Plugin
        B
        C
        D
    end
```

### 1.2 The Rendering Loop

When an upstream element pushes a buffer to `mxlsink`, the plugin invokes its specific rendering logic (`render_video.rs` or `render_audio.rs`). The process involves:
1. **Time Mapping:** Converting the GStreamer buffer's Presentation Timestamp (PTS) to the corresponding absolute MXL time.
2. **Index Resolution:** Calculating the exact grain index (or sample index) in the MXL ringbuffer based on the MXL time and the media format's rate.
3. **Memory Access & Copy:** Requesting a lock/access to the specific grain index (`open_grain`), copying the raw media payload into the ringbuffer, and committing the change (`commit`).

## 2. Clock Management & Synchronization

The most complex and critical aspect of `mxlsink` is reconciling two fundamentally different timing systems:
- **MXL Clock (`mxl_now`):** An absolute, monotonically increasing clock mapped to the domain (typically synchronized via PTP, representing nanoseconds since the Unix epoch).
- **GStreamer Clock (`gst_now` & `PTS`):** A relative running time that starts at `0` when the pipeline enters the `PLAYING` state.

### 2.1 The Initial Offset (Anchoring the Clocks)

To write a buffer into MXL, the plugin must translate the relative GStreamer time to the absolute MXL time. This is done by capturing a one-time "offset" at the very beginning of the stream.

**The Crucial Distinction:**
The offset must **not** be calculated based on the GStreamer pipeline's current running time (`gst_now`). Because upstream elements (like a source reading from a socket or `videoconvert`) take time to initialize and process the first frame, `gst_now` might already be `400ms` when the first buffer arrives.

Instead, the offset is calculated against the **Presentation Timestamp (PTS)** of the first arriving buffer:

```rust
// The offset links the absolute MXL time to the relative buffer PTS
let mxl_to_gst_offset = mxl_time_now - buffer_pts;
```

This guarantees that the "instant 0" of the GStreamer stream aligns perfectly with the "current instant" of MXL, effectively negating any startup latency introduced by upstream processing.

### 2.2 Applying the Offset

For the first buffer, and every subsequent buffer in the stream, the target MXL write time is calculated as:

```rust
let mxl_pts = current_buffer_pts + mxl_to_gst_offset;
```

This resulting `mxl_pts` is then converted to an MXL grain index. If the pipeline operates smoothly, this calculation ensures that the media is written exactly on time, resulting in an observed MXL latency of `0` grains.

## 3. Real-World Handling: Drift, Jitter, and Discontinuities

In a live, continuous streaming environment, perfect synchronization is rarely maintained indefinitely due to hardware clock drift and network jitter. `mxlsink` implements several safety mechanisms to handle these anomalies.

### 3.1 Preventing Negative Latency (Writing into the Future)

If the upstream GStreamer pipeline is slightly faster than real-time (or experiences minor jitter causing a frame to arrive a millisecond early), the calculated index might point to a grain in the "future" compared to MXL's current index.

To prevent writing ahead of the clock (which can cause underflows or chaotic latency readings), the plugin clamps the target index:

```rust
if calculated_index > current_mxl_index {
    // Clamp to the present moment to prevent negative latency
    target_index = current_mxl_index;
}
```

### 3.2 Hardware Clock Drift Auto-Correction

If GStreamer's local hardware clock ticks slightly slower than MXL's PTP clock, the GStreamer stream will gradually fall behind. Over hours or days, the calculated target index will slowly drift into the past, increasing the observed latency.

To combat this, `mxlsink` monitors the drift. If the calculated index falls behind the current MXL time by an unacceptable margin (e.g., more than 2 frames), it triggers an auto-correction:

1. The plugin detects the excessive delay.
2. It completely invalidates the initial offset (`state.initial_time = None`).
3. It forces the current frame to be written at the exact current MXL index (resetting latency to 0).
4. On the next frame, a brand new `mxl_to_gst_offset` is calculated, perfectly resynchronizing the two clocks.

### 3.3 Discontinuity & Looping Handling

If the upstream source loops (e.g., a test file reaching the end and restarting) or experiences a severe network drop, the GStreamer PTS will reset to `0` or jump significantly.

GStreamer flags these events using the `GST_BUFFER_FLAG_DISCONT` flag. `mxlsink` listens for this flag. Upon detecting a discontinuity, it immediately discards the established clock offset:

```rust
if buffer.flags().contains(gst::BufferFlags::DISCONT) {
    state.initial_time = None;
}
```

This ensures that the plugin seamlessly adapts to looped sources or restarting streams without writing into the extreme past.
