# Cat Visual Location Tracker — Updated Implementation Plan

## 1. Project Goal

Build a self-hosted Docker Compose application that continuously determines the latest visible location of a cat using TP-Link Tapo C100 RTSP cameras and YOLO object detection.

The PRIMARY output is NOT merely a room name.

The PRIMARY output is:

```text
The latest visual confirmation of the cat.
```

The system should always attempt to answer:

```text
Where is the cat right now?
Show me the latest picture.
```

or:

```text
Where was the cat last seen?
Show me the last known picture.
```

The image becomes the primary state object.

Room information is metadata attached to the image.

---

## 2. Core Product Philosophy

The application should be:

```text
visual-first
```

NOT:

```text
room-first
```

Humans do not actually want:

```text
Bedroom
```

Humans want:

```text
The cat was last seen in the bedroom 12 seconds ago.
[show image]
```

Therefore:

- snapshots are NOT debugging artifacts
- snapshots ARE the core application state

---

## 3. Main User Experience

## Dashboard Example

### Cat currently visible

```text
Current Location
┌─────────────────────────────┐
│ [latest cat image]          │
│ Bedroom                     │
│ Seen 12 seconds ago         │
│ Confidence: 0.92            │
│ Camera: bedroom_cam         │
└─────────────────────────────┘
```

### Cat not currently visible

```text
Last Known Location
┌─────────────────────────────┐
│ [last cat image]            │
│ Kitchen                     │
│ Seen 8 minutes ago          │
│ Camera: kitchen_cam         │
└─────────────────────────────┘
```

The image is the most important part of the UI.

---

## 4. Core Architecture

```text
RTSP Cameras
    |
    v
Periodic Snapshot Capture
    |
    v
YOLO GPU Inference
    |
    v
Detection Selection
    |
    v
Best Snapshot Selection
    |
    v
Visual State Manager
    |
    +--------------------+
    |                    |
    v                    v
MQTT / HA          Dashboard/API
```

---

## 5. Key Design Change

The previous architecture treated snapshots as debugging output.

This updated architecture treats snapshots as:

```text
the canonical source of truth
```

The state manager should store:

- latest image
- latest crop
- latest annotated image
- latest room
- latest timestamp
- latest camera
- latest confidence

---

## 6. Primary State Object

The central application state should look like:

```json
{
  "currently_visible": true,
  "current_detection": {
    "room": "Bedroom",
    "camera_id": "bedroom_cam",
    "camera_name": "Bedroom Camera",
    "timestamp": "2026-05-16T03:12:44-04:00",
    "confidence": 0.92,

    "raw_snapshot_path": "/data/snapshots/raw/...",
    "annotated_snapshot_path": "/data/snapshots/annotated/...",
    "crop_snapshot_path": "/data/snapshots/crops/...",
    "thumbnail_path": "/data/snapshots/thumbnails/...",

    "raw_snapshot_url": "/snapshots/raw/...",
    "annotated_snapshot_url": "/snapshots/annotated/...",
    "crop_snapshot_url": "/snapshots/crops/...",
    "thumbnail_url": "/snapshots/thumbnails/...",

    "bbox": [120, 80, 300, 260]
  },

  "last_known_detection": {
    "room": "Bedroom",
    "camera_id": "bedroom_cam",
    "camera_name": "Bedroom Camera",
    "timestamp": "2026-05-16T03:12:44-04:00",
    "confidence": 0.92,

    "raw_snapshot_path": "...",
    "annotated_snapshot_path": "...",
    "crop_snapshot_path": "...",
    "thumbnail_path": "...",

    "raw_snapshot_url": "...",
    "annotated_snapshot_url": "...",
    "crop_snapshot_url": "...",
    "thumbnail_url": "...",

    "bbox": [120, 80, 300, 260]
  }
}
```

---

## 7. Visual Asset Types

For every accepted cat detection, generate:

### 1. Raw frame

Full original frame.

Example:

```text
/data/snapshots/raw/
```

### 2. Annotated frame

Full frame with bounding box overlay.

Example:

```text
/data/snapshots/annotated/
```

### 3. Cropped cat image

Only the cat region.

This is EXTREMELY important for mobile notifications and dashboard readability.

Example:

```text
/data/snapshots/crops/
```

### 4. Thumbnail

Smaller preview image.

Example:

```text
/data/snapshots/thumbnails/
```

---

## 8. Why Cropped Images Matter

Without crop:

```text
┌──────────────────────┐
│ room furniture       │
│                      │
│      tiny cat        │
│                      │
└──────────────────────┘
```

With crop:

```text
┌────────────┐
│    CAT     │
└────────────┘
```

The crop should be the primary image used in:

- dashboard cards
- Home Assistant picture cards
- mobile notifications
- future push notifications

---

## 9. Best Snapshot Selection

The newest frame is NOT always the best frame.

The system should select the best snapshot using weighted scoring.

---

## 10. Snapshot Quality Score

Each detection should receive a score.

Suggested formula:

