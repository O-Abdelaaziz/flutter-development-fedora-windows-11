# 🐦 Flutter Dev Setup — Fedora 43 + Windows 11 Emulator

> A complete guide to installing Flutter on **Fedora 43**, setting up the Android SDK manually (**no Android Studio required — on either OS**), and connecting to an Android emulator running on a **Windows 11** host via VMware.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1 — Install System Dependencies](#step-1--install-system-dependencies)
- [Step 2 — Install Flutter SDK](#step-2--install-flutter-sdk)
- [Step 3 — Install Android Command Line Tools](#step-3--install-android-command-line-tools)
- [Step 4 — Configure Environment Variables](#step-4--configure-environment-variables)
- [Step 5 — Finalize Android Setup](#step-5--finalize-android-setup)
- [Step 6 — Verify Installation](#step-6--verify-installation)
- [Windows 11: Run the Emulator & Connect via ADB](#windows-11-run-the-emulator--connect-via-adb)
  - [Windows SDK Setup (No Android Studio)](#windows-sdk-setup-no-android-studio)
    - [1. Download & Extract Command Line Tools](#1-download--extract-command-line-tools)
    - [2. Configure Environment Variables](#2-configure-environment-variables)
    - [3. Install SDK Components](#3-install-sdk-components)
    - [4. Create and Start the Emulator](#4-create-and-start-the-emulator)
  - [VMware Network Setup](#vmware-network-setup)
  - [Windows Firewall Rule](#windows-firewall-rule)
  - [Connect ADB from Fedora](#connect-adb-from-fedora)
- [Troubleshooting](#troubleshooting)
  - [ADB Device Issues](#adb-device-issues)
  - [Port Proxy (NAT / VMnet8)](#port-proxy-nat--vmnet8)
  - [DNS & Network Errors](#dns--network-errors)

---

## Prerequisites

| Requirement | Details |
|---|---|
| OS (Host) | Windows 11 — Android SDK + Emulator via CLI only (no Android Studio) |
| OS (Guest) | Fedora 43 (Workstation) running in VMware Workstation |
| Shell | `bash` (commands target `~/.bashrc`) |
| Internet | Required for SDK downloads |

---

## Step 1 — Install System Dependencies

Install all required build tools and libraries for Flutter's Linux toolchain:

```bash
sudo dnf install clang cmake ninja-build pkgconf-pkg-config \
  gtk3-devel xz-devel mesa-libGLU-devel libX11-devel -y
```

> **Note:** `xz-devel` replaces the older `liblzma-devel` package on Fedora 43.

---

## Step 2 — Install Flutter SDK

**1.** Download the latest stable Flutter SDK from the official archive:
👉 [https://docs.flutter.dev/install/manual](https://docs.flutter.dev/install/manual)

**2.** Create a development directory and extract the SDK:

```bash
mkdir -p ~/Development
tar -xf ~/Downloads/flutter_linux_*.tar.xz -C ~/Development
```

This will create `~/Development/flutter/`.

---

## Step 3 — Install Android Command Line Tools

Since we're **not** using Android Studio, download the **"Command line tools only"** package:
👉 [https://developer.android.com/studio](https://developer.android.com/studio) → scroll down to *Command line tools only*

**1.** Create the Android SDK directory structure:

```bash
mkdir -p ~/Development/android/cmdline-tools
```

**2.** Extract the downloaded zip:

```bash
unzip ~/Downloads/commandlinetools-linux-*.zip \
  -d ~/Development/android/cmdline-tools
```

**3.** ⚠️ Critical step — rename the extracted folder to `latest`:

```bash
mv ~/Development/android/cmdline-tools/cmdline-tools \
   ~/Development/android/cmdline-tools/latest
```

**4.** Directory structure after setup:
```bash
~/development/
├── flutter/
├── android/
│   └── cmdline-tools/
│       └── latest/
│           ├── bin/
│           ├── lib/
│           └── ...
```

> The SDK manager **requires** this exact directory name to locate the SDK root correctly. Skipping this will cause tool lookup failures.

---

## Step 4 — Configure Environment Variables

Add the following to the **end** of your `~/.bashrc` file:

```bash
# ── Flutter ────────────────────────────────────────────────
export PATH="$PATH:$HOME/Development/flutter/bin"

# ── Android SDK ────────────────────────────────────────────
export ANDROID_HOME="$HOME/Development/android"
export PATH="$PATH:$ANDROID_HOME/cmdline-tools/latest/bin"
export PATH="$PATH:$ANDROID_HOME/platform-tools"
export PATH="$PATH:$ANDROID_HOME/emulator"

# ── Browser for Flutter Web ────────────────────────────────
export browser_flutter="firefox"
export CHROME_EXECUTABLE=$(which $browser_flutter)

# ── ADB Host (for remote emulator) ─────────────────────────
export ADBHOST=localhost

# ── Optional: Force X11 backend ────────────────────────────
# export QT_QPA_PLATFORM=xcb
```

**Reload your shell:**

```bash
## bashrc
source ~/.bashrc
## zshrc
source ~/.zshrc
```

> **Pro tip:** Use `$ANDROID_HOME` in your PATH exports to keep things DRY:
> ```bash
> export PATH="$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools"
> ```

---

## Step 5 — Finalize Android Setup

### 5a. Install SDK Components

Use `sdkmanager` to download the required platform tools and build images:

```bash
sdkmanager --sdk_root=$HOME/Development/android \
  "platform-tools" \
  "platforms;android-36" \
  "build-tools;36.0.0"
```

Or use the dynamic variable if your environment is already sourced:

```bash
sdkmanager --sdk_root=$ANDROID_HOME \
  "platform-tools" \
  "platforms;android-36" \
  "build-tools;36.0.0"
```

> **Fedora-specific gotcha:** If the Android emulator fails to start later, you may need a 32-bit compatibility library:
> ```bash
> sudo dnf install ncurses-compat-libs
> ```

### 5b. Accept Android Licenses

```bash
flutter doctor --android-licenses
```

Accept all prompts with `y`.

### 5c. Link Flutter to Your SDK (optional, to make sure everything is correct!!)
**- You can jump to step 6 **

Point Flutter at your manually installed SDK:

```bash
flutter config --android-sdk ~/Development/android
```

---

## Step 6 — Verify Installation

```bash
flutter doctor
```

A healthy output looks like:

```
[✓] Flutter (Channel stable, ...)
[✓] Android toolchain - develop for Android devices (Android SDK version 36.0.0)
[✓] Chrome - develop for the web
[✓] Linux toolchain - develop for Linux desktop
[✓] Connected device (N available)
[✓] Network resources
```

---

## Windows 11: Run the Emulator & Connect via ADB

This section covers **two parts**:
1. Installing the Android SDK and creating an emulator on Windows **without Android Studio** — using only the Command Line Tools.
2. Connecting that emulator to Flutter running on Fedora inside VMware.

---

### Windows SDK Setup (No Android Studio)

Just like on Fedora, we'll install only the Command Line Tools and use `sdkmanager` directly. No IDE required.

#### 1. Download & Extract Command Line Tools

**1.** Visit the [Android Studio Downloads](https://developer.android.com/studio/install) page, scroll down to the **Command line tools only** section, and download the Windows `.zip` package.

**2.** Create your SDK base directory:

```
C:\Android\
```

**3.** ⚠️ Critical — the tools require a specific nested folder structure. Create it manually:

```
C:\Android\cmdline-tools\latest\
```

**4.** Extract the downloaded `.zip`. Inside it you'll find a `cmdline-tools` folder containing `bin\`, `lib\`, etc. Move the **contents** of that inner folder directly into `C:\Android\cmdline-tools\latest\`.

Your final structure should look like this:

```
C:\Android\
└── cmdline-tools\
    └── latest\
        ├── bin\
        │   ├── sdkmanager.bat
        │   └── avdmanager.bat
        └── lib\
```

> Skipping this step causes `sdkmanager` to fail with SDK root lookup errors — the same gotcha as on Linux.

---

#### 2. Configure Environment Variables

**1.** Open **"Edit the system environment variables"** from the Windows Search bar.

**2.** Click **Environment Variables**.

**3.** Under **User variables**, click **New** and add:

| Variable Name | Variable Value |
|---|---|
| `ANDROID_HOME` | `C:\Android` |

**4.** Select the **Path** variable under User variables and click **Edit**. Add these three new entries:

```
%ANDROID_HOME%\cmdline-tools\latest\bin
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\emulator
```

**5.** Click OK on all dialogs, then open a **new** Command Prompt or PowerShell to pick up the changes.

---

#### 3. Install SDK Components

In your new terminal, run:

```powershell
# Accept all SDK licenses first
sdkmanager --licenses

# Install required packages
sdkmanager "platform-tools" "platforms;android-35" "emulator" "system-images;android-35;google_apis;x86_64"
```

> **Note:** Replace `android-35` with your preferred API level if needed. API 35 matches Android 15.

---

#### 4. Create and Start the Emulator

**1.** Create a new AVD (Android Virtual Device):

```powershell
avdmanager create avd -n MyEmulator -k "system-images;android-35;google_apis;x86_64" --device "pixel_8"
```

When prompted *"Do you wish to create a custom hardware profile?"*, type `no` and press Enter.

**2.** Start the emulator:

```powershell
emulator -avd MyEmulator
```

> **Performance tip:** For the best emulator performance, ensure:
> - **Virtualization (VT-x/AMD-V)** is enabled in your BIOS.
> - **Windows Hypervisor Platform** is enabled — search for *"Turn Windows features on or off"* and check it.

---

### VMware Network Setup

With the emulator running on Windows, configure VMware so Fedora can reach it over the network.

**1.** Shut down the Fedora VM.

**2.** Open **VMware Virtual Machine Settings** → **Network Adapter**.

**3.** Set to **Bridged: Connected directly to the physical network**.

**4.** *(Recommended)* Check **"Replicate physical network connection state"**.

**5.** Start the Fedora VM.

> Bridged mode gives your Fedora VM an IP in the same subnet as Windows, enabling direct ADB communication.

---

### Windows Firewall Rule

Open **Windows Defender Firewall with Advanced Security** and create an inbound rule:

| Setting | Value |
|---|---|
| Rule Type | Port |
| Protocol | TCP |
| Local Port | `5555/5037` |
| Action | Allow the connection |
| Profiles | Domain, Private, Public |
| Name | `ADB Emulator` |

---

### Connect ADB from Fedora

**1.** Find your Windows 11 IP address — run in Windows CMD:

```cmd
ipconfig
```

Note the **IPv4 Address** (e.g., `192.168.1.50`).

**2.** On Windows, verify the emulator port and enable TCP mode:

```cmd
adb devices
adb tcpip 5555
```

**3.** From your Fedora terminal, connect:

```bash
adb connect 192.168.1.50:5555
```

**4.** Confirm the connection:

```bash
adb devices
flutter devices
```

---

## Troubleshooting

### ADB Device Issues

**Symptom:** `adb devices` shows `emulator-5554  offline`

**Resolution steps (in order):**

```powershell
# 1. Stop the emulator first (close the emulator window or use: adb emu kill)

# 2. Kill and restart the ADB server
adb kill-server
adb start-server

# 3. Check what's using port 5555 (run in Windows CMD/PowerShell)
netstat -ano | findstr :5555

# 4. Kill the conflicting process
taskkill /pid <PID> /f

# 5. Start ADB in verbose mode to monitor connections
adb -a nodaemon server start

# 6. Validate only one device is connected
adb devices
```

**Make sure the emulator is listening on TCP (run in Windows):**

```powershell
adb tcpip 5555
```

---

### Port Proxy (NAT / VMnet8)

If you're using **NAT networking** (VMnet8) instead of Bridged, `adb connect` may fail because the emulator isn't reachable from the VM directly. Use Windows port proxying to bridge the gap.

**1.** Open **PowerShell as Administrator** on Windows and create a port proxy:

```powershell
netsh interface portproxy add v4tov4 `
  listenaddress=192.168.204.1 `
  listenport=5555 `
  connectaddress=127.0.0.1 `
  connectport=5555
```

> Replace `192.168.204.1` with your actual **VMnet8 IP address** (check via `ipconfig` on Windows).

**2.** Allow the port through the firewall:

```powershell
New-NetFirewallRule -DisplayName "Allow ADB Port 5555" `
  -Direction Inbound -Action Allow `
  -Protocol TCP -LocalPort 5555
```

**3.** Connect from Fedora:

```bash
adb connect 192.168.204.1:5555
```

#### Managing Port Proxy Rules

```powershell
# View all rules
netsh interface portproxy show all

# Remove a specific rule
netsh interface portproxy delete v4tov4 `
  listenaddress=192.168.204.1 `
  listenport=5555

# Nuclear option — wipe ALL portproxy rules
netsh interface portproxy reset
```

> **Note:** If you added a custom firewall rule, delete it separately:
> ```powershell
> netsh advfirewall firewall delete rule name="Allow ADB Port 5555"
> ```

---

### DNS & Network Errors

**Symptom:** `flutter doctor` reports network failures:

```
[!] Network resources
    ✗ Failed host lookup: 'pub.dev'
    ✗ Failed host lookup: 'storage.googleapis.com'
```

This is usually VMware NAT not passing DNS settings to Fedora.

**Fix 1 — Set DNS manually in Fedora:**

```bash
sudo nano /etc/resolv.conf
```

Add at the top:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`) and test:

```bash
ping google.com
```

**Fix 2 — Restart VMware NAT service on Windows:**

If `ping 192.168.204.2` (your gateway) fails from inside Fedora, the NAT service may be stuck.

1. Open `services.msc` on Windows.
2. Find **VMware NAT Service**.
3. Right-click → **Restart**.

**Fix 3 — Install `mesa-demos` (optional, cleans up `eglinfo` warning):**

```bash
sudo dnf install mesa-demos
```

This resolves the `! Unable to access driver information using 'eglinfo'` warning in `flutter doctor`.

---

## Final Checklist

- [ ] `flutter doctor` shows all green checkmarks (or only the `eglinfo` warning)
- [ ] `adb devices` shows exactly **one** device
- [ ] `flutter devices` lists your Windows emulator
- [ ] DNS resolution works from inside Fedora (`ping pub.dev`)
- [ ] `flutter run` launches an app on the emulator successfully

---

<div align="center">

Made with ☕ for the Fedora + Flutter community

</div>
