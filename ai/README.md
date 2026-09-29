# Nexus AI: CCTV Intrusion Detection

The AI side of the Nexus security platform. It reads video from any number of cameras, detects people with a fine-tuned YOLOv8s model, decides whether the activity is suspicious, and sends alerts (with screenshots) to the backend.

The backend (NestJS) lives in a separate project. This repo contains only the AI services.

## How it works

```
Cameras (RTSP / USB / Pi Camera / HTTP)
        |
   edge-gateway        reads cameras, filters motion, pushes frames
        |
      Queue            Redis Streams
        |
 inference-worker      YOLOv8s detection (run N copies to scale)
        |
      Queue
        |
   rules-engine        tracking, zones, dwell time, cooldown
        |
     Backend           alerts + screenshots, feedback, live stream
```

Cameras and workers never talk directly. The queue sits between them, so adding cameras only means adding workers.

## Project structure

```
ai/
├── edge-gateway/       runs on the Pi / mini-PC at each site
│   ├── config/         device.yaml + one YAML per camera
│   ├── cameras/        adapters: rtsp, usb, picamera, http
│   ├── motion/         motion filter (drops idle frames)
│   ├── publisher/      pushes frames to the queue
│   ├── comms/          backend socket, heartbeat, live stream
│   └── storage/        offline buffer, snapshot cleanup
├── inference-worker/   YOLOv8s detection, batched, stateless
├── rules-engine/       zones, tracking, dwell time, alerts
├── shared/             message formats all services agree on
├── training/           dataset, training and export (PC / Colab)
├── infra/              docker-compose for local runs
├── requirements.txt
└── .env.example
```

## Setup

```bash
cd ~/Nexus/ai
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Activate the venv in every new terminal:

```bash
source ~/Nexus/ai/venv/bin/activate
```

## Configuration

**`.env`** holds secrets (never commit it):

```
DEVICE_ID=
DEVICE_TOKEN=
BACKEND_URL=
REDIS_URL=redis://localhost:6379
CAM_FRONT_USER=
CAM_FRONT_PASS=
```

**Camera file** (`edge-gateway/config/cameras/front_gate.yaml`):

```yaml
camera_id: front_gate
type: rtsp                  # rtsp | usb | picamera | http
url: rtsp://${CAM_FRONT_USER}:${CAM_FRONT_PASS}@192.168.1.10:554/stream1
target_fps: 4
resize_to: 640

detection:
  confidence: 0.45
  classes: [person]

zones:
  - name: gate_area
    polygon: [[0.1,0.3],[0.9,0.3],[0.9,0.95],[0.1,0.95]]   # fractions 0-1
    active_hours: "18:00-06:00"
    min_seconds_in_zone: 3
```

Zones are stored as fractions (0 to 1), so they keep working if the camera resolution changes.

## Training the model

1. Put images and labels in `training/dataset/` (YOLO format, matching filenames):
```
   images/train/front_gate_0001.jpg
   labels/train/front_gate_0001.txt
```
2. Edit `training/dataset.yaml`:
```yaml
   path: dataset
   train: images/train
   val: images/val
   names:
     0: person
```
3. Train:
```bash
   cd training
   yolo detect train model=yolov8s.pt data=dataset.yaml epochs=50 imgsz=640 batch=16 patience=15 name=cctv_v1
```
4. Copy `runs/detect/cctv_v1/weights/best.pt` to `inference-worker/model/best.pt`.
5. For the Pi fallback, export: `yolo export model=best.pt format=ncnn`.

Training tips:
- Split train/val by camera or by day, never by random frame.
- Include night, rain, and IR footage.
- Include empty frames (with empty label files) to cut false alarms.
- Feed owner feedback (false alarms and confirmed intrusions) back into the next training round.

## Running (development)

Start Redis, then each service in its own terminal, with the venv active:

```bash
# 1. queue
docker run -p 6379:6379 redis

# 2. inference worker
python inference-worker/worker.py

# 3. rules engine
python rules-engine/main.py

# 4. edge gateway
python edge-gateway/main.py
```

## Message formats

Defined in `shared/`:

- `frame_message.json`: `{camera_id, site_id, timestamp, jpeg}`
- `detection_message.json`: `{camera_id, timestamp, boxes, scores, classes}`

## Design rules

1. The gateway always connects outward. It never accepts incoming connections.
2. All settings live in YAML/.env, never in code.
3. The detector is one replaceable file. A new model is just a new file in `inference-worker/model/`.
4. Alerts are buffered locally and retried when the internet is down.
5. Every alert carries a `camera_id` and stores owner feedback for retraining.

## Roadmap

- [ ] Fine-tune YOLOv8s on own footage
- [ ] Inference worker with batching
- [ ] Rules engine (zones, tracking, dwell time)
- [ ] Edge gateway with one RTSP camera
- [ ] Alert + screenshot upload to backend
- [ ] Live stream on request
- [ ] Multi-camera support
- [ ] Worker autoscaling and monitoring

## Notes for the Raspberry Pi

- The gateway needs only opencv, redis, requests, socketio, yaml and dotenv. Do not install `ultralytics` there unless it runs local fallback detection.
- Install the Pi camera library with `sudo apt install python3-picamera2`.
- Run the gateway as a `systemd` service so it restarts after power cuts.
