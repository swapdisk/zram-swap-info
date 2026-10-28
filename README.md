# Memory Architecture & Swap Optimization Guide

**Platform:** Lenovo ThinkPad X1 Carbon Gen 12 (32 GB Soldered LPDDR5x)  
**OS:** Fedora Linux (Btrfs on top of LUKS2 Encryption)  
**Objective:** Eliminate memory ceiling limits, maximize active RAM efficiency, and maintain smooth system responsiveness under heavy workloads without risk of OOM (Out-Of-Memory) crashes.

---

## 1. High-Level Architecture Overview

Modern high-performance Linux memory management separates swapping into two distinct tiers: **compressed memory (zram)** and **non-volatile fallback storage (SSD swapfile)**.

```text
[ Physical LPDDR5x RAM: 32 GB ]
 ├── Active Anonymous Memory & File Buffers / Caches
 └── Tier 1: /dev/zram0 (RAM Block Device)
      ├── Size: 24 GiB (ram * 0.75)
      ├── Priority: 100
      └── Compression: zstd (nanosecond latency)
            │
            │ (Spillover when zram is full)
            ▼
[ NVMe SSD (Encrypted LUKS2 Container) ]
 └── Tier 2: /swap/swapfile (Btrfs Dedicated Subvolume)
      ├── Size: 32 GiB
      ├── Priority: 10
      └── Attr: No-COW, Contiguous, 0600 permissions
```

### The Two-Tier Strategy

1. **Tier 1 — High-Priority Compressed In-RAM Swap (`/dev/zram0`, `pri=100`):**
   * Dynamically compresses cold, inactive application pages and stores them in RAM.
   * Compressing/decompressing memory with modern multi-core CPUs requires nanoseconds, preventing the severe I/O stalls associated with physical drive swapping.
   * At typical compression ratios (~2.5x to 3.0x), a 24 GiB virtual block device can hold ~60–70 GiB of uncompressed idle application pages.
2. **Tier 2 — Low-Priority Overflow Safety Net (`/swap/swapfile`, `pri=10`):**
   * Placed on the internal NVMe drive inside the encrypted LUKS container.
   * Strictly acts as an emergency pressure valve. Linux only writes to this device if the entire 24 GiB zram buffer is exhausted, preventing `systemd-oomd` or the kernel OOM killer from killing processes.

---

## 2. Component Implementation & Setup

### A. zram Configuration
Fedora manages zram devices using `systemd-zram-generator`.

* **Config File:** `/etc/systemd/zram-generator.conf.d/override.conf`
  ```ini
  [zram0]
  zram-size = ram * 0.75
  compression-algorithm = zstd
  ```
* **Rationale:**
  * `zram-size = ram * 0.75`: Creates a 24 GiB virtual swap device. Setting it to 75% of RAM guarantees that at least 8 GiB of physical memory remains exclusively available for page tables, kernel allocations, and uncompressed working sets.
  * `compression-algorithm = zstd`: Replaces the default `lzo-rle`. Zstandard achieves significantly higher compression ratios on typical mixed workloads while remaining virtually unnoticeable on modern Intel Core Ultra processors.
* **Activation Commands:**
  ```bash
  sudo swapoff /dev/zram0
  sudo systemctl restart systemd-zram-setup@zram0.service
  ```

---

### B. Btrfs Dedicated Swapfile on NVMe (LUKS2)
Btrfs requires specialized handling for swapfiles: Copy-on-Write (CoW), filesystem-level compression, and snapshotting must be completely disabled for the swap file to ensure direct, contiguous sector writes.

1. **Dedicated Subvolume Creation:**
   ```bash
   sudo btrfs subvolume create /swap
   ```
   * *Rationale:* Placing the swapfile inside its own subvolume ensures that root filesystem snapshots (e.g., via Snapper or Timeshift) do not capture the massive, rapidly changing swapfile.

2. **File Allocation:**
   ```bash
   sudo btrfs filesystem mkswapfile --size 32g /swap/swapfile
   sudo chmod 600 /swap/swapfile
   ```
   * *Rationale:* `btrfs filesystem mkswapfile` automatically sets the `+C` (nodatacow) attribute, turns off file compression, preallocates contiguous disk blocks, and formats the swap signature in a single safe operation. Permissions are restricted to `0600` for security.

