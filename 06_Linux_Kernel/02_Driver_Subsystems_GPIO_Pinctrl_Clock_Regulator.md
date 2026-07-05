# Linux Kernel Driver Subsystems — GPIO, Pinctrl, Clock, Regulator, DMA

> **Why These 5 Frameworks Matter:**  
> Every embedded Linux driver depends on these. GPIO for control signals, Pinctrl to configure them,  
> Clock to enable hardware, Regulator for power, DMA for data movement.  
> Master these 5 and you can understand ANY driver in the kernel.

---

## The Dependency Chain

```
Hardware block (e.g., SPI controller)
    │
    ├── needs Power      → Regulator Framework
    ├── needs Clock      → Clock Framework  
    ├── needs Pins       → Pinctrl Subsystem
    ├── needs GPIO       → GPIO Subsystem (often via Pinctrl)
    └── needs DMA        → DMA Engine Framework

Every driver's probe() usually:
  1. devm_clk_get() + clk_prepare_enable()
  2. devm_regulator_get() + regulator_enable()
  3. pinctrl_get() + pinctrl_select_state() [usually auto via DT]
  4. devm_gpiod_get() [for control signals]
  5. dma_request_chan() [if DMA needed]
```

---

## Part 1: GPIO Subsystem

### What Is GPIO?

GPIO = General Purpose Input/Output. A digital signal pin on the SoC that can be:
- **Input**: reads 0 or 1 from external hardware
- **Output**: drives 0 or 1 to external hardware

Every SoC has dozens to hundreds of GPIOs. They are organized into **banks** (usually 32 pins each).

```
Example: RK3588 GPIO banks
  GPIO0: pins 0-31   (GPIO0_A0 to GPIO0_D7)
  GPIO1: pins 0-31
  GPIO2: pins 0-31
  GPIO3: pins 0-31
  GPIO4: pins 0-31

GPIO4_B2 = bank 4, group B, pin 2
  → Linux GPIO number = 4*32 + 1*8 + 2 = 138
```

### GPIO Subsystem Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   GPIO Consumer                         │
│  (your driver: gpiod_get, gpiod_set_value, etc.)        │
└─────────────────────┬───────────────────────────────────┘
                      │ calls gpio_desc functions
                      ▼
┌─────────────────────────────────────────────────────────┐
│              gpiolib (drivers/gpio/gpiolib.c)            │
│  • Manages gpio_desc for every GPIO                     │
│  • Request/free tracking                                │
│  • Direction/value abstraction                          │
└─────────────────────┬───────────────────────────────────┘
                      │ calls gpiochip operations
                      ▼
┌─────────────────────────────────────────────────────────┐
│              GPIO Controller Driver                     │
│  (drivers/gpio/gpio-rockchip.c, gpio-pl061.c, etc.)     │
│  • Implements: get/set/direction/to_irq                 │
│  • Reads/writes hardware registers                      │
└─────────────────────────────────────────────────────────┘
```

### GPIO Controller Driver

```c
/* drivers/gpio/gpio-myvendor.c — implement a GPIO controller */
#include <linux/gpio/driver.h>

struct myvendor_gpio {
    struct gpio_chip    chip;
    void __iomem       *base;
    spinlock_t          lock;
};

/* Register layout */
#define GPIO_DATA_REG   0x00   /* data register: read/write pin values */
#define GPIO_DIR_REG    0x04   /* direction: 0=input, 1=output */

/* gpio_chip operations */
static int myvendor_gpio_get(struct gpio_chip *chip, unsigned int offset)
{
    struct myvendor_gpio *priv = gpiochip_get_data(chip);
    return !!(readl(priv->base + GPIO_DATA_REG) & BIT(offset));
}

static void myvendor_gpio_set(struct gpio_chip *chip, unsigned int offset,
                               int value)
{
    struct myvendor_gpio *priv = gpiochip_get_data(chip);
    unsigned long flags;
    u32 reg;

    spin_lock_irqsave(&priv->lock, flags);
    reg = readl(priv->base + GPIO_DATA_REG);
    if (value)
        reg |= BIT(offset);
    else
        reg &= ~BIT(offset);
    writel(reg, priv->base + GPIO_DATA_REG);
    spin_unlock_irqrestore(&priv->lock, flags);
}

