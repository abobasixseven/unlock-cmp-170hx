═══════════════════════════════════════════════════════════════════════════
  CMP 90HX (GA102) COMPUTE UNLOCK — PATCHEX ДЛЯ nvidia-open 610.43.03
  Аналог amoghmunikote/cmpunlocker patch 0001, адаптированный для GA102
═══════════════════════════════════════════════════════════════════════════

## 1. ЧТО ДЕЛАЕТ ПАТЧ

Разблокирует compute throttle на NVIDIA CMP 90HX (GA102):
- FMA/IMLA throttle OFF (1/32 → full speed)
- Gfx speed select override
- SM speed select override (SS0 + SS1)


## 2. ОБОРУДОВАНИЕ

Карта: NVIDIA CMP 90HX
- GPU: GA102 (Ampere)
- PCI Device ID: 10de:2684
- Memory: GDDR6x (не заблокирована, compute-only unlock)

## 3. ЦЕЛЕВЫЕ РЕГИСТРЫ (GA102)

### PLM регистры (открываются через SEC2 Booter):

```
Address       Name                                      Value       Purpose
──────────────────────────────────────────────────────────────────────────
0x00823804    FEAT_OVR_SM_SPEED_SELECT_PLM              0xFFFFF3FF  Open SM speed PLM
0x00823B04    FEAT_OVR_GFX_SPEED_SELECT_PLM             0xFFFFF3FF  Open Gfx speed PLM
```

### Compute регистры (пишутся через GPU_REG_WR32 после PLM-open):

```
Address       Name                                      Value       Purpose
──────────────────────────────────────────────────────────────────────────
0x0082381C    FEAT_OVR_SM_SPEED_SELECT (SS0)            0x88888888  FMA/IMLA throttle OFF
0x00823820    FEAT_OVR_SM_SPEED_SELECT_1 (SS1)          0x00000008  IMLA4 throttle OFF
0x00823830    FEAT_OVR_GFX_SPEED_SELECT                 0x00000004  Gfx speed override
```

### Сравнение с CMP 170HX (GA100):

```
                    CMP 170HX (GA100)          CMP 90HX (GA102)
PLM FEAT            0x823804 → 0xFFFFFFFF      0x823804 → 0xFFFFF3FF
PLM GFX             (не используется)           0x823B04 → 0xFFFFF3FF (НОВЫЙ!)
SS0                 0x82381C → 0x88888888      0x82381C → 0x88888888 (тот же)
SS1                 0x823820 → 0x00000008      0x823820 → 0x00000008 (тот же)
GFX_SPEED           (не используется)           0x823830 → 0x00000004 (НОВЫЙ!)
FBPA_CFG1           0x9A0204 → 0x02779000      (НЕ НУЖЕН — GDDR6)
LMR                 0x100CE0 → 0x0000020B       (НЕ НУЖЕН)
fb_length patch     да (64GB)                   (НЕ НУЖЕН)
PMA extension       да                          (НЕ НУЖЕН)
```

Ключевые отличия GA102:
1. PLM value = 0xFFFFF3FF (не 0xFFFFFFFF) — другие биты fused
2. Дополнительный PLM регистр: 0x823B04 (GFX speed select)
3. Дополнительный compute регистр: 0x823830 (GFX speed select)
4. Нет VRAM unlock — не нужен FBPA/LMR/fb_length/PMA

## 4. ФАЙЛ ПАТЧА

Создай файл `0001-cmp90-compute-unlock.patch` со следующим содержимым:

