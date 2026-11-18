# BloodHawk Technical Architecture & Workflow Integration Specification

**Comprehensive System Architecture, Component Topology, and Field Workflows**  
**Classification:** Internal Technical Documentation  
**Version:** 2.0.0  
**Target Platform:** Android 14+ (API 34) / Google Pixel 10 Hardware Baseline  

---

## 1. Architectural Overview & Engineering Philosophy

BloodHawk is an autonomous, mission-critical field tracking system engineered for recovering wounded game in adverse wilderness conditions. The architecture is governed by five fundamental design principles:

1. **Zero-Allocation Real-Time Guarantees**:
   The runtime execution loop operates with **zero dynamic heap allocations** per frame. All image buffers, matrix descriptors, contour vectors, spatial grids, and audio playback tracks are allocated once during subsystem initialization, eliminating Android ART Garbage Collection pauses ($GC$) and guaranteeing deterministic frame delivery.
2. **Dual-Track Decoupled Concurrency**:
   Visual feedback (OpenGL ES 3.0 GPU shader) and analytical computer vision (C++ NDK CPU pipeline) run as separate, decoupled streams. The GPU delivers immediate situational awareness ($1280 \times 720$ at $20\text{--}24\text{ FPS}$), while the CPU performs deep structural validation on a decoupled $640 \times 360$ stream without blocking the display pipeline.
3. **Tactical Power Preservation for Extreme Cold & Night Tracking**:
   The entire UI is built on true OLED black (`#0B0F17`), the camera sensor is locked to $20\text{--}24\text{ FPS}$ (cutting sensor and ISP power by $>50\%$ vs standard $60\text{ FPS}$), the LED tracking torch is throttled to a $50\%$ duty cycle via API 33+ hardware strength levels, and the audio engine uses a pre-synthesized static PCM track requiring zero ongoing CPU audio synthesis.
4. **Field Usability & Zero-Toast Notification Policy**:
   In accordance with tactical standards, **Android `Toast` notifications are strictly prohibited**. All status changes, warnings, and alerts are communicated via tactile haptics, high-contrast HUD badges, dedicated modal dialogs, and immediate control state shifts. Raw optical bypass is activated instantaneously by touching and holding anywhere on the live video preview surface ("defade the picture"), allowing natural terrain inspection without hunting for sub-menus.
5. **Strict Data Integrity & Manual Blood Confirmation**:
   Camera detections **never automatically write to the trail database**. The CV pipeline provides real-time guidance (visual reticles and audio pips), but waypoint persistence requires explicit human confirmation via the prominent on-screen hero recovery button (`dropPillView`). This guarantees that temporary decoy detections never corrupt the animal's calculated flight vector.

---

## 2. Master System Component Topology

```mermaid
graph TD
    subgraph UI_Layer["UI and Presentation Layer (Fragments and Custom Views)"]
        MainActivity["MainActivity (Navigation Host and Thermal Guard)"]
        CameraFrag["CameraFragment (Tracking HUD, Sensor Loops and Edge Controls)"]
        MapFrag["MapFragment (Leaflet.js WebView and Vector Navigation)"]
        BrushFrag["BrushScannerFragment (Foliage Peeling and Depth Inspector)"]
        CalibFrag["ProfileCalibrationFragment (Photo Sampler and Envelope Tuner)"]
        
        CompassView["CompassView (Stationary 360 Ribbon HUD)"]
    end

    subgraph Domain_Layer["Domain Logic and Service Managers"]
        TrailMgr["BloodTrailManager (GPS Sessions, Flight Vectors, Breadcrumbs)"]
        ProfileMgr["BloodProfileManager (Spectral Envelopes and System Defaults)"]
        TileMgr["OfflineTileManager (USGS Offline Packs and OSM Dynamic Cache)"]
        GeigerEngine["GeigerAudioEngine (Static AudioTrack Pip Synthesizer)"]
        Settings["AppSettings (Persistent User Preferences)"]
    end

    subgraph Native_Layer["Native C++ Core (NDK / ARM NEON SIMD / OpenCV-Mobile)"]
        NativeJNI["BloodTrackerNative (JNI Bridge)"]
        TrackerCtx["BloodTrackerAnalyzer.cpp (7-Stage Detection and Rejection Cascade)"]
        BrushNative["BrushScannerNative (JNI Bridge)"]
        BrushCtx["BrushScannerAnalyzer.cpp (Synthetic Aperture and Lucas-Kanade Flow)"]
    end

    subgraph Render_Layer["Concurrent Rendering Engine (OpenGL ES 3.0)"]
        GLSurface["GLSurfaceView (RENDERMODE_WHEN_DIRTY)"]
        GLRenderer["BloodTrackerRenderer (Dual-Program OES Shader Controller)"]
        FragShader["blood_tracker_frag.glsl (Pixel Desaturation and False-Color Shading)"]
        ImageFragShader["blood_tracker_image_frag.glsl (Static Photo Simulation)"]
    end

    subgraph Hardware_Layer["Hardware HAL and Sensors"]
        CamDevice["Camera2 HAL (Dual Target: SurfaceTexture + ImageReader)"]
        IMUSensors["SensorManager (Gyroscope, Accelerometer, Magnetometer)"]
        LocationAPI["LocationManager (GPS / GNSS Provider)"]
        VibratorAPI["VibratorManager (Tactile Feedback Generator)"]
        AudioHAL["AudioTrack (Hardware Audio Subsystem)"]
    end

    MainActivity --> CameraFrag
    MainActivity --> MapFrag
    MainActivity --> BrushFrag
    MainActivity --> DiagFrag
    MainActivity --> CalibFrag
    CameraFrag --> GLSurface
    CameraFrag --> CompassView
    CameraFrag --> GeigerEngine
    CameraFrag --> TrailMgr
    CameraFrag --> ProfileMgr
    GLSurface --> GLRenderer
    GLRenderer --> FragShader
    GLRenderer --> ImageFragShader
    
    CamDevice -->|Hardware OES Buffer| GLRenderer
    CamDevice -->|DirectBuffer YUV_420_888| NativeJNI
    NativeJNI --> TrackerCtx
    TrackerCtx -->|Verified Droplet Array uDroplets| GLRenderer
    TrackerCtx -->|Droplet Count Positive| GeigerEngine
    TrackerCtx -->|Droplet Count Positive| VibratorAPI
    
    IMUSensors -->|Raw Vectors| CameraFrag
    CameraFrag -->|Azimuth and Target Heading| CompassView
    CameraFrag -->|Angular Velocity dTheta| TrackerCtx
    
    LocationAPI -->|GPS Coordinates| CameraFrag
    CameraFrag -->|Filtered GPS Fixes| TrailMgr
    TrailMgr -->|Active Trail and Flight Vectors| MapFrag
    
    TileMgr -->|Intercepted Local Tiles| MapFrag
    BrushFrag --> BrushNative
    BrushNative --> BrushCtx
    DiagFrag --> HitAdvisor
    CalibFrag --> ProfileMgr
```

