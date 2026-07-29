# TASK: Unlock PCIe Gen2 on NVIDIA CMP 170HX (GA100) — Full Setup Prompt

This is a self-contained prompt for an AI agent to configure PCIe Gen2 unlock on a Linux machine with an NVIDIA CMP 170HX mining card. It includes ALL steps discovered through real-world testing, including the critical timing fix that makes Gen2 actually work.

---

## System Requirements

- NVIDIA CMP 170HX (PCI ID `10de:20c2` for 8GB or `10de:2082` for 10GB)
- Linux (x86-64), systemd
- nvidia-open driver 610.43.03 or 610.43.02 installed
- Kernel headers matching running kernel: `linux-headers-$(uname -r)`
- Secure Boot disabled
- Root access
- Network access (downloads source + patches)
- `pciutils` (setpci, lspci) installed

## What Gets Unlocked

| Feature | Before | After |
|---|---|---|
| Memory | 8 GB | 64 GB |
| SM Clock | 1005 MHz | 1695 MHz |
| PCIe Link Speed | Gen1 (2.5 GT/s) | Gen2 (5.0 GT/s) |
| PCIe Bandwidth x4 | ~0.78 GB/s | ~1.56 GB/s |

---

## Step-by-step Instructions

### STEP 1: Clone cmpunlocker from amoghmunikote

```bash
cd ~ && rm -rf cmpunlocker-amogh
git clone https://github.com/amoghmunikote/cmpunlocker.git cmpunlocker-amogh
```

This repo has patches 0001-0006 (memory + SM unlock). We need to add the Gen2 patches.

### STEP 2: Download the Gen2 patches from bendy2

```bash
cd ~/cmpunlocker-amogh/driver/patches/
wget -O 0007-pcie-gen2.patch https://raw.githubusercontent.com/bendy2/cmpunlocker/combined-multiple-cards-gen2/driver/patches/0007-pcie-gen2.patch
wget -O 0008-pcie-gen2-probe-retrain.patch https://raw.githubusercontent.com/bendy2/cmpunlocker/combined-multiple-cards-gen2/driver/patches/0008-pcie-gen2-probe-retrain.patch
```

### STEP 3: Verify all 8 patches are present

```bash
ls ~/cmpunlocker-amogh/driver/patches/
```

Expected output:
```
0001-sec2-postbl-plm-ss-cfg.patch
0002-booter-verify.patch
0003-late-pma.patch
0004-bar0-pramin-clamp.patch
0005-ce-scrub-workarounds.patch
0006-persistent-sw-state.patch
0007-pcie-gen2.patch
0008-pcie-gen2-probe-retrain.patch
```

### STEP 4: Build and install patched modules

```bash
cd ~/cmpunlocker-amogh && sudo ./install.sh --profile=8gb
```

This downloads open-gpu-kernel-modules source from NVIDIA, applies all 8 patches, builds, and installs patched kernel modules into `/lib/modules/$(uname -r)/updates/cmpunlocker/`.

Verify the Gen2 patch was applied:
```bash
strings /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | grep -c "SEC2_DEBUG: PCIe"
# Should output a number > 0 (typically ~19)
```

### STEP 5: Install the Gen2 hammer service from studebaker8

**THIS IS THE CRITICAL STEP.** Without it, Gen2 will NOT work.

The driver patch (0007) opens a Gen2 window during GSP bootstrap (~8-14 seconds after boot), but patch 0008 runs too late to catch it. The hammer service from `studebaker8/cmp170hx-gen2` fires root-port retrains every 50ms across the entire window, ensuring Gen2 is actually trained.

```bash
cd /tmp && git clone --depth 1 https://github.com/studebaker8/cmp170hx-gen2.git
cd /tmp/cmp170hx-gen2
sudo bash cmp170hx-gen2-setup.sh --install
```

This installs:
- `/usr/local/sbin/cmp170hx-gen2-hammer` — the retrain loop script
- `/etc/systemd/system/cmp170hx-gen2.service` — systemd service that runs at early boot

