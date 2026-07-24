# M-Technics ПРИДЕТСЯ ИЗВИНИТЬСЯ

═══════════════════════════════════════════════════════════════════════════
##  FULL UNLOCK OF NVIDIA CMP 170HX — FINAL SOLUTION
  amoghmunikote/cmpunlocker: 6 in-driver kernel patches for 610.43.03
  Result: 64GB VRAM + full compute + working CUDA + persistence
═══════════════════════════════════════════════════════════════════════════

## 1. WHAT WE ARE DOING

We are installing the patched nvidia-open kernel module 610.43.03 with 6 patches
from amoghmunikote (the original author of the exploit). The patches embed
PLM-open + register writes + VRAM configuration DIRECTLY into the driver —
before GSP-RM boots. No need to patch firmware, no need for a ROP chain
in userspace, no need for the FLR dance.

Repository: https://github.com/amoghmunikote/cmpunlocker

## 2. HARDWARE

Card: NVIDIA CMP 170HX
- GPU: GA100 (Ampere) rev a1
- PCI Device ID: 10de:20c2 (8GB physical variant)
- PCI Address: 0000:01:00.0
- BAR0: 16MB @ /sys/bus/pci/devices/0000:01:00.0/resource0
- HBM2e: 5 stacks × 16GB = 64GB physically (Hynix), software-locked to 8GB

Server: Ubuntu 24.04, kernel 7.0.13-3-liquorix-amd64

## 3. UNLOCK PROFILE

8GB Card (0x20C2) → profile=8gb → 64GB unlock:

  FBPA_CFG1 (0x009A0204) = 0x02779000   (64GB HBM2e geometry)
  MMU_LMR   (0x00100CE0) = 0x0000020B   (MMU limit register)
  SS0       (0x0082381C) = 0x88888888   (FMA/IMLA throttle OFF)
  SS1       (0x00823820) = 0x00000008   (IMLA4 throttle OFF)
  FEAT_PLM  (0x00823804) = 0xFFFFFFFF   (FEAT PLM open)
  FBPA_PLM  (0x009A0148) = 0xFFFFFFFF   (FBPA PLM open)
  WPR_PLM   (0x001FA7C4) = 0xFFFFFFFF   (WPR PLM open)
  WPR_CFG   (0x001FA7CC) = 0xFFFFF0FF   (WPR_CFG PLM open)
  FB_BYTES  = 0x1000000000              (64GB for static info)

IMPORTANT: 4 PLM registers are opened (not 1!). This is critical —
only opening ALL FOUR allows GSP-RM to correctly
work with 64GB.

## 4. WHAT THE 6 PATCHES DO

### Patch 0001: sec2-postbl-plm-ss-cfg (19 KB — CORE)
- Weakens WPR2 check (warning instead of return error)
- Opens 4 PLM registers via Booter_Load BEFORE GSP-RM boot
- For each PLM: refill signature payload → kgspExecuteBooterLoad_HAL
  → verify that the register is open
- After PLM-open: GPU_REG_WR32 for SS0/SS1/CFG1/LMR
- Per-device values: 0x20C2 → 64GB, 0x2082 → 40GB
- Rebuild stock signature → GSP-RM boots normally
- Patch fb_length in GspStaticConfigInfo → 64GB

### Patch 0002: booter-verify (3.9 KB)
- Adds SEC2_DEBUG nv_printf to kernel_gsp_tu102.c
- Logs: PLM/SS0/SS1/CFG1/LMR values before and after booter load

### Patch 0003: late-pma (10 KB)
- Extends PMA region for high memory (above 8GB)
- Registers the [8GB, 64GB] region as public
- Disables compression for CMP 170HX

### Patch 0004: bar0-pramin-clamp (841 B)
- Clamps BAR0/PRAMIN offset to stock 8GB if FB > 8GB

### Patch 0005: ce-scrub-workarounds (1.6 KB)
- NV_MMU_PTE_KIND_GENERIC_MEMORY instead of COMPRESSIBLE_DISABLE_PLC
- Do not use VAS for CE memory ops

### Patch 0006: persistent-sw-state (584 B)
- NV_FLAG_PERSISTENT_SW_STATE for 0x20C2/0x2082

## 5. PREPARATION — CLEANUP FROM PREVIOUS EXPERIMENTS

Before installation, you need to completely clean the system of all
previous modifications.

### 5.1. Remove blacklists and diverts

```bash
# Remove nvidia blacklist if it exists
sudo rm -f /etc/modprobe.d/blacklist-nvidia-manual.conf

# Remove all dpkg-diverts
sudo dpkg-divert --list | grep "local diversion" | \
  awk '{print $4}' | xargs -I{} sudo dpkg-divert --rename --remove {} 2>/dev/null

# Restore backup libs if they are in nvidia-lib-bak
if [ -d /usr/lib/x86_64-linux-gnu/nvidia-lib-bak ]; then
    sudo mv /usr/lib/x86_64-linux-gnu/nvidia-lib-bak/* \
             /usr/lib/x86_64-linux-gnu/ 2>/dev/null
    sudo rmdir /usr/lib/x86_64-linux-gnu/nvidia-lib-bak 2>/dev/null
fi
```

### 5.2. Restore stock firmware

```bash
# Restore stock firmware for 610
if [ -f /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.stock ]; then
    sudo cp /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.stock \
            /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin
elif [ -f /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.backup ]; then
    sudo cp /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.backup \
            /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin
elif [ -f /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.bak ]; then
    sudo cp /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.bak \
            /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin
fi

# Restore stock firmware for 580 if it was touched
if [ -f /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock ]; then
    sudo cp /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin.stock \
            /lib/firmware/nvidia/580.173.02/gsp_tu10x.bin
fi

# Verify that firmware is not patched
md5sum /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin
```

### 5.3. Ensure 610.43.03 driver is installed

```bash
# Check that nvidia-open 610.43.03 is installed
dpkg -l | grep nvidia | grep 610

# If not installed — install it
sudo apt install -y nvidia-driver-610-open nvidia-utils-610 \
    libnvidia-compute-610 linux-modules-nvidia-610-$(uname -r)

# If 610 is not in apt — check alternative sources
# or use the .run file from NVIDIA

# Check the version
modinfo nvidia 2>/dev/null | grep version
# Expected: version: 610.43.03 (or 610.43.02)
```

### 5.4. Lock user-space libs to 610

