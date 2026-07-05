# 23 — Radxa 5B+ Labs

> 62 hands-on labs on RK3588 hardware. Every lab has been designed to build practical, demonstrable skills.

---

## Section Structure

```
23_Radxa_5B_Plus_Labs/
├── 00_Board_Setup/
│   ├── Lab_01_Flash_OS_Image.md          ← Get Ubuntu on eMMC
│   ├── Lab_02_Serial_Console_Setup.md    ← UART debug console
│   ├── Lab_03_SSH_And_Dev_Environment.md ← Headless development setup
│   └── Lab_04_JTAG_Setup.md             ← Debug probe connection
│
├── 01_BSP_Labs/
│   ├── Lab_05_Build_Kernel_From_Source.md
│   ├── Lab_06_Build_U_Boot.md
│   ├── Lab_07_Custom_Device_Tree.md
│   ├── Lab_08_Custom_Kernel_Module.md
│   └── Lab_09_Full_Yocto_BSP.md
│
├── 02_Driver_Labs/
│   ├── Lab_10_I2C_BMP280_Driver.md       ← Write I2C temperature/pressure driver
│   ├── Lab_11_SPI_OLED_Driver.md         ← Write SPI display driver
│   ├── Lab_12_GPIO_IRQ_Driver.md         ← GPIO interrupt driver
│   ├── Lab_13_PWM_LED_Driver.md          ← PWM brightness control
│   └── Lab_14_UART_Protocol_Driver.md    ← Custom UART protocol
│
├── 03_AI_Labs/
│   ├── Lab_15_Install_RKNN_Toolkit.md    ← NPU SDK setup
│   ├── Lab_16_First_NPU_Inference.md     ← MobileNetV2 on NPU
│   ├── Lab_17_YOLOv5_On_NPU.md          ← Object detection
│   ├── Lab_18_LLM_On_CPU.md             ← llama.cpp on A76
│   ├── Lab_19_LLM_On_NPU.md             ← LLM acceleration with RKNN
│   └── Lab_20_AI_Sensor_Pipeline.md     ← Camera → AI → action
│
├── 04_Debug_Labs/
│   ├── Lab_21_KGDB_Remote_Debug.md
│   ├── Lab_22_Ftrace_Driver_Analysis.md
│   ├── Lab_23_Perf_Flamegraph.md
│   └── Lab_24_Boot_Time_Optimization.md
│
└── 05_Project_Labs/
    ├── Lab_25_Complete_AI_Gateway.md     ← Full project: sensor+NPU+MQTT
    ├── Lab_26_Kernel_Contribution_Prep.md ← Prepare a real patch
    └── Lab_27_Portfolio_Documentation.md  ← Document your work
```

---

## Lab 01: Flash OS Image

```bash
# What you need:
# - Radxa Rock 5B+ board
# - USB-A to USB-A cable (for rkdeveloptool)
# - SD card (≥ 16GB) or eMMC module

# Step 1: Download official image
wget https://github.com/radxa/debos-radxa/releases/latest/download/rock-5b-ubuntu-jammy-server-arm64.img.xz

# Verify (always verify downloaded firmware)
sha256sum rock-5b-ubuntu-jammy-server-arm64.img.xz
# Compare with published SHA256

# Step 2: Flash to SD card (from Linux host)
xzcat rock-5b-ubuntu-jammy-server-arm64.img.xz | \
  sudo dd of=/dev/sdX bs=4M status=progress oflag=sync
# Replace /dev/sdX with your SD card device

# Step 3: First boot
# Insert SD card → power on
# Default login: rock / rock (or check image docs)
# Or via serial console (see Lab 02)

# Step 4: Expand rootfs (if needed)
sudo resize2fs /dev/mmcblk1p5
```

---

## Lab 05: Build Linux Kernel from Source for Radxa 5B+

```bash
# On Ubuntu 22.04 host machine
# Estimated time: 30-45 minutes (first build)

# 1. Get Radxa's kernel tree (has RK3588 patches)
git clone https://github.com/radxa/kernel --depth=1 \
  --branch linux-5.10-gen-rkr3.4 radxa-kernel
cd radxa-kernel

# 2. Configure
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# Use Radxa defconfig (RK3588 optimized)
make rockchip_linux_defconfig

# Add your custom options
make menuconfig
# Example: add KASAN for debugging
# Kernel hacking → Memory Debugging → KASAN

# 3. Build
make Image -j$(nproc)
make dtbs         # build all device trees

# 4. Copy to board (via SCP)
scp arch/arm64/boot/Image rock@192.168.1.x:/boot/
scp arch/arm64/boot/dts/rockchip/rk3588-rock-5b.dtb rock@192.168.1.x:/boot/
ssh rock@192.168.1.x "sudo reboot"

# 5. Verify on board
uname -r   # should show new version

# Kernel module build (separate step)
make modules -j$(nproc)
make INSTALL_MOD_PATH=./staging modules_install
# Copy staging/ to board
```

