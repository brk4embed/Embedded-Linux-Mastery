# Advanced Kernel Frameworks — V4L2, DRM, ALSA, Power Management, Security

> These are the "upper deck" frameworks — built on top of the GPIO/Clock/Regulator foundation.  
> V4L2 for cameras, DRM for displays, ALSA for audio, PM for power, Security for TrustZone.

---

## Part 1: V4L2 — Video for Linux 2 (Camera Framework)

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    V4L2 Software Stack                           │
│                                                                  │
│  User Space App (GStreamer, OpenCV, ffmpeg, custom)              │
│       │ ioctl(fd, VIDIOC_STREAMON, ...) on /dev/video0           │
│       ▼                                                          │
│  V4L2 Core (drivers/media/v4l2-core/)                           │
│       │ video_ioctl2() → dispatches to driver ops               │
│       ▼                                                          │
│  Media Controller (drivers/media/mc/)                           │
│       │ Manages pipeline: sensor → ISP → CSI2 → DMA              │
│       ▼                                                          │
│  Camera Sensor Driver (drivers/media/i2c/ov5647.c etc.)         │
│       │ I2C writes to configure resolution/fps/format             │
│       ▼                                                          │
│  CSI2 Receiver / ISP Driver (SoC-specific)                      │
│       │ receives MIPI CSI2 data, processes it                    │
│       ▼                                                          │
│  DMA Buffer Management (VB2 — Video Buffer 2)                   │
│       │ allocates DMA buffers, fills them, notifies user         │
└──────────────────────────────────────────────────────────────────┘
```

### V4L2 Core Concepts

```c
/* Key data structures in V4L2 */

/* video_device: represents /dev/videoX */
struct video_device {
    char name[32];
    struct v4l2_file_operations *fops;
    const struct v4l2_ioctl_ops *ioctl_ops;
    /* ... */
};

/* v4l2_subdev: represents a sub-component (sensor, ISP block) */
struct v4l2_subdev {
    /* connected via media_entity links */
    const struct v4l2_subdev_ops *ops;
    struct v4l2_subdev_video_ops {
        int (*s_stream)(struct v4l2_subdev *sd, int enable);
    };
    struct v4l2_subdev_pad_ops {
        int (*get_fmt)(struct v4l2_subdev *sd, ...);
        int (*set_fmt)(struct v4l2_subdev *sd, ...);
        int (*enum_mbus_code)(struct v4l2_subdev *sd, ...);
    };
};

/* vb2_queue: manages video frame buffers */
struct vb2_queue {
    const struct vb2_ops *ops;     /* buf_prepare, buf_queue, start_streaming */
    const struct vb2_mem_ops *mem_ops;  /* dma_contig, dma_sg, vmalloc */
    u32 io_modes;    /* VB2_MMAP | VB2_DMABUF | VB2_USERPTR */
    /* ... */
};
```

### Writing a Minimal V4L2 Camera Sensor Driver

```c
/* drivers/media/i2c/my_sensor.c */
#include <linux/i2c.h>
#include <media/v4l2-subdev.h>
#include <media/v4l2-mediabus.h>

struct my_sensor {
    struct v4l2_subdev     sd;
    struct media_pad       pad;
    struct i2c_client      *client;
    struct v4l2_mbus_framefmt fmt;
    bool streaming;
};

/* Helper to write sensor register via I2C */
static int reg_write(struct i2c_client *client, u16 reg, u8 val)
{
    u8 buf[3] = { reg >> 8, reg & 0xFF, val };
    return i2c_master_send(client, buf, 3) == 3 ? 0 : -EIO;
}

/* Sensor initialization table */
static const struct {u16 reg; u8 val;} sensor_init_regs[] = {
    { 0x3103, 0x11 },   /* system clock from PLL */
    { 0x3008, 0x82 },   /* software reset */
    { 0x3017, 0xFF },   /* FREX, VSYNC, HREF, PCLK, D[9:6] output */
    /* ... hundreds more sensor-specific regs */
    { 0xFFFF, 0xFF },   /* sentinel */
};

