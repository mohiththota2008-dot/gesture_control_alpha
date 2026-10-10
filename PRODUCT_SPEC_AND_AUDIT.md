# AETHER HUD: Technical Architecture, System Audit & Product Specification (PRD)

**Version:** 1.0.0-PROD  
**Target Platform:** Web (Desktop/Laptop with Camera + Microphone)  
**Core Technologies:** Google MediaPipe Hands, Web Speech API, Web Audio API, Vanilla HTML5/CSS3/ES6+  

---

## 1. System Overview & Executive Summary

AETHER HUD is a touchless, spatial-computing web interface inspired by Iron Man holographic heads-up displays (HUD). It enables users to operate a simulated desktop interface through:
1. **Optical Hand Tracking** via webcam and Google MediaPipe Hands (BlazePalm + Hand Landmark detector).
2. **Acoustic Voice Control** via the browser's native Web Speech Recognition API (`SpeechRecognition`).
3. **Synthetic Audio Feedback** using the Web Audio API (procedural oscillators without external audio asset dependencies).
4. **Holographic UI Engine** with glassmorphic styling, CSS animations, and low-latency cursor interpolation.

---

## 2. Function Call Graph & Subsystem Interconnections

The application architecture operates as an asynchronous, event-driven reactive loop. Below is the relationship map and execution flow:

```
[ User Action: "Engage Spatial Tracking" (startTrackingBtn) ]
                     │
                     ├───> initAudio() [Resumes AudioContext]
                     ├───> playSciFiSound('wake')
                     ├───> setupHandTracking() ──────────────────────────────────────────────┐
                     │          │                                                            │
                     │          ├──> navigator.mediaDevices.getUserMedia({video})            │
                     │          └──> new Hands() & handsModel.setOptions()                   │
                     │                    │                                                  │
                     │                    └──> processVideoFrame() (requestAnimationFrame)   │
                     │                              │                                        │
                     │                              └──> handsModel.send({image: videoEl})   │
                     │                                        │                              │
                     │                                        └──> onHandResults(results)    │
                     │                                                  │                    │
                     │         ┌────────────────────────────────────────┴──────────────┐     │
                     │         │                                                       │     │
                     │    [Single Hand: Primary]                             [Dual Hand]     │
                     │         │                                                   │         │
                     │         ├──> Landmark 8 (Index Tip)                         └──> Index1 ↔ Index2
                     │         │    └──> Virtual Cursor Lerp Mapping                     Dist Delta
                     │         │    └──> checkHoverUnderCursor(x, y)                       │
                     │         │                                                           └──> applyZoom(delta)
                     │         ├──> Pinch Distance (Thumb 4 ↔ Index 8)
                     │         │    └──> Threshold check (< 0.055)
                     │         │         └──> executeClickAt(x, y, 'GESTURE')
                     │         │                   │
                     │         │                   ├──> createRipple(x, y)
                     │         │                   ├──> element.click()
                     │         │                   ├──> playSciFiSound('click')
                     │         │                   └──> logEvent('gesture', ...)
                     │         │
                     │         └──> Single Hand Span (Thumb 4 ↔ Pinky 20)
                     │              └──> applyZoom(spanDelta)
                     │
                     └───> startVoiceRecognition() ──────────────────────────────────────────┘
                                │
                                ├──> getUserMedia({audio: true}) [Permission preflight]
                                ├──> new SpeechRecognition()
                                ├──> continuous = true, interimResults = true, lang = 'en-US'
                                ├──> onstart ──> updateVoiceHUD('LISTENING')
                                ├──> onresult(event)
                                │         ├──> interim ─> updateLiveTranscript(interimText, false)
                                │         └──> isFinal ─> handleFinalVoiceCommand(chunk, confidence)
                                │                             │
                                │                             ├──> Debounce check (850ms window)
                                │                             ├──> "Click" ──────────> executeClickAt(cursor.x, cursor.y, 'VOICE')
                                │                             ├──> "Zoom in" ────────> applyZoom(+0.25)
                                │                             ├──> "Zoom out" ───────> applyZoom(-0.25)
                                │                             ├──> "Scroll up" ──────> scrollFeed(-120)
                                │                             ├──> "Scroll down" ────> scrollFeed(+120)
                                │                             ├──> "System sleep" ───> toggleSystemSleep(true)
                                │                             └──> "Wake up" ────────> toggleSystemSleep(false)
                                ├──> onerror ──> 'not-allowed' (BLOCKED) / 'audio-capture' (NO MIC) / 'network'
                                └──> onend   ──> auto-reconnect if unintentional interruption, or stay OFF if user clicked stop
```