```bash
# Install 610 libraries
sudo apt install --reinstall libnvidia-compute-610 nvidia-utils-610

# Lock symlinks
sudo ln -sf libcuda.so.610.43.03 /usr/lib/x86_64-linux-gnu/libcuda.so.1
sudo ln -sf libnvidia-ml.so.610.43.03 /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
sudo ln -sf libnvidia-ptxjitcompiler.so.610.43.03 \
           /usr/lib/x86_64-linux-gnu/libnvidia-ptxjitcompiler.so.1
sudo ldconfig

# Verify
ls -la /usr/lib/x86_64-linux-gnu/libcuda.so.1
ls -la /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
# Expected: both → ...610.43.03

# Ensure there are no conflicts with 580/595
ls /usr/lib/x86_64-linux-gnu/libcuda.so.5* 2>/dev/null
ls /usr/lib/x86_64-linux-gnu/libcuda.so.6* 2>/dev/null
# If there are 580/595 versions — remove or rename them
```

### 5.5. Stop all nvidia processes

```bash
# Stop daemons
sudo systemctl stop nvidia-persistenced 2>/dev/null
sudo systemctl stop cmpunlocker 2>/dev/null
sudo systemctl disable cmpunlocker 2>/dev/null

# Stop display manager
sudo systemctl stop gdm3 2>/dev/null
sudo systemctl stop sddm 2>/dev/null
sudo systemctl stop lightdm 2>/dev/null

# Kill all processes using the GPU
sudo killall -9 nvidia-smi nvidia-cuda-mps-control Xorg Xwayland 2>/dev/null

# Unload modules
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia 2>/dev/null
sleep 2

# Verify they are unloaded
lsmod | grep nvidia || echo "nvidia modules unloaded"
```

### 5.6. Remove old custom modules

```bash
# Remove old patched modules if they exist
sudo rm -f /lib/modules/$(uname -r)/updates/dkms/nvidia.ko.610
sudo rm -f /lib/modules/$(uname -r)/updates/dkms/nvidia.ko.580

# Remove old cmpunlocker if installed
sudo rm -rf /opt/cmpunlocker
sudo rm -f /etc/systemd/system/cmpunlocker.service
sudo systemctl daemon-reload

# If there is an old nvidia-open-build — do not touch it, but do not use it
# cmpunlocker install.sh will download fresh source code
```

### 5.7. Save a backup of the current state

```bash
# Backup current module
sudo cp /lib/modules/$(uname -r)/updates/dkms/nvidia.ko \
        /home/aboba67/nvidia.ko.before_cmpunlock 2>/dev/null

# Backup firmware
sudo cp /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin \
        /home/aboba67/gsp_tu10x.bin.before_cmpunlock 2>/dev/null

# Backup registers
sudo python3 -c "
import mmap, os, struct
try:
    fd = os.open('/sys/bus/pci/devices/0000:01:00.0/resource0', os.O_RDWR)
    mm = mmap.mmap(fd, 0x1000000, access=mmap.ACCESS_READ)
    with open('/home/aboba67/registers_before.txt', 'w') as f:
        for off, name in [(0x823804,'PLM'),(0x82381C,'SS0'),(0x823820,'SS1'),(0x9A0204,'FBPA'),(0x100CE0,'LMR')]:
            v = struct.unpack_from('<I', mm, off)[0]
            f.write(f'{name} = 0x{v:08X}\n')
    mm.close(); os.close(fd)
    print('Registers saved')
except Exception as e:
    print(f'Cannot read registers: {e}')
" 2>/dev/null

echo "Backups saved to /home/aboba67/"
```

## 6. INSTALLATION of amoghmunikote/cmpunlocker

### 6.1. Clone the repository

```bash
cd /home/aboba67
git clone https://github.com/amoghmunikote/cmpunlocker.git cmpunlocker-amogh
cd cmpunlocker-amogh

# Check the structure
ls -la
ls -la driver/
ls -la driver/patches/
cat driver/VERSION
# Expected: 610.43.03 and 610.43.02

# Check install.sh
head -50 install.sh
```

### 6.2. Check dependencies

```bash
# Python3
python3 --version

# PyYAML
python3 -c "import yaml; print('PyYAML:', yaml.__version__)"

# Kernel headers
ls /usr/src/linux-headers-$(uname -r)/Makefile
# If missing — install
sudo apt install -y linux-headers-$(uname -r)

# Build tools
which gcc make patch
# If missing — install
sudo apt install -y build-essential patch

# Secure Boot must be disabled
mokutil --sb-state 2>/dev/null
# Expected: SecureBoot disabled
# If enabled — disable in BIOS/UEFI
```

### 6.3. Run the installation

```bash
cd /home/aboba67/cmpunlocker-amogh

# Our card is 8GB (0x20C2) → profile=8gb → 64GB unlock
sudo ./install.sh --profile=8gb
```

### What will happen during installation:

1. Root check, GPU detection (10de:20c2)
2. Download open-gpu-kernel-modules-610.43.03.tar.gz from GitHub
3. Extract to /tmp/
4. Apply 6 patches:
   - 0001-sec2-postbl-plm-ss-cfg.patch
   - 0002-booter-verify.patch
   - 0003-late-pma.patch
   - 0004-bar0-pramin-clamp.patch
   - 0005-ce-scrub-workarounds.patch
   - 0006-persistent-sw-state.patch
5. Python script substitutes CFG1=0x02779000, LMR=0x0000020B,
   FB_BYTES=0x1000000000 into kernel_gsp.c (for profile=8gb)
6. make modules (compile nvidia.ko + submodules)
7. Install to /lib/modules/$(uname -r)/updates/cmpunlocker/
8. depmod -a
9. update-initramfs -u (or dracut/mkinitcpio)
10. Attempt hot-reload (might not work — cold reboot required)

### 6.4. If install.sh fails

Possible issues and solutions:

```bash
# Issue: "nvidia-open 610.43.0x not found"
# Solution: install nvidia-driver-610-open
sudo apt install -y nvidia-driver-610-open nvidia-utils-610

# Issue: "kernel headers not found"
# Solution:
sudo apt install -y linux-headers-$(uname -r)

# Issue: "patch failed" 
# Solution: verify that tar.gz downloaded correctly
ls -la /tmp/open-gpu-kernel-modules-610.43.03/
# If directory is empty — download manually:
# https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/610.43.03.tar.gz

# Issue: "make failed" — compiler errors
# Solution: check GCC version
gcc --version
# Requires gcc 13+. If older — update.

# Issue: "module already loaded"
# Solution:
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia 2>/dev/null
sudo rmmod -f nvidia 2>/dev/null
# If module is stuck (refcount -1) — reboot is required

# Issue: "Secure Boot enabled"
# Solution: disable Secure Boot in BIOS/UEFI
# Or sign the module via mokutil
```