The service fires at `sysinit.target` (very early boot) and hammers retrains until the link reports Gen2. On reference hardware it succeeds around iteration 30 (~1.5 seconds of hammering).

### STEP 6: Cold reboot (MANDATORY)

```bash
sudo systemctl poweroff
```

**CRITICAL:** You MUST unplug the power cable for at least 60 seconds before plugging back in. A normal `reboot` does NOT work — the SEC2 controller needs a full power cycle to reset its state.

After plugging back in, let the system boot normally.

### STEP 7: Verify after boot

Check the hammer log FIRST:
```bash
cat /var/log/cmp170hx-gen2.log
```

Expected output on success:
```
[timestamp] start (max_iter=600 interval=0.05s)
[timestamp] 0000:01:00.0: hammering retrains via root port 0000:00:01.0, target Gen2
[timestamp] 0000:01:00.0: *** GEN2 TRAINED *** iter=XX LnkSta=0x1042 LnkCap2=0x00000006
[timestamp] end
```

The key line is `*** GEN2 TRAINED ***`. If you see `gave up after 600 iterations`, Gen2 failed.

Check sysfs (most reliable):
```bash
cat /sys/bus/pci/devices/0000:01:00.0/current_link_speed
# Expected: 5.0 GT/s PCIe
```

Check lspci:
```bash
sudo lspci -vvv -s 01:00.0 | grep "LnkSta:"
# Expected: Speed 5GT/s, Width x4 (downgraded)
# Note: "overdriven" is also valid
```

Check nvidia-smi (may show cached value):
```bash
nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max --format=csv
# May show 1,1 due to driver cache — trust sysfs and lspci instead
```

Run bandwidth test:
```python
python3 -c "
import ctypes, time
lib = ctypes.CDLL('libcuda.so.1')
lib.cuInit(0)
dev = ctypes.c_int()
lib.cuDeviceGet(ctypes.byref(dev), 0)
ctx = ctypes.c_void_p()
lib.cuCtxCreate(ctypes.byref(ctx), 0, dev.value)
size = 256*1024*1024
hbuf = (ctypes.c_char * size)()
dbuf = ctypes.c_void_p()
lib.cuMemAlloc(ctypes.byref(dbuf), size)
lib.cuMemcpyHtoD(dbuf, hbuf, size)
lib.cuCtxSynchronize()
n = 30
t0 = time.perf_counter()
for _ in range(n): lib.cuMemcpyHtoD(dbuf, hbuf, size)
lib.cuCtxSynchronize()
h2d = n*size/(time.perf_counter()-t0)/(1024**3)
t0 = time.perf_counter()
for _ in range(n): lib.cuMemcpyDtoH(hbuf, dbuf, size)
lib.cuCtxSynchronize()
d2h = n*size/(time.perf_counter()-t0)/(1024**3)
lib.cuMemFree(dbuf); lib.cuCtxDestroy(ctx)
print(f'H2D: {h2d:.2f} GB/s | D2H: {d2h:.2f} GB/s')
print(f'Expected Gen1 x4: ~0.8 GB/s | Expected Gen2 x4: ~1.6 GB/s')
"
```

Expected: ~1.5-1.6 GB/s for both H2D and D2H (Gen2 x4).
If you see ~0.8 GB/s, Gen2 failed.

---

## Troubleshooting

### Gen2 not working after reboot

1. Check the hammer log: `cat /var/log/cmp170hx-gen2.log`
   - If no log file exists, the service didn't run. Check: `systemctl status cmp170hx-gen2.service`
   - If log shows `gave up after 600 iterations`, the Gen2 window didn't open. Verify the driver has patch 0007:
     ```bash
     strings /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | grep "SEC2_DEBUG: PCIe"
     ```

2. Verify the cmpunlocker driver is loaded:
   ```bash
   cat /proc/driver/nvidia/version
   # Should NOT say "dvs-builder" (that's the stock driver)
   lsmod | grep nvidia
   ```

