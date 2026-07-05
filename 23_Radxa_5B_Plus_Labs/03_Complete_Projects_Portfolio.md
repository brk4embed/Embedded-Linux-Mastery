# Radxa Rock 5B+ — Complete Portfolio Projects Guide

> **Your Hardware Advantage:** The Radxa Rock 5B+ with RK3588 is one of the most capable  
> single-board computers available. 6 TOPS NPU, PCIe 3.0, HDMI 2.1, 4K60 output, and real  
> ARM big.LITTLE make it ideal for a professional embedded AI portfolio.

---

## Hardware Reference

```
Radxa Rock 5B+ Specifications:
  SoC:   Rockchip RK3588
         4× Cortex-A76 @ 2.4GHz (big) + 4× Cortex-A55 @ 1.8GHz (LITTLE)
         Mali-G610 GPU (OpenGL ES 3.2, Vulkan 1.1, OpenCL 2.0)
         6 TOPS NPU (Neural Processing Unit)
  RAM:   4GB / 8GB / 16GB LPDDR5
  eMMC:  32/64/128GB on-module
  I/O:   40-pin GPIO header (compatible with Raspberry Pi pinout)
         USB 3.0 × 2, USB 2.0 × 2, USB-C 3.1
         PCIe 3.0 M.2 (for NVMe SSD or Coral/Hailo accelerator)
         2× HDMI 2.1 (4K120 + 4K60)
         eDP (LCD)
         2× MIPI CSI (cameras)
         MIPI DSI (display)
         Gigabit Ethernet
  Debug: 3-pin UART debug header (1.5Mbaud)

GPIO Debug Header Pinout (for UART connection):
  Pin 6: GND
  Pin 8: TX (board → PC)
  Pin 10: RX (PC → board)
  Baud rate: 1500000 (1.5Mbaud)
```

---

## Setup: Connecting Debug Console

```bash
# On your Ubuntu laptop
sudo apt install tio minicom screen

# Connect USB-to-UART adapter to pins 6/8/10
# Then connect to board
sudo tio /dev/ttyUSB0 -b 1500000

# Power on Rock 5B+. You should see U-Boot messages.
```

---

## Beginner Projects

### Project B1: GPIO LED Blink

**Goal:** Control an LED connected to GPIO pin from both shell and Python.  
**Hardware:** LED + 220Ω resistor connected to GPIO4_B2 (pin 36 on header).

```bash
# Method 1: sysfs (legacy but easy to understand)
# Find GPIO number: GPIO4_B2 = bank4 × 32 + group_B × 8 + pin2 = 128+8+2 = 138
echo 138 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio138/direction
echo 1 > /sys/class/gpio/gpio138/value   # LED ON
sleep 1
echo 0 > /sys/class/gpio/gpio138/value   # LED OFF

# Method 2: libgpiod (modern)
sudo apt install gpiod

# Find the chip and line
gpiodetect
# gpiochip0 [gpio0-rockchip] (32 lines)
# gpiochip4 [gpio4-rockchip] (32 lines)

gpioinfo gpiochip4 | grep "line 10"  # GPIO4_B2 = line 10 in bank4

# Toggle
gpioset gpiochip4 10=1   # ON
gpioset gpiochip4 10=0   # OFF

# Blink loop
for i in $(seq 1 10); do
    gpioset gpiochip4 10=1
    sleep 0.5
    gpioset gpiochip4 10=0
    sleep 0.5
done
```

```python
# Method 3: Python with gpiod library
import gpiod
import time

chip = gpiod.Chip('gpiochip4')
line = chip.get_line(10)  # GPIO4_B2

line.request(consumer="blink", type=gpiod.LINE_REQ_DIR_OUT)

for i in range(10):
    line.set_value(1)  # LED ON
    time.sleep(0.5)
    line.set_value(0)  # LED OFF
    time.sleep(0.5)

line.release()
```

```python
# Method 4: Kernel module for GPIO control
# (See 07_Device_Drivers/ for the full driver)
```

---

### Project B2: I2C Sensor Reading

**Goal:** Read temperature and humidity from an I2C sensor (HDC1080 or SHT31).  
**Hardware:** SHT31 sensor connected to I2C1 (pins 3/5 on 40-pin header).