/* pad_ops: negotiate format with pipeline */
static int my_sensor_get_fmt(struct v4l2_subdev *sd,
                              struct v4l2_subdev_state *sd_state,
                              struct v4l2_subdev_format *format)
{
    struct my_sensor *sensor = container_of(sd, struct my_sensor, sd);
    format->format = sensor->fmt;
    return 0;
}

static int my_sensor_set_fmt(struct v4l2_subdev *sd,
                              struct v4l2_subdev_state *sd_state,
                              struct v4l2_subdev_format *format)
{
    struct my_sensor *sensor = container_of(sd, struct my_sensor, sd);
    /* Accept any format, but clamp to what we support */
    format->format.width = 1920;
    format->format.height = 1080;
    format->format.code = MEDIA_BUS_FMT_SRGGB10_1X10;   /* RAW10 Bayer */
    format->format.field = V4L2_FIELD_NONE;
    
    if (format->which == V4L2_SUBDEV_FORMAT_ACTIVE)
        sensor->fmt = format->format;
    return 0;
}

/* video_ops: start/stop streaming */
static int my_sensor_s_stream(struct v4l2_subdev *sd, int enable)
{
    struct my_sensor *sensor = container_of(sd, struct my_sensor, sd);
    int ret;

    if (enable) {
        /* Write all init registers to sensor */
        for (int i = 0; sensor_init_regs[i].reg != 0xFFFF; i++) {
            ret = reg_write(sensor->client, sensor_init_regs[i].reg,
                           sensor_init_regs[i].val);
            if (ret) return ret;
        }
        sensor->streaming = true;
    } else {
        /* Standby */
        reg_write(sensor->client, 0x3008, 0x42);  /* software standby */
        sensor->streaming = false;
    }
    return 0;
}

static const struct v4l2_subdev_video_ops my_sensor_video_ops = {
    .s_stream = my_sensor_s_stream,
};

static const struct v4l2_subdev_pad_ops my_sensor_pad_ops = {
    .get_fmt = my_sensor_get_fmt,
    .set_fmt = my_sensor_set_fmt,
};

static const struct v4l2_subdev_ops my_sensor_ops = {
    .video = &my_sensor_video_ops,
    .pad   = &my_sensor_pad_ops,
};

static int my_sensor_probe(struct i2c_client *client)
{
    struct my_sensor *sensor;
    int ret;

    sensor = devm_kzalloc(&client->dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor) return -ENOMEM;
    sensor->client = client;

    /* Initialize v4l2_subdev */
    v4l2_i2c_subdev_init(&sensor->sd, client, &my_sensor_ops);
    sensor->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;

    /* Initialize media pad */
    sensor->pad.flags = MEDIA_PAD_FL_SOURCE;
    ret = media_entity_pads_init(&sensor->sd.entity, 1, &sensor->pad);
    if (ret) return ret;

    /* Set default format */
    sensor->fmt.width  = 1920;
    sensor->fmt.height = 1080;
    sensor->fmt.code   = MEDIA_BUS_FMT_SRGGB10_1X10;

    /* Register subdev */
    ret = v4l2_async_register_subdev(&sensor->sd);
    if (ret) {
        media_entity_cleanup(&sensor->sd.entity);
        return ret;
    }

    dev_info(&client->dev, "sensor probe OK: 1920x1080 RAW10\n");
    return 0;
}
```

### Testing V4L2 Camera

```bash
# List video devices
v4l2-ctl --list-devices

# Check capabilities
v4l2-ctl -d /dev/video0 --info

# List supported formats
v4l2-ctl -d /dev/video0 --list-formats-ext

