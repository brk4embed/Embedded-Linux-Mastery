# LLM Deployment on Radxa Rock 5B+ — Complete Guide

> **Goal:** Run actual LLM models (LLaMA, Whisper, Stable Diffusion) on your Radxa Rock 5B+ board from scratch. Covers CPU inference, NPU acceleration, benchmarking, and building an embedded AI application.

---

## Your Hardware Capabilities

```
Radxa Rock 5B+ (RK3588):
  CPU:   4× Cortex-A76 @ 2.4GHz + 4× Cortex-A55 @ 1.8GHz (big.LITTLE)
  RAM:   Up to 16GB LPDDR5 (critical for LLM)
  NPU:   Built-in 6 TOPS (tera operations per second)
  GPU:   Mali-G610 (can assist with inference)
  PCIe:  M.2 slot (attach GPU like Hailo-8 or Coral for more TOPS)
  
What this means for LLMs:
  7B parameter model (FP16): needs ~14GB RAM → fits in 16GB version
  7B parameter model (INT4): needs ~4GB RAM → fits in any version
  3B parameter model (INT4): needs ~2GB RAM → comfortable on any version
```

---

## Part 1: Prepare the Board

### Step 1: Flash Latest OS

```bash
# Download Radxa Ubuntu/Debian image
# https://wiki.radxa.com/Rock5/downloads

# Flash to microSD (from your laptop):
# Download radxa-rock-5b-ubuntu-focal-server-arm64-*.img.xz
xz -d radxa-rock-5b-ubuntu-focal-server-arm64-*.img.xz
sudo dd if=radxa-rock-5b-ubuntu-focal-server-arm64-*.img of=/dev/sdX bs=4M status=progress
sync

# Boot from microSD, login: rock/rock
```

### Step 2: Initial Setup

```bash
# SSH into board
ssh rock@192.168.1.xxx  # replace with your board's IP

# Update packages
sudo apt update && sudo apt upgrade -y

# Install build tools
sudo apt install -y build-essential cmake git python3 python3-pip \
                    wget curl htop tmux vim

# Check memory
free -h
# Should show ~14-15GB available (some used by OS/GPU/NPU)

# Check CPU
cat /proc/cpuinfo | grep "model name" | head -2
# ARM Cortex-A76 @ 2400MHz

# Check NPU
ls /dev/rknpu*
# /dev/rknpu  → NPU device present
```

---

## Part 2: CPU-Only Inference with llama.cpp

### What is llama.cpp?

`llama.cpp` is a C/C++ implementation of LLaMA inference that:
- Runs on CPU without GPU
- Supports quantization (INT4, INT8) to reduce memory
- Is highly optimized for ARM NEON instructions
- Can run 7B models at 3-8 tokens/second on Radxa 5B+

### Build llama.cpp

```bash
# On Radxa board:
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

# Build with ARM NEON optimizations
cmake -B build \
    -DGGML_NATIVE=ON \
    -DGGML_ARM_NEON=ON \
    -DGGML_ARM_FMA=ON \
    -DCMAKE_BUILD_TYPE=Release

cmake --build build -j$(nproc)
# Takes ~10 minutes on Radxa

ls build/bin/
# llama-cli        ← main inference binary
# llama-bench      ← benchmark tool
# llama-server     ← HTTP API server
```

### Download and Run LLaMA 3.2 3B (Recommended Start)

```bash
# Download a quantized model (INT4 = small, fast, slightly less accurate)
# 3B parameter model, Q4_K_M quantization = ~2GB
cd ~/models
wget https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/resolve/main/Llama-3.2-3B-Instruct-Q4_K_M.gguf

# Run inference
cd ~/llama.cpp
./build/bin/llama-cli \
    -m ~/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
    -n 256 \
    --temp 0.7 \
    -p "Explain what a Linux device driver is in simple terms:"
```

**Expected output (after ~30 seconds loading):**
```
A Linux device driver is a piece of software that acts as a translator
between the operating system and hardware devices...

llama_print_timings: load time = 8542.34 ms
llama_print_timings: eval time = 45234.12 ms / 256 runs
llama_print_timings: speed: 5.66 tokens/second
```

