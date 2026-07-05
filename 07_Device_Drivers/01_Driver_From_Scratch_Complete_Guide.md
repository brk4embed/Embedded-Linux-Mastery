# Writing Linux Device Drivers From Scratch — Complete Guide

> **Goal:** By the end of this file you will have written 5 real drivers: a character driver, a platform driver, an I2C driver, a SPI driver, and a driver with DMA. Every concept is explained from zero.

---

## The Driver Development Mindset

Before writing a single line of code, understand this:

```
Every Linux driver does exactly 3 things:
1. CLAIM hardware resources (registers, IRQs, clocks, GPIOs)
   → In probe()
2. SERVE user-space requests (read, write, ioctl)
   → In file_operations or subsystem callbacks
3. RELEASE all resources when done
   → In remove()

If you understand these 3 things, you can write ANY driver.
```

---

## Part 1: Hello World — Simplest Possible Kernel Module

### What This Teaches: Module mechanics, printk, basic compilation

```c
/* hello_driver.c */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

/* Module metadata — required for upstream submission */
MODULE_AUTHOR("Ravi Kumar Bokka <brk4embed@gmail.com>");
MODULE_DESCRIPTION("Hello World kernel module");
MODULE_LICENSE("GPL");   /* Must be GPL to use GPL-only kernel symbols */
MODULE_VERSION("1.0");

/* Called when module is loaded (insmod / modprobe) */
static int __init hello_init(void)
{
    pr_info("hello_driver: Hello, Kernel World!\n");
    pr_info("hello_driver: Running on CPU %d\n", smp_processor_id());
    return 0;   /* 0 = success, negative = error */
}

/* Called when module is unloaded (rmmod) */
static void __exit hello_exit(void)
{
    pr_info("hello_driver: Goodbye, Kernel World!\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

```makefile
# Makefile (must be named exactly 'Makefile')
# Replace /path/to/linux with your kernel source or:
# KERNELDIR := /lib/modules/$(shell uname -r)/build

obj-m := hello_driver.o
KERNELDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean
```

```bash
# Build
make

# Load
sudo insmod hello_driver.ko

# See kernel log
dmesg | tail -5
# [12345.678] hello_driver: Hello, Kernel World!
# [12345.679] hello_driver: Running on CPU 0

# Check it's loaded
lsmod | grep hello_driver

# Unload
sudo rmmod hello_driver
dmesg | tail -3
# [12350.123] hello_driver: Goodbye, Kernel World!
```

---

## Part 2: Character Driver — Read and Write to /dev/mydevice

### What This Teaches: file_operations, /dev entry, read/write from user space

A character driver exposes a `/dev/` file that user space can open, read, and write.

```c
/* chardev.c — A complete character driver */
#include <linux/module.h>
#include <linux/fs.h>        /* file_operations, alloc_chrdev_region */
#include <linux/cdev.h>      /* cdev_init, cdev_add */
#include <linux/device.h>    /* class_create, device_create */
#include <linux/uaccess.h>   /* copy_to_user, copy_from_user */
#include <linux/slab.h>      /* kmalloc, kfree */

#define DEVICE_NAME "mydevice"
#define CLASS_NAME  "myclass"
#define BUF_SIZE    1024

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka <brk4embed@gmail.com>");
MODULE_DESCRIPTION("Simple character driver with read/write");

/* Per-device state */
struct chardev_data {
    struct cdev     cdev;           /* kernel's char device structure */
    char            *buf;           /* kernel buffer */
    size_t          buf_len;        /* current data length */
    struct mutex    lock;           /* protects buf and buf_len */
};

/* Device numbers and class — module-level */
static dev_t dev_number;           /* major:minor number */
static struct class *dev_class;    /* /sys/class/myclass/ */
static struct chardev_data *mydev; /* our device data */

/* ─── file_operations ────────────────────────────────────────── */

static int chardev_open(struct inode *inode, struct file *file)
{
    /* Get our per-device data from the cdev embedded in inode */
    struct chardev_data *dev = container_of(inode->i_cdev,
                                            struct chardev_data, cdev);
    /* Store in file->private_data for fast access in read/write */
    file->private_data = dev;
    
    pr_info("chardev: device opened (pid=%d)\n", current->pid);
    return 0;
}