```diff
--- a/src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
+++ b/src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
@@ -4942,6 +4942,120 @@
                      devId);
         }
 
+        /*
+         * CMP 90HX (GA102) compute unlock.
+         * Opens PLM on FEAT_OVR + GFX_SPEED_SELECT, then writes
+         * SM_SPEED_SELECT (SS0/SS1) and GFX_SPEED_SELECT overrides.
+         *
+         * Device ID: 0x2684 (CMP 90HX)
+         * Compute only — no VRAM unlock needed (GDDR6, not HBM2e).
+         */
+        if (devId == 0x2684)
+        {
+            #define CMP90_PCI_DEVICE_ID                     0x2684
+            #define CMP90_SEC2_POSTBL_SIGNATURE_SIZE        0x0000f800ULL
+            #define CMP90_SEC2_POSTBL_FILL_DWORD            0x000004a7U
+
+            /* PLM registers to open via SEC2 Booter */
+            static const struct { NvU32 addr; NvU32 value; const char *name; } cmp90PlmTable[] = {
+                { 0x00823804U, 0xFFFFF3FFU, "FEAT_OVR_SM_SPD_PLM"   },
+                { 0x00823B04U, 0xFFFFF3FFU, "FEAT_OVR_GFX_SPD_PLM"  },
+            };
+
+            /* Compute registers to write via GPU_REG_WR32 after PLM-open */
+            static const struct { NvU32 addr; NvU32 value; const char *name; } cmp90WriteTable[] = {
+                { 0x0082381CU, 0x88888888U, "FEAT_OVR_SM_SPD (SS0)"      },
+                { 0x00823820U, 0x00000008U, "FEAT_OVR_SM_SPD_1 (SS1)"    },
+                { 0x00823830U, 0x00000004U, "FEAT_OVR_GFX_SPD"           },
+            };
+
+            NvU32 cmp90i, cmp90attempt;
+            NV_STATUS cmp90Status;
+            const NvU32 cmp90PlmCount = sizeof(cmp90PlmTable) / sizeof(cmp90PlmTable[0]);
+            const NvU32 cmp90WriteCount = sizeof(cmp90WriteTable) / sizeof(cmp90WriteTable[0]);
+
+            /* Save WPR2 bounds for Booter calls */
+            NvU32 cmp90Wpr2Lo = GPU_REG_RD32(pGpu, 0x001fa824U);
+            NvU32 cmp90Wpr2Hi = GPU_REG_RD32(pGpu, 0x001fa828U);
+
+            NV_PRINTF(LEVEL_ERROR,
+                      "CMP90_DEBUG: Starting compute unlock for devId=0x%04x\n",
+                      devId);
+
+            /* Phase 1: Open PLM registers via SEC2 Booter */
+            for (cmp90i = 0; cmp90i < cmp90PlmCount; cmp90i++)
+            {
+                NvBool wrote = NV_FALSE;
+                NvU32 beforeVal = GPU_REG_RD32(pGpu, cmp90PlmTable[cmp90i].addr);
+
+                NV_PRINTF(LEVEL_ERROR,
+                          "CMP90_DEBUG: PLM[%u] %s(0x%x) before=0x%08x target=0x%08x\n",
+                          cmp90i, cmp90PlmTable[cmp90i].name,
+                          cmp90PlmTable[cmp90i].addr, beforeVal,
+                          cmp90PlmTable[cmp90i].value);
+
+                for (cmp90attempt = 0; cmp90attempt < 2 && !wrote; cmp90attempt++)
+                {
+                    /* Restore WPR2 bounds before Booter call */
+                    GPU_REG_WR32(pGpu, 0x001fa824U, cmp90Wpr2Lo);
+                    GPU_REG_WR32(pGpu, 0x001fa828U, cmp90Wpr2Hi);
+
+                    cmp90Status = kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp,
+                        cmp90PlmTable[cmp90i].addr, cmp90PlmTable[cmp90i].value);
+                    if (cmp90Status != NV_OK)
+                    {
+                        NV_PRINTF(LEVEL_ERROR,
+                                  "CMP90_DEBUG: PLM[%u] refill failed status=0x%x\n",
+                                  cmp90i, cmp90Status);
+                        continue;
+                    }
+
+                    cmp90Status = kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
+                        memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));
+
+                    NvU32 afterVal = GPU_REG_RD32(pGpu, cmp90PlmTable[cmp90i].addr);
+                    NV_PRINTF(LEVEL_ERROR,
+                              "CMP90_DEBUG: PLM[%u] %s attempt=%u status=0x%x "
+                              "before=0x%08x after=0x%08x target=0x%08x\n",
+                              cmp90i, cmp90PlmTable[cmp90i].name,
+                              cmp90attempt, cmp90Status,
+                              beforeVal, afterVal,
+                              cmp90PlmTable[cmp90i].value);
+
+                    if (afterVal == cmp90PlmTable[cmp90i].value)
+                        wrote = NV_TRUE;
+                }
+
+                if (!wrote)
+                    NV_PRINTF(LEVEL_ERROR,
+                              "CMP90_DEBUG: PLM[%u] %s FAILED to set\n",
+                              cmp90i, cmp90PlmTable[cmp90i].name);
+            }
+
+            /* Phase 2: Write compute registers via host BAR0 */
+            for (cmp90i = 0; cmp90i < cmp90WriteCount; cmp90i++)
+            {
+                GPU_REG_WR32(pGpu, cmp90WriteTable[cmp90i].addr,
+                             cmp90WriteTable[cmp90i].value);
+                NvU32 rdBack = GPU_REG_RD32(pGpu, cmp90WriteTable[cmp90i].addr);
+                NV_PRINTF(LEVEL_ERROR,
+                          "CMP90_DEBUG: WRITE %s(0x%x) = 0x%08x (readback=0x%08x %s)\n",
+                          cmp90WriteTable[cmp90i].name,
+                          cmp90WriteTable[cmp90i].addr,
+                          cmp90WriteTable[cmp90i].value,
+                          rdBack,
+                          rdBack == cmp90WriteTable[cmp90i].value ? "OK" : "MISMATCH");
+            }
+
+            /* Restore WPR2 bounds */
+            GPU_REG_WR32(pGpu, 0x001fa824U, cmp90Wpr2Lo);
+            GPU_REG_WR32(pGpu, 0x001fa828U, cmp90Wpr2Hi);
+
+            NV_PRINTF(LEVEL_ERROR,
+                      "CMP90_DEBUG: Compute unlock complete for devId=0x%04x\n",
+                      devId);
+        }
+
         plmStatus = kgspSec2PostblTimingRebuildStockSignature(pGpu, pKernelGsp);
         if (plmStatus != NV_OK)
         {
```

