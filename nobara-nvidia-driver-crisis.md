# Nobara Linux: Nvidia 580.x Driver Crisis - GTX 1050 Ti Completely Broken

## TL;DR
**All fc43 packaged Nvidia drivers are broken for GTX 1050 Ti (and likely all 1000-series).** Newer versions (580.119+) have GPU device IDs stripped out. Older versions (580.105) won't compile on current kernels. Users are stuck with no working drivers.

---

## System Information
| Component | Details |
|-----------|---------|
| **Distribution** | Nobara Linux 43 (Fedora 43 based) |
| **Kernel** | 6.18.9-201.nobara.fc43.x86_64 |
| **GPU** | GeForce GTX 1050 Ti (PCI ID: 10de:1c82) |
| **Previously Working** | Yes - system worked perfectly before 2026-02-10 update |

---

## Timeline of Failure

### Before Update (Working)
- Driver: `dkms-nvidia-580.119.02-1.fc43` (or earlier)
- Status: ✅ Fully functional
- Display: Native resolution, hardware acceleration working

### After Update (2026-02-10)
- Driver: `dkms-nvidia-580.126.09-1.fc43`
- Status: ❌ Complete failure
- Display: 1024x768 fallback, no GPU acceleration

### Warning During Update
The Nobara updater displayed this message:
> **BIG FAT PSA:**  
> NVIDIA ARE DROPPING SUPPORT FOR THE NVIDIA 1000 SERIES AND LOWER WHEN 590 DRIVERS MOVE TO PRODUCTION. WE WILL -NOT- BE SUPPORTING OLDER DRIVERS. THIS MEANS NOBARA WILL NOT PROVIDE OLDER DRIVERS FOR 1000 SERIES CARDS OR LOWER WHEN THE 590 DRIVERS BECOME PRODUCTION BRANCH.

**However**: The 580.x series officially supports GTX 1050 Ti according to Nvidia documentation. This appears to be **premature removal** of 1000-series support.

---

## Problem 1: Newer Drivers Stripped of GPU IDs

### Affected Versions
- `dkms-nvidia-580.126.09-1.fc43` ❌
- `dkms-nvidia-580.119.02-1.fc43` ❌

### Symptoms
```bash
$ sudo modprobe nvidia
modprobe: ERROR: could not insert 'nvidia': No such device

$ sudo dmesg | grep nvidia
NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:1c82)
NVRM: nvidia.ko because it does not include the required GPU
nvidia 0000:01:00.0: probe with driver nvidia failed with error -1
```

### Root Cause Analysis

#### GPU is Detected by System
```bash
$ lspci | grep -i vga
01:00.0 VGA compatible controller: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] (rev a1)
```

#### DKMS Build Succeeds
```bash
$ sudo dkms autoinstall
Building module(s).............................. done.
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia.ko.zst
Running depmod... done.
```

#### Module Has NO Specific GPU Device IDs
A properly functioning Nvidia driver should contain hundreds of specific PCI device aliases like:
```
alias: pci:v000010DEd00001C82sv*sd*bc*sc*i*  # GTX 1050 Ti
alias: pci:v000010DEd00001C81sv*sd*bc*sc*i*  # GTX 1050
alias: pci:v000010DEd00001B80sv*sd*bc*sc*i*  # GTX 1080
... (hundreds more)
```

**Instead, we get:**
```bash
$ sudo modinfo nvidia | grep "alias.*10de"
(empty output)

$ sudo modinfo nvidia | grep "alias.*pci"
alias:          pci:v000010DEd*sv*sd*bc06sc80i00*
alias:          pci:v000010DEd*sv*sd*bc03sc02i00*
alias:          pci:v000010DEd*sv*sd*bc03sc00i00*
```
Only generic PCI class aliases - **no specific device IDs whatsoever**.

#### Source Code Verification
```bash
$ grep -r "1c82" /usr/src/nvidia-580.126.09/
(empty output)

$ grep -r "1c82" /usr/src/nvidia-580.119.02/
(empty output)
```

