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
