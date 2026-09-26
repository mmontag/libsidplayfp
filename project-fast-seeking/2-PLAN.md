# Fast Seeking Project - Radical Implementation Plan

## 1. Objective
Achieve 500x-1000x real-time seeking speed by implementing a "Headless Radical Seek" mode. This mode prioritizes instruction throughput over cycle-exact fidelity, accepting
stale chip states (envelopes, filters) during the seek process.

## 2. Strategy

### 2.1 Backward Seeking
- **Clean Reset**: Call `initialise()` (which handles `m_c64.reset()`, memory re-loading, and driver installation) to ensure a perfectly clean state at Time 0.

### 2.2 Radical Headless Seek Mode (The "Warp" Drive)

#### 2.2.1 reSID Backend Optimization (`src/builders/resid-builder`)
- **Bypass Clocking**: In `resid-emu.cpp`, if seeking is active, `sidemu::clock()` will become a NOP.
- **Immediate Writes**: Ensure `sidemu::write()` calls `reSID::SID::write()` which, in `SAMPLE_FAST` mode (or forced during seek), bypasses the 8580 pipeline delay for
  immediate register updates.
- **Post-Seek Sync**: After the seek loop finishes, a single call to `reSID::SID::clock(delta_t)` with a `nullptr` buffer will be made to bring the SID internal clocks up to
  the current system time without generating any intermediate audio samples.

#### 2.2.2 Fast CPU (Predictive Batching)
- **The Loop**: Inside `MOS6510::eventWithoutSteals`, if `m_isSeeking` is true, implement a predictive batching loop.
- **Logic**:
  1. Check `eventScheduler.nextEventTime()`.
  2. If the next event (e.g., CIA timer) is $N$ cycles away, the CPU can execute instructions in a tight `while` loop for $N$ cycles without returning control to the
     `EventScheduler`.
  3. This eliminates thousands of linked-list pops/pushes in the scheduler.
- **Loose Interrupts**: Only check for interrupts (IRQ/NMI) at the end of this batch or at instruction boundaries.

#### 2.2.3 VIC-II Lite
- **Force BA High**: During seek, `MOS656X` will always report `BA=true`. This prevents the CPU from being stalled by "bad lines" or sprite DMA, which are irrelevant for SID
  playback logic but normally consume ~10-25% of CPU cycles.
- **Skip Clock Logic**: `MOS656X::event()` will skip the complex `clockPAL/NTSC` state machine and only update the bare minimum: `rasterY` counter and `rasterYIRQ` triggers.

#### 2.2.4 Mixer/Player Integration
- **Zero-Latency Mixer**: `Mixer::doMix` and `Mixer::clockChips` will return immediately.
- **Target Logic**: `Player::seek(ms)` will drive the `m_c64.clock()` loop.

## 3. Detailed Implementation Steps

### 3.1 `src/player.h/cpp`
- Add `bool m_isSeeking`.
- Implement `void seek(uint_least32_t ms)`:
  - If target `ms < current_ms`, call `initialise()`.
  - Set `m_isSeeking = true`.
  - Loop `m_c64.clock()` until `timeMs() >= ms`.
  - Set `m_isSeeking = false`.
  - Call `m_mixer.clockChips()` once to synchronize SIDs to the final time.

### 3.2 `src/c64/CPU/mos6510.cpp`
- Modify `eventWithoutSteals` to implement the predictive loop.
- Example:
  ```
  if (m_isSeeking) {
    event_clock_t nextEvent = eventScheduler.nextEventTime();
    while (currentTime < nextEvent) {
      execute_one_cycle(); // tight internal loop
      currentTime++;
    }
  }
  ```

### 3.3 `src/builders/resid-builder/resid-emu.cpp`
- Update `residemu::write` to ensure immediate register application.
- Update `residemu::clock` to check `player->isSeeking()`.

## 4. Performance Goals
- **Real-time**: 1 second of music takes 1 second to emulate.
- **Current Headless**: ~20x-50x speed.
- **Radical Plan**: >500x speed (Seeking to 5:00 in < 0.6 seconds).
