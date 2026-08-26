# Session Context: TRACKER power-saving cycle & telemetry-before-sleep

This file is a synthesis of an AI-assisted session on this repo, kept so any Copilot/agent
session (including in forks of this repo) can pick up the context quickly. Referenced from
`.github/copilot-instructions.md`.

## Topic

Deep-dive on the TRACKER/TAK_TRACKER power-saving sleep cycle in
[`src/modules/PositionModule.cpp`](../../src/modules/PositionModule.cpp), triggered from
`PositionModule::sendOurPosition(NodeNum dest, bool wantReplies, uint8_t channel)`.

## Sleep cycle recap

For a device with `config.device.role` in `{TRACKER, TAK_TRACKER}` and `config.power.is_power_saving == true`:

1. `sendOurPosition()` builds and sends the position packet via `service->sendToMesh(p, RX_SRC_LOCAL, true)`.
2. It then (as of this session) also sends a device telemetry packet — see "Change 2" below.
3. It sends a `ClientNotification` ("Sending position and sleeping for Ns interval in a moment").
4. It sets `sleepOnNextExecution = true` and calls `setIntervalFromNow(wakeMs)`, where `wakeMs` is
   how long to stay awake before actually sleeping (see "Change 1" below).
5. On the next `runOnce()` tick, since `sleepOnNextExecution` is true, it calls
   `doDeepSleep(nightyNightMs, false, false)` where `nightyNightMs` is
   `Default::getConfiguredOrDefaultMs(config.position.position_broadcast_secs)` — i.e. the device
   sleeps until it's next due to broadcast its position again.

## Change 1 — configurable pre-sleep wake window (done, verified)

**Before:** hardcoded `ONE_MINUTE_MS` (60s) delay between sending the position and going to deep sleep.

**After:** reuses the existing `config.power.min_wake_secs` proto field (already used the same way
in `src/PowerFSM.cpp`) via the standard helper:

```cpp
uint32_t wakeMs = Default::getConfiguredOrDefaultMs(config.power.min_wake_secs, default_min_wake_secs);
LOG_DEBUG("Start next execution in %ims, then sleep", wakeMs);
setIntervalFromNow(wakeMs);
```

`min_wake_secs` is documented in `protobufs/meshtastic/config.proto` (`PowerConfig.min_wake_secs`,
default 10s) as "how long to stay awake in no-BLE mode after handling a packet in light sleep" —
reusing it here lets one setting govern all "how long do we linger awake" behavior.

## Change 2 — send telemetry (battery, etc.) right before sleeping (done, verified)

**Request:** for the TRACKER role, send device telemetry (battery status in particular) right
after the position packet and before going to deep sleep, so a listener gets both position and
battery state in the same short "awake window" instead of waiting for the next telemetry interval
(which the tracker may never reach if `is_power_saving` keeps it mostly asleep).

**Implementation** (4 files):

- `src/modules/Telemetry/DeviceTelemetry.h`:
  - `sendTelemetry(NodeNum dest = NODENUM_BROADCAST, bool phoneOnly = false)` moved from
    `protected:` to `public:`.
  - Added `extern DeviceTelemetryModule *deviceTelemetryModule;` after the class, mirroring the
    existing singleton-pointer pattern used by `positionModule`, `nodeInfoModule`, etc.
- `src/modules/Telemetry/DeviceTelemetry.cpp`:
  - Added the global definition `DeviceTelemetryModule *deviceTelemetryModule;`.
- `src/modules/Modules.cpp`:
  - Changed `new DeviceTelemetryModule();` to `deviceTelemetryModule = new DeviceTelemetryModule();`
    inside the existing `#if HAS_TELEMETRY` guard.
- `src/modules/PositionModule.cpp`:
  - `#include "modules/Telemetry/DeviceTelemetry.h"` added.
  - Inside the TRACKER/TAK_TRACKER power-saving block in `sendOurPosition(NodeNum, bool, uint8_t)`,
    right after `service->sendToMesh(p, RX_SRC_LOCAL, true)` and before the client notification /
    `setIntervalFromNow(wakeMs)`:
    ```cpp
    #if HAS_TELEMETRY
        // Report battery/device state alongside position since we're about to sleep and won't be asked again soon
        if (deviceTelemetryModule && moduleConfig.telemetry.device_telemetry_enabled)
            deviceTelemetryModule->sendTelemetry();
    #endif
    ```
    Guarded by the `HAS_TELEMETRY` compile flag (matches how the module itself is conditionally
    instantiated), a null-check on the singleton, and the existing user-facing config toggle
    `moduleConfig.telemetry.device_telemetry_enabled` so this respects the user's telemetry
    preference instead of forcing telemetry sends.