static int chardev_release(struct inode *inode, struct file *file)
{
    pr_info("chardev: device closed\n");
    return 0;
}

static ssize_t chardev_read(struct file *file, char __user *ubuf,
                             size_t count, loff_t *ppos)
{
    struct chardev_data *dev = file->private_data;
    ssize_t ret;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;   /* interrupted by signal */

    /* Nothing to read? */
    if (dev->buf_len == 0) {
        ret = 0;
        goto unlock;
    }

    /* Don't read more than available */
    if (count > dev->buf_len)
        count = dev->buf_len;

    /* Copy from kernel buffer to user space */
    if (copy_to_user(ubuf, dev->buf, count)) {
        ret = -EFAULT;   /* bad user pointer */
        goto unlock;
    }

    /* Consume the data (shift remaining bytes) */
    dev->buf_len -= count;
    memmove(dev->buf, dev->buf + count, dev->buf_len);

    ret = count;
    pr_info("chardev: read %zu bytes\n", count);

unlock:
    mutex_unlock(&dev->lock);
    return ret;
}

static ssize_t chardev_write(struct file *file, const char __user *ubuf,
                              size_t count, loff_t *ppos)
{
    struct chardev_data *dev = file->private_data;
    ssize_t ret;

    if (count > BUF_SIZE)
        return -EINVAL;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;

    /* Copy from user space to kernel buffer */
    if (copy_from_user(dev->buf, ubuf, count)) {
        ret = -EFAULT;
        goto unlock;
    }

    dev->buf_len = count;
    ret = count;
    pr_info("chardev: wrote %zu bytes\n", count);

unlock:
    mutex_unlock(&dev->lock);
    return ret;
}

/* ioctl — custom commands */
#define CHARDEV_MAGIC      'c'
#define CHARDEV_CLEAR      _IO(CHARDEV_MAGIC, 0)
#define CHARDEV_GET_LEN    _IOR(CHARDEV_MAGIC, 1, int)

static long chardev_ioctl(struct file *file, unsigned int cmd,
                           unsigned long arg)
{
    struct chardev_data *dev = file->private_data;
    int len;

    switch (cmd) {
    case CHARDEV_CLEAR:
        mutex_lock(&dev->lock);
        dev->buf_len = 0;
        mutex_unlock(&dev->lock);
        pr_info("chardev: buffer cleared\n");
        return 0;

    case CHARDEV_GET_LEN:
        mutex_lock(&dev->lock);
        len = dev->buf_len;
        mutex_unlock(&dev->lock);
        if (copy_to_user((int __user *)arg, &len, sizeof(len)))
            return -EFAULT;
        return 0;

    default:
        return -ENOTTY;   /* unknown ioctl command */
    }
}

static const struct file_operations chardev_fops = {
    .owner          = THIS_MODULE,
    .open           = chardev_open,
    .release        = chardev_release,
    .read           = chardev_read,
    .write          = chardev_write,
    .unlocked_ioctl = chardev_ioctl,
};

/* ─── Module init/exit ───────────────────────────────────────── */

static int __init chardev_init(void)
{
    int ret;

    /* Allocate device state */
    mydev = kzalloc(sizeof(*mydev), GFP_KERNEL);
    if (!mydev) return -ENOMEM;

    mydev->buf = kmalloc(BUF_SIZE, GFP_KERNEL);
    if (!mydev->buf) { ret = -ENOMEM; goto err_free_dev; }

    mutex_init(&mydev->lock);

    /* Allocate a major:minor device number dynamically */
    ret = alloc_chrdev_region(&dev_number, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        pr_err("chardev: Failed to allocate device number\n");
        goto err_free_buf;
    }
    pr_info("chardev: major=%d minor=%d\n", MAJOR(dev_number), MINOR(dev_number));

    /* Initialize cdev and link to our file_operations */
    cdev_init(&mydev->cdev, &chardev_fops);
    mydev->cdev.owner = THIS_MODULE;

    /* Add cdev to kernel (makes /dev/mydevice accessible) */
    ret = cdev_add(&mydev->cdev, dev_number, 1);
    if (ret < 0) goto err_unreg;

    /* Create /sys/class/myclass/ */
    dev_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(dev_class)) {
        ret = PTR_ERR(dev_class);
        goto err_cdev_del;
    }

    /* Create /dev/mydevice (udev/mdev will create the file) */
    if (IS_ERR(device_create(dev_class, NULL, dev_number,
                              NULL, DEVICE_NAME))) {
        ret = -ENOMEM;
        goto err_class;
    }

    pr_info("chardev: /dev/%s created\n", DEVICE_NAME);
    return 0;

