# M-Technics ПРИДЕТСЯ ИЗВИНИТЬСЯ

# NVIDIA CMP 170HX Unlock Guide

> **Research in progress.** Document reflects current findings as of July 2026.  
> Based on the [cmpunlocker](https://github.com/kinako404/cmpunlocker) tool by @amoghmunikote (originally [fulracoco/cmpunlocker](https://github.com/fulracoco/cmpunlocker)), with additional fixes from [Issue #1](https://github.com/abobasixseven/unlock-cmp-170hx/issues/1) and empirical testing on GA100 rev a1 silicon.
>
> **Document version:** 2.0 (corrected)

---

## 1. OBJECTIVE

Unlock the NVIDIA CMP 170HX (GA100, Device ID 0x20C2) for CUDA compute at full performance:
- **64GB VRAM** (HBM2e Hynix) — *experimental, see §10*
- **No Throttle** (FMA/IMLA 1/32 → full speed) — ✅ confirmed working
- **No Power Limit**
- **Working CUDA**

---

## 2. HARDWARE ARCHITECTURE

### Card

```
Manufacturer: NVIDIA
Model: CMP 170HX (Cryptocurrency Mining Processor)
GPU: GA100 (Ampere) — rev a1
PCI Device ID: 0x20C2 (Vendor: 0x10DE)
PCI Address: 0000:01:00.0
BAR0 size: 16MB (0x01000000)
HBM2e: 5 stacks × 16GB × 8 layers = 64GB physically (Hynix), software-locked to 8GB
```

### Target Registers (BAR0 offsets)

```
Address       Name                  Value       Purpose
──────────────────────────────────────────────────────────────
0x00823804    FEAT_OVR_PLM          0xFFFFFFFF  Master Power Limit OFF
0x0082381C    FEAT_OVR_SM_SPD       0x88888888  FMA/IMLA Throttle OFF (SS0)
0x00823820    FEAT_OVR_SM_SPD_1     0x00000008  IMLA4 Throttle OFF (SS1)
0x009A0204    FBPA_CFG1             0x02779000  HBM2e config (64GB stable)
0x00100CE0    LMR                   0x0000020B  Logical Memory Region
```


### Hardware Fuse Locks

```
FUSE_DIS_SW_OVR    (0x00820084) = 0x1  → SW override PERMANENTLY BLOCKED
FUSE_MEM_LOCKED    (0x00820340) = 0x1  → Memory config locked
FUSE_SS_FMLA32     (0x008207D8) = 0x5  → 1/32 FP32 compute fuse (HARDWARE)
FUSE_FBPA_MEM_WR_SEC (0x00820618) = 0x1 → FBPA write security active
FUSE_NVLINK_DIS    (0x00820684) = 0x7  → NVLink disabled
FUSE_PCIE_GEN23_DIS (varies)           → PCIe Gen2/3 disabled
```

**BUT:** The `FEAT_OVR_*` registers have hardware priority over fuses and can be written via the SEC2 CSB mailbox, which bypasses the MMIO firewall.

### Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  MMIO Firewall — blocks BAR0 writes from Host after init    │
├─────────────────────────────────────────────────────────────┤
│  SEC2 (Falcon) — Security Boot Controller                   │
│  ├─ Verifies RSA firmware signatures                        │
│  ├─ Configures WPR2 (Write Protect Region)                  │
│  ├─ Has access to CSB mailbox (privileged)                  │
│  └─ Vulnerability: DMA overflow in bounce buffer (4096 B)   │
├─────────────────────────────────────────────────────────────┤
│  GSP-RM (RISC-V) — Resource Manager firmware                │
│  ├─ Manages GPU runtime                                     │
│  ├─ Requires page tables from SEC2                          │
│  └─ Needed for CUDA in driver 580.x                         │
├─────────────────────────────────────────────────────────────┤
│  CSB Mailbox Protocol (bypasses MMIO Firewall)              │
│  MAILBOX_ADDR = 0xFF01C100  (target register address)       │
│  MAILBOX_VAL  = 0xFF01C200  (value to write)                │
│  MAILBOX_CMD  = 0xFF01C000  (0x800000F2 = WRITE command)    │
│  Access: ONLY from SEC2, not from Host or GSP-RM            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. EXPLOIT MECHANISM

### Combo-strategy: Firmware Patching + DMA Overflow + FLR Reset

The exploit uses a **ROP chain** (NOT a TLV-format payload) injected into the GSP firmware's `.fwsignature_ga100` ELF section. When SEC2 loads the firmware and attempts signature validation, the DMA overflow corrupts the SEC2 stack, triggering the ROP chain which writes register values via CSB (Command Submission Buffer) — bypassing the MMIO firewall entirely.

**Critical insight:** The ROP chain and GSP-RM boot happen in **SEPARATE cycles**. The exploit runs in Phase 1, unlocks registers, then GSP-RM boots normally in Phase 2 with stock firmware — seeing the already-unlocked registers.

```
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: EXPLOIT (patched firmware)                          │
│ 1. Patch firmware on disk (ELF section .fwsignature_ga100)   │
│    - Replace signature data with ROP payload (0xF800 bytes)  │
│    - UPDATE sh_size in section header (CRITICAL!)            │
│ 2. Load driver → firmware read → SEC2 DMA                    │
│ 3. DMA 0xF800 → 4096 bounce buffer → OVERFLOW               │
│ 4. Canary check: SEC2 reads 0xFACEB13D at DMEM 0x6340       │
│ 5. ROP chain executes via reg_write_indirect gadget (0x10B9) │
│ 6. CSB writes unlock registers in silicon                    │
│ 7. Tail frame returns to 0x810D (booter main)                │
│ 8. GSP-RM may fail to boot — THIS IS EXPECTED                │
├──────────────────────────────────────────────────────────────┤
│ PHASE 2: FLR RESET (registers persist!)                      │
│ echo 1 > /sys/bus/pci/devices/0000:01:00.0/reset             │
│ - Resets PCIe endpoint                                       │
│ - Clears controllers and state                               │
│ - BUT silicon registers (PLM, FBPA) SURVIVE in always-on     │
│   island!                                                    │
├──────────────────────────────────────────────────────────────┤
│ PHASE 3: BAR0 WRITES (Compute Unlock)                        │
│ After FLR, MMIO firewall is partially open:                  │
│ - FEAT_OVR_SM_SPD = 0x88888888 (SS0) via host BAR0          │
│ - FEAT_OVR_SM_SPD_1 = 0x00000008 (SS1) via host BAR0        │
│ (PLM is already open from ROP chain — CSB can set fused      │
│  bits that host BAR0 writes cannot)                          │
├──────────────────────────────────────────────────────────────┤
│ PHASE 4: CLEAN BOOT (stock firmware)                         │
│ 1. Restore original firmware from backup                     │
│ 2. Reload driver (580.x)                                    │
│ 3. SEC2 validates stock signature → passes                   │
│ 4. GSP-RM starts normally (page tables intact)               │
│ 5. CUDA works with UNLOCKED registers                       │
└──────────────────────────────────────────────────────────────┘
```

### Why two phases?

On GA100 rev a1, GSP RISC-V does not start in the same boot cycle as the ROP chain. The ROP chain corrupts SEC2's execution state, preventing proper GSP release. The solution:

1. **Phase 1:** ROP runs → registers unlocked → GSP fails (OK!)
2. **Aggressive unload + FLR** → registers persist in always-on island
3. **Phase 2:** Stock firmware → SEC2 normal → GSP boots → CUDA works

The `FEAT_OVR_*` registers survive FLR because they reside in an always-on power domain. Stock GSP-RM does not touch these registers — it uses the GPU as-is with the unlocked configuration.

---

## 4. SEC2 GADGETS (GA100 booter)

From annotated disassembly of the SEC2 booter:

```
Address   Name                  Purpose
────────────────────────────────────────────────────────
0x10B9    bar0_write_gadget     CSB write: r2=value → [r3]=addr
                                 (ROP return address for each write frame)
0x810D    tail_return           Return to booter main flow
                                 (clean exit, NOT booter_success)
```

### Gadget addresses NOT to use

```
❌ 0x27FA (G_BOOTER_SUCCESS)  — reinitializes SEC2, resets ROP writes
❌ 0x7E76 (G_SECURE_TEARDOWN) — kills SEC2, GSP-RM never starts
❌ 0x1D9F (G_MPOPADDRET)      — outdated gadget, not needed
❌ 0x8262 (G_RET)             — plain ret, crashes SEC2 stack
```

Only `0x10B9` (write gadget) and `0x810D` (tail return) are used in the working payload.

### CSB Write Sequence (inside bar0_write_gadget)

```
SEC2 executes:
1. Read r3 = target register address
2. Read r2 = value to write
3. CSB write: MAILBOX_ADDR ← r3
4. CSB write: MAILBOX_VAL  ← r2
5. CSB write: MAILBOX_CMD  ← 0x800000F2 (WRITE)
→ Hardware writes value to register (bypasses MMIO firewall!)
```

---

## 5. PAYLOAD STRUCTURE (ROP Chain)

**The payload is a pure ROP chain — NO TLV header, NO TLV tags.**

The SEC2 booter's signature validation function reads the `.fwsignature_ga100` section data into DMEM at offset 0x0800. By replacing the signature with our ROP payload, the DMA overflow corrupts the SEC2 stack, and the canary check triggers the ROP chain execution.

```python
PAYLOAD_SIZE = 0xF800   # 63488 bytes
DMA_TARGET   = 0x0800   # DMEM base address where payload is loaded

CANARY_VALUE = 0xFACEB13D
CANARY_DMEM  = 0x00006340   # DMEM address where canary is written

FRAME_START  = 0xFF48   # First ROP frame (DMEM address)
FRAME_STRIDE = 0x18     # 24 bytes = 6 dwords per frame

GADGET_WRITE = 0x10B9   # bar0_write_gadget (return addr for write frames)
TAIL_RETURN  = 0x810D   # tail_return (return addr for exit frame)
```

### Frame Layout (6 dwords, 24 bytes each)

```
Offset  Field         Purpose
──────  ───────────   ──────────────────────────────────────
0x00    r0            canary_addr (0x00006340)
0x04    r1            0x00000000
0x08    r2            VALUE to write to register
0x0C    r3            REGISTER address (BAR0 offset)
0x10    saved_reg     canary (0xFACEB13D)
0x14    return_addr   0x10B9 (gadget) or 0x810D (tail)
```

### ROP Writes (3 writes + 1 tail frame)

The ROP chain performs **3 register writes** via CSB. SS0/SS1 are NOT in the payload — they are written by the host via BAR0 after FLR (Phase 3).

```
Frame 1 @ DMEM 0xFF48: write FBPA_CFG1
  r0=0x00006340, r1=0x00000000, r2=0x02779000, r3=0x009A0204,
  saved=0xFACEB13D, RA=0x000010B9

Frame 2 @ DMEM 0xFF60: write LMR
  r0=0x00006340, r1=0x00000000, r2=0x0000020B, r3=0x00100CE0,
  saved=0xFACEB13D, RA=0x000010B9

Frame 3 @ DMEM 0xFF78: write PLM (unlock)
  r0=0x00006340, r1=0x00000000, r2=0xFFFFFFFF, r3=0x00823804,
  saved=0xFACEB13D, RA=0x000010B9

Tail frame @ DMEM 0xFF90: return to booter main
  r0=0x00000000, r1=0x00000000, r2=0x00000000, r3=0x00000000,
  saved=0xFACEB13D, RA=0x0000810D
```

### Memory Map of Payload Buffer

```
File offset    DMEM addr    Content
─────────────  ──────────   ─────────────────────────────
0x0000         0x0800       zeros (padding)
0x5B40         0x6340       canary = 0xFACEB13D
0x5B44-0xF747  0x6344-0xFF47  zeros (padding)
0xF748         0xFF48       Frame 1 (FBPA write)
0xF760         0xFF60       Frame 2 (LMR write)
0xF778         0xFF78       Frame 3 (PLM write)
0xF790         0xFF90       Tail frame (return to 0x810D)
0xF7A8-0xF7FF  0xFFA8-0xFFFF  zeros (padding)
```

### Why SS0/SS1 are NOT in the ROP chain

SS0 (`0x0082381C`) and SS1 (`0x00823820`) are in the `FEAT_OVR` register block — the same block as PLM (`0x00823804`). Once PLM is unlocked by the ROP chain (writing `0xFFFFFFFF` via CSB), the host can write SS0/SS1 directly via BAR0 mmap. This is simpler and more reliable than including them in the ROP chain.

The CSB write path (used by ROP) can set **fused bits** in PLM that host BAR0 writes cannot. This is why PLM must be unlocked by the ROP chain, but SS0/SS1 can be written by the host afterward.

---

## 6. CRITICAL ERRORS TO AVOID

### Error #1: NOT updating sh_size in ELF section header

```python
# ❌ BROKEN CODE (does not update sh_size):
data[sig_off:sig_off + len(payload)] = payload
# sh_size remains original (0x1000)
# Driver copies only 4096 bytes → overflow does NOT happen!

# ✅ CORRECT CODE (cmpunlocker/gsp_patch.py):
gsp[sig_file_off : sig_file_off + payload_size] = payload
struct.pack_into("<Q", shdrs, sig_idx * e_shentsize + 0x20, payload_size)
# sh_size updated to 0xF800
# Driver copies full 63488 bytes → overflow happens!
```

### Error #2: Using TLV format instead of ROP chain

```
❌ TLV header (type=2, length=0x180) at DMEM 0x0800
   - SEC2 enters signature validation code path
   - TLV tags are parsed, but this is NOT how the exploit works
   - May cause SEC2 to take unexpected code paths

✅ Pure ROP chain (no TLV header)
   - Payload starts with zeros until canary at 0x6340
   - ROP frames at 0xFF48 trigger via stack corruption
   - Canary check passes → CSB writes execute
```

### Error #3: Using G_BOOTER_SUCCESS (0x27FA) as exit

```
❌ G_BOOTER_SUCCESS (0x27FA):
   - Reinitializes SEC2
   - Resets our ROP register writes
   - SEC2 enters clean boot path, may overwrite PLM

❌ G_SECURE_TEARDOWN (0x7E76):
   - Kills SEC2 state
   - GSP-RM never starts → nvidia-smi "No devices"

✅ tail_return (0x810D):
   - Returns to booter main function
   - Clean exit without reinitialization
   - Register writes persist
```

### Error #4: Putting SS0/SS1 in the ROP chain

```
❌ 5 writes in ROP (FBPA, LMR, PLM, SS0, SS1)
   - Unnecessary complexity
   - SS0/SS1 can be written by host BAR0 after PLM is open
   - More frames = more chances for ROP chain to fail

✅ 3 writes in ROP (FBPA, LMR, PLM) + host BAR0 for SS0/SS1
   - Minimal ROP chain
   - SS0/SS1 written via apply_unlock() after FLR
   - Matches the working cmpunlocker pipeline
```

### Error #5: Using 610.x or 595.x driver

```
❌ 610.x driver:
   - Different firmware structure
   - GSP-RM boot fails on GA100 rev a1 (interrupt table init error)
   - Version check rejects 580.x firmware

❌ 595.x driver:
   - GSP-RM init fails (intrInitInterruptTable_HAL assertion)
   - Built-in CMP170HX support is incomplete

✅ 580.173.02 (nvidia-open):
   - Gadget addresses 0x10B9/0x810D verified correct
   - GSP-RM boots normally with stock firmware in Phase 2
   - Official CMP 170HX support
```

### Error #6: Writing PLM via host BAR0 (kernel driver)

```
❌ Kernel GPU_REG_WR32(pGpu, 0x00823804, 0xFFFFFFFF)
   - Host BAR0 writes CANNOT set fused bits (4-6) in PLM
   - Results in PLM = 0xFFFFFF8F (partially locked)
   - Overwrites the ROP chain's CSB write which DID set all bits

✅ ROP chain writes PLM via CSB
   - CSB bypasses MMIO firewall
   - CSB can set fused bits
   - PLM = 0xFFFFFFFF (fully open)
   - Host kernel must NOT write PLM for CMP170HX
```

---

## 7. ENVIRONMENT

### System Requirements

```bash
OS: Ubuntu 24.04 (or any Linux with kernel ≥6.8)
Kernel: 7.0.13-3-liquorix-amd64 (recommended)
RAM: ≥32GB
Disk: ≥20GB free
Python3: with modules mmap, struct, pyyaml
GCC: 13.3.0+ (only if building driver from source)
```

### Install Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) \
    zstd git python3 python3-pip
pip3 install --user pyyaml
```

### Driver

```bash
# WORKING VERSION: 580.173.02 (nvidia-open)
sudo apt install -y nvidia-driver-580-open nvidia-utils-580

# Verify:
modinfo nvidia | grep version
# Should show: version: 580.173.02

# DO NOT USE 610.x or 595.x (see §6 Error #5)
```

### Firmware

```bash
# Location:
/lib/firmware/nvidia/580.173.02/gsp_tu10x.bin

# Backup (mandatory!):
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock
```

---

## 8. TOOLS

### cmpunlocker — REQUIRED

Use the [kinako404/cmpunlocker](https://github.com/kinako404/cmpunlocker) fork (includes VRAM configuration and bug fixes):

```bash
cd /home/aboba67
git clone https://github.com/kinako404/cmpunlocker.git
cd cmpunlocker
```

**Key files:**
- `payload/gsp_patch.py` — patches firmware (UPDATES sh_size correctly!)
- `payload/build.py` — generates ROP payload (3 writes + tail)
- `payload/pipeline.py` — full pipeline (load → exploit → FLR → BAR0 → restore)
- `payload/driver.py` — module management (load/unload/aggressive_unload/flr_reset)
- `payload/bar0.py` — BAR0 read/write via mmap
- `common/constants.yaml` — all constants (registers, gadgets, DMEM layout)
- `unlock/compute.py` — SS0/SS1 apply + PLM check
- `daemon/watchdog.py` — systemd daemon for auto-reapply

### Required Fix: constants.yaml

The upstream `constants.yaml` is **missing** the `plm` section under `host_bar0_writes`. Without it, `compute.py` crashes with `TypeError: 'NoneType' object cannot be interpreted as an integer`. This is [Issue #1](https://github.com/abobasixseven/unlock-cmp-170hx/issues/1).

**Fix:** Add `plm` section to `common/constants.yaml`:

```yaml
host_bar0_writes:
  plm:                              # ← ADD THIS SECTION (missing in upstream!)
    addr:  0x00823804
    value: 0xFFFFFFFF
    note: "FEAT_OVR_PLM — unlock marker"
  ss0:
    addr:  0x0082381C
    value: 0x88888888
    note: "FEAT_OVR_SM_SPD (all SMs max speed)"
  ss1:
    addr:  0x00823820
    value: 0x00000008
    note: "FEAT_OVR_SM_SPD_1 (IMLA4 override)"
```

Also update `unlock/compute.py` to use `plm` (not `feat_ovr_plm`):

```python
# In is_plm_open() and apply_unlock(), change:
plm_addr = get('host_bar0_writes.plm.addr')        # was: feat_ovr_plm.addr
plm_open = get('host_bar0_writes.plm.value')       # was: feat_ovr_plm.value
```

### Required Fix: enumerate_gpu() in pipeline

If `nvidia-persistenced` is not installed or enabled, loading the driver does not automatically probe the GPU. The firmware exploit never runs, and PLM stays locked. This is [Issue #1](https://github.com/abobasixseven/unlock-cmp-170hx/issues/1).

**Fix:** Add `enumerate_gpu()` to `payload/driver.py`:

```python
def enumerate_gpu() -> None:
    """Force the driver to probe the GPU by running nvidia-smi."""
    subprocess.run(["nvidia-smi"], capture_output=True, text=True, check=False)
```

And call it in `payload/pipeline.py` after `load_module()`:

```python
load_module()
enumerate_gpu()    # ← ADD THIS (needed if nvidia-persistenced not running)
time.sleep(5)
```

### probe_170hx.py (register reader)

```python
#!/usr/bin/env python3
import mmap, struct, os

PCI_DEV = "0000:01:00.0"
RESOURCE_PATH = f"/sys/bus/pci/devices/{PCI_DEV}/resource0"

REGISTERS = [
    (0x00823804, "FEAT_OVR_PLM",    0xFFFFFFFF),
    (0x0082381C, "FEAT_OVR_SM_SPD", 0x88888888),
    (0x00823820, "FEAT_OVR_SM_SPD_1", 0x00000008),
    (0x009A0204, "FBPA_CFG1",       0x02779000),
    (0x00100CE0, "LMR",             0x0000020B),
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

## 9. FULL PIPELINE (Step-by-Step Execution)

### Step 1: Preparation

```bash
# 1.1. Verify 580 driver is loaded
cat /proc/driver/nvidia/version | head -1
# Should show: 580.173.02

# 1.2. Backup firmware
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock

# 1.3. Clone cmpunlocker (kinako404 fork)
cd /home/aboba67
git clone https://github.com/kinako404/cmpunlocker.git
cd cmpunlocker

# 1.4. Apply fixes from Issue #1
# - Add plm section to common/constants.yaml (see §8)
# - Add enumerate_gpu() to payload/driver.py (see §8)
# - Update unlock/compute.py to use 'plm' key (see §8)
```

### Step 2: Cold Boot (recommended for first run)

```bash
sudo poweroff
# PHYSICALLY UNPLUG POWER CABLE FOR 60 SECONDS
# This resets WPR2 latch and SEC2 capacitors
# Warm reboot may NOT work for the first run
# (Subsequent runs can use FLR only — no cold boot needed)
```

### Step 3: Configure VRAM Target (optional)

By default, cmpunlocker uses `unlocked_64gb` (0x02779000, stable). To change:

```bash
# Edit common/constants.yaml:
# vram_config:
#   fb_ctrl:
#     default_target: "unlocked_64gb"   # stable (recommended)
```

> **Warning:** VRAM unlock is experimental. See §10 for details.

### Step 4: Run Pipeline

```bash
# Option A: Run cmpunlocker pipeline directly
cd /home/aboba67/cmpunlocker
sudo python3 payload/pipeline.py "0000:01:00.0" \
    "/lib/firmware/nvidia/580.173.02/gsp_tu10x.bin"

# Option B: Install with daemon (auto-reapply on reboot)
sudo ./install.sh
```

### What the pipeline does:

1. **Stop display manager** + unload nvidia modules
2. **Backup firmware** → `.cmpunlocker.bak`
3. **Build ROP payload** (3 writes: FBPA, LMR, PLM + tail frame)
4. **Patch firmware** via gsp_patch.py (updates sh_size!)
5. **Load module** + `enumerate_gpu()` → SEC2 boots with patched firmware
   - DMA overflow → canary check → ROP chain → CSB writes
   - Registers UNLOCKED (PLM, FBPA, LMR via CSB)
   - GSP-RM may fail — **this is expected**
6. **Verify PLM** (read 0x00823804 — should be 0xFFFFFFFF)
7. **FLR Reset #1** → GPU resets, registers persist
8. **Aggressive unload** (kill -9 /dev/nvidia* holders, rmmod -f)
9. **FLR Reset #2** → another hardware reset
10. **Apply compute unlock** → HOST BAR0 writes SS0/SS1 (PLM already open)
11. **Apply VRAM unlock** → HOST BAR0 write FBPA_CFG1 (if PLM open)
12. **Restore original firmware** from backup
13. **Reload driver** with stock firmware
    - SEC2 validates stock signature → passes
    - GSP-RM boots normally
    - CUDA works with unlocked registers

### Step 5: Verify Result

```bash
# Check registers
sudo python3 /home/aboba67/cmpunlocker/probe_170hx.py
# Expected:
# [0x00823804] FEAT_OVR_PLM        = 0xFFFFFFFF ✅ UNLOCKED
# [0x0082381C] FEAT_OVR_SM_SPD     = 0x88888888 ✅ UNLOCKED
# [0x00823820] FEAT_OVR_SM_SPD_1   = 0x00000008 ✅ UNLOCKED
# [0x009A0204] FBPA_CFG1           = 0x02779000 ✅ UNLOCKED (64GB)
# [0x00100CE0] LMR                 = 0x0000020B ✅ UNLOCKED

```

```
Expected nvidia-smi output:

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

### Step 6: Verify CUDA Performance

```bash
# Check SM clock cap is removed
nvidia-smi --query-gpu=clocks.max.sm --format=csv,noheader

# Check VRAM capacity
nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits

# CUDA compute test
python3 -c "
import ctypes
lib = ctypes.CDLL('libcuda.so.1')
r = lib.cuInit(0)
print(f'cuInit: {r} (0 = success)')
if r == 0:
    count = ctypes.c_int()
    lib.cuDeviceGetCount(ctypes.byref(count))
    print(f'CUDA devices: {count.value}')
"
```

---

## 10. VRAM UNLOCK STATUS & LIMITATIONS

### What works (confirmed)

- ✅ **FMA/IMLA throttle removal** (SS0/SS1) — FP32 ~12 TFLOPS, FP64 ~6.3 TFLOPS
- ✅ **PLM unlock** — registers writable via CSB
- ✅ **Driver 580.173.02** — GSP-RM boots normally in Phase 2
- ✅ **FLR persistence** — registers survive FLR in always-on island
- ✅ **Daemon watchdog** — auto-reapplies unlock every second

### What does NOT work (known limitations)

- ❌ **VRAM capacity unlock** — even after ROP chain writes FBPA_CFG1=0x02779000 (64GB), `nvidia-smi` may still show 8GB. The FBPA_CFG1 register write via CSB succeeds (register value changes), but GSP-RM may reconfigure memory based on fuse values during boot. This is an **open issue**.
- ❌ **FBPA_CFG1 via host BAR0** — the host cannot write FBPA_CFG1 directly (read-only from host PL0). Only CSB writes (from ROP chain) can modify it.
- ❌ **LMR via host BAR0** — same limitation as FBPA_CFG1.

### VRAM troubleshooting

If VRAM shows 8GB after unlock:

1. Verify FBPA_CFG1 register value via `probe_170hx.py` — if it shows `0x02779000`, the ROP write succeeded but GSP-RM overrides it
2. Check if `vram_config.refresh_interval` needs to be set (see cmpunlocker constants.yaml)
3. The memory unlock may require additional registers not yet identified

### Why compute unlock works but VRAM doesn't

SS0/SS1 (`FEAT_OVR_SM_SPD`) are in the `FEAT_OVR` register block, which GSP-RM does not reconfigure during boot. Once PLM is open and SS0/SS1 are written, they persist.

FBPA_CFG1 (`0x009A0204`) is in the FB (Frame Buffer) controller block. GSP-RM may reinitialize the FB controller during boot, reading fuse values and overriding our CSB write. This is why the register value changes but `nvidia-smi` still shows 8GB.

---

## 11. RECOVERY (If something goes wrong)

### Restore firmware

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

### Full cleanup

```bash
sudo poweroff
# Unplug power for 60 seconds
# Plug back in — GPU in factory state
```

---

## 12. TROUBLESHOOTING

### Problem 1: PLM stays 0xFFFFFF8F (stock)

**Cause:** `sh_size` not updated in ELF section header, OR GPU was not probed after module load.

**Solution:**
1. Use `cmpunlocker/gsp_patch.py` (correctly updates sh_size), NOT a custom patcher
2. Ensure `enumerate_gpu()` is called after `load_module()` (see §8)
3. Delete stale `.cmpunlocker.bak` files before running

### Problem 2: PLM = 0xFFFFFE8E (partially changed)

**Cause:** Host BAR0 write to PLM after ROP chain. Host writes cannot set fused bits (4-6).

**Solution:** Do NOT write PLM from the kernel or host BAR0. The ROP chain handles PLM via CSB. Check that your kernel driver does not have `GPU_REG_WR32(pGpu, 0x00823804, ...)` for CMP170HX.

### Problem 3: PLM = 0xFFFFFFFF but nvidia-smi shows "No devices"

**Cause:** GSP-RM failed to boot. This is expected in Phase 1 (with patched firmware). You need to complete Phase 2 (restore stock firmware + reload).

**Solution:**
```bash
# Complete Phase 2:
sudo rmmod -f nvidia
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/reset
sleep 5
sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock \
        /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin
sudo modprobe nvidia
nvidia-smi
```

### Problem 4: SEC2 mailbox = 0x31 (timeout)

**Cause:** SEC2 is waiting for GSP to respond. This is normal — GSP init takes time.

**Solution:** This is expected behavior. The cmpunlocker pipeline bypasses the SEC2 timeout and continues. Do not try to fix the SEC2 timeout.

### Problem 5: SEC2 mailbox = 0x47 (different error)

**Cause:** The ROP chain executed but SEC2 encountered a different state. This may indicate the payload format is incorrect or the booter version differs.

**Solution:**
1. Verify payload structure matches §5 exactly
2. Verify canary = 0xFACEB13D at DMEM 0x6340
3. Verify frame start = 0xFF48 (not 0xFF3C)
4. Verify tail return = 0x810D (not 0x27FA or 0x1D9F)

### Problem 6: GSP RISC-V not active after ROP chain

**Cause:** On GA100 rev a1, GSP does not start in the same boot cycle as the ROP chain. The ROP chain corrupts SEC2's execution state.

**Solution:** This is expected. Use the two-phase approach:
1. Phase 1: ROP runs → registers unlocked → GSP fails (OK!)
2. FLR → registers persist
3. Phase 2: Stock firmware → GSP boots → CUDA works

### Problem 7: "Failed to initialize NVML: Driver/library version mismatch"

**Cause:** User-space libraries don't match the kernel module version.

**Solution:**
```bash
sudo ln -sf libnvidia-ml.so.580.173.02 /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
sudo ln -sf libcuda.so.580.173.02 /usr/lib/x86_64-linux-gnu/libcuda.so.1
sudo ldconfig
```

### Problem 8: CUDA works but is slow (~46 tok/s instead of ~1000+)

**Cause:** FMA throttle not removed (SS0/SS1 not written).

**Solution:**
```bash
sudo python3 probe_170hx.py | grep SM_SPD
# Should show: 0x88888888
# If 0x00000000, run apply_unlock() from compute.py
```

### Problem 9: TypeError in compute.py

```
TypeError: 'NoneType' object cannot be interpreted as an integer
```

**Cause:** `constants.yaml` missing `plm` section (Issue #1).

**Solution:** Add `plm` section to `common/constants.yaml` (see §8).

### Problem 10: "Compute unlock: PLM not open (0xFFFFFF8F)"

**Cause:** GPU was not probed after module load. The firmware exploit never ran.

**Solution:** Add `enumerate_gpu()` call after `load_module()` in pipeline (see §8), or manually run `nvidia-smi` to trigger GPU init.

---

## 13. ALTERNATIVE APPROACHES (DO NOT WORK)

### ❌ Driver-level exploit (610 driver)

```
Patching _kgspCreateSignatureMemdesc in kernel_gsp.c
Problem: 610 firmware has different structure, GSP-RM init fails
(intrInitInterruptTable_HAL assertion error on GA100 rev a1)
Only 580.x driver works reliably.
```

### ❌ BAR0 writes on COLD BOOT

```
Attempting to write registers via mmap before driver load
Problem: MMIO firewall is active from power-on
All reads return 0xFFFFFFFF (garbage), all writes are dropped
```

### ❌ Custom VBIOS via CH341A

```
Physical reprogramming of SPI flash
Problem: Boot ROM verifies RSA-3072 signature
Private keys are not in any leak
Custom VBIOS = brick
```

### ❌ Hardware Strap Mod (resistor modification)

```
Physical modification of strap resistors on PCB
Problem: Not confirmed by research
FUSE_MEM_LOCKED blocks runtime config
```

### ❌ PCIe Capacitor Mod

```
Soldering 24 capacitors for x4 → x16
Result: ✅ Works (confirmed April 2026)
BUT: only gives 4x bandwidth improvement
Does NOT solve FMA throttle or 64GB VRAM
```

### ❌ TLV-format payload

```
Using TLV header (type=2, length=0x180) at DMEM 0x0800
Problem: The exploit works via ROP chain (stack corruption),
NOT via TLV tag parsing. TLV header causes SEC2 to enter
unexpected code paths.
```

### ❌ 5 writes in ROP chain

```
Including SS0/SS1 in the ROP payload (5 frames instead of 3)
Problem: Unnecessary. SS0/SS1 can be written by host BAR0
after PLM is open. More frames = more failure points.
```

---

## 14. EXPECTED RESULTS

### Stock 580 (without exploit)

```
VRAM: 8GB (FBPA_CFG1 = 0x02449000)
Compute: 1/32 throttle (FEAT_OVR_SM_SPD = 0x00000000)
CUDA: ✅ works (but throttled)
Performance: ~46 tok/s (llama.cpp tg128)
```

### After successful compute unlock (confirmed working)

```
VRAM: 8GB (VRAM unlock may not work — see §10)
Compute: No throttle (FEAT_OVR_SM_SPD = 0x88888888)
CUDA: ✅ works at full speed
Performance: ~1000+ tok/s (llama.cpp pp64)
FP32: ~12 TFLOPS
FP64: ~6.3 TFLOPS
```

### After successful VRAM unlock (experimental)

```
VRAM: 64GB (FBPA_CFG1 = 0x02779000, stable)
Compute: No throttle
CUDA: ✅ works
```

---

## 15. SOURCES & REFERENCES

### Research

- **cmpunlocker (original):** https://github.com/fulracoco/cmpunlocker
  - Original PoC by @amoghmunikote
  - Early draft, has bugs (see Issue #1)

- **cmpunlocker (kinako404 fork):** https://github.com/kinako404/cmpunlocker
  - Improved fork with VRAM configuration
  - Configurable VRAM target via constants.yaml

- **170th Street Research:** https://170th-street.gitbook.io/hx/
  - PCIe Capacitor Mod (confirmed)
  - VBIOS Flash Guide
  - Falcon Security Architecture
  - Open Research Problems

- **Jon Pry (ASU):** "A Canary in the Crypto Mine" (June 2026)
  - Academic description of DMA overflow vulnerability
  - Canary bypass method
  - CSB mailbox protocol

### Tools

- **Ghidra (NSA):** https://github.com/NationalSecurityAgency/ghidra
  - For firmware analysis (gadget hunting)

---

## 16. FINAL CHECKLIST

```
Before starting:
□ Ubuntu 24.04 + kernel ≥6.8
□ 580.173.02 driver installed (NOT 610.x or 595.x)
□ cmpunlocker cloned (kinako404 fork)
□ constants.yaml fixed (plm section added)
□ enumerate_gpu() added to pipeline
□ probe_170hx.py created
□ Firmware backup made (.stock)

Execution:
□ Cold boot (unplug 60s) — for first run only
□ Run pipeline: sudo python3 payload/pipeline.py "0000:01:00.0" "<firmware_path>"
□ Pipeline completes Phase 1 (exploit) → FLR → Phase 2 (stock reload)
□ Check probe_170hx.py → PLM=0xFFFFFFFF, SS0=0x88888888
□ nvidia-smi → GPU visible
□ CUDA test → cuInit returns 0

If something doesn't work:
□ Check sh_size is updated (use cmpunlocker/gsp_patch.py, NOT custom patcher)
□ Check tail_return = 0x810D (NOT 0x27FA G_BOOTER_SUCCESS)
□ Check canary = 0xFACEB13D (NOT zeros)
□ Check frame_start = 0xFF48 (NOT 0xFF3C)
□ Check 3 writes in ROP (NOT 5 — SS0/SS1 via BAR0)
□ Check PLM not overwritten by kernel BAR0 writes
□ Check enumerate_gpu() called after load_module()
□ Delete stale .cmpunlocker.bak files before re-running
```

---

## 17. CONCLUSIONS

### What WORKS:

1. ✅ **Firmware-level DMA Overflow** via cmpunlocker/gsp_patch.py
2. ✅ **ROP chain** (3 writes: FBPA, LMR, PLM) with canary 0xFACEB13D
3. ✅ **Tail return 0x810D** for clean exit to booter main
4. ✅ **FLR Reset** preserves silicon registers in always-on island
5. ✅ **BAR0 writes after FLR** for SS0/SS1 (MMIO firewall partially open)
6. ✅ **Two-phase approach** (Phase 1: exploit → FLR → Phase 2: stock firmware)
7. ✅ **CUDA on 580 driver** with full compute throughput
8. ✅ **Daemon watchdog** for auto-reapply on reboot

### What does NOT work:

1. ❌ **TLV-format payload** (type=2, length=0x180) — exploit uses ROP, not TLV
2. ❌ **5 writes in ROP** (SS0/SS1 should be via BAR0, not ROP)
3. ❌ **G_BOOTER_SUCCESS (0x27FA)** as exit — reinitializes SEC2
4. ❌ **G_SECURE_TEARDOWN (0x7E76)** as exit — kills SEC2
5. ❌ **Driver-level exploit** (610 driver) — GSP-RM init fails
6. ❌ **BAR0 writes on cold boot** — firewall active from power-on
7. ❌ **Custom VBIOS** — RSA-3072 signature check
8. ❌ **VRAM capacity unlock** (64GB/80GB) — GSP-RM may override FBPA_CFG1
9. ❌ **Kernel BAR0 write to PLM** — cannot set fused bits (results in 0xFFFFFF8F)

### Key to success:

**Use the kinako404/cmpunlocker fork with Issue #1 fixes applied:**
- `gsp_patch.py` — correctly updates sh_size
- `build.py` — generates 3-write ROP payload with canary 0xFACEB13D
- `pipeline.py` — two-phase approach (exploit → FLR → stock reload)
- `compute.py` — SS0/SS1 via host BAR0 after FLR
- `constants.yaml` — with `plm` section added (Issue #1 fix)
- `driver.py` — with `enumerate_gpu()` added (Issue #1 fix)

**Driver 580.173.02 only.** Gadget addresses 0x10B9/0x810D verified for this version.

