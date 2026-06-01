# Linux Disk IOPS & Performance Measurement Commands

A reference guide for diagnosing slow disks and measuring IOPS on Linux using native and installable CLI tools.

***

## 1. `iostat` — Primary IOPS Monitor

Part of the `sysstat` package. The most widely used tool for per-device IOPS, throughput, and utilization stats.

**Install:**
```bash
sudo apt install sysstat      # Debian/Ubuntu
sudo yum install sysstat      # RHEL/CentOS/Fedora
```

**Examples:**
```bash
# Basic disk stats, refresh every 2 seconds
iostat -d 2

# Extended stats with KB units, refresh every 2 seconds
iostat -d -x -k 2

# Target a specific device
iostat -d -x -k /dev/sda 2

# Include CPU wait stats alongside disk stats
iostat -c -d -x 2
```

## List disk
```
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

**Key columns to watch:**

| Column  | Meaning                                          | Threshold to Investigate   |
|---------|--------------------------------------------------|----------------------------|
| `r/s`   | Read IOPS (requests per second)                  | Context-dependent          |
| `w/s`   | Write IOPS (requests per second)                 | Context-dependent          |
| `rkB/s` | Read throughput (KB/s)                           | Context-dependent          |
| `wkB/s` | Write throughput (KB/s)                          | Context-dependent          |
| `await` | Average I/O wait time in milliseconds            | > 10ms (SSD), > 20ms (HDD)|
| `svctm` | Average service time per I/O in ms               | High = slow device         |
| `%util` | Percentage of time device is busy                | > 80% = approaching saturation |

***

## 2. `iotop` — Per-Process Disk Activity

Like `top` but for disk I/O. Identifies *which process* is responsible for disk load.

**Install:**
```bash
sudo apt install iotop         # Debian/Ubuntu
sudo yum install iotop         # RHEL/CentOS
```

**Examples:**
```bash
# Interactive mode — show only active I/O processes
sudo iotop -o

# Non-interactive snapshot (useful for scripting)
sudo iotop -b -n 3

# Show processes only (not threads), accumulated I/O, active only
sudo iotop -oPa

# Watch a specific PID
sudo iotop -p 1234
```

**Key columns:**

| Column       | Meaning                              |
|--------------|--------------------------------------|
| `DISK READ`  | Current read rate for the process    |
| `DISK WRITE` | Current write rate for the process   |
| `SWAPIN`     | Time waiting for swap I/O            |
| `IO>`        | Percentage of time in I/O wait       |

***

## 3. `ioping` — Disk Latency & Seek Rate

Measures disk **latency** specifically — the best tool for diagnosing sluggish response times on HDDs or degraded SSDs.

**Install:**
```bash
sudo apt install ioping
```

**Examples:**
```bash
# Send 20 I/O requests to a device, show per-op latency
ioping -c 20 /dev/sda

# Test a directory/mount point instead
ioping -c 10 /var/log

# Disk seek rate benchmark
ioping -R /dev/sda

# Sequential I/O benchmark
ioping -RL /dev/sda

# Set custom block size (4K is typical for DB workloads)
ioping -c 20 -s 4k /dev/sda
```

**Sample output explained:**
```
4 KiB <<< /dev/sda (block device 931.5 GiB): request=1 time=1.23 ms
```
- `time` is the round-trip latency per request — anything consistently > 10ms on an SSD or > 20ms on an HDD is a red flag.

***

## 4. `dstat` — Combined Real-Time Dashboard

Combines disk, CPU, network, and memory stats in a single view. Useful for correlating I/O wait (`wa`) spikes with disk activity.

**Install:**
```bash
sudo apt install dstat
```

**Examples:**
```bash
# Disk read/write stats only, every 2 seconds
dstat -d 2

# Disk + CPU + memory combined (default view)
dstat 2

# Show disk stats by device name
dstat -D sda,sdb 2

# Full stats with timestamps (useful for logging)
dstat --time --cpu --disk --mem --net 2
```

***

## 5. `top` / `htop` — I/O Wait Signal

While not disk-specific, the `wa` (I/O wait) percentage in `top` is a quick first signal that storage is a bottleneck.

**Examples:**
```bash
# Standard top — watch '%wa' in the CPU line at the top
top

# htop — more visual, same concept
htop
# Press F2 > Columns > add IO_READ_RATE, IO_WRITE_RATE
```

**Rule of thumb:** If `%wa` is consistently above **5%**, disk I/O is likely throttling CPU performance.

***

## 6. `atop` — Historical & Detailed Per-Device Stats

More detailed than `top`, includes per-device disk stats and can log history for post-mortem analysis.

**Install:**
```bash
sudo apt install atop
```

**Examples:**
```bash
# Launch atop — press 'd' to switch to disk view
atop

