# Study Plan — 29_QEMU Embedded AI Labs

> **Goal:** Complete mastery of QEMU for embedded development and AI deployment.  
> By end of this plan you will: build a BSP from scratch, run LLMs inside QEMU,  
> write a custom device model, and build an AI-powered debug pipeline.

---

## Prerequisites Checklist

Complete these before starting Week 1:

```
□ C programming — comfortable with pointers, structs, function pointers
  Ref: ../01_C_Programming/01_Multithreading_And_Patterns.md

□ Linux kernel basics — modules, drivers, device tree
  Ref: ../06_Linux_Kernel/01_Kernel_Internals_Deep_Dive.md

□ Device driver fundamentals — platform driver, IRQ, MMIO
  Ref: ../07_Device_Drivers/01_Driver_From_Scratch_Complete_Guide.md

□ Tools installed on your laptop:
  sudo apt-get install -y \
    gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    qemu-system-aarch64 \
    bc flex bison libssl-dev libncurses-dev \
    device-tree-compiler cpio
  
□ Python 3.10+ with pip

□ 50 GB free disk space (kernel + toolchain + QEMU builds)

□ 16 GB RAM recommended (run QEMU with 4GB guest)
```

---

## Overview: 6-Week Plan

```
Week 1 ── Lab Environment + First QEMU Boot (Beginner)
Week 2 ── Kernel, U-Boot, Custom Rootfs (Beginner → Intermediate)
Week 3 ── Device Drivers Inside QEMU + GDB Debugging (Intermediate)
Week 4 ── AI/ML Frameworks Running on ARM64 QEMU (Intermediate)
Week 5 ── LLMs Inside QEMU + AI Debug Pipeline (Advanced)
Week 6 ── Custom QEMU Device Model (Advanced → Expert)
```

---

## Week 1 — Lab Environment + First QEMU Boot

**Time:** 8-10 hours total  
**Goal:** QEMU boots ARM64 Linux from your own built components  
**Success metric:** See `/ #` shell prompt running your custom initramfs

### Day 1 (2 hours): Environment Setup

```bash
# Step 1: Install all tools
sudo apt-get install -y \
  gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
  qemu-system-aarch64 \
  bc flex bison libssl-dev libncurses-dev \
  device-tree-compiler cpio gzip

# Step 2: Verify everything
qemu-system-aarch64 --version
aarch64-linux-gnu-gcc --version
dtc --version

# Step 3: Create workspace
mkdir -p ~/qemu-lab/{toolchain,u-boot,linux,busybox,rootfs,images,drivers}
echo "export CROSS=aarch64-linux-gnu-" >> ~/.bashrc
echo "export ARCH=arm64" >> ~/.bashrc
source ~/.bashrc
```

**Study:** Read README.md sections:
- "Why QEMU for Embedded AI" 
- "What QEMU Can and Can't Simulate" table

**Q&A practice:**
- Q: What is the difference between `qemu-system-aarch64` and `qemu-user`?
- Q: What is an initramfs and why is it used for quick boot?

---

### Day 2 (2 hours): Build U-Boot for QEMU

```bash
cd ~/qemu-lab
git clone https://source.denx.de/u-boot/u-boot.git --depth=1 --branch v2024.01
cd u-boot

make ARCH=arm64 CROSS_COMPILE=$CROSS qemu_arm64_defconfig
make ARCH=arm64 CROSS_COMPILE=$CROSS -j$(nproc)

# Test U-Boot
qemu-system-aarch64 \
  -machine virt -cpu cortex-a57 -m 1G \
  -bios u-boot.bin -nographic -serial mon:stdio

# At U-Boot prompt: try these commands
# => help
# => version  
# => printenv
# => bdinfo
# Ctrl-A X to exit
```

**Checkpoint questions:**
- Can you see the U-Boot prompt?
- What does `bdinfo` show? (board info — memory address, CPU)
- What does `printenv` show? (boot environment variables)

---

### Day 3 (2 hours): Build Linux Kernel

