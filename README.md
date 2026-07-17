# BarGuardAI

An AI-assisted CCTV monitoring system for bars and restaurants — built in Java with OpenCV. It watches your camera feed and handles three things automatically: detecting fights, counting customers, and recognizing staff vs. strangers, with WhatsApp alerts for anything that needs attention.

Everything currently runs **locally** — no cloud calls, no per-frame API costs. A Claude Vision integration (for smarter context-aware detection: theft, restricted-zone monitoring, etc.) is planned as a second phase once this local pipeline is proven stable.

---

## What it does today

- **🥊 Fight detection** — a YOLO11-Pose model estimates a 17-point skeleton for every person in frame. Wrist speed, raised fists, punch/kick extension, proximity between people, and repeated arm acceleration all feed into a confidence score. An alert only fires once that confidence stays high for several consecutive frames, so a single fast gesture doesn't trigger a false alarm.
- **👥 Customer counting** — headcount is sampled from the same pose data (or face count as a fallback) and reported over WhatsApp on a fixed interval (default: every 10 minutes).
- **🧑‍💼 Worker recognition & attendance** — enroll staff faces once (live capture or by dropping a photo in `persons/`). From then on, every time that person's face is seen on camera, they're automatically marked present — with a WhatsApp message (e.g. *"Douglas Mkuyuni Reported"*) sent at most once every 24 hours per person.
- **🔍 Person search** — every face the system ever sees (staff or not) is fingerprinted and logged with a timestamp and camera ID. Upload a photo of anyone and see every camera/time they were recorded.
- **📱 WhatsApp alerts** — fight alerts, headcount reports, and attendance pings are all delivered via [CallMeBot](https://www.callmebot.com/blog/free-api-whatsapp-messages/)'s free WhatsApp API.

## Coming next

- **Claude Vision integration** for context-aware analysis the local pipeline can't do on its own: theft detection, restricted-zone monitoring ("is someone behind the bar who shouldn't be"), and general scene understanding. `AppConfig` already has the toggles and API key field wired in for this.

---

## Architecture

```
Camera → FaceDetector ─────► FaceIdentityService ──► SightingLog / WorkerAttendanceService
      │                                                        │
      └→ PoseDetector → FightDetector → SkeletonDrawer         └→ WhatsAppService
      │
      └→ MotionDetector

DashboardUI ties all of the above together and renders the live annotated feed.
```

| Package | Responsibility |
|---|---|
| `com.barguard` | `DashboardUI` — the Swing app and main camera pipeline |
| `com.barguard.camera` | Webcam/RTSP capture |
| `com.barguard.vision` | Pose estimation, skeleton drawing, fight detection, face detection, motion detection |
| `com.barguard.faces` | Face embedding, identity matching, worker directory, sighting log, attendance throttling |
| `com.barguard.ui` | Add Worker and Search Persons dialogs |
| `com.barguard.alerts` | WhatsApp delivery |
| `com.barguard.util` | Customer count reporting |
| `com.barguard.config` | `config.properties` loading |

### Fight detection pipeline

Pose keypoints are matched to the same person across frames (nearest-centroid tracking), then scored by `AggressionAnalyzer` on: arm/leg/head speed, body lean, raised fists, punch/kick extension ratio, and inter-person distance. `FightDetector` layers rule-based bonuses on top (multiple aggressive people at once, repeated arm acceleration, sustained proximity) and only reports a fight once confidence stays above threshold for multiple consecutive frames.

### Face recognition pipeline

Faces are converted into 128-dimension embedding vectors using **SFace**, run through OpenCV's core DNN module (`org.opencv.dnn`) — deliberately avoiding OpenCV's `opencv_contrib` face module, which isn't included in most prebuilt OpenCV Java distributions. Two faces are considered the same person if their embeddings' cosine similarity exceeds `0.363` (OpenCV's own published SFace threshold). Every new face becomes a new identity automatically; naming an identity (enrolling a worker) is a separate, deliberate step.

---

## Requirements

- **JDK 21+** (a JRE alone will not compile the project)
- **OpenCV 4.12.0** Java bindings (`opencv-4120.jar`) + matching native library for your OS
- **IntelliJ IDEA** (Community Edition is fine) — plain Java project, no Maven/Gradle
- Two ONNX models (see below)
- A webcam or RTSP camera stream

## Setup

1. **Add OpenCV to the project**
   `File → Project Structure → Libraries → +` → point at `opencv-4120.jar`, and make sure the native library (`opencv_java4120.dll`/`.so`) is on your system `PATH` or loaded via `System.loadLibrary(...)`.

2. **Download the two models** into a `models/` folder at the project root:
   - `models/yolo11n-pose.onnx` — pose estimation (export from Ultralytics YOLO11, or use a pre-converted copy)
   - `models/face_recognition_sface_2021dec.onnx` — face embeddings, from the official OpenCV model zoo:
     [opencv/opencv_zoo — face_recognition_sface](https://github.com/opencv/opencv_zoo/blob/main/models/face_recognition_sface/face_recognition_sface_2021dec.onnx)

3. **Add the face detection cascade** at `resources/haarcascade_frontalface_default.xml` — grab it from [OpenCV's GitHub](https://github.com/opencv/opencv/blob/4.x/data/haarcascades/haarcascade_frontalface_default.xml).

4. **Configure WhatsApp alerts** (free, via CallMeBot):
   - Save `+34 644 59 78 02` as "CallMeBot" in your phone's contacts
   - WhatsApp them: *"I allow callmebot to send me messages"*
   - They'll reply with your API key

5. **Run the app once** — it generates a `config.properties` template on first launch if one doesn't exist. Fill in your WhatsApp phone/API key and adjust settings, then restart.

## `config.properties`

```properties
# WhatsApp Alerts via CallMeBot (free)
whatsapp.phone=+254712345678
whatsapp.api_key=YOUR_CALLMEBOT_API_KEY

# Camera streams — WEBCAM for testing, or RTSP URL(s) for real cameras
cameras=WEBCAM

# Known persons folder — drop named JPG/PNG files here (e.g. john_doe.jpg -> "John Doe")
persons.folder=persons

# Minimum gap before re-alerting the same event type (seconds)
# Fight alerts always enforce a 60s floor regardless of this value.
alert.cooldown_seconds=30

# Enable/disable detection modules
enable.fights=true
enable.customer_count=true
enable.zones=true
enable.theft=true
```

## Usage

- **Start Camera** — begins the live feed and all enabled detection modules.
- **Stop Camera** — releases the camera. Required before using Add Worker (most webcams can't be opened by two capture sessions at once).
- **Add Worker** — opens a dedicated camera preview; capture a face, type a name, save. That person is now recognized automatically and reported once per 24 hours when seen.
- **Search Persons** — upload a photo; if it matches anyone the system has ever seen, every camera and timestamp they were recorded at is listed.

## Known limitations

- Face crops aren't landmark-aligned before recognition (no eye/nose/mouth alignment step), so accuracy is decent but not state-of-the-art. Good, well-lit, front-facing enrollment photos matter.
- `CameraManager` currently only opens the local webcam (`VideoCapture(0)`); the RTSP multi-camera support implied by `config.properties` isn't wired up yet.
- Customer counting reports a point-in-time snapshot at each interval, not a running total of unique visitors.
- All detection runs on CPU via OpenCV's DNN backend — no GPU acceleration configured.