static int myvendor_gpio_direction_input(struct gpio_chip *chip,
                                          unsigned int offset)
{
    struct myvendor_gpio *priv = gpiochip_get_data(chip);
    unsigned long flags;
    u32 reg;

    spin_lock_irqsave(&priv->lock, flags);
    reg = readl(priv->base + GPIO_DIR_REG);
    reg &= ~BIT(offset);   /* 0 = input */
    writel(reg, priv->base + GPIO_DIR_REG);
    spin_unlock_irqrestore(&priv->lock, flags);
    return 0;
}

static int myvendor_gpio_direction_output(struct gpio_chip *chip,
                                           unsigned int offset, int value)
{
    struct myvendor_gpio *priv = gpiochip_get_data(chip);
    unsigned long flags;
    u32 reg;

    /* Set value before direction (prevents glitch) */
    myvendor_gpio_set(chip, offset, value);

    spin_lock_irqsave(&priv->lock, flags);
    reg = readl(priv->base + GPIO_DIR_REG);
    reg |= BIT(offset);    /* 1 = output */
    writel(reg, priv->base + GPIO_DIR_REG);
    spin_unlock_irqrestore(&priv->lock, flags);
    return 0;
}

static int myvendor_gpio_probe(struct platform_device *pdev)
{
    struct myvendor_gpio *priv;
    int ret;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv) return -ENOMEM;
    spin_lock_init(&priv->lock);

    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base)) return PTR_ERR(priv->base);

    /* Initialize gpio_chip */
    priv->chip.label             = dev_name(&pdev->dev);
    priv->chip.parent            = &pdev->dev;
    priv->chip.owner             = THIS_MODULE;
    priv->chip.base              = -1;          /* dynamic assignment */
    priv->chip.ngpio             = 32;          /* 32 pins per bank */
    priv->chip.can_sleep         = false;       /* register access is fast */
    priv->chip.get               = myvendor_gpio_get;
    priv->chip.set               = myvendor_gpio_set;
    priv->chip.direction_input   = myvendor_gpio_direction_input;
    priv->chip.direction_output  = myvendor_gpio_direction_output;

    /* Register with gpiolib */
    ret = devm_gpiochip_add_data(&pdev->dev, &priv->chip, priv);
    if (ret) return ret;

    dev_info(&pdev->dev, "GPIO controller: %d pins at %p\n",
             priv->chip.ngpio, priv->base);
    return 0;
}
```

### GPIO Consumer (In Your Driver)

```c
/* How to USE GPIO from another driver */
#include <linux/gpio/consumer.h>

struct my_device {
    struct gpio_desc *reset_gpio;
    struct gpio_desc *irq_gpio;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_device *dev;