```bash
cd ~/qemu-lab
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git \
  --depth=1 --branch v6.6.30
cd linux

make ARCH=arm64 CROSS_COMPILE=$CROSS defconfig
make ARCH=arm64 CROSS_COMPILE=$CROSS Image -j$(nproc)

echo "Done: $(ls -lh arch/arm64/boot/Image)"
```

**Exploration:** Open menuconfig and understand these sections:
```bash
make ARCH=arm64 CROSS_COMPILE=$CROSS menuconfig
# Explore (don't change, just read):
# → Kernel type → Kernel compression mode
# → Device Drivers → Character devices → Serial drivers
# → Device Drivers → Virtio drivers
# → Kernel hacking → Memory debugging
```

**Q&A practice:**
- Q: What is `defconfig` and how is it different from `allmodconfig`?
- Q: What does `CONFIG_VIRTIO` enable and why is it needed for QEMU?

---

### Day 4 (2 hours): Build Minimal RootFS + First Complete Boot

Follow the "Build Minimal RootFS" section in README.md exactly.

```bash
cd ~/qemu-lab
# ... (full commands in README.md)

# FINAL BOOT COMMAND:
qemu-system-aarch64 \
  -machine virt,gic-version=3 \
  -cpu cortex-a76 -smp 4 -m 4G \
  -kernel linux/arch/arm64/boot/Image \
  -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init loglevel=7" \
  -nographic -no-reboot
```

**Week 1 Deliverable:**  
Screenshot of `/ #` prompt showing these commands working:
```sh
/ # cat /proc/cpuinfo | grep "model name"
/ # cat /proc/meminfo | head -5
/ # ls /dev/
/ # dmesg | head -20
```

---

## Week 2 — Kernel Configuration + Custom Devices

**Time:** 8-10 hours total  
**Goal:** Boot with network, a virtio block device, and your own init script

### Day 5 (2 hours): Add Network to QEMU

```bash
# New QEMU launch with networking
qemu-system-aarch64 \
  -machine virt,gic-version=3 \
  -cpu cortex-a76 -smp 4 -m 4G \
  -kernel linux/arch/arm64/boot/Image \
  -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init" \
  -nographic -no-reboot \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0

# Inside QEMU guest:
# / # ifconfig eth0 10.0.2.15 netmask 255.255.255.0
# / # ping 10.0.2.2   (host gateway)
```

**Study:** Why does QEMU use virtio-net instead of a real Ethernet model?  
Read: README.md "What QEMU Can and Can't Simulate" table again — focus on VirtIO row.

---

### Day 6 (2 hours): Host Folder Sharing (9P Filesystem)

```bash
# Create shared folder
mkdir -p ~/qemu-lab/shared

# Add 9P sharing to QEMU:
qemu-system-aarch64 \
  ... \
  -virtfs local,path=/home/$USER/qemu-lab/shared,mount_tag=host0,\
security_model=passthrough,id=host0

# Inside QEMU:
# / # mount -t 9p -o trans=virtio host0 /mnt
# / # ls /mnt   # sees ~/qemu-lab/shared from host
```

This is how you'll transfer binaries into QEMU for the AI labs later.

---

### Day 7 (2 hours): Add a Simple Kernel Module to Your Rootfs

```bash
# Write a hello world kernel module
mkdir -p ~/qemu-lab/drivers/hello
cat > ~/qemu-lab/drivers/hello/hello.c << 'EOF'
#include <linux/module.h>
#include <linux/init.h>
static int __init hello_init(void) {
    pr_info("Hello from QEMU ARM64!\n");
    return 0;
}
static void __exit hello_exit(void) {
    pr_info("Goodbye from QEMU ARM64!\n");
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Hello QEMU");
EOF

cat > ~/qemu-lab/drivers/hello/Makefile << 'EOF'
obj-m := hello.o
KDIR := /home/$USER/qemu-lab/linux
all:
    make -C $(KDIR) M=$(PWD) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
clean:
    make -C $(KDIR) M=$(PWD) clean
EOF

make -C ~/qemu-lab/drivers/hello

# Copy hello.ko to shared folder, load inside QEMU:
# / # insmod /mnt/hello.ko
# / # dmesg | tail -3
# / # rmmod hello
```