### Benchmark Different Models

```bash
# Benchmark tool
./build/bin/llama-bench \
    -m ~/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
    -n 128 \
    -ngl 0        # 0 GPU layers (pure CPU)

# Results table:
# model          |    t/s (eval)   |    t/s (pp)
# ─────────────────────────────────────────────
# LLaMA-3.2-3B   |     5.66 t/s    |   47.23 t/s

# Try with more threads (optimal = number of big cores = 4)
./build/bin/llama-cli \
    -m ~/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
    -t 4 \
    -p "Hello"
```

### Run as HTTP API Server

```bash
# Start API server on port 8080
./build/bin/llama-server \
    -m ~/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
    -c 2048 \
    --host 0.0.0.0 \
    --port 8080 &

# Test API from your laptop
curl http://192.168.1.xxx:8080/completion \
    -H "Content-Type: application/json" \
    -d '{"prompt": "What is a kernel driver?", "n_predict": 128}'
```

---

## Part 3: NPU Inference with RKNN

### What is RKNN?

The RK3588 has a dedicated **NPU (Neural Processing Unit)** with 6 TOPS. It runs models much faster than CPU for image/audio tasks but requires models to be converted to RKNN format first.

### Install rknn-toolkit2

```bash
# On your LAPTOP (x86-64) — toolkit for model conversion
pip3 install rknn-toolkit2

# On Radxa board — runtime library
git clone https://github.com/airockchip/rknn-toolkit2.git
cd rknn-toolkit2/rknn-toolkit-lite2/packages/
pip3 install rknn_toolkit_lite2-*.whl

# Verify
python3 -c "from rknnlite.api import RKNNLite; print('RKNN OK')"
```

### Convert YOLOv8 to RKNN and Run on NPU

```python
# convert_yolo.py — Run on LAPTOP to convert model
from rknn.api import RKNN

# Initialize
rknn = RKNN(verbose=True)

# Load YOLOv8 ONNX model (download from Ultralytics)
rknn.config(mean_values=[[0, 0, 0]], std_values=[[255, 255, 255]],
            target_platform='rk3588')
rknn.load_onnx(model='yolov8n.onnx')

# Build RKNN model
# do_quantization=True = INT8 (faster NPU inference, small accuracy loss)
rknn.build(do_quantization=True, dataset='dataset.txt')  # dataset: calibration images

# Export
rknn.export_rknn('yolov8n.rknn')
rknn.release()
print("Converted: yolov8n.rknn")
```

```python
# run_yolo_npu.py — Run on RADXA BOARD
import cv2
import numpy as np
from rknnlite.api import RKNNLite

# Initialize RKNN runtime
rknn = RKNNLite()
rknn.load_rknn('yolov8n.rknn')
rknn.init_runtime(core_mask=RKNNLite.NPU_CORE_0_1_2)  # use all 3 NPU cores!

# Load and preprocess image
img = cv2.imread('test.jpg')
img_resized = cv2.resize(img, (640, 640))
img_rgb = cv2.cvtColor(img_resized, cv2.COLOR_BGR2RGB)
input_data = np.expand_dims(img_rgb, axis=0)

# Run inference on NPU
import time
start = time.time()
outputs = rknn.inference(inputs=[input_data])
elapsed = time.time() - start

print(f"NPU inference time: {elapsed*1000:.1f}ms")
print(f"FPS: {1/elapsed:.1f}")
# Expected: ~30ms → ~33 FPS on RK3588 NPU

# Post-process outputs (YOLO bounding boxes)
# ... (see full YOLO post-processing code)

rknn.release()
```

### Benchmark: CPU vs NPU

