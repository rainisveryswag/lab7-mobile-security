# Android Dynamic Analysis Lab — MobSF + DIVA

**Author:** Yousra Zarri  
**Platform:** macOS (Apple Silicon) + Docker  
**Framework:** MobSF (Mobile Security Framework)  
**Target App:** DIVA — Damn Insecure and Vulnerable Android App

---

## Overview

This lab demonstrates how to perform both static and dynamic security analysis on an Android application using MobSF. The target is DIVA, an intentionally vulnerable app containing 13 real-world security challenges covering insecure storage, hardcoded secrets, input validation flaws, and access control issues.

The goal is to identify vulnerabilities at runtime — intercepting network traffic, capturing logs, and injecting Frida scripts — simulating what an attacker or security auditor would do against a real Android application.

---

## Environment Setup

### Requirements

| Tool | Version | Purpose |
|---|---|---|
| Android Studio | Panda 3 / 2025.3.3 | AVD creation and SDK management |
| Docker Desktop | 28.3.2 | Running MobSF container |
| ADB | 1.0.41 | Device communication |
| MobSF | latest | Static + dynamic analysis |
| DIVA APK | beta | Vulnerable target application |

### Emulator Configuration

An Android Virtual Device (AVD) was created without Google Play Store to ensure a clean analysis environment — no background noise from Google Services, no interference with the proxy, and full root access for MobSF instrumentation.

```
Device:       Pixel 5
API Level:    29 (Android 10)
Architecture: ARM64-v8a (Apple Silicon)
Play Store:   Disabled
```

> API 29 is the recommended version for MobSF dynamic analysis — newer APIs restrict system-level write access required by Frida.

### PATH Configuration

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$ANDROID_HOME/tools
```

---

## Running the Lab

### 1. Start the emulator via MobSF script

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
chmod +x scripts/start_avd.sh
./scripts/start_avd.sh MobSF_API29
```

Verify the emulator is detected:

```bash
adb devices
# emulator-5554   device
```

### 2. Launch MobSF via Docker

```bash
docker pull opensecurity/mobile-security-framework-mobsf:latest

docker run -it --rm \
  --net=host \
  -p 8000:8000 \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest
```

Access the interface at `http://127.0.0.1:8000` — credentials: `mobsf / mobsf`

> `--net=host` is required on macOS/Linux for MobSF to communicate with ADB over the local network.

### 3. Upload and analyze DIVA

```bash
curl -L -o ~/Downloads/diva-beta.apk \
  https://github.com/payatu/diva-android/raw/master/diva-beta.apk
```

In MobSF → **Upload & Analyze** → select `diva-beta.apk`

---

## Dynamic Analysis

Once the static scan completes, click **Start Dynamic Analyzer**. MobSF automatically:

- Installs DIVA on the emulator via ADB
- Starts the Frida server
- Configures a global HTTPS proxy with its own CA certificate
- Begins capturing logs, network traffic, and file system activity

![Dynamic Analyzer — DIVA running](screenshots/dynamic_analyzer.png)

### Active Frida Scripts

MobSF injects the following hooks by default:

| Script | Purpose |
|---|---|
| API Monitoring | Intercepts sensitive Android API calls |
| SSL Pinning Bypass | Forces acceptance of MobSF's CA certificate |
| Root Detection Bypass | Prevents the app from detecting the rooted environment |
| Debugger Check Bypass | Bypasses anti-debug protections |
| Clipboard Monitor | Captures clipboard reads/writes |

![Frida instrumentation and shell access](screenshots/frida_shell.png)

---

## DIVA — 13 Challenges

DIVA exposes 13 vulnerability categories visible directly in the emulator screen:

![DIVA challenge list on emulator](screenshots/diva_challenges.png)

| # | Challenge | Vulnerability Type |
|---|---|---|
| 1 | Insecure Logging | Sensitive data written to Logcat |
| 2 | Hardcoding Issues Part 1 | Credentials in source code |
| 3 | Insecure Data Storage Part 1 | SharedPreferences in plaintext |
| 4 | Insecure Data Storage Part 2 | SQLite database unencrypted |
| 5 | Insecure Data Storage Part 3 | Temporary files with sensitive data |
| 6 | Insecure Data Storage Part 4 | External storage exposure |
| 7 | Input Validation Issues Part 1 | SQL Injection |
| 8 | Input Validation Issues Part 2 | Directory traversal |
| 9 | Access Control Issues Part 1 | Exported activity without protection |
| 10 | Access Control Issues Part 2 | Hardcoded credentials bypass |
| 11 | Access Control Issues Part 3 | Intent-based access bypass |
| 12 | Hardcoding Issues Part 2 | API keys in native code |
| 13 | Input Validation Issues Part 3 | Remote code execution via input |

### Key findings from runtime analysis

**Insecure Logging** — credentials appear in plaintext in Logcat, visible via MobSF's live log stream.

**Insecure Data Storage** — sensitive files found under `/data/data/jakhar.aseem.diva/` with no encryption:
```bash
adb shell cat /data/data/jakhar.aseem.diva/shared_prefs/*.xml
```

**Exported Activities** — activities accessible without authentication, launchable directly:
```bash
adb shell am start -n jakhar.aseem.diva/.APICredsActivity
```

**Network Traffic** — HTTP requests intercepted in plaintext through MobSF's built-in proxy.

---

## Generating the Report

After completing the dynamic testing, click **Generate Report** in MobSF to export a consolidated PDF containing all findings, captured traffic, Frida logs, and vulnerability classifications.

---

## References

- [MobSF Documentation](https://mobsf.github.io/docs)
- [DIVA Android — Payatu](https://github.com/payatu/diva-android)
- [Frida Documentation](https://frida.re/docs/home/)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