```text
score =
    confidence_weight * confidence
    + bbox_weight * normalized_bbox_area
    + sharpness_weight * image_sharpness
```

Optional penalties:

- blurry frame penalty
- motion blur penalty
- low brightness penalty

---

## 11. Recommended Initial Snapshot Scoring

```yaml
snapshot_selection:
  enabled: true

  weights:
    confidence: 0.5
    bbox_area: 0.3
    sharpness: 0.2

  minimum_bbox_area_ratio: 0.01
  minimum_confidence: 0.60
```

---

## 12. Snapshot Replacement Rules

The latest detection should replace the current visible snapshot only if:

- it is newer
- AND not significantly worse

Avoid rapid replacement with blurry images.

Example logic:

```text
If newer frame has:
    lower confidence
    much smaller bbox
    lower sharpness

Then keep existing snapshot.
```

This creates MUCH better UX.

---

## 13. Current vs Last Known State

## Current visible state

A cat is considered currently visible if:

```yaml
current_visibility_timeout_seconds: 30
```

If the latest detection is older than 30 seconds:

```json
{
  "currently_visible": false
}
```

BUT:

```text
last_known_detection
```

must remain forever until replaced.

---

## 14. Dashboard Requirements

The dashboard is now image-centric.

---

## 15. Dashboard Layout

```text
Header
--------------------------------

Current Detection Card
--------------------------------
Large image
Room
Time ago
Confidence
Camera

Last Known Detection Card
--------------------------------
Large image
Room
Time ago
Confidence
Camera

Recent Detection Timeline
--------------------------------
image | room | timestamp

Camera Health Table
--------------------------------
camera | room | status | last frame | last error

Diagnostics
--------------------------------
GPU
MQTT
Inference Queue
Model
Warnings
```

---

## 16. Dashboard Image Priority

Use:

```text
1. crop image
2. annotated image
3. raw image
```

for primary UI display.

---

## 17. Dashboard Routes

```text
GET /
GET /api/state
GET /api/current
GET /api/last-known
GET /api/recent
GET /api/cameras
GET /api/diagnostics

GET /snapshots/raw/<path>
GET /snapshots/annotated/<path>
GET /snapshots/crops/<path>
GET /snapshots/thumbnails/<path>

GET /health
GET /ready
GET /metrics
```

---

## 18. Detection Pipeline

```text
RTSP worker
    |
    v
Frame capture
    |
    v
Inference queue
    |
    v
YOLO detection
    |
    v
Cat detection extraction
    |
    v
Image quality scoring
    |
    v
Snapshot generation
    |
    v
Visual state manager
```

---

## 19. Camera Strategy

Use periodic snapshots only.

Default:

```yaml
snapshot_interval_seconds: 2
```

The system does NOT need continuous video inference.

---

## 20. Model Strategy

Use:

- YOLO11n
- YOLO11s

The config must support runtime switching.

---

## 21. GPU Requirement

GPU ONLY.

If CUDA is unavailable:

- mark system unhealthy
- stop inference
- show dashboard warning
- publish MQTT diagnostics

Do NOT silently fall back to CPU.

---

## 22. RTSP Strategy

Use OpenCV RTSP ingestion.

Each camera worker must:

- reconnect automatically
- detect stale streams
- detect repeated failures
- expose diagnostics

---

## 23. Snapshot Storage Layout

```text
/data/snapshots/
├── raw/
├── annotated/
├── crops/
├── thumbnails/
├── current/
└── latest/
```

---

## 24. Special Current Snapshot Symlinks

Maintain stable symlinks or copies:

```text
/data/snapshots/current/current_raw.jpg
/data/snapshots/current/current_crop.jpg
/data/snapshots/current/current_annotated.jpg

/data/snapshots/latest/latest_raw.jpg
/data/snapshots/latest/latest_crop.jpg
/data/snapshots/latest/latest_annotated.jpg
```

This greatly simplifies:

- Home Assistant
- notifications
- dashboard caching
- mobile apps

---

## 25. MQTT Philosophy

MQTT should publish:

- metadata
- image URLs

NOT binary image payloads.

---

## 26. MQTT Topic Structure

```text
quiet98k/cat_tracker/state
quiet98k/cat_tracker/current
quiet98k/cat_tracker/last_known

quiet98k/cat_tracker/diagnostics
quiet98k/cat_tracker/availability

quiet98k/cat_tracker/cameras/<camera_id>/state
```

---

## 27. MQTT Current Detection Payload

```json
{
  "currently_visible": true,

  "room": "Bedroom",
  "camera_id": "bedroom_cam",
  "camera_name": "Bedroom Camera",

  "timestamp": "2026-05-16T03:12:44-04:00",

  "confidence": 0.92,

  "raw_snapshot_url": "/snapshots/current/current_raw.jpg",
  "annotated_snapshot_url": "/snapshots/current/current_annotated.jpg",
  "crop_snapshot_url": "/snapshots/current/current_crop.jpg",

  "bbox": [120, 80, 300, 260]
}
```

---

## 28. Home Assistant Integration

Use MQTT Discovery.

