# QEMU Embedded Linux Projects — Beginner to Expert (12 Complete Projects)

> **Why QEMU for Embedded?**  
> QEMU lets you develop and test embedded Linux systems without hardware.  
> You can crash the kernel, corrupt the filesystem, experiment freely —  
> then apply what you learn to real boards.

---

## QEMU Setup (One-Time)

```bash
# Install QEMU and build tools
sudo apt install -y qemu-system-arm qemu-system-aarch64 \
    gcc-aarch64-linux-gnu gcc-arm-linux-gnueabihf \
    u-boot-tools device-tree-compiler \
    python3 python3-pip git make flex bison \
    libssl-dev libelf-dev bc

# Verify
qemu-system-aarch64 --version
# QEMU emulator version 6.2.0

aarch64-linux-gnu-gcc --version
# aarch64-linux-gnu-gcc 11.4.0

# Create workspace
mkdir -p ~/qemu-projects && cd ~/qemu-projects

# Clone kernel (do this once, all projects share it)
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
# OR for faster download:
git clone --depth=1 -b v6.6 https://github.com/torvalds/linux.git
```

---

## Project 1: Hello World Embedded Linux (Beginner)

**Goal:** Run a minimal Linux system on QEMU with custom init script.  
**Skills:** Kernel compilation, BusyBox rootfs, QEMU basics.

### Step 1: Build Minimal Kernel

```bash
cd ~/qemu-projects/linux

# Configure for ARM64 QEMU virtual machine
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)

# Output: arch/arm64/boot/Image
ls -lh arch/arm64/boot/Image
# -rwxr-xr-x 1 ... 25M Image
```

### Step 2: Build BusyBox Root Filesystem

```bash
cd ~/qemu-projects
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar -xjf busybox-1.36.0.tar.bz2
cd busybox-1.36.0

# Configure for static linking (no shared libs needed)
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make menuconfig
# Settings → Build BusyBox as a static binary → YES

ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make install
# Installs to ./_install/
```

### Step 3: Create Root Filesystem Image

```bash
cd ~/qemu-projects
mkdir -p rootfs && cd rootfs

# Copy BusyBox
cp -r ../busybox-1.36.0/_install/* .

# Create required directories
mkdir -p proc sys dev tmp etc/init.d

# Create init script (PID 1)
cat > etc/init.d/rcS << 'EOF'
#!/bin/sh
echo "=== My Embedded Linux System ==="
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
echo "Kernel version: $(uname -r)"
echo "CPU info: $(grep 'model name' /proc/cpuinfo | head -1)"
echo "Memory: $(free -m | grep Mem)"
echo "Boot complete!"
echo ""
EOF
chmod +x etc/init.d/rcS

# Create inittab
cat > etc/inittab << 'EOF'
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
EOF

# Package as cpio initramfs
find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz
echo "rootfs size: $(du -sh ../initramfs.cpio.gz)"
```

### Step 4: Run!

```bash
cd ~/qemu-projects

qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 512M \
    -nographic \
    -kernel linux/arch/arm64/boot/Image \
    -initrd initramfs.cpio.gz \
    -append "console=ttyAMA0 rdinit=/sbin/init"

# Expected output:
# === My Embedded Linux System ===
# Kernel version: 6.6.0
# CPU info: model name : Cortex-A57
# Memory: ...
# Boot complete!
# (press Enter for shell)
# / #
```

```bash
# Exit QEMU:
# Ctrl+A, then X
```

---

## Project 2: Virtual Sensor Driver (Beginner+)

**Goal:** Write a kernel driver for a simulated hardware sensor.  
**Skills:** Character driver, platform driver, sysfs, kernel module with QEMU.

### Step 1: Create Virtual Sensor Hardware (QEMU Device)

```bash
# For simplicity, simulate with /dev/mem access to a fixed memory region
# Real approach: write QEMU device in C (Project 6 covers this)
# Here: simulate with a kernel driver that generates fake sensor data
```

### Step 2: Kernel Module

