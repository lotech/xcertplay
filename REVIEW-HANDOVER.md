# Code review handover: lotech/xcertplay `binnacle` fork changes

For: the Claude session doing the main work on this repo, which will make the fixes.
From: a review session, 2026-11-01. Nothing below has been changed in the code yet.

## Scope and state

- **Repo:** `lotech/xcertplay`. **Branch reviewed:** `binnacle` at `45c01bd`. This is the repo's
  default branch, and fork PRs #1 and #3–#7 were merged into it.
- **Base:** `master` at `1298fb3` is plain upstream shilapi/xcertplay 1.4.2, with no fork work.
- **Diff reviewed:** `git diff origin/master..origin/binnacle`. That's 33 commits and about
  1,000 lines across 25 files. All line numbers below are for `binnacle` at `45c01bd`.
- **Not verified:** nothing was built or run. The review environment had no Android SDK. The
  fork has never had a GitHub Actions run on any branch, so
  `./gradlew :shared:testDebugUnitTest` and the lint tasks in `.github/workflows/android.yml`
  have never run on this code. Run them locally first, before and after the fixes.

Path shorthand: `S/` = `shared/src/main/java/com/shilapi/xcertplay/`,
`T/` = `shared/src/test/java/com/shilapi/xcertplay/`,
`C/` = `common/src/main/java/com/shilapi/xcertplay/`.

---

## P1: confirmed bugs (fix first)

### 1. The echo suppressor counts not-yet-heard playback as active right away

- **Where:** `S/media/CallEchoSuppressor.kt:39-46` (`FarEndActivity.recentMaxDb`). The input
  comes from `S/media/AndroidMediaSink.kt:1160`.
- **What happens:**
  - Commit 3dc1795 stamps each playback chunk with when it will be heard:
    `System.currentTimeMillis() + queuedMs`, which can be up to 1.5 s in the future.
  - `recentMaxDb` keeps an entry when `nowMs - times[i] <= windowMs`. For a future timestamp
    the difference is negative, so the check always passes.
  - So each chunk counts as "far end active" from the moment it's **written** until 600 ms
    after it's **heard**.
- **Effect:**
  - The call mic is muted up to about 1.5 s before the caller's audio actually plays.
  - The fix was meant to shift the suppression window later. Instead it only lengthens it, so
    the half-duplex suppression is more aggressive than before that commit.
- **Fix:** count an entry only when `times[i] <= nowMs && nowMs - times[i] <= windowMs`.
- **Test:** none exists for `FarEndActivity`. Add one to `T/media/CallEchoSuppressorTest.kt`:
  1. Call `report(..., nowMs = 1_000 + 800)` with a loud chunk (a future timestamp).
  2. Assert `recentMaxDb(600, nowMs = 1_000)` is `SILENT_DB`.
  3. Assert it is loud at `nowMs = 1_900`.
  4. Assert it is silent again at `nowMs = 2_500`.

  `FarEndActivity` is a global `object`, so call `reset()` in `@Before`. `reset()` is
  currently never called anywhere.

### 2. An AirPlay side connection closing wipes the live CarPlay session's audio state

- **Where:** `S/airplay/CarPlayMediaEngine.kt:333-351` (`onSessionClosed`), specifically
  `audioMeta.clear()`, `audioCaptures.values.forEach(close)` and `audioCaptures.clear()`.
  Related: `onFeedback` at `:283`.
- **Why it matters now:**
  - Upstream tore everything down on any session end, so global clearing was harmless there.
  - The fork changed `S/orchestration/CarPlayController.kt:272-289` (`onSessionEnded`) to
    *keep* the iPhone session alive when another AirPlay client ends. An example is a Music
    Assistant server probing port 7000.
  - The engine's `onSessionClosed` still runs for that side session, and it clears state that
    belongs to the iPhone session.
- **Effect:**
  - `audioMeta` becomes empty, so `onFeedback` returns null. Audio timing feedback
    (`sampleTime`/`timestamp`) to the phone stops until the next audio SETUP.
  - The live session's audio packet captures get closed.
- **Fix:**
  - `audioMeta` and `audioCaptures` are now keyed by `StreamKey`, which includes the session.
    Remove only that session's entries:
    `audioMeta.keys.removeIf { it.session === session }`, and the same for `audioCaptures`,
    closing each one.
  - Also make `onFeedback(session)` report only `audioMeta` entries for that session. It
    currently reports all sessions' streams.