    /* Get GPIO from DT */
    /* DT: reset-gpios = <&gpio4 2 GPIO_ACTIVE_LOW>; */
    dev->reset_gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(dev->reset_gpio))
        return dev_err_probe(&pdev->dev, PTR_ERR(dev->reset_gpio),
                             "Failed to get reset GPIO\n");

    /* Optional GPIO */
    dev->irq_gpio = devm_gpiod_get_optional(&pdev->dev, "irq", GPIOD_IN);
    /* Returns NULL if not in DT (not an error) */

    /* Use GPIO */
    gpiod_set_value_cansleep(dev->reset_gpio, 1);  /* assert reset */
    mdelay(10);
    gpiod_set_value_cansleep(dev->reset_gpio, 0);  /* deassert reset */

    /* Read GPIO */
    int val = gpiod_get_value(dev->irq_gpio);
    dev_info(&pdev->dev, "IRQ pin state: %d\n", val);

    /* Get IRQ from GPIO */
    int irq = gpiod_to_irq(dev->irq_gpio);
    devm_request_irq(&pdev->dev, irq, my_irq_handler,
                     IRQF_TRIGGER_FALLING, "my-irq", dev);

    return 0;
}
```

```dts
/* Device Tree for GPIO consumer */
my_sensor: sensor@48 {
    compatible = "myvendor,my-sensor";
    reg = <0x48>;
    
    /* GPIO references: <phandle, pin_number, flags> */
    reset-gpios = <&gpio4 2 GPIO_ACTIVE_LOW>;   /* active low = assert=0 */
    irq-gpios   = <&gpio3 15 GPIO_ACTIVE_HIGH>;
};
```

---

## Part 2: Pinctrl Subsystem

### What Is Pinctrl?

GPIO handles on/off. Pinctrl handles **which function a pin serves**. 

Most SoC pins are multiplexed — the same physical pin can be:
- GPIO (generic I/O)
- UART TX
- I2C SDA
- SPI MISO
- PWM output

Pinctrl configures the **mux** and **electrical properties** (pull-up/down, drive strength, slew rate).

```
Physical Pin 42 on RK3588 can be:
  Function 0: GPIO3_A4    ← generic I/O
  Function 1: I2C3_SDA    ← I2C data line
  Function 2: SPI1_MISO   ← SPI data
  Function 3: UART3_RX    ← UART receive

Pinctrl selects which function is active.
```

### Pinctrl in Device Tree

```dts
/* Pin configuration defined in SoC DTSI (e.g., rk3588.dtsi) */
&pinctrl {
    i2c3 {
        /* State "default" when device is active */
        i2c3_pins: i2c3-default {
            rockchip,pins =
                <3 RK_PA4 2 &pcfg_pull_up>,  /* GPIO3_A4 → Function 2 (I2C SDA) */
                <3 RK_PA5 2 &pcfg_pull_up>;  /* GPIO3_A5 → Function 2 (I2C SCL) */
        };
        
        /* State "sleep" when device is suspended */
        i2c3_pins_sleep: i2c3-sleep {
            rockchip,pins =
                <3 RK_PA4 0 &pcfg_pull_none>, /* back to GPIO, no pull */
                <3 RK_PA5 0 &pcfg_pull_none>;
        };
    };
    
    uart3 {
        uart3_pins: uart3-default {
            rockchip,pins =
                <3 RK_PA4 3 &pcfg_pull_up>,  /* → Function 3 (UART RX) */
                <3 RK_PA5 3 &pcfg_pull_up>;
        };
    };
};

/* Consumer: reference pinctrl states in device node */
&i2c3 {
    status = "okay";
    pinctrl-names = "default", "sleep";   /* state names */
    pinctrl-0 = <&i2c3_pins>;            /* state 0 = "default" */
    pinctrl-1 = <&i2c3_pins_sleep>;      /* state 1 = "sleep" */
};
```

### How Pinctrl Auto-Applies Pins

```c
/* The kernel automatically applies pinctrl states at the right times */

/* In drivers/base/pinctrl.c: */
int pinctrl_bind_pins(struct device *dev)
{
    /* During device probe: */
    dev->pins->p = devm_pinctrl_get(dev);        /* get pin controller */
    dev->pins->default_state = pinctrl_lookup_state(dev->pins->p, "default");
    dev->pins->sleep_state = pinctrl_lookup_state(dev->pins->p, "sleep");
    
    /* Apply "default" state before probe() is called */
    pinctrl_select_state(dev->pins->p, dev->pins->default_state);
    /* Now your I2C pins are configured as I2C! */
}

/* During suspend: */
pinctrl_select_state(dev->pins->p, dev->pins->sleep_state);
/* Pins go to low-power configuration */
```

### Pinctrl Driver Implementation

```c
/* drivers/pinctrl/pinctrl-myvendor.c */
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinmux.h>
#include <linux/pinctrl/pinconf.h>