```c
/* virtual_sensor.c */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/random.h>
#include <linux/jiffies.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka");
MODULE_DESCRIPTION("Virtual temperature sensor for QEMU learning");

static struct platform_device *virt_pdev;

/* Simulate temperature: 20-80°C with slow drift */
static int read_temperature(void)
{
    static int base_temp = 25000;  /* millidegrees */
    int noise;
    
    get_random_bytes(&noise, sizeof(noise));
    noise = (noise % 2000) - 1000;  /* ±1°C random noise */
    
    base_temp += noise / 10;
    base_temp = clamp(base_temp, 20000, 80000);
    
    return base_temp;
}

static ssize_t temp_input_show(struct device *dev,
                                struct device_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "%d\n", read_temperature());
}
DEVICE_ATTR_RO(temp_input);

static ssize_t name_show(struct device *dev,
                          struct device_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "virtual_sensor\n");
}
DEVICE_ATTR_RO(name);

static struct attribute *sensor_attrs[] = {
    &dev_attr_temp_input.attr,
    &dev_attr_name.attr,
    NULL,
};
ATTRIBUTE_GROUPS(sensor);

static int virtual_sensor_probe(struct platform_device *pdev)
{
    int ret = device_add_groups(&pdev->dev, sensor_groups);
    if (ret) return ret;
    dev_info(&pdev->dev, "Virtual sensor ready at %s\n",
             dev_name(&pdev->dev));
    return 0;
}

static int virtual_sensor_remove(struct platform_device *pdev)
{
    device_remove_groups(&pdev->dev, sensor_groups);
    return 0;
}

static struct platform_driver virtual_sensor_driver = {
    .probe  = virtual_sensor_probe,
    .remove = virtual_sensor_remove,
    .driver = { .name = "virtual-sensor" },
};

static int __init vsensor_init(void)
{
    int ret;

    ret = platform_driver_register(&virtual_sensor_driver);
    if (ret) return ret;

    /* Create platform device (normally from DT, here manually) */
    virt_pdev = platform_device_alloc("virtual-sensor", 0);
    if (!virt_pdev) {
        platform_driver_unregister(&virtual_sensor_driver);
        return -ENOMEM;
    }

    ret = platform_device_add(virt_pdev);
    if (ret) {
        platform_device_put(virt_pdev);
        platform_driver_unregister(&virtual_sensor_driver);
        return ret;
    }

    return 0;
}

static void __exit vsensor_exit(void)
{
    platform_device_unregister(virt_pdev);
    platform_driver_unregister(&virtual_sensor_driver);
}

module_init(vsensor_init);
module_exit(vsensor_exit);
```

### Step 3: Build and Load in QEMU

```bash
# Makefile
obj-m := virtual_sensor.o
KERNELDIR := ~/qemu-projects/linux

all:
	$(MAKE) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	    -C $(KERNELDIR) M=$(PWD) modules

# Build
make

# Copy to rootfs
cp virtual_sensor.ko ~/qemu-projects/rootfs/

# Boot QEMU and load module
/ # insmod virtual_sensor.ko
/ # cat /sys/bus/platform/devices/virtual-sensor.0/temp_input
# 27543

# Read 10 times rapidly
/ # for i in $(seq 1 10); do cat /sys/bus/platform/devices/virtual-sensor.0/temp_input; done
# 27543
# 27823
# 26991
# ...

# Watch temperature with while loop
/ # while true; do cat /sys/.../temp_input; sleep 1; done
```

---

## Project 3: Virtual UART Driver (Intermediate)

**Goal:** Implement a complete tty/serial driver that works with QEMU's virtual UART.  
**Skills:** TTY subsystem, serial driver, tty_driver operations.

