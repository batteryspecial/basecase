<img width="333" height="222" alt="image" src="https://github.com/user-attachments/assets/9e29f643-3852-4a0f-8bde-ba241bc4ce06" />


# Basecase

Physical AI is limited by how its hardware was programmed, not by the hardware itself. One webcam, driven by agents, becomes a people counter, a follow-cam, a security guard, a privacy camera, a traffic counter, or a typing guide. Every switch happens from one sentence, with no retraining. Applies to other pieces of hardware as well.

## Demo
https://github.com/user-attachments/assets/13ea6aa6-4f38-44e5-9fa2-760f84c2c9b9

## Components

| Component | Port | Stack | What it does |
| --- | --- | --- | --- |
| Perception (reflex layer) | 8001 | Python, PyTorch, Ultralytics YOLOE, ByteTrack, CLIP | Runs every frame: detect, track, check attributes, run behaviors, draw overlays, emit events |
| Orchestrator (agent layer) | 8000 | Python, LangGraph, LanceDB | LLM agent with tools. Turns instructions into behaviors and queries; reacts to events |
| Frontend | 3000 (dev) / served by 8000 | React 19, TypeScript, Vite | Enriched live feed, chat with image upload, agent trace, events, behaviors, switch timer |
| Camera service *(optional)* | 8003 | Python | Serves raw frames over HTTP; later drives the stepper via Arduino. Perception can read the webcam directly instead |

`linker/schemas.py` is the shared contract (pydantic v2) between the two Python services. Deeper docs: [`docs/PROJECT.md`](docs/PROJECT.md), [`perception/README.md`](perception/README.md), [`orchestrator/README.md`](orchestrator/README.md), [`frontend/README.md`](frontend/README.md).

## Requirements

| | Minimum (logic / dev) | Recommended (live demo) |
| --- | --- | --- |
| GPU | None (`DEVICE=cpu`, or the `fake` detector) | NVIDIA GPU with CUDA 12.x, ~6 GB+ VRAM |
| CPU / RAM | Any modern laptop, 8 GB | 6+ cores, 16 GB (MediaPipe, RTMPose and EasyOCR run on CPU next to the detector) |
| Camera | A video file or stream URL | USB webcam that supports MJPG at 1280×720 (e.g. Logitech C920) |
| Disk | ~3 GB for Python deps | + ~1.2 GB of model weights, downloaded on first boot |
| Network | Internet for the LLM API | Same; download weights once beforehand |

Apple Silicon works for development, but YOLOE is forced to CPU there (its text encoder needs float64, which MPS lacks) leading to a low frame rate. On a CUDA GPU, `yoloe-11s-seg` measured ~38-40 ms/frame.

### Software

- **Python 3.12+** (developed and verified on 3.14), one venv per service
- **Node.js 20+** and npm, for the frontend
- **NVIDIA Driver + CUDA 12.x** on the GPU host, with a CUDA build of PyTorch
- An **OpenAI key** (default) or an **Anthropic key** (`ORCH_PROVIDER=anthropic`)

### Packages

| Service | Key Packages | File |
| --- | --- | --- |
| Perception | `torch`, `torchvision`, `ultralytics==8.4.155`, `supervision==0.30.4`, `open_clip_torch`, `opencv-python`, `fastapi`, `uvicorn`; optional `mediapipe`, `rtmlib`, `onnxruntime`, `easyocr` | [`perception/requirements.txt`](perception/requirements.txt) |
| Orchestrator | `langgraph`, `langchain-core`, `langchain-openai`, `lancedb`, `tantivy`, `httpx`, `fastapi`, `uvicorn` | [`orchestrator/requirements.txt`](orchestrator/requirements.txt) |
| Frontend | `react@19`, `vite@7`, `typescript@5.9` | [`frontend/package.json`](frontend/package.json) |

A model that can't load reports `available: false` with a reason and the service still boots.

## Running

Commands below are for Linux. On Windows use `.venv\Scripts\` instead of `.venv/bin/`.

**1. Perception (:8001)**

```bash
cd perception
python -m venv .venv
# Install the CUDA build of PyTorch first, matching your CUDA version
.venv/bin/pip install torch torchvision --index-url https://download.pytorch.org/whl/cu126
.venv/bin/pip install -r requirements.txt

VIDEO_SOURCE=0 DEVICE=cuda .venv/bin/uvicorn main:app --host 0.0.0.0 --port 8001
```

The first boot downloads ~1.2 GB of weights (YOLOE, YOLO11, CLIP, and a ~570 MB text encoder). Check `http://localhost:8001/health` for fps and the loaded model, or open `http://localhost:8001/` for a debug page with the feed.

Confirm the GPU is in use using torch.

```bash
.venv/bin/python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

**2. Orchestrator**

```bash
cd orchestrator
python -m venv .venv
.venv/bin/pip install -r requirements.txt
cp .env.example .env            # then fill in OPENAI_API_KEY
.venv/bin/python main.py
```

**3. Frontend**

```bash
cd frontend
npm install
cp .env.example .env.local
npm run dev        # http://localhost:3000
```

Or run `npm run build` once and the orchestrator serves the console at `http://localhost:8000`.

Then type an instruction such as *"Highlight everyone"*, *"Watch my phone and tell me if it disappears"*, or *"Blur everyone's face except mine"*.

### No GPU Environment

```bash
DEVICE=cpu IMGSZ=320 VIDEO_SOURCE=0 .venv/bin/uvicorn main:app --port 8001
```

Switch to the `fake` detector (`POST /model {"name": "fake"}`) to exercise the loop, hot-swap and UI with no weights at all.

### Remote GPU Host

Run perception on the GPU box and everything else locally. Bind perception to `0.0.0.0`, then set `PERCEPTION_BASE=http://<gpu-host>:8001` in `orchestrator/.env` and `VITE_PERCEPTION_URL=http://<gpu-host>:8001` in `frontend/.env.local`.

## Configuration

| Variable | Service | Default | Notes |
| --- | --- | --- | --- |
| `VIDEO_SOURCE` | perception | `0` | Webcam index, file path (loops), or stream URL |
| `DEVICE` | perception | `auto` | `auto` → cuda → mps → cpu |
| `IMGSZ` | perception | `640` | Detector input size; 320 on CPU |
| `CAPTURE_WIDTH` / `CAPTURE_HEIGHT` | perception | `1280` / `720` | Webcams often crop the sensor at lower resolutions |
| `CAMERA_FOURCC` | perception | `MJPG` | YUY2 at 720p over USB 2.0 caps out around 10 fps |
| `CONF_THRESHOLD` | perception | `0.15` | Open-vocabulary scores run lower than COCO |
| `ORCH_PROVIDER` | orchestrator | `openai` | Or `anthropic` |
| `ORCH_MODEL` | orchestrator | `gpt-6-astra` | Must support tool calls and images |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` | orchestrator | | In `orchestrator/.env` |
| `PERCEPTION_BASE` | orchestrator | `http://localhost:8001` | |
| `VITE_ORCHESTRATOR_URL` / `VITE_PERCEPTION_URL` | frontend | `localhost:8000` / `:8001` | In `frontend/.env.local` |