---

## 3. Function Breakdown & Connection Matrix

| Function Name | Inputs | Primary Responsibilities | Connected / Invoked Functions |
| :--- | :--- | :--- | :--- |
| `initAudio()` | None | Unlocks / resumes the Web Audio `AudioContext` on user interaction. | Called by `startTrackingBtn`, `toggleVoiceBtn`, `footerVoiceBtn`. |
| `playSciFiSound(type)` | `'click' \| 'zoom' \| 'voice' \| 'sleep' \| 'wake'` | Synthesizes custom oscillator frequencies (sine, triangle, sawtooth ramps) without external MP3 files. | Invoked by `executeClickAt`, `applyZoom`, `handleFinalVoiceCommand`, `toggleSystemSleep`. |
| `logEvent(type, text)` | `type: string, text: string` | Prepends an entry into the real-time event telemetry stream; capped at 40 rows. | Invoked across all subsystems whenever state changes occur. |
| `createRipple(x, y)` | `x, y` (viewport pixels) | Spawns a glowing neon circular ripple animation that self-destructs after 550ms. | Invoked by `executeClickAt`. |
| `triggerElementClickEffect(el)` | `HTMLElement` | Adds `.simulated-active` CSS class to give visual feedback and plays a click audio cue. | Invoked by `executeClickAt`. |
| `executeClickAt(x, y, source)` | `x, y, source: string` | Temporarily hides cursor element to perform `document.elementFromPoint(x, y)`, finds nearest actionable button or card, and triggers `.click()`. | Invoked by pinch gesture detector, voice command `"Click"`, and fallback simulation buttons. |
| `applyZoom(delta)` | `delta: number` | Clamps zoom scale between `0.4` and `2.8`, applies CSS 3D matrix transform to `#target-element`, updates HUD scale badge, triggers glow. | Invoked by dual-hand pinch, single-hand thumb-pinky spread, voice `"Zoom in"`/`"Zoom out"`, and zoom buttons. |
| `scrollFeed(amount)` | `amount: number` (pixels) | Performs smooth scroll on the `#scrollable-feed` log element and updates percentage position readout. | Invoked by voice commands `"Scroll up"` and `"Scroll down"`. |
| `toggleSystemSleep(state)` | `boolean \| undefined` | Activates/deactivates the stasis overlay `#sleep-overlay`, stops or restarts active tracking, plays power-down or power-up sound. | Invoked by voice `"System sleep"` / `"Wake up"`, `#sleep-btn`, `#wake-btn`. |
| `setupHandTracking()` | None | Loads `@mediapipe/hands`, requests webcam stream via `getUserMedia`, binds video element, and initiates RAF loop. | Called once on onboarding click. |
| `processVideoFrame()` | None | Self-scheduling `requestAnimationFrame` loop that checks video readiness (`readyState >= 2`) and pipes frames into `handsModel.send()`. Also calculates real-time FPS. | Recursively schedules itself; feeds `handsModel`. |
| `onHandResults(results)` | `MediaPipe HandResults` | Synchronizes canvas dimensions, mirrors coordinates, draws joint connectors/dots, calculates cursor positions with exponential smoothing (lerp), checks pinch distance threshold, and checks zoom distance. | Invoked directly by MediaPipe inference callback; invokes `checkHoverUnderCursor`, `executeClickAt`, `applyZoom`. |
| `checkHoverUnderCursor(x, y)` | `x, y` (viewport coords) | Detects buttons beneath virtual cursor and applies `.simulated-hover` glowing styles before clicks occur. | Invoked during every frame in `onHandResults`. |
| `updateVoiceHUD(status, errMsg)` | `status, errMsg` | Truthfully updates speech sensor dot (`off`, `starting`, `active`, `reconnecting`, `error`), text, animated wave bars, toggle buttons, and error hint. | Called by `startVoiceRecognition`, `stopVoiceRecognition`, recognition callbacks. |
| `updateLiveTranscript(text, isFinal, conf, isExec)` | `text, isFinal, conf, isExec` | Displays real-time interim speech and confirmed final commands in the HUD transcript bar. | Called during `onresult` in SpeechRecognition. |
| `startVoiceRecognition()` | None | Preflights microphone permissions, ensures singleton recognizer lifecycle, configures `continuous`/`interimResults`/`en-US`, binds resilient handlers. | Triggered by user via `toggleVoiceBtn`, `footerVoiceBtn`, or onboarding button. |
| `stopVoiceRecognition(isUserAction)` | `boolean` | Aborts speech recognition, marks intentional stop (`isIntentionalStop = true`) to prevent unwanted restart loops. | Triggered by user when clicking disable voice. |
| `toggleVoiceRecognition()` | None | Switches voice recognition between active listening and stopped state. | Invoked by HUD voice toggle buttons. |
| `normalizeVoiceText(text)` | `string` | Cleans transcript: lowercases, strips punctuation, normalizes whitespace. | Invoked inside `handleFinalVoiceCommand`. |
| `handleFinalVoiceCommand(transcript, conf)` | `transcript, conf` | Normalizes speech, deduplicates within 850ms window, and executes the 7 target actions. | Invoked on finalized recognition tokens. |