- **Test:** `T/airplay/CarPlayMediaEngineTest.kt` already builds engines with fake sessions.
  1. Set up audio on session A.
  2. Close session B.
  3. Assert `onFeedback(A)` is still non-null.

---

## P2: gaps and risks (look into these, then decide)

### 3. The side-connection fix only works after RECORD

- **Where:** `S/orchestration/CarPlayController.kt:272-289`.
- **Problem:**
  - `activeSession` is set only in `onSessionActive`, which fires on RTSP RECORD (see
    `S/airplay/AirPlaySession.kt` `handle()`).
  - Suppose a side client connects and disconnects while the iPhone is still in its handshake.
    `activeSession` is null, so the early return doesn't apply. The code then calls
    `uiListener?.onSessionEnded(session)`.
  - `C/CarPlayHostActivity.kt` (`createSessionListener` → `onSessionEnded`) responds by calling
    `reconnectAfterLoss`, which tears down the iPhone's handshake.
- **Possible fix:** forward the end to the UI only for a session that was ever active.
  Alternatively, track the "CarPlay session candidate" separately from `activeSession`.

### 4. Two streams of the same type can now be open at once

- **Where:** `S/airplay/CarPlayMediaEngine.kt:135` (`onAudio`) and `:312` (`onTeardown`). This
  came from upstream PR shilapi/xcertplay#44.
- **What changed:** `StreamKey` is now (session, type, audioType). A SETUP for type 100 with a
  different `audioType` (e.g. `speechRecognition` → `telephony`) no longer closes the earlier
  type-100 stream.
- **Risk:**
  - If the phone switches audio type without a TEARDOWN in between, the old `AudioStream`,
    its renderer and possibly a `MicrophoneUplink` stay open.
  - Two mic uplinks would mean two `AudioRecord`s competing for the mic.
  - TEARDOWN only carries the stream type, so it now tears down every audio type of that
    stream type at once.
- **Next step:**
  - Check a debug log of a Siri → phone call sequence. Look at the `airplay SETUP` and
    `airplay TEARDOWN types=` lines.
  - If there's no TEARDOWN between them, allow only one mic uplink per session and stream
    type, or replace the stream when its audio type changes.

### 5. The call AGC probably amplifies road noise

- **Where:** `S/media/Pcm16VoiceAgc.kt:16` (`gateDb = -55.0`, `maxGainDb = 30.0`,
  `riseDbPerBlock = 0.5`).
- **Problem:**
  - Gain is adjusted only on blocks louder than −55 dBFS.
  - A dash mic in a moving car likely sits above −55 dBFS from road noise alone. Pauses then
    count as speech, and gain climbs toward +30 dB at about 25 dB/s (blocks are about 20 ms).
  - Result: noise swells between words, and gain pumps.
- **Next step:**
  - Compare the `raw` and `sent` recordings in `files/audio-captures/mic/`. They're only
    written when "Audio diagnostic capture" is on.
  - Then raise the gate, or track a noise floor and gate relative to it.

### 6. Echo suppressor tuning, and you can't see its levels in the car

- **Where:** `S/media/CallEchoSuppressor.kt` (`farActiveDb = -55.0`, `doubleTalkMarginDb = 8.0`)
  and `S/media/MicrophoneUplink.kt:89`.
- **Problems:**
  - The far-end level comes from decoded PCM before track or system volume is applied. Turning
    the volume down therefore doesn't reduce suppression.
  - The −55 dBFS "active" threshold may also catch the caller's background noise or comfort
    noise, which would mute the mic for much of a call.
  - The level diagnostics go only to logcat (`log = { Log.i(TAG, it) }`), not the app's debug
    file log. You can't read them in the car.
- **Next step:**
  - Send the `call echo far=… mic=… gain=…` line to the debug log. The uplink would need the
    controller's `debugLog`, or a callback passed in via `AndroidMediaSink`.
  - Then tune from real logs.

### 7. Wi-Fi Direct group on a DFS channel may fail without falling back