```c
/* virtual_uart.c — minimal serial driver for QEMU emulated UART */
#include <linux/module.h>
#include <linux/tty.h>
#include <linux/tty_driver.h>
#include <linux/tty_flip.h>
#include <linux/serial.h>
#include <linux/timer.h>

#define VUART_MAJOR     240
#define VUART_MINORS    4
#define VUART_BUF_SIZE  256

struct vuart_port {
    struct tty_port     port;
    int                 minor;
    struct timer_list   rx_timer;    /* simulate incoming data */
    spinlock_t          lock;
    char                rx_buf[VUART_BUF_SIZE];
    int                 rx_head, rx_tail;
};

static struct tty_driver *vuart_driver;
static struct vuart_port vuart_ports[VUART_MINORS];

/* Simulate receiving data via timer (for testing) */
static void vuart_rx_timer_fn(struct timer_list *t)
{
    struct vuart_port *p = from_timer(p, t, rx_timer);
    struct tty_port *port = &p->port;
    char data[] = "Hello from virtual UART!\r\n";
    
    /* Push characters into tty flip buffer */
    for (int i = 0; i < sizeof(data)-1; i++) {
        tty_insert_flip_char(port, data[i], TTY_NORMAL);
    }
    tty_flip_buffer_push(port);

    /* Schedule next simulated receive in 5 seconds */
    mod_timer(&p->rx_timer, jiffies + 5 * HZ);
}

/* tty operations */
static int vuart_open(struct tty_struct *tty, struct file *file)
{
    struct vuart_port *p = &vuart_ports[tty->index];
    return tty_port_open(&p->port, tty, file);
}

static void vuart_close(struct tty_struct *tty, struct file *file)
{
    struct vuart_port *p = &vuart_ports[tty->index];
    tty_port_close(&p->port, tty, file);
}

static int vuart_write(struct tty_struct *tty, const unsigned char *buf, int count)
{
    struct vuart_port *p = &vuart_ports[tty->index];
    /* In a real driver: write to UART TX FIFO */
    /* Here: just echo back to aid testing */
    pr_info("vuart%d write: %.*s\n", p->minor, count, buf);
    return count;
}

static int vuart_write_room(struct tty_struct *tty)
{
    return VUART_BUF_SIZE;   /* always ready */
}

static const struct tty_operations vuart_ops = {
    .open       = vuart_open,
    .close      = vuart_close,
    .write      = vuart_write,
    .write_room = vuart_write_room,
};

static int __init vuart_init(void)
{
    int ret;

    vuart_driver = tty_alloc_driver(VUART_MINORS,
                                     TTY_DRIVER_REAL_RAW |
                                     TTY_DRIVER_DYNAMIC_DEV);
    if (IS_ERR(vuart_driver)) return PTR_ERR(vuart_driver);

    vuart_driver->driver_name   = "vuart";
    vuart_driver->name          = "ttyV";
    vuart_driver->major         = VUART_MAJOR;
    vuart_driver->minor_start   = 0;
    vuart_driver->type          = TTY_DRIVER_TYPE_SERIAL;
    vuart_driver->subtype       = SERIAL_TYPE_NORMAL;
    vuart_driver->ops           = &vuart_ops;
    
    for (int i = 0; i < VUART_MINORS; i++) {
        vuart_ports[i].minor = i;
        spin_lock_init(&vuart_ports[i].lock);
        tty_port_init(&vuart_ports[i].port);
        timer_setup(&vuart_ports[i].rx_timer, vuart_rx_timer_fn, 0);
    }

    ret = tty_register_driver(vuart_driver);
    if (ret) { tty_driver_kref_put(vuart_driver); return ret; }

    /* Start simulated RX after 2 seconds */
    mod_timer(&vuart_ports[0].rx_timer, jiffies + 2 * HZ);

    pr_info("Virtual UART driver registered: /dev/ttyV0..%d\n",
            VUART_MINORS - 1);
    return 0;
}

module_init(vuart_init);
/* ... module_exit and cleanup ... */
MODULE_LICENSE("GPL");
```

```bash
# Test in QEMU
insmod virtual_uart.ko
ls /dev/ttyV*      # /dev/ttyV0 to /dev/ttyV3

# Open in one terminal
cat /dev/ttyV0 &   # background reader

# Wait 2 seconds for simulated RX
# Output: Hello from virtual UART!

# Write to UART
echo "test message" > /dev/ttyV0
# Kernel log: vuart0 write: test message
```