```bash
# Enable I2C1 in device tree overlay
# (Radxa provides overlays: /boot/overlays/)
ls /boot/overlays/ | grep i2c
# rk3588-rock-5b-i2c1-m2.dtbo

# Load overlay
echo "overlays=rk3588-rock-5b-i2c1-m2" >> /boot/armbianEnv.txt
reboot

# After reboot
ls /dev/i2c*
# /dev/i2c-0  /dev/i2c-1  /dev/i2c-2  /dev/i2c-6

# Detect SHT31 (address 0x44)
i2cdetect -y 6
# Should show 0x44

# Read temperature (SHT31 protocol)
# Command: 0x2C 0x06 (single shot, high repeatability)
i2ctransfer -y 6 w2@0x44 0x2C 0x06
sleep 0.1
# Read 6 bytes: temp_msb, temp_lsb, crc, hum_msb, hum_lsb, crc
i2ctransfer -y 6 r6@0x44
# 0x60 0xa4 0x8f 0x58 0x2e 0xff
# temp = (0x60a4 × 175/65535) - 45 = 22.3°C
```

```python
# Python SHT31 reader
import smbus2
import time

bus = smbus2.SMBus(6)  # i2c-6

SHT31_ADDR = 0x44

def read_sht31():
    # Send measurement command
    bus.write_i2c_block_data(SHT31_ADDR, 0x2C, [0x06])
    time.sleep(0.1)
    
    # Read 6 bytes
    data = bus.read_i2c_block_data(SHT31_ADDR, 0x00, 6)
    
    # Convert to temperature and humidity
    temp_raw = (data[0] << 8) | data[1]
    hum_raw  = (data[3] << 8) | data[4]
    
    temp = -45 + (175 * temp_raw / 65535)
    humidity = 100 * hum_raw / 65535
    
    return temp, humidity

while True:
    t, h = read_sht31()
    print(f"Temperature: {t:.1f}°C  Humidity: {h:.1f}%")
    time.sleep(2)
```

---

### Project B3: SPI Device Communication

```bash
# SPI on Radxa: SPI4 available on 40-pin header
# Enable SPI4 overlay
echo "overlays=rk3588-rock-5b-spi4-m2" >> /boot/armbianEnv.txt
reboot

# Test SPI loopback (connect MOSI to MISO pin)
ls /dev/spidev*
# /dev/spidev4.0

# Python SPI test
python3 -c "
import spidev
spi = spidev.SpiDev()
spi.open(4, 0)          # SPI bus 4, device 0
spi.max_speed_hz = 1000000  # 1 MHz
spi.mode = 0            # CPOL=0, CPHA=0

# Loopback test: send bytes, receive same bytes back
tx = [0xDE, 0xAD, 0xBE, 0xEF]
rx = spi.xfer2(tx)
print(f'TX: {[hex(b) for b in tx]}')
print(f'RX: {[hex(b) for b in rx]}')
# With loopback: TX == RX
spi.close()
"
```

---

## Intermediate Projects

### Project I1: AI Object Detection Pipeline (30+ FPS)

**Goal:** Real-time object detection on camera input using YOLOv8 on NPU.  
**Hardware:** USB camera or MIPI camera module, HDMI monitor.

```bash
# Install dependencies
sudo apt install -y python3-opencv
pip3 install ultralytics rknn-toolkit-lite2

# Convert YOLOv8 to RKNN (on x86 laptop)
# See 23_Radxa_5B_Plus_Labs/02_LLM_Deployment_Guide.md for full conversion steps

# Deploy on Radxa
```

```python
#!/usr/bin/env python3
# object_detection_npu.py — Real-time detection on NPU
import cv2
import numpy as np
import time
from rknnlite.api import RKNNLite

# Initialize RKNN
rknn = RKNNLite()
rknn.load_rknn('/opt/models/yolov8n.rknn')
rknn.init_runtime(core_mask=RKNNLite.NPU_CORE_0_1_2)

# COCO class labels
CLASSES = ['person','bicycle','car','motorbike','aeroplane','bus','train',
           'truck','boat','traffic light',...] 

# Open camera
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

fps_counter = 0
fps_start = time.time()

while True:
    ret, frame = cap.read()
    if not ret: break

    # Preprocess
    img = cv2.resize(frame, (640, 640))
    img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    input_data = np.expand_dims(img_rgb, 0)

    # NPU inference
    t_start = time.time()
    outputs = rknn.inference(inputs=[input_data])
    inference_ms = (time.time() - t_start) * 1000

    # Post-process YOLO outputs → bounding boxes
    # (standard YOLO NMS post-processing)
    boxes, scores, classes = postprocess_yolo(outputs, frame.shape)

    # Draw results
    for box, score, cls in zip(boxes, scores, classes):
        x1, y1, x2, y2 = box
        label = f"{CLASSES[cls]}: {score:.2f}"
        cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
        cv2.putText(frame, label, (x1, y1-10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2)

    # Show FPS
    fps_counter += 1
    if fps_counter % 30 == 0:
        fps = 30 / (time.time() - fps_start)
        fps_start = time.time()
    cv2.putText(frame, f"NPU: {inference_ms:.0f}ms | FPS: {fps:.0f}",
                (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)

    cv2.imshow("Object Detection (NPU)", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
rknn.release()
```