# Capture single frame
v4l2-ctl -d /dev/video0 \
    --set-fmt-video=width=1920,height=1080,pixelformat=RG10 \
    --stream-mmap=1 \
    --stream-to=frame.raw

# Live preview with GStreamer
gst-launch-1.0 v4l2src device=/dev/video0 ! \
    video/x-raw,format=RGGB10,width=1920,height=1080 ! \
    videoconvert ! autovideosink

# Use media-ctl to configure ISP pipeline
media-ctl -p -d /dev/media0   # print pipeline
media-ctl -d /dev/media0 \
    --set-v4l2 '"my-sensor 3-003c":0[fmt:SRGGB10_1X10/1920x1080]'
```

---

## Part 2: DRM/KMS — Display Framework

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Application (Wayland/X11/DRM direct)                            │
│  wl_compositor / DRM GEM buffer                                  │
│       ▼                                                          │
│  DRM Core (drivers/gpu/drm/drm_*.c)                             │
│  DRM GEM (Graphics Execution Manager — buffer management)        │
│  DRM KMS (Kernel Mode Setting — display configuration)          │
│       ▼                                                          │
│  KMS Objects:                                                    │
│    CRTC: timing generator (one per display pipeline)            │
│    Encoder: format conversion (CRTC → HDMI/MIPI/LVDS)          │
│    Connector: physical port (HDMI port, DSI connector)          │
│    Plane: overlay layer (cursor, primary, overlay)              │
│       ▼                                                          │
│  Display Driver (drm-rockchip, drm-msm, drm-imx, etc.)         │
│       ▼                                                          │
│  Display Hardware (VOP2 on RK3588, etc.)                        │
└──────────────────────────────────────────────────────────────────┘
```

### Display Pipeline on RK3588

```
Framebuffer (DMA buffer)
    │
    ▼
VOP2 (Versatile Output Processor 2)   ← CRTC driver
    │ scanout, overlay mixing, color conversion
    ▼
HDMI TX / MIPI DSI / DP               ← Encoder driver
    │
    ▼
HDMI Connector / LCD Panel            ← Connector driver
    │
    ▼
Display (TV/monitor/LCD)
```

### Simple DRM Driver Skeleton

```c
/* drivers/gpu/drm/myvendor/myvendor_drm.c */
#include <drm/drm_drv.h>
#include <drm/drm_fb_helper.h>
#include <drm/drm_gem_cma_helper.h>
#include <drm/drm_plane_helper.h>

DEFINE_DRM_GEM_CMA_FOPS(myvendor_drm_fops);

static struct drm_driver myvendor_drm_driver = {
    .driver_features = DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC,
    .fops            = &myvendor_drm_fops,
    .name            = "myvendor-drm",
    .desc            = "My Vendor Display Driver",
    .date            = "20240101",
    .major = 1, .minor = 0,
    DRM_GEM_CMA_DRIVER_OPS,
};

static int myvendor_drm_probe(struct platform_device *pdev)
{
    struct drm_device *drm;
    int ret;

    drm = devm_drm_dev_alloc(&pdev->dev, &myvendor_drm_driver,
                              struct myvendor_drm, drm);
    if (IS_ERR(drm)) return PTR_ERR(drm);

    /* Initialize display hardware */
    ret = myvendor_setup_crtc(drm);   /* timing generator */
    ret = myvendor_setup_encoder(drm); /* signal encoder */
    ret = myvendor_setup_connector(drm); /* output connector */

    ret = drm_dev_register(drm, 0);
    if (ret) return ret;

    drm_fbdev_generic_setup(drm, 32);   /* fallback fbdev for console */
    return 0;
}
```

### Testing DRM

```bash
# Check DRM devices
ls /dev/dri/
# card0  card1  renderD128  renderD129

# modetest — show display capabilities
modetest -M rockchip   # or just: modetest

# Set display mode
modetest -M rockchip -s 89:1920x1080@RG16

# Use kmscube for 3D rendering test
kmscube

# Direct DRM buffer write (simple framebuffer)
cat /dev/urandom | head -c $((1920*1080*4)) > /dev/fb0
```