---

## Project 4: Virtual I2C Device + Driver (Intermediate)

**Goal:** Create both an I2C device emulator and its Linux driver.  
**Skills:** I2C subsystem, i2c_algorithm, slave simulation.

### Step 1: I2C Software Master Driver

```c
/* virtual_i2c.c — software I2C master using shared memory for simulation */
#include <linux/i2c.h>
#include <linux/platform_device.h>
#include <linux/module.h>

/* Simulate I2C bus in memory */
static u8 i2c_slave_regs[256] = {
    [0x00] = 0x5A,    /* device ID */
    [0x01] = 0x01,    /* status: OK */
    [0x10] = 0x00,    /* temp MSB */
    [0x11] = 0x18,    /* temp LSB: 0x0018 = 24°C */
    [0xFF] = 0x00,    /* reserved */
};
static u8 i2c_slave_addr = 0x48;   /* our simulated device address */

static int virtual_i2c_xfer(struct i2c_adapter *adap,
                              struct i2c_msg *msgs, int num)
{
    for (int i = 0; i < num; i++) {
        struct i2c_msg *msg = &msgs[i];

        if (msg->addr != i2c_slave_addr)
            return -ENXIO;  /* no device at this address */

        if (msg->flags & I2C_M_RD) {
            /* Read from slave */
            for (int j = 0; j < msg->len; j++) {
                /* Auto-increment address (like most real I2C chips) */
                static u8 reg_ptr = 0;
                msg->buf[j] = i2c_slave_regs[reg_ptr++];
            }
        } else {
            /* Write to slave */
            if (msg->len >= 1) {
                /* First byte is register address */
                u8 reg = msg->buf[0];
                for (int j = 1; j < msg->len; j++)
                    i2c_slave_regs[reg + j - 1] = msg->buf[j];
            }
        }
    }
    return num;
}

static u32 virtual_i2c_func(struct i2c_adapter *adap)
{
    return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL;
}

static const struct i2c_algorithm virtual_i2c_algo = {
    .master_xfer = virtual_i2c_xfer,
    .functionality = virtual_i2c_func,
};

static struct i2c_adapter virtual_i2c_adapter = {
    .owner      = THIS_MODULE,
    .class      = I2C_CLASS_HWMON,
    .algo       = &virtual_i2c_algo,
    .name       = "Virtual I2C",
};

static int __init virtual_i2c_init(void)
{
    /* Set adapter number (bus number) */
    virtual_i2c_adapter.nr = 5;  /* /dev/i2c-5 */
    return i2c_add_numbered_adapter(&virtual_i2c_adapter);
}
```

### Step 2: Test I2C Device

```bash
# After loading virtual_i2c.ko
ls /dev/i2c*           # /dev/i2c-5 should appear

# Detect devices on virtual bus
i2cdetect -y 5
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --

# Read register 0x00 (device ID)
i2cget -y 5 0x48 0x00
# 0x5a

# Read temperature registers 0x10, 0x11
i2cget -y 5 0x48 0x10
# 0x00
i2cget -y 5 0x48 0x11
# 0x18   → temperature = 0x0018 = 24°C

# Write to register 0x10 (change temperature)
i2cset -y 5 0x48 0x10 0x00 0x28  # 40°C
i2cget -y 5 0x48 0x11
# 0x28
```

---

## Project 5: Buildroot Custom Embedded Linux (Intermediate+)

**Goal:** Build a complete minimal Linux distribution using Buildroot.  
**Skills:** Buildroot, cross-compilation, custom packages, kernel integration.

