# Nobara Linux Nvidia Driver Fix for GTX 1000-Series

> **TL;DR**: Nobara's packaged Nvidia drivers are broken for GTX 1000-series cards. This repo has the fix + automation to keep it working.

[![Status](https://img.shields.io/badge/status-verified%20working-success)](https://github.com/yourusername/nobara-nvidia-fix)
[![GPU](https://img.shields.io/badge/tested-GTX%201050%20Ti-blue)](https://github.com/yourusername/nobara-nvidia-fix)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## 🔴 The Problem

**After updating Nobara Linux on 2026-02-10, GTX 1000-series GPUs stopped working completely.**

### What Happened?

Nobara's packaged Nvidia drivers (`dkms-nvidia`) had **all GPU device IDs stripped out** in preparation for dropping 1000-series support. The drivers build successfully but refuse to load:

```bash
$ sudo modprobe nvidia
modprobe: ERROR: could not insert 'nvidia': No such device

$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver
```

### Why Is This Bad?

- ❌ Display stuck at 1024x768 resolution
- ❌ No hardware acceleration (CPU rendering only)
- ❌ Games unplayable
- ❌ Video editing/CUDA apps broken
- ❌ **Multiple driver versions affected** (580.126.09, 580.119.02, 580.105.08)

### Affected Hardware

- **All GTX 1000-series**: 1080 Ti, 1080, 1070, 1060, 1050 Ti, 1050
- **All GTX 900-series and older**
- Potentially **millions of users**

---

## ✅ The Solution

**Use Nvidia's official `.run` installer instead of Nobara's broken packages.**

The official driver has all GPU IDs intact and actually works.

### Quick Fix (5 minutes)

```bash
# 1. Remove broken Nobara drivers
sudo dnf remove 'nvidia-*' 'dkms-nvidia' --exclude=nvidia-gpu-firmware -y

# 2. Download official Nvidia driver
cd ~/Downloads
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.126.09/NVIDIA-Linux-x86_64-580.126.09.run
chmod +x NVIDIA-Linux-x86_64-580.126.09.run

# 3. Install
sudo ./NVIDIA-Linux-x86_64-580.126.09.run
# Answer "Yes" to all prompts

# 4. Enable Wayland support
sudo nano /etc/default/grub
# Add nvidia-drm.modeset=1 to GRUB_CMDLINE_LINUX
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# 5. Reboot
sudo reboot
```

**Result**: 1920x1080 resolution, hardware acceleration, everything works. ✨

---

## 📚 What's in This Repo

| File | Description |
|------|-------------|
| [`nobara-nvidia-driver-crisis.md`](nobara-nvidia-driver-crisis.md) | **Full bug report** - Complete analysis of what went wrong, technical evidence, impact assessment |
| [`nvidia-complete-fix-guide.md`](nvidia-complete-fix-guide.md) | **Step-by-step fix guide** - Detailed instructions, automation scripts, troubleshooting |
| [`nvidia-reinstall`](nvidia-reinstall) | **Automation script** - One-command reinstall after kernel updates |

---

## 🚀 Features

### ✅ Complete Fix
- Removes all broken Nobara packages
- Installs working Nvidia official driver
- Configures Wayland support
- Restores full GPU functionality

### 🤖 Automated Maintenance
- One-command reinstall after kernel updates: `sudo nvidia-reinstall`
- Optional automatic reinstalls via DNF hooks
- Cached installer (no re-download needed)

### 📖 Comprehensive Documentation
- Full incident timeline
- Root cause analysis with evidence
- Troubleshooting for common issues
- Prevention tips

---

## 🎯 Who Is This For?

✅ **You need this if:**
- You have a GTX 1000-series (or older) GPU
- You're running Nobara Linux 43
- Your display is stuck at 1024x768 after an update
- `nvidia-smi` doesn't work
- Games run like garbage

❌ **You don't need this if:**
- You have a RTX 2000-series or newer (590 drivers will support you)
- You're on a different distro (problem is Nobara-specific)
- You're using Nouveau (open source driver)

---

## ⚡ Quick Start

### Option 1: Manual Install (Recommended First Time)

Follow the [Complete Fix Guide](nvidia-complete-fix-guide.md) for detailed instructions.

### Option 2: Automated Script

```bash
# Download and install the automation script
sudo curl -o /usr/local/bin/nvidia-reinstall https://raw.githubusercontent.com/yourusername/nobara-nvidia-fix/main/nvidia-reinstall
sudo chmod +x /usr/local/bin/nvidia-reinstall

# Run it
sudo nvidia-reinstall
```

---

## 🔧 Maintenance

### After Every Kernel Update

The `.run` installer bypasses package management, so you must reinstall after kernel updates.

**Easy way:**
```bash
sudo nvidia-reinstall
```

**Manual way:**
```bash
sudo ~/Downloads/NVIDIA-Linux-x86_64-580.126.09.run
```

### Checking Status

```bash
# Verify driver is working
nvidia-smi

# Check GPU is rendering
glxinfo | grep "OpenGL renderer"
# Should show: NVIDIA GeForce GTX 1050 Ti (not llvmpipe)

# Check resolution
xrandr | grep connected
# Should show: 1920x1080 (or your native resolution)
```

---

## 📊 Before & After

### Before Fix ❌
```
Resolution: 1024x768 (fallback)
Renderer:   llvmpipe (software)
GPU:        Not detected
nvidia-smi: Failed
Performance: Terrible
```

### After Fix ✅
```
Resolution: 1920x1080 (native)
Renderer:   NVIDIA GeForce GTX 1050 Ti
GPU:        Fully functional
nvidia-smi: Working
Performance: Full speed
```

---

## 🐛 Troubleshooting

### "Black screen after reboot"
- Boot into recovery mode (select older kernel)
- Switch to X11 session at login screen
- See [Troubleshooting Guide](nvidia-complete-fix-guide.md#troubleshooting)

### "Still shows 1024x768"
- Check if Wayland config was applied: `cat /proc/cmdline | grep nvidia-drm.modeset`
- Verify driver is loaded: `nvidia-smi`
- Reinstall if needed: `sudo nvidia-reinstall`

### "nvidia-smi: command not found"
- Driver not installed correctly
- Rerun installer: `sudo ~/Downloads/NVIDIA-Linux-x86_64-580.126.09.run`

**Full troubleshooting guide:** [nvidia-complete-fix-guide.md#troubleshooting](nvidia-complete-fix-guide.md#troubleshooting)

---

## 📝 Technical Details

### Root Cause

Nobara's packaged drivers are missing GPU device IDs in the kernel module:

```bash
# Broken Nobara driver
$ sudo modinfo nvidia | grep "alias.*10de"
(empty - zero GPU device IDs)

# Working Nvidia driver
$ sudo modinfo nvidia | grep "alias.*10de" | wc -l
500+ (hundreds of GPU device IDs)
```

The source code itself is missing GPU IDs:
```bash
$ grep -r "1c82" /usr/src/nvidia-580.126.09/
(empty - GTX 1050 Ti device ID not in source)
```

**Full technical analysis:** [nobara-nvidia-driver-crisis.md](nobara-nvidia-driver-crisis.md)

---

## 🤝 Contributing

Found a bug? Have a better solution? PRs welcome!

### How to Help

1. **Test on your hardware** - Especially other GTX models
2. **Report issues** - Open a GitHub issue if you hit problems
3. **Share improvements** - Got a better automation script? Submit a PR
4. **Spread the word** - Help other Nobara users find this fix

---

## ⚠️ Important Notes

### Trade-offs

**Pros:**
- ✅ Actually works (unlike Nobara's drivers)
- ✅ Full hardware acceleration
- ✅ Native resolution
- ✅ DKMS auto-rebuilds on kernel updates

**Cons:**
- ❌ Bypasses package manager (manual maintenance)
- ❌ Must reinstall after kernel updates
- ❌ Not "officially" supported by Nobara

### Why Not Just Use Nobara's Drivers?

Because they're broken. We tried:
- ✗ 580.126.09 - No GPU IDs
- ✗ 580.119.02 - No GPU IDs
- ✗ 580.105.08 - Won't compile on new kernel

**All fc43 packaged drivers are gutted.** The .run installer is the only working solution.

---

## 📞 Support

- **Issues**: [GitHub Issues](https://github.com/yourusername/nobara-nvidia-fix/issues)
- **Discussions**: [GitHub Discussions](https://github.com/yourusername/nobara-nvidia-fix/discussions)
- **Nobara Discord**: Share this fix with others!

---

## 📜 License

MIT License - Use freely, share widely, save GPUs.

---

## 🙏 Acknowledgments

- **Investigation**: AI-assisted troubleshooting marathon
- **Testing**: GTX 1050 Ti on Nobara Linux 43
- **Documentation**: This repo
- **Community**: Everyone affected by this issue

---

## 📅 Timeline

| Date | Event |
|------|-------|
| 2026-02-10 | Nobara update breaks all GTX 1000-series drivers |
| 2026-02-11 | Investigation, fix, and documentation completed |
| 2026-02-11 | This repo created to help others |

---

## ⭐ Star This Repo

If this saved your GPU, give it a star! ⭐ 

It helps other Nobara users find the fix.

---

**Current Status**: ✅ **Verified Working** on GTX 1050 Ti, Nobara Linux 43, Kernel 6.18.9

**Last Updated**: 2026-02-11