- **Where:** `S/network/WifiP2pGroupManager.kt:109-149` and `groupConfig` at `:170`.
- **Problem:**
  - The group is created on the home Wi-Fi's 5 GHz frequency. The code falls back to "any
    5 GHz" only when `createGroup` is rejected outright with a non-BUSY reason.
  - On a DFS channel (5260–5720 MHz), many drivers won't run a group owner. If the driver
    accepts the request and the group never forms, `awaitGroupCreated` times out. Timeouts are
    never retried, so wireless start-up fails.
- **Fix:** have `connectedStation5GhzFrequency()` return null for 5260..5720. Alternatively,
  fall back on a timeout of the station-frequency attempt too, being careful that a late group
  may still appear.

### 8. Steering-wheel key mapping

- **Where:** `C/CarPlayHostActivity.kt:872-898` (`dispatchKeyEvent`).
- **Notes:**
  - `KEYCODE_BACK` always goes to CarPlay while a session is up and no menu is open, so the
    system back button can no longer reach Android. This is probably intended; confirm.
  - `KEYCODE_HEADSETHOOK` maps to play/pause, so it won't answer an incoming call.
  - The telephony HID device is already declared, and `AirPlaySession.sendTelephony(index)`
    exists at `S/airplay/AirPlaySession.kt:185`. Consider mapping `KEYCODE_CALL` and
    `KEYCODE_ENDCALL`, and `HEADSETHOOK` while a call is active, to answer/hang up. The
    telephony usages are in `AirPlayHid` `telephonyDescriptor`; 0x20 is the hook switch. This
    would need a pass-through `sendTelephony` on `CarPlayController`, like the existing
    `sendMedia`.

---

## P3: cleanup

- **Stale ndk-build files:** delete `shared/src/main/jni/Android.mk` and `Application.mk`.
  `shared/build.gradle` now uses `CMakeLists.txt`, which is equivalent: same module, no STL,
  android-28.
- **`S/media/CallVoiceProcessing.kt`:**
  - The header comment says the platform effects aren't used by default. `attachEchoCanceller`
    is now always attached for calls; update the comment.
  - `attachEffects()` is dead code.
  - `audioSourceFor()` ignores its parameter and always returns MIC. Inline it or drop the
    parameter.
- **`FarEndActivity.reset()`:** never called. Use it in tests (see #1), or reset when a call
  mic starts.
- **`S/orchestration/HeadUnitCallState.kt`:** `onModesChanged` parsing has no tests. The
  shapes are `{appStateID=2, entity=1}`, `entity=0`, and the boolean `state`. It needs a
  `Context`, so either pull the parsing into a pure function or use a fake context.
- **`S/media/AndroidMediaSink.kt:176`:** `mediaTap?.onAudioStopped(id.type)` stops web-display
  audio forwarding for *every* stream of that type when only one audio type stops. Minor; it
  only affects the remote web display.
- **CI:** consider enabling Actions on the fork so `android.yml` runs on pushes to `binnacle`.

## Reviewed and fine (no action)

- **Hotspot and Wi-Fi Direct retry loops** (`LocalOnlyHotspotManager`, `WifiP2pGroupManager`):
  - They stay within the start-up deadline and guard against `wait(0)`.
  - They retry only explicit BUSY/generic rejections and never retry a timeout.
  - They remove a leftover group only once.
- **`HeadUnitCallState`:** the broadcast is limited to `com.xbs.canapp` via `setPackage`. It
  sends OFF at start-up and on session end or controller close, and acts only for the active
  session.
- **Key mapping mechanics:** `sendKnob` sends momentary press/release reports. The media
  indices 1–5 match `AirPlayHid.mediaDescriptor` (Play, Pause, Play/Pause, Next, Previous).
- **`CallMicRecorder`:** opt-in, writes to app-private `filesDir`, and keeps the last three
  calls.
- **MFi:** the 5 ms settle delay after register select is small and well commented.
- **OEM tile:** `onHostUiRequested` → HOME intent is fine.
- **Defaults:** the "Binnacle" OEM label and the edge-swipe default being on are intentional
  fork defaults.

## Suggested order

1. Run the unit tests and lint locally to get a baseline.
2. Fix #1 and #2, with tests. These are small and confirmed.
3. Fix #7, since the DFS guard is one line, and the P3 cleanups.
4. Investigate #3 and #4 from debug logs before changing behaviour.
5. Tune #5 and #6 from in-car recordings after routing the suppressor log to the debug log.