/* Describe all pins */
static const struct pinctrl_pin_desc myvendor_pins[] = {
    PINCTRL_PIN(0, "PIN_0"),
    PINCTRL_PIN(1, "PIN_1"),
    /* ... */
    PINCTRL_PIN(127, "PIN_127"),
};

/* Mux operations */
static int myvendor_get_functions_count(struct pinctrl_dev *pctldev)
{
    return ARRAY_SIZE(myvendor_functions);
}

static int myvendor_set_mux(struct pinctrl_dev *pctldev,
                             unsigned int func_selector,
                             unsigned int group_selector)
{
    /* Write to IOMUX registers to select function */
    u32 reg = myvendor_iomux_offset(group_selector);
    u32 val = myvendor_iomux_value(func_selector, group_selector);
    regmap_write(priv->regmap, reg, val);
    return 0;
}

/* Config operations */
static int myvendor_pin_config_set(struct pinctrl_dev *pctldev,
                                    unsigned int pin,
                                    unsigned long *configs,
                                    unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);
        
        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            set_pull(priv, pin, PULL_UP);
            break;
        case PIN_CONFIG_BIAS_PULL_DOWN:
            set_pull(priv, pin, PULL_DOWN);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            set_drive_strength(priv, pin, arg);
            break;
        case PIN_CONFIG_SLEW_RATE:
            set_slew_rate(priv, pin, arg);
            break;
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}
```

---

## Part 3: Clock Framework

### What Is a Clock?

Every digital circuit needs a timing reference — a clock signal. The SoC has:
- A crystal oscillator (e.g., 24MHz) as the primary source
- PLLs (Phase-Locked Loops) that multiply the crystal to GHz speeds
- Dividers and muxes that derive hundreds of clocks from the PLLs

```
Crystal (24MHz)
    │
    ├─→ PLL0 (APLL) → 2400MHz → CPU clock
    │                         → CPU divider (/1, /2, /4, ...) → CPU core clocks
    │
    ├─→ PLL1 (DPLL) → 800MHz → DDR clock
    │
    ├─→ PLL2 (CPLL) → 1000MHz → Peripheral clocks
    │                          → /5 → 200MHz → eMMC clock
    │                          → /10 → 100MHz → SPI clock
    │                          → /4 → 250MHz → PCIe clock
    │
    └─→ PLL3 (GPLL) → 1188MHz → GPU clock
```

### Clock Framework Architecture

```
Consumer Driver
   │ clk_get(), clk_prepare_enable(), clk_set_rate()
   ▼
Common Clock Framework (CCF) — drivers/clk/clk.c
   │ struct clk_hw operations
   ▼
Clock Driver (drivers/clk/rockchip/clk-rk3588.c)
   │ implements: enable/disable/set_rate/recalc_rate
   ▼
Hardware Registers (PLL, divider, mux, gate registers)
```

### Clock Types

```c
/* The CCF has these building blocks: */

/* 1. Fixed rate clock (crystal oscillator) */
/* DT: osc24m: oscillator { compatible = "fixed-clock"; clock-frequency = <24000000>; }; */
clk = clk_register_fixed_rate(dev, "osc24m", NULL, 0, 24000000);

/* 2. Gate clock (just an enable/disable switch) */
/* DT: uart_clk: clock { compatible = "tbd"; clocks = <&pclk>; clock-output-names = "uart_clk"; }; */
clk = clk_register_gate(dev, "uart_gate", "pclk", 0, base + GATE_REG, 3, 0, &lock);

/* 3. Divider clock (divides parent by N) */
clk = clk_register_divider(dev, "uart_div", "uart_gate", 0,
                            base + DIV_REG, 8, 4,   /* bit offset=8, width=4 */
                            CLK_DIVIDER_ONE_BASED, &lock);

/* 4. Mux clock (selects between multiple parents) */
const char *mux_parents[] = {"pll_a", "pll_b", "osc24m"};
clk = clk_register_mux(dev, "uart_mux", mux_parents, 3, 0,
                        base + MUX_REG, 4, 2, 0, &lock);