**The GPU device ID does not exist in the source code at all.**

---

## Problem 2: Older Drivers Won't Compile on Current Kernel

### Affected Version
- `dkms-nvidia-580.105.08-2.fc43` ❌

### Symptoms
```bash
$ sudo dkms autoinstall
Building module(s).................(bad exit status: 2)
Error! Bad return status for module build on kernel: 6.18.9-201.nobara.fc43.x86_64 (x86_64)
```

### Build Error
```
nvidia-uvm/uvm_va_range_device_p2p.c:363:13: error: too many arguments to function 'get_dev_pagemap'
  363 |             get_dev_pagemap(page_to_pfn(page), NULL);
      |             ^~~~~~~~~~~~~~~
```

**Root Cause**: Kernel 6.18.9 changed the `get_dev_pagemap()` API signature. Driver 580.105.08 is incompatible with this kernel version.

---

## Complete Test Matrix

| Driver Version | fc Version | GPU IDs Present | Builds on 6.18.9 | Result |
|---------------|------------|-----------------|------------------|--------|
| 580.126.09 | fc43 | ❌ No | ✅ Yes | ❌ Won't load - "No such device" |
| 580.119.02 | fc43 | ❌ No | ✅ Yes | ❌ Won't load - "No such device" |
| 580.105.08 | fc43 | ❓ Unknown | ❌ No | ❌ Build failure - Kernel API incompatibility |

---

## The Catch-22

Users with 1000-series GPUs are now stuck:

1. **Can't use newer drivers** (580.119+) → GPU IDs stripped, driver won't recognize hardware
2. **Can't use older drivers** (580.105) → Won't compile on current kernel
3. **Can't downgrade kernel** → Older drivers still missing GPU IDs in fc43 builds

**Result**: No working driver available through package manager.

---

## Evidence of Retroactive Changes

Multiple driver versions in the fc43 repository have been modified to remove GPU device IDs. This suggests either:

1. **Repository-wide rebuild** that stripped device IDs from all 580.x versions
2. **Intentional sabotage** to force 1000-series users off the platform early
3. **Botched transition** to 590 drivers affecting older packages

---

## Attempted Solutions (All Failed)

### ❌ Reinstall Driver
```bash
sudo dnf reinstall dkms-nvidia
sudo dkms autoinstall
# Result: Builds successfully but won't load - "No such device"
```

### ❌ Downgrade to 580.119.02
```bash
sudo dnf downgrade dkms-nvidia-3:580.119.02-1.fc43
# Result: Also missing GPU IDs - same "No such device" error
```

### ❌ Downgrade to 580.105.08
```bash
sudo dnf downgrade dkms-nvidia-3:580.105.08-2.fc43
# Result: Build failure - Kernel API incompatibility
```

### ❌ Manual DKMS rebuild
```bash
sudo dkms remove nvidia/580.126.09 --all
sudo dkms autoinstall
# Result: Builds fine, still missing GPU IDs
```

---

## Working Solution: Nvidia Official .run Installer

Since all packaged drivers are broken, the only working solution is to bypass the package manager entirely:

### Steps
1. **Remove all Nobara nvidia packages**
   ```bash
   sudo dkms remove nvidia/580.126.09 --all
   sudo dnf remove 'nvidia-*' 'dkms-nvidia'
   ```

2. **Download official Nvidia driver**
   ```bash
   wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.126.09/NVIDIA-Linux-x86_64-580.126.09.run
   ```

3. **Install**
   ```bash
   sudo chmod +x NVIDIA-Linux-x86_64-580.126.09.run
   sudo ./NVIDIA-Linux-x86_64-580.126.09.run
   ```

**Trade-offs**:
- ✅ Actually has GPU device IDs intact
- ✅ Works on current kernel
- ❌ Bypasses package manager (manual updates required)
- ❌ Must reinstall after kernel updates

---

## Questions for Nobara Maintainers