### Project I2: Industrial Monitoring System

**Goal:** Monitor 4 temperature sensors + 2 vibration sensors + dashboard.  
**Hardware:** 4× SHT31 (different I2C addresses), 2× ADXL345 accelerometers.

```python
#!/usr/bin/env python3
# industrial_monitor.py
import asyncio
import json
import time
from datetime import datetime
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
import uvicorn

# Sensor data storage
sensor_data = {
    "temperatures": {f"sensor_{i}": [] for i in range(4)},
    "vibrations": {f"sensor_{i}": [] for i in range(2)},
    "alerts": []
}

TEMP_ALERT_HIGH = 60.0   # degrees C
VIBRATION_ALERT = 2.0    # g-force

async def read_sensors():
    """Background task: read all sensors every second"""
    import smbus2
    buses = [smbus2.SMBus(i) for i in [1, 2, 3, 6]]  # I2C buses
    sht31_addrs = [0x44, 0x45, 0x44, 0x45]   # sensors on different buses/addrs
    
    while True:
        ts = datetime.now().isoformat()
        
        for i, (bus, addr) in enumerate(zip(buses, sht31_addrs)):
            try:
                bus.write_i2c_block_data(addr, 0x2C, [0x06])
                await asyncio.sleep(0.1)
                data = bus.read_i2c_block_data(addr, 0, 6)
                temp_raw = (data[0] << 8) | data[1]
                temp = -45 + (175 * temp_raw / 65535)
                
                sensor_data["temperatures"][f"sensor_{i}"].append(
                    {"ts": ts, "value": temp}
                )
                sensor_data["temperatures"][f"sensor_{i}"] = \
                    sensor_data["temperatures"][f"sensor_{i}"][-100:]  # keep 100 samples
                
                if temp > TEMP_ALERT_HIGH:
                    sensor_data["alerts"].append({
                        "ts": ts, "type": "HIGH_TEMP",
                        "sensor": f"temperature_{i}", "value": temp
                    })
            except Exception as e:
                pass
        
        await asyncio.sleep(1.0)

app = FastAPI(title="Industrial Monitor")

@app.get("/", response_class=HTMLResponse)
async def dashboard():
    return """
    <html><head><title>Industrial Monitor</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    </head><body>
    <h1>Industrial Monitoring — Radxa Rock 5B+</h1>
    <div id="temp-chart"><canvas id="myChart"></canvas></div>
    <div id="alerts"><h2>Alerts</h2><ul id="alert-list"></ul></div>
    <script>
        // WebSocket for live updates
        const ws = new WebSocket("ws://localhost:8000/ws");
        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            updateCharts(data);
        };
    </script>
    </body></html>
    """

@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    while True:
        await ws.send_json(sensor_data)
        await asyncio.sleep(1.0)

@app.get("/api/sensors")
async def get_sensors():
    return sensor_data

@app.get("/api/alerts")
async def get_alerts():
    return sensor_data["alerts"][-50:]  # last 50 alerts

@app.on_event("startup")
async def startup():
    asyncio.create_task(read_sensors())

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Advanced Projects

### Project A1: Smart Surveillance Camera System

**Goal:** IP camera with object detection, recording, alerts, web interface.

```
Architecture:
  Camera (MIPI CSI2)
       │ V4L2 capture
       ▼
  ISP Processing (RK3588 hardware ISP)
       │ NV12 frames
       ▼
  NPU Object Detection (YOLOv8n, 30 FPS)
       │ detected objects + metadata
       ▼
  Decision Engine
       ├── If person detected: start recording + send alert
       ├── If no motion: standby
       └── Always: stream RTSP
       ▼
  Storage (NVMe via M.2)      RTSP Stream      Alert System
  H.264 recording             (port 8554)      (email/webhook)
       │                           │
       ▼                           ▼
  Web Interface (port 8080) ← Review recordings + live view