err_class:
    class_destroy(dev_class);
err_cdev_del:
    cdev_del(&mydev->cdev);
err_unreg:
    unregister_chrdev_region(dev_number, 1);
err_free_buf:
    kfree(mydev->buf);
err_free_dev:
    kfree(mydev);
    return ret;
}

static void __exit chardev_exit(void)
{
    device_destroy(dev_class, dev_number);
    class_destroy(dev_class);
    cdev_del(&mydev->cdev);
    unregister_chrdev_region(dev_number, 1);
    kfree(mydev->buf);
    kfree(mydev);
    pr_info("chardev: module removed\n");
}

module_init(chardev_init);
module_exit(chardev_exit);
```

### Test the Character Driver

```bash
sudo insmod chardev.ko
ls /dev/mydevice      # should exist now

# Write data
echo "Hello from user space" > /dev/mydevice

# Read data back
cat /dev/mydevice
# Hello from user space

# Test with Python
python3 -c "
with open('/dev/mydevice', 'wb') as f:
    f.write(b'Binary data: \x01\x02\x03\x04')
with open('/dev/mydevice', 'rb') as f:
    data = f.read()
    print('Got:', data.hex())
"
```

---

## Part 3: Platform Driver — The Standard Embedded Driver Pattern

Platform drivers are the most common driver type in embedded Linux. They handle SoC-integrated peripherals described in the Device Tree.

### The Device Tree Entry (Hardware Description)

```dts
/* In your board's .dts file */
my_counter: counter@fe650000 {
    compatible = "myvendor,my-counter";   /* Must match driver's of_match_table */
    reg = <0x0 0xfe650000 0x0 0x1000>;   /* MMIO base addr + size */
    interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk_controller 5>;
    clock-names = "counter_clk";
    status = "okay";
};
```

### The Platform Driver

```c
/* platform_counter.c — Complete platform driver with IRQ, clocks, sysfs */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/io.h>           /* ioremap, readl, writel */
#include <linux/interrupt.h>
#include <linux/clk.h>
#include <linux/of.h>           /* of_match_table, device_property_read_u32 */
#include <linux/sysfs.h>
#include <linux/pm_runtime.h>   /* power management */

#define REG_CTRL        0x00
#define REG_COUNT       0x04
#define REG_STATUS      0x08
#define REG_IRQ_STATUS  0x0C
#define REG_IRQ_CLEAR   0x10

#define CTRL_ENABLE     BIT(0)
#define CTRL_RESET      BIT(1)
#define IRQ_OVERFLOW    BIT(0)

struct counter_priv {
    void __iomem        *base;          /* mapped MMIO base address */
    struct clk          *clk;           /* counter clock */
    int                 irq;            /* IRQ number */
    u32                 overflow_count; /* software overflow counter */
    spinlock_t          lock;           /* protects registers + overflow_count */
    struct device       *dev;           /* back-pointer to device */
};

/* ─── sysfs attributes ───────────────────────────────────────── */

static ssize_t count_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct counter_priv *priv = dev_get_drvdata(dev);
    unsigned long flags;
    u32 count;

    spin_lock_irqsave(&priv->lock, flags);
    count = readl(priv->base + REG_COUNT);
    spin_unlock_irqrestore(&priv->lock, flags);

    return sysfs_emit(buf, "%u\n", count);
}