# Non-interactive: show only disk stats
atop | grep DSK

# Log every 10 seconds to a file for later review
sudo atop -w /tmp/atop.log 10

# Replay a previous log
atop -r /tmp/atop.log
```

**Key DSK columns:** `busy%` (utilization), `read` (IOPS), `write` (IOPS), `avio` (average I/O ms).

***

## 7. `/proc/diskstats` — Raw Kernel Counters

The raw source of truth that tools like `iostat` read from. Useful in scripts or when no external tools are installed.

**Examples:**
```bash
# View raw disk stats
cat /proc/diskstats

# Watch changes in real time (sda only)
watch -n 1 "grep sda /proc/diskstats"

# Quick IOPS calculation: field 4 = reads completed, field 8 = writes completed
awk '$3=="sda" {print "reads:", $4, "writes:", $8}' /proc/diskstats
```

**Field reference (columns 4–11):**
- Field 4: reads completed
- Field 8: writes completed
- Field 13: I/Os currently in progress
- Field 14: time spent doing I/Os (ms)

***

## 8. `vmstat` — Block I/O & CPU Wait

Shows system-wide block I/O operations and CPU wait time in a single compact line.

**Examples:**
```bash
# Refresh every 2 seconds
vmstat 2

# Display timestamps with output
vmstat -t 2

# Display in MB instead of KB
vmstat -S m 2
```

**Key columns:**

| Column | Meaning                                    |
|--------|--------------------------------------------|
| `bi`   | Blocks received from a block device (read) |
| `bo`   | Blocks sent to a block device (write)      |
| `wa`   | CPU time waiting for I/O (%)               |

***

## 9. `fio` — Full IOPS Benchmark

The industry standard for synthetic benchmarking. Measures the maximum IOPS a device can sustain under various workload patterns.

**Install:**
```bash
sudo apt install fio
```

**Examples:**
```bash
# Random read IOPS (4K block — typical DB/VM workload)
fio --name=randread \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --filename=/dev/sda \
    --group_reporting

# Random write IOPS
fio --name=randwrite \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randwrite \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --filename=/tmp/testfile \
    --group_reporting

# Sequential read throughput
fio --name=seqread \
    --ioengine=libaio \
    --rw=read \
    --bs=1M \
    --direct=1 \
    --size=4G \
    --filename=/tmp/testfile \
    --group_reporting

# Mixed 70% read / 30% write (realistic app workload)
fio --name=mixed \
    --ioengine=libaio \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --filename=/tmp/testfile \
    --group_reporting
```

> **Warning:** When targeting a raw device (`/dev/sda`) with `fio`, all data on that device will be overwritten. Use a test file on a mounted filesystem for safety.

***

## 10. `dd` — Quick Sequential Throughput Test

Built into every Linux system. No install required. Tests raw sequential read/write speed. Always use `direct` flags to bypass page cache.

**Examples:**
```bash
# Sequential write test (1 GB file, 1M block size)
dd if=/dev/zero of=/tmp/testfile bs=1M count=1024 oflag=direct

# Sequential read test
dd if=/tmp/testfile of=/dev/null bs=1M iflag=direct

# Write with progress display (Linux 3.3+)
dd if=/dev/zero of=/tmp/testfile bs=1M count=1024 oflag=direct status=progress

# Cleanup test file
rm /tmp/testfile
```

> **Note:** `dd` only tests *sequential* I/O and does **not** measure IOPS directly. Use `fio` for random I/O patterns.

***

## Recommended Diagnostic Workflow

```
1. Run: iostat -d -x -k 2
   -> Check %util (> 80%) and await (> 10ms SSD / > 20ms HDD)

2. Run: sudo iotop -o
   -> Identify which process is causing the I/O load

3. Run: ioping -c 20 /dev/sda
   -> Measure raw disk latency

4. Run: fio (randread/randwrite test)
   -> Benchmark peak IOPS to see how far device is from its ceiling
```

***

## IOPS Reference Ranges

| Disk Type       | Typical IOPS         | Good Latency    | `%util` Safe Zone |
|-----------------|----------------------|-----------------|-------------------|
| HDD (spinning)  | 100 – 200            | < 20 ms         | < 70%             |
| SATA SSD        | 10,000 – 50,000      | < 10 ms         | < 80%             |
| NVMe SSD        | 100,000 – 1,000,000+ | < 1 ms          | < 80%             |
| Cloud EBS (AWS) | Varies by tier       | < 5 ms typical  | < 80%             |