```

```python
#!/usr/bin/env python3
# smart_camera.py — simplified architecture
import cv2
import threading
import time
from rknnlite.api import RKNNLite
from pathlib import Path
import subprocess

class SmartCamera:
    def __init__(self):
        # Initialize NPU
        self.rknn = RKNNLite()
        self.rknn.load_rknn('/opt/models/yolov8n.rknn')
        self.rknn.init_runtime(core_mask=RKNNLite.NPU_CORE_0)
        
        # Camera
        self.cap = cv2.VideoCapture(0)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1920)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 1080)
        
        self.recording = False
        self.recorder = None
        self.recording_dir = Path("/var/recordings")
        self.recording_dir.mkdir(exist_ok=True)
        
        # Alert cooldown (don't spam alerts)
        self.last_alert_time = 0
        self.alert_cooldown = 30  # seconds
    
    def start_recording(self):
        if not self.recording:
            ts = time.strftime("%Y%m%d_%H%M%S")
            filename = self.recording_dir / f"event_{ts}.mp4"
            fourcc = cv2.VideoWriter_fourcc(*'mp4v')
            self.recorder = cv2.VideoWriter(str(filename), fourcc, 30, (1920, 1080))
            self.recording = True
            print(f"Recording: {filename}")
            # Auto-stop after 30 seconds
            threading.Timer(30, self.stop_recording).start()
    
    def stop_recording(self):
        if self.recording:
            self.recording = False
            if self.recorder:
                self.recorder.release()
                self.recorder = None
    
    def send_alert(self, objects_detected):
        now = time.time()
        if now - self.last_alert_time < self.alert_cooldown:
            return
        self.last_alert_time = now
        
        # Send webhook
        import requests
        try:
            requests.post("https://your-webhook.com/alert", json={
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
                "objects": objects_detected,
                "camera": "rock5b_camera_1"
            }, timeout=5)
        except:
            pass
        print(f"ALERT: {objects_detected} detected!")
    
    def run(self):
        while True:
            ret, frame = self.cap.read()
            if not ret: break
            
            # NPU inference
            img_resized = cv2.resize(frame, (640, 640))
            img_rgb = cv2.cvtColor(img_resized, cv2.COLOR_BGR2RGB)
            outputs = self.rknn.inference(inputs=[img_rgb[np.newaxis]])
            
            # Post-process
            persons_detected = self.detect_persons(outputs)
            
            if persons_detected:
                self.start_recording()
                self.send_alert(["person"] * persons_detected)
                # Draw bounding boxes
                cv2.putText(frame, f"PERSON DETECTED: {persons_detected}",
                           (50, 50), cv2.FONT_HERSHEY_SIMPLEX, 2, (0,0,255), 3)
            
            if self.recording and self.recorder:
                self.recorder.write(frame)

if __name__ == "__main__":
    cam = SmartCamera()
    cam.run()
```

### Project A2: Edge AI Gateway

**Goal:** A device that receives data from 10 IoT sensors over MQTT, processes it with AI, and provides dashboards + alerts.

```
IoT Sensors → MQTT → Radxa 5B+ Gateway
                           │
                     ┌─────┴──────┐
                     │  Services  │
                     │  ─────── │
                     │ MQTT Broker│ (mosquitto)
                     │ ─────── │
                     │ AI Engine  │ (anomaly detection)
                     │ ─────── │
                     │ Time-series│ (InfluxDB)
                     │ ─────── │
                     │ Dashboard  │ (Grafana)
                     └─────┬──────┘
                           │
                      Cloud Upload (optional)
                      Local Alert System
```

```bash
# Install services on Radxa
sudo apt install -y mosquitto mosquitto-clients
pip3 install paho-mqtt influxdb-client scikit-learn

# Start MQTT broker
sudo systemctl enable mosquitto
sudo systemctl start mosquitto