```bash
cd ~/qemu-projects
git clone https://github.com/buildroot/buildroot.git
cd buildroot

# Configure for QEMU ARM64 virtual machine
make qemu_aarch64_virt_defconfig

# Customize (optional)
make menuconfig
# Target options → Target Architecture: AArch64
# Toolchain → C library: glibc
# System configuration → System hostname: ravi-embedded
# Target packages → Networking apps → openssh: YES
# Target packages → Development tools → strace: YES

# Build (takes 30-60 min first time)
make -j$(nproc) 2>&1 | tee build.log

# Check outputs
ls output/images/
# Image            ← kernel
# rootfs.ext4      ← root filesystem
# start-qemu.sh    ← launch script

# Run
output/images/start-qemu.sh
# OR
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 1G \
    -nographic \
    -kernel output/images/Image \
    -drive file=output/images/rootfs.ext4,if=virtio,format=raw \
    -append "root=/dev/vda console=ttyAMA0 rootwait" \
    -netdev user,id=eth0 \
    -device virtio-net-device,netdev=eth0

# Login: root (no password)
# ravi-embedded login: root
# #
```

### Adding a Custom Package to Buildroot

```bash
# Create package directory
mkdir -p package/my-app

# Package config
cat > package/my-app/Config.in << 'EOF'
config BR2_PACKAGE_MY_APP
    bool "my-app"
    help
      My custom embedded application
EOF

# Package Makefile
cat > package/my-app/my-app.mk << 'EOF'
MY_APP_VERSION = 1.0
MY_APP_SITE = $(BR2_EXTERNAL_TREE)/src/my-app
MY_APP_SITE_METHOD = local

define MY_APP_BUILD_CMDS
    $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D)
endef

define MY_APP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/my-app $(TARGET_DIR)/usr/bin/my-app
endef

$(eval $(generic-package))
EOF

# Register package
echo 'source "package/my-app/Config.in"' >> package/Config.in

make menuconfig  # Enable BR2_PACKAGE_MY_APP
make
```

---

## Project 6: QEMU Virtual Device (Writing QEMU Device in C) (Advanced)

**Goal:** Write an actual QEMU device model (extends QEMU source).  
**Skills:** QEMU device model API, MMIO regions, IRQ injection, DMA.

```bash
# Get QEMU source
git clone https://github.com/qemu/qemu.git
cd qemu
```

```c
/* hw/misc/my_virtual_device.c */
#include "qemu/osdep.h"
#include "hw/sysbus.h"
#include "hw/qdev-properties.h"
#include "qemu/log.h"
#include "trace.h"

#define TYPE_MY_DEV "my-virtual-device"
OBJECT_DECLARE_SIMPLE_TYPE(MyDevState, MY_DEV)

/* Register layout */
#define REG_ID       0x00    /* Device ID: reads 0xDEADBEEF */
#define REG_STATUS   0x04    /* Status register */
#define REG_CTRL     0x08    /* Control register */
#define REG_DATA     0x0C    /* Data register */
#define REG_IRQ_STS  0x10    /* IRQ status */
#define REG_IRQ_CLR  0x14    /* Write to clear IRQ */

struct MyDevState {
    SysBusDevice    parent;
    MemoryRegion    iomem;    /* MMIO region */
    qemu_irq        irq;      /* interrupt line to CPU */
    uint32_t        status;
    uint32_t        ctrl;
    uint32_t        data;
    uint32_t        irq_status;
};

static uint64_t my_dev_read(void *opaque, hwaddr offset, unsigned size)
{
    MyDevState *s = MY_DEV(opaque);
    uint64_t val = 0;

    switch (offset) {
    case REG_ID:
        val = 0xDEADBEEF;   /* device signature */
        break;
    case REG_STATUS:
        val = s->status;
        break;
    case REG_CTRL:
        val = s->ctrl;
        break;
    case REG_DATA:
        val = s->data;
        /* Side effect: generate IRQ when data is read */
        s->irq_status |= BIT(0);
        qemu_set_irq(s->irq, 1);   /* assert IRQ */
        break;
    case REG_IRQ_STS:
        val = s->irq_status;
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "my-dev: bad read offset 0x%x\n", (int)offset);
    }
    return val;
}

static void my_dev_write(void *opaque, hwaddr offset, uint64_t val, unsigned size)
{
    MyDevState *s = MY_DEV(opaque);

    switch (offset) {
    case REG_CTRL:
        s->ctrl = val;
        if (val & BIT(0)) {  /* reset bit */
            s->status = 0;
            s->data = 0;
        }
        break;
    case REG_DATA:
        s->data = val;
        s->status |= BIT(1);   /* data written flag */
        break;
    case REG_IRQ_CLR:
        s->irq_status &= ~val;  /* clear requested bits */
        if (!s->irq_status)
            qemu_set_irq(s->irq, 0);  /* deassert IRQ */
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "my-dev: bad write offset 0x%x val 0x%llx\n",
                      (int)offset, val);
    }
}

static const MemoryRegionOps my_dev_ops = {
    .read  = my_dev_read,
    .write = my_dev_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = { .min_access_size = 4, .max_access_size = 4 },
};

static void my_dev_realize(DeviceState *dev, Error **errp)
{
    MyDevState *s = MY_DEV(dev);
    SysBusDevice *sbd = SYS_BUS_DEVICE(dev);

    /* Initialize MMIO region — 0x1000 bytes */
    memory_region_init_io(&s->iomem, OBJECT(dev), &my_dev_ops,
                          s, TYPE_MY_DEV, 0x1000);
    sysbus_init_mmio(sbd, &s->iomem);

    /* Initialize IRQ line */
    sysbus_init_irq(sbd, &s->irq);

    /* Default state */
    s->status = 0;
    s->data = 0xA5A5A5A5;  /* initial data */
}

static void my_dev_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    dc->realize = my_dev_realize;
    dc->desc    = "My Virtual Device";
}

static const TypeInfo my_dev_info = {
    .name          = TYPE_MY_DEV,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(MyDevState),
    .class_init    = my_dev_class_init,
};

static void my_dev_register_types(void)
{
    type_register_static(&my_dev_info);
}
type_init(my_dev_register_types)
```