**Week 2 Deliverable:**  
`dmesg | grep "Hello from QEMU"` shows your message inside QEMU.

---

## Week 3 — QEMU + GDB Kernel Debugging

**Time:** 8-10 hours total  
**Goal:** Set breakpoints in running kernel code from your laptop  
**Reference:** `../09_Debugging/02_GDB_KGDB_Ftrace_eBPF_Complete.md`

### Day 8 (2 hours): Connect GDB to QEMU Kernel

```bash
# Terminal 1: Launch QEMU with GDB server
qemu-system-aarch64 \
  -machine virt,gic-version=3 \
  -cpu cortex-a76 -smp 4 -m 4G \
  -kernel ~/qemu-lab/linux/arch/arm64/boot/Image \
  -initrd ~/qemu-lab/initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init nokaslr" \
  -nographic -no-reboot \
  -s -S   # -s: gdb port 1234, -S: freeze at start

# Terminal 2: Connect GDB
aarch64-linux-gnu-gdb ~/qemu-lab/linux/vmlinux

# In GDB:
# (gdb) target remote :1234
# (gdb) break start_kernel
# (gdb) c            ← continue until start_kernel
# (gdb) bt           ← backtrace
# (gdb) n            ← next line
# (gdb) p init_uts_ns.name.version   ← print kernel version
```

---

### Day 9-10 (4 hours): GDB Debug Your Hello Module

```bash
# Build module with debug symbols
make -C ~/qemu-lab/drivers/hello KCFLAGS="-g -O0"

# Load module inside QEMU, find its load address:
# / # insmod /mnt/hello.ko
# / # cat /proc/modules | grep hello
# hello 16384 0 - Live 0xffff800000123000

# In GDB (add symbols at load address):
# (gdb) add-symbol-file ~/qemu-lab/drivers/hello/hello.ko 0xffff800000123000
# (gdb) break hello_init
# (gdb) c
# Load the module in QEMU → GDB stops at hello_init!
# (gdb) list        ← show source
# (gdb) info locals ← local variables
```

**Day 10:** Use ftrace to trace module function calls:
```bash
# Inside QEMU:
# / # echo function > /sys/kernel/debug/tracing/current_tracer
# / # echo hello_init > /sys/kernel/debug/tracing/set_ftrace_filter
# / # echo 1 > /sys/kernel/debug/tracing/tracing_on
# / # insmod /mnt/hello.ko
# / # cat /sys/kernel/debug/tracing/trace
```

**Week 3 Deliverable:**  
GDB session screenshot showing a breakpoint hit inside your kernel module.

---

## Week 4 — AI Frameworks on ARM64 QEMU

**Time:** 10-12 hours total  
**Goal:** Run TFLite and ONNX Runtime inside QEMU, benchmark inference  
**Reference:** `../18_AI_For_Embedded/01_Complete_Embedded_AI_Track.md`

### Day 11 (2 hours): Cross-Compile TFLite for ARM64

```bash
# Install CMake (needed for TFLite)
sudo apt-get install -y cmake ninja-build

# Clone TensorFlow Lite
git clone --depth=1 https://github.com/tensorflow/tensorflow ~/qemu-lab/tensorflow
cd ~/qemu-lab/tensorflow

# Cross-compile for aarch64
mkdir build-arm64 && cd build-arm64
cmake ../tensorflow/lite \
  -DCMAKE_TOOLCHAIN_FILE=../tensorflow/lite/tools/cmake/toolchains/aarch64_linux.cmake \
  -DTFLITE_ENABLE_XNNPACK=ON \
  -DCMAKE_BUILD_TYPE=Release
cmake --build . -j$(nproc)

# Output: libtensorflow-lite.a (static library for ARM64)
ls -lh libtensorflow-lite.a
```

---

### Day 12-13 (4 hours): Run Inference Inside QEMU

