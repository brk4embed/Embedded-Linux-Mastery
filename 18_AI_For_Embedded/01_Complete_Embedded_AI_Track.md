# Embedded AI Complete Track — ML Fundamentals to Edge Deployment

> **Why This Track?** The embedded engineer who can ALSO deploy AI has a 2-3× salary premium.  
> This guide takes you from "what is a neural network" to deploying LLMs on real hardware.

---

## Chapter 1: AI/ML Fundamentals (No Math Assumed)

### What is Machine Learning?

Traditional programming:
```
Rules + Data → Output
if temp > 80°C: alert()   ← you write the rule
```

Machine learning:
```
Data + Output → Rules (learned automatically)
model.fit(sensor_data, alert_labels)  ← machine learns the rule
```

The machine **learns the rules from examples** instead of you writing them.

### What is a Neural Network?

Imagine neurons in the brain. Each neuron:
1. Receives signals from many inputs
2. Applies a weight to each (some inputs matter more)
3. Sums them up
4. Fires if sum exceeds threshold

```python
# One artificial neuron in Python:
import numpy as np

def neuron(inputs, weights, bias):
    # Weighted sum
    z = np.dot(inputs, weights) + bias
    # Activation function (fire/not fire)
    return 1 / (1 + np.exp(-z))   # sigmoid: output 0 to 1

# Example: temperature sensor neuron
inputs  = [25.0, 60.0]        # temperature=25°C, humidity=60%
weights = [0.8, 0.2]          # temperature matters more
bias    = -20                  # threshold shift
output  = neuron(inputs, weights, bias)
print(f"Output: {output:.3f}")  # closer to 1 = more anomalous
```

### Neural Network = Many Neurons in Layers

```
Input Layer    Hidden Layer 1    Hidden Layer 2    Output Layer
(sensors)      (16 neurons)      (8 neurons)       (prediction)

temp  ──┐
         ├── neuron ──┬── neuron ──┬── "normal"
hum  ──┤             ├── neuron ──┴── "anomaly"
         ├── neuron ──┘
press──┘
       ...16 neurons    ...8 neurons
```

**Training**: Show the network thousands of (input, correct_answer) pairs.
The network adjusts its weights to minimize prediction errors.

### Types of Neural Networks

| Type | Used For | Example Models |
|------|---------|---------------|
| Fully Connected (MLP) | Tabular data, classification | Simple sensor anomaly detection |
| CNN (Convolutional NN) | Images, patterns | YOLOv8, ResNet, MobileNet |
| RNN/LSTM | Sequences, time series | Time series prediction |
| Transformer | Text, code, general | GPT-4, Claude, LLaMA |
| GNN | Graph data | Routing, molecular |

---

## Chapter 2: Practical ML with Python

### Setup: Python ML Environment

```bash
# On Radxa 5B+ or Ubuntu laptop
pip3 install numpy pandas scikit-learn matplotlib
pip3 install torch torchvision  # PyTorch
# OR for lighter install:
pip3 install tflite-runtime     # for TFLite only

# Verify GPU (on Radxa, CPU only; on x86 with GPU):
python3 -c "import torch; print(torch.cuda.is_available())"
```

### Lab 1: Sensor Anomaly Detection (Classical ML)

