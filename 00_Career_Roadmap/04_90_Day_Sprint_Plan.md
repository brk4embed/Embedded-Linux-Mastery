# 04 — 90-Day Sprint Plan

> **Start today.** These 90 days will produce more visible progress than the last 2 years.  
> The plan is aggressive but realistic given your existing foundation.

---

## The Philosophy

- **Build, don't just read.** Every week must produce a working artifact.
- **Show your work.** Every project goes on GitHub. Every learning goes on LinkedIn.
- **AI accelerates, not replaces.** Use Claude/Copilot for everything, but understand what you build.
- **Time box.** 2 hours minimum per day. Non-negotiable.

---

## Month 1 (Days 1–30): Foundation Rebuild + QEMU BSP Mastery

### Week 1 (Days 1–7): Environment + First Build

**Day 1–2: Dev Environment Setup**
```bash
# On your Ubuntu 22.04/24.04 host machine:

# Install QEMU 8.x
sudo apt update && sudo apt install -y qemu-system-aarch64 qemu-user-static

# Install ARM64 cross-compiler (Linaro)
sudo apt install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# Verify
aarch64-linux-gnu-gcc --version
qemu-system-aarch64 --version

# Install development tools
sudo apt install -y \
  git vim neovim tmux build-essential cmake ninja-build \
  libssl-dev libncurses-dev flex bison bc \
  device-tree-compiler fdtdump \
  minicom tio picocom \
  gdb-multiarch

# Install Python for AI work
sudo apt install -y python3 python3-pip python3-venv
pip3 install langchain langgraph anthropic openai

# Install Ollama (local LLM)
curl -fsSL https://ollama.com/install.sh | sh
ollama pull tinyllama    # 669MB — your offline AI assistant
```

**Day 3–5: Build U-Boot for QEMU from scratch**
```bash
mkdir ~/embedded-lab && cd ~/embedded-lab

# Clone U-Boot
git clone https://source.denx.de/u-boot/u-boot.git --depth=1
cd u-boot

# Configure for QEMU ARM64
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- qemu_arm64_defconfig

# Build
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# Test — you should see the U-Boot prompt
qemu-system-aarch64 -M virt -cpu cortex-a57 -bios u-boot.bin -nographic

# At U-Boot prompt:
# => help
# => version
# => printenv
# => Ctrl-A, then X to exit QEMU
```

**Goal:** U-Boot prompt in QEMU by end of Day 5.

**Day 6–7: Build Linux kernel for QEMU**
```bash
cd ~/embedded-lab

# Clone kernel (shallow for speed — full clone is 4GB)
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git \
  --depth=1 --branch v6.6 linux
cd linux

# Configure for ARM64
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

# Enable virtio drivers (needed for QEMU disk/network)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
# Navigate: Device Drivers → Virtio drivers → Enable all virtio
# Navigate: File systems → Enable ext4 and 9P

# Build kernel image
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j$(nproc)
# Output: arch/arm64/boot/Image
```

**Goal:** Compiled kernel Image by end of Day 7. Push to GitHub.

---

### Week 2 (Days 8–14): RootFS + First Complete Boot

**Day 8–10: Build BusyBox rootfs**
```bash
cd ~/embedded-lab

# Download and extract BusyBox
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xjf busybox-1.36.1.tar.bz2 && cd busybox-1.36.1

# Configure
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
# Enable: Settings → Build static binary (no shared libs)

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- install

# Create rootfs directory structure
cd ..
mkdir -p rootfs/{bin,dev,etc,lib,proc,run,sys,tmp,usr/{bin,lib,sbin},var/log}

# Copy BusyBox install
cp -a busybox-1.36.1/_install/. rootfs/

# Create /etc/inittab
cat > rootfs/etc/inittab << 'EOF'
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyAMA0 115200 vt100
::shutdown:/bin/umount -a -r
EOF

# Create init script
mkdir -p rootfs/etc/init.d
cat > rootfs/etc/init.d/rcS << 'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
echo "Embedded Linux booted!"
EOF
chmod +x rootfs/etc/init.d/rcS

# Create initramfs
cd rootfs
find . | cpio -H newc -o 2>/dev/null | gzip > ../initramfs.cpio.gz
cd ..
```

**Day 11–14: Complete first QEMU boot**
```bash
# Boot: kernel + initramfs (no disk image yet)
qemu-system-aarch64 \
  -M virt,gic-version=3 \
  -cpu cortex-a76 \
  -smp 4 \
  -m 2G \
  -kernel linux/arch/arm64/boot/Image \
  -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init" \
  -nographic

# You should see the kernel boot log and end at:
# "Embedded Linux booted!"
# / #    (BusyBox shell)

# Test inside QEMU:
# / # uname -a
# / # cat /proc/cpuinfo
# / # cat /proc/meminfo
# / # ls /dev
```

