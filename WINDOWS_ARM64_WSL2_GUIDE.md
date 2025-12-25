# Innioasis Updater on Windows ARM64 via WSL2

This guide shows you how to use Innioasis Updater on Windows ARM64 devices by running it in WSL2 (Windows Subsystem for Linux).

## Why WSL2?

Windows ARM64 cannot run the x86/x64 binaries (SP Flash Tool, DLLs) required for firmware flashing. WSL2 provides a native ARM64 Linux environment where the Python-based MTKClient works perfectly.

## Prerequisites

- **Windows 11 ARM64** (Build 22000 or later)
- **Stock Innioasis Y1** that needs initial firmware installation
- **Administrator access** on Windows

## Step 1: Install WSL2

Open PowerShell as Administrator and run:

```powershell
wsl --install -d Ubuntu
```

After installation, restart your computer when prompted.

## Step 2: Set Up Ubuntu in WSL2

1. Launch Ubuntu from the Start menu
2. Create a username and password when prompted
3. Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 3: Install usbipd-win (Windows Side)

This enables USB device passthrough from Windows to WSL2.

### Option A: Using winget (recommended)

In PowerShell as Administrator:

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

### Option B: Manual installation

1. Download from: https://github.com/dorssel/usbipd-win/releases/latest
2. Install the `.msi` file
3. Restart PowerShell to load the `usbipd` command

## Step 4: Install Innioasis Updater in WSL2

In your Ubuntu WSL2 terminal:

```bash
# Install dependencies
sudo apt install -y python3 git libusb-1.0-0 python3-pip libfuse2

# Clone the repository
git clone https://github.com/ryan-specter/Innioasis-Updater.git
cd Innioasis-Updater

# Install Python dependencies
pip3 install -r requirements.txt
pip3 install .

# Set up USB rules
sudo usermod -a -G plugdev $USER
sudo usermod -a -G dialout $USER
sudo cp mtkclient/Setup/Linux/*.rules /etc/udev/rules.d
sudo udevadm control -R
sudo udevadm trigger
```

**Important:** After adding user to groups, you need to restart WSL2:

In PowerShell:
```powershell
wsl --shutdown
```

Then reopen Ubuntu.

## Step 5: Connect Your Y1 for Firmware Installation

### A. Prepare the Y1 for Bootrom Mode

1. **Power off** your Y1 completely
2. **Do NOT connect USB yet**

### B. Share the USB Device with WSL2

In PowerShell as **Administrator**:

```powershell
# List all USB devices
usbipd list
```

You should see something like:
```
BUSID  VID:PID    DEVICE                           STATE
1-4    0e8d:0003  MediaTek USB Port                Not shared
```

**Important:** The VID:PID `0e8d:0003` is the MediaTek bootrom mode identifier.

```powershell
# Bind the device (one-time setup per device)
usbipd bind --busid 1-4

# Verify it's shared
usbipd list
```

Now the STATE should show "Shared".

### C. Boot Y1 into Bootrom Mode and Attach

1. In PowerShell (keep it open):
```powershell
# Keep WSL2 active
wsl
```

2. **While holding VOL DOWN + POWER**, connect the USB cable to your Y1
3. Wait 2-3 seconds, then release the buttons

4. In another PowerShell window as Administrator:
```powershell
# Attach the device to WSL2
usbipd attach --wsl --busid 1-4
```

5. Verify in your WSL2 Ubuntu terminal:
```bash
lsusb
```

You should see:
```
Bus 001 Device 002: ID 0e8d:0003 MediaTek Inc. MT6227 phone
```

## Step 6: Run Innioasis Updater

In your WSL2 Ubuntu terminal:

```bash
cd ~/Innioasis-Updater
python3 firmware_downloader.py
```

The GUI should launch! You can now:
1. Select a firmware from the list
2. Click "Install / Restore"
3. Follow the on-screen instructions

The app will use MTKClient (Python-based) to flash your Y1.

## Troubleshooting

### GUI doesn't launch

Ensure WSLg is working:

```bash
# Test with a simple GUI app
sudo apt install x11-apps
xcalc
```

If xcalc doesn't appear, WSLg isn't working. Try:
```powershell
wsl --shutdown
wsl --update
```

### USB device not detected

In PowerShell:
```powershell
# Detach and reattach
usbipd detach --busid 1-4
usbipd attach --wsl --busid 1-4
```

In WSL2:
```bash
# Check if device is visible
lsusb
dmesg | tail -20
```

### Permission denied errors

Make sure you restarted WSL2 after adding user to groups:
```powershell
wsl --shutdown
```

Then reopen Ubuntu and try again.

### Y1 won't enter bootrom mode

- Try **VOL UP + POWER** instead of VOL DOWN + POWER
- Make sure device is completely powered off first
- Some Y1s require holding buttons for 5+ seconds after connecting USB

## After First Firmware Install

Once you've installed a custom ROM (like "Rockbox + Original Software"):
- Your Y1 will have root access and Fast Update capability
- You can use **Fast Update** directly from Windows (no WSL2 needed!)
- Fast Update works via ADB which is pure Python - fully ARM64 compatible

## USB Device Management

### To disconnect the Y1:

In PowerShell:
```powershell
usbipd detach --busid 1-4
```

Then physically unplug the USB cable.

### To reuse for another flash:

Just repeat Step 5C (no need to bind again).

## Performance Notes

- MTKClient in WSL2 ARM64 runs at **native ARM64 speed**
- No emulation overhead like running x86 tools would have
- Firmware flashing takes approximately 10-30 minutes depending on ROM size

## Alternative: Native Windows ARM64 Support

This WSL2 method is a workaround. Native Windows ARM64 support is being explored in [Issue #17](https://github.com/ryan-specter/Innioasis-Updater/issues/17).

## Support

If you encounter issues:
1. Check the [main README](README.md) for general troubleshooting
2. Review the [MTKClient documentation](https://github.com/bkerler/mtkclient)
3. File an issue at https://github.com/ryan-specter/Innioasis-Updater/issues

Include:
- Your Windows ARM64 device model
- WSL2 version (`wsl --version`)
- Ubuntu version (`lsb_release -a`)
- Any error messages from the terminal

---

**Credits:**
- MTKClient by bkerler
- USB/IP support via usbipd-win project
- Guide created for Issue #17