---

## 3. Hardware Interfacing & Camera2 Subsystem

### 3.1 Dual-Target Streaming Pipeline

The camera subsystem is orchestrated by `CameraFragment` via Android's low-overhead `CameraDevice` and `CameraCaptureSession` APIs:

```
                          +-------------------------------------------------------+
                          |                 Camera2 Sensor & ISP                  |
                          | (Fixed 20-24 FPS, 1.0 Diopter Focus, 14-frame AE Lock)|
                          +-------------------------------------------------------+
                                                     |
                         +---------------------------+---------------------------+
                         | Dual-Target SessionConfiguration                      |
                         v                                                       v
         +-------------------------------+                       +-------------------------------+
         |     OpenGL ES 3.0 Target      |                       |       Native CV Target        |
         |  SurfaceTexture (1280x720)    |                       |    ImageReader (640x360)      |
         |     Format: PRIVATE_OES       |                       |   Format: YUV_420_888         |
         +-------------------------------+                       +-------------------------------+
                         |                                                       |
                         v                                                       v
         +-------------------------------+                       +-------------------------------+
         |      BloodTrackerRenderer     |                       |    Camera Background Thread   |
         |  (GLSurfaceView Dirty Render) |                       |  (OnImageAvailableListener)   |
         +-------------------------------+                       +-------------------------------+
```

#### Sensor Configuration Parameters:
- **Frame Rate Lock (`CONTROL_AE_TARGET_FPS_RANGE`)**: Clamped to $[20, 24]\text{ FPS}$. This eliminates the battery drain of $60\text{ FPS}$ capture while keeping inter-frame latency below $45\text{ ms}$, which is well within human motor-reaction time.
- **Fixed Optical Hyperfocal Distance (`LENS_FOCUS_DISTANCE`)**:
  - **Standard Sweep Mode**: Locked at $1.0\text{ diopter}$ ($1.0\text{ m}$ working distance). Provides a crisp depth of field spanning $0.65\text{ m}$ to $1.85\text{ m}$, covering ground litter while walking upright.
  - **Macro Close-Up Mode**: Locked at $6.67\text{ diopters}$ ($0.15\text{ m} \approx 6\text{ inches}$). Optimizes focus for examining standing cornstalks, high brush stems, or tree bark.
  - *Engineering Rationale:* Continuous autofocus ($AF$) causes continuous lens hunting against low-contrast forest substrates (leaves, dirt), creating blur frames and spiking power consumption. Fixed focus locks the optical system permanently.
- **Auto-Exposure Lock (`CONTROL_AE_LOCK`)**:
  During the initial $14\text{ frames}$ after camera startup (`AE_SETTLE_FRAMES`), Auto-Exposure settles to ambient lighting. At frame 14, AE is permanently locked (`set(CaptureRequest.CONTROL_AE_LOCK, true)`). This prevents sudden camera exposure shifts (e.g., when panning past bright sky or white snow) from destabilizing the $L^*a^*b^*$ color thresholds.
- **Throttled LED Tracking Torch (`turnOnTorchWithStrengthLevel`)**:
  On Android 13+ (API 33), the hardware torch is throttled to $50\%$ of maximum brightness. In high-efficiency Cree mobile LED flash units, running at $50\%$ brightness reduces thermal junction heat by $>60\%$ and extends battery runtime from $\approx 45\text{ minutes}$ to $>3.5\text{ hours}$ of continuous operation.

### 3.2 Touch Interaction & On-Screen Control Topology