### 6.5. Cold Reboot (MANDATORY!)

```bash
sudo systemctl poweroff
```

PHYSICAL ACTIONS:
1. Wait for complete shutdown (SSH will drop, fans will stop)
2. TURN OFF PSU / unplug the power cable
3. WAIT 60 SECONDS (capacitor discharge, WPR2 reset)
4. Plug the power back in
5. Turn on the server with the power button
6. Wait for it to boot, connect via SSH

## 7. VERIFICATION AFTER REBOOT

### 7.1. Verify that the patched module loaded

```bash
# Driver version
cat /proc/driver/nvidia/version | head -1
# Expected: 610.43.03

# Which module is loaded
modinfo nvidia | grep filename
# Expected: /lib/modules/.../updates/cmpunlocker/nvidia.ko

# Verify that the module has the patches
strings /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | grep -c "SEC2_DEBUG"
# Expected: > 0

# Check lsmod
lsmod | grep nvidia
# Expected: nvidia, nvidia_uvm are loaded
```

### 7.2. Check nvidia-smi

```bash
nvidia-smi
```

Expected output:
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 610.43.03   Driver Version: 610.43.03   CUDA Version: 13.0      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA CMP 170HX    Off  | 00000000:01:00.0 Off |                    0 |
| N/A   42C    P0    34W / 250W |      0MiB / 65536MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
```

KEY PARAMETERS:
- Memory: 65536 MiB (64GB) — VRAM is unlocked!
- GPU name: NVIDIA CMP 170HX (or Unknown — this is normal)
- Driver Version: 610.43.03

### 7.3. Check VRAM and SM clock

```bash
# VRAM capacity
nvidia-smi --query-gpu=memory.total --format=csv,noheader
# Expected: 65536 MiB

# SM clock max (without throttle)
nvidia-smi --query-gpu=clocks.max.sm --format=csv,noheader
# Expected: high value (not throttled)
```

### 7.4. Check registers via BAR0

```bash
sudo python3 -c "
import mmap, os, struct

fd = os.open('/sys/bus/pci/devices/0000:01:00.0/resource0', os.O_RDWR)
mm = mmap.mmap(fd, 0x1000000, access=mmap.ACCESS_READ)

checks = [
    (0x00823804, 'FEAT_OVR_PLM',      0xFFFFFFFF),
    (0x0082381C, 'FEAT_OVR_SM_SPD',   0x88888888),
    (0x00823820, 'SS1',               0x00000008),
    (0x009A0204, 'FBPA_CFG1',         0x02779000),
    (0x00100CE0, 'LMR',               0x0000020B),
    (0x009A0148, 'FBPA_PLM',          0xFFFFFFFF),
    (0x001FA7C4, 'WPR_PLM',           0xFFFFFFFF),
    (0x001FA7CC, 'WPR_CFG',           0xFFFFF0FF),
]

print('=== CMP 170HX Register State ===')
all_ok = True
for off, name, expected in checks:
    v = struct.unpack_from('<I', mm, off)[0]
    ok = '✅' if v == expected else '❌'
    if v != expected:
        all_ok = False
    print(f'  {ok} [{off:08X}] {name:20s} = 0x{v:08X} (expected 0x{expected:08X})')

mm.close(); os.close(fd)

print()
if all_ok:
    print('✅ ALL REGISTERS UNLOCKED — SUCCESS!')
else:
    print('❌ Some registers not unlocked — check dmesg')
"
```

### 7.5. Check dmesg SEC2_DEBUG

```bash
sudo dmesg | grep SEC2_DEBUG | tail -40
```

Expected output (key lines):
```
SEC2_DEBUG: PLM FEAT before: 0xFFFFFF8F
SEC2_DEBUG: PLM FEAT after: 0xFFFFFFFF
SEC2_DEBUG: PLM FBPA before: 0xXXXXXXXX
SEC2_DEBUG: PLM FBPA after: 0xFFFFFFFF
SEC2_DEBUG: PLM WPR before: 0xXXXXXXXX
SEC2_DEBUG: PLM WPR after: 0xFFFFFFFF
SEC2_DEBUG: PLM WPR_CFG before: 0xXXXXXXXX
SEC2_DEBUG: PLM WPR_CFG after: 0xFFFFF0FF
SEC2_DEBUG: SS0 = 0x88888888
SEC2_DEBUG: SS1 = 0x00000008
SEC2_DEBUG: CFG1 = 0x02779000
SEC2_DEBUG: LMR = 0x0000020B
```

### 7.6. Check CUDA

```bash
python3 -c "
import ctypes

lib = ctypes.CDLL('libcuda.so.1')
r = lib.cuInit(0)
print(f'cuInit: {r} (0 = success)')

if r == 0:
    count = ctypes.c_int()
    lib.cuDeviceGetCount(ctypes.byref(count))
    print(f'CUDA devices: {count.value}')

    if count.value > 0:
        name = ctypes.create_string_buffer(256)
        lib.cuDeviceGetName(name, 256, 0)
        print(f'Device 0: {name.value.decode()}')

        mem = ctypes.c_size_t()
        lib.cuDeviceTotalMem(ctypes.byref(mem), 0)
        print(f'VRAM: {mem.value / 1024**3:.1f} GB')

        major = ctypes.c_int()
        minor = ctypes.c_int()
        lib.cuDeviceGetAttribute(ctypes.byref(major), 75, 0)
        lib.cuDeviceGetAttribute(ctypes.byref(minor), 76, 0)
        print(f'Compute capability: {major.value}.{minor.value}')
"
```

Expected output:
```
cuInit: 0 (0 = success)
CUDA devices: 1
Device 0: NVIDIA CMP 170HX
VRAM: 64.0 GB
Compute capability: 8.0
```

### 7.7. PyTorch test (if installed)

```bash
python3 -c "
import torch
print(f'PyTorch: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')

if torch.cuda.is_available():
    print(f'Device: {torch.cuda.get_device_name(0)}')
    props = torch.cuda.get_device_properties(0)
    print(f'VRAM: {props.total_memory / 1024**3:.1f} GB')
    print(f'Compute capability: {props.major}.{props.minor}')

    # Quick matmul benchmark
    x = torch.randn(8192, 8192, device='cuda')
    y = torch.randn(8192, 8192, device='cuda')
    torch.cuda.synchronize()

    import time
    t0 = time.time()
    for _ in range(10):
        z = x @ y
    torch.cuda.synchronize()
    t1 = time.time()

    tflops = 10 * 2 * 8192**3 / (t1-t0) / 1e12
    print(f'Matmul TFLOPS: {tflops:.2f}')
    print(f'Time for 10 matmuls: {t1-t0:.3f}s')