```bash
# Create benchmark script
cat > bench_npu.py << 'EOF'
import time, numpy as np
from rknnlite.api import RKNNLite

rknn = RKNNLite()
rknn.load_rknn('yolov8n.rknn')

# CPU inference
rknn.init_runtime()
input_data = np.random.randn(1, 640, 640, 3).astype(np.float32)
start = time.time()
for _ in range(100):
    rknn.inference(inputs=[input_data])
cpu_time = (time.time() - start) / 100 * 1000
print(f"CPU: {cpu_time:.1f}ms per frame ({1000/cpu_time:.0f} FPS)")

# NPU inference (all 3 cores)
rknn.init_runtime(core_mask=RKNNLite.NPU_CORE_0_1_2)
start = time.time()
for _ in range(100):
    rknn.inference(inputs=[input_data])
npu_time = (time.time() - start) / 100 * 1000
print(f"NPU: {npu_time:.1f}ms per frame ({1000/npu_time:.0f} FPS)")
print(f"NPU speedup: {cpu_time/npu_time:.1f}x")
EOF

python3 bench_npu.py
# CPU: 180ms per frame (5 FPS)
# NPU: 28ms per frame (35 FPS)
# NPU speedup: 6.4x
```

---

## Part 4: Run Whisper (Speech-to-Text) on Radxa

### Install whisper.cpp

```bash
git clone https://github.com/ggerganov/whisper.cpp
cd whisper.cpp

# Build with NEON
cmake -B build \
    -DGGML_ARM_NEON=ON \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# Download small English model (~150MB)
bash ./models/download-ggml-model.sh base.en
# OR for better accuracy:
bash ./models/download-ggml-model.sh small.en
```

### Run Speech-to-Text

```bash
# Record audio from microphone (install microphone USB or use existing)
arecord -D default -f S16_LE -r 16000 -c 1 -d 5 test.wav

# Transcribe
./build/bin/whisper-cli \
    -m models/ggml-base.en.bin \
    -f test.wav \
    --language en

# Output:
# [00:00:00.000 --> 00:00:03.500]   Hello, this is a test recording
# Time: 1.2s

# Real-time streaming (microphone → text)
./build/bin/whisper-stream \
    -m models/ggml-base.en.bin \
    --step 3000 \
    --length 10000
```

---

## Part 5: Build an Embedded AI Application

### Voice-Controlled Embedded Assistant

```python
#!/usr/bin/env python3
# embedded_assistant.py
# Voice → STT (Whisper) → LLM (llama.cpp) → TTS → Speaker

import subprocess
import requests
import json
import time

LLAMA_SERVER = "http://localhost:8080"

def record_audio(duration=5, filename="/tmp/query.wav"):
    """Record from microphone"""
    subprocess.run([
        "arecord", "-D", "default", "-f", "S16_LE",
        "-r", "16000", "-c", "1", "-d", str(duration),
        filename
    ], check=True)
    return filename

def speech_to_text(audio_file):
    """Transcribe using whisper.cpp"""
    result = subprocess.run([
        "/home/rock/whisper.cpp/build/bin/whisper-cli",
        "-m", "/home/rock/whisper.cpp/models/ggml-base.en.bin",
        "-f", audio_file,
        "--output-json",
        "--no-prints"
    ], capture_output=True, text=True, check=True)
    
    data = json.loads(result.stdout)
    text = " ".join([seg["text"] for seg in data["transcription"]])
    return text.strip()

def ask_llm(question, system_prompt=None):
    """Send question to llama.cpp server"""
    if not system_prompt:
        system_prompt = """You are a helpful embedded Linux engineering assistant. 
Give concise answers suitable for audio output. Keep responses under 3 sentences."""
    
    prompt = f"<|system|>{system_prompt}<|user|>{question}<|assistant|>"
    
    response = requests.post(f"{LLAMA_SERVER}/completion", json={
        "prompt": prompt,
        "n_predict": 150,
        "temperature": 0.7,
        "stop": ["<|user|>", "<|system|>"]
    })
    
    return response.json()["content"].strip()

def text_to_speech(text):
    """Simple TTS using espeak"""
    subprocess.run(["espeak", "-s", "140", text])

def main():
    print("=== Embedded AI Assistant ===")
    print("Press Enter to ask a question (5 seconds recording)...")
    
    while True:
        input()
        print("Recording... speak now!")
        
        try:
            # Step 1: Record
            audio = record_audio(duration=5)
            
            # Step 2: Speech to text
            print("Transcribing...")
            question = speech_to_text(audio)
            print(f"You said: {question}")
            
            if not question or len(question) < 3:
                print("Didn't catch that, try again")
                continue
            
            # Step 3: Ask LLM
            print("Thinking...")
            start = time.time()
            answer = ask_llm(question)
            elapsed = time.time() - start
            print(f"Answer ({elapsed:.1f}s): {answer}")
            
            # Step 4: Speak answer
            text_to_speech(answer)
            
        except Exception as e:
            print(f"Error: {e}")
        
        print("\nPress Enter for next question...")

if __name__ == "__main__":
    main()
```