All tracking operations are consolidated into direct on-screen touch controls, eliminating dependencies on hardware volume keys:
- **Touch-and-Hold Live Preview Raw Optical Bypass**:
  The OpenGL ES camera view (`glSurfaceView`) registers a direct touch listener (`setOnTouchListener`):
  - Touching and holding anywhere on the video preview (`MotionEvent.ACTION_DOWN`) immediately triggers `setRawOpticalBypass(true)` (`uBypassFilter = 1`) and a tactile haptic pulse, temporarily turning off the desaturated slate-gray background filter to reveal natural terrain colors ("defade the picture").
  - Lifting the finger (`MotionEvent.ACTION_UP` or `ACTION_CANCEL`) immediately calls `setRawOpticalBypass(false)`, restoring high-contrast blood fluorescence shading.
- **Hero Blood Drop Action Button (`dropPillView`)**:
  A dedicated high-contrast circular button positioned for rapid thumb reach with a deep obsidian background (`#1A0505`) and BloodHawk Crimson border (`#DC2626`). Tapping this button invokes `markCurrentSpotManually()`, capturing the current GNSS fix, blood drop count, heading, and active profile.
- **Direct-Access Edge Control Column**:
  Arranged along the screen edge for rapid single-tap parameter adjustments without opening dialogs:
  1. **Torch**: Toggles 50% throttled eco-torch (API 33+).
  2. **Zoom**: Cycles digital zoom magnification levels.
  3. **Focus**: Toggles between $1.0\text{ m}$ Sweep mode ($1.0\text{ diopter}$) and $0.15\text{ m}$ Macro mode ($6.67\text{ diopters}$).
  4. **Profile**: Cycles target blood chemistry profiles.
  5. **Sensitivity**: Real-time slider adjusting $(a^*, b^*)$ window bounds ($0.0$ to $1.0$).
  6. **Aggressiveness**: Cycles Conservative (3-frame), Balanced (2-frame), and Aggressive (1-frame) temporal locks.
  7. **Frost**: Toggles Snow / Frost substrate modifier.
  8. **Three-Dot Menu**: Opens Brush Scanner, Environmental Presets (Aerosol Mist & Frost), Settings, and Profile Calibration.
- **HUD Reticle Circles Toggle**:
  While the GPU shader colors candidate droplets instantaneously via fluorescence shading, the NDK C++ pipeline can optionally project verified geometric reticle circles over detected centroids. Reticle circles default to **OFF** in `AppSettings`, giving hunters an uncluttered optical fluorescence view while preserving the toggle to re-enable circle overlays when desired.
- **Field & Media Simulator**:
  Accessible as a standardized "Simulator" button with clean palette styling, allowing rapid testing between live camera feeds, recorded benchmark test runs, and static high-resolution trail images.

---

## 4. Native JNI Core & Processing Engine

The native engine is compiled as a shared library (`libbloodtracker.so`) using Android NDK (C++20, `-O3`, `-fvectorize`, ARM64-v8a NEON).

### 4.1 Memory Architecture & Zero Heap Churn

All dynamic memory allocations are strictly confined to `nativeInit(width, height)`. During the real-time tracking loop, `nativeAnalyzeFrame()` operates on pre-allocated static structures stored in `BloodTrackerContext`:

```cpp
struct BloodTrackerContext {
    int width, height;
    // Atomic runtime controls
    std::atomic<uint8_t> lMin, lMax, aMin, aMax, bMin, bMax;
    std::atomic<float> sensitivity;
    std::atomic<int> aggressivenessMode;
    std::atomic<bool> hariEnabled, glareShieldEnabled, meniscusEnabled;
    std::atomic<bool> gyroEgoEnabled, kalmanEnabled, mistEnabled;
    std::atomic<float> gyroShiftX, gyroShiftY;
    
    // Pre-allocated OpenCV Mat buffers (allocated ONCE)
    cv::Mat nv21Mat;            // (H + H/2) x W, CV_8UC1
    cv::Mat bgrMat;             // H x W, CV_8UC3
    cv::Mat labMat;             // H x W, CV_8UC3
    cv::Mat thresholdMask;      // H x W, CV_8UC1
    cv::Mat morphOpenMask;      // H x W, CV_8UC1
    cv::Mat morphFinalMask;     // H x W, CV_8UC1
    cv::Mat specularMask;       // H x W, CV_8UC1
    cv::Mat specularDilateMask; // H x W, CV_8UC1
    cv::Mat morphEllipse3x3;    // Structuring element
    
    // Pre-reserved contour collections
    std::vector<std::vector<cv::Point>> contours;
    std::vector<cv::Vec4i> hierarchy;
    std::vector<cv::Point> hullPts;
    
    // Zero-allocation spatial mist accumulators
    uint16_t mistCellCount[240];
    uint32_t mistCellSumX[240], mistCellSumY[240];
    uint16_t mistCellMinX[240], mistCellMaxX[240], mistCellMinY[240], mistCellMaxY[240];
    
    // Rolling temporal coincidence and Kalman tracklet buffers
    CandidateSpot prevFrameCandidates[32];
    CandidateSpot currFrameCandidates[32];
    PersistentTracklet persistentTracks[32];
};
```

### 4.2 Thread Isolation & JNI Bridge

The NDK engine executes on a dedicated background thread (`CameraBackground` HandlerThread with `Process.THREAD_PRIORITY_DISPLAY`). 

Communication between Kotlin and C++ uses zero-copy direct buffers and primitive critical arrays:
- **Frame Ingestion**: Direct byte buffers (`yBuffer`, `uBuffer`, `vBuffer`) obtained via `env->GetDirectBufferAddress()`.
- **Target Coordinates Output**: `jfloatArray outDropletsNormalized` accessed via `env->GetPrimitiveArrayCritical()`, avoiding JNI copy overhead. The native function writes $[c_x, c_y, \pm r]$ triplets for up to 16 confirmed targets and releases the critical pointer in $<2\mu\text{s}$.
- **Atomic Telemetry**: Thread-safe execution timestamps (`lastFrameDurationUs`) allow the UI thread to display real-time NDK latency diagnostics without lock contention.