"
```

### 7.8. llama.cpp benchmark (if installed)

```bash
# If llama.cpp is available
cd /path/to/llama.cpp
./llama-bench -m /path/to/model.gguf -p 64 -n 128 -t 1

# Expected results (on an unlocked card):
# pp64: ~1000+ tok/s
# tg128: ~100+ tok/s
```

## 8. TROUBLESHOOTING

### If nvidia-smi shows "No devices were found"

```bash
# 1. Verify that the module is loaded
lsmod | grep nvidia
# If not loaded — load it
sudo modprobe nvidia

# 2. Check dmesg for errors
sudo dmesg | grep -iE "NVRM|EXPLOIT|SEC2|GSP|fail|error" | tail -30

# 3. If "RmInitAdapter failed" — check SEC2_DEBUG
sudo dmesg | grep SEC2_DEBUG | tail -20

# 4. If PLM didn't open — another cold boot might be needed
# (sometimes the first boot after installation doesn't work cleanly)
sudo systemctl poweroff
# 60 sec without power → turn on → check again
```

### If VRAM shows 8GB instead of 64GB

```bash
# 1. Check registers
sudo python3 -c "
import mmap, os, struct
fd = os.open('/sys/bus/pci/devices/0000:01:00.0/resource0', os.O_RDWR)
mm = mmap.mmap(fd, 0x1000000, access=mmap.ACCESS_READ)
print(f'FBPA_CFG1: 0x{struct.unpack_from(\"<I\", mm, 0x9A0204)[0]:08X}')
print(f'LMR: 0x{struct.unpack_from(\"<I\", mm, 0x100CE0)[0]:08X}')
mm.close(); os.close(fd)
"

# 2. If FBPA_CFG1 = 0x02779000 but nvidia-smi shows 8GB:
#    GSP-RM didn't receive the fb_length patch
#    Check dmesg for PMA errors:
sudo dmesg | grep -i "pma\|fb_length\|fb region\|static info" | tail -20

# 3. Verify that profile=8gb was selected
cat /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | strings | grep "02779000"
# Should contain 02779000

# 4. If not — rebuild with the correct profile
cd /home/aboba67/cmpunlocker-amogh
sudo ./install.sh --profile=8gb
# + cold reboot
```

### If CUDA doesn't work (cuInit != 0)

```bash
# 1. Check for version mismatch
cat /proc/driver/nvidia/version
nvidia-smi 2>&1 | head -3

# 2. If version mismatch — reinstall libs
sudo apt install --reinstall libnvidia-compute-610 nvidia-utils-610
sudo ldconfig

# 3. Check symlinks
ls -la /usr/lib/x86_64-linux-gnu/libcuda.so.1
# Should be → libcuda.so.610.43.03

# 4. Check /dev/nvidia*
ls -la /dev/nvidia*
# Should be: /dev/nvidia0, /dev/nvidiactl, /dev/nvidia-uvm
```

### If SEC2 timeout (mailbox 0x31 or 0x47)

```bash
# This is normal for the first boot — PLM-open via Booter_Load
# can take some time. Verify that PLM eventually opened:
sudo dmesg | grep "SEC2_DEBUG.*PLM.*after" | tail -10

# If PLM after = 0xFFFFFFFF — everything is OK
# If PLM after = 0xFFFFFF8F — PLM didn't open

# Solution: another cold boot
sudo systemctl poweroff
# 60 sec → turn on → check
```

### If the module doesn't compile

```bash
# 1. Check kernel headers
ls /lib/modules/$(uname -r)/build/Makefile
# If missing:
sudo apt install -y linux-headers-$(uname -r)

# 2. Check build log
cat /tmp/open-gpu-kernel-modules-610.43.03/build.log 2>/dev/null | tail -30

# 3. Possible issues:
#    - Kernel version incompatibility → use a compatible kernel
#    - GCC version → requires GCC 13+
#    - Missing dependencies → sudo apt install build-essential patch

# 4. Try version 610.43.02 if 610.43.03 doesn't build
# (modify driver/VERSION)
```

## 9. ROLLBACK

If something goes wrong:

### 9.1. Quick rollback

```bash
# Stop nvidia
sudo systemctl stop nvidia-persistenced 2>/dev/null
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia 2>/dev/null

# Remove patched module
sudo rm -rf /lib/modules/$(uname -r)/updates/cmpunlocker/
sudo depmod -a

# Restore stock module (if backup exists)
if [ -f /home/aboba67/nvidia.ko.before_cmpunlock ]; then
    sudo cp /home/aboba67/nvidia.ko.before_cmpunlock \
            /lib/modules/$(uname -r)/updates/dkms/nvidia.ko
    sudo depmod -a
fi

# Reinstall stock 610 driver
sudo apt install --reinstall nvidia-driver-610-open

# Cold reboot
sudo systemctl poweroff
# 60 sec → turn on
```

### 9.2. Use remove.sh from cmpunlocker

```bash
cd /home/aboba67/cmpunlocker-amogh
sudo ./remove.sh --yes

# Cold reboot
sudo systemctl poweroff
```

### 9.3. Full rollback to 580 driver

```bash
# Remove 610
sudo apt remove -y nvidia-driver-610-open nvidia-utils-610 libnvidia-compute-610
sudo apt autoremove -y

# Install 580
sudo apt install -y nvidia-driver-580-open nvidia-utils-580 libnvidia-compute-580
sudo ldconfig

