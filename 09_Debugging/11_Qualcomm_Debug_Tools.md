# 11 — Qualcomm Debug Tools

> Cross-reference: See `13_Qualcomm_Debugging/README.md` for the complete deep-dive.  
> This file provides a quick-access cheatsheet for Qualcomm-specific debug tools used alongside the generic kernel debug techniques in this section.

---

## Tools at a Glance

| Tool | Purpose | Access |
|------|---------|--------|
| **QXDM / QCAT** | Real-time Diag protocol log streaming | Proprietary (Windows) |
| **Ramdump** | Full DDR dump on crash | `/dev/ramdump*` |
| **Minidump** | Selective crash dump (lightweight) | `CONFIG_QCOM_MINIDUMP` |
| **CoreSight ETM** | ARM instruction trace hardware | `/sys/bus/coresight/` |
| **Trace32 (T32)** | JTAG hardware debugger | Lauterbach probe |
| **TZDBG** | TrustZone / QTEE log access | `/sys/kernel/debug/tzdbg/` |
| **cbmem** | Coreboot console log | `cbmem -c` |
| **SMEM** | Subsystem shared memory state | debugfs / remoteproc |
| **adb bugreport** | Full ChromeOS/Android system snapshot | ADB over USB |

---

## 5-Minute Debug Runbook

### Boot Hang (No Serial Output)
```bash
# 1. Connect Trace32 JTAG probe
# 2. Halt CPU: SYStem.Up in T32
# 3. Check PC register — tells you exactly where it's stuck
# 4. Check if DDR init completed: read 0x80000000, should be non-zero
# 5. Check QSPI/eMMC init: look at boot media controller registers
```

### Kernel Panic on SC7180/SC7280
```bash
# 1. Collect ramdump (automatic if CONFIG_QCOM_MEMORY_DUMP_V2=y)
ls /dev/ramdump_apss

# 2. Parse with crash tool
crash vmlinux /dev/ramdump_apss
crash> bt          # backtrace
crash> log         # kernel message buffer
crash> dis <addr>  # disassemble crash location

# 3. Alternatively: read /proc/last_kmsg after reboot
cat /proc/last_kmsg | grep -A10 "Oops\|panic\|BUG"
```

### TrustZone / QTEE Issue
```bash
# Check TZ log
cat /sys/kernel/debug/tzdbg/log
cat /sys/kernel/debug/tzdbg/qsee_log

# Check SMC call return codes in dmesg
dmesg | grep -i "scm\|qtee\|qseecom\|trustzone"

# If getting -EPERM on SMC calls:
# → fuse configuration may not allow this command
# → check QFPROM secure fuse status
```

### Modem/ADSP Subsystem Crash
```bash
# Watch for crash notification
dmesg -w | grep -i "mpss\|adsp\|subsystem\|remoteproc"

# Collect subsystem ramdump
cat /dev/ramdump_modem > mpss_dump.bin    # MPSS
cat /dev/ramdump_adsp > adsp_dump.bin     # ADSP

# Check subsystem state
cat /sys/class/remoteproc/remoteproc*/state

# Restart subsystem manually (if supported)
echo restart > /sys/class/remoteproc/remoteproc0/state
```

---

## CoreSight Quick Setup

```bash
# Verify CoreSight devices present
ls /sys/bus/coresight/devices/

# Enable ETM trace on CPU0
cd /sys/bus/coresight/devices/etm0
echo 1 > enable_source

# Collect trace from ETB
dd if=/dev/tmc_etf0 of=/tmp/trace.bin bs=4k count=16

# Stop tracing
echo 0 > /sys/bus/coresight/devices/etm0/enable_source
```

---

## Reference
→ **Full coverage:** [13_Qualcomm_Debugging/README.md](../13_Qualcomm_Debugging/README.md)