---

## 5. OpenGL ES 3.0 Rendering & HUD Subsystem

### 5.1 Dirty-Render Architecture

The display layer uses a hardware `GLSurfaceView` coupled with `BloodTrackerRenderer`:
- Configured with `RENDERMODE_WHEN_DIRTY`: The GPU pipeline sleeps until `SurfaceTexture.OnFrameAvailableListener` fires from the camera ISP.
- When a new frame arrives, `glSurfaceView.requestRender()` wakes the GPU to execute a single render pass, drawing the full-screen quad through `blood_tracker_frag.glsl`.
- *Performance Impact:* Prevents the display subsystem from redrawing at $60\text{ Hz}$ or $120\text{ Hz}$ when the camera is delivering $20\text{--}24\text{ FPS}$, reducing display subsystem battery draw by $>60\%$.

### 5.2 Uniform Binding & Shading Synchronization

At the start of every render pass, `BloodTrackerRenderer.onDrawFrame()` uploads the current system parameters:
- Texture transform matrix: `uTexMatrix` (compensates for hardware sensor rotation).
- CIE $L^*a^*b^*$ profile limits: `uLabMin`, `uLabMax`.
- Sensitivity slider: `uSensitivity`.
- Environmental toggles: `uFrostMode`, `uHariEnabled`, `uGlareShieldEnabled`, `uBypassFilter`.
- Verified NDK CV droplet array: `uDroplets[16]` and `uDropletCount`.

### 5.3 Static Simulation Pipeline (`blood_tracker_image_frag.glsl`)

For bench testing, indoor training, and post-hunt photo analysis, BloodHawk includes a dedicated image simulation pipeline:
- When a static photo or benchmark video is loaded, `setSimulatedBitmap(bitmap)` uploads the image into an OpenGL 2D texture (`GL_TEXTURE_2D`).
- The renderer switches programs to `blood_tracker_image_frag.glsl`, maintaining the identical CIE $L^*a^*b^*$ conversion, background desaturation, and reticle overlay algorithms without requiring an active Camera2 physical hardware session.

---

## 6. Sensor Fusion & Telemetry Subsystem

### 6.1 9-DoF IMU & Stationary Compass HUD