# Cold reboot
sudo systemctl poweroff
# 60 sec → turn on
```

## 10. KEY DIFFERENCES FROM PREVIOUS ATTEMPTS

1. NO need to patch firmware (gsp_tu10x.bin)
2. NO need for a ROP chain in userspace
3. NO need for the FLR dance (2× FLR + aggressive_unload)
4. NO need for a watchdog daemon (patches are in the driver, they work at boot)
5. NO need to open only 1 PLM — 4 PLM registers are opened
6. NO need to write SS0/SS1 via BAR0 after FLR
7. Patches embed PLM-open BEFORE GSP-RM boot
8. fb_length patch in static info → GSP-RM sees 64GB
9. PMA region extension → kernel sees high memory
10. Stock signature rebuild → GSP-RM boots normally

## 11. TECHNICAL DETAILS FOR REFERENCE

### Structure of the patched module

```
/lib/modules/$(uname -r)/updates/cmpunlocker/
├── nvidia.ko          (patched — main module with 6 patches)
├── nvidia-modeset.ko  (stock)
├── nvidia-uvm.ko      (stock)
├── nvidia-drm.ko      (stock)
└── nvidia-peermem.ko  (stock)
```

depmod gives priority to updates/cmpunlocker/ > updates/dkms/ > kernel/drivers/
Therefore, the patched nvidia.ko will load instead of the stock one.

### Execution order during boot

```
1. kernel loads nvidia.ko (from updates/cmpunlocker/)
2. nv_pci_probe → detect GPU 0x20C2
3. kgspInitRm_IMPL → IS_GSP_CLIENT=TRUE
4. kgspPrepareForBootstrap_HAL:
   a. kgspPopulateWprMeta_HAL
   b. [PATCHED] For 0x20C2/0x2082:
      - PLM open loop (4 registers × 2 attempts):
        WPR_CFG (0x001FA7CC) → 0xFFFFF0FF
        FBPA    (0x009A0148) → 0xFFFFFFFF
        WPR     (0x001FA7C4) → 0xFFFFFFFF
        FEAT    (0x00823804) → 0xFFFFFFFF
      - Host writes:
        SS0  (0x0082381C) → 0x88888888
        SS1  (0x00823820) → 0x00000008
        CFG1 (0x009A0204) → 0x02779000 (64GB)
        LMR  (0x00100CE0) → 0x0000020B
      - Rebuild stock signature
      - Patch fb_length → 0x1000000000 (64GB)
   c. kgspPrepareForBootstrap (normal)
5. kgspBootstrap_HAL → SEC2 boots → validates stock signature → OK
6. GSP-RM boots normally (registers already unlocked)
7. GSP-RM reads fb_length=64GB → configures memory for 64GB
8. [PATCHED] Late PMA: register high FB region [8GB, 64GB]
9. CUDA ready — 64GB VRAM, full compute
```

### constants.yaml (HEAD version)

```yaml
driver_versions:
  - "610.43.03"
  - "610.43.02"

gpu:
  vendor_id: "10de"
  device_ids:
    - "20c2"
    - "2082"

compute:
  ss0: "0x88888888"
  ss1: "0x00000008"

profiles:
  "8gb":
    stock_mib: 8192
    unlocked_mib: 65536
    cfg1: "0x02779000"
    lmr: "0x0000020B"
    fb_bytes: "0x0000001000000000"
    comment: "8GB physical card → 64GB geometry"
  "10gb":
    stock_mib: 10240
    unlocked_mib: 40960
    cfg1: "0x02669000"
    lmr: "0x0000028A"
    fb_bytes: "0x0000000A00000000"
    comment: "10GB physical card → 40GB geometry"
```

### PLM registers (4 total, all are opened)

```
Register      Address      Value       Purpose
─────────────────────────────────────────────────
FEAT_PLM      0x00823804   0xFFFFFFFF  Feature override PLM
FBPA_PLM      0x009A0148   0xFFFFFFFF  FBPA PLM
WPR_PLM       0x001FA7C4   0xFFFFFFFF  WPR PLM
WPR_CFG_PLM   0x001FA7CC   0xFFFFF0FF  WPR config PLM
```

### Host writes after PLM-open

```
Register      Address      8GB→64GB      10GB→40GB
─────────────────────────────────────────────────────
SS0           0x0082381C   0x88888888    0x88888888
SS1           0x00823820   0x00000008    0x00000008
FBPA_CFG1     0x009A0204   0x02779000    0x02669000
MMU_LMR       0x00100CE0   0x0000020B    0x0000028A
```

### FB static info patch

```
fb_length = 0x1000000000  (64GB)  for 0x20C2
fb_length = 0x0A00000000  (40GB)  for 0x2082
stock fb_length = 0x0200000000  (8GB)
```

## 12. FINAL CHECKLIST

```
Preparation:
□ All blacklists and diverts removed
□ Stock firmware restored
□ nvidia-open 610.43.03 installed
□ User-space libs locked to 610
□ All nvidia processes stopped
□ Old custom modules removed
□ Backups created

Installation:
□ git clone https://github.com/amoghmunikote/cmpunlocker.git
□ sudo ./install.sh --profile=8gb
□ Installation completed without errors
□ Modules installed in /lib/modules/.../updates/cmpunlocker/

Cold Reboot:
□ sudo systemctl poweroff
□ Power cable disconnected for 60 seconds
□ Server turned on
□ Booted successfully

Verification:
□ cat /proc/driver/nvidia/version → 610.43.03
□ nvidia-smi → 65536 MiB VRAM
□ nvidia-smi → GPU visible
□ Registers: PLM=0xFFFFFFFF, SS0=0x88888888, FBPA=0x02779000
□ dmesg | grep SEC2_DEBUG → PLM opened, writes done
□ CUDA cuInit → 0 (success)
□ CUDA VRAM → 64 GB
□ PyTorch benchmark → TFLOPS are adequate

