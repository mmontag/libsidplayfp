# Global Tempo Control Project - Research Stage

Act as a C++ engineer familiar with libsidplayfp. 
Our goal is to implement a global tempo control that scales the playback speed without changing the audio pitch.

You will work in three separate stages:

1. RESEARCH
2. PLAN
3. IMPLEMENT

During the RESEARCH stage, you will gather prerequisite knowledge about SID playback and about the structure of the libsidplayfp library in particular, and compile it into a research document that will be used to PLAN a solution. Include details like the basic emulation architecture, relevant timing mechanisms, specific pointers to key files, and the public API of libsidplayfp.  This document should serve as a map and reference guide for the PLANNING stage.

Append the research document below this line.

---

## Research Document: Global Tempo Control for libsidplayfp

### 1. Library Architecture & High-Level Design

The `libsidplayfp` library is a cycle-accurate Commodore 64 and SID chip emulation engine. Its architecture is layered to decouple the host-side audio production from the guest-side (emulated) machine state.

*   **Public API (`sidplayfp`)**: The primary interface for host applications.
*   **Internal Player (`libsidplayfp::Player`)**: Manages the life cycle of the emulation, handles state transitions (PLAYING, STOPPED, etc.), and orchestrates the interaction between the C64 core and the audio mixer.
*   **C64 Core (`libsidplayfp::c64`)**: A container for the emulated C64 system components. It aggregates the CPU (6510), VIC-II, CIA timers, and the memory map (MMU).
*   **Event Scheduler (`libsidplayfp::EventScheduler`)**: The heart of the library's timing. It maintains a priority queue of `Event` objects sorted by their `triggerTime`. The entire emulation advances by popping the next event from this queue and advancing the "system time" to that event's trigger time.
*   **Mixer (`libsidplayfp::Mixer`)**: Responsible for collecting digital audio samples from the SID emulators and performing final mixing, volume scaling, and optional downsampling/fast-forwarding.

### 2. The Emulation Loop: Host vs. Guest Time

A common misconception is that the library's `play()` method is synchronized with guest-side interrupts like the VIC vertical blank. In reality, they are decoupled:

#### 2.1 The Host-Side `play()` Method
The top-level `sidplayfp::play(short *buffer, uint_least32_t count)` method is a request from the host to produce a specific number of audio samples. 
*   It does **not** represent a fixed amount of guest time (like a frame). Instead, it runs the emulation for as many cycles as necessary to generate the requested `count` of samples.
*   Internally, `Player::play` enters a loop:
    ```cpp
    while (m_mixer.notFinished()) {
        if (!m_mixer.wait())
            run(CYCLES); // Advances C64 by a fixed chunk of cycles (e.g., 3000)
        m_mixer.clockChips();
        m_mixer.doMix();
    }
    ```
*   `run(CYCLES)` simply calls `m_c64.clock()` repeatedly. Each `clock()` call executes exactly one event from the `EventScheduler`.

#### 2.2 Guest-Side Timing: The Event Scheduler
The guest (C64) system time is measured in half-cycles (Phi1/Phi2). 
*   The `EventScheduler` tracks `currentTime` in these units.
*   Hardware components (CPU, CIA, VIC) schedule themselves for future execution. For example, the CPU schedules its next cycle's work, and the CIA timers schedule an event for when they will underflow.
*   **VIC-II Interrupts**: The VIC-II schedules an event for the start of the Vertical Blank (VBlank). When this event fires, it may trigger a CPU IRQ if the guest software has enabled it.
*   **CIA Timers**: These are independent of the VIC. They can be programmed to fire at arbitrary cycle counts, allowing for "multi-speed" music players that update the SID more frequently than the $50/60\text{Hz}$ VBlank rate.

### 3. SID Playback Fundamentals