# Start InfluxDB
# (install from influxdata.com for ARM64)
wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.7.0-linux-arm64.tar.gz
tar -xzf influxdb2-2.7.0-linux-arm64.tar.gz
./influxd &
```

```python
# gateway.py
import paho.mqtt.client as mqtt
from influxdb_client import InfluxDBClient, Point
from sklearn.ensemble import IsolationForest
import numpy as np
import json

# Train anomaly detector on "normal" data
class AnomalyDetector:
    def __init__(self):
        self.model = IsolationForest(contamination=0.1)
        self.history = []
        self.trained = False
    
    def add_sample(self, features):
        self.history.append(features)
        if len(self.history) >= 100 and not self.trained:
            self.model.fit(self.history)
            self.trained = True
            print("Anomaly detector trained on 100 samples")
    
    def is_anomaly(self, features):
        if not self.trained:
            return False
        score = self.model.decision_function([features])[0]
        return score < 0   # negative = anomaly

detector = AnomalyDetector()
influx = InfluxDBClient(url="http://localhost:8086", token="mytoken", org="myorg")
write_api = influx.write_api()

def on_message(client, userdata, msg):
    data = json.loads(msg.payload)
    sensor_id = data["sensor_id"]
    temp = data["temperature"]
    humidity = data["humidity"]
    
    # Store in InfluxDB
    point = Point("sensor_data") \
        .tag("sensor_id", sensor_id) \
        .field("temperature", temp) \
        .field("humidity", humidity)
    write_api.write("mydb", "myorg", point)
    
    # Check for anomaly
    features = [temp, humidity]
    detector.add_sample(features)
    
    if detector.is_anomaly(features):
        print(f"ANOMALY detected from {sensor_id}: temp={temp} hum={humidity}")

mqtt_client = mqtt.Client()
mqtt_client.on_message = on_message
mqtt_client.connect("localhost", 1883)
mqtt_client.subscribe("sensors/#")
mqtt_client.loop_forever()
```

---

## Expert Projects

### Project E1: Local LLM Assistant (No Cloud Required)

```bash
# Already covered in 23_Radxa_5B_Plus_Labs/02_LLM_Deployment_Guide.md
# Here: add voice interface + display

# Hardware: 
#   USB microphone
#   HDMI monitor
#   USB speaker or 3.5mm speaker

# Complete pipeline:
# Voice → Whisper STT → LLaMA 3.2 3B → eSpeak TTS → Speaker

# Plus: 
# HDMI display shows conversation and system stats
# Physical buttons (GPIO) for wake word / push-to-talk

# Integration with display
pip3 install pygame

python3 - << 'EOF'
import pygame
import subprocess
import requests

pygame.init()
screen = pygame.display.set_mode((1920, 1080))
font_large = pygame.font.Font(None, 72)
font_small = pygame.font.Font(None, 36)

def display_conversation(question, answer):
    screen.fill((0, 0, 0))  # black background
    q_text = font_small.render(f"You: {question[:80]}", True, (0, 255, 0))
    a_text = font_large.render(answer[:60], True, (255, 255, 255))
    screen.blit(q_text, (50, 400))
    screen.blit(a_text, (50, 500))
    pygame.display.flip()

# Main loop: voice → LLM → display
while True:
    # Wait for button press (GPIO) or voice detection
    question = listen_for_voice()
    answer = ask_llm(question)
    display_conversation(question, answer)
    speak(answer)
EOF
```

### Project E2: AI-Powered Embedded Debugging Assistant

**Goal:** A system that watches kernel logs in real-time and automatically diagnoses issues.

```python
#!/usr/bin/env python3
# kernel_debug_assistant.py
"""
Real-time kernel log monitoring + AI analysis + automated fixes.
Runs on Radxa 5B+ and watches /dev/kmsg continuously.
"""
import asyncio
import subprocess
import anthropic
from datetime import datetime

client = anthropic.Anthropic()

ERROR_PATTERNS = {
    "BUG:": "kernel bug",
    "KASAN:": "memory corruption",
    "WARNING:": "kernel warning",
    "Oops:": "kernel oops",
    "Call Trace:": "stack trace",
    "UBSAN:": "undefined behavior",
    "hung_task_timeout": "hung task",
    "Out of memory": "OOM killer",
}