If something doesn't work:
□ Check dmesg for errors
□ Try another cold boot
□ Verify that profile=8gb was selected correctly
□ Ensure there are no conflicts with other nvidia modules
□ Rollback via remove.sh or reinstall stock driver
```

## 13. DECISION-MAKING CONTEXT

- Card CMP 170HX 8GB (0x20C2), GA100 rev a1
- Previous attempts: ROP chain via firmware patch → PLM opened,
  but GSP-RM didn't start in the same boot cycle
- Reason for failure: PLM-open and GSP-RM boot in the same cycle conflict
- amoghmunikote's solution: PLM-open BEFORE GSP-RM boot via in-driver patch
  in kgspPrepareForBootstrap + fb_length patch + PMA extension
- 4 PLM registers (not 1) — critical for VRAM unlock
- Cold reboot is mandatory after installation
- Profile 8gb → 64GB for our card
- Driver 610.43.03 (not 580, not 595)
- Secure Boot must be disabled


═══════════════════════════════════════════════════════════════════════════

EXECUTE STEP BY STEP: 5 (preparation) → 6 (installation) → 6.5 (cold reboot)
→ 7 (verification). Report the result at each step. If something is unclear
or adaptation is needed — ask before executing.

MAIN GOAL: nvidia-smi shows 65536 MiB, CUDA works,
registers ALL UNLOCKED.

═══════════════════════════════════════════════════════════════════════════

# 14. PCIe Gen1 → Gen2 Unlock (Optional)

### Overview

The CMP 170HX ships with PCIe locked to **Gen1 (2.5 GT/s)** — an artificial cap that limits host↔GPU bandwidth to ~1 GB/s. This is independent from the compute and VRAM locks covered in earlier sections.

A community patch (`0007-pcie-gen2.patch` from the [bendy2/cmpunlocker](https://github.com/bendy2/cmpunlocker/tree/combined-multiple-cards-gen2) fork) raises the link to **Gen2 (5.0 GT/s)**, doubling bandwidth to ~2 GB/s — purely in software, no hardware modification required.

| Property | Before | After |
|----------|--------|-------|
| PCIe Link Speed | Gen1 (2.5 GT/s) | Gen2 (5.0 GT/s) |
| Bandwidth (x4) | ~1 GB/s | ~2 GB/s |
| Bandwidth (x16 with solder mod) | ~4 GB/s | ~8 GB/s |
| Method | — | In-driver patch (SEC2 Booter + BAR0) |
| Hardware mod required | — | No |

> **Note:** Gen2 is the software ceiling. Gen3/Gen4 are blocked by an OTP silicon fuse (`FUSE_PCIE_GEN23_DIS = 0x1`) that cannot be bypassed in software. Gen2 bypasses the fuse-gated SerDes by using register-level overrides that re-enable Gen2 signaling without touching the fuse itself.

---

### How It Works

The Gen1 cap is enforced by multiple gates. The patch defeats them through a **combined sequence** of SEC2 Booter writes (for PLM-protected registers) and BAR0 writes (for standard registers), executed **before** GSP-RM boot — same mechanism as the main compute/VRAM unlock.

#### Gates bypassed by the patch:

| Gate | Register | Mechanism | How patch defeats it |
|------|----------|-----------|---------------------|
| XP3G PHY PLM | `0x8E1B0–0x8E1BC` | PLM-locked, blocks PHY rate config | Opened via SEC2 Booter → `0xFFFFFFFF` |
| XVE config PLM | `0x88FE8–0x88FF0` | PLM-locked, blocks XVE overrides | Opened via SEC2 Booter → `0xFFFFFFFF` |
| OPT_GEN23 disable | `0x82057C` | `0x1` = Gen2/3 disabled (fuse shadow) | Written to `0x0` via SEC2 Booter |
| FEAT_OVR ECC PLM | `0x823800` | PLM-locked | Opened via SEC2 Booter → `0xFFFFFFFF` |
| OPTB fuse shadows | `0x8200D0–0x8200F4` | PLM-locked | Opened via SEC2 Booter → `0xFFFFFFFF` |
| CYA_0 Gen2 disable | `0x8C2C0` bit 2 | `DIS_G2` bit set | Cleared via BAR0 |
| LINK_CONFIG_0 max rate | `0x8C040` bits [19:18] | Max rate = Gen1 | Set to `0x2` (Gen2) via BAR0 |
| LINK_CTRL_2 target speed | `0x880A8` bits [3:0] | Target = Gen1 | Set to `0x2` (Gen2) via BAR0 |
| PRIV_MISC_1 Gen2 enable | `0x8841C` bits 11–14 | Gen2 disabled | Bits 11,13 set; bits 12,14 cleared via Booter + late BAR0 |
| PL_LINK_RATE | `0x8C1C0` | Gen1 rate | Set to `0x00240036` (Gen2) via BAR0 |
| VSEC_HIERARCHY | `0x88610` bit 12 | Gates PRIV_MISC_1 reprogram | Bit 12 cleared, bit 0 set via BAR0 |
| VSEC_DEVICE | `0x8860C` bit 0 | Device gate | Bit 0 set via Booter |
| LTSSM retrain | `0x8872C` | — | Triggered with `0x6` (retrain) via BAR0 |

The patch also includes a **late retrain** in `kernel_gsp_tu102.c` — after GSP-RM boot completes, it re-applies the Gen2 register settings and retrains the link again. This ensures GSP-RM doesn't override the Gen2 configuration during its own initialization.

#### Why earlier research failed

The [170th-street research](https://170th-street.gitbook.io/hx/) tested registers **individually** and concluded all paths were closed. The patch succeeds because it:

1. Opens **22 PLM-protected registers** in a single combined sequence (not one at a time)
2. Clears the `DIS_G2` bit in `CYA_0` (a register not covered in earlier research)
3. Sets `MAX_RATE=2` in `LINK_CONFIG_0` (another register not covered)
4. Applies settings **twice**: once pre-GSP (via Booter) and once post-GSP (late retrain)
5. Uses the SEC2 Booter's CSB write path to reach registers that host BAR0 writes cannot touch

---

### Installation

#### Prerequisites

- Working cmpunlocker installation (Section 9) — compute + VRAM unlock must be active
- Driver 610.43.03 patched with patches 0001–0006 from amoghmunikote/cmpunlocker
- CMP 170HX with PCI Device ID `0x20C2` or `0x2082`
- Cold reboot capability (physical power cycle)

#### Option A: Fresh install with bendy2 fork (recommended)

The [bendy2/cmpunlocker](https://github.com/bendy2/cmpunlocker/tree/combined-multiple-cards-gen2) fork includes all 7 patches (original 6 + Gen2). It also adds multi-card support.

```bash
# Remove existing cmpunlocker installation
cd /home/g/cmpunlocker-amogh  # or wherever your current install is
sudo ./remove.sh --yes 2>/dev/null || true

# Clone the bendy2 fork with Gen2 branch
cd /home/g
git clone -b combined-multiple-cards-gen2 \
    https://github.com/bendy2/cmpunlocker.git cmpunlocker-gen2

# Install (same profile as before)
cd cmpunlocker-gen2
sudo ./install.sh --profile=8gb

# Cold reboot (MANDATORY)
sudo systemctl poweroff
# Unplug power cable for 60 seconds
# Plug back in and boot
```

#### Option B: Add Gen2 patch to existing installation

If you already have amoghmunikote/cmpunlocker installed and want to add Gen2:

```bash
# Download the Gen2 patch
cd /home/g/cmpunlocker-amogh/driver/patches/
wget -O 0007-pcie-gen2.patch \
    https://raw.githubusercontent.com/bendy2/cmpunlocker/combined-multiple-cards-gen2/driver/patches/0007-pcie-gen2.patch

