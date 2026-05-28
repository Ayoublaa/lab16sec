# 🔐 Android SSL Pinning Bypass with Objection

> **Disable SSL pinning at runtime.** Intercept HTTPS traffic from hardened Android apps using Frida + Objection.

![Android](https://img.shields.io/badge/Platform-Android-3DDC84?style=flat-square&logo=android&logoColor=white)
![Frida](https://img.shields.io/badge/Runtime-Frida-black?style=flat-square)
![Objection](https://img.shields.io/badge/Tool-Objection-brightgreen?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

---

## ⚠️ Ethical Notice

> **LEGAL DISCLAIMER:** This lab is for **authorized security testing only**. Only test apps you own or have explicit written permission to audit. Bypassing security on third-party apps without authorization violates laws in most jurisdictions (CFAA, GDPR, Computer Misuse Act, etc.). Use this knowledge responsibly.

---

## 🎯 Lab Objectives

By the end, you will know how to:

| # | Goal |
|-|---------|
| **1** | Install Objection + Frida on Windows/macOS/Linux |
| **2** | Deploy `frida-server` on an Android device or emulator |
| **3** | Configure Burp Suite / mitmproxy as an HTTPS interceptor |
| **4** | Attach Objection at app launch and bypass SSL pinning with one command |
| **5** | Capture and decrypt HTTPS traffic in plaintext |
| **6** | Troubleshoot common attach/spawn errors |

---

## 📋 Prerequisites

### Required Tools

| Tool | Version | Role |
|------|---------|------|
| **Frida** | 16.x+ | Runtime instrumentation framework |
| **Objection** | 1.11.x+ | CLI wrapper around Frida |
| **ADB** | 31+ | Android Debug Bridge |
| **Burp Suite CE** OR **mitmproxy** | Latest | HTTPS proxy / traffic capture |
| **Android Emulator** OR **Rooted Device** | API 25+ | Target to intercept |
| **Python** | 3.9+ | Objection dependency |

### Installation

#### macOS / Linux
```bash
# Install Frida
pip install frida-tools

# Install Objection
pip install objection

# Verify
objection --version
frida --version
```

#### Windows (PowerShell as Admin)
```powershell
pip install frida-tools objection
```

### Device Setup

- **Emulator:** Android Studio's default emulator (API 29+)
- **Physical Device:** Rooted Android 10+ (or use Frida's bridging mode)
- **USB Debugging:** Enabled in Developer Options

---

## 🔧 Step-by-Step

### Step 1 — Download and Run frida-server

Download the correct architecture for your device:

```bash
# List your device
adb devices

# Detect architecture
adb shell getprop ro.product.cpu.abi
# → arm64-v8a or armeabi-v7a

# Download frida-server (match your architecture)
# From: https://github.com/frida/frida/releases
wget https://github.com/frida/frida/releases/download/16.0.10/frida-server-16.0.10-android-arm64.xz

# Extract, push, and run
xz -d frida-server-16.0.10-android-arm64.xz
adb push frida-server-16.0.10-android-arm64 /data/local/tmp/
adb shell chmod +x /data/local/tmp/frida-server-16.0.10-android-arm64
adb shell /data/local/tmp/frida-server-16.0.10-android-arm64 &
```

Verify frida-server is running:
<img width="577" height="84" alt="image" src="https://github.com/user-attachments/assets/60d97715-e165-4b21-9efe-5e4867700774" />


```bash
frida-ps -U
# → list of running processes
```

---

### Step 2 — Configure Your Proxy

#### Option A: Burp Suite

1. **Start Burp** → Proxy tab → Options
2. **Proxy Listeners:**
   - Bind to: `127.0.0.1:8080` (or `0.0.0.0:8080` for network access)
   - Enable "Invisible Proxy" (optional, for harder targets)
3. **Export CA certificate:**
   - Burp menu → Settings → Network → Certificates
   - Export certificate as `.der`

#### Option B: mitmproxy

```bash
mitmproxy --mode transparent -p 8080
# or with certificate export:
mitmproxy --save-stream ~/mitm.flows -p 8080
```
<img width="313" height="348" alt="image" src="https://github.com/user-attachments/assets/50a7d40a-f341-441f-951b-9d45a0fd9eb5" />

---

### Step 3 — Install CA Certificate on Device

#### For Emulator (easy)

```bash
# Convert DER to PEM
openssl x509 -inform DER -in burp.der -out burp.pem

# Extract certificate hash
openssl x509 -inform PEM -subject_hash_old -in burp.pem | head -1
# → e.g., c8750f0d

# Rename and push
cp burp.pem c8750f0d.0
adb push c8750f0d.0 /system/etc/security/cacerts/
adb shell chmod 644 /system/etc/security/cacerts/c8750f0d.0



# Restart to apply
adb reboot
```
<img width="181" height="319" alt="installcertif" src="https://github.com/user-attachments/assets/345a976a-0950-492a-a223-82cc74257ced" />

#### For Rooted Device

Same steps, but push to `/system/etc/security/cacerts/` (requires writable `/system`).

---

### Step 4 — Attach Objection & Disable Pinning

#### Launch target app first (keep it in background)

```bash
adb shell am start -n com.example.app/.MainActivity
```

#### Spawn with Objection (recommended for new sessions)

```bash
objection -N -s "android sslpinning disable" explore --startup-command "android sslpinning disable"
```

#### Or attach to running app

```bash
# List running apps
objection -N explore -p com.example.app

# Once attached, in Objection REPL:
(com.example.app): > android sslpinning disable
```

#### What this does

The command:
1. Hooks `SSLContext.init()` / `TrustManager` methods
2. Replaces certificate validation with a no-op
3. Allows any certificate (including self-signed ones)

---

### Step 5 — Capture Traffic

#### Set proxy on emulator

```bash
adb shell settings put global http_proxy 127.0.0.1:8080
adb shell settings put global https_proxy 127.0.0.1:8080
```

Or use system proxy settings in Android Settings → Network → Proxy.
<img width="955" height="502" alt="burp" src="https://github.com/user-attachments/assets/36f61180-1d45-401f-821a-ba82260909f6" />



#### Monitor in Burp/mitmproxy

```bash
# In Burp: Proxy → HTTP History → see all requests in plaintext
# In mitmproxy: commands view shows live request/response pairs
```

#### Verify decryption

- App makes an HTTPS request
- Request appears **unencrypted** in proxy (headers, body readable)
- Certificate shown in proxy is your CA, not the server's

---

## 🐛 Troubleshooting

### "Error: Unable to find PID for com.example.app"
- **Cause:** App not running or process name wrong
- **Fix:** `adb shell pm list packages | grep example` to find full name; launch app first before attaching

### "frida: not found"
- **Cause:** Frida not installed or not in PATH
- **Fix:** `pip install frida-tools --upgrade` and restart terminal

### "Error: Failed to establish connection"
- **Cause:** frida-server not running on device
- **Fix:** 
  ```bash
  adb shell ps | grep frida
  # If not listed, restart it:
  adb shell /data/local/tmp/frida-server-16.0.10-android-arm64 &
  ```

### Objection attaches but SSL pinning still active
- **Cause:** App loads certificate validation *after* attach; pinning code not yet instrumented
- **Fix:** Use `--startup-command` to attach **at launch**, or add delay:
  ```bash
  # Wait 5 seconds after attach, then disable
  (app): > %resume
  (app): > android sslpinning disable
  ```
<img width="959" height="141" alt="image" src="https://github.com/user-attachments/assets/3f6161af-b885-42b5-b8f6-2e4076a554dc" />

### "Cannot mount /system as writable"
- **Cause:** Device is not rooted or frida-server is not running as root
- **Fix:** For emulator: `adb root` then remount; for physical device: ensure Magisk/SuperSU

### Traffic still encrypted in proxy
- **Cause:** CA not installed correctly; pinning hook didn't activate
- **Fix:**
  1. Verify CA is in `/system/etc/security/cacerts/`
  2. Reboot device after installing CA
  3. In Burp, check certificate is listed under "CA Certificates"
  4. Restart Objection with `-s` and re-run `android sslpinning disable`

---

## ✅ Good Practices

1. **Always use spawning (`-s`)** — safer than attaching to running processes; avoids partial instrumentation
2. **Disable pinning BEFORE first request** — if pinning check runs before hook, it may cache result
3. **Monitor logcat** — `adb logcat | grep -i ssl` to see pinning logs
4. **Test with HTTPS sites** — verify decryption is working (e.g., app calling `https://api.example.com`)
5. **Document findings** — note which libraries implement pinning (OkHttp, Retrofit, Conscrypt, etc.)

---
<img width="959" height="497" alt="httpburp" src="https://github.com/user-attachments/assets/dde8d9c0-40bc-4bc6-9449-77457790f1dc" />

## ⚡ One-Liner Quick Start

```bash
# Terminal 1: Start frida-server
adb shell /data/local/tmp/frida-server-16.0.10-android-arm64 &

# Terminal 2: Burp listening on 8080 + CA exported

# Terminal 3: Full chain
adb shell settings put global http_proxy 127.0.0.1:8080 && \
adb shell am start -n com.example.app/.MainActivity && \
objection -N -s "android sslpinning disable" explore
```

Then launch requests in the app and monitor in Burp.
<img width="959" height="214" alt="image" src="https://github.com/user-attachments/assets/ed756ca0-1939-4b6c-9680-aa6c7cd0398a" />

---

## 📖 Reference

- [Frida Docs](https://frida.re/docs/)
- [Objection GitHub](https://github.com/sensepost/objection)
- [OWASP Testing Guide — Android](https://owasp.org/www-project-web-security-testing-guide/)
- [Android Security & Privacy Year in Review](https://security.googleblog.com/)
- [Burp Suite User Guide](https://portswigger.net/burp/documentation)

---

## 🚀 Next Steps

- Try bypassing pinning on **multiple apps** (OkHttp, Retrofit, Conscrypt)
- Modify requests in Burp and observe app behavior
- Combine with other hooks: `android hooking list activities`, `android intent launch`
- Build custom Frida scripts for deeper inspection

---

<div align="center">

**Made with ☕ and Frida hooks**

Happy intercepting — **with permission!**

</div>