---

## Lab 10: Write I2C Driver for BMP280 (Temperature/Pressure Sensor)

```bash
# Hardware setup:
# BMP280 module → Radxa 5B+ I2C bus 7
# Connections:
#   BMP280 VCC → 3.3V
#   BMP280 GND → GND
#   BMP280 SDA → PIN3 (I2C7_SDA_M3)
#   BMP280 SCL → PIN5 (I2C7_SCL_M3)

# DT node to add to rk3588-rock-5b.dts:
```

```dts
&i2c7 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&i2c7m3_xfer>;
    
    bmp280: bmp280@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;         /* I2C address */
        status = "okay";
    };
};
```

```c
/* bmp280_driver.c — Write a real I2C driver (educational version) */
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/iio/iio.h>
#include <linux/iio/sysfs.h>
#include <linux/delay.h>

#define BMP280_CHIP_ID_REG    0xD0
#define BMP280_CHIP_ID        0x60  /* BMP280 chip ID */
#define BMP280_CTRL_MEAS      0xF4
#define BMP280_TEMP_MSB       0xFA

struct bmp280_priv {
    struct i2c_client *client;
    struct iio_dev *iio_dev;
};

static int bmp280_read_byte(struct i2c_client *client, u8 reg)
{
    return i2c_smbus_read_byte_data(client, reg);
}

static int bmp280_get_temp_raw(struct i2c_client *client, s32 *temp_raw)
{
    u8 buf[3];
    int ret;
    
    ret = i2c_smbus_read_i2c_block_data(client, BMP280_TEMP_MSB, 3, buf);
    if (ret < 0) return ret;
    
    *temp_raw = (buf[0] << 12) | (buf[1] << 4) | (buf[2] >> 4);
    return 0;
}

static int bmp280_read_raw(struct iio_dev *iio_dev,
                            struct iio_chan_spec const *chan,
                            int *val, int *val2, long mask)
{
    struct bmp280_priv *priv = iio_priv(iio_dev);
    s32 raw;
    int ret;

    if (mask != IIO_CHAN_INFO_PROCESSED) return -EINVAL;

    ret = bmp280_get_temp_raw(priv->client, &raw);
    if (ret) return ret;

    /* Very simplified temp compensation — real driver uses full Bosch formula */
    *val = raw / 100;   /* rough: divide ADC count */
    *val2 = 0;
    return IIO_VAL_INT;
}

static const struct iio_chan_spec bmp280_channels[] = {
    {
        .type = IIO_TEMP,
        .info_mask_separate = BIT(IIO_CHAN_INFO_PROCESSED),
    },
};

static const struct iio_info bmp280_iio_info = {
    .read_raw = bmp280_read_raw,
};

static int bmp280_probe(struct i2c_client *client)
{
    struct bmp280_priv *priv;
    struct iio_dev *iio_dev;
    int chip_id, ret;

    /* Check device presence */
    chip_id = bmp280_read_byte(client, BMP280_CHIP_ID_REG);
    if (chip_id < 0)
        return dev_err_probe(&client->dev, chip_id, "Failed to read chip ID\n");
    if (chip_id != BMP280_CHIP_ID)
        return dev_err_probe(&client->dev, -ENODEV,
                              "Wrong chip ID: 0x%02x\n", chip_id);

    /* Allocate IIO device with embedded priv */
    iio_dev = devm_iio_device_alloc(&client->dev, sizeof(*priv));
    if (!iio_dev) return -ENOMEM;

    priv = iio_priv(iio_dev);
    priv->client = client;
    priv->iio_dev = iio_dev;
    i2c_set_clientdata(client, priv);

    /* Configure IIO device */
    iio_dev->name = "bmp280";
    iio_dev->info = &bmp280_iio_info;
    iio_dev->channels = bmp280_channels;
    iio_dev->num_channels = ARRAY_SIZE(bmp280_channels);

    /* Configure hardware: forced mode, oversampling 1× */
    ret = i2c_smbus_write_byte_data(client, BMP280_CTRL_MEAS, 0x27);
    if (ret) return dev_err_probe(&client->dev, ret, "Failed to configure\n");

    ret = devm_iio_device_register(&client->dev, iio_dev);
    if (ret) return dev_err_probe(&client->dev, ret, "Failed to register IIO\n");

    dev_info(&client->dev, "BMP280 registered, chip ID=0x%02x\n", chip_id);
    return 0;
}

static const struct of_device_id bmp280_of_match[] = {
    { .compatible = "bosch,bmp280" },
    { }
};
MODULE_DEVICE_TABLE(of, bmp280_of_match);

static const struct i2c_device_id bmp280_id[] = {
    { "bmp280", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, bmp280_id);

static struct i2c_driver bmp280_driver = {
    .driver = {
        .name = "bmp280",
        .of_match_table = bmp280_of_match,
    },
    .probe = bmp280_probe,
    .id_table = bmp280_id,
};
module_i2c_driver(bmp280_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka");
MODULE_DESCRIPTION("BMP280 I2C driver (educational)");
```