```bash
# Download a tiny pre-quantized model
wget https://storage.googleapis.com/download.tensorflow.org/models/tflite/mobilenet_v1_1.0_224_quant.tgz
tar xzf mobilenet_v1_1.0_224_quant.tgz
# → mobilenet_v1_1.0_224_quant.tflite (about 4MB)

# Copy model to shared folder
cp mobilenet_v1_1.0_224_quant.tflite ~/qemu-lab/shared/

# Write a minimal inference benchmark
cat > ~/qemu-lab/shared/bench_tflite.py << 'EOF'
#!/usr/bin/env python3
import time
import numpy as np

try:
    import tflite_runtime.interpreter as tflite
except ImportError:
    import tensorflow.lite as tflite

MODEL = "/mnt/mobilenet_v1_1.0_224_quant.tflite"

interp = tflite.Interpreter(model_path=MODEL, num_threads=4)
interp.allocate_tensors()

inp = interp.get_input_details()[0]
out = interp.get_output_details()[0]
dummy = np.random.randint(0, 255, inp['shape'], dtype=np.uint8)

# Warmup
for _ in range(5):
    interp.set_tensor(inp['index'], dummy)
    interp.invoke()

# Benchmark
runs = 50
t0 = time.perf_counter()
for _ in range(runs):
    interp.set_tensor(inp['index'], dummy)
    interp.invoke()
elapsed = (time.perf_counter() - t0) / runs * 1000

print(f"Model: MobileNetV1 INT8")
print(f"Threads: 4")
print(f"Inference: {elapsed:.1f} ms/frame")
print(f"FPS: {1000/elapsed:.1f}")
EOF

# Inside QEMU (with Python + tflite-runtime installed):
# / # pip3 install tflite-runtime
# / # python3 /mnt/bench_tflite.py
```

---

### Day 14 (2 hours): ONNX Runtime on QEMU

```bash
# Download pre-built ONNX Runtime for aarch64
pip3 download onnxruntime==1.17.0 --platform linux_aarch64 \
  --python-version 310 --only-binary=:all: \
  --dest ~/qemu-lab/shared/

# Inside QEMU:
# / # pip3 install /mnt/onnxruntime-1.17.0-cp310-cp310-linux_aarch64.whl
```

**Benchmark comparison table — fill this in:**

| Framework | Model | Threads | ms/frame | FPS |
|-----------|-------|---------|---------|-----|
| TFLite INT8 | MobileNetV1 | 4 | ? | ? |
| ONNX Runtime | MobileNetV1 | 4 | ? | ? |
| TFLite INT8 | MobileNetV1 | 1 | ? | ? |

**Week 4 Deliverable:**  
Completed benchmark table with real numbers from your QEMU run.

---

## Week 5 — LLMs Inside QEMU + AI Debug Pipeline

**Time:** 10-12 hours total  
**Goal:** Run a 1B parameter LLM inside QEMU, build the kernel log analyzer  
**Reference:** README.md sections "Running LLMs Inside QEMU" and "Project 4"

### Day 15 (2 hours): Cross-Compile llama.cpp

```bash
cd ~/qemu-lab
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

mkdir build-arm64 && cd build-arm64
cmake .. \
  -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc \
  -DCMAKE_CXX_COMPILER=aarch64-linux-gnu-g++ \
  -DLLAMA_NATIVE=OFF \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_OPENMP=OFF
cmake --build . -j$(nproc) --target llama-cli

# Copy to shared
cp bin/llama-cli ~/qemu-lab/shared/
```

---

### Day 16 (2 hours): Download a Tiny Model + Run Inside QEMU

```bash
# Download TinyLLaMA 1.1B Q4 (about 650MB)
cd ~/qemu-lab/shared/
wget https://huggingface.co/TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf

# Inside QEMU:
# / # /mnt/llama-cli \
#     -m /mnt/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf \
#     -p "Explain Linux device probe() function in 3 sentences" \
#     -n 100 -t 4

# Note the tokens/second — this is CPU-only ARM64 performance
```

---

### Day 17-18 (4 hours): Build the AI Kernel Log Analyzer

Copy the `log_analyzer.py` script from README.md and run it:

```bash
# On your HOST machine (Claude API or local Ollama)
cp ~/qemu-lab/shared/log_analyzer.py ~/

# Test with a real dmesg log:
dmesg > /tmp/my_kernel.log
python3 log_analyzer.py /tmp/my_kernel.log --model local

# Install Ollama locally for free inference:
curl -fsSL https://ollama.com/install.sh | sh
ollama pull codellama:7b

# Run analyzer with CodeLlama
python3 log_analyzer.py /tmp/my_kernel.log --model local \
    --ollama-model codellama:7b
```

**Extend the analyzer:**
1. Add a `--watch` mode that monitors dmesg in real-time
2. Add severity filtering: only analyze CRITICAL/HIGH
3. Add a summary email/notification when CRITICAL errors occur

---

### Day 19 (2 hours): LLM as Interactive Debug Assistant

```bash
# Build a REPL-style debug session
cat > ~/qemu-lab/shared/debug_repl.py << 'EOF'
#!/usr/bin/env python3
"""Interactive embedded debug session powered by local LLM."""
import requests
import sys

SYSTEM = """You are an expert embedded Linux kernel engineer.
You help debug kernel issues, driver problems, and boot failures.
Give concise, actionable answers. Always suggest the exact command to run next."""

history = []

def ask(question):
    history.append({"role": "user", "content": question})
    resp = requests.post(
        "http://localhost:11434/api/chat",
        json={"model": "codellama:7b", "messages": [
            {"role": "system", "content": SYSTEM}
        ] + history, "stream": False}
    )
    answer = resp.json()["message"]["content"]
    history.append({"role": "assistant", "content": answer})
    return answer

print("Embedded Debug Assistant (local AI). Type 'exit' to quit.")
while True:
    try:
        q = input("\n[debug] > ")
    except (EOFError, KeyboardInterrupt):
        break
    if q.strip().lower() == "exit":
        break
    print(ask(q))
EOF

python3 debug_repl.py
# Try: "My driver probe returns -ENODEV, what should I check?"
# Try: "What does 'BUG: sleeping function called from invalid context' mean?"
```

**Week 5 Deliverable:**  
`log_analyzer.py` running against a real dmesg log, producing structured JSON analysis.

---

## Week 6 — Custom QEMU Device Model

**Time:** 12-15 hours total  
**Goal:** Write a working QEMU device model, test with a Linux driver  
**Reference:** README.md "Custom QEMU Device Model" + QEMU Architecture Diagram

### Day 20 (3 hours): Build QEMU from Source

```bash
# Build QEMU from source (needed to add your device model)
cd ~/qemu-lab
git clone https://gitlab.com/qemu-project/qemu.git --depth=1 --branch v8.2.0
cd qemu

# Configure
./configure \
  --target-list=aarch64-softmmu \
  --enable-debug \
  --disable-docs \
  --prefix=/home/$USER/qemu-lab/qemu-install

make -j$(nproc)
make install

~/qemu-lab/qemu-install/bin/qemu-system-aarch64 --version
```

---

### Day 21-22 (4 hours): Add Your Device Model

```bash
# Copy the minimal MMIO device from README.md
mkdir -p ~/qemu-lab/qemu/hw/misc/
cp ~/qemu-lab/ravi_demo_device.c ~/qemu-lab/qemu/hw/misc/

# Register it in the build system
echo 'system_ss.add(files("ravi_demo_device.c"))' >> \
  ~/qemu-lab/qemu/hw/misc/meson.build

# Add to virt machine (hw/arm/virt.c) — declare as a sysbus device:
# DeviceState *dev = qdev_new("ravi-demo-device");
# sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
# sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, 0x09000000);

# Rebuild QEMU
cd ~/qemu-lab/qemu
make -j$(nproc) && make install
```

---

### Day 23-24 (4 hours): Write the Linux Driver for Your Device

