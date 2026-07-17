# NVIDIA CMP 170HX Unlock Guide

> **Status**: Research in progress. Document reflects current findings as of July 2026.

## Table of Contents

1. [Objective](#objective)
2. [Target Hardware](#target-hardware)
3. [Protection Architecture](#protection-architecture)
4. [Exploit Mechanism](#exploit-mechanism)
5. [Payload Structure](#payload-structure)
6. [Implementation](#implementation)
7. [Troubleshooting](#troubleshooting)
8. [Recovery Procedures](#recovery-procedures)
9. [Expected Results](#expected-results)
10. [Sources & References](#sources--references)

---

## 1. Objective

Unlock the NVIDIA CMP 170HX (GA100, Device ID 0x20C2) for CUDA computing at full performance:

- **64GB VRAM** (HBM2e Hynix)
- **No Throttle** (FMA/IMLA 1/32 → full speed)
- **No Power Limit**
- **Working CUDA**

---

## 2. Target Hardware

### Card Specifications

| Parameter | Value |
|-----------|-------|
| Manufacturer | NVIDIA |
| Model | CMP 170HX (Cryptocurrency Mining Processor) |
| GPU | GA100 (Ampere) |
| PCI Device ID | 0x20C2 (Vendor: 0x10DE) |
| PCI Address | 0000:01:00.0 |
| BAR0 size | 16MB (0x01000000) |
| HBM2e | 4 stacks × 16Gb × 8 layers = 64GB physical (Hynix) |

### Target Registers (BAR0 offsets)

| Address | Name | Target Value | Purpose |
|---------|------|-------------|---------|
| 0x00823804 | FEAT_OVR_PLM | 0xFFFFFFFF | Master Power Limit OFF |
| 0x0082381C | FEAT_OVR_SM_SPD | 0x88888888 | FMA/IMLA Throttle OFF (SS0) |
| 0x00823820 | FEAT_OVR_SM_SPD_1 | 0x00000008 | IMLA4 Throttle OFF (SS1) |
| 0x009A0204 | FBPA_CFG1 | 0x02779000 | 64GB Hynix HBM2e config |
| 0x00100CE0 | LMR | 0x0000020B | Logical Memory Region 64GB |

### Hardware Fuse Locks

| Fuse | Address | Value | Effect |
|------|---------|-------|--------|
| FUSE_DIS_SW_OVR | 0x00820084 | 0x1 | SW override PERMANENTLY BLOCKED |
| FUSE_MEM_LOCKED | 0x00820340 | 0x1 | Memory config locked |
| FUSE_SS_FMLA32 | 0x008207D8 | 0x5 | 1/32 FP32 compute fuse (HARDWARE) |
| FUSE_FBPA_MEM_WR_SEC | 0x00820618 | 0x1 | FBPA write security active |
| FUSE_NVLINK_DIS | 0x00820684 | 0x7 | NVLink disabled |
| FUSE_PCIE_GEN23_DIS | varies | | PCIe Gen2/3 disabled |

**BUT:** `FEAT_OVR_*` registers have hardware priority over fuses and can be written via SEC2 CSB mailbox.

---

## 3. Protection Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  MMIO Firewall — blocks BAR0 writes from Host after init    │
├─────────────────────────────────────────────────────────────┤
│  SEC2 (Falcon) — Security Boot Controller                   │
│  ├─ Verifies RSA firmware signatures                        │
│  ├─ Configures WPR2 (Write Protect Region)                  │
│  ├─ Has CSB mailbox access (privileged)                     │
│  └─ Vulnerability: DMA overflow in bounce buffer (4096 B)   │
├─────────────────────────────────────────────────────────────┤
│  GSP-RM (RISC-V) — Resource Manager firmware                │
│  ├─ Manages GPU runtime                                     │
│  ├─ Requires page tables from SEC2                          │
│  └─ Required for CUDA in driver 610.x                       │
├─────────────────────────────────────────────────────────────┤
│  CSB Mailbox Protocol (bypasses MMIO Firewall)              │
│  MAILBOX_ADDR = 0xFF01C100  (target register address)       │
│  MAILBOX_VAL  = 0xFF01C200  (value to write)                │
│  MAILBOX_CMD  = 0xFF01C000  (0x800000F2 = WRITE command)   │
│  Access: ONLY from SEC2, not from Host or GSP-RM            │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Exploit Mechanism

### Combo Strategy: Firmware Patching + DMA Overflow + FLR Reset

```
┌──────────────────────────────────────────────────────────┐
│ PHASE 1: EXPLOIT                                         │
│ 1. Patch firmware on disk (ELF section .fwsignature_*)   │
│    - Replace data with our payload (0xF800 bytes)        │
│    - UPDATE sh_size in section header (CRITICAL!)        │
│ 2. Load driver → firmware is read → SEC2 DMA             │
│ 3. DMA 0xF800 → 4096 bounce buffer → OVERFLOW            │
│ 4. Canary bypass: buffer filled with zeros, D[0x6340]=0  │
│ 5. TLV tags parsed by SEC2 booter                        │
│ 6. CSB writes via reg_write_indirect (0x10B9)            │
│ 7. Registers UNLOCKED in silicon                         │
│ 8. G_BOOTER_SUCCESS (0x27FA) → SEC2 continues boot      │
├──────────────────────────────────────────────────────────┤
│ PHASE 2: FLR RESET (KEY MOMENT!)                         │
│ echo 1 > /sys/bus/pci/devices/0000:01:00.0/reset         │
│ - Resets PCIe endpoint                                   │
│ - Clears controllers and state                           │
│ - BUT silicon registers (PLM, FBPA) ARE PRESERVED!       │
├──────────────────────────────────────────────────────────┤
│ PHASE 3: BAR0 WRITES (Compute Unlock)                    │
│ After FLR, MMIO firewall is partially open:              │
│ - FEAT_OVR_SM_SPD = 0x88888888 (SS0)                     │
│ - FEAT_OVR_SM_SPD_1 = 0x00000008 (SS1)                   │
├──────────────────────────────────────────────────────────┤
│ PHASE 4: CLEAN BOOT                                       │
│ 1. Restore original firmware from backup                  │
│ 2. Reload driver (580.x)                                  │
│ 3. GSP-RM starts normally (page tables in place)          │
│ 4. CUDA works with UNLOCKED registers                     │
└──────────────────────────────────────────────────────────┘
```

---

## 5. Payload Structure

### TLV Format (NOT ROP chain!)

SEC2 booter parses TLV tags.

```
PAYLOAD_SIZE = 0xF800  # 63488 bytes
DMA_TARGET = 0x0800    # DMEM base address

# TLV Header (first 8 bytes)
[0x0000]: 0x00000002  # TLV type = LS_SIG_VALIDATION_ID_2
[0x0004]: 0x00000180  # TLV length = 384 (RSA3K signature size)

# Canary bypass (filled with zeros)
[0x0020..0x5B3F]: 0x00  # zeros → D[0x6340] = 0
                        # __stack_chk_fail does not trigger

# ROP frames start at DMEM 0xFF3C
# Frame stride = 0x18 (24 bytes) for mpush $r3
# Frame stride = 0x24 (36 bytes) for mpush $r6 (first slot)
```

### ROP Slots (TLV tags via CSB)

```
Slot 1 @ DMEM 0xFF3C (mpush $r6, 9 dwords):
  r6=0, r5=0, r4=0, r3=0x6340 (canary addr), r2=0
  r1=0x02779000 (FBPA value)
  r0=0x009A0204 (FBPA addr)
  canary=0
  RA=0x000010B9 (reg_write_indirect)

Slot 2 @ DMEM 0xFF60 (mpush $r3, 6 dwords):
  r3=0x6340, r2=0
  r1=0xFFFFFFFF (PLM value)
  r0=0x00823804 (PLM addr)
  canary=0
  RA=0x000010B9

Slot 3 @ DMEM 0xFF78:
  r1=0x88888888 (SM_SPD), r0=0x0082381C, RA=0x10B9

Slot 4 @ DMEM 0xFF90:
  r1=0x0000020B (LMR), r0=0x00100CE0, RA=0x10B9

Slot 5 @ DMEM 0xFFA8:
  r1=0x00000008 (SS1), r0=0x00823820, RA=0x10B9

# Exit chain @ DMEM 0xFFC0:
[0xFFC0]: 0x00001D9F  # G_MPOPADDRET
[0xFFC4]: 0x00000000  # r0 = 0 (success)
[0xFFC8]: 0x000027FA  # G_BOOTER_SUCCESS → continue boot
```

### Critical: G_BOOTER_SUCCESS vs G_SECURE_TEARDOWN

```
❌ G_SECURE_TEARDOWN (0x7E76):
   - Attempts to clear DMEM
   - Hangs for 60 seconds
   - Kills SEC2 state
   - GSP-RM does not start → nvidia-smi "No devices"

✅ G_BOOTER_SUCCESS (0x27FA):
   - SEC2 thinks signature is valid
   - Continues normal boot flow
   - Configures WPR2 itself
   - Loads GSP-RM firmware
   - Performs clean HALT with mailbox=0
   - GSP-RM starts → CUDA works!
```

---

## 6. Implementation

### System Requirements

| Component | Requirement |
|-----------|-------------|
| OS | Ubuntu 24.04 (or any Linux with kernel ≥6.8) |
| Kernel | 7.0.13-3-liquorix-amd64 (recommended) |
| RAM | ≥32GB |
| Disk | ≥20GB free |
| Python3 | with mmap, struct, pyyaml, pyelftools |
| GCC | 13.3.0+ (for driver build if needed) |

### Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) \
    zstd git python3 python3-pip
pip3 install pyyaml pyelftools
```

### Driver

```bash
# WORKING VERSION: 580.173.02 (via apt)
sudo apt install -y nvidia-driver-580-open nvidia-utils-580

# Verify:
modinfo nvidia | grep version
# Expected: version: 580.173.02

# DO NOT USE 610.x (different firmware structure, version check)
```

### Firmware

```bash
# Location:
/lib/firmware/nvidia/580.173.02/gsp_tu10x.bin

# Backup (mandatory!):
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock
```

### Tools

**cmpunlocker — MANDATORY**

```bash
sudo mkdir /home/aboba67
cd /home/aboba67
git clone https://github.com/fulracoco/cmpunlocker.git
cd cmpunlocker
```

Key files:
- `payload/gsp_patch.py` — patches firmware (UPDATES sh_size!)
- `payload/build.py` — generates payload
- `payload/pipeline.py` — full pipeline (load → exploit → FLR → BAR0 → restore)

### payload_v2.py (TLV format)

```python
#!/usr/bin/env python3
"""CMP 170HX TLV Payload Generator"""
import struct, hashlib

PAYLOAD_SIZE = 0xF800
DMA_TARGET   = 0x0800

G_REG_WRITE      = 0x000010B9
G_BOOTER_SUCCESS = 0x000027FA
G_MPOPADDRET     = 0x00001D9F

WRITES = [
    (0x009A0204, 0x02779000),  # FBPA_CFG1 → 64GB
    (0x00823804, 0xFFFFFFFF),  # PLM → unlock
    (0x0082381C, 0x88888888),  # SM_SPD → no throttle
    (0x00100CE0, 0x0000020B),  # LMR → memory config
    (0x00823820, 0x00000008),  # SS1 → IMLA4
]

def w32(buf, off, v):
    struct.pack_into("<I", buf, off - DMA_TARGET, v & 0xFFFFFFFF)

def build_payload():
    buf = bytearray(PAYLOAD_SIZE)  # all zeros → canary bypass

    # TLV Header
    w32(buf, 0x0800, 0x00000002)  # type
    w32(buf, 0x0804, 0x00000180)  # length

    # Slot 1 (mpush $r6, 9 dwords)
    a, v = WRITES[0]
    for i, x in enumerate([0, 0, 0, 0x6340, 0, v, a, 0, G_REG_WRITE]):
        w32(buf, 0xFF3C + i*4, x)

    # Slots 2-5 (mpush $r3, 6 dwords)
    addr = 0xFF60
    for a, v in WRITES[1:]:
        for i, x in enumerate([0x6340, 0, v, a, 0, G_REG_WRITE]):
            w32(buf, addr + i*4, x)
        addr += 0x18

    # Exit chain
    w32(buf, 0xFFC0, G_MPOPADDRET)
    w32(buf, 0xFFC4, 0)
    w32(buf, 0xFFC8, G_BOOTER_SUCCESS)

    return bytes(buf)

def main():
    payload = build_payload()
    with open('payload_v2.bin', 'wb') as f:
        f.write(payload)
    print(f'Written: payload_v2.bin ({len(payload)} bytes)')
    print(f'MD5: {hashlib.md5(payload).hexdigest()}')

if __name__ == '__main__':
    main()
```

### probe_170hx.py (register readout)

```python
#!/usr/bin/env python3
import mmap, struct, os

PCI_DEV = "0000:01:00.0"
RESOURCE_PATH = f"/sys/bus/pci/devices/{PCI_DEV}/resource0"

REGISTERS = [
    (0x00823804, "FEAT_OVR_PLM", 0xFFFFFFFF),
    (0x0082381C, "FEAT_OVR_SM_SPD", 0x88888888),
    (0x009A0204, "FBPA_CFG1", 0x02779000),
]

def main():
    fd = os.open(RESOURCE_PATH, os.O_RDONLY)
    mm = mmap.mmap(fd, 16777216, mmap.MAP_SHARED, mmap.PROT_READ)

    print("=== CMP 170HX Register Probe ===\n")
    for offset, name, target in REGISTERS:
        val = struct.unpack_from('<I', mm, offset)[0]
        status = "✅ UNLOCKED" if val == target else "❌ LOCKED"
        print(f"[0x{offset:08X}] {name:20s} = 0x{val:08X} {status}")

    mm.close()
    os.close(fd)

if __name__ == '__main__':
    main()
```

---

## 7. Full Pipeline (Step-by-step)

### Step 1: Preparation

```bash
# 1.1. Ensure 580 driver is loaded
cat /proc/driver/nvidia/version
# Expected: 580.173.02

# 1.2. Backup firmware
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock

# 1.3. Clone cmpunlocker
cd /home/aboba67
git clone https://github.com/fulracoco/cmpunlocker.git
```

### Step 2: COLD BOOT (MANDATORY!)

```bash
sudo poweroff
# UNPLUG POWER CORD FOR 60 SECONDS
# This resets WPR2 latch and SEC2 capacitors
# Warm reboot WILL NOT WORK!
```

### Step 3: Generate Payload

```bash
cd /home/aboba67/cmp170hx-unlock
python3 payload_v2.py
# Created: payload_v2.bin (63488 bytes)
```

### Step 4: Patch Firmware via cmpunlocker

```bash
# Use cmpunlocker's gsp_patch.py (CORRECTLY updates sh_size!)
cd /home/aboba67/cmpunlocker

python3 -c "
import sys
sys.path.insert(0, '.')
from payload.gsp_patch import patch_gsp

payload = open('/home/aboba67/cmp170hx-unlock/payload_v2.bin', 'rb').read()
patch_gsp(
    '/lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock',
    payload,
    '/tmp/gsp_patched.bin'
)
"

# Replace firmware
sudo cp /tmp/gsp_patched.bin /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin
```

### Step 5: Run Pipeline

```bash
# Use ORIGINAL cmpunlocker pipeline
cd /home/aboba67/cmpunlocker
sudo python3 payload/pipeline.py "0000:01:00.0" \
    "/lib/firmware/nvidia/580.173.02/gsp_tu10x.bin"
```

**Pipeline actions:**

1. Stop display manager + unload nvidia
2. Backup firmware (already done, but does it again)
3. Build payload (uses its own build.py — can skip if payload_v2.bin exists)
4. Patch firmware (gsp_patch.py)
5. Load module + bind → DMA overflow → ROP → registers UNLOCKED
6. Verify PLM (CRITICAL CHECK!)
7. FLR Reset #1
8. Aggressive unload
9. FLR Reset #2
10. Re-enable PCI + BAR0 writes (SS0/SS1)
11. Restore firmware
12. Reload clean driver

### Step 6: Verify Results

```bash
# Check registers
sudo python3 /home/aboba67/cmp170hx-unlock/probe_170hx.py

# Expected:
# [0x00823804] FEAT_OVR_PLM = 0xffffffff ✅ UNLOCKED
# [0x0082381c] FEAT_OVR_SM_SPD = 0x88888888 ✅ UNLOCKED
# [0x009a0204] FBPA_CFG1 = 0x02779000 ✅ 64GB UNLOCKED

# Check nvidia-smi
nvidia-smi
```

**Expected output:**

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 580.173.02   Driver Version: 580.173.02   CUDA Version: 12.8     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA Graphics...  Off  | 00000000:01:00.0 Off |                    0 |
| N/A   42C    P0    34W / 300W |      0MiB / 65536MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
```

---

## 8. Recovery Procedures

### Restore Firmware

```bash
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin
sudo modprobe -r nvidia
sudo modprobe nvidia
```

### Hard Reset GPU

```bash
# PCI remove + rescan
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/remove
sleep 2
echo 1 | sudo tee /sys/bus/pci/rescan

# Or FLR
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/reset
```

### Full Clean

```bash
sudo poweroff
# Unplug power cord for 60 seconds
# Power on — GPU is in virgin state
```

---

## 9. Troubleshooting

### Problem 1: PLM stays at 0xFFFFFF8F (stock)

**Cause:** sh_size not updated in ELF section header

**Fix:** Use `cmpunlocker/gsp_patch.py`, NOT our `pipeline_580.py`

### Problem 2: PLM = 0xFFFFFE8E (partially changed)

**Cause:** BAR0 write 0xFFFFFFFF after FLR (W1C behavior)

**Fix:** This is NORMAL if it happened AFTER FLR. Check PLM BEFORE FLR (Step 6 of pipeline).

### Problem 3: nvidia-smi "No devices found"

**Cause:** GSP-RM did not start (G_SECURE_TEARDOWN used instead of G_BOOTER_SUCCESS)

**Fix:** Use `G_BOOTER_SUCCESS = 0x27FA` in exit chain

### Problem 4: CUDA works but slow (~46 tok/s instead of 1123)

**Cause:** FMA throttle not removed (SS0/SS1 not written)

**Fix:** Verify BAR0 writes after FLR passed:

```bash
sudo python3 probe_170hx.py | grep SM_SPD
# Should be 0x88888888
```

### Problem 5: "Failed to initialize NVML: Driver/library version mismatch"

**Cause:** User-space libraries from different driver

**Fix:**

```bash
sudo ln -sf /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.580.173.02 \
            /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
sudo ldconfig
```

---

## 10. Alternative Approaches (DO NOT WORK)

### ❌ Driver-level exploit (610 driver)

Patch `_kgspCreateSignatureMemdesc` in kernel_gsp.c

**Problem:** 610 firmware has patched SEC2 booter → DMA overflow does not work at firmware level → only firmware-level approach works

### ❌ BAR0 writes at COLD BOOT

Attempt to write registers via mmap before driver load

**Problem:** MMIO firewall active from power-on → all reads return 0xFFFFFFFF (garbage) → all writes discarded

### ❌ Custom VBIOS via CH341A

Physical reprogramming of SPI flash

**Problem:** Boot ROM verifies RSA-3072 signature → private keys not available in leaks → custom VBIOS = brick

### ❌ Hardware Strap Mod (resistor rework)

Physical modification of strap resistors on PCB

**Problem:** Not confirmed by 170th Street research → "Claims are speculative" → FUSE_MEM_LOCKED blocks runtime config

### ❌ PCIe Capacitor Mod

Soldering 24 capacitors for x4 → x16

**Result:** ✅ WORKS (confirmed April 2026)

**BUT:** only gives 4× bandwidth improvement, does NOT solve FMA throttle or 64GB VRAM

---

## 11. Expected Results

### Stock 580 (no exploit):

```
VRAM: 8GB (FBPA_CFG1 = 0x02449000)
Compute: 1/32 throttle (FEAT_OVR_SM_SPD = 0x00000000)
CUDA: ✅ works
```

### After successful unlock:

```
VRAM: 64GB (FBPA_CFG1 = 0x02779000)
Compute: No throttle (FEAT_OVR_SM_SPD = 0x88888888)
CUDA: ✅ works
```

### Comparison with A100 80GB:

| Metric | A100 80GB | CMP 170HX | % of A100 |
|--------|-----------|-----------|-----------|
| VRAM | 80 GB | 64 GB | 80% |
| FP32 Performance | 19.5 TFLOPS | ~18 TFLOPS | ~92% |
| llama.cpp pp64 | ~1200 tok/s | 1123 tok/s | 94% |
| llama.cpp tg128 | ~125 tok/s | 118 tok/s | 94% |

---

## 12. Sources & References

### Research

**170th Street Research:** https://170th-street.gitbook.io/hx/
- PCIe Capacitor Mod (confirmed)
- VBIOS Flash Guide
- Falcon Security Architecture
- Runtime PCIe Unlock Attempt (failed)
- Open Research Problems

**JRex286 Gist:** https://gist.github.com/JRex286
- GA100 Fuse & Register Reference Table
- 11 Ampere cards, 120 registers
- Confirmation of hardware fuse locks

**Jon Pry (ASU):** "A Canary in the Crypto Mine" (June 2026)
- Academic description of DMA overflow vulnerability
- Canary bypass method
- CSB mailbox protocol

### Tools

**Ghidra (NSA):** https://github.com/NationalSecurityAgency/ghidra
- For firmware analysis (if gadget hunting needed)

**sass-king:** https://github.com/florianmattana/sass-king
- SASS instruction set reverse engineering
- Post-unlock optimization

---

## 13. Final Checklist

### Before starting:

- [ ] Ubuntu 24.04 + kernel ≥6.8
- [ ] 580.173.02 driver installed
- [ ] cmpunlocker cloned
- [ ] payload_v2.py created
- [ ] probe_170hx.py created
- [ ] Firmware backup made

### Execution:

- [ ] COLD BOOT (unplug for 60 sec)
- [ ] python3 payload_v2.py → payload_v2.bin
- [ ] cmpunlocker/gsp_patch.py → patch firmware
- [ ] cmpunlocker/pipeline.py → full pipeline
- [ ] Check probe_170hx.py → all ✅ UNLOCKED
- [ ] nvidia-smi → GPU visible, 65536 MiB
- [ ] llama-bench → ~1123 tok/s

### If something doesn't work:

- [ ] Verify sh_size updated (not our pipeline_580.py!)
- [ ] Verify G_BOOTER_SUCCESS in exit chain (not secure_teardown)
- [ ] Check PLM BEFORE FLR (not after!)
- [ ] COLD BOOT mandatory (not warm reboot)

---

## 14. Conclusions

### What WORKS:

- ✅ Firmware-level DMA Overflow via cmpunlocker/gsp_patch.py
- ✅ TLV tags instead of ROP (SEC2 parses, does not execute)
- ✅ G_BOOTER_SUCCESS (0x27FA) for clean boot flow
- ✅ FLR Reset preserves silicon registers
- ✅ BAR0 writes after FLR (MMIO firewall partially open)
- ✅ CUDA on 580 driver

### What DOES NOT WORK:

- ❌ Driver-level exploit (610 firmware patched)
- ❌ BAR0 writes at COLD BOOT (firewall from power-on)
- ❌ Custom VBIOS (RSA signature check)
- ❌ Hardware strap mod (not confirmed)
- ❌ ROP-format payload (SEC2 uses M-stack)

### Key to success:

**Use ORIGINAL cmpunlocker WITHOUT modifications:**

- `gsp_patch.py` — correctly updates sh_size
- `pipeline.py` — correct FLR + BAR0 approach
- Driver 580.173.02

---

**Document Version:** 1.0  
**Last Updated:** July 2026  
**Status:** Research in progress