/* 5. PLL clock (complex, usually platform-specific) */
static const struct clk_ops rk3588_pll_ops = {
    .prepare    = rk3588_pll_enable,
    .unprepare  = rk3588_pll_disable,
    .recalc_rate = rk3588_pll_recalc_rate,
    .round_rate  = rk3588_pll_round_rate,
    .set_rate    = rk3588_pll_set_rate,
};
```

### Using Clocks in Your Driver

```c
#include <linux/clk.h>

struct my_driver_data {
    struct clk  *core_clk;
    struct clk  *bus_clk;
    struct clk  *ref_clk;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_driver_data *priv;
    int ret;

    /* Get clocks from DT */
    /* DT: clocks = <&cru CLK_MY_CORE>, <&cru PCLK_MY_BUS>; */
    /*     clock-names = "core", "bus"; */
    priv->core_clk = devm_clk_get(&pdev->dev, "core");
    if (IS_ERR(priv->core_clk))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->core_clk),
                             "Failed to get core clock\n");

    priv->bus_clk = devm_clk_get(&pdev->dev, "bus");
    if (IS_ERR(priv->bus_clk))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->bus_clk),
                             "Failed to get bus clock\n");

    /* Optional clock */
    priv->ref_clk = devm_clk_get_optional(&pdev->dev, "ref");

    /* Enable clocks (prepare = power up PLL; enable = ungate clock) */
    ret = clk_prepare_enable(priv->bus_clk);
    if (ret) return ret;

    ret = clk_prepare_enable(priv->core_clk);
    if (ret) {
        clk_disable_unprepare(priv->bus_clk);
        return ret;
    }

    /* Check/set frequency */
    unsigned long rate = clk_get_rate(priv->core_clk);
    dev_info(&pdev->dev, "core clock: %lu Hz\n", rate);

    /* Request specific rate */
    ret = clk_set_rate(priv->core_clk, 200000000);  /* 200 MHz */
    if (ret)
        dev_warn(&pdev->dev, "Can't set rate to 200MHz, got %lu\n",
                 clk_get_rate(priv->core_clk));

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    struct my_driver_data *priv = platform_get_drvdata(pdev);
    /* Disable in reverse order of enable */
    clk_disable_unprepare(priv->core_clk);
    clk_disable_unprepare(priv->bus_clk);
    /* devm_ handles devm_clk_get resources */
    return 0;
}
```

### Debug: Clock Tree

```bash
# View entire clock tree
cat /sys/kernel/debug/clk/clk_summary
# clock                        enable_cnt  prepare_cnt  rate           accuracy
# ─────────────────────────────────────────────────────────────────────────────
# osc24m                       0           0            24000000       0
#  apll                        0           0            2400000000     0
#   armclk                     2           2            1800000000     0
# cpll                         3           3            1000000000     0
#  pclk_bus                    5           5            100000000      0
#   pclk_uart3                 1           1            100000000      0

# Check specific clock
cat /sys/kernel/debug/clk/pclk_uart3/clk_rate   # 100000000

# Disable a clock temporarily (for testing)
echo 0 > /sys/kernel/debug/clk/pclk_uart3/clk_enable_count
```

---

## Part 4: Regulator Framework

### What Is a Regulator?

A regulator is a power supply circuit that provides a **stable, controlled voltage** to a hardware block. Different blocks need different voltages:

```
SoC power domains (RK3588 example):
  VDD_CPU_LIT:  0.75V - 1.0V  (little CPU cores)
  VDD_CPU_BIG0: 0.75V - 1.05V (big CPU cores 0-3)
  VDD_CPU_BIG1: 0.75V - 1.05V (big CPU cores 4-5)
  VDD_GPU:      0.75V - 0.95V (GPU Mali-G610)
  VDD_NPU:      0.75V - 0.95V (NPU 6TOPS)
  VCC_3V3:      3.3V           (UART, SPI, I2C)
  VCC_1V8:      1.8V           (eMMC, DDR signaling)
  VCC_0V9:      0.9V           (PCIe, USB3)