# Verify the patch downloaded correctly
head -5 0007-pcie-gen2.patch
# Should show: --- a/src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c

# Rebuild and reinstall
cd /home/g/cmpunlocker-amogh
sudo ./install.sh --profile=8gb

# Cold reboot (MANDATORY)
sudo systemctl poweroff
# Unplug power cable for 60 seconds
# Plug back in and boot
```

> **Important:** The `install.sh` script downloads fresh source from NVIDIA's GitHub and applies all patches in `driver/patches/` in numerical order. Adding `0007-pcie-gen2.patch` to the patches directory is sufficient — it will be applied automatically after patches 0001–0006.

---

### Verification

After cold reboot, verify the PCIe link speed:

#### Check PCIe link speed via lspci

```bash
sudo lspci -d 10de:20c2 -vv | grep -E "LnkCap:|LnkSta:"
```

Expected output (Gen2):
```
LnkCap:  Port #0, Speed 5GT/s, Width x4    ← Capability shows 5GT/s
LnkSta:  Speed 5GT/s, Width x4             ← Current link is 5GT/s (Gen2)
```

If you still see `Speed 2.5GT/s`, the patch did not take effect — check dmesg (below).

#### Check via nvidia-smi

```bash
nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max --format=csv
```

Expected output:
```
2, 2
```

- `current = 2` means link is operating at Gen2
- `max = 2` means Gen2 is the maximum supported

If output shows `1, 1`, the patch did not take effect.

#### Check dmesg for SEC2_DEBUG PCIe logs

```bash
sudo dmesg | grep "SEC2_DEBUG.*PCIe" | head -40
```

Expected output (key lines):
```
SEC2_DEBUG: PCIe pre  CAP=0x... CAP2=0x... ... speed=1
SEC2_DEBUG: PCIe xp3g booter XP3G_PLM(0x8e1b0)=0xffffffff ... status=0x0
SEC2_DEBUG: PCIe xp3g booter OPT_GEN23(0x82057c)=0x00000000 ... status=0x0
SEC2_DEBUG: PCIe xp3g booter PRIV_MISC_1(0x8841c)=0x... ... status=0x0
SEC2_DEBUG: PCIe CYA_0 after clear DIS_G2: 0x... (bit2=0)
SEC2_DEBUG: PCIe LINK_CONFIG_0 after MAX_RATE=2: 0x...
SEC2_DEBUG: PCIe retrain done ... speed=2
SEC2_DEBUG: PCIe PRIV_MISC_1 late pre=0x... post=0x...
SEC2_DEBUG: PCIe XVE_OVR late=0x00000006
```

Key indicators of success:
- `OPT_GEN23` = `0x00000000` (was `0x1`)
- `DIS_G2` bit2 = `0` (was `1`)
- `speed=2` in retrain done line
- No `FAILED to set` messages

#### Bandwidth test

```bash
# Simple bandwidth test using CUDA
python3 -c "
import ctypes
import time

lib = ctypes.CDLL('libcuda.so.1')
lib.cuInit(0)

# Get device
dev = ctypes.c_int()
lib.cuDeviceGet(ctypes.byref(dev), 0)

# Allocate host and device buffers
size = 256 * 1024 * 1024  # 256 MB
host_buf = (ctypes.c_char * size)()
dev_buf = ctypes.c_void_p()

lib.cuMemAlloc(ctypes.byref(dev_buf), size)

# Warm up
lib.cuMemcpyHtoD(dev_buf, host_buf, size)
lib.cuDeviceSynchronize()

# Time multiple transfers
import time
n = 20
t0 = time.time()
for _ in range(n):
    lib.cuMemcpyHtoD(dev_buf, host_buf, size)
lib.cuDeviceSynchronize()
t1 = time.time()

bandwidth = n * size / (t1 - t0) / (1024**3)
print(f'H2D bandwidth: {bandwidth:.2f} GB/s')
print(f'Expected Gen1 x4: ~0.9 GB/s')
print(f'Expected Gen2 x4: ~1.8 GB/s')

lib.cuMemFree(dev_buf)
"
```

---

### Expected Results

| Metric | Gen1 x4 (before) | Gen2 x4 (after) | Improvement |
|--------|-------------------|------------------|-------------|
| Link speed | 2.5 GT/s | 5.0 GT/s | 2× |
| H2D bandwidth | ~0.9 GB/s | ~1.8 GB/s | 2× |
| D2H bandwidth | ~0.9 GB/s | ~1.8 GB/s | 2× |
| Model loading (70B) | ~70 sec | ~35 sec | 2× faster |
| CUDA kernel launch | ~50 µs | ~35 µs | ~1.4× faster |

#### With x16 solder mod (24 capacitors)

| Metric | Gen1 x4 (stock) | Gen2 x16 (Gen2 patch + solder) | Improvement |
|--------|------------------|--------------------------------|-------------|
| Link speed | 2.5 GT/s | 5.0 GT/s | 2× |
| Link width | x4 | x16 | 4× |
| H2D bandwidth | ~0.9 GB/s | ~7.5 GB/s | **8×** |
| D2H bandwidth | ~0.9 GB/s | ~7.5 GB/s | **8×** |
| Model loading (70B) | ~70 sec | ~9 sec | **8× faster** |

---

### Troubleshooting

#### Link stays at Gen1 after patch

```bash
# 1. Check if patch was applied during build
strings /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | grep "PCIe pre"
# Should show: SEC2_DEBUG: PCIe pre  CAP=
# If empty — patch was not applied, rebuild

# 2. Check dmesg for errors
sudo dmesg | grep "SEC2_DEBUG.*PCIe.*FAILED" | tail -10
# If FAILED messages appear — SEC2 Booter could not write to PLM registers

# 3. Check OPT_GEN23 value
sudo dmesg | grep "OPT_GEN23" | tail -5
# Should show 0x00000000 after patch
# If still 0x1 — SEC2 Booter write failed

# 4. Try another cold reboot (sometimes first boot doesn't take)
sudo systemctl poweroff
# 60 seconds without power → boot → check again
```

#### Link trains at Gen2 but drops back to Gen1

This can happen if the host root port or PCIe slot is not Gen2-capable, or if there's signal integrity issues.

```bash
# Check host root port capability
sudo lspci -s 00:01.0 -vv | grep LnkCap
# Should show Speed 5GT/s or higher

# Check for PCIe errors
sudo dmesg | grep -i "AER\|pcie\|link\|training" | tail -20