---

## 29. Home Assistant Entities

### Main entities

```text
sensor.cat_current_room
sensor.cat_last_known_room
sensor.cat_last_seen

binary_sensor.cat_currently_visible
```

### Image entities

```text
camera.cat_current_snapshot
camera.cat_current_crop
camera.cat_current_annotated
```

### Diagnostic entities

```text
sensor.cat_tracker_gpu_status
sensor.cat_tracker_active_model
sensor.cat_tracker_queue_size
sensor.cat_tracker_last_error
```

---

## 30. Home Assistant UI Goal

The desired Home Assistant card should look like:

```text
Cat Tracker
--------------------------------

[cat image]

Current Room: Bedroom
Seen: 12 seconds ago
Confidence: 0.92
```

---

## 31. Recommended HA Cards

Use:

- Picture Entity Card
- Markdown Card
- Mushroom Cards
- Conditional Card

---

## 32. Notification Architecture

The architecture should prepare for future notifications.

Future examples:

```text
Cat entered office
[image]
```

or:

```text
Cat has not been seen for 4 hours
Last known:
[image]
```

---

## 33. Visual State Manager

The VisualStateManager replaces the old room-centric StateManager.

Responsibilities:

- maintain latest visible detection
- maintain last known detection
- manage snapshot references
- manage image URLs
- apply timeout logic
- select best snapshot
- persist latest state

---

## 34. Persistence

Persist:

```text
/data/state/latest_state.json
```

including snapshot paths.

---

## 35. Persistence Requirements

After restart:

- latest image should still appear
- last known room should still appear
- latest crop should still exist

---

## 36. API Contracts

## GET /api/current

```json
{
  "currently_visible": true,

  "room": "Bedroom",
  "camera_name": "Bedroom Camera",

  "timestamp": "2026-05-16T03:12:44-04:00",

  "confidence": 0.92,

  "crop_snapshot_url": "/snapshots/current/current_crop.jpg",
  "annotated_snapshot_url": "/snapshots/current/current_annotated.jpg",
  "raw_snapshot_url": "/snapshots/current/current_raw.jpg"
}
```

---

## 37. GET /api/last-known

```json
{
  "room": "Kitchen",
  "camera_name": "Kitchen Camera",

  "timestamp": "2026-05-16T02:51:22-04:00",

  "confidence": 0.87,

  "crop_snapshot_url": "/snapshots/latest/latest_crop.jpg",
  "annotated_snapshot_url": "/snapshots/latest/latest_annotated.jpg",
  "raw_snapshot_url": "/snapshots/latest/latest_raw.jpg"
}
```

---

## 38. Snapshot Generation Pipeline

For every accepted detection:

```text
1. save raw image
2. draw bbox overlay
3. generate annotated image
4. crop bbox region
5. generate thumbnail
6. update current snapshot symlinks
7. update latest snapshot symlinks
8. publish MQTT update
9. update dashboard state
```

---

## 39. Recent Timeline

Maintain recent detection metadata.

Example:

```json
[
  {
    "room": "Bedroom",
    "timestamp": "...",
    "crop_snapshot_url": "...",
    "confidence": 0.92
  }
]
```

Keep only:

```yaml
recent_detection_limit: 100
```

No full database needed yet.

---

## 40. Docker Compose

Still use Docker Compose.

No architecture change needed.

---

## 41. CI/CD

Still use GitHub Actions.

No architecture change needed.

---

## 42. New Recommended Directory Structure

```text
/data/
├── state/
│   └── latest_state.json
├── snapshots/
│   ├── raw/
│   ├── annotated/
│   ├── crops/
│   ├── thumbnails/
│   ├── current/
│   └── latest/
└── logs/
```

---

## 43. New Internal Models

## VisualDetection

```python
class VisualDetection(BaseModel):
    room: str
    camera_id: str
    camera_name: str

    timestamp: datetime

    confidence: float

    bbox: tuple[int, int, int, int]

    raw_snapshot_path: str
    annotated_snapshot_path: str
    crop_snapshot_path: str
    thumbnail_path: str

    raw_snapshot_url: str
    annotated_snapshot_url: str
    crop_snapshot_url: str
    thumbnail_url: str
```

---

## 44. New VisualState

```python
class VisualState(BaseModel):
    currently_visible: bool

    current_detection: VisualDetection | None

    last_known_detection: VisualDetection | None

    updated_at: datetime
```

---

## 45. Recommended Development Order

1. RTSP workers
2. GPU inference
3. crop generation
4. snapshot generation
5. VisualStateManager
6. dashboard image rendering
7. MQTT discovery
8. HA integration
9. diagnostics
10. CI/CD

---

## 46. Most Important UX Rule

The user should NEVER need to ask:

```text
Which room is my cat in?
```

and then separately open cameras.

The answer should already include:

```text
room + image
```

together.

---

## 47. Final Recommendation

The project should think of itself as:

```text
a visual cat finder
```

NOT:

```text
a room occupancy sensor
```

That single mindset change should drive all architecture decisions.