```python
#!/usr/bin/env python3
# sensor_anomaly_detector.py
"""
Detect anomalies in industrial sensor data using Isolation Forest.
No neural network needed for this task — traditional ML works great.
"""
import numpy as np
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
import matplotlib.pyplot as plt

# Generate synthetic sensor data
np.random.seed(42)
n_samples = 1000

# Normal operation: temperature 20-40°C, pressure 900-1100 hPa
normal_temp = np.random.normal(30, 5, n_samples)
normal_pressure = np.random.normal(1000, 50, n_samples)
normal_data = np.column_stack([normal_temp, normal_pressure])

# Inject 50 anomalies (equipment failure conditions)
anomaly_temp = np.random.normal(70, 10, 50)      # overheating
anomaly_pressure = np.random.normal(500, 50, 50)  # pressure drop
anomaly_data = np.column_stack([anomaly_temp, anomaly_pressure])

# Combine
all_data = np.vstack([normal_data, anomaly_data])
true_labels = np.hstack([np.ones(n_samples), -np.ones(50)])

# Normalize
scaler = StandardScaler()
all_data_scaled = scaler.fit_transform(all_data)

# Train Isolation Forest
model = IsolationForest(
    contamination=0.05,   # expect ~5% anomalies
    n_estimators=100,
    random_state=42
)
model.fit(all_data_scaled[:n_samples])  # train on normal data only

# Predict
predictions = model.predict(all_data_scaled)
anomaly_scores = model.decision_function(all_data_scaled)

# Evaluate
from sklearn.metrics import classification_report
print(classification_report(true_labels, predictions,
                            target_names=['anomaly', 'normal']))

# Save model for deployment
import pickle
model_package = {
    'model': model,
    'scaler': scaler,
    'feature_names': ['temperature', 'pressure']
}
with open('anomaly_detector.pkl', 'wb') as f:
    pickle.dump(model_package, f)
print("Model saved: anomaly_detector.pkl")

# Deploy: load and use
def detect_anomaly(temperature, pressure):
    features = scaler.transform([[temperature, pressure]])
    score = model.decision_function(features)[0]
    is_anomaly = score < 0
    return is_anomaly, score

# Test
t, s = detect_anomaly(30, 1000)   # normal
print(f"Normal reading (30°C, 1000hPa): anomaly={t}, score={s:.3f}")
t, s = detect_anomaly(75, 450)    # anomaly
print(f"Anomaly reading (75°C, 450hPa): anomaly={t}, score={s:.3f}")
```

### Lab 2: Image Classification with CNN

```python
#!/usr/bin/env python3
# train_defect_detector.py
"""
Train a CNN to detect manufacturing defects in images.
Uses MobileNetV2 (small, fast, suitable for edge deployment).
"""
import torch
import torch.nn as nn
import torchvision.models as models
import torchvision.transforms as transforms
from torch.utils.data import DataLoader, ImageFolder

# Data preprocessing
transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                        std=[0.229, 0.224, 0.225])
])

# Dataset structure:
# data/train/normal/   ← images of good parts
# data/train/defect/   ← images of defective parts
# data/val/normal/
# data/val/defect/

train_data = ImageFolder('data/train', transform=transform)
val_data   = ImageFolder('data/val',   transform=transform)
train_loader = DataLoader(train_data, batch_size=32, shuffle=True)
val_loader   = DataLoader(val_data,   batch_size=32)

# Use pretrained MobileNetV2 (transfer learning)
model = models.mobilenet_v2(pretrained=True)

# Replace final layer for binary classification
model.classifier[1] = nn.Linear(1280, 2)  # 2 classes: normal/defect

# Train only the classifier layer first (much faster)
for param in model.features.parameters():
    param.requires_grad = False

optimizer = torch.optim.Adam(model.classifier.parameters(), lr=0.001)
criterion = nn.CrossEntropyLoss()

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

# Training loop
for epoch in range(10):
    model.train()
    train_loss = 0
    
    for images, labels in train_loader:
        images, labels = images.to(device), labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        
        train_loss += loss.item()
    
    # Validation
    model.eval()
    correct = total = 0
    with torch.no_grad():
        for images, labels in val_loader:
            images, labels = images.to(device), labels.to(device)
            outputs = model(images)
            _, predicted = outputs.max(1)
            correct += predicted.eq(labels).sum().item()
            total += labels.size(0)
    
    print(f"Epoch {epoch+1}: loss={train_loss/len(train_loader):.4f} "
          f"val_acc={100*correct/total:.1f}%")

# Export to ONNX for deployment
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(model, dummy_input, 'defect_detector.onnx',
                  opset_version=12,
                  input_names=['image'],
                  output_names=['logits'])
print("Exported: defect_detector.onnx")
```

---

## Chapter 3: Quantization — Making Models Edge-Ready

### What Is Quantization?

```python
# Float32 (standard): 4 bytes per weight
weight = 0.12345678  # 32-bit float

# INT8 quantization: 1 byte per weight (4× smaller)
# Map float range [-0.5, 0.5] to int8 range [-128, 127]
scale = 0.5 / 127
zero_point = 0
weight_int8 = round(weight / scale) + zero_point  # 25

# INT4 quantization: 0.5 byte per weight (8× smaller)
# Range [-0.5, 0.5] → int4 [-8, 7]
weight_int4 = round(weight / (0.5/7))  # 2

# Memory comparison for LLaMA 7B (7 billion parameters):
# FP32:  7B × 4 bytes = 28 GB (too much!)
# INT8:  7B × 1 byte  = 7 GB  (needs 8GB RAM)
# INT4:  7B × 0.5 bytes = 3.5 GB (fits in 4GB RAM)
```