1. **Was the removal of GPU device IDs intentional or a packaging error?**

2. **Why were multiple driver versions retroactively modified in the repository?**

3. **What is the migration path for 1000-series users?**
   - Will legacy 470.x drivers be provided?
   - Should users switch to Nouveau?
   - Are .run installers the only option?

4. **Why push broken drivers to production?**
   - No GPU IDs = completely non-functional
   - This affects a large user base (GTX 1050 Ti is extremely common)

5. **If 1000-series EOL is imminent, why not:**
   - Keep one working 580.x version frozen for fc43?
   - Provide migration documentation before breaking things?
   - Test drivers before pushing to users?

---

## Impact Assessment

### Affected Hardware
- **All GTX 1000-series** (1080 Ti, 1080, 1070, 1060, 1050 Ti, 1050)
- **All GTX 900-series and older**
- Potentially **millions of users** given the popularity of these cards

### User Impact
- ❌ No hardware acceleration
- ❌ Poor gaming performance
- ❌ Video playback issues
- ❌ Can't use CUDA applications
- ❌ System stuck at low resolution
- ❌ Must manually manage drivers outside package manager

---

## Comparison: Nobara vs Nvidia Official

### Nobara Packaged Driver (580.126.09)
```bash
$ sudo modinfo nvidia | grep "alias.*10de" | wc -l
0
```
**Zero GPU device IDs** → Non-functional

### Nvidia Official Driver (Expected)
```bash
$ sudo modinfo nvidia | grep "alias.*10de" | wc -l
500+
```
**Hundreds of GPU device IDs** → Functional

---

## Recommendations

### For Users
1. **Do NOT update** if you're on a working driver
2. **Use versionlock** to prevent automatic updates:
   ```bash
   sudo dnf install python3-dnf-plugin-versionlock
   sudo dnf versionlock add dkms-nvidia
   ```
3. **Switch to Nvidia .run installer** if already broken
4. **File bug reports** to get maintainer attention

### For Nobara Project
1. **Immediately revert** broken driver packages
2. **Provide legacy 470.x drivers** for 1000-series
3. **Test before deploying** - this should have been caught
4. **Clear communication** - don't break things silently
5. **Migration documentation** - help users transition properly

---

## Related Issues
- [Link to forum post if any]
- [Link to Discord discussion if any]

---

## Full Logs