---

## 4. Root Cause Analysis of Previous Voice Input Failure

1. **Microphone Permission Deadlock:**
   The original startup flow requested camera video via `navigator.mediaDevices.getUserMedia({ video: ... })` but never prompted for audio microphone permissions. Calling `SpeechRecognition.start()` without prior user microphone permission led to immediate `not-allowed` errors or silent failures in sandbox/iframe environments.
2. **Infinite Restart Loop on Error:**
   The previous `onend` handler blindly called `speechRecognizer.start()` whenever `state.micActive` was set, regardless of whether a fatal error (like `not-allowed` or `audio-capture`) had occurred. In Chromium, this caused rapid restart recursion that froze the Web Speech subsystem.
3. **Missing User Activation Controls:**
   Modern browsers require user activation gestures before accessing audio hardware. There was no dedicated "ENABLE VOICE" button in the HUD to independently request mic permissions or restart the engine.
4. **Duplicate Command Execution & Lack of Interim Handling:**
   Without interim results and final token deduplication, multiple result events for the same phrase triggered repeated duplicate clicks or zooms.
5. **Untruthful Status Representation:**
   The UI showed `LISTENING` and played wave animations based on optimistic state rather than genuine confirmation from the SpeechRecognition `onstart` event.

---

## 5. Implementation Fixes & Hardening

1. **Direct Microphone Preflight:** `startVoiceRecognition()` uses `navigator.mediaDevices.getUserMedia({ audio: true })` inside user click handlers to explicitly trigger the browser permission prompt and verify hardware availability before instantiating `SpeechRecognition`.
2. **Intentional vs. Unintentional Interruption Recovery:** The `isIntentionalStop` flag distinguishes when a user deliberately disables voice control (stays OFF) versus when Chromium times out after silence (smoothly auto-reconnects with backoff, up to 4 attempts).
3. **Accurate HUD Feedback States:** The speech indicator explicitly transitions between `OFF`, `STARTING`, `LISTENING`, `RECONNECTING`, `BLOCKED`, `NO MIC`, and `UNSUPPORTED`. Wave animations run ONLY when genuine listening is confirmed.
4. **Live Transcript Banner:** Added `#voice-live-banner` displaying the live speech token with confidence score and visual feedback when commands are executed.
5. **Dedicated HUD Controls:** Added `#toggle-voice-btn` in the right sensor card and `#footer-voice-btn` in the footer.
6. **Command Deduplication:** Spoken commands are normalized and debounced using an 850ms window and a finalized index registry (`processedFinalIndices`), eliminating duplicate firings.
7. **Zero Impact on Gesture Tracking:** Webcam hand tracking and MediaPipe inference run completely independently in their own animation loop.