### Post-Training Quantization with ONNX + TFLite

```python
#!/usr/bin/env python3
# quantize_model.py

# Step 1: Convert ONNX to TFLite with INT8 quantization
import tensorflow as tf
import numpy as np

# Load representative dataset (used for calibrating quantization)
def representative_dataset():
    for _ in range(100):
        # Replace with your actual calibration data
        data = np.random.random((1, 224, 224, 3)).astype(np.float32)
        yield [data]

# Convert ONNX → TF SavedModel (using onnx-tf)
# !pip install onnx-tf
import onnx
from onnx_tf.backend import prepare
onnx_model = onnx.load("defect_detector.onnx")
tf_rep = prepare(onnx_model)
tf_rep.export_graph("saved_model")

# Convert TF → TFLite INT8
converter = tf.lite.TFLiteConverter.from_saved_model("saved_model")
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8

tflite_model = converter.convert()
with open("defect_detector_int8.tflite", 'wb') as f:
    f.write(tflite_model)

# Compare sizes
import os
onnx_size = os.path.getsize("defect_detector.onnx") / 1024 / 1024
tflite_size = os.path.getsize("defect_detector_int8.tflite") / 1024 / 1024
print(f"ONNX FP32: {onnx_size:.1f} MB")
print(f"TFLite INT8: {tflite_size:.1f} MB ({onnx_size/tflite_size:.1f}× smaller)")
```

---

## Chapter 4: Model Deployment Frameworks

### Framework Comparison

| Framework | Platform | Best For | Quantization |
|-----------|---------|---------|-------------|
| TFLite | Any (CPU/GPU) | TensorFlow models | INT8 |
| RKNN | RK3588/3566 NPU | Rockchip boards | INT8 |
| TensorRT | NVIDIA GPU/Jetson | High-performance inference | INT8/FP16 |
| OpenVINO | Intel CPU/GPU/VPU | Intel hardware | INT8 |
| ONNX Runtime | Any | Cross-platform | INT8 |
| llama.cpp | Any | LLMs on CPU/GPU | INT4/INT8 |

### TFLite Deployment on ARM

```python
#!/usr/bin/env python3
# deploy_tflite.py — Run INT8 model on Radxa 5B+
import numpy as np
import cv2
import time

try:
    import tflite_runtime.interpreter as tflite
except ImportError:
    import tensorflow.lite as tflite

# Load model
interpreter = tflite.Interpreter(model_path="defect_detector_int8.tflite",
                                  num_threads=4)  # use 4 CPU cores
interpreter.allocate_tensors()

# Get input/output details
input_details  = interpreter.get_input_details()
output_details = interpreter.get_output_details()
print(f"Input shape: {input_details[0]['shape']}")   # [1, 224, 224, 3]
print(f"Input dtype: {input_details[0]['dtype']}")   # int8
print(f"Input scale: {input_details[0]['quantization']}")  # (scale, zero_point)

def preprocess(image_path, input_details):
    """Preprocess image for INT8 inference"""
    img = cv2.imread(image_path)
    img = cv2.resize(img, (224, 224))
    img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    
    # Quantize: float → int8
    scale, zero_point = input_details[0]['quantization']
    img_normalized = img.astype(np.float32) / 255.0
    img_int8 = (img_normalized / scale + zero_point).astype(np.int8)
    
    return np.expand_dims(img_int8, 0)

def predict(image_path):
    input_data = preprocess(image_path, input_details)
    
    # Set input
    interpreter.set_tensor(input_details[0]['index'], input_data)
    
    # Run inference
    start = time.perf_counter()
    interpreter.invoke()
    elapsed_ms = (time.perf_counter() - start) * 1000
    
    # Get output
    output = interpreter.get_tensor(output_details[0]['index'])
    
    # Dequantize: int8 → float
    scale, zero_point = output_details[0]['quantization']
    logits = (output.astype(np.float32) - zero_point) * scale
    
    # Softmax
    probs = np.exp(logits) / np.exp(logits).sum()
    classes = ['normal', 'defect']
    predicted = classes[np.argmax(probs[0])]
    confidence = np.max(probs[0])
    
    return predicted, confidence, elapsed_ms

# Benchmark
print("Benchmarking...")
times = []
for _ in range(100):
    _, _, ms = predict("test_image.jpg")
    times.append(ms)

print(f"Inference time: {np.mean(times):.1f}ms ± {np.std(times):.1f}ms")
print(f"FPS: {1000/np.mean(times):.0f}")
```