# If signal integrity issues — try a different PCIe slot or shorter riser cable
```

#### nvidia-smi shows gen.current=2 but lspci shows 2.5GT/s

This is a reporting mismatch. Trust `lspci` — it reads the actual hardware link state:

```bash
# lspci is authoritative
sudo lspci -d 10de:20c2 -vv | grep LnkSta
# If LnkSta shows Speed 5GT/s → Gen2 is working
# If LnkSta shows Speed 2.5GT/s → still Gen1
```

---

### Limitations

1. **Gen2 is the software ceiling.** Gen3/Gen4 require an OTP fuse (`FUSE_PCIE_GEN23_DIS`) to be cleared, which is a physical silicon modification. No software path exists.

2. **x4 width is not changed by this patch.** The PCIe width (x4 vs x16) is a separate hardware limitation. To get x16, solder 24 × 0402 0.22µF AC-coupling capacitors (C1100–C1350) on the PCB.

3. **Host root port must support Gen2.** Most modern motherboards (AMD Raphael/Genoa, Intel 12th gen+) support Gen2 natively. If the host only supports Gen1, the link will stay at Gen1 regardless of the GPU's capability.

4. **PCIe riser cables.** Some cheap riser cables are Gen1-only. Use a Gen2 or Gen4 rated riser if testing with a riser.

5. **Stability.** Gen2 at x4 is well within GA100's native capabilities. No stability issues expected. If instability occurs, check signal integrity (riser, slot, board).

---

### Comparison with 170th-street research

The [170th-street PCIe field manual](https://170th-street.gitbook.io/hx/) concluded that PCIe Gen1 was an unbreakable hardware wall. Their research was thorough and correct about Gen3/Gen4 being fuse-locked. However, they missed the Gen2 path because:

| What 170th-street tested | What the patch does differently |
|---------------------------|--------------------------------|
| Individual register writes (one at a time) | Combined 22-register sequence in one pass |
| Did not clear `CYA_0` bit 2 (`DIS_G2`) | Clears `DIS_G2` — critical Gen2 enable bit |
| Did not set `LINK_CONFIG_0` `MAX_RATE=2` | Sets `MAX_RATE=2` — programs Gen2 as maximum |
| Tested host BAR0 writes only | Uses SEC2 Booter CSB path for PLM-protected registers |
| No late retrain after GSP-RM boot | Re-applies Gen2 settings post-GSP and retrains |
| Concluded `OPT_GEN23` is hard RO | SEC2 Booter successfully writes `0x0` to `OPT_GEN23` |

The key insight: **Gen2 does not require clearing the OTP fuse.** The fuse (`FUSE_PCIE_GEN23_DIS`) gates Gen3 SerDes, but Gen2 can be enabled through register overrides (`CYA_0`, `OPT_GEN23`, `PRIV_MISC_1`) that are upstream of the fuse gate. The patch exploits this by opening the PLM on all relevant registers via SEC2 Booter, then programming Gen2 configuration before GSP-RM boot.

---

### Technical Reference: All Registers Modified by Patch 0007

#### SEC2 Booter writes (PLM-protected, via `kgspExecuteBooterLoad_HAL`):

| Address | Name | Value | Purpose |
|---------|------|-------|---------|
| `0x0008E1B0` | XP3G_PLM | `0xFFFFFFFF` | Open PHY PLM |
| `0x0008E1B4` | XP3G_PLM4 | `0xFFFFFFFF` | Open PHY PLM |
| `0x0008E1B8` | XP3G_PLM8 | `0xFFFFFFFF` | Open PHY PLM |
| `0x0008E1BC` | XP3G_PLMC | `0xFFFFFFFF` | Open PHY PLM |
| `0x00088FE8` | XVE_D0 | `0xFFFFFFFF` | Open XVE config PLM |
| `0x00088FEC` | XVE_D4 | `0xFFFFFFFF` | Open XVE config PLM |
| `0x00088FF0` | XVE_D8 | `0xFFFFFFFF` | Open XVE config PLM |
| `0x008200D0`–`0x008200F4` | OPTB_D0–F4 (9 regs) | `0xFFFFFFFF` | Open OPTB fuse shadow PLM |
| `0x00823800` | FEAT_OVR_ECC_PLM | `0xFFFFFFFF` | Open FEAT_OVR ECC PLM |
| `0x0082057C` | OPT_GEN23 | `0x00000000` | Clear Gen2/3 disable (was `0x1`) |
| `0x0008E120` | XP3G_VAL0 | `0x00000000` | PHY rate override value 0 |
| `0x0008E110` | XP3G_OVR0 | `0x00000001` | PHY rate override enable 0 |
| `0x0008E12C` | XP3G_VAL3 | `0x00200000` | PHY rate override value 3 |
| `0x0008E11C` | XP3G_OVR3 | `0x00000004` | PHY rate override enable 3 |
| `0x0008860C` | VSEC_DEVICE | `current \| (1<<0)` | Enable device VSEC |
| `0x0008841C` | PRIV_MISC_1 | `set bits 11,13; clear bits 12,14` | Gen2 enable |

#### Direct BAR0 writes (host, after Booter PLM-open):

| Address | Name | Value | Purpose |
|---------|------|-------|---------|
| `0x00088610` | VSEC_HIERARCHY | `clear bit 12, set bit 0` | Gate PRIV_MISC_1 reprogram |
| `0x000880A8` | LINK_CTRL_2 | `bits[3:0]=0x2, bits[19:16]=0xF` | Target speed = Gen2 |
| `0x0008C2C0` | CYA_0 | `clear bit 2` | Clear `DIS_G2` (Gen2 disable) |
| `0x0008C040` | LINK_CONFIG_0 | `bits[19:18]=0x2` | `MAX_RATE = Gen2` |
| `0x0008C1C0` | PL_LINK_RATE | `0x00240036` | PHY link rate for Gen2 |
| `0x0008872C` | LTSSM | `0x00000006` | Trigger link retrain |

#### Late writes (post-GSP-RM boot, in `kernel_gsp_tu102.c`):

| Address | Name | Value | Purpose |
|---------|------|-------|---------|
| `0x0008841C` | PRIV_MISC_1 | `set bits 11,13; clear bits 12,14` | Re-apply Gen2 enable |
| `0x0008C2C0` | CYA_0 | `clear bit 2` | Re-clear `DIS_G2` |
| `0x0008C040` | LINK_CONFIG_0 | `bits[19:18]=0x2` | Re-set `MAX_RATE = Gen2` |
| `0x0008872C` | LTSSM | `0x00000006` | Re-trigger link retrain |