```bash
# Add to QEMU Kconfig and hw/misc/meson.build
# Then build QEMU
mkdir build && cd build
../configure --target-list=aarch64-softmmu
ninja -j$(nproc)

# Use custom device in QEMU launch
./qemu-system-aarch64 -M virt -cpu cortex-a57 -m 512M \
    -device my-virtual-device,mmio=0xFE650000 \
    -device: my-virtual-device gets mapped at 0xFE650000
```

---

## Project 7: Virtual Network Driver + TCP/IP (Intermediate+)

```bash
# QEMU already provides virtio-net (much better than most real NICs)
qemu-system-aarch64 \
    -M virt -cpu cortex-a57 -m 512M -nographic \
    -kernel Image -initrd initramfs.cpio.gz \
    -append "console=ttyAMA0 rdinit=/sbin/init ip=dhcp" \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-net-device,netdev=net0

# Inside QEMU:
# Setup IP
/ # ip addr add 10.0.2.15/24 dev eth0
/ # ip link set eth0 up
/ # ip route add default via 10.0.2.2

# Test connectivity
/ # wget -O- http://10.0.2.2/  # host webserver

# From host: SSH into QEMU
ssh -p 2222 root@localhost
```

---

## Project 8: Secure Boot Chain in QEMU (Advanced)

**Goal:** Implement complete verified boot: U-Boot → signed FIT → kernel.  
**Skills:** RSA signatures, x.509 certificates, FIT image signing.