static ssize_t enable_store(struct device *dev,
                             struct device_attribute *attr,
                             const char *buf, size_t count)
{
    struct counter_priv *priv = dev_get_drvdata(dev);
    bool enable;
    int ret;
    u32 ctrl;

    ret = kstrtobool(buf, &enable);
    if (ret)
        return ret;

    spin_lock(&priv->lock);
    ctrl = readl(priv->base + REG_CTRL);
    if (enable)
        ctrl |= CTRL_ENABLE;
    else
        ctrl &= ~CTRL_ENABLE;
    writel(ctrl, priv->base + REG_CTRL);
    spin_unlock(&priv->lock);

    return count;
}

static DEVICE_ATTR_RO(count);      /* read-only: count_show only */
static DEVICE_ATTR_WO(enable);     /* write-only: enable_store only */
/* For read-write: DEVICE_ATTR_RW(name) — needs name_show AND name_store */

static struct attribute *counter_attrs[] = {
    &dev_attr_count.attr,
    &dev_attr_enable.attr,
    NULL,
};
ATTRIBUTE_GROUPS(counter);    /* creates counter_groups[] */

/* ─── IRQ handler ────────────────────────────────────────────── */

static irqreturn_t counter_irq_handler(int irq, void *dev_id)
{
    struct counter_priv *priv = dev_id;
    u32 status;

    status = readl(priv->base + REG_IRQ_STATUS);
    if (!(status & IRQ_OVERFLOW))
        return IRQ_NONE;

    /* Clear interrupt */
    writel(IRQ_OVERFLOW, priv->base + REG_IRQ_CLEAR);

    /* Track overflows */
    spin_lock(&priv->lock);
    priv->overflow_count++;
    spin_unlock(&priv->lock);

    dev_dbg(priv->dev, "overflow #%u\n", priv->overflow_count);
    return IRQ_HANDLED;
}

/* ─── probe() — called by kernel when DT matches driver ─────── */

static int counter_probe(struct platform_device *pdev)
{
    struct counter_priv *priv;
    struct resource *res;
    int ret;

    dev_info(&pdev->dev, "probe() called\n");

    /* Allocate private data (auto-freed on remove via devm) */
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->dev = &pdev->dev;
    spin_lock_init(&priv->lock);

    /* Get MMIO resource from DT and map it */
    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);

    dev_info(&pdev->dev, "registers mapped at %p\n", priv->base);

    /* Get clock from DT */
    priv->clk = devm_clk_get(&pdev->dev, "counter_clk");
    if (IS_ERR(priv->clk))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->clk),
                             "Failed to get clock\n");

    /* Enable the clock */
    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "Failed to enable clock\n");

    /* Get IRQ from DT */
    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0)
        return priv->irq;

    /* Request IRQ */
    ret = devm_request_irq(&pdev->dev, priv->irq,
                            counter_irq_handler,
                            0,                    /* flags */
                            dev_name(&pdev->dev), /* name in /proc/interrupts */
                            priv);
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "Failed to request IRQ\n");

    /* Initialize hardware */
    writel(CTRL_RESET, priv->base + REG_CTRL);   /* reset counter */
    writel(0, priv->base + REG_CTRL);             /* release reset */

    /* Read optional DT property */
    u32 init_value = 0;
    device_property_read_u32(&pdev->dev, "myvendor,initial-value", &init_value);
    writel(init_value, priv->base + REG_COUNT);

    /* Store priv in device for access from sysfs/IRQ */
    platform_set_drvdata(pdev, priv);

    /* Create sysfs attributes: /sys/devices/.../count, enable */
    ret = device_add_groups(&pdev->dev, counter_groups);
    if (ret)
        return ret;

    dev_info(&pdev->dev, "probe() succeeded — IRQ=%d\n", priv->irq);
    return 0;
}

static int counter_remove(struct platform_device *pdev)
{
    struct counter_priv *priv = platform_get_drvdata(pdev);

    /* Remove sysfs attributes */
    device_remove_groups(&pdev->dev, counter_groups);

    /* Disable hardware */
    writel(0, priv->base + REG_CTRL);

    /* Disable clock (devm handles irq, ioremap, kfree automatically) */
    clk_disable_unprepare(priv->clk);

    dev_info(&pdev->dev, "remove() called\n");
    return 0;
}