---

## Part 3: ALSA / ASoC — Audio Framework

### ASoC (ALSA System on Chip) Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  User Space: aplay, arecord, PulseAudio, PipeWire                │
│       ▼ /dev/snd/pcmC0D0p (playback) /dev/snd/pcmC0D0c (record) │
│  ALSA Core (sound/core/)                                         │
│       ▼                                                          │
│  ASoC Machine Driver  ← YOUR BOARD-LEVEL DRIVER                  │
│  Connects: codec ↔ I2S ↔ CPU                                     │
│       │                                                          │
│       ├── Codec Driver (sound/soc/codecs/es8316.c)               │
│       │   Controls: DAC/ADC, mixers, EQ, volume                  │
│       │   Via I2C to audio codec chip                            │
│       │                                                          │
│       └── Platform/CPU Driver (sound/soc/rockchip/rockchip_i2s.c)│
│           Controls: I2S controller (the digital audio bus)      │
│           DMA: transfers PCM data CPU↔I2S FIFO                  │
└──────────────────────────────────────────────────────────────────┘
```

### ASoC Machine Driver

```c
/* sound/soc/rockchip/rk3588-es8316.c — Machine driver */
#include <sound/soc.h>
#include <sound/jack.h>

static struct snd_soc_dai_link rk3588_dai_links[] = {
    {
        .name           = "ES8316",
        .stream_name    = "ES8316 PCM",
        /* CPU side: I2S controller */
        .cpu_dai_name   = "rockchip-i2s.2",
        .platform_name  = "rockchip-i2s.2",
        /* Codec side: ES8316 audio chip */
        .codec_dai_name = "ES8316 HiFi",
        .codec_name     = "es8316.2-0011",  /* I2C bus 2, addr 0x11 */
        /* Configuration */
        .dai_fmt = SND_SOC_DAIFMT_I2S |
                   SND_SOC_DAIFMT_NB_NF |      /* normal bit/frame clock */
                   SND_SOC_DAIFMT_CBS_CFS,     /* codec is slave */
        .init = rk3588_es8316_init,
    },
};

static struct snd_soc_card rk3588_soundcard = {
    .name           = "rk3588-es8316",
    .owner          = THIS_MODULE,
    .dai_link       = rk3588_dai_links,
    .num_links      = ARRAY_SIZE(rk3588_dai_links),
    .controls       = rk3588_controls,
    .num_controls   = ARRAY_SIZE(rk3588_controls),
    .dapm_widgets   = rk3588_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(rk3588_dapm_widgets),
};

static int rk3588_es8316_probe(struct platform_device *pdev)
{
    struct snd_soc_card *card = &rk3588_soundcard;
    card->dev = &pdev->dev;
    return devm_snd_soc_register_card(&pdev->dev, card);
}
```

### Testing Audio

```bash
# List audio devices
aplay -l
# **** List of PLAYBACK Hardware Devices ****
# card 0: rk3588es8316 [rk3588-es8316], device 0: ES8316 HiFi es8316-hifi-0

# Play audio
aplay -D plughw:0,0 test.wav

# Record
arecord -D plughw:0,0 -f S16_LE -r 44100 -c 2 output.wav

# Check ALSA mixer controls
alsamixer -c 0

# Test with speaker-test
speaker-test -c 2 -t wav
```

---

## Part 4: Power Management

### Linux PM States

```
System Power States:
  S0: Running (normal)
  S1: Standby (not commonly used on embedded)
  S2: Not used
  S3: Suspend to RAM (STR) — contents in RAM, CPU off
  S4: Suspend to Disk (STD/Hibernate) — contents saved to disk
  S5: Powered off

Runtime PM (per-device):
  RPM_ACTIVE: device is active
  RPM_SUSPENDED: device is suspended (may be powered off)
  RPM_RESUMING: transition to active
  RPM_SUSPENDING: transition to suspended