---

## Chapter 5: LLM Internals and Deployment

### How LLM Inference Works (Step by Step)

```python
"""
LLM inference pipeline — what happens when you call llama.cpp or the Anthropic API

Input: "What is a Linux device driver?"

Step 1: TOKENIZATION
  "What" → token 3921
  " is" → token 374
  " a" → token 264
  " Linux" → token 8074
  ...
  Input sequence: [3921, 374, 264, 8074, ...]

Step 2: EMBEDDING
  Each token ID → 4096-dimensional vector (for 7B model)
  [3921] → [0.123, -0.456, 0.789, ...] (4096 numbers)

Step 3: TRANSFORMER LAYERS (repeated 32 times in 7B model)
  Each layer has:
  a) Multi-head attention: tokens attend to each other
     Q = input × W_Q   (queries: "what am I looking for?")
     K = input × W_K   (keys: "what do I have?")
     V = input × W_V   (values: "what info do I provide?")
     Attention = softmax(Q × K^T / sqrt(d)) × V
     
  b) Feed-forward network:
     intermediate = relu(input × W1 + b1)  (hidden: 16384 dim)
     output = intermediate × W2 + b2       (back to 4096 dim)

Step 4: OUTPUT PROJECTION
  Last layer output × W_out → vocabulary logits (50,000 numbers)
  Each number = probability of that token being next

Step 5: SAMPLING
  Pick next token based on probabilities (temperature controls randomness)
  Append to sequence, go back to step 3 for next token

Step 6: DETOKENIZATION
  [4277, 13073, ...] → "A device driver is..."
"""

# Memory required for 7B model:
model_params = 7e9          # 7 billion parameters
fp32_bytes = model_params * 4  # 28 GB
int8_bytes = model_params * 1  # 7 GB  
int4_bytes = model_params * 0.5  # 3.5 GB

print(f"7B FP32: {fp32_bytes/1e9:.0f} GB")   # 28 GB
print(f"7B INT8: {int8_bytes/1e9:.0f} GB")    # 7 GB
print(f"7B INT4: {int4_bytes/1e9:.1f} GB")   # 3.5 GB
```

### KV Cache — Why Token Generation Speeds Up

```python
"""
Without KV Cache:
  Token 1: compute attention over 1 token
  Token 2: compute attention over 2 tokens (RE-COMPUTES token 1!)
  Token 3: compute attention over 3 tokens (RE-COMPUTES tokens 1+2!)
  ...
  Very slow and wasteful!

With KV Cache:
  Token 1: compute K,V → store in cache
  Token 2: compute K,V for NEW token only → use cached K,V from token 1
  Token 3: compute K,V for NEW token only → use cached K,V from tokens 1+2
  ...
  Only compute new token's K,V each step = much faster!

KV Cache memory (for 7B model, sequence length 2048):
  n_layers = 32
  n_heads = 32
  head_dim = 128
  seq_len = 2048
  
  KV_size = 2 (K and V) × n_layers × n_heads × head_dim × seq_len × 2 bytes (fp16)
          = 2 × 32 × 32 × 128 × 2048 × 2
          = 1,073,741,824 bytes = 1 GB

So 7B INT4 on 8GB RAM:
  Model: 3.5 GB
  KV Cache: 1 GB
  OS + other: 1 GB
  Available: 2.5 GB buffer
  → 8GB RAM needed minimum
"""
```

### Running LLMs Efficiently on Embedded