async def monitor_kernel_log():
    """Stream kernel messages in real-time"""
    proc = await asyncio.create_subprocess_exec(
        'sudo', 'cat', '/dev/kmsg',
        stdout=asyncio.subprocess.PIPE
    )
    
    error_buffer = []
    
    async for line in proc.stdout:
        line = line.decode(errors='ignore').strip()
        
        # Check for errors
        is_error = any(pattern in line for pattern in ERROR_PATTERNS)
        
        if is_error:
            error_buffer.append(line)
            
            # Collect context (wait for stack trace to complete)
            if len(error_buffer) >= 5 or "---[ end trace" in line:
                await analyze_error(error_buffer)
                error_buffer = []

async def analyze_error(error_lines):
    """Send error context to AI for analysis"""
    error_text = "\n".join(error_lines)
    
    print(f"\n{'='*60}")
    print(f"[{datetime.now().strftime('%H:%M:%S')}] ERROR DETECTED")
    print(f"{'='*60}")
    print(error_text)
    print("\n[AI Analysis]...")
    
    # Use streaming for real-time output
    with client.messages.stream(
        model="claude-3-haiku-20240307",
        max_tokens=400,
        messages=[{
            "role": "user",
            "content": f"""You are a Linux kernel expert. Analyze this kernel error:

{error_text}

Provide:
1. Root cause (1 sentence)
2. Why it happened (2-3 sentences)
3. Immediate fix (command or config change)
4. Long-term fix (code change needed)

Be concise. This is a real-time diagnostic tool."""
        }]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
    
    print("\n")

async def main():
    print("=== Kernel Debug Assistant ===")
    print("Monitoring /dev/kmsg for errors...")
    print("Ctrl+C to stop\n")
    await monitor_kernel_log()

asyncio.run(main())
```

---

## Portfolio Structure: How to Present These Projects

```markdown
# GitHub Repository Layout for Portfolio

Embedded-Linux-Mastery/
├── 23_Radxa_5B_Plus_Labs/
│   ├── project_B1_gpio_led/
│   │   ├── README.md          ← project summary + demo link
│   │   ├── led_blink.py       ← working code
│   │   ├── gpio_driver.c      ← kernel module version
│   │   └── demo.gif           ← screen recording
│   ├── project_I1_object_detection/
│   │   ├── README.md
│   │   ├── object_detection_npu.py
│   │   ├── benchmark_results.md  ← CPU vs NPU numbers
│   │   └── demo.mp4 (link)
│   ├── project_A1_smart_camera/
│   │   ├── README.md
│   │   ├── architecture.png
│   │   ├── src/
│   │   └── demo_video.md (YouTube link)
│   └── project_E2_debug_assistant/
│       ├── README.md
│       ├── kernel_debug_assistant.py
│       └── example_session.md
```

### README Template for Each Project

```markdown
# [Project Name]

## What This Project Does
[1-2 sentences. Non-technical person can understand.]

## Why I Built This
[Your motivation. Links to the problem it solves.]

## Technical Highlights
- Achieves X FPS with NPU vs Y FPS on CPU (6.4x speedup)
- Handles Z events per second on edge hardware
- Full offline operation — no cloud dependency

## Architecture
[ASCII diagram or image]

## Hardware Used
- Radxa Rock 5B+ (RK3588)
- [Other components]

## How to Run
[Exact commands. Someone else can reproduce in 30 minutes.]

## Demo
[YouTube link or GIF]

## Skills Demonstrated
- V4L2 camera pipeline
- RKNN NPU inference  
- Real-time Python on ARM64
- Linux device driver integration
```

---

## Benchmark Reference Table (Radxa Rock 5B+)

| Task | CPU Only | NPU Only | CPU+NPU | Power Draw |
|------|----------|---------|---------|-----------|
| YOLOv8n 640×640 | 5-7 FPS | 30-35 FPS | 35 FPS | CPU: 8W, NPU: 11W |
| YOLOv8s 640×640 | 2-3 FPS | 20-25 FPS | 25 FPS | 12W |
| MobileNetV2 (cls) | 200 FPS | 800 FPS | 800 FPS | 3W |
| ResNet50 (cls) | 50 FPS | 200 FPS | 200 FPS | 5W |
| Whisper base.en | 8× realtime | N/A | 8× realtime | 6W |
| LLaMA 3.2 3B Q4 | 5-7 tok/s | N/A | 5-7 tok/s | 10W |
| LLaMA 3.2 1B Q4 | 12-15 tok/s | N/A | 12-15 tok/s | 8W |
