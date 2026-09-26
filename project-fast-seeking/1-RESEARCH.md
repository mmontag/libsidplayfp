# Fast Seeking Project - Research and Planning Stage

Act as an expert C++ engineer familiar with libsidplayfp.
Our goal is to improve seeking performance for use in an interactive music player host application.

You will work in separate stages:

1. RESEARCH
2. PLAN
3. IMPLEMENT

During the RESEARCH stage, you will gather prerequisite knowledge about SID playback and about the structure of the libsidplayfp library in particular, and compile it into a research document that will be used to PLAN a solution. Include details like the basic emulation architecture, relevant timing mechanisms, specific pointers to key files, and the public API of libsidplayfp.  This document should serve as a map and reference guide for the PLANNING stage.

Note that the "fastForward" mode doesn't help to reduce CPU cycles at all.

Write the research document below this line.

---

## Research Document: libsidplayfp Architecture and Seeking Performance

### 1. Basic Emulation Architecture

`libsidplayfp` is a cycle-exact (or nearly cycle-exact) Commodore 64 emulator specialized for SID music playback. Its architecture is event-driven, centered around a main timing engine that synchronizes various hardware components.

- **Event-Driven Core**: The library uses an `EventScheduler` to manage all timed actions (CPU instructions, VIC-II cycles, CIA timer decrements, etc.).
- **Component Wrapper**: The `c64` class aggregates all emulated chips (MOS6510 CPU, MOS656x VIC-II, MOS6526 CIA, SID, MMU).
- **Player Layer**: The `Player` class (private) and `sidplayfp` class (public API) drive the `c64` emulation loop and manage the audio `Mixer`.
- **SID Emulation**: SID chips are emulated via "builders" (e.g., reSID, reSIDfp). These are clocked periodically to generate audio samples based on the elapsed system cycles.

### 2. Timing Mechanisms

- **EventScheduler**: Maintains a priority queue of `Event` objects. Each event has a `triggerTime` measured in half-cycles (PHI1/PHI2).
  - `PHI1`: Auxiliary chip activity (VIC, CIA).
  - `PHI2`: CPU activity.
- **System Clock**: The `currentTime` in the scheduler advances as events are processed. The CPU frequency depends on the C64 model (PAL vs NTSC).
- **SID Synchronization**: SID chips stay in sync with the system clock by calculating "delta cycles" (cycles since last SID clock) using `eventScheduler.getTime()`.

### 3. Key Files and Classes

- `src/sidplayfp/sidplayfp.h`: The public API entry point.
- `src/player.h/cpp`: The internal engine that drives the emulation loop.
- `src/c64/c64.h/cpp`: The C64 system container.
- `src/EventScheduler.h`: The core timing mechanism.
- `src/mixer.h/cpp`: Handles audio buffering and mixing from multiple SID chips.
- `src/builders/resid-builder/resid-emu.cpp`: Implementation of the reSID engine wrapper.
- `src/builders/resid-builder/resid/sid.cc`: The actual SID emulation logic (reSID).

### 4. Public API Summary

- `sidplayfp::play(short *buffer, uint_least32_t count)`: Main playback method. Advances emulation and fills the buffer with mixed audio.
- `sidplayfp::play(unsigned int cycles)`: Advances emulation for a specific number of cycles and returns generated sample count.
- `sidplayfp::timeMs()`: Returns current playback time in milliseconds.
- `sidplayfp::fastForward(unsigned int percent)`: Sets a speed factor. **Critically**, this is implemented in the `Mixer` using a boxcar filter that averages samples; it does NOT skip emulation cycles and actually adds slight CPU overhead.

### 5. Seeking Performance Analysis

Currently, seeking to a specific time requires calling `play()` repeatedly. Even with `fastForward` set to a high value, the following bottlenecks exist:

- **Full Emulation**: Every single hardware event must still be processed via `m_c64.clock()`. This is necessary as SID music is usually a program running on the C64 CPU.
- **Audio Generation**: In current `play()` calls, SID chips are clocked to generate audio samples, which are then mixed or discarded. Generating samples (especially with high-quality resampling) is the most expensive part of the process.
- **Mixer Overhead**: The `Mixer` still performs logic to handle buffers, even if they are ultimately ignored or averaged out.

### 6. Opportunities for Fast Seeking

- **Silent Clocking**: The reSID engine (and others) supports a "silent" mode when the output buffer is `nullptr`. In `SID::clock(cycles, nullptr, ...)`, it skips all sample generation, interpolation, and filtering logic, only updating the internal state (envelopes, oscillators, etc.).
- **Bypassing Mixer**: A dedicated seek/fast-emulation loop can bypass the `Mixer::doMix()` and `Mixer::clockChips()` overhead by calling `m_c64.clock()` directly and only periodically updating the SID state silently.
- **Cycle Skipping**: While skipping C64 cycles is generally not possible without breaking the music program's state, running the emulation "headless" (no audio output) provides a significant speedup (typically 5x-10x faster than real-time playback).

### 7. Brainstormed Ideas for Further Optimization

- **CPU "Fast-Path"**: Implementing a less accurate but faster CPU execution mode for seeking. This could involve executing instructions in a single step rather than multiple sub-cycle events.
- **Selective Component Disabling**: If a tune doesn't use the VIC-II for timing (raster interrupts), its emulation could be "hibernated" during seeking to reduce the event load on the `EventScheduler`.
- **Idle Loop Detection**: Detecting and skipping CPU idle loops that don't perform I/O or SID writes.
- **State Snapshots**: Periodically saving the entire machine state to allow near-instant seeking to previously visited or cached points.
- **Batch Event Processing**: Advancing the `EventScheduler` in larger time steps during "headless" emulation to reduce the overhead of event management.