```bash
mkdir -p ~/qemu-projects/secure-boot && cd ~/qemu-projects/secure-boot

# Step 1: Generate RSA key pair
openssl genrsa -F4 -out sign.key 2048
openssl req -new -x509 -days 3650 -key sign.key -out sign.crt \
    -subj "/CN=Ravi Signing Key/"
openssl rsa -in sign.key -pubout -out sign.pub.pem

# Step 2: Create U-Boot with embedded key
# The key goes into U-Boot's DT at /signature/key-dev
dtc -I dtb -O dts u-boot.dtb -o u-boot.dts
# Add public key manually or via mkimage

# Step 3: Create and sign FIT image
cat > boot.its << 'EOF'
/dts-v1/;
/ {
    images {
        kernel { data = /incbin/("Image"); type = kernel; arch = arm64; 
                 os = linux; compression = none; load = <0x40200000>; 
                 entry = <0x40200000>; hash-1 { algo = sha256; }; };
        fdt    { data = /incbin/("virt.dtb"); type = flat_dt; arch = arm64;
                 compression = none; hash-1 { algo = sha256; }; };
    };
    configurations {
        default = "cfg";
        cfg { kernel = "kernel"; fdt = "fdt";
              signature { algo = sha256,rsa2048; key-name-hint = "sign";
                          sign-images = "kernel", "fdt"; }; };
    };
};
EOF

# Build unsigned FIT
mkimage -f boot.its boot.itb

# Sign FIT
mkimage -F -k . -K u-boot.dtb -r boot.itb

# Step 4: Rebuild U-Boot with the key embedded in its dtb
make DEVICE_TREE=u-boot.dtb   # U-Boot picks up key from dtb

# Step 5: Boot — U-Boot will verify signature before booting kernel
qemu-system-aarch64 -M virt -bios u-boot.bin \
    -drive if=none,file=boot.itb,format=raw,id=fw \
    -device virtio-blk-device,drive=fw \
    -nographic

# U-Boot output:
# Verifying Hash Integrity ... sha256,rsa2048:sign+ OK
# Starting kernel ...
```

---

## Project 9: OTA Update Framework (Advanced)

**Goal:** Build an Over-The-Air update system with A/B partitions and rollback.  
**Skills:** SWUpdate/Mender, A/B boot, U-Boot environment for boot counting.

```bash
# Use SWUpdate (industry-standard OTA framework)
# https://sbabic.github.io/swupdate/

# Enable in Buildroot:
# BR2_PACKAGE_SWUPDATE=y
# BR2_PACKAGE_SWUPDATE_HANDLER_RAW=y

# Create update package
cat > sw-description << 'EOF'
software = {
    version = "2.0";
    description = "My Embedded System v2.0";
    
    hardware-compatibility = ["1.0", "1.1"];
    
    images: (
        {
            filename = "Image";
            type = "rawfile";
            device = "/dev/mmcblk0p3";   /* slot B kernel partition */
            sha256 = "abc123...";
        },
        {
            filename = "rootfs.ext4";
            type = "raw";
            device = "/dev/mmcblk0p4";   /* slot B rootfs */
        }
    );
    
    scripts: (
        {
            filename = "post_install.sh";
            type = "shellscript";
        }
    );
};
EOF

# Build SWU package
mkswu sw-description Image rootfs.ext4 post_install.sh -o update_v2.0.swu

# Deploy OTA update
# On device: starts SWUpdate listener
swupdate -l 4 -w "-document_root /var/www" &

# From host: push update
curl -F file=@update_v2.0.swu \
     http://device-ip:8080/upload
```

---

## Project 10: AI-Powered Log Analyzer in QEMU (Advanced)

**Goal:** Run a kernel log analyzer agent inside QEMU.  
**Skills:** llama.cpp on ARM, Python, QEMU networking.

```bash
# In QEMU with 2GB RAM:
qemu-system-aarch64 -M virt -cpu cortex-a57 -m 2G -nographic \
    -kernel Image -drive file=rootfs.ext4,if=virtio \
    -append "root=/dev/vda rw rootwait console=ttyAMA0" \
    -netdev user,id=net0,hostfwd=tcp::8080-:8080 \
    -device virtio-net-device,netdev=net0

# Install minimal dependencies
apt install python3 python3-pip
pip3 install anthropic  # or use local model

# Create log analyzer
cat > /usr/local/bin/log_analyzer.py << 'PYEOF'
#!/usr/bin/env python3
"""AI-powered kernel log analyzer"""
import subprocess
import anthropic

def get_kernel_log():
    result = subprocess.run(['dmesg', '--level=err,warn,crit'], 
                           capture_output=True, text=True)
    return result.stdout[-3000:]  # last 3000 chars

def analyze_log(log_text):
    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-3-haiku-20240307",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": f"Analyze this kernel log and identify any issues:\n{log_text}"
        }]
    )
    return message.content[0].text

if __name__ == "__main__":
    print("Getting kernel log...")
    log = get_kernel_log()
    print("Analyzing with AI...")
    analysis = analyze_log(log)
    print("\n=== AI Analysis ===")
    print(analysis)
PYEOF

chmod +x /usr/local/bin/log_analyzer.py
ANTHROPIC_API_KEY=your_key python3 /usr/local/bin/log_analyzer.py
```