```bash
cat > ~/qemu-lab/drivers/ravi_demo/ravi_demo_driver.c << 'EOF'
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/io.h>
#include <linux/interrupt.h>

#define REG_STATUS    0x00
#define REG_CONTROL   0x04
#define REG_DATA      0x08
#define REG_IRQ_STATUS 0x0C
#define REG_IRQ_ENABLE 0x10

struct ravi_dev {
    void __iomem *base;
    int irq;
};

static irqreturn_t ravi_irq_handler(int irq, void *dev_id)
{
    struct ravi_dev *d = dev_id;
    u32 status = readl(d->base + REG_IRQ_STATUS);
    writel(status, d->base + REG_IRQ_STATUS); /* clear (W1C) */
    pr_info("ravi_demo: IRQ! status=0x%x\n", status);
    return IRQ_HANDLED;
}

static ssize_t trigger_store(struct device *dev,
    struct device_attribute *attr, const char *buf, size_t count)
{
    struct ravi_dev *d = dev_get_drvdata(dev);
    u32 val;
    if (kstrtou32(buf, 0, &val))
        return -EINVAL;
    writel(val, d->base + REG_DATA);
    writel(0x01, d->base + REG_CONTROL);  /* trigger operation */
    return count;
}
static DEVICE_ATTR_WO(trigger);

static int ravi_demo_probe(struct platform_device *pdev)
{
    struct ravi_dev *d;
    struct resource *res;

    d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
    if (!d) return -ENOMEM;

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    d->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(d->base)) return PTR_ERR(d->base);

    d->irq = platform_get_irq(pdev, 0);
    if (d->irq < 0) return d->irq;

    if (devm_request_irq(&pdev->dev, d->irq, ravi_irq_handler,
                          0, "ravi_demo", d))
        return -EBUSY;

    platform_set_drvdata(pdev, d);
    device_create_file(&pdev->dev, &dev_attr_trigger);

    dev_info(&pdev->dev, "probed at %pa, irq %d\n", &res->start, d->irq);
    return 0;
}

static const struct of_device_id ravi_demo_ids[] = {
    { .compatible = "ravi,demo-device" },
    {}
};
MODULE_DEVICE_TABLE(of, ravi_demo_ids);

static struct platform_driver ravi_demo_driver = {
    .probe = ravi_demo_probe,
    .driver = { .name = "ravi_demo", .of_match_table = ravi_demo_ids },
};
module_platform_driver(ravi_demo_driver);
MODULE_LICENSE("GPL");
EOF

# Build and load
make -C ~/qemu-lab/drivers/ravi_demo

# Inside QEMU (with device tree node for ravi,demo-device):
# / # insmod /mnt/ravi_demo_driver.ko
# / # dmesg | grep ravi
# / # echo 0x42 > /sys/devices/platform/ravi_demo/trigger
# → IRQ fires, kernel message appears!
```

---

### Day 25 (2 hours): Week 6 Wrap-Up + Lessons from DW-UFS 4.0

Study the architectural diagram in README.md ("QEMU Device Model Architecture").

**Reflection exercise** — answer these from your DW-UFS 4.0 experience:

```
1. What was the hardest part of the DW-UFS QEMU device model?
   (DDR training? UPIU parsing? Command Queue scheduling? DMA descriptor ring?)

2. How did you test the UFS device model?
   (compared behavior against real hardware traces? used QEMU monitor to inspect state?)

3. What would you change if you started the DW-UFS model today?
   (better VMState for snapshot support? QTest for automated testing?)
```

**Week 6 Deliverable:**  
Custom QEMU device model compiled, loaded driver, IRQ fires on write — proven with dmesg output.

---

## Final Project: End-to-End Embedded AI Debug Platform

**After Week 6, build this capstone project:**

```
Architecture:
  QEMU ARM64 guest (Linux 6.6)
       ↓ runs
  Your custom test driver
       ↓ generates
  Kernel error logs (/dev/kmsg)
       ↓ piped to
  log_analyzer.py on host
       ↓ analyzed by
  Local LLM (Ollama/codellama)
       ↓ produces
  JSON report with debug steps
       ↓ displayed in
  Simple terminal dashboard

Challenge: Can your AI correctly identify the injected bug?
```