**Goal:** Working embedded Linux system in QEMU by Day 14. **Document and push to GitHub.** Post on LinkedIn.

---

### Week 3 (Days 15–21): First Kernel Driver

**Write a character driver from scratch:**
```bash
# Create driver directory
mkdir -p ~/embedded-lab/drivers/hello_char
cd ~/embedded-lab/drivers/hello_char
```

```c
/* hello_char.c — Your first kernel driver from scratch */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/device.h>
#include <linux/slab.h>

#define DRIVER_NAME "hello_char"
#define BUFFER_SIZE 4096

static struct {
    struct cdev cdev;
    struct class *class;
    struct device *device;
    dev_t devno;
    char *buffer;
    size_t data_len;
} hello_dev;

static ssize_t hello_read(struct file *f, char __user *buf,
                           size_t count, loff_t *ppos)
{
    size_t to_copy = min(count, hello_dev.data_len - (size_t)*ppos);
    if (to_copy == 0) return 0;
    if (copy_to_user(buf, hello_dev.buffer + *ppos, to_copy))
        return -EFAULT;
    *ppos += to_copy;
    return to_copy;
}

static ssize_t hello_write(struct file *f, const char __user *buf,
                            size_t count, loff_t *ppos)
{
    size_t to_copy = min(count, (size_t)(BUFFER_SIZE - 1));
    if (copy_from_user(hello_dev.buffer, buf, to_copy))
        return -EFAULT;
    hello_dev.buffer[to_copy] = '\0';
    hello_dev.data_len = to_copy;
    pr_info("hello_char: received %zu bytes: %s\n", to_copy, hello_dev.buffer);
    return to_copy;
}

static const struct file_operations hello_fops = {
    .owner = THIS_MODULE,
    .read  = hello_read,
    .write = hello_write,
};

static int __init hello_init(void)
{
    alloc_chrdev_region(&hello_dev.devno, 0, 1, DRIVER_NAME);
    cdev_init(&hello_dev.cdev, &hello_fops);
    cdev_add(&hello_dev.cdev, hello_dev.devno, 1);
    hello_dev.class = class_create(DRIVER_NAME);
    hello_dev.device = device_create(hello_dev.class, NULL,
                                      hello_dev.devno, NULL, DRIVER_NAME);
    hello_dev.buffer = kzalloc(BUFFER_SIZE, GFP_KERNEL);
    pr_info("hello_char: loaded. Major=%d\n", MAJOR(hello_dev.devno));
    return 0;
}

static void __exit hello_exit(void)
{
    kfree(hello_dev.buffer);
    device_destroy(hello_dev.class, hello_dev.devno);
    class_destroy(hello_dev.class);
    cdev_del(&hello_dev.cdev);
    unregister_chrdev_region(hello_dev.devno, 1);
    pr_info("hello_char: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka");
MODULE_DESCRIPTION("First character driver from scratch");
```

```makefile
# Makefile
obj-m := hello_char.o
KDIR  := $(HOME)/embedded-lab/linux
CROSS := aarch64-linux-gnu-

all:
	make -C $(KDIR) M=$(PWD) ARCH=arm64 CROSS_COMPILE=$(CROSS) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# Build
make
# Load in QEMU:
# Copy hello_char.ko to rootfs → rebuild initramfs → reboot QEMU
# insmod hello_char.ko
# echo "Hello Kernel" > /dev/hello_char
# cat /dev/hello_char
# dmesg | tail
```

**Goal:** Working character driver loaded and tested in QEMU by Day 21.

---

### Week 4 (Days 22–30): Platform Driver + Device Tree

**Write a platform driver with DT binding (mirrors real driver work):**

```c
/* gpio_led_platform.c — Platform driver with DT binding */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/gpio/consumer.h>
#include <linux/sysfs.h>

struct led_priv {
    struct gpio_desc *gpio;
    struct device *dev;
};

static ssize_t led_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct led_priv *priv = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", gpiod_get_value(priv->gpio));
}

static ssize_t led_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
    struct led_priv *priv = dev_get_drvdata(dev);
    int val;
    if (kstrtoint(buf, 0, &val)) return -EINVAL;
    gpiod_set_value(priv->gpio, !!val);
    return count;
}

static DEVICE_ATTR(brightness, 0644, led_show, led_store);

static int led_probe(struct platform_device *pdev)
{
    struct led_priv *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv) return -ENOMEM;

    priv->gpio = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    if (IS_ERR(priv->gpio))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->gpio), "GPIO not found\n");

    priv->dev = &pdev->dev;
    platform_set_drvdata(pdev, priv);
    device_create_file(&pdev->dev, &dev_attr_brightness);

    dev_info(&pdev->dev, "LED driver probed, GPIO acquired\n");
    return 0;
}

static const struct of_device_id led_of_match[] = {
    { .compatible = "ravi,gpio-led" },
    {}
};
MODULE_DEVICE_TABLE(of, led_of_match);

static struct platform_driver led_driver = {
    .probe  = led_probe,
    .driver = {
        .name = "ravi-gpio-led",
        .of_match_table = led_of_match,
    },
};
module_platform_driver(led_driver);
MODULE_LICENSE("GPL");
```