```bash
# Install espeak for TTS
sudo apt install espeak

# Start llama.cpp server (in tmux pane 1)
tmux new -s ai
./llama.cpp/build/bin/llama-server \
    -m ~/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
    -c 2048 --host 0.0.0.0 --port 8080

# Run assistant (in tmux pane 2)
tmux split-pane
python3 embedded_assistant.py
```

---

## Part 6: Benchmarking and Monitoring

```bash
# Monitor CPU/NPU/thermal during inference
watch -n1 'cat /sys/class/thermal/thermal_zone*/temp | awk "{print \$1/1000\" C\"}"'

# CPU frequency governor
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# NPU utilization
cat /sys/kernel/debug/rknpu/load 2>/dev/null || \
    cat /sys/class/misc/rknpu/device/subsystem/rknpu/load

# Memory usage during inference
watch -n1 'free -h && nvidia-smi 2>/dev/null || true'

# Power consumption estimate
# RK3588 TDP: ~10W (CPU) + ~3W (NPU) = ~13W under full load
# With 5V/3A charger = 15W max → adequate
```

---

## Part 7: Model Comparison Table

| Model | Size (INT4) | RAM Needed | CPU Speed | NPU Support |
|-------|-------------|-----------|-----------|-------------|
| LLaMA 3.2 1B | ~700MB | 1.5GB | 12-15 t/s | No (LLM) |
| LLaMA 3.2 3B | ~1.8GB | 3GB | 5-7 t/s | No (LLM) |
| LLaMA 3.1 7B | ~4.1GB | 6GB | 2-3 t/s | No (LLM) |
| Whisper base | 145MB | 0.5GB | 8× realtime | Yes (RKNN) |
| Whisper small | 466MB | 1GB | 4× realtime | Yes (RKNN) |
| YOLOv8n | 6MB | 0.1GB | 5 FPS | 35 FPS (RKNN) |
| YOLOv8s | 22MB | 0.2GB | 3 FPS | 25 FPS (RKNN) |
| MobileNet v3 | 5MB | 0.05GB | 100 FPS | 500+ FPS |

---

## Part 8: Building Your Embedded AI Portfolio Project

This becomes **Portfolio Project #5** in `24_Industry_Project_Portfolio/`:

```markdown
# Project: Radxa 5B+ Embedded AI Assistant

## Elevator Pitch
"A voice-controlled embedded AI system running entirely on-device — 
no cloud, no latency, no privacy concerns — on a $100 Radxa Rock 5B+ board."

## Technical Stack
- Hardware: Radxa Rock 5B+ (RK3588 SoC, 16GB LPDDR5, 6 TOPS NPU)
- STT: whisper.cpp (base.en, ~8× realtime)
- LLM: llama.cpp (LLaMA 3.2 3B Q4_K_M, 5-7 t/s)
- Object Detection: YOLOv8n via RKNN (~35 FPS on NPU)
- Custom Linux kernel driver for NPU DMA optimization

## Results
- Latency: <2 seconds from speech end to spoken response
- Privacy: 100% offline (no data leaves the device)
- Power: ~8W average under inference load
- Cost: $100 hardware + $0 cloud costs

## Code
github.com/brk4embed/Embedded-Linux-Mastery/23_Radxa_5B_Plus_Labs/
```