## 5. ДОПОЛНИТЕЛЬНЫЙ ПАТЧ ДЛЯ kernel_gsp_tu102.c

Также нужен патч для отключения WPR2 check (как в оригинальном 0001):

```diff
--- a/src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
+++ b/src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
@@ -4713,14 +4713,14 @@
     NV_ASSERT_OR_RETURN(pbRetry != NULL, NV_ERR_INVALID_ARGUMENT);
     *pbRetry = NV_FALSE;
 
-    // Fail early if WPR2 is up
-    if (kgspIsWpr2Up_HAL(pGpu, pKernelGsp) &&
-        (!pGpu->getProperty(pGpu, PDB_PROP_GPU_PREINITIALIZED_WPR_REGION)))
-    {
-        NV_PRINTF(LEVEL_ERROR, "unexpected WPR2 already up, cannot proceed with booting GSP\n");
-        NV_PRINTF(LEVEL_ERROR, "(the GPU is likely in a bad state and may need to be reset)\n");
-        return NV_ERR_INVALID_STATE;
-    }
+    // Bypass WPR2 check (for CMP 90HX compute unlock)
+    // if (kgspIsWpr2Up_HAL(pGpu, pKernelGsp) &&
+    //     (!pGpu->getProperty(pGpu, PDB_PROP_GPU_PREINITIALIZED_WPR_REGION)))
+    // {
+    //     NV_PRINTF(LEVEL_ERROR, "unexpected WPR2 already up, cannot proceed with booting GSP\n");
+    //     NV_PRINTF(LEVEL_ERROR, "(the GPU is likely in a bad state and may need to be reset)\n");
+    //     return NV_ERR_INVALID_STATE;
+    // }
```