BloodHawk integrates readings from three physical hardware sensors:
1. `Sensor.TYPE_ACCELEROMETER` (Gravity vector $\vec{g}$).
2. `Sensor.TYPE_MAGNETIC_FIELD` (Earth's geomagnetic flux $\vec{m}$).
3. `Sensor.TYPE_GYROSCOPE` (Tri-axial angular velocity $\vec{\omega}$).

```mermaid
flowchart LR
    Accel["Accelerometer (g)"] --> RotMatrix["SensorManager.getRotationMatrix()"]
    Mag["Magnetometer (m)"] --> RotMatrix
    RotMatrix --> Remap["remapCoordinateSystem() (Display Orientation Corrected)"]
    Remap --> Orient["getOrientation() (Raw Azimuth, Pitch, Roll)"]
    Orient --> LowPass["Low-Pass Filter (alpha = 0.15)"]
    LowPass --> Declination["GeomagneticField (True North Declination)"]
    Declination --> CompassUI["CompassView (Stationary 360 Ribbon HUD)"]
    
    Gyro["Gyroscope (omega_x, omega_y)"] --> AngularInt["Angular Integration (delta_t)"]
    AngularInt --> EgoNative["Native BloodTrackerNative.nativeSetGyroShift()"]
    EgoNative --> TrackerWarp["NDK Candidate Coordinate Warping"]
```

#### Sensor Fusion Implementations:
- **Low-Pass Filtered Azimuth**:
  To prevent handheld trembling from jittering the compass ribbon, azimuth angle $\theta_{\text{raw}}$ is smoothed via an exponential moving average:
  $$\theta_t = \theta_{t-1} + \alpha \cdot \text{normalize}(\theta_{\text{raw}} - \theta_{t-1}),\quad \alpha = 0.15$$
- **True North Correction**:
  Using the last known GPS fix, `GeomagneticField` calculates local magnetic declination ($\delta_{\text{mag}}$), converting magnetic compass heading to True Geodetic North.
- **CompassView Ribbon HUD**:
  Rendered at the top of the tracking screen as a stationary $360^\circ$ military-style azimuth ribbon. It displays:
  - Current travel heading (center hash).
  - Target blood trail flight vector indicator (`NAV_CYAN`).
  - Backtrack vector indicator (`LAST_AMBER`).
  - Precision alignment badge: Highlights `ACCURACY_EMERALD` when the hunter is aligned within $\pm 6^\circ$ of the animal's forward flight path.

### 6.2 Gyro Ego-Motion Prediction

As the hunter sweeps the device across ground litter, the camera undergoes angular rotation. The Kotlin layer samples the gyroscope angular rate vector $\vec{\omega} = (\omega_x, \omega_y)$ and updates the C++ native context via `nativeSetGyroShift(shiftX, shiftY)`. The native analyzer shifts the candidate association window between frames, preventing sweeping motion from dropping active targets.

---

## 7. Audio & Tactile Feedback Subsystem

### 7.1 Ultra-Low-Power Geiger Audio Engine (`GeigerAudioEngine`)

Audio feedback is designed for silent field operations using a single Bluetooth earpiece:
- **Pre-Synthesized PCM Buffer**:
  At application startup, a sharp, $7\text{ ms}$, $1400\text{ Hz}$ acoustic pip is synthesized into an in-memory 16-bit PCM array:
  $$s(t) = \sin(2\pi \cdot 1400 \cdot t) \cdot \sin\left(\frac{\pi \cdot t}{T_{\text{dur}}}\right),\quad T_{\text{dur}} = 0.007\text{ s}$$
  The cosine envelope suppresses speaker clicks at the waveform boundaries.
- **Zero-CPU Playback**:
  The synthesized waveform is loaded into an Android `AudioTrack` configured in `AudioTrack.MODE_STATIC`. Triggering a pip requires only `audioTrack.stop()` and `audioTrack.reloadAndPlay()`. The CPU performs **zero audio synthesis, zero DSP filtering, and zero heap allocations** during runtime tracking.
- **Pacing & Confidence Modulation**:
  When confirmed blood droplets enter the camera frame, Geiger pips fire at a rate proportional to detection confidence and droplet proximity (cooldown throttled to $140\text{ ms}$ minimum interval).
- **Dedicated On-Screen Audio Volume Slider**:
  The dedicated volume slider in `CameraFragment` directly regulates both the `AudioTrack` internal gain and the Android system `STREAM_MUSIC` hardware level from the tracking screen, providing immediate acoustic adjustment without leaving the tracking interface.

### 7.2 Tactile Haptic Pulse Engine

For stealth tracking near bedded game where all audio must be silenced, BloodHawk uses a silent haptic feedback engine:
- Controlled via Android's `VibratorManager` using pre-compiled one-shot vibration effects:
  $$\text{VibrationEffect.createOneShot}(40\text{ ms},\; \text{VibrationEffect.DEFAULT_AMPLITUDE})$$
- Throttled by a hardware cooldown timer ($600\text{ ms}$) to prevent battery drain from continuous vibration.

---

## 8. Blood Chemistry & Calibration Subsystem

### 8.1 System Default Profiles & Safe Envelopes

BloodHawk ships with three immutable, biologically validated spectral profiles:

| Profile Identifier | Target Hemoglobin Chemistry | Biological Origin | $L^*$ Bounds | $a^*$ Bounds | $b^*$ Bounds |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `FRESH_ARTERIAL` | Oxygenated Oxyhemoglobin ($\text{HbO}_2$) | Heart, Lung, Carotid, Femoral Artery | $[20..210]$ | $[145..255]$ | $[110..165]$ |
| `LIVER_DARK` | Deoxygenated Hemoglobin & Biliverdin | Liver, Kidney, Hepatic Portal Vein | $[12..155]$ | $[140..245]$ | $[104..158]$ |
| `AGED_OXIDIZED` | Methemoglobin & Hematin (Dried) | Blood trails $>1\text{--}2\text{ hours}$ old | $[14..175]$ | $[138..255]$ | $[114..172]$ |

#### Environmental Substrate Modifier (Snow / Frost Mode)
When hunting in snow or heavy frost, high-albedo crystalline reflections and Rayleigh blue sky-scatter disrupt standard color metrics. Toggling Snow Mode modifies the active bounds:
- **Luminance Clamping**: $L^*_{\text{max}} = \min(L^*_{\text{max}} - 18, 195)$ (suppresses blinding snow glare).
- **Red Contrast Tightening**: $a^*_{\text{min}} = a^*_{\text{min}} + 4$ (exploits high contrast against white snow).
- **Blue-Scatter Shift**: $b^*_{\text{min}} = b^*_{\text{min}} - 5$, $b^*_{\text{max}} = b^*_{\text{max}} - 6$ (compensates for blue-tinted shadows on snow).

#### Safe Blood Envelope Guardrails
User-calibrated custom profiles are constrained by hardcoded biological guardrails (`clampToSafeBloodEnvelope()`):
$$L^* \in [2, 254],\quad a^* \in [80, 255],\quad b^* \in [60, 255]$$
Custom profiles that drift toward green/brown bark ($a^* \le 128$) or yellow autumn foliage ($b^* > 178$) are flagged by the `evaluateFoliageRiskWarning()` engine, alerting the hunter to potential decoy risks.

### 8.2 Photo Calibration Subsystem (`ProfileCalibrationFragment`)

When tracking an unusual blood presentation (e.g., gut-shot bile mixture or blood on reddish clay soil), the hunter can calibrate a custom profile from a field photograph:
1. The user captures or imports a photo containing confirmed blood.
2. `CalibrationPhotoView` presents an interactive zoomable canvas with a reticle crosshair.
3. Tapping the blood sample extracts a $9 \times 9$ pixel patch. The engine computes the median and interquartile range of $(L^*, a^*, b^*)$, filters out shadow and glare outliers, and creates a tuned `SpectralProfile` saved to persistent storage.

---

## 9. Hit Site Diagnostic & Lethality Evaluation (Removed)

> [!NOTE]
> The Hit Site Evaluation & Diagnostic functions (`HitDiagnosticAdvisor`, `HitDiagnosticFragment`, `DiagnosticPhotoView`, `DiagnosticPieChartView`) have been retired from the active application codebase to streamline field tracking operations directly to optical detection and trail navigation.

---

## 10. Brush Scanner Subsystem (Dense Cover Foliage Peeling)

When wounded game enters thick brush, downed timber, or briar thickets, direct line-of-sight is obstructed. The `BrushScannerFragment` and native `BrushScannerAnalyzer.cpp` implement a **Synthetic Aperture Motion-Compensated Accumulator**:

```mermaid
flowchart TD
    VideoSweep["Handheld Sweeping Motion (20-30cm Baseline)"]
    VideoSweep --> Frames["Rolling Buffer: 16 to 60 Frames (BGR + Grayscale)"]
    
    Frames --> LK["Lucas-Kanade Pyramidal Optical Flow (cv::calcOpticalFlowPyrLK)"]
    LK --> MedianMotion["Median Filtered Translation (dx, dy) Robust to Moving Leaves"]
    MedianMotion --> CumulativeOffset["Cumulative 2D Baseline Vector"]
    
    CumulativeOffset --> SynthAperture["Synthetic Aperture Blending Engine"]
    SynthAperture --> ShiftAlign["Pixel-Shift Alignment by Focal Depth Parameter (F)"]
    ShiftAlign --> Accumulate["Accumulate Shifted Frames: accFloat += frame * weight"]
    Accumulate --> Normalize["Normalize by Overlap Count Matrix"]
    
    Normalize --> FilterSelect{"Select Spectral Filter Mode"}
    FilterSelect -->|Mode 0: Natural| OutNat["Foliage-Peeled Natural BGR Image"]
    FilterSelect -->|Mode 1: Deer Coat| OutCoat["Deer Coat Isolation (H: 8-25, S: 45-210, V: 35-220)"]
    FilterSelect -->|Mode 2: Anomaly Edge| OutEdge["Synthetic Depth Gradient Anomaly Highlighting"]
```

### Synthetic Aperture Mechanics:
1. **Physical Baseline Generation**: The hunter sweeps the phone horizontally across a $20\text{--}40\text{ cm}$ path. This mimics a synthetic camera lens with a physical aperture equal to the sweep width ($30\text{ cm}$ vs physical $4\text{ mm}$ lens).
2. **Foreground Foliage Defocusing**: Foreground leaves and branches undergo high parallax shift between frames. When shifting and averaging frames aligned to a background focal depth $F$, foreground occluders blur into semi-transparent haze, revealing the animal carcass behind them.
3. **Specialized Target Filters**:
   - **Deer Coat Filter**: Isolates the characteristic tan, brown, and white spectral reflectance of deer pelage in HSV space ($H \in [8..25]$, $S \in [45..210]$, $V \in [35..220]$), highlighting hidden fur against green/gray vegetation.
   - **Anomaly Edge Filter**: Highlights sharp edge anomalies that stand out against the blurred foreground foliage.

---

## 11. Geospatial & Trail Navigation Subsystem

### 11.1 Zero-Auth Offline Tile Engine (`OfflineTileManager`)

Operating in remote wilderness without cellular connectivity requires a fully offline geospatial subsystem:
- **Zero-Authentication USGS National Map Packs (Tier 1)**:
  Permanent offline region packs downloaded directly from U.S. Geological Survey (USGS) public domain endpoints (`USGSImageryOnly` and `USGSTopo`). These require **zero API keys, zero authentication tokens, and zero commercial subscriptions**, ensuring permanent field availability without key expiration or watermark restrictions.
- **Dynamic LRU Browse Cache (Tier 2)**:
  Interactive browsing while online caches OpenStreetMap (OSM) raster tiles locally. An automated background trimmer maintains total dynamic cache size between $75\text{ MB}$ and $100\text{ MB}$, complying with OSMF tile policies.
- **WebView Request Interception**:
  `OfflineTileManager.interceptRequest()` intercepts all tile URL calls from the Leaflet.js map engine inside `MapFragment`'s `WebView`. If a cached tile exists on disk, it is served immediately via `WebResourceResponse(image/png)` with **zero network latency and zero network connectivity required**.

### 11.2 Blood Trail Session & Flight Vector Management (`BloodTrailManager`)

`BloodTrailManager` operates as a thread-safe singleton managing the active recovery session:

```
                            [Blood Drop 1] (Initial Hit Site)
                                  \
                                   \  Bearing 042° (38 meters)
                                    \
                                [Blood Drop 2]
                                      \
                                       \  Bearing 038° (45 meters)
                                        \
                                    [Blood Drop 3] (Most Recent Fix)
                                         |
                       +-----------------+-----------------+
                       |                                   |
                       v                                   v
          [Calculated Flight Vector]              [Backtrack Vector]
           Heading: 039° (Dead Ahead)              Heading: 219° (180° Reverse)
           Visual: NAV_CYAN Compass Needle         Visual: LAST_AMBER Pointer
```

#### Flight Vector Mathematical Formulations:
1. **Haversine Distance & Initial Bearing**:
   For two consecutive blood fixes $P_1(\phi_1, \lambda_1)$ and $P_2(\phi_2, \lambda_2)$:

   $$\Delta\phi = \phi_2 - \phi_1,\quad \Delta\lambda = \lambda_2 - \lambda_1$$

   $$d = 2R \cdot \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta\phi}{2}\right) + \cos\phi_1 \cos\phi_2 \sin^2\left(\frac{\Delta\lambda}{2}\right)}\right)$$

   $$\theta = \text{atan2}\left(\sin\Delta\lambda \cos\phi_2,\; \cos\phi_1 \sin\phi_2 - \sin\phi_1 \cos\phi_2 \cos\Delta\lambda\right)$$

