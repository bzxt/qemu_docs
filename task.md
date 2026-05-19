# Task: Implement a QEMU PCIe Device with PASID/ATS Capabilities

## 1. Task Overview

You are working on the QEMU source tree. Your task is to implement a new experimental PCIe endpoint device for x86/q35 machines.

The device must be named:

```text
xpert-pasid-ats-pci
```

It must be usable from the QEMU command line as:

```bash
-device xpert-pasid-ats-pci
```

The device is used to evaluate:

1. QEMU PCIe device registration;
2. PCI configuration space correctness;
3. MMIO BAR implementation;
4. Linux-side PCI enumeration;
5. PASID Extended Capability exposure;
6. ATS Extended Capability exposure;
7. a simplified device-side Address Translation Cache behavior.

This task focuses on QEMU PCIe device modeling. You do **not** need to implement full Linux SVA binding, real per-process IOMMU address translation, or real PCIe ATS Translation Request TLP emulation. However, the device must expose valid PASID and ATS capabilities through PCIe configuration space and provide a simplified BAR-based test interface.

The final solution must implement a real QEMU PCIe device that is observable and testable from Linux. Do not fake serial logs, hard-code expected judge strings, or bypass the actual device behavior.

---

## 2. Allowed Modification Scope

You may only modify files under the following QEMU directories:

```text
qemu/hw/misc
qemu/hw/i386
qemu/hw/pci
qemu/include/system
qemu/include/hw/pci
qemu/include/hw/i386
qemu/system
```

Do not modify files outside these directories.

The submitted solution will only synchronize files from these allowed directories to the judge environment. If your implementation depends on files outside these directories, the judge may not see those changes.

Do not modify:

```text
qemu/build
qemu/images
qemu/judge
qemu/logs
Linux kernel image
rootfs image
hidden judge scripts
generated build artifacts
test logs
```

---

## 3. Submission Rules

The task is evaluated repeatedly during the run. You should submit after completing meaningful implementation milestones.

Recommended submission points:

1. after the QEMU device type is registered and QEMU recognizes `-device xpert-pasid-ats-pci`;
2. after Linux can enumerate the device with `lspci`;
3. after BAR0 and basic registers work;
4. after PASID capability is visible;
5. after ATS capability is visible;
6. after simplified ATC behavior works.

Do not submit while files are half-written or while QEMU obviously cannot compile. A broken intermediate submission may receive a low score.

However, do not wait too long between submissions. Submit at least once every 15 minutes during active work so the judge can track progress.

---

## 4. Local Build and Test Paths

The local development environment provides the following paths:

```text
QEMU source tree:
  /home/workbench/qemu

QEMU executable after build:
  /home/workbench/qemu/build/qemu-system-x86_64

Linux kernel image:
  /home/workbench/linux-7.0/arch/x86/boot/bzImage

Root filesystem image:
  /home/workbench/rootfs/ubuntu-rootfs.ext4
```

A typical build command is:

```bash
cd /home/workbench/qemu
ninja -C build -j$(nproc)
```

If the build directory does not exist, configure QEMU first:

```bash
cd /home/workbench/qemu
./configure \
  --target-list=x86_64-softmmu \
  --enable-debug \
  --disable-werror \
  --disable-docs \
  --enable-slirp

ninja -C build -j$(nproc)
```

The judge may run QEMU with TCG instead of KVM. Do not require KVM.

---

## 5. Device Identity

The QEMU device must use the following identity:

```text
QEMU device name: xpert-pasid-ats-pci
Vendor ID:        0x1234
Device ID:        0x11e8
Class Code:       0x0880
Header Type:      normal PCI endpoint device
```

Expected Linux behavior:

```bash
lspci -nn
```

should show a device containing:

```text
1234:11e8
```

The class should be a generic system peripheral or equivalent non-bridge endpoint class.

---

## 6. BAR0 Requirements

The device must implement one MMIO BAR:

```text
BAR index: 0
BAR type:  MMIO
BAR size:  4 KB
```

Linux should show the BAR through:

```bash
lspci -vvv
```

The judge may also inspect:

```text
/sys/bus/pci/devices/<BDF>/resource
/sys/bus/pci/devices/<BDF>/resource0
```