## 6. УСТАНОВКА

### 6.1. Подготовка

```bash
# Установить nvidia-open 610.43.03
sudo apt install -y nvidia-driver-610-open nvidia-utils-610 libnvidia-compute-610

# Проверить
modinfo nvidia | grep version
# Ожидание: version: 610.43.03

# Проверить GPU
lspci -nn | grep 10de:2684
# Ожидание: NVIDIA Corporation GA102 [CMP 90HX]
```

### 6.2. Скачать исходники и применить патч

```bash
cd /home/g

# Скачать open-gpu-kernel-modules
wget https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/610.43.03.tar.gz
tar xzf 610.43.03.tar.gz
cd open-gpu-kernel-modules-610.43.03

# Создать файл патча
cat > /tmp/0001-cmp90-compute-unlock.patch << 'PATCH_EOF'
<СОДЕРЖИМОЕ ПАТЧА ИЗ СЕКЦИИ 4 ВЫШЕ>
PATCH_EOF

# Применить патч
patch -p1 < /tmp/0001-cmp90-compute-unlock.patch

# Проверить что патч применился
grep -c "CMP90_DEBUG" src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
# Ожидание: > 0
```

### 6.3. Компиляция

```bash
cd /home/g/open-gpu-kernel-modules-610.43.03

# Компиляция
make modules -j$(nproc) 2>&1 | tail -5

# Проверить что модуль собрался
ls -la kernel-open/nvidia.ko
strings kernel-open/nvidia.ko | grep "CMP90_DEBUG" | head -3
# Ожидение: строки с CMP90_DEBUG
```

### 6.4. Установка модуля

```bash
# Создать директорию для patched module
sudo mkdir -p /lib/modules/$(uname -r)/updates/cmp90-unlock/

# Установить модули
sudo cp kernel-open/nvidia.ko /lib/modules/$(uname -r)/updates/cmp90-unlock/
sudo cp kernel-open/nvidia-uvm.ko /lib/modules/$(uname -r)/updates/cmp90-unlock/ 2>/dev/null
sudo cp kernel-open/nvidia-modeset.ko /lib/modules/$(uname -r)/updates/cmp90-unlock/ 2>/dev/null
sudo cp kernel-open/nvidia-drm.ko /lib/modules/$(uname -r)/updates/cmp90-unlock/ 2>/dev/null

# Обновить module dependencies
sudo depmod -a

# Пересобрать initramfs
sudo update-initramfs -u -k $(uname -r)
```

### 6.5. Cold Reboot

```bash
sudo systemctl poweroff
# Отключить питание на 60 секунд
# Включить
```

## 7. ПРОВЕРКА

### 7.1. Проверить что patched модуль загрузился

```bash
cat /proc/driver/nvidia/version | head -1
# Ожидание: 610.43.03

modinfo nvidia | grep filename
# Ожидание: /lib/modules/.../updates/cmp90-unlock/nvidia.ko

strings /lib/modules/$(uname -r)/updates/cmp90-unlock/nvidia.ko | grep -c "CMP90_DEBUG"
# Ожидание: > 0
```

### 7.2. Проверить nvidia-smi

```bash
nvidia-smi
# Ожидание: GPU виден, CUDA работает

nvidia-smi --query-gpu=clocks.max.sm --format=csv,noheader
# Ожидание: высокое значение (без throttle)
```

### 7.3. Проверить регистры через BAR0

```bash
sudo python3 -c "
import mmap, os, struct

fd = os.open('/sys/bus/pci/devices/0000:01:00.0/resource0', os.O_RDWR)
mm = mmap.mmap(fd, 0x1000000, access=mmap.ACCESS_READ)

checks = [
    (0x00823804, 'FEAT_OVR_PLM',      0xFFFFF3FF),
    (0x00823B04, 'GFX_SPD_PLM',       0xFFFFF3FF),
    (0x0082381C, 'SM_SPD (SS0)',      0x88888888),
    (0x00823820, 'SM_SPD_1 (SS1)',    0x00000008),
    (0x00823830, 'GFX_SPD',           0x00000004),
]

print('=== CMP 90HX Register State ===')
for off, name, expected in checks:
    v = struct.unpack_from('<I', mm, off)[0]
    ok = '✅' if v == expected else '❌'
    print(f'  {ok} [{off:08X}] {name:25s} = 0x{v:08X} (expected 0x{expected:08X})')

mm.close(); os.close(fd)
"
```