/* ─── Power Management ───────────────────────────────────────── */

static int counter_suspend(struct device *dev)
{
    struct counter_priv *priv = dev_get_drvdata(dev);
    /* Save state, disable hardware */
    clk_disable_unprepare(priv->clk);
    return 0;
}

static int counter_resume(struct device *dev)
{
    struct counter_priv *priv = dev_get_drvdata(dev);
    /* Re-enable hardware */
    return clk_prepare_enable(priv->clk);
}

static DEFINE_SIMPLE_DEV_PM_OPS(counter_pm_ops,
                                 counter_suspend, counter_resume);

/* ─── Device Tree match table ───────────────────────────────── */

static const struct of_device_id counter_of_match[] = {
    { .compatible = "myvendor,my-counter" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, counter_of_match);

/* ─── Platform driver registration ─────────────────────────── */

static struct platform_driver counter_driver = {
    .probe  = counter_probe,
    .remove = counter_remove,
    .driver = {
        .name           = "my-counter",
        .of_match_table = counter_of_match,
        .pm             = pm_sleep_ptr(&counter_pm_ops),
    },
};
module_platform_driver(counter_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka <brk4embed@gmail.com>");
MODULE_DESCRIPTION("Example platform driver with IRQ, clocks, sysfs, PM");
```

---

## Part 4: I2C Driver — Talking to I2C Devices

### How I2C Works (Beginner)

```
I2C bus: 2 wires — SCL (clock), SDA (data)
  Master (SoC): controls the bus, initiates transactions
  Slave (sensor): responds to its address

Transaction format:
  START | ADDR(7-bit) | R/W | ACK | DATA | DATA | ... | STOP

Writing register 0x10 with value 0x42 to device at address 0x48:
  START | 0x48 | W | ACK | 0x10 | ACK | 0x42 | ACK | STOP

Reading register 0x10 from device 0x48:
  START | 0x48 | W | ACK | 0x10 | ACK |
  REPEATED START | 0x48 | R | ACK | [data] | NACK | STOP
```

### I2C Driver for a Temperature Sensor (LM75 style)

```c
/* i2c_temp_sensor.c */
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/hwmon.h>
#include <linux/hwmon-sysfs.h>

struct temp_sensor_data {
    struct i2c_client   *client;
    struct device       *hwmon_dev;
};

/* Read temperature register */
static int read_temperature(struct i2c_client *client)
{
    u8 reg_addr = 0x00;     /* temperature register address */
    s16 raw;
    int ret;

    /* Method 1: i2c_smbus (for standard SMBus operations) */
    ret = i2c_smbus_read_word_swapped(client, reg_addr);
    if (ret < 0)
        return ret;
    raw = (s16)ret;
    /* Convert: 9-bit 2's complement, 0.5°C resolution */
    return (raw >> 7) * 500;   /* return in millidegrees */
}

/* Hwmon sysfs: /sys/class/hwmon/hwmonX/temp1_input */
static umode_t temp_is_visible(const void *data, enum hwmon_sensor_types type,
                                u32 attr, int channel)
{
    return 0444;   /* readable */
}

static int temp_read(struct device *dev, enum hwmon_sensor_types type,
                     u32 attr, int channel, long *val)
{
    struct temp_sensor_data *priv = dev_get_drvdata(dev);
    int temp;

    if (type != hwmon_temp || attr != hwmon_temp_input)
        return -EOPNOTSUPP;

    temp = read_temperature(priv->client);
    if (temp < 0)
        return temp;

    *val = temp;
    return 0;
}

static const struct hwmon_ops temp_ops = {
    .is_visible = temp_is_visible,
    .read = temp_read,
};

static const struct hwmon_channel_info * const temp_info[] = {
    HWMON_CHANNEL_INFO(temp, HWMON_T_INPUT),
    NULL,
};

static const struct hwmon_chip_info temp_chip_info = {
    .ops = &temp_ops,
    .info = temp_info,
};

/* probe — called when I2C device is detected */
static int temp_sensor_probe(struct i2c_client *client)
{
    struct temp_sensor_data *priv;
    int temp;

    dev_info(&client->dev, "probe: addr=0x%02x\n", client->addr);

    /* Verify I2C adapter supports smbus_read_word */
    if (!i2c_check_functionality(client->adapter,
                                  I2C_FUNC_SMBUS_READ_WORD_DATA))
        return -EOPNOTSUPP;

    priv = devm_kzalloc(&client->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->client = client;
    i2c_set_clientdata(client, priv);

    /* Test read: get first temperature */
    temp = read_temperature(client);
    if (temp < 0)
        return dev_err_probe(&client->dev, temp,
                             "Failed to read temperature\n");

    dev_info(&client->dev, "initial temp: %d.%03d C\n",
             temp / 1000, temp % 1000);

    /* Register with hwmon subsystem */
    priv->hwmon_dev = devm_hwmon_device_register_with_info(
        &client->dev, "temp_sensor", priv, &temp_chip_info, NULL);

    return PTR_ERR_OR_ZERO(priv->hwmon_dev);
}

/* Device Tree match */
static const struct of_device_id temp_sensor_of_match[] = {
    { .compatible = "myvendor,temp-sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, temp_sensor_of_match);

/* I2C device IDs (for non-DT systems) */
static const struct i2c_device_id temp_sensor_id[] = {
    { "temp-sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, temp_sensor_id);

static struct i2c_driver temp_sensor_driver = {
    .driver = {
        .name           = "temp-sensor",
        .of_match_table = temp_sensor_of_match,
    },
    .probe          = temp_sensor_probe,
    .id_table       = temp_sensor_id,
};
module_i2c_driver(temp_sensor_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ravi Kumar Bokka <brk4embed@gmail.com>");
MODULE_DESCRIPTION("I2C temperature sensor driver");
```

### Device Tree Entry

```dts
&i2c0 {
    status = "okay";
    
    temp_sensor: temp-sensor@48 {
        compatible = "myvendor,temp-sensor";
        reg = <0x48>;   /* 7-bit I2C address */
    };
};
```

### Test on Radxa 5B+

```bash
# Load driver
sudo insmod i2c_temp_sensor.ko

# Check hwmon
ls /sys/class/hwmon/
# hwmon0  hwmon1  hwmon2  (new entry appeared)

# Read temperature
cat /sys/class/hwmon/hwmon2/temp1_input
# 25500   (= 25.500°C)

# Use sensors command
sudo apt install lm-sensors
sensors
```

---

## Part 5: DMA Driver — Zero-Copy Data Transfer

### What is DMA and Why It Matters

Without DMA:
```
CPU reads 1MB from UART: 1,000,000 × (read register → store to RAM) = CPU busy for milliseconds
```

With DMA:
```
CPU programs DMA controller once: "copy from UART FIFO to 0x80000000, 1MB"
DMA hardware does the copy independently
CPU receives interrupt when done: "1MB transferred"
```

```c
/* dma_driver.c — DMA buffer allocation and usage */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/dma-mapping.h>    /* dma_alloc_coherent, dma_map_single */
#include <linux/dmaengine.h>      /* dma_chan, dmaengine_prep_dma_memcpy */

struct dma_dev_priv {
    struct device           *dev;
    struct dma_chan         *dma_chan;   /* DMA channel */
    void                    *src_buf;   /* source buffer (kernel virtual) */
    dma_addr_t              src_dma;    /* source DMA address (device sees this) */
    void                    *dst_buf;   /* destination buffer */
    dma_addr_t              dst_dma;
    size_t                  buf_size;
    struct completion       dma_done;   /* signals DMA completion */
};

/* DMA completion callback — called from softirq context */
static void dma_complete_callback(void *data)
{
    struct dma_dev_priv *priv = data;
    dev_info(priv->dev, "DMA transfer complete!\n");
    complete(&priv->dma_done);   /* wake up waiter */
}

/* Perform a DMA memory copy */
static int do_dma_transfer(struct dma_dev_priv *priv, size_t len)
{
    struct dma_async_tx_descriptor *tx;
    dma_cookie_t cookie;
    int ret;

    /* Fill source buffer with test data */
    memset(priv->src_buf, 0xAB, len);
    memset(priv->dst_buf, 0x00, len);

    /* Prepare DMA descriptor */
    tx = dmaengine_prep_dma_memcpy(
        priv->dma_chan,     /* DMA channel to use */
        priv->dst_dma,      /* destination physical address */
        priv->src_dma,      /* source physical address */
        len,                /* number of bytes to copy */
        DMA_CTRL_ACK | DMA_PREP_INTERRUPT  /* flags */
    );
    if (!tx)
        return -ENOMEM;

    /* Set callback to be called when DMA finishes */
    tx->callback = dma_complete_callback;
    tx->callback_param = priv;

    /* Submit to DMA engine queue */
    cookie = dmaengine_submit(tx);
    if (dma_submit_error(cookie))
        return -EIO;

    /* Synchronize memory before DMA starts */
    dma_sync_single_for_device(priv->dev, priv->src_dma,
                                len, DMA_TO_DEVICE);

    /* Start the DMA transfer */
    dma_async_issue_pending(priv->dma_chan);

    /* Wait for completion (with 5 second timeout) */
    ret = wait_for_completion_timeout(&priv->dma_done, msecs_to_jiffies(5000));
    if (!ret) {
        dev_err(priv->dev, "DMA timeout!\n");
        dmaengine_terminate_sync(priv->dma_chan);
        return -ETIMEDOUT;
    }

    /* Sync for CPU access after DMA */
    dma_sync_single_for_cpu(priv->dev, priv->dst_dma, len, DMA_FROM_DEVICE);

    /* Verify transfer */
    if (memcmp(priv->src_buf, priv->dst_buf, len)) {
        dev_err(priv->dev, "DMA data mismatch!\n");
        return -EIO;
    }

    dev_info(priv->dev, "DMA transferred %zu bytes successfully\n", len);
    return 0;
}

static int dma_driver_probe(struct platform_device *pdev)
{
    struct dma_dev_priv *priv;
    int ret;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv) return -ENOMEM;
    priv->dev = &pdev->dev;
    priv->buf_size = 1 * 1024 * 1024;   /* 1MB */
    init_completion(&priv->dma_done);

    /* Get DMA channel (from DT: dmas = <&dma_controller 0>) */
    priv->dma_chan = dma_request_chan(&pdev->dev, "dma0");
    if (IS_ERR(priv->dma_chan))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->dma_chan),
                             "Failed to get DMA channel\n");

    /* Allocate DMA-coherent buffers */
    /* dma_alloc_coherent: physically contiguous, CPU + device can access */
    priv->src_buf = dma_alloc_coherent(&pdev->dev, priv->buf_size,
                                        &priv->src_dma, GFP_KERNEL);
    if (!priv->src_buf) { ret = -ENOMEM; goto err_chan; }

    priv->dst_buf = dma_alloc_coherent(&pdev->dev, priv->buf_size,
                                        &priv->dst_dma, GFP_KERNEL);
    if (!priv->dst_buf) { ret = -ENOMEM; goto err_src; }

    platform_set_drvdata(pdev, priv);

    /* Run test transfer */
    ret = do_dma_transfer(priv, priv->buf_size);
    if (ret) goto err_dst;

    dev_info(&pdev->dev, "DMA driver probe success\n");
    return 0;

err_dst:
    dma_free_coherent(&pdev->dev, priv->buf_size, priv->dst_buf, priv->dst_dma);
err_src:
    dma_free_coherent(&pdev->dev, priv->buf_size, priv->src_buf, priv->src_dma);
err_chan:
    dma_release_channel(priv->dma_chan);
    return ret;
}

static int dma_driver_remove(struct platform_device *pdev)
{
    struct dma_dev_priv *priv = platform_get_drvdata(pdev);
    dma_free_coherent(&pdev->dev, priv->buf_size, priv->dst_buf, priv->dst_dma);
    dma_free_coherent(&pdev->dev, priv->buf_size, priv->src_buf, priv->src_dma);
    dma_release_channel(priv->dma_chan);
    return 0;
}
```

---

## Part 6: Common Driver Mistakes and How to Avoid Them

### Mistake 1: Sleeping in Atomic Context

```c
/* WRONG — kmalloc(GFP_KERNEL) may sleep, called in IRQ handler */
static irqreturn_t my_irq(int irq, void *data)
{
    void *buf = kmalloc(100, GFP_KERNEL);   /* BUG: may sleep! */
    ...
}

/* RIGHT — use GFP_ATOMIC in interrupt context */
static irqreturn_t my_irq(int irq, void *data)
{
    void *buf = kmalloc(100, GFP_ATOMIC);   /* correct */
    ...
}
```

### Mistake 2: Missing devm_ — Causing Memory/Resource Leaks

```c
/* WRONG — if probe() fails halfway, irq_req and ioremap leak */
static int my_probe(struct platform_device *pdev)
{
    void __iomem *base = ioremap(res->start, res->end - res->start);
    int irq = platform_get_irq(pdev, 0);
    request_irq(irq, handler, 0, "drv", priv);
    
    ret = some_other_init();
    if (ret) {
        /* FORGOT to free_irq and iounmap! */
        return ret;
    }
}

/* RIGHT — devm_ resources auto-freed on any failure or remove() */
static int my_probe(struct platform_device *pdev)
{
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base)) return PTR_ERR(base);
    
    int irq = platform_get_irq(pdev, 0);
    devm_request_irq(&pdev->dev, irq, handler, 0, "drv", priv);
    /* All resources auto-freed if probe returns error or remove called */
}
```

### Mistake 3: Not Using dev_err_probe()

```c
/* OK but verbose */
if (ret < 0) {
    dev_err(&pdev->dev, "Failed to get clock: %d\n", ret);
    return ret;
}

/* BETTER — dev_err_probe: handles -EPROBE_DEFER silently */
/* -EPROBE_DEFER means "try again later when dependency is ready" */
if (IS_ERR(priv->clk))
    return dev_err_probe(&pdev->dev, PTR_ERR(priv->clk),
                         "Failed to get clock\n");
/* dev_err_probe does NOT print for -EPROBE_DEFER (avoids log spam) */
```

### Mistake 4: Not Checking copy_to_user Return Value

```c
/* WRONG */
copy_to_user(ubuf, kbuf, count);   /* return value ignored! */

/* RIGHT */
if (copy_to_user(ubuf, kbuf, count))
    return -EFAULT;
```

---

## Interview Questions — Driver Development

| Level | Question | Key Answer |
|-------|----------|-----------|
| **Beginner** | What is `module_init` and `module_exit`? | Macros that register init/exit functions called by kernel on insmod/rmmod |
| **Beginner** | What is `copy_to_user`? Why can't you just use memcpy? | User pointers may be invalid/unmapped; copy_to_user validates and handles page faults safely |
| **Intermediate** | What is a `devm_` function? When should you use it? | Resource-managed: auto-freed on device removal or probe failure |
| **Intermediate** | What is the difference between `dev_err` and `pr_err`? | dev_err includes device name in message; pr_err is module-global |
| **Intermediate** | What is `platform_set_drvdata` and why is it used? | Stores driver private data in device struct for retrieval in other callbacks |
| **Advanced** | Explain `GFP_KERNEL` vs `GFP_ATOMIC`. When must you use each? | GFP_KERNEL can sleep (process context); GFP_ATOMIC no sleep (IRQ/spinlock) |
| **Advanced** | What is `EPROBE_DEFER` and when should a driver return it? | Dependency (clock, GPIO, IRQ) not yet ready; kernel will retry probe later |
| **Advanced** | How does the kernel's `of_match_table` work? | Kernel compares DT `compatible` strings to driver's table; calls probe on match |
| **Expert** | Explain DMA coherence. What is the difference between coherent and streaming DMA? | Coherent: always CPU+device in sync (slower); streaming: explicit sync points (faster) |
| **Expert** | How would you add power management to a platform driver? What suspend/resume must do? | Implement dev_pm_ops; suspend: save state, disable HW, disable clocks; resume: reverse |