**Build verification:** `pio run -e tbeam` (the workspace's default build task) compiled and
linked successfully after these 4 changes (RAM 8.5%, Flash 89.6% on tbeam). A follow-up attempt to
also build `heltec-wireless-tracker` failed both times with
`xtensa-esp32s3-elf-g++: error: CreateProcess: No such file or directory` — this is a known,
pre-existing local toolchain/AV issue on this machine (see
`/memories/repo/platformio-toolchain-cc1plus-missing.md` in repo memory), **not** related to
these code changes. Full `heltec-wireless-tracker` build verification was deferred to GitLab CI
per the user's decision.

## Reliability semantics clarified during this session (no code change)

Two independent mechanisms, easy to conflate:

- **`want_ack` (bool on `meshtastic_MeshPacket`)** — mesh-level reliability. When true,
  `ReliableRouter`/`NextHopRouter` automatically retransmits up to `NUM_RELIABLE_RETX` (3) times
  if no ACK is seen. Position packets from TRACKER/TAK_TRACKER set
  `p->priority = meshtastic_MeshPacket_Priority_RELIABLE` but this is a *priority* field, not the
  same as `want_ack` — check the current code before assuming retransmission behavior.
- **`decoded.want_response` (bool)** — application-level "please reply" flag, handled generically
  in `MeshModule.cpp`. If nobody replies, the receiver-side generic handler sends a NAK
  (`NO_RESPONSE`) back to the *original sender*; this is **not** read by the sender to trigger its
  own resend. In `PositionModule::sendOurPosition`, `want_response` is forced to `false` for the
  TRACKER role (`p->decoded.want_response = config.device.role == ...TRACKER ? false : wantReplies;`).

**Conclusion reached:** there is no automatic "renew the position send if nobody answered" logic.
The only thing that resembles a retry is mesh-level `want_ack` retransmission (if set), and/or the
next periodic position broadcast (`runOnce()` cycle / next wake from deep sleep).

## Power config parameter clarified during this session (no code change)

`config.power.on_battery_shutdown_after_secs` (UI label: "Arrêt en cas de perte d'alimentation" /
"Shutdown on power loss"):

- Defined in `protobufs/meshtastic/config.proto` (`PowerConfig.on_battery_shutdown_after_secs`,
  field 2): "If non-zero, the device will fully power off this many seconds after external power
  is removed."
- "Power loss" = external power (USB / DC input) unplugged, i.e. `powerStatus->getHasUSB()`
  becomes false — the device is now running on its internal battery only.
- Checked in `src/PowerFSMThread.h::runOnce()`: while `getHasUSB()` is true, `timeLastPowered` is
  continuously refreshed; once USB is gone, if `millis() - timeLastPowered` exceeds the configured
  duration, `powerFSM.trigger(EVENT_SHUTDOWN)` fires.
- "Shutdown" here means a **full power-off**, not sleep — `EVENT_SHUTDOWN` is documented in
  `src/PowerFSM.h` as "force a full shutdown now (not just sleep)" and transitions the state
  machine to `stateSHUTDOWN` from any state (`ON`, `LS`, `NB`, `DARK`, `SERIAL`, `BOOT`).
- `src/modules/AdminModule.cpp` enforces a 30s floor: values between 1 and 29 get clamped up to 30.
- `0` disables the feature entirely (device stays on battery indefinitely until it dies naturally
  or the user powers it off manually).

## Files touched (final state at end of session)

- `src/modules/PositionModule.cpp` — wakeMs fix + telemetry-before-sleep call
- `src/modules/Telemetry/DeviceTelemetry.h` — public `sendTelemetry()` + extern global pointer
- `src/modules/Telemetry/DeviceTelemetry.cpp` — global pointer definition
- `src/modules/Modules.cpp` — assign the global pointer at instantiation

No other pending work from this session; all requested changes were implemented and at least
partially build-verified (tbeam full success; heltec-wireless-tracker build deferred to CI by the
user's own choice, unrelated local toolchain issue only).