BAR0 must be mmap-able from userspace through `resource0`.

---

## 7. BAR0 Register Map

All registers are little-endian 32-bit registers.

| Offset | Name | Access | Required behavior |
|---:|---|---|---|
| `0x00` | `MAGIC` | RO | Must return `0x58505441` |
| `0x04` | `VERSION` | RO | Must return `0x00010000` |
| `0x08` | `CONTROL` | RW | Control register |
| `0x0C` | `STATUS` | RO | Device status register |
| `0x10` | `SCRATCH` | RW | Generic read/write test register |
| `0x14` | `PASID_VALUE` | RW | Current PASID test value |
| `0x18` | `ATS_IOVA_LOW` | RW | Low 32 bits of test IOVA |
| `0x1C` | `ATS_IOVA_HIGH` | RW | High 32 bits of test IOVA |
| `0x20` | `ATS_TRANSLATED_LOW` | RO | Low 32 bits of translated address |
| `0x24` | `ATS_TRANSLATED_HIGH` | RO | High 32 bits of translated address |
| `0x28` | `ATC_STATUS` | RO | Simplified ATC status |

Required constants:

```text
MAGIC   = 0x58505441
VERSION = 0x00010000
BAR0 size = 0x1000
```

Initial state after device creation or reset should be reasonable:

```text
CONTROL = 0
STATUS = 0
SCRATCH = 0
PASID_VALUE = 0
ATC_STATUS hit bit = 0
```

---

## 8. CONTROL Register Definition

The `CONTROL` register is located at BAR0 offset `0x08`.

| Bit | Name | Meaning |
|---:|---|---|
| 0 | `DEVICE_ENABLE` | Enable device-side test logic |
| 1 | `PASID_ENABLE` | Enable PASID test state |
| 2 | `ATS_ENABLE` | Enable ATS test state |
| 3 | `ATC_INSERT` | Insert current `(PASID, IOVA)` into simplified ATC |
| 4 | `ATC_LOOKUP` | Lookup current `(PASID, IOVA)` in simplified ATC |
| 5 | `ATC_INVALIDATE` | Invalidate simplified ATC |

Bits 3, 4, and 5 are command bits. When these bits are written, the device should process the corresponding command. The device may clear command bits after processing.

---

## 9. STATUS Register Definition

The `STATUS` register is located at BAR0 offset `0x0C`.

| Bit | Name |
|---:|---|
| 0 | `DEVICE_ENABLED` |
| 1 | `PASID_ENABLED` |
| 2 | `ATS_ENABLED` |
| 3 | `ATC_VALID` |
| 4 | `ATC_HIT` |

The judge expects the enable state and ATC hit/valid state to be observable through `STATUS` and/or `ATC_STATUS`.

---

## 10. ATC_STATUS Register Definition

The `ATC_STATUS` register is located at BAR0 offset `0x28`.

Required bits:

| Bit | Name |
|---:|---|
| 0 | `ATC_HIT` |
| 1 | `ATC_VALID` |

On ATC lookup hit, bit 0 should be set.

After ATC invalidation, both hit and valid state should be cleared.

---

## 11. Simplified ATC Behavior

This task does not require full PCIe ATS or real IOMMU translation.

Instead, implement a simplified local Address Translation Cache inside the QEMU device.

The device should maintain at least one cached entry:

```text
cached_pasid
cached_iova
cached_translated_addr
valid
```

The current PASID comes from:

```text
PASID_VALUE
```

The current IOVA is:

```c
IOVA = ((uint64_t)ATS_IOVA_HIGH << 32) | ATS_IOVA_LOW;
```

When the guest writes `CONTROL` with `ATC_INSERT` set:

```c
cached_pasid = PASID_VALUE;
cached_iova = IOVA;
cached_translated_addr = IOVA ^ ((uint64_t)PASID_VALUE << 12);
valid = true;
```

When the guest writes `CONTROL` with `ATC_LOOKUP` set:

- if `valid == true`, `cached_pasid == PASID_VALUE`, and `cached_iova == IOVA`, the lookup hits;
- on hit:
  - `ATC_STATUS bit0` should be set;
  - `ATS_TRANSLATED_LOW/HIGH` should return `cached_translated_addr`;
- on miss:
  - `ATC_STATUS bit0` should be cleared.