```

### Implementing Suspend/Resume in Your Driver

```c
#include <linux/pm.h>
#include <linux/pm_runtime.h>

/* System suspend/resume (S3) */
static int my_driver_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    
    /* 1. Stop DMA */
    dmaengine_terminate_sync(priv->dma_chan);
    
    /* 2. Disable IRQ */
    disable_irq(priv->irq);
    
    /* 3. Save hardware state if needed */
    priv->saved_ctrl = readl(priv->base + REG_CTRL);
    priv->saved_cfg  = readl(priv->base + REG_CONFIG);
    
    /* 4. Disable hardware */
    writel(0, priv->base + REG_CTRL);
    
    /* 5. Disable clocks (frees power) */
    clk_disable_unprepare(priv->clk);
    
    /* 6. Disable regulator */
    regulator_disable(priv->vdd);
    
    dev_dbg(dev, "suspended\n");
    return 0;
}

static int my_driver_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    int ret;
    
    /* Reverse of suspend */
    ret = regulator_enable(priv->vdd);
    if (ret) return ret;
    usleep_range(1000, 2000);
    
    ret = clk_prepare_enable(priv->clk);
    if (ret) { regulator_disable(priv->vdd); return ret; }
    
    /* Restore hardware state */
    writel(priv->saved_cfg,  priv->base + REG_CONFIG);
    writel(priv->saved_ctrl, priv->base + REG_CTRL);
    
    enable_irq(priv->irq);
    
    dev_dbg(dev, "resumed\n");
    return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(my_driver_pm_ops,
                                 my_driver_suspend, my_driver_resume);

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-driver",
        .pm   = pm_sleep_ptr(&my_driver_pm_ops),  /* NULL if no CONFIG_PM_SLEEP */
    },
    .probe  = my_probe,
    .remove = my_remove,
};
```

### Runtime PM — Per-Device Auto Power Management

```c
/* Runtime PM: device powers off when idle, wakes on demand */

static int my_probe(struct platform_device *pdev)
{
    /* ... normal probe ... */

    /* Enable runtime PM */
    pm_runtime_set_active(&pdev->dev);   /* start in active state */
    pm_runtime_enable(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, 100);  /* 100ms idle → suspend */
    pm_runtime_use_autosuspend(&pdev->dev);
    
    return 0;
}

/* Runtime suspend callback */
static int my_runtime_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    clk_disable_unprepare(priv->clk);
    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    return clk_prepare_enable(priv->clk);
}

static const struct dev_pm_ops my_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(my_driver_suspend, my_driver_resume)
    SET_RUNTIME_PM_OPS(my_runtime_suspend, my_runtime_resume, NULL)
};

/* In your driver's read/write function: */
static ssize_t my_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    struct my_priv *priv = file->private_data;
    int ret;
    
    /* Wake device (powers on if suspended) */
    ret = pm_runtime_get_sync(priv->dev);
    if (ret < 0) return ret;
    
    /* Do work */
    ret = do_read_data(priv, buf, count);
    
    /* Allow device to auto-suspend after idle delay */
    pm_runtime_mark_last_busy(priv->dev);
    pm_runtime_put_autosuspend(priv->dev);
    
    return ret;
}
```

### Testing Power Management

```bash
# System suspend
echo mem > /sys/power/state    # S3 suspend to RAM
# Board suspends. Press power button to wake.

# Check runtime PM status
cat /sys/bus/platform/drivers/my-driver/*/power/runtime_status
# suspended

# Force resume
cat /sys/bus/platform/drivers/my-driver/*/my_reg_attr  # read triggers resume