---

## Project 11: Complete Virtual Camera Pipeline (Expert)

**Goal:** Implement a full V4L2 virtual camera that generates synthetic video frames.  
**Skills:** V4L2, videobuf2, media controller, DMA buffers.

```bash
# Linux already has a virtual camera driver!
# It's called vivid (Virtual Video Test Driver)
modprobe vivid
ls /dev/video*  # /dev/video0 appears

# Vivid generates test patterns
v4l2-ctl -d /dev/video0 --list-formats-ext
v4l2-ctl -d /dev/video0 --set-ctrl test_pattern=0  # color bars
gst-launch-1.0 v4l2src device=/dev/video0 ! autovideosink

# Build a CUSTOM virtual camera
# See drivers/media/platform/vivid/vivid-core.c as reference
# Your implementation should:
# 1. Register V4L2 device (/dev/videoN)
# 2. Implement vb2 operations (buf_prepare, buf_queue, start_streaming)
# 3. Use timer/kthread to generate frames and push to vb2 queue
# 4. Support common formats: YUYV, NV12, RGB24
```

---

## Project 12: Multi-Agent Embedded AI Platform (Expert)

**Goal:** Deploy multiple AI agents on QEMU: log analyzer + DT validator + code reviewer.  
**Skills:** Python async, FastAPI, LangGraph, QEMU networking.

```bash
# This combines Project 10 with the agents from 19_AI_Agents/
# Run the full agent platform in QEMU

# 1. Build Buildroot image with Python + networking
# 2. Deploy agent server (from 19_AI_Agents/01_Complete_Agent_Building.md)
# 3. Expose via FastAPI on port 8080
# 4. Access from host browser: http://localhost:8080

# Launch script
cat > /etc/init.d/S99ai_agents << 'EOF'
#!/bin/sh
case "$1" in
  start)
    cd /opt/ai_agents
    uvicorn api.main:app --host 0.0.0.0 --port 8080 &
    echo "AI Agents started on port 8080"
    ;;
  stop)
    killall uvicorn
    ;;
esac
EOF
```

---

## Project Progression Summary

| # | Project | Level | Key Skills | Time |
|---|---------|-------|-----------|------|
| 1 | Hello World Embedded Linux | Beginner | Kernel build, BusyBox, QEMU | 2h |
| 2 | Virtual Sensor Driver | Beginner+ | Platform driver, sysfs | 3h |
| 3 | Virtual UART Driver | Intermediate | TTY subsystem, serial | 5h |
| 4 | Virtual I2C Device | Intermediate | I2C algorithm, slave sim | 4h |
| 5 | Buildroot Custom Distro | Intermediate+ | Buildroot, packages, rootfs | 6h |
| 6 | QEMU Virtual Device | Advanced | QEMU device model, MMIO | 8h |
| 7 | Network Driver + TCP/IP | Intermediate+ | virtio-net, networking | 4h |
| 8 | Secure Boot Chain | Advanced | RSA, FIT signing, U-Boot | 8h |
| 9 | OTA Update Framework | Advanced | SWUpdate, A/B partitions | 8h |
| 10 | AI Log Analyzer | Advanced | LLM, Python, QEMU | 6h |
| 11 | Virtual Camera V4L2 | Expert | V4L2, vb2, media ctrl | 12h |
| 12 | Multi-Agent AI Platform | Expert | LangGraph, FastAPI, async | 16h |
