# 15 — UART Logging & Serial Debug

> Serial console is the most reliable debug channel. It works before the kernel boots, through panics, and even over network. Master it.

---

## Section Structure

```
15_UART_Logging/
├── 01_Serial_Console_Setup.md        ← minicom, tio, picocom, cu setup
├── 02_Early_Printk.md                ← earlycon, earlyprintk, UART before kernel
├── 03_Kgdb_Over_Serial.md            ← Remote kernel debugging via serial
├── 04_USB_Serial_Adapters.md         ← CP2102, CH340, FTDI — choosing the right adapter
├── 05_Logformat_Mastery.md           ← dmesg formatting, kernel log levels, dev_dbg
└── 06_Embedded_Log_Analysis.md       ← Parsing logs with grep/awk/Python
```

---

## Terminal Tool Comparison

| Tool | Install | Best For |
|------|---------|---------|
| `tio` | `apt install tio` | Modern, clean, auto-reconnect |
| `minicom` | `apt install minicom` | Traditional, full-featured |
| `picocom` | `apt install picocom` | Lightweight, scriptable |
| `screen` | built-in | Quick serial access |
| `cu` | `apt install cu` | POSIX standard |

```bash
# tio — recommended
tio -b 115200 /dev/ttyUSB0
tio -b 115200 -d 8 -p none -s 1 /dev/ttyUSB0  # explicit params

# minicom
minicom -b 115200 -D /dev/ttyUSB0
# Ctrl+A Z = menu, Ctrl+A X = exit

# picocom
picocom -b 115200 /dev/ttyUSB0
# Ctrl+A Ctrl+X = exit

# Identify USB-serial adapter
dmesg | grep -i "tty\|usb.*serial\|cp210\|ch341\|ftdi"
ls /dev/ttyUSB* /dev/ttyACM*
```

---

## earlycon — Debug Before Driver Init

Add to kernel command line for serial output before UART driver probes:

```bash
# ARM64 UART (PL011)
earlycon=pl011,0xfe890000,115200

# Qualcomm UART (GENI)
earlycon=qcom_geni,0x00a90000,115200

# Generic (uses DT uart-clock-frequency)
earlycon

# In /boot/extlinux/extlinux.conf or U-Boot bootargs:
setenv bootargs "console=ttyS0,115200 earlycon=uart8250,mmio,0xfe215040"
```

---

## Kernel Log Levels

```c
/* In driver code */
dev_err(dev, "Fatal error: %d\n", ret);      /* Level 3 — always shown */
dev_warn(dev, "Retrying...\n");               /* Level 4 */
dev_info(dev, "Probe successful\n");          /* Level 6 */
dev_dbg(dev, "Register value: 0x%x\n", val); /* Level 7 — only with debug */

/* Enable dynamic debug for a module at runtime */
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

/* Set console log level */
echo 8 > /proc/sys/kernel/printk   /* show all levels */
dmesg -n 8                          /* same via dmesg */
```

---

## Log Parsing One-Liners

```bash
# Filter kernel errors only
dmesg --level=err,crit,alert,emerg

# Timestamps in human-readable format
dmesg -T

# Watch live kernel log
dmesg -w

# Filter for your driver
dmesg | grep -i "my_driver\|probe\|error"

# Extract timing from boot log
dmesg -T | awk '/\[.*\]/ {print $1, $2}' | head -50

# Find all probe failures
dmesg | grep "probe.*failed\|ENODEV\|EPROBE_DEFER"
```