Voltage changes with frequency (DVFS):
  CPU at 1.8GHz: needs 1.0V
  CPU at 2.4GHz: needs 1.05V
  CPU at 600MHz: needs 0.75V (power saving)
```

### Regulator Framework Architecture

```
Consumer (CPU freq driver, GPU driver, your driver)
   │ regulator_get(), regulator_enable(), regulator_set_voltage()
   ▼
Regulator Core (drivers/regulator/core.c)
   │ struct regulator_ops
   ▼
Regulator Driver (PMIC driver)
   e.g., drivers/regulator/rk808-regulator.c (Rockchip RK806 PMIC)
   │ implements enable/disable/set_voltage via I2C writes to PMIC
   ▼
PMIC Hardware (I2C device)
```

### Regulator Driver (PMIC driver skeleton)

```c
/* drivers/regulator/myvendor-pmic.c */
#include <linux/regulator/driver.h>
#include <linux/regulator/machine.h>
#include <linux/i2c.h>

/* Voltage table: register value → millivolts */
static const unsigned int ldo1_volt_table[] = {
    700000, 725000, 750000, 775000,   /* 700mV ... 1100mV */
    800000, 825000, 850000, 875000,
    900000, 925000, 950000, 975000,
    1000000, 1025000, 1050000, 1100000,
};

static const struct regulator_ops ldo_ops = {
    .list_voltage   = regulator_list_voltage_table,
    .map_voltage    = regulator_map_voltage_ascend,
    .get_voltage_sel = regulator_get_voltage_sel_regmap,
    .set_voltage_sel = regulator_set_voltage_sel_regmap,
    .enable         = regulator_enable_regmap,
    .disable        = regulator_disable_regmap,
    .is_enabled     = regulator_is_enabled_regmap,
};

/* Describe each regulator in the PMIC */
static const struct regulator_desc myvendor_regs[] = {
    {
        .name           = "LDO1",
        .of_match       = "LDO1",
        .regulators_node = "regulators",
        .ops            = &ldo_ops,
        .type           = REGULATOR_VOLTAGE,
        .volt_table     = ldo1_volt_table,
        .n_voltages     = ARRAY_SIZE(ldo1_volt_table),
        .enable_reg     = MYVENDOR_LDO1_EN_REG,
        .enable_mask    = BIT(0),
        .vsel_reg       = MYVENDOR_LDO1_VSEL_REG,
        .vsel_mask      = 0x0F,
        .owner          = THIS_MODULE,
    },
    /* LDO2, LDO3, BUCK1, BUCK2 ... */
};
```

### Using Regulators in Your Driver

```c
#include <linux/regulator/consumer.h>

struct my_device {
    struct regulator *vdd;     /* power supply */
    struct regulator *vio;     /* I/O power supply */
};

static int my_probe(struct platform_device *pdev)
{
    /* DT: vdd-supply = <&ldo1>; */
    dev->vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(dev->vdd))
        return dev_err_probe(&pdev->dev, PTR_ERR(dev->vdd),
                             "Failed to get VDD supply\n");

    /* Optional supply */
    dev->vio = devm_regulator_get_optional(&pdev->dev, "vio");
    if (IS_ERR(dev->vio))
        dev->vio = NULL;   /* not critical */

    /* Set voltage range [min, max] in microvolts */
    int ret = regulator_set_voltage(dev->vdd, 900000, 1100000);  /* 0.9V-1.1V */
    if (ret)
        return ret;

    /* Enable power */
    ret = regulator_enable(dev->vdd);
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "Failed to enable VDD\n");

    /* Allow hardware to stabilize after power-on */
    usleep_range(1000, 2000);   /* 1-2ms */

    if (dev->vio) {
        regulator_enable(dev->vio);
        usleep_range(100, 200);
    }

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);
    
    if (dev->vio)
        regulator_disable(dev->vio);
    regulator_disable(dev->vdd);
    return 0;
}
```

---

## Part 5: DMA Engine Framework

### What Is the DMA Engine Framework?

The DMA Engine framework provides a **unified API** for all DMA controllers. Without it, every driver would need to know the specific DMA hardware. With it, drivers just call standard functions.

```
Your Driver
   │ dma_request_chan(), dmaengine_prep_dma_memcpy()
   ▼
