# Sonar Vision — Technical Documentation

Sonar Vision is a browser-based assistive prototype that fuses an
ultrasonic (sonar) distance sensor with in-browser computer vision to help
a blind or low-vision user sense nearby obstacles and recognize known
people. It runs as a single-page web client served by a small backend that
relays sonar readings over Socket.IO.

## 0. Current status

This repository currently contains **only the browser client** (`index.html`).
Everything below describes the intended full system, but not everything is
implemented here yet — this section says which is which.

**Implemented and working today, client-side, no hardware required:**
- Object detection (COCO-SSD / TensorFlow.js)
- Face recognition + enrollment (`@vladmandic/face-api`), stored locally in IndexedDB
- Audio sonification, Web Speech identity/alert announcements, vibration feedback
- Graceful fallback to "sonar only" or "camera unavailable" modes
- Zone calibration, saved to `localStorage`

**Described in this README but not yet in this repository:**
- The Arduino/ESP32 + HC-SR04 sonar firmware
- The Node.js + Socket.IO backend that relays real distance readings to the client
- Automated tests

The client already listens for a `distance` Socket.IO event in the shape
described in §4.1 — it just has nothing producing that event yet. Building
the sensor + backend is the next milestone; until then, the sonar-fusion
behavior described in this document is designed but unverified end-to-end.


## 1. What it does

- Reads live distance (and closing speed) from an ultrasonic sensor
  (e.g. an Arduino/ESP32 running an HC-SR04-style sketch) pushed to the
  browser over Socket.IO.
- Runs COCO-SSD (TensorFlow.js) on the device camera to detect and label
  mobility-relevant objects (people, vehicles, furniture, etc.).
- Runs face recognition (`@vladmandic/face-api`) so enrolled family
  members and friends are announced by name instead of just "person."
- Converts distance into continuous audio ("sonification") — a beep that
  gets faster and higher-pitched the closer something is — plus stereo
  panning toward the side of frame an object/face is on, and short
  vibration pulses in sync with the beeps on devices that support them.
- Speaks identity ("Mom detected", "Chair detected") using the Web Speech
  API, separately from the continuous tone that communicates distance.
- Lets the user calibrate the "near" and "close" distance boundaries to
  their own body/cane reach, saved locally so it persists across sessions.
- Falls back gracefully to sonar-only mode if the camera or either model
  fails to load.

## 2. Architecture

```
 ┌────────────┐      distance readings       ┌───────────────────┐
 │  Sonar HW  │ ───────────────────────────▶ │  Backend (Node +  │
 │ (Arduino/  │        (serial → server)     │   Socket.IO)      │
 │  ESP32)    │                              └─────────┬─────────┘
 └────────────┘                                        │ 'distance' event
                                                        ▼
                                          ┌─────────────────────────────┐
                                          │        Browser client        │
                                          │  index.html (this project)  │
                                          │                              │
                                          │  Camera ──▶ COCO-SSD ───┐   │
                                          │                          ├─▶│ Fusion → Speech
                                          │  Camera ──▶ face-api ────┘   │  + Web Audio
                                          │                              │  sonification
                                          │  Socket.IO ──▶ distance/speed│
                                          └─────────────────────────────┘
```

Everything except the sonar readings themselves (distance + speed, pushed
over Socket.IO) runs client-side in the browser. No video, images, or face
data are ever sent to a server.

## 3. Key files

| Path | Purpose |
|---|---|
| `index.html` | The entire client: markup, styling, and all JavaScript logic described below. |

The backend (Socket.IO server relaying sonar readings, and whatever serves
`/style.css` and `/socket.io/socket.io.js`) is a separate piece not covered
by this file — this doc only describes the client.

## 4. Core subsystems

### 4.1 Sonar input
The client listens for a `distance` Socket.IO event shaped like:
```js
{ distanceCm: number | null, speedCmPerSec?: number }
```
`distanceCm: null` readings are ignored. Every valid reading updates the
HUD, drives the sonification tone, and re-runs the fusion logic.

### 4.2 Object detection (COCO-SSD)
Runs on a throttled loop (`DETECTION_INTERVAL_MS`, default 300ms).
Detections are filtered to a `RELEVANT_CLASSES` allow-list (person,
vehicles, common furniture, etc.) and a minimum confidence
(`CONFIDENCE_THRESHOLD`, default 0.6) to cut down on irrelevant or noisy
announcements (e.g. "kite," "frisbee").

### 4.3 Face recognition (`@vladmandic/face-api`)
Runs on its own, slightly less frequent loop
(`FACE_DETECTION_INTERVAL_MS`, default 500ms). Uses `TinyFaceDetector` +
`FaceLandmark68Net` + `FaceRecognitionNet`. Recognized faces are matched
against enrolled descriptors with `faceapi.FaceMatcher` at a distance
threshold of `FACE_MATCH_THRESHOLD` (default 0.55 — lower is stricter).

**Enrollment**: the user points the camera at someone, types a name, and
taps "Add from camera." One face descriptor (a numeric vector, not a
photo) is captured and stored.

**Storage**: descriptors are stored in IndexedDB (`sonarVisionFaces` →
`faces` store), keyed by an auto-increment id, each row `{ name,
descriptor }`. Nothing is uploaded. Deleting a person removes their row
and rebuilds the matcher.