```bash
# Strategy 1: llama.cpp (best for CPU, works on Radxa)
# See 23_Radxa_5B_Plus_Labs/02_LLM_Deployment_Guide.md

# Strategy 2: Ollama (Docker-like for LLMs)
# Install on Radxa:
curl -fsSL https://ollama.com/install.sh | sh

# Pull and run models
ollama pull llama3.2:1b       # 1B params, fast on ARM
ollama pull llama3.2:3b       # 3B params, good quality
ollama pull phi3:mini         # 3.8B, optimized for edge

# Run
ollama run llama3.2:1b "Explain what a Linux driver is in 3 sentences"

# API (compatible with OpenAI API)
curl http://localhost:11434/api/generate \
    -d '{"model": "llama3.2:1b", "prompt": "What is a device driver?"}'

# Strategy 3: llama.cpp server
./llama-server -m llama3.2-3b.Q4_K_M.gguf -c 2048 --host 0.0.0.0 --port 8080

# Benchmark: how fast?
# LLaMA 3.2 1B Q4 on Radxa:  12-15 tokens/second (good for chat)
# LLaMA 3.2 3B Q4 on Radxa:   5-7 tokens/second (acceptable)
# LLaMA 3.1 7B Q4 on Radxa:   2-3 tokens/second (slow but works)
```

---

## Chapter 6: Building an Embedded AI Product

### Product Concept: Predictive Maintenance Box

```
Product: "PredictAI Box"
  Hardware: Radxa Rock 5B+ in industrial enclosure
  Sensors: 4× vibration (ADXL345) + 4× temperature (SHT31)
  AI: Anomaly detection model (Isolation Forest → RKNN CNN)
  Dashboard: Local web UI + optional cloud sync
  Price: ₹25,000 hardware + ₹500/month SaaS
  
Revenue model:
  Year 1: Sell hardware + setup (₹25K-50K per installation)
  Year 2: Add SaaS tier (₹5K-15K/month per customer)
  Year 3: 10 customers × ₹10K/month = ₹12 lakhs ARR
```

### Complete Tech Stack

```python
# requirements.txt for complete embedded AI product
numpy==1.24.0
scikit-learn==1.3.0
onnxruntime==1.16.0           # for ONNX inference on CPU
tflite-runtime==2.13.0        # for TFLite on ARM
rknn-toolkit-lite==1.6.0      # for NPU inference (Rockchip only)
fastapi==0.103.0
uvicorn==0.23.0
paho-mqtt==1.6.1               # IoT message bus
influxdb-client==1.37.0        # time-series database
smbus2==0.4.3                  # I2C sensors
spidev==3.6                    # SPI sensors
gpiod==2.0.2                   # GPIO control
anthropic==0.25.0              # Claude API for AI assistant
langchain==0.1.0               # AI agent framework
```

---

## Chapter 7: AI Era Career Positioning

### The Skills Pyramid for Embedded AI Engineers

```
           ╔══════════════════════╗
           ║  AI PRODUCT BUILDER  ║  ← Top 1% — design products
           ╟──────────────────────╢
           ║  EDGE AI ARCHITECT   ║  ← Top 5% — design AI systems
           ╟──────────────────────╢
           ║  EMBEDDED AI DEV     ║  ← Top 15% — deploy AI on hardware
           ╟──────────────────────╢
           ║  EMBEDDED LINUX DEV  ║  ← Top 30% — current state
           ╚══════════════════════╝

Ravi's target: "Embedded AI Architect" in 24 months
```

### Your 6-Month Learning Path

```
Month 1-2: ML Fundamentals
  ✓ Complete Chapter 1-2 of this guide
  ✓ Build: sensor anomaly detector (scikit-learn)
  ✓ Build: defect detector CNN (PyTorch + MobileNetV2)
  ✓ Deploy both on Radxa 5B+

Month 3-4: Edge Deployment
  ✓ Learn: RKNN conversion pipeline
  ✓ Build: YOLOv8 on NPU (35 FPS)
  ✓ Build: Whisper STT system
  ✓ Benchmark: CPU vs NPU comparison table

Month 5-6: LLMs + Agents
  ✓ Deploy: LLaMA 3.2 3B on Radxa
  ✓ Build: kernel log analyzer agent (from 19_AI_Agents/)
  ✓ Build: voice assistant (Whisper + LLaMA + eSpeak)
  ✓ GitHub: publish all projects with benchmarks

Output: 3 portfolio projects with measurable results
```