DMA Engine Core (drivers/dma/dmaengine.c)
   │ struct dma_device operations
   ▼
DMA Controller Driver (e.g., drivers/dma/pl330.c for ARM PL330)
   │ programs DMA controller registers
   ▼
DMA Hardware (PL330, XDMA, RK DMA, etc.)
```

### DMA Transaction Types

```c
/* The DMA Engine supports these transfer types: */

/* 1. memcpy: memory → memory */
dmaengine_prep_dma_memcpy(chan, dst_addr, src_addr, len, flags);

/* 2. memset: fill memory with pattern */
dmaengine_prep_dma_memset(chan, dst_addr, value, len, flags);

/* 3. slave_sg: scatter-gather for device I/O */
/* Used for: DMA from peripheral (UART/SPI/I2C FIFO) to scatter list */
dmaengine_prep_slave_sg(chan, sgl, sg_len, DMA_FROM_DEVICE, flags);

/* 4. cyclic: ring buffer (audio, video) */
dmaengine_prep_dma_cyclic(chan, buf_addr, buf_len, period_len,
                           DMA_FROM_DEVICE, flags);
/* Callback fired every 'period_len' bytes — perfect for audio */
```

### Complete DMA Slave Example (UART RX via DMA)

```c
/* DMA-driven UART receive — used in uart-pl011.c and others */
struct uart_dma_rx {
    struct dma_chan         *chan;
    struct dma_async_tx_descriptor *desc;
    dma_addr_t              buf_dma;
    void                    *buf;
    size_t                  buf_size;
    size_t                  period_size;
    struct completion       completion;
};

static int uart_dma_rx_setup(struct uart_dma_rx *rx, struct device *dev,
                              size_t buf_size, size_t period_size)
{
    struct dma_slave_config cfg = {};
    int ret;

    /* Request DMA channel (from DT: dmas = <&dma 2>; dma-names = "rx"; ) */
    rx->chan = dma_request_chan(dev, "rx");
    if (IS_ERR(rx->chan))
        return PTR_ERR(rx->chan);

    /* Allocate coherent DMA buffer */
    rx->buf = dma_alloc_coherent(dev, buf_size, &rx->buf_dma, GFP_KERNEL);
    if (!rx->buf) {
        dma_release_channel(rx->chan);
        return -ENOMEM;
    }
    rx->buf_size = buf_size;
    rx->period_size = period_size;
    init_completion(&rx->completion);

    /* Configure DMA channel for peripheral slave mode */
    cfg.direction = DMA_DEV_TO_MEM;         /* peripheral → memory */
    cfg.src_addr = UART_RX_FIFO_ADDR;       /* UART RX FIFO address */
    cfg.src_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;  /* 8-bit FIFO */
    cfg.src_maxburst = 4;                    /* burst 4 bytes */
    ret = dmaengine_slave_config(rx->chan, &cfg);
    if (ret) goto err;

    /* Prepare cyclic transfer: buffer wraps around */
    /* Callback fired every period_size bytes received */
    rx->desc = dmaengine_prep_dma_cyclic(
        rx->chan,
        rx->buf_dma,          /* DMA address of receive buffer */
        buf_size,             /* total buffer size */
        period_size,          /* callback every period_size bytes */
        DMA_DEV_TO_MEM,
        DMA_PREP_INTERRUPT
    );
    if (!rx->desc) { ret = -ENOMEM; goto err; }

    rx->desc->callback = uart_dma_rx_callback;
    rx->desc->callback_param = rx;

    /* Submit and start */
    dmaengine_submit(rx->desc);
    dma_async_issue_pending(rx->chan);

    return 0;
err:
    dma_free_coherent(dev, buf_size, rx->buf, rx->buf_dma);
    dma_release_channel(rx->chan);
    return ret;
}