SID music is essentially a program running on the emulated 6510 CPU.
1.  **Initialization**: The `.sid` file header specifies an `init` address. The library loads the machine code into the emulated RAM and calls this routine. It typically sets up the music driver's internal state and configures the timing source (VIC or CIA).
2.  **The Play Routine**: The header also specifies a `play` address. This routine is responsible for updating the SID chip's registers (registers `$D400`–`$D41C`) to change the sound being produced.
3.  **Triggering the Play Routine**: In most tunes, the guest OS or a small loader stub sets up an interrupt (IRQ) that calls the `play` routine. 
    *   If tied to the **VIC-II**, the music updates once per video frame ($50/60\text{Hz}$).
    *   If tied to a **CIA Timer**, the music can update at much higher frequencies, enabling smoother transitions and more complex sounds.

### 4. ReSID Backend & Cycle-Accurate Sound

This project utilizes the **ReSID** (and ReSIDfp) backends. These are highly accurate emulators that model the SID chip at the cycle level.
*   **Clocking the SID**: ReSID is not advanced cycle-by-cycle by the `EventScheduler` directly. Instead, it is clocked "on-demand" or at the end of a cycle chunk.
*   **Access Tracking**: When the CPU writes to a SID register, the ReSID emulator is first brought up to the current system time (clocked) to ensure that any pending internal state changes (like envelope transitions) occur before the new register value takes effect.
*   **Sample Generation**: ReSID generates audio samples based on the number of guest cycles that have passed. The ratio between the SID's internal $1\text{MHz}$ clock and the host's output sample rate (e.g., $44.1\text{kHz}$) determines how many samples are produced per guest cycle.

### 5. The Tempo vs. Pitch Problem Space

The goal of this project is to implement a global tempo control that scales the speed of the music without altering the pitch of the SID's hardware oscillators.

#### 5.1 Defining "Global Tempo"
In the context of `libsidplayfp`, "Global Tempo" means scaling the speed at which the **guest C64 system** advances relative to **host real time**.
*   If we want to double the tempo (2.0x), the guest CPU should execute twice as many cycles in the same amount of host time.
*   This means that the VIC and CIA interrupts will fire twice as often, causing the guest music driver to call the `play` routine twice as often.

#### 5.2 Pitch Preservation
The pitch of a SID oscillator is determined by the value in its frequency registers and the frequency of the clock signal driving the chip.
*   To keep the pitch constant while increasing tempo, the SID chip must still "perceive" that it is being clocked at its nominal rate (e.g., $1.011\text{MHz}$ for PAL), even if the CPU is advancing faster.

#### 5.3 Digital Samples ("Digis") and Overclocking
Digital audio on the C64 is typically "CPU-driven." The CPU must manually update the SID's volume register at a high frequency to play back a sample.
*   If we "overclock" the CPU to increase tempo, the CPU will execute the sample-playback code faster.
*   This will naturally result in the digital samples playing at a higher pitch.
*   **Accepted Behavior**: This pitch shift for "digis" is considered the physically correct behavior for an accelerated C64 and is an accepted side effect of this implementation.

### 6. Existing Speed Control Mechanisms

#### 6.1 Fast Forward
The library currently implements a `fastForward` factor in the `Mixer`.
*   This is a **pitch-shifting** speedup.
*   It works by having the SID generate its normal amount of samples, but then the `Mixer` downsamples/averages them (e.g., if FF is 2, it might average every 2 samples into 1).
*   This is akin to speeding up a tape recorder; both tempo and pitch increase.

#### 6.2 Cycle Stepping
The `Player::play(unsigned int cycles)` method allows the host to run the emulation for a specific number of cycles. While this gives the host control over the *rate* of emulation, it doesn't provide a way to decouple the CPU speed from the SID frequency for pitch-invariant tempo control.

### 7. Key Technical Challenges

1.  **Event Scheduling Consistency**: Scaling the `EventScheduler`'s time might break the relative timing between components if not handled carefully.
2.  **Mixer Synchronization**: The `Mixer` expects a certain number of samples from the SID chips based on the elapsed cycles. If the SID is clocked at a different rate than the rest of the system, the sample counts will no longer match the expected buffer sizes.
3.  **API Integration**: A new way to set the tempo must be added to the `sidplayfp` class and propagated down to the `Player`, `c64`, and eventually the `sidemu` components.