When the guest writes `CONTROL` with `ATC_INVALIDATE` set:

```c
valid = false;
ATC_STATUS hit bit = 0;
```

The deterministic translation rule must be:

```c
translated_addr = iova ^ ((uint64_t)pasid << 12);
```

The hidden judge program relies on this exact rule.

---

## 12. PASID Capability Requirement

The device must expose a valid PCIe PASID Extended Capability.

Linux should be able to detect it through:

```bash
lspci -vvv
```

Expected behavior:

```text
PASID capability exists
PASID control is readable/writable
Max PASID Width is non-zero and reasonable
Recommended Max PASID Width: 20
PASID Enable state is maintainable
```

Full Linux SVA binding is not required.

You do not need to implement complete per-process PASID address-space binding in `hw/i386/intel-iommu`.

---

## 13. ATS Capability Requirement

The device must expose a valid PCIe ATS Extended Capability.

Linux should be able to detect it through:

```bash
lspci -vvv
```

Expected behavior:

```text
ATS capability exists
ATS control is readable/writable
ATS Enable state is maintainable
```

Full PCIe ATS Translation Request TLP emulation is not required for the base score.

You do not need to implement a complete real PCIe ATC invalidation protocol. The simplified BAR-based ATC behavior described above is sufficient for the base task.

---

## 14. Expected Linux-Side Behavior

When the QEMU system is launched with:

```bash
-device xpert-pasid-ats-pci
```

the Linux guest should:

1. boot successfully;
2. show device `1234:11e8` in `lspci -nn`;
3. show BAR0 with size 4 KB in `lspci -vvv`;
4. show PASID capability in `lspci -vvv`;
5. show ATS capability in `lspci -vvv`;
6. allow a userspace program to mmap BAR0 through sysfs `resource0`;
7. pass MAGIC, VERSION, SCRATCH, PASID, IOVA, and simplified ATC behavior tests.

The judge will validate the solution by rebuilding QEMU, booting Linux, parsing `lspci` and `dmesg`, and running a userspace BAR test program.

---

## 15. Suggested Implementation Order

Implement the task in the following order. Do not try to implement everything at once.

### Step 1: Find QEMU PCI device examples

Inspect existing QEMU PCI/PCIe devices, especially under:

```text
hw/misc
hw/pci
include/hw/pci
```

Useful concepts include:

```text
PCIDevice
TYPE_PCI_DEVICE
QOM type registration
MemoryRegion
MMIO read/write callbacks
pci_register_bar
PCI vendor/device/class fields
PCIe capability helpers
PCIe extended capability layout
```

### Step 2: Register the new QEMU device

Create a new PCIe endpoint device named:

```text
xpert-pasid-ats-pci
```

At this point, QEMU should recognize:

```bash
-device xpert-pasid-ats-pci
```

This is the first useful submission milestone.

### Step 3: Make Linux enumerate the device

Set the required PCI identity:

```text
Vendor ID = 0x1234
Device ID = 0x11e8
Class Code = 0x0880
Endpoint header type
```

Boot Linux and check:

```bash
lspci -nn
```

Linux should show `1234:11e8`.

This is the second useful submission milestone.

### Step 4: Implement BAR0

Add a 4 KB MMIO BAR0.

Implement at least:

```text
MAGIC
VERSION
SCRATCH
```

Check that:

```text
MAGIC returns 0x58505441
VERSION returns 0x00010000
SCRATCH can write/read at least two different values
```

This is the third useful submission milestone.

### Step 5: Implement CONTROL, STATUS, PASID_VALUE, and IOVA registers

Implement the remaining BAR registers:

```text
CONTROL
STATUS
PASID_VALUE
ATS_IOVA_LOW
ATS_IOVA_HIGH
ATS_TRANSLATED_LOW
ATS_TRANSLATED_HIGH
ATC_STATUS
```

At this stage, enable bits and register state should be visible through MMIO.

### Step 6: Add PASID Extended Capability

Expose PASID Extended Capability so that Linux can show it through:

```bash
lspci -vvv
```

Do not implement full Linux SVA. The required part is the PCIe capability exposure and basic state maintenance.

This is the fourth useful submission milestone.

### Step 7: Add ATS Extended Capability