**Device tree node to add to QEMU virt DT:**
```dts
/* Add to arch/arm64/boot/dts/qemu/qemu-virt.dts */
ravi_led: led@0 {
    compatible = "ravi,gpio-led";
    led-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
    status = "okay";
};
```

**Goal:** Platform driver with DT binding tested in QEMU by Day 30. **Push to GitHub with README.** This is your first portfolio project.

---

## Month 2 (Days 31–60): Kernel Depth + Open Source Prep

### Focus Areas

**Days 31–40: Kernel Subsystem Deep Dives**
- Study and implement: workqueue example, deferred work, kthread
- Study: RCU read side, RCU reclaim
- Study: KASAN and UBSAN — intentionally trigger, read the report
- Lab: Use ftrace to trace your own driver's execution
- Lab: Use GDB + KGDB to set breakpoints in your platform driver

**Days 41–50: Upstream Contribution Prep**
- Read `Documentation/process/submitting-patches.rst` (kernel tree)
- Set up git-send-email
- Find a kernel bug in a subsystem you know (drivers/nvmem/, drivers/mmc/, drivers/ufs/)
- Write a fix with proper commit message
- Submit to `linux-kernel@vger.kernel.org` — CC subsystem maintainer

**Days 51–60: AI Agent Building**
```python
# Build your first AI agent: Kernel Log Analyzer
# File: kernel_log_agent.py
# Uses: LangGraph + local Ollama (TinyLlama) or Claude API
# Input: dmesg output (paste or file)
# Output: structured analysis — error type, severity, suggested debug steps

# Start with: 29_QEMU_Embedded_AI_Labs/05_Embedded_AI_Projects/Project_04_LLM_Log_Analyzer/
```

**Month 2 Deliverable:** One upstream patch submitted + one working AI agent on GitHub.

---

## Month 3 (Days 61–90): Visibility + Portfolio

### Week 9–10 (Days 61–74): Portfolio Documentation

**Document your QEMU DW-UFS 4.0 project:**
- Write a README with architecture diagram (Mermaid)
- Add code snippets showing UFSHCI register model
- Add test results
- Record a 5-minute demo video

**Document your AI Pre-Silicon Validation Platform:**
- Architecture diagram
- Example input/output
- How it reduces engineering time

**Both projects → push to GitHub → add to LinkedIn Featured section.**

### Week 11–12 (Days 75–90): Radxa 5B+ First Labs

```bash
# Start Section 23_Radxa_5B_Plus_Labs/00_Board_Setup/
# Goal: Radxa 5B+ booted, SSH working, cross-compiled kernel module loaded
```

**Month 3 Deliverables:**
- 2 GitHub portfolio projects with README + diagram
- 1 LinkedIn post about your QEMU device modeling work
- Radxa 5B+ set up and first custom kernel module loaded
- Updated resume reflecting AI engineer capabilities

---

## 90-Day Success Metrics

By the end of day 90, you should have:

| Metric | Target |
|--------|--------|
| GitHub repositories | 3+ (BSP from scratch, char driver, AI agent) |
| Kernel drivers written | 3+ (character, platform, one subsystem driver) |
| QEMU boots completed | 10+ (different configurations) |
| Upstream patches submitted | 1+ (even if not merged yet — shows intention) |
| AI agent projects | 1 working (log analyzer or similar) |
| LinkedIn posts | 3+ (QEMU demo, driver demo, AI agent demo) |
| Interview questions practiced | 100+ |
| Resume updated | Yes — added AI engineer to title |

---

## Daily Minimum (Non-Negotiable)

Even on busy days:
- **2 hours** minimum: 1h lab/coding + 1h reading
- **5 interview questions** reviewed
- **1 git commit** pushed (even if tiny)
- **1 AI-assisted task** (use Copilot/Claude for something at work)

---

*Next: [05_1_Year_Roadmap.md](05_1_Year_Roadmap.md) → [11_KPI_Tracking.md](11_KPI_Tracking.md)*