⚠️ **Load-order caveat**: face-api's browser bundle ships its own copy of
TensorFlow.js and reassigns the global `tf`. It's loaded *after* the
`tf` + `coco-ssd` `<script>` tags specifically so coco-ssd (which already
captured its own reference at script-eval time) keeps working. If object
detection ever silently stops working right after the face models finish
loading, this global `tf` collision is the first thing to check.

### 4.4 Identity fusion
`getIdentity()` decides what to announce: a fresh, named face match takes
priority over a generic object class, so a recognized person beats a
plain "Person detected." Both object and face results carry a timestamp;
anything older than `STALE_MS` (700ms) is treated as unreliable and
ignored, so a stale detection can't get paired with a fresh sonar reading
or vice versa.

### 4.5 Sonification (Web Audio API)
A single oscillator + gain node (+ stereo panner where supported) plays a
continuous tone whose **pitch and beep rate** are driven by sonar
distance, bucketed into five zones:

| Zone | Distance | Behavior |
|---|---|---|
| far | > 300 cm (or above calibrated near) | silent |
| approach | 100–300 cm | slow beep |
| near | 50–100 cm (calibratable) | medium beep |
| close | 20–50 cm (calibratable) | fast beep |
| contact | < 20 cm | solid tone |

Panning is driven separately by the **camera's** frame position of the
top identity match (face or object), not the sonar. This is a known
limitation: a single ultrasonic sensor has no directional information of
its own, so pan and pitch can point at different things if the sonar's
cone and the camera's field of view aren't both centered on the same
object. Genuine per-side distance would require multiple ultrasonic
sensors (see §6).

### 4.6 Speech (Web Speech API)
Speech is reserved for **identity** ("Mom detected", "Chair detected")
and urgent alerts ("Stop! Person approaching fast") — never raw distance,
which the sonification tone already communicates faster than a sentence
can be spoken. Cooldowns (`SPEECH_COOLDOWN_MS`, `STOP_COOLDOWN_MS`)
prevent repeated announcements of the same identity. Known iOS Safari
workarounds are included: a silent "unlock" utterance fired synchronously
on the Start button tap, a short delay after `cancel()` before the next
`speak()`, and a periodic `resume()` nudge to prevent the speech queue
from going idle-dead.

### 4.7 Haptics
`navigator.vibrate()` pulses in sync with each beep and with a distinct
stronger pattern for the urgent "stop" alert. Wrapped in try/catch since
iOS Safari doesn't support the Vibration API — it silently no-ops there.

### 4.8 Calibration
The "near" and "close" zone boundaries can be recalibrated by holding an
object at the desired distance and tapping the matching button; the new
boundary is read directly from the latest sonar value and saved to
`localStorage` (`sonarVisionZones`) so it persists across reloads. A
"Reset to defaults" button restores the shipped boundaries.

### 4.9 Graceful degradation
- If COCO-SSD fails to load, the Start button becomes "Start (sonar
  only)" and the app runs on sonar + audio/speech alone, no camera
  required.
- If the camera permission/stream fails, the same sonar-only fallback
  kicks in without blocking the rest of the app.
- If face-api's models fail to load, face recognition is silently
  skipped; object detection and sonar continue normally.

## 5. Configuration reference

All tunables live as constants near the top of the `<script>` block:

| Constant | Default | Meaning |
|---|---|---|
| `DETECTION_INTERVAL_MS` | 300 | Object-detection loop interval |
| `FACE_DETECTION_INTERVAL_MS` | 500 | Face-detection loop interval |
| `SPEECH_COOLDOWN_MS` | 3000 | Min. gap between normal announcements |
| `STOP_COOLDOWN_MS` | 2000 | Min. gap between urgent "stop" alerts |
| `APPROACH_SPEED_THRESHOLD_CM_S` | 30 | Closing speed that triggers "approaching fast" |
| `STALE_MS` | 700 | Max age before a reading is treated as stale |
| `CONFIDENCE_THRESHOLD` | 0.6 | Min. COCO-SSD confidence to keep a detection |
| `FACE_MATCH_THRESHOLD` | 0.55 | Max. descriptor distance to count as a match |
| `RELEVANT_CLASSES` | ~20 classes | Allow-list of COCO-SSD classes that get announced |
| `DEFAULT_ZONES` | 5 zones, 300/100/50/20/0 cm | Sonification distance buckets |

## 6. Known limitations / next steps

- **Direction is camera-derived, not sonar-derived.** A single ultrasonic
  sensor can't tell left from right; true per-side distance needs
  multiple sensors (e.g. left/center/right) or a scanning/servo-mounted
  sensor.
- **Pan and pitch can reference different objects** when the camera's
  top match and the sonar's cone aren't aimed at the same thing.
- **Relevant-class list and confidence threshold are hand-tuned**, not
  validated against real-world testing conditions.
- **COCO-SSD has no "stairs" or "door" class** — a custom-trained model
  would be needed to detect those directly.
- **Face match threshold (0.55) is a starting point**, not verified
  against the actual camera/lighting conditions this will be used in.
- **No real blind/low-vision user testing has been done yet.** The
  beep/pan/vibration mapping is a design guess and should be validated
  with an actual user before relying on it.

## 7. Privacy notes

- Face data is stored only as numeric descriptors in the browser's
  IndexedDB, never as photos, and never transmitted to any server.
- Enrollment is deliberate and manual — the app never scans or stores
  faces automatically in the background.
- Removing a person from the "Family & friends" list deletes their
  stored descriptor immediately.
  