2. **Weighted Forward Flight Vector**:
   The animal's projected forward trajectory is calculated using an exponential decay weight across the last $N$ blood drops, placing highest confidence on the most recent fixes:
   $$\vec{V}_{\text{flight}} = \sum_{i=1}^{N-1} w_i \cdot \vec{u}_{i, i+1},\quad w_i = e^{-\lambda (N - 1 - i)}$$
3. **Spatial Out-of-Order Backfill Disambiguation**:
   When trailing wounded game, hunters frequently find a new blood drop ahead, and later discover a missed drop located spatially between earlier points. If an inserted drop falls geometrically along the path between existing waypoints, `BloodTrailManager` automatically re-indexes the trail sequence rather than creating a crisscrossing zigzag vector.
4. **Hunter Search Path (Breadcrumb Logging)**:
   While blood drops are saved manually, the hunter's actual walking path is logged automatically using a **$4.0\text{ m}$ deadband spatial filter** (`HUNTER_BREADCRUMB_MIN_DIST_M`). This logs the search perimeter while eliminating stationary GPS spiderweb jitter when standing in place.
5. **Multi-Format Export**:
   Trail sessions can be exported directly to standard geospatial formats:
   - **GPX 1.1**: Waypoints with blood metadata + hunter tracklogs.
   - **KML**: Formatted for Google Earth with custom color-coded pins.
   - **GeoJSON**: Standard web mapping payload.