Ожидаемый вывод:
```
=== CMP 90HX Register State ===
  ✅ [00823804] FEAT_OVR_PLM              = 0xFFFFF3FF
  ✅ [00823B04] GFX_SPD_PLM               = 0xFFFFF3FF
  ✅ [0082381C] SM_SPD (SS0)              = 0x88888888
  ✅ [00823820] SM_SPD_1 (SS1)            = 0x00000008
  ✅ [00823830] GFX_SPD                   = 0x00000004
```

### 7.4. Проверить dmesg

```bash
sudo dmesg | grep CMP90_DEBUG | tail -30
```

Ожидаемый вывод:
```
CMP90_DEBUG: Starting compute unlock for devId=0x2684
CMP90_DEBUG: PLM[0] FEAT_OVR_SM_SPD_PLM(0x823804) before=0xXXXXXXXX target=0xFFFFF3FF
CMP90_DEBUG: PLM[0] FEAT_OVR_SM_SPD_PLM attempt=0 status=0x0 before=0xXXXXXXXX after=0xFFFFF3FF target=0xFFFFF3FF
CMP90_DEBUG: PLM[1] FEAT_OVR_GFX_SPD_PLM(0x823B04) before=0xXXXXXXXX target=0xFFFFF3FF
CMP90_DEBUG: PLM[1] FEAT_OVR_GFX_SPD_PLM attempt=0 status=0x0 before=0xXXXXXXXX after=0xFFFFF3FF target=0xFFFFF3FF
CMP90_DEBUG: WRITE FEAT_OVR_SM_SPD (SS0)(0x82381c) = 0x88888888 (readback=0x88888888 OK)
CMP90_DEBUG: WRITE FEAT_OVR_SM_SPD_1 (SS1)(0x823820) = 0x00000008 (readback=0x00000008 OK)
CMP90_DEBUG: WRITE FEAT_OVR_GFX_SPD(0x823830) = 0x00000004 (readback=0x00000004 OK)
CMP90_DEBUG: Compute unlock complete for devId=0x2684
```

### 7.5. CUDA тест

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

### 7.6. Compute benchmark

```bash
python3 -c "
import torch
print(f'PyTorch: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')
if torch.cuda.is_available():
    print(f'Device: {torch.cuda.get_device_name(0)}')
    props = torch.cuda.get_device_properties(0)
    print(f'VRAM: {props.total_memory / 1024**3:.1f} GB')
    print(f'Compute: {props.major}.{props.minor}')
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
"
```

## 8. ОТКАТ

```bash
# Удалить patched module
sudo rm -rf /lib/modules/$(uname -r)/updates/cmp90-unlock/
sudo depmod -a

# Переустановить stock driver
sudo apt install --reinstall nvidia-driver-610-open

# Cold reboot
sudo systemctl poweroff
```

## 9. КЛЮЧЕВЫЕ ОТЛИЧИЯ ОТ CMP 170HX ПАТЧА

| Параметр | CMP 170HX (GA100) | CMP 90HX (GA102) |
|---|---|---|
| Device ID | 0x20C2 | 0x2684 |
| PLM FEAT value | 0xFFFFFFFF | 0xFFFFF3FF |
| PLM GFX | не используется | 0x823B04 → 0xFFFFF3FF |
| GFX_SPEED | не используется | 0x823830 → 0x4 |
| SS0 | 0x82381C → 0x88888888 | 0x82381C → 0x88888888 (тот же) |
| SS1 | 0x823820 → 0x8 | 0x823820 → 0x8 (тот же) |
| VRAM unlock | да (FBPA, LMR, fb_length, PMA) | нет (GDDR6, не нужен) |
| fb_length patch | да | нет |
| PMA extension | да | нет |
| BAR0/PRAMIN clamp | да | нет |
| CE scrub workarounds | да | нет |
| Persistent SW state | да | нет |
| Кол-во патчей | 6 | 1 (compute only) |