# Runtime PM statistics
cat /sys/bus/platform/devices/*/power/runtime_suspended_time
# 45234      (milliseconds suspended)
```

---

## Part 5: Security Frameworks

### TrustZone / OP-TEE in Linux

```
ARM64 Exception Levels:
  EL0: User space applications
  EL1: Linux kernel (Normal World)
  EL2: Hypervisor (optional, Xen/KVM)
  EL3: Secure Monitor (ARM Trusted Firmware)

TrustZone splits hardware into two worlds:
  Normal World: Linux kernel + applications
  Secure World: OP-TEE OS + Trusted Applications (TAs)

SMC (Secure Monitor Call): crossing worlds
  Normal World → EL3 (ATF) → Secure World (OP-TEE)
```

### TEE Client API (from Normal World)

```c
/* How Linux applications communicate with OP-TEE */
#include <tee_client_api.h>

/* TA UUID: unique identifier for your Trusted Application */
#define MY_TA_UUID { 0x12345678, 0x1234, 0x1234, \
                     { 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc, 0xde, 0xf0 } }

void use_tee(void)
{
    TEEC_Context ctx;
    TEEC_Session session;
    TEEC_Operation op = {0};
    uint32_t err_origin;

    /* 1. Initialize TEE context */
    TEEC_InitializeContext(NULL, &ctx);

    /* 2. Open session with your TA */
    TEEC_UUID uuid = MY_TA_UUID;
    TEEC_OpenSession(&ctx, &session, &uuid,
                     TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);

    /* 3. Invoke TA command */
    op.paramTypes = TEEC_PARAM_TYPES(
        TEEC_MEMREF_TEMP_INPUT,    /* param[0]: input buffer */
        TEEC_MEMREF_TEMP_OUTPUT,   /* param[1]: output buffer */
        TEEC_NONE, TEEC_NONE
    );
    
    char input[64] = "Encrypt this data";
    char output[64] = {0};
    
    op.params[0].tmpref.buffer = input;
    op.params[0].tmpref.size   = strlen(input);
    op.params[1].tmpref.buffer = output;
    op.params[1].tmpref.size   = sizeof(output);

    TEEC_InvokeCommand(&session, 0x00000001 /* CMD_ENCRYPT */, 
                       &op, &err_origin);

    printf("Encrypted: ");
    for (int i = 0; i < op.params[1].tmpref.size; i++)
        printf("%02x", (uint8_t)output[i]);
    printf("\n");

    /* 4. Cleanup */
    TEEC_CloseSession(&session);
    TEEC_FinalizeContext(&ctx);
}
```

### Trusted Application (runs in Secure World)

```c
/* my_ta/my_ta.c — Trusted Application */
#include <tee_internal_api.h>

/* Called when first session opens */
TEE_Result TA_CreateEntryPoint(void) { return TEE_SUCCESS; }
void TA_DestroyEntryPoint(void) {}
TEE_Result TA_OpenSessionEntryPoint(...) { return TEE_SUCCESS; }
void TA_CloseSessionEntryPoint(...) {}

/* Handle commands from Normal World */
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx, uint32_t cmd_id,
                                       uint32_t param_types,
                                       TEE_Param params[4])
{
    switch (cmd_id) {
    case 0x00000001:  /* CMD_ENCRYPT */
        {
            /* Access Normal World buffer (safely, within bounds) */
            uint8_t *input  = params[0].memref.buffer;
            size_t   in_len = params[0].memref.size;
            uint8_t *output = params[1].memref.buffer;

            /* Use OP-TEE crypto APIs (run in secure world!) */
            TEE_OperationHandle op;
            TEE_AllocateOperation(&op, TEE_ALG_AES_CBC_NOPAD,
                                  TEE_MODE_ENCRYPT, 128);
            /* ... AES encrypt ... */
            TEE_FreeOperation(op);
        }
        return TEE_SUCCESS;
    default:
        return TEE_ERROR_NOT_IMPLEMENTED;
    }
}
```

### Secure Boot with U-Boot and Linux

```bash
# Full secure boot chain:
# 1. BootROM verifies SPL signature (using OTP key hash)
# 2. SPL verifies U-Boot signature
# 3. U-Boot verifies FIT image (kernel + DTB) signature
# 4. Linux verifies kernel module signatures (if CONFIG_MODULE_SIG_FORCE=y)