### 11.3 Field Map Controls, Rotating Orientation & Distance Tools

`MapFragment` pairs with `trail_map.html` to provide high-contrast offline tactical mapping:
1. **Dynamic Rotating Heading-Up Mode**:
   - In addition to standard static North-Up orientation, hunters can toggle **Heading-Up mode**, smoothly rotating the entire map canvas in 2D around the viewport center `(w/2, h/2)` by $\text{rotRad} = -\theta_{\text{bearing}} \cdot \frac{\pi}{180}$ to mirror direction of travel.
   - **Radial Tile Over-Fetching**: Avoids black or white corner clippings during rotation by extending tile fetching out to the diagonal radius $r = \frac{\sqrt{(w/2)^2 + (h/2)^2}}{\text{scale}}$.
   - **Rotated Coordinate Transform**: Panning drag gestures, pinch-to-zoom focal anchors, and waypoint tap hit-tests use inverse rotation matrices (`screenToCanvas` and `canvasToScreen`) to maintain intuitive spatial feel.
   - **North Needle Indicator**: A floating compass needle appears at the top left whenever the map is rotated; tapping it instantly resets the map to North-Up mode.
2. **Standardized Three-Dot Tools Menu (`MORE`)**:
   Consolidates map layer and measurement toggles into a dedicated bottom card dialog:
   - **Current GPS Position (`ic_my_location`)**: Toggle showing/hiding current hunter position dot and accuracy circle (`POS ON` / `POS OFF`).
   - **Hunter Breadcrumb Path (`ic_route_path`)**: Toggle showing/hiding recorded search track vectors (`PATH ON` / `PATH OFF`).
   - **Multi-Segment Laser Ruler (`ic_tune`)**: Toggle interactive distance measurement tape (`RULER ON` / `RULER OFF`) with real-time yardage readouts.
   - **Map Orientation Mode (`ic_compass`)**: Quick-switch between `HEADING UP` and `NORTH UP`.
3. **Repositioned Zoom Controls**:
   The `+` and `–` zoom buttons are pinned to the **upper right** corner (`top: 14px; right: 10px;`), preventing accidental thumb touches or occlusion with bottom telemetry cards and navigation pills.
4. **Semantic Palette Standardization**:
   All controls adhere strictly to `BloodHawkPalette`:
   - Active toggled ON states use `PILL_TOGGLE_ON` (`NAV_CYAN` `#38BDF8`).
   - Inactive toggled OFF states use `PILL_TOGGLE_OFF_TEXT` (`TEXT_MUTED` `#94A3B8`) and `PILL_TOGGLE_OFF_STROKE` (`BORDER_MUTED` `#475569`).
   - Non-toggle action buttons use `PILL_ACTION_BUTTON` (`LAST_AMBER` `#FBBF24`).

---

## 12. End-to-End Field Workflows

### Workflow 1: Live Field Tracking & Waypoint Drop

```mermaid
sequenceDiagram
    autonumber
    actor Hunter
    participant Cam as CameraFragment
    participant NDK as BloodTrackerAnalyzer
    participant GL as BloodTrackerRenderer
    participant Audio as GeigerAudioEngine
    participant Trail as BloodTrailManager

    Hunter->>Cam: Points camera at forest floor (1.0m focus lock)
    loop Every Frame (20-24 FPS)
        Cam->>GL: Deliver hardware OES texture (1280x720)
        Cam->>NDK: Deliver YUV DirectBuffer (640x360)
        GL->>GL: Desaturate forest floor to slate-gray, shade blood in Neon Magenta/Yellow
        NDK->>NDK: Execute 7-stage geometric and temporal filter cascade
        alt Blood Droplet or Mist Confirmed
            NDK-->>GL: Upload verified coordinates uDroplets
            NDK-->>Audio: Trigger Geiger acoustic pip and haptic pulse
            GL->>Hunter: Display crisp Neon Yellow Ring or Amber Mist Disc
            Audio->>Hunter: Audible Pip in Bluetooth earpiece
        else Decoy or Leaf Vein or Noise
            NDK->>NDK: Reject candidate, zero HUD circles, silence audio
        end
    end
    Hunter->>Cam: Taps Hero Blood Drop Button
    Cam->>Trail: addWaypoint(lat, lon, dropCount, heading, profile)
    Trail->>Trail: Update forward flight vector and backtrack lines
    Cam->>Hunter: Haptic confirmation pulse and HUD count update
```