## 10. ВОЗМОЖНЫЕ ПРОБЛЕМЫ

### Если PLM не открывается (after != target)

```bash
# Проверить dmesg
sudo dmesg | grep "CMP90_DEBUG.*FAILED" | tail -10

# Возможные причины:
# 1. SEC2 Booter не может достучаться до регистра
# 2. GSP firmware не загружается (проверить dmesg | grep NVRM)
# 3. WPR2 check не отключён (проверить патч в секции 5)

# Решение: ещё один cold boot
sudo systemctl poweroff
# 60 сек → включить → проверить
```

### Если nvidia-smi "No devices"

```bash
# Проверить что модуль загрузился
lsmod | grep nvidia

# Проверить dmesg
sudo dmesg | grep -iE "NVRM|GSP|fail" | tail -20

# Если GSP-RM не стартует — возможно нужен patch 0002 (booter-verify)
# или patch 0006 (persistent-sw-state) от amoghmunikote
```

### Если compute работает но медленно

```bash
# Проверить регистры
sudo python3 -c "
import mmap, os, struct
fd = os.open('/sys/bus/pci/devices/0000:01:00.0/resource0', os.O_RDWR)
mm = mmap.mmap(fd, 0x1000000, access=mmap.ACCESS_READ)
ss0 = struct.unpack_from('<I', mm, 0x82381C)[0]
ss1 = struct.unpack_from('<I', mm, 0x823820)[0]
gfx = struct.unpack_from('<I', mm, 0x823830)[0]
print(f'SS0=0x{ss0:08X} SS1=0x{ss1:08X} GFX=0x{gfx:08X}')
mm.close(); os.close(fd)
"

# Если SS0 != 0x88888888 — PLM не открыт, проверить Booter
# Если GFX != 0x4 — GFX PLM не открыт
```

## 11. ЧЕКЛИСТ

```
Подготовка:
□ nvidia-open 610.43.03 установлен
□ CMP 90HX обнаружен (lspci | grep 2684)
□ kernel headers установлены
□ Secure Boot отключён

Установка:
□ Исходники скачаны (open-gpu-kernel-modules-610.43.03)
□ Патч применён (patch -p1)
□ Компиляция успешна (nvidia.ko создан)
□ Модуль установлен в updates/cmp90-unlock/
□ depmod -a выполнен
□ initramfs пересобран

Cold Reboot:
□ sudo systemctl poweroff
□ Питание отключено 60 секунд
□ Сервер включен

Проверка:
□ cat /proc/driver/nvidia/version → 610.43.03
□ nvidia-smi → GPU виден
□ Регистры: PLM=0xFFFFF3FF, SS0=0x88888888, SS1=0x8, GFX=0x4
□ dmesg | grep CMP90_DEBUG → PLM opened, writes OK
□ CUDA cuInit → 0
□ Matmul benchmark → адекватные TFLOPS
```

═══════════════════════════════════════════════════════════════════════════

ПРИМЕЧАНИЕ: Этот патч создан по аналогии с amoghmunikote/cmpunlocker
patch 0001 для CMP 170HX. Механизм тот же (SEC2 Booter PLM-open +
GPU_REG_WR32), но регистры адаптированы для GA102 (CMP 90HX).

Если патч не работает с первого раза:
1. Проверить dmesg на ошибки CMP90_DEBUG
2. Попробовать добавить patches 0002 (booter-verify) и 0006 (persistent-sw-state)
   от amoghmunikote, адаптировав их для 0x2684
3. Убедиться что WPR2 check отключён (секция 5)
4. Попробовать cold boot ещё раз

═══════════════════════════════════════════════════════════════════════════
