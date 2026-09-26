# Global Tempo Control Project - Planning Stage

Act as an expert C++ engineer familiar with libsidplayfp.
Our goal is to implement a global tempo control that scales the playback speed without changing the audio pitch.

You will work in three separate stages:

1. RESEARCH
2. PLAN
3. IMPLEMENT

During the PLAN stage, you will use knowledge gained in the RESEARCH stage to formulate a comprehensive high-level strategy for attacking the tempo control problem. Start by reading RESEARCH.md (in the same folder as PLAN.md). 

Consult the libsidplayfp source code continually throughout the PLAN generation process. Become so familiar with libsidplayfp that you could rewrite it yourself. If you deem it necessary, break the problem into multiple checkpoints that can be verified before proceeding. Note that this stage does NOT involve code changes. You will make code changes according to the PLAN in the IMPLEMENT stage. Therefore, you must make sure the PLAN is detailed enough to guide code changes.

Write the plan document below this line.

---

## Detailed Implementation Plan: Global Tempo Control

### 1. Core Concept: Decoupled Clocking
To scale tempo without affecting pitch, we decouple the **System Clock** (C64 core: CPU, VIC, CIA) from the **SID Clock**. 

*   **System Clock (Logical Time)**: The C64 core (CPU, VIC, CIA) always runs at its nominal frequency (e.g., $1.011\text{MHz}$ for PAL). This represents the "logical" progress of the music. Interrupts and music updates happen at fixed logical intervals.
*   **SID Clock (Aural Time)**: The SID chip is clocked at a rate proportional to the system clock but scaled by the inverse of the tempo: `SID_Rate = System_Rate / Tempo`.
*   **Audio Output**: The host consumes samples at a fixed rate (e.g., $44.1\text{kHz}$). 

**Example**:
If $Tempo = 2.0$ (Double Speed):
1.  The C64 executes $2000$ cycles of logical time.
2.  The SID is clocked for only $2000 / 2.0 = 1000$ cycles.
3.  The SID produces half the normal number of samples for those $2000$ logical cycles.
4.  Since the host still pulls samples at $44.1\text{kHz}$, those $2000$ logical cycles are exhausted in half the real-world time.
5.  **Result**: Music plays twice as fast, but because the SID only saw $1000$ cycles of clocking, its internal oscillators generated waveforms at their original frequencies (preserving pitch).

### 2. Detailed Source File Changes

#### 2.1 Base Emulation Layer: `src/sidemu.h` and `src/sidemu.cpp`
We will centralize the scaling logic in the `sidemu` base class. This ensures all SID emulators behave consistently and avoids redundant logic.

**`src/sidemu.h` Changes**:
*   Add `double m_tempo` (default `1.0`).
*   Add `double m_sidTime` to accumulate fractional SID cycles.
*   Add `event_clock_t m_lastSystemTime` to track the last `eventScheduler` time.
*   Add `void setTempo(double tempo)` to update the scale factor.
*   Add `event_clock_t getDeltaCycles()`:
    ```cpp
    deltaSystem = eventScheduler->getTime(EVENT_CLOCK_PHI1) - m_lastSystemTime;
    m_lastSystemTime += deltaSystem;
    m_sidTime += deltaSystem / m_tempo;
    deltaSid = floor(m_sidTime);
    m_sidTime -= deltaSid;
    return deltaSid;
    ```
*   Add `void consumeDeltaCycles(event_clock_t remaining)` to put unconsumed cycles back into `m_sidTime`.

**`src/sidemu.cpp` Changes**:
*   Initialize new members in `lock(EventScheduler *scheduler)`.
*   Reset `m_lastSystemTime` and `m_sidTime` to $0$.

#### 2.2 ReSID Implementation: `src/builders/resid-builder/resid-emu.cpp`
The ReSID engine must be updated to use the scaled cycles.

**`src/builders/resid-builder/resid-emu.cpp` Changes**:
*   Modify `ReSID::clock()`:
    *   Call `getDeltaCycles()` to get the scaled delta.
    *   Pass this delta to `m_sid.clock()`.
    *   ReSID's `clock()` method returns unconsumed cycles by reference; pass these back to the base class via `consumeDeltaCycles()`.

#### 2.3 Mixer Orchestration: `src/mixer.h` and `src/mixer.cpp`
The `Mixer` acts as the bridge between the player and the individual SID chips.

**`src/mixer.h` Changes**:
*   Add `void setTempo(double tempo)`.

**`src/mixer.cpp` Changes**:
*   Implement `setTempo()`: Iterate through `m_chips` (vector of `sidemu*`) and call `chip->setTempo(tempo)` on each.

#### 2.4 Player Control: `src/player.h` and `src/player.cpp`
The `Player` manages the high-level emulation state.

**`src/player.h` Changes**:
*   Add `void setTempo(double tempo)`.

**`src/player.cpp` Changes**:
*   Implement `setTempo()`: Call `m_mixer.setTempo(tempo)`.
*   Ensure `m_cfg.tempo` is updated if we decide to keep it in config, but the primary mechanism should be the direct call to avoid a full reset in `config()`.

#### 2.5 Public API: `src/sidplayfp/sidplayfp.h` and `src/sidplayfp/sidplayfp.cpp`
Expose the functionality to users without requiring a configuration reset.

**`src/sidplayfp/sidplayfp.h` Changes**:
*   Add `void setTempo(double tempo)` to the `sidplayfp` class.

**`src/sidplayfp/sidplayfp.cpp` Changes**:
*   Implement `setTempo()`: Delegate to `sidplayer.setTempo(tempo)`.

### 3. Tempo-Invariant Time Reporting
The requirement is that `time()` and `timeMs()` report the position in the song, regardless of how fast it is playing.

*   **Analysis of `src/c64/c64.h`**:
    ```cpp
    uint_least32_t getTimeMs() const {
        return (eventScheduler.getTime(EVENT_CLOCK_PHI1) * 1000) / cpuFrequency;
    }
    ```
*   Since `eventScheduler.getTime()` returns the total number of **logical** cycles executed by the C64 core, and `cpuFrequency` is the **nominal** clock rate (e.g., $1011000$ for PAL), this calculation is already tempo-invariant.
*   **Result**: No changes are required in `c64` or `Player::timeMs()` to satisfy this requirement. The "logical" clock advances exactly as the music driver expects, while the "aural" clock (SID) is slowed down or sped up.

### 4. Implementation Checkpoints

1.  **Checkpoint 1: Core Plumbing (`sidemu`)**: Add members and the `getDeltaCycles` logic to the base class.
2.  **Checkpoint 2: ReSID Integration**: Update the ReSID builder to utilize the scaled cycle deltas.
3.  **Checkpoint 3: API Propagation**: Connect `sidplayfp` -> `Player` -> `Mixer` -> `sidemu`.
4.  **Checkpoint 4: Refinement**: Ensure precision is maintained using `double` for `m_sidTime` to prevent temporal drift.