```bash
# Script to inject a controlled bug in QEMU guest:
# / # echo "BUG: unable to handle kernel NULL pointer dereference" > /dev/kmsg
# / # echo "PC is at ravi_demo_probe+0x24/0x80 [ravi_demo_driver]" >> /dev/kmsg

# On host, watch and analyze:
ssh -p 2222 root@localhost 'dmesg -w' | python3 log_analyzer.py --model local
```

---

## Progress Tracking Table

| Week | Lab | Deliverable | Status | Date Done |
|------|-----|-------------|--------|-----------|
| 1 | QEMU first boot | `/ #` shell prompt | ☐ | |
| 1 | U-Boot prompt | `=>` U-Boot shell | ☐ | |
| 2 | Network in QEMU | `ping 10.0.2.2` works | ☐ | |
| 2 | Hello module | `dmesg` shows Hello | ☐ | |
| 3 | GDB breakpoint | Hit `start_kernel` in GDB | ☐ | |
| 3 | Module debugging | Breakpoint in hello_init | ☐ | |
| 4 | TFLite benchmark | ms/frame number | ☐ | |
| 4 | ONNX benchmark | Comparison table done | ☐ | |
| 5 | LLM in QEMU | tokens/sec number | ☐ | |
| 5 | Log analyzer | JSON analysis from dmesg | ☐ | |
| 6 | Device model | IRQ fires on write | ☐ | |
| 6 | Driver + model | Full round-trip working | ☐ | |
| Final | AI debug platform | Auto-analyzes injected bug | ☐ | |

---

## Key Interview Questions — Track Progress

After completing each week, answer these without looking:

**After Week 1-2:**
- [ ] What are the components of a minimal embedded Linux system?
- [ ] How does an initramfs differ from a full rootfs on disk?
- [ ] What does the `-smp 4` QEMU option control?

**After Week 3:**
- [ ] How do you connect GDB to a running QEMU kernel?
- [ ] What is `nokaslr` and why do you need it for kernel debugging?
- [ ] Explain how ftrace `function` tracer works.

**After Week 4-5:**
- [ ] What is the difference between TFLite and ONNX Runtime?
- [ ] What is INT8 quantization and why does it matter for embedded AI?
- [ ] How does KV cache speed up LLM token generation?

**After Week 6:**
- [ ] Walk through a QEMU guest `writel()` → device model C function call chain.
- [ ] What is `VMState` in QEMU and why is it needed for upstream contribution?
- [ ] Design a QEMU device model for a DMA-capable AI accelerator.

---

## Related Study Files (Read Alongside This Plan)

| When | Read This | Why |
|------|-----------|-----|
| Before Week 1 | [../07_Device_Drivers/01_Driver_From_Scratch_Complete_Guide.md](../07_Device_Drivers/01_Driver_From_Scratch_Complete_Guide.md) | Understand driver patterns before writing QEMU device |
| Week 2 | [../05_Bootloaders/01_Complete_Boot_Flow_Deep_Dive.md](../05_Bootloaders/01_Complete_Boot_Flow_Deep_Dive.md) | Understand what U-Boot does, why SPL exists |
| Week 3 | [../09_Debugging/02_GDB_KGDB_Ftrace_eBPF_Complete.md](../09_Debugging/02_GDB_KGDB_Ftrace_eBPF_Complete.md) | Full GDB + ftrace + eBPF reference |
| Week 4-5 | [../18_AI_For_Embedded/01_Complete_Embedded_AI_Track.md](../18_AI_For_Embedded/01_Complete_Embedded_AI_Track.md) | ML fundamentals, quantization, deployment |
| Week 5 | [../19_AI_Agents/01_Complete_Agent_Building.md](../19_AI_Agents/01_Complete_Agent_Building.md) | LangGraph agents, extend the log analyzer |
| Week 6 | [../17_QEMU_Virtualization/01_QEMU_Projects_Complete_Guide.md](../17_QEMU_Virtualization/01_QEMU_Projects_Complete_Guide.md) | 12 QEMU projects for additional practice |
| Any time | [../12_Open_Source/01_Kernel_Contribution_Complete_Roadmap.md](../12_Open_Source/01_Kernel_Contribution_Complete_Roadmap.md) | How to upstream your QEMU device model |