3. Check dmesg for SEC2_DEBUG errors:
   ```bash
   sudo dmesg | grep "SEC2_DEBUG.*PCIe.*FAILED"
   # If you see FAILED to set OPT_GEN23 or FAILED VSEC_DEVICE,
   # the SEC2 Booter couldn't write to PLM registers on this revision.
   # Some CMP 170HX PCB revisions have stricter PLM protection.
   ```

4. Make sure you did a proper cold reboot (power cable unplugged for 60+ seconds).

### PCIe stays at Gen1 on some revisions

Some CMP 170HX PCB revisions have OPT_GEN23 and VSEC_DEVICE registers permanently locked at the SEC2 controller level. The symptoms are:
- `FAILED to set OPT_GEN23` in dmesg (value stays 0x1)
- `FAILED VSEC_DEVICE booter` in dmesg
- All other registers write successfully

On these revisions, PCIe Gen2 cannot be unlocked in software. Memory (64 GB) and SM (1695 MHz) will still work.

The hammer service from studebaker8 has been verified working on:
- CMP 170HX 8GB (10de:20c2), VBIOS 92.00.6D.00.0A, AMD B650M host
- CMP 170HX 10GB (10de:2082), VBIOS 92.00.66.00.02

### nvidia-smi shows Gen1 but sysfs shows Gen2

nvidia-smi caches the link generation from driver probe time. After the hammer trains Gen2, nvidia-smi may still show 1. This is a reporting issue only — the actual link is Gen2. Trust:
- `cat /sys/bus/pci/devices/BDF/current_link_speed` (most reliable)
- `lspci -vvv -s BDF | grep LnkSta:` (second most reliable)
- Bandwidth test (ground truth)

---

## How It Works (Technical Background)

The Gen2 unlock requires TWO components working together:

1. **Driver patch (0007):** Opens internal GPU registers during GSP bootstrap (~8-14s after boot). This makes the endpoint transiently advertise `LnkCap2=0x06` (Gen2 support). The window is brief — after GSP boot completes, the registers revert.

2. **Hammer service (studebaker8):** Fires root-port retrains every 50ms from early boot, across the entire Gen2 window. Userspace has access to the upstream PCIe bridge (which the driver patch cannot reach). Once the link trains Gen2 inside the window, it STAYS trained after the window closes.

Previous attempts failed because:
- Patch 0007 alone can only poke LTSSM, not do a real retrain (no bridge access from RM context)
- Patch 0008 does a proper retrain but runs ~3 seconds too late (after the window closes)
- Userspace retrain scripts ran at end of boot (window long closed)

The hammer catches the window because it starts at `sysinit.target` and hammers continuously.

---

## Rollback

```bash
cd ~/cmpunlocker-amogh && sudo ./remove.sh --yes
sudo systemctl disable --now cmp170hx-gen2.service
sudo rm -f /etc/systemd/system/cmp170hx-gen2.service /usr/local/sbin/cmp170hx-gen2-hammer
sudo systemctl daemon-reload
# Cold reboot again
```

---

## Summary Checklist

- [ ] Clone amoghmunikote/cmpunlocker
- [ ] Download 0007-pcie-gen2.patch and 0008-pcie-gen2-probe-retrain.patch from bendy2
- [ ] Verify all 8 patches present
- [ ] Run `sudo ./install.sh --profile=8gb`
- [ ] Clone studebaker8/cmp170hx-gen2 and run `sudo bash cmp170hx-gen2-setup.sh --install`
- [ ] Cold reboot (power off + unplug cable 60s + power on)
- [ ] Check `/var/log/cmp170hx-gen2.log` for `*** GEN2 TRAINED ***`
- [ ] Verify: `cat /sys/bus/pci/devices/BDF/current_link_speed` shows `5.0 GT/s`
- [ ] Bandwidth test shows ~1.5-1.6 GB/s