# Generate RSA keys for signing
mkdir keys && cd keys
openssl genrsa -F4 -out dev.key 2048
openssl req -new -x509 -days 3650 -key dev.key \
    -out dev.crt -subj "/CN=Dev Sign Key/"
# Extract public key for embedding in U-Boot DTS
openssl rsa -in dev.key -pubout -out dev.pub.pem

# Sign FIT image
mkimage -F -k keys/ -K u-boot.dtb -r boot.itb

# Verify (same as what U-Boot does at boot)
mkimage -F -k keys/ boot.itb  # should say "Verified OK"

# Kernel module signing
scripts/sign-file sha256 signing_key.pem signing_key.x509 my_driver.ko

# Verify module signature
modinfo my_driver.ko | grep sig
```

---

## Part 6: Storage Frameworks (eMMC / UFS / NVMe)

### Block Layer Architecture

```
Application: open/read/write file
    │
    ▼
VFS (Virtual Filesystem)
    │
    ▼
Block Layer (bio → request queue → blk-mq)
    │
    ▼
Storage Driver:
  eMMC: drivers/mmc/host/sdhci.c
  UFS:  drivers/ufs/core/ufshcd.c  ← Ravi's expertise!
  NVMe: drivers/nvme/host/pci.c
    │
    ▼
Physical Storage Device
```

### UFS Deep Dive (Ravi's Background)

```c
/* UFS (Universal Flash Storage) protocol stack */

/*
 * UFS Transaction Layers:
 *   UPIU: UFS Protocol Information Unit (the packet)
 *   UTP: UFS Transfer Protocol (sends UPIUs)
 *   UFSHCI: UFS Host Controller Interface (hardware register spec)
 *
 * Command flow for a READ 10 SCSI command:
 *   1. Build SCSI CDB (Command Descriptor Block)
 *   2. Wrap in Command UPIU
 *   3. Build UTRD (UTP Transfer Request Descriptor)
 *   4. Put UTRD in Transfer Request List (OCS slot N)
 *   5. Ring doorbell register (bit N)
 *   6. UFS HC sends UPIU to device
 *   7. Device processes, sends Response UPIU + Data-In UPIU
 *   8. UFS HC triggers interrupt
 *   9. Driver reads response, calls scsi_done()
 */

/* UFSHCI register write (from drivers/ufs/core/ufshcd.c) */
static inline void ufshcd_writel(struct ufs_hba *hba, u32 val, u32 reg)
{
    writel(val, hba->mmio_base + reg);
}

/* Your DW-UFS 4.0 QEMU model implements these register behaviors */
#define REG_CONTROLLER_ENABLE       0x34
#define REG_INTERRUPT_STATUS        0x20
#define REG_UTP_TRANSFER_REQ_DOOR_BELL  0x58

void ufshcd_enable_controller(struct ufs_hba *hba)
{
    ufshcd_writel(hba, CONTROLLER_ENABLE, REG_CONTROLLER_ENABLE);
}
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Beginner** | What is V4L2? What devices use it? |
| **Beginner** | What is DRM/KMS? How does it differ from old fbdev? |
| **Intermediate** | Explain the ASoC machine driver. What 3 components does it connect? |
| **Intermediate** | What is the difference between system suspend and runtime PM? |
| **Advanced** | How does OP-TEE communicate with the Linux kernel? What is the SMC call? |
| **Advanced** | Explain UFS UPIU. What types of UPIUs exist and when are they used? |
| **Expert** | Describe the media controller pipeline for a RAW camera → ISP → display path. |
| **Expert** | How would you implement DVFS? What kernel frameworks are involved? |