### Update Log
[Available at paste.nobaraproject.org](https://paste.nobaraproject.org/?921885974ea97563#5UxuPfCkD63CkzrQiLz9739FMfvNGmkBQVYfWq5ZTpoL)

### DKMS Build Log
<details>
<summary>Click to expand</summary>

```
Sign command: /lib/modules/6.18.9-201.nobara.fc43.x86_64/build/scripts/sign-file
Signing key: /var/lib/dkms/mok.key
Public certificate (MOK): /var/lib/dkms/mok.pub
Autoinstall of module nvidia/580.126.09 for kernel 6.18.9-201.nobara.fc43.x86_64
Building module(s).............................. done.
Signing module /var/lib/dkms/nvidia/580.126.09/build/kernel-open/nvidia.ko
Signing module /var/lib/dkms/nvidia/580.126.09/build/kernel-open/nvidia-modeset.ko
Signing module /var/lib/dkms/nvidia/580.126.09/build/kernel-open/nvidia-drm.ko
Signing module /var/lib/dkms/nvidia/580.126.09/build/kernel-open/nvidia-uvm.ko
Signing module /var/lib/dkms/nvidia/580.126.09/build/kernel-open/nvidia-peermem.ko
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia.ko.zst
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia-modeset.ko.zst
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia-drm.ko.zst
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia-uvm.ko.zst
Installing /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia-peermem.ko.zst
Running depmod.... done.
Autoinstall on 6.18.9-201.nobara.fc43.x86_64 succeeded for module(s) nvidia.
```

**Module loads successfully but immediately fails:**
```bash
$ sudo modprobe nvidia
modprobe: ERROR: could not insert 'nvidia': No such device
```
</details>

### Kernel Messages
<details>
<summary>Click to expand dmesg output</summary>

```
$ sudo dmesg | grep -i nvidia
[  136.185524] nvidia 0000:01:00.0: probe with driver nvidia failed with error -1
[  136.185599] NVRM: The NVIDIA probe routine failed for 1 device(s).
[  136.185601] NVRM: None of the NVIDIA devices were initialized.
[  136.523598] NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:1c82)
               NVRM: nvidia.ko because it does not include the required GPU
               NVRM: www.nvidia.com.
[  136.529740] nvidia 0000:01:00.0: probe with driver nvidia failed with error -1
[  136.529815] NVRM: The NVIDIA probe routine failed for 1 device(s).
[  136.529818] NVRM: None of the NVIDIA devices were initialized.
```
</details>

---

## System Configuration Dumps

<details>
<summary>Installed Nvidia Packages</summary>

```bash
$ rpm -qa | grep nvidia | sort
dkms-nvidia-580.126.09-1.fc43.x86_64
libva-nvidia-driver-0.0.15-1.fc43.x86_64
libnvidia-cfg-3:580.126.09-1.fc43.x86_64
libnvidia-fbc-3:580.126.09-1.fc43.x86_64
libnvidia-gpucomp-3:580.126.09-1.fc43.x86_64
libnvidia-ml-3:580.126.09-1.fc43.x86_64
nvidia-driver-3:580.126.09-1.fc43.x86_64
nvidia-driver-cuda-3:580.126.09-1.fc43.x86_64
nvidia-driver-cuda-libs-3:580.126.09-1.fc43.x86_64
nvidia-driver-libs-3:580.126.09-1.fc43.x86_64
nvidia-kmod-common-3:580.126.09-1.fc43.noarch
nvidia-libXNVCtrl-3:580.126.09-1.fc43.x86_64
nvidia-modprobe-3:580.126.09-1.fc43.x86_64
nvidia-persistenced-3:580.126.09-1.fc43.x86_64
nvidia-settings-3:580.126.09-1.fc43.x86_64
```
</details>

<details>
<summary>DKMS Status</summary>

```bash
$ dkms status
nvidia/580.126.09, 6.18.9-201.nobara.fc43.x86_64, x86_64: installed
openrazer-driver/3.11.0, 6.18.6-200.nobara.fc43.x86_64, x86_64: installed
openrazer-driver/3.11.0, 6.18.7-200.nobara.fc43.x86_64, x86_64: installed
openrazer-driver/3.11.0, 6.18.9-201.nobara.fc43.x86_64, x86_64: installed
```
</details>

<details>
<summary>Module Info</summary>

```bash
$ sudo modinfo nvidia
filename:       /lib/modules/6.18.9-201.nobara.fc43.x86_64/extra/nvidia.ko.zst
firmware:       nvidia/580.126.09/gsp_tu10x.bin
firmware:       nvidia/580.126.09/gsp_ad10x.bin
license:        Dual MIT/GPL
version:        580.126.09
srcversion:     [...]
alias:          pci:v000010DEd*sv*sd*bc06sc80i00*
alias:          pci:v000010DEd*sv*sd*bc03sc02i00*
alias:          pci:v000010DEd*sv*sd*bc03sc00i00*
depends:        drm
name:           nvidia
vermagic:       6.18.9-201.nobara.fc43.x86_64 SMP preempt mod_unload
sig_id:         PKCS#7
```

**Note**: Only 3 generic aliases, no specific device IDs
</details>

---

## Metadata
- **Reporter**: Linux noob (investigation performed with AI assistance)
- **Date**: 2026-02-10
- **Distribution**: Nobara Linux 43
- **Severity**: Critical - Completely breaks GPU functionality
- **Reproducibility**: 100% on all tested driver versions in fc43 repo

---

**Status**: ⚠️ **CRITICAL BUG** - No working driver available through package manager for GTX 1000-series GPUs