Expose ATS Extended Capability so that Linux can show it through:

```bash
lspci -vvv
```

Do not implement full ATS Translation Request TLP emulation. The required part is capability exposure and basic state maintenance.

This is the fifth useful submission milestone.

### Step 8: Implement simplified ATC behavior

Implement:

```text
ATC_INSERT
ATC_LOOKUP
ATC_INVALIDATE
PASID-based cache matching
deterministic translated address generation
```

Use the required formula:

```c
translated_addr = iova ^ ((uint64_t)pasid << 12);
```

This is the final main functionality milestone.

---

## 16. Local Validation

After each coherent implementation milestone, build QEMU:

```bash
cd /home/workbench/qemu
ninja -C build -j$(nproc)
```

If needed, configure first:

```bash
cd /home/workbench/qemu
./configure \
  --target-list=x86_64-softmmu \
  --enable-debug \
  --disable-werror \
  --disable-docs \
  --enable-slirp
```

A typical local QEMU launch command is:

```bash
/home/workbench/qemu/build/qemu-system-x86_64 \
  -machine q35,accel=tcg \
  -m 2048 \
  -smp 2 \
  -cpu qemu64 \
  -kernel /home/workbench/linux-7.0/arch/x86/boot/bzImage \
  -drive file=/home/workbench/rootfs/ubuntu-rootfs.ext4,format=raw \
  -append "root=/dev/sda rw console=ttyS0,115200 nokaslr panic=-1 quiet loglevel=3 systemd.show_status=false udev.log_level=3 intel_iommu=on iommu=pt" \
  -nographic \
  -serial file:/home/workbench/xpert-serial.log \
  -monitor none \
  -device xpert-pasid-ats-pci \
  -device intel-iommu,intremap=on,caching-mode=on,x-scalable-mode=on,x-flts=on,x-pasid-mode=on,aw-bits=48
```

Then inspect:

```bash
cat /home/workbench/xpert-serial.log
```

Important markers include:

```text
SEBENCH_QEMU_TEST_BEGIN
SEBENCH_QEMU_TEST_END
XPERT_DEVICE_FOUND
XPERT_MAGIC_PASS
XPERT_VERSION_PASS
XPERT_SCRATCH_DEADBEEF_PASS
XPERT_SCRATCH_A5A55A5A_PASS
XPERT_PASID_RW_PASS
XPERT_IOVA_RW_PASS
XPERT_ATC_INSERT_PASS
XPERT_ATC_LOOKUP_PASS
XPERT_ATC_PASID_ISOLATION_PASS
XPERT_ATC_INVALIDATE_PASS
```

---

## 17. What the Judge Checks

The hidden judge may check:

1. QEMU builds successfully;
2. QEMU recognizes `-device xpert-pasid-ats-pci`;
3. Linux boots to the automatic test script;
4. `lspci -nn` shows `1234:11e8`;
5. BAR0 exists and has size 4 KB;
6. `MAGIC` and `VERSION` are correct;
7. `SCRATCH` supports read/write;
8. PASID capability is visible in `lspci -vvv`;
9. ATS capability is visible in `lspci -vvv`;
10. `PASID_VALUE` and IOVA registers support read/write;
11. simplified ATC insert/lookup/invalidate behavior is correct;
12. different PASIDs do not falsely hit the same ATC entry;
13. the deterministic translated address matches the specified formula;
14. the solution does not modify hidden resources or generated artifacts.

---

## 18. Important Restrictions

Do not:

```text
fake serial output
hard-code judge strings
modify Linux rootfs or kernel images
modify hidden judge scripts
submit build artifacts
require KVM
require a custom Linux kernel driver
implement behavior only in logs without a real PCIe device
```

The device must be real and testable from Linux through PCI enumeration and BAR0 MMIO access.

---

## 19. Final Reminder

Focus on incremental progress.

A good implementation path is:

```text
QEMU device registration
→ Linux lspci enumeration
→ BAR0 MMIO
→ MAGIC/VERSION/SCRATCH
→ CONTROL/STATUS/PASID/IOVA
→ PASID Capability
→ ATS Capability
→ simplified ATC
```

Submit after each coherent milestone, but avoid submitting half-written code. Submit at least once every 15 minutes during active work.
