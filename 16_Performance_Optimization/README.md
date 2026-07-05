# 16 — Performance Optimization

> Performance analysis for embedded Linux: CPU profiling, I/O bottlenecks, memory bandwidth, boot time, and power optimization.

---

## Section Structure

```
16_Performance_Optimization/
├── 01_Perf_Tool_Mastery.md           ← perf stat, perf record, perf report
├── 02_Flamegraphs.md                 ← Brendan Gregg's flamegraphs for kernel/user
├── 03_Boot_Time_Optimization.md      ← bootchart, systemd-analyze, initcall debug
├── 04_Memory_Bandwidth.md            ← stream benchmark, cache analysis
├── 05_IO_Performance.md              ← fio, iostat, blktrace for storage
├── 06_CPU_Frequency_Scaling.md       ← cpufreq, thermal throttling, big.LITTLE
├── 07_Power_Optimization.md          ← powertop, suspend/resume, idle states
└── 08_Tracing_With_eBPF.md           ← bpftrace, bcc-tools for performance
```

---

## perf Quick Reference

```bash
# CPU-wide statistics (1 second sample)
perf stat -a sleep 1

# Profile a specific process
perf record -g -p <PID> sleep 10
perf report --stdio

# System-wide call graph
perf record -ag sleep 5
perf report -g

# Count specific events
perf stat -e cache-misses,cache-references,instructions,cycles ./my_app

# Live top (like htop but for perf events)
perf top -g

# Kernel function call frequency
perf stat -e 'irq:*' sleep 1
```

---

## Boot Time Analysis

```bash
# systemd boot breakdown
systemd-analyze
systemd-analyze blame                     # time per service
systemd-analyze critical-chain           # longest dependency chain
systemd-analyze plot > boot.svg           # visual timeline

# Kernel initcall timing
# Add to kernel cmdline: initcall_debug
dmesg | grep "initcall\|calling\|initcall_debug" | head -30

# bootchart (full visual)
apt install bootchart
bootchartd start
# reboot with kernel arg: init=/sbin/bootchartd

# Quick boot time baseline
systemd-analyze | head -1
# "Startup finished in 2.314s (kernel) + 8.543s (userspace) = 10.858s"
```

---

## Flamegraph Generation

```bash
# Install flamegraph scripts
git clone https://github.com/brendangregg/FlameGraph.git

# Record with stacks
perf record -ag -F 99 sleep 30

# Generate flamegraph
perf script | ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > flame.svg

# Open in browser
firefox flame.svg
```

---

## I/O Performance (fio)

```bash
# Sequential read test
fio --name=seqread --rw=read --bs=1m --size=1g \
    --filename=/dev/mmcblk0 --direct=1

# Random 4K read/write (typical storage benchmark)
fio --name=randtest --rw=randrw --rwmixread=70 \
    --bs=4k --size=512m --filename=/tmp/testfile \
    --direct=1 --numjobs=4 --runtime=60

# UFS 4.0 sequential write (relevant to your DW-UFS project)
fio --name=ufs_seq_write --rw=write --bs=128k \
    --size=4g --filename=/dev/sda --direct=1 \
    --iodepth=32 --numjobs=1
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What does `perf stat` measure? What is CPI (cycles per instruction)? |
| **Intermediate** | How do you find which kernel function is consuming most CPU time? |
| **Advanced** | How do you optimize boot time by 30%? Name at least 5 specific techniques. |
| **Expert** | Explain cache coherency overhead in a big.LITTLE system. How does it affect your driver design? |