```bash
# Test after loading:
insmod bmp280_driver.ko
ls /sys/bus/iio/devices/
# iio:device0

cat /sys/bus/iio/devices/iio:device0/in_temp_input
# 2350  (= 23.50°C)

# Real BMP280 driver in kernel: drivers/iio/pressure/bmp280-i2c.c
# Compare your code with the upstream driver!
```

---

## Lab 16: First NPU Inference on RK3588

```bash
# On Radxa 5B+ running Ubuntu

# Step 1: Install RKNN Toolkit Lite (runs ON the board, not host)
pip3 install rknn-toolkit-lite2

# Step 2: Get a pre-converted RKNN model
wget https://github.com/airockchip/rknn_model_zoo/raw/main/models/CV/image_classification/mobilenet/mobilenetv2.rknn

# Step 3: Run inference
python3 << 'EOF'
from rknnlite.api import RKNNLite
import numpy as np
from PIL import Image

# Initialize RKNN runtime
rknn = RKNNLite()
rknn.load_rknn('./mobilenetv2.rknn')
rknn.init_runtime()

# Load and preprocess test image
img = Image.open('test.jpg').resize((224, 224))
img_np = np.array(img).reshape([1, 224, 224, 3])

# Run inference on NPU
outputs = rknn.inference(inputs=[img_np])

# Get top prediction
top1_idx = np.argmax(outputs[0])
print(f"Top prediction: class {top1_idx}, score {outputs[0][0][top1_idx]:.2f}")

# Check NPU utilization
import subprocess
npu_util = subprocess.run(['cat', '/sys/kernel/debug/rknpu/load'], capture_output=True, text=True)
print(f"NPU utilization: {npu_util.stdout}")

rknn.release()
EOF

# Monitor NPU usage:
watch -n 0.5 'cat /sys/kernel/debug/rknpu/load'
# or:
npu-smi info  # if npu-smi is installed
```

---

## RK3588 Hardware Quick Reference

```
Radxa Rock 5B+ — Key Hardware:
─────────────────────────────────────────────────────────
SoC:       RK3588
           4× Cortex-A76 @ 2.4GHz + 4× Cortex-A55 @ 1.8GHz
           Mali-G610 MP4 GPU
           NPU 3.0: 6 TOPS (INT8), 3 TOPS (INT4)

RAM:       LPDDR4X, 4GB/8GB/16GB
Storage:   eMMC 5.1 onboard + M.2 M-Key PCIe 3.0 x4 (NVMe)
           SD card slot, SPI NOR flash

USB:       USB3.1 Gen1 Type-A ×2, USB3.1 Gen1 Type-C ×1 (OTG)
           USB2.0 Type-A ×2

Display:   HDMI 2.1 (8K@60Hz), DP 1.4 (via USB-C)
           MIPI DSI ×2 (via FPC connector)

Camera:    MIPI CSI ×2 (4-lane + 2-lane)

Network:   2.5G Ethernet ×1 (via PCIe)
           M.2 E-Key for WiFi/BT

Debug:     3-pin UART (1.5Mbaud) on GPIO header
           USB OTG for rkdeveloptool/Maskrom

GPIO:      40-pin GPIO header
           I2C ×5, SPI ×2, UART ×5, PWM ×4, ADC ×2

Power:     USB-C PD (12V/2A or more) or DC barrel (5.5/2.5mm)
```

---

*Related: [04_Embedded_Linux/](../04_Embedded_Linux/) | [07_Device_Drivers/](../07_Device_Drivers/) | [29_QEMU_Embedded_AI_Labs/](../29_QEMU_Embedded_AI_Labs/)*