3. **Persistent Mount (`/etc/fstab`):**
   ```text
   /swap/swapfile none swap defaults,pri=10 0 0
   ```
   * *Rationale:* Explicitly defining `pri=10` ensures it has a lower priority than `/dev/zram0` (which defaults to `pri=100`). The kernel will never write a single byte to the SSD until zram is 100% full. Because `/swap` lives inside the mapped LUKS partition (`luks-...`), swapped data is automatically encrypted at rest.

4. **Activation:**
   ```bash
   sudo swapon -a
   ```

---

## 3. Kernel Sysctl Memory Tuning

* **Config File:** `/etc/sysctl.d/99-zram-tuning.conf`

```ini
# Prioritize swapping cold pages into compressed zram over evicting file cache
vm.swappiness = 150

# Avoid read-ahead page clustering on swap (ideal for non-rotational/zram)
vm.page-cluster = 0

# Smooth out memory reclaim triggers
vm.watermark_scale_factor = 125
vm.watermark_boost_factor = 0
```

* **Activation Command:**
  ```bash
  sudo sysctl --system
  ```

### Parameter Explanations

#### `vm.swappiness = 150`
* **Mechanism:** Controls the balance between reclaiming anonymous memory (allocations made by programs) versus file-backed memory (disk caches and libraries). In modern kernels (Linux 5.8+), the valid range is `0` to `200`.
* **Why 150 instead of 10 or 60?**
  * With slow mechanical hard drives or raw SSDs, a low swappiness (e.g., `10`) avoids disk I/O penalties at the cost of starving the file cache.
  * With **zram**, memory paging is near-instantaneous. Raising swappiness to `150` instructs the kernel to eagerly identify stale, idle memory pages (e.g., background browser tabs, dormant IDE daemons) and compact them into zram.
  * This preserves the physical file cache, ensuring application startups, code compilation, and disk reads stay hot in cache and free of disk thrashing.

#### `vm.page-cluster = 0`
* **Mechanism:** Controls page-in readahead. A value of $N$ instructs the kernel to read $2^N$ contiguous pages from swap on every fault. The default is `3` ($2^3 = 8$ pages = 32 KB).
* **Why 0?**
  * Sequential readahead clustering is designed to minimize head travel on spinning hard drives.
  * On zram, each swapped page is an independently compressed block in memory. Reading unrequested adjacent pages wastes CPU cycles on unnecessary decompression.
  * Setting `vm.page-cluster = 0` enforces single-page ($2^0 = 1$ page = 4 KB) reads, minimizing decompression latency.

#### `vm.watermark_scale_factor = 125`
* **Mechanism:** Defines the distance between the kernel's low and high memory watermarks (measured in units of 10,000, where `125` = 1.25% of total system RAM). Default is `10` (0.1%).
* **Why 125?**
  * A wider gap instructs `kswapd` (the kernel's asynchronous memory reclamation daemon) to wake up earlier and work more gradually in the background.
  * This prevents the system from suddenly hitting the "emergency minimum" watermark during heavy memory spikes, avoiding direct-reclaim pauses and stuttering.

#### `vm.watermark_boost_factor = 0`
* **Mechanism:** Introduced in Linux 5.0 to handle memory fragmentation by temporarily inflating watermarks when allocations fail high-order zone tests.
* **Why 0?**
  * Boost factor often misinterprets sudden bursty memory spikes as chronic fragmentation, causing aggressive and unnecessary background page evictions.
  * Disabling it (`0`) stabilizes reclaim behavior when paired with a generous `watermark_scale_factor`.

---

## 4. Verification & Diagnostics

### Check Active Swap Priority & Usage
```bash
swapon --show
```
*Expected Output:*
```text
NAME           TYPE      SIZE  USED PRIO
/dev/zram0     partition  24G    0B  100
/swap/swapfile file       32G    0B   10
```

### Inspect zram Compression & Efficiency
```bash
sudo zramctl --output-all
```
* Key metrics: `COMP-RATIO` (typically ~2.5x–3.0x with `zstd`), `DATA` (uncompressed size), and `TOTAL` / `MEM-USED` (actual physical RAM consumed).

### Verify Active Kernel Sysctl Parameters
```bash
sysctl vm.swappiness vm.page-cluster vm.watermark_scale_factor vm.watermark_boost_factor
```

### Monitor Memory Stalls in Real Time (PSI)
```bash
cat /proc/pressure/memory
```
* **`some`**: Percentage of time some tasks were delayed waiting for memory.
* **`full`**: Percentage of time all tasks were completely stalled waiting for memory. 
* A healthy, responsive system maintains `full avg10` close to `0.00`.