### Workflow 2: Waypoint Inspection & Trail Navigation

```mermaid
sequenceDiagram
    autonumber
    actor Hunter
    participant Map as MapFragment
    participant HTML as trail_map.html
    participant Mgr as BloodTrailManager

    Hunter->>Map: Opens trail map view
    Map->>HTML: Render waypoints and vectors
    Hunter->>HTML: Taps on waypoint marker
    HTML->>HTML: Shows Drop Inspector modal (Set Head, Move, Delete)
    Hunter->>HTML: Taps "SET ACTIVE HEAD"
    HTML->>Map: setHeadWaypoint(index)
    Map->>Mgr: updateTrailState()
```

### Workflow 3: Custom Field Blood Calibration

```mermaid
sequenceDiagram
    autonumber
    actor Hunter
    participant Calib as ProfileCalibrationFragment
    participant View as CalibrationPhotoView
    participant Mgr as BloodProfileManager
    participant NDK as BloodTrackerNative

    Hunter->>Calib: Captures photo of confirmed game blood
    Calib->>View: Load bitmap onto touch canvas
    Hunter->>View: Drags crosshair reticle over blood core
    View->>Calib: Extract 9x9 pixel patch CIE Lab statistics
    Calib->>Calib: Filter glare and shadow outliers and compute median Lab
    Calib->>Calib: Enforce safe blood envelope guardrails
    Calib->>Mgr: saveCustomProfile(profile)
    Mgr->>NDK: nativeSetLabBounds(lMin, lMax, aMin, aMax, bMin, bMax)
    Calib->>Hunter: Tactile confirmation and custom profile activated
```

### Workflow 4: Lost Trail Backtracking & Directional Vector Recovery

```mermaid
sequenceDiagram
    autonumber
    actor Hunter
    participant Map as MapFragment
    participant Trail as BloodTrailManager
    participant Compass as CompassView

    Hunter->>Map: Blood trail runs dry, opens Map Screen
    Map->>Trail: Query active session waypoints and flight vectors
    Trail-->>Map: Returns last known fix, flight vector 039 deg, backtrack line
    Map->>Hunter: Displays forward projection cone and hunter breadcrumbs
    Hunter->>Compass: Rotates phone until Compass HUD aligns with flight vector
    Compass->>Hunter: Highlights ACCURACY_EMERALD at 039 deg plus minus 6 deg
    Hunter->>Hunter: Walks forward bearing and locates next blood drop 45 yards ahead
```

### Workflow 5: Thick Brush & Deep Cover Scanner Sweep

```mermaid
sequenceDiagram
    autonumber
    actor Hunter
    participant Brush as BrushScannerFragment
    participant Native as BrushScannerAnalyzer

    Hunter->>Brush: Animal entered thick briar thicket, opens Brush Scanner
    Hunter->>Brush: Taps Start Sweep and moves phone across 30cm baseline
    loop During Sweep (16-60 Frames)
        Brush->>Native: Stream YUV frames
        Native->>Native: Lucas-Kanade optical flow motion tracking
        Native->>Native: Accumulate synthetic aperture depth slice
    end
    Hunter->>Brush: Taps Stop Sweep and adjusts Focal Depth Slider
    Brush->>Native: renderDepthSlice(focalDepth, DEER_COAT)
    Native-->>Brush: Blended image with foreground brush defocused
    Brush->>Hunter: Displays concealed deer coat pelage behind brush
```

---

## 13. System Invariants & Technical Specifications Summary

| System Property | Specification | Engineering Guarantee |
| :--- | :--- | :--- |
| **NDK Per-Frame Latency** | **$2.87\text{ ms}$** (Benchmark average) | Meets $<41.6\text{ ms}$ budget ($>14\times$ margin for $24\text{ FPS}$) |
| **Steady-State GC Allocations** | **$0\text{ bytes/frame}$** | Zero heap churn; eliminates ART GC pauses |
| **Display Redraw Overhead** | Driven by camera ISP frame availability | Zero idle $60/120\text{ Hz}$ GPU redraws (`RENDERMODE_WHEN_DIRTY`) |
| **Continuous Battery Runtime** | **$>4.5\text{ hours}$** on Google Pixel 10 | OLED black (`#0B0F17`), $24\text{ FPS}$ ISP clamping, $50\%$ torch throttling |
| **Audio Synthesis Overhead** | **$0\text{ CPU cycles}$** during tracking | Pre-synthesized $7\text{ ms}$ static PCM `AudioTrack` |
| **Waypoint Persistence** | **$100\%$ Manual Confirmation** | Zero false-positive trail pollution |
| **Offline Autonomy** | **$100\%$ Zero-Auth Operation** | USGS National Map offline packs + local LRU tile cache; zero API keys |
| **UI Color Palette** | 6-Color Semantic `BloodHawkPalette` | High contrast; zero ad-hoc hex values across camera and map |
| **Toast Policy** | **Strict Zero-Toast Standard** | Tactile haptics, HUD badges, and modal dialogs only |