static void uart_dma_rx_callback(void *data)
{
    struct uart_dma_rx *rx = data;
    struct dma_tx_state state;
    size_t residue, received;

    /* How much data has been received? */
    dmaengine_tx_status(rx->chan, rx->desc->cookie, &state);
    residue = state.residue;
    received = rx->buf_size - residue;

    /* Process received data */
    process_uart_data(rx->buf, received);
}
```

---

## Putting It All Together: The Complete Driver Init Sequence

```c
/* Correct order for driver probe() with all 5 frameworks */
static int complete_driver_probe(struct platform_device *pdev)
{
    struct driver_priv *priv;
    int ret;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv) return -ENOMEM;

    /* 1. Get MMIO registers */
    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base)) return PTR_ERR(priv->base);

    /* 2. Get power supply (Regulator) */
    priv->vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(priv->vdd))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->vdd), "No VDD\n");

    /* 3. Get clocks */
    priv->clk = devm_clk_get(&pdev->dev, "core");
    if (IS_ERR(priv->clk))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->clk), "No clock\n");

    /* 4. Get GPIO (Pinctrl is applied automatically before probe) */
    priv->reset = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(priv->reset))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->reset), "No reset GPIO\n");

    /* 5. Get DMA channel */
    priv->dma_chan = dma_request_chan(&pdev->dev, "tx");
    if (IS_ERR(priv->dma_chan))
        return dev_err_probe(&pdev->dev, PTR_ERR(priv->dma_chan), "No DMA\n");

    /* === Hardware initialization (in correct power-up order) === */

    /* Power on */
    ret = regulator_enable(priv->vdd);
    if (ret) return ret;
    usleep_range(1000, 2000);   /* power stabilization */

    /* Enable clock */
    ret = clk_prepare_enable(priv->clk);
    if (ret) { regulator_disable(priv->vdd); return ret; }

    /* Release reset (deassert = bring hardware out of reset) */
    gpiod_set_value_cansleep(priv->reset, 0);
    usleep_range(100, 200);

    /* Get IRQ */
    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0) { /* cleanup and return */ }
    
    ret = devm_request_irq(&pdev->dev, priv->irq, my_irq_handler,
                           0, dev_name(&pdev->dev), priv);
    if (ret) { /* cleanup and return */ }

    platform_set_drvdata(pdev, priv);
    dev_info(&pdev->dev, "probe succeeded\n");
    return 0;
}
```

---

## Interview Questions

| Level | Question | Key Answer |
|-------|----------|-----------|
| **Beginner** | What is GPIO? Give an example use case | General Purpose I/O; example: LED, button, reset signal |
| **Beginner** | What is pin multiplexing? | Same physical pin can be UART/I2C/GPIO/SPI; Pinctrl selects which |
| **Intermediate** | What is `clk_prepare_enable` vs `clk_enable` alone? | prepare = power up PLL (may sleep); enable = ungate (fast, no sleep); must call both |
| **Intermediate** | What does `devm_regulator_get_optional` return if the DT property is missing? | Returns NULL pointer (not error), indicating the supply is absent but acceptable |
| **Intermediate** | What is DVFS and which frameworks does it use? | Dynamic Voltage/Frequency Scaling; uses CPUFreq (frequency) + Regulator (voltage) together |
| **Advanced** | Explain the difference between DMA_DEV_TO_MEM and DMA_MEM_TO_DEV | DEV_TO_MEM = peripheral (UART/SPI FIFO) → RAM (receive); MEM_TO_DEV = RAM → peripheral (transmit) |
| **Advanced** | How does the pinctrl "default" state get applied? By whom, when? | `pinctrl_bind_pins()` in drivers/base/pinctrl.c, called before driver probe() automatically |
| **Expert** | How would you debug "I can see my driver probe but hardware doesn't respond"? | Check 1) clocks enabled (`clk_summary`), 2) regulators on (`regulator_summary`), 3) pinctrl correct (`pinctrl` debug), 4) register writes valid (logic analyzer) |
