# 🔐 Lab 4 : Static Analysis of an Android APK

## 🎯 Lab Overview
The goal of this lab is to explore the internal architecture of an Android application (`.apk` file), dissect its key components, and identify potential security weaknesses — all without running the application on a device. This approach is known as **static analysis**.

## 🛠️ Work Environment
* **Operating System :** Windows 10/11
* **Shell :** PowerShell (Administrator)
* **Target Application :** `UnCrackable-Level1.apk` (OWASP MSTG challenge)

---

## 📝 Task 1 : Workspace Setup & APK Integrity Check

Before diving into analysis, a clean and isolated working environment was established, and the APK file was verified to ensure its integrity had not been compromised.

### 1. Setting Up the Working Directory
A dedicated folder `C:\APK-Analysis` was created and the `.apk` file was placed inside.

![Workspace creation and initial file verification](images/1.png)

### 2. Verifying the ZIP Magic Bytes
Android APK files are ZIP archives under the hood. PowerShell was used to read the file's first bytes in hexadecimal, confirming the presence of the ZIP magic signature (`50 4B` → `PK`).

### 3. Listing Internal APK Contents
By leveraging .NET's compression libraries directly in PowerShell, the 20 first entries of the APK were listed without extracting anything. Key files observed: `AndroidManifest.xml`, `classes.dex`, layout resources.

### 4. Computing the SHA-256 Hash
A SHA-256 fingerprint was calculated to establish a baseline — ensuring the file remains untouched throughout the audit.

![APK contents listing and SHA-256 hash computation](images/2.png)

> **SHA-256 :** `1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21`

---

## 📦 Task 2 : APK Acquisition & Validation

**Summary :** This step focuses on confirming the APK's presence in the environment and documenting its origin before any analysis begins.

**Validation Checklist :**
* ✅ **File Present :** The APK is confirmed in `C:\APK-Analysis`.
* ✅ **Origin :** OWASP MSTG UnCrackable Level 1 — a public security training application.
* ✅ **File Size :** 66 KB (66,651 bytes).

![APK visible in the working directory](images/3.png)

---

## 🔍 Task 3 : Manifest Analysis with JADX GUI

**Summary :** JADX GUI was used to decompile the APK and inspect its `AndroidManifest.xml`. This file acts as the application's declaration — it defines permissions, components, and security configurations.

### 1. Application Identity
Extracted from the manifest:
* **Package name :** `owasp.mstg.uncrackable1`
* **versionName :** `1.0`
* **minSdkVersion :** `19` (Android 4.4 KitKat)
* **targetSdkVersion :** `28` (Android 9 Pie)

### 2. Permissions Analysis
* **Finding :** Zero `<uses-permission>` tags found. The application requests no system-level permissions (no camera, no location, no contacts access).

### 3. Declared Components
* One activity declared : `sg.vantagepoint.uncrackable1.MainActivity`
* ⚠️ **Security note :** This activity includes an `<intent-filter>` (`android.intent.action.MAIN`), which implicitly exports it, making it reachable by other apps or automation tools.

### 4. Security Configuration Flags
* **CleartextTraffic / Debuggable :** Neither `android:usesCleartextTraffic="true"` nor `android:debuggable="true"` were found — good security hygiene.
* 🚨 **Issue detected :** `android:allowBackup="true"` is present. This flag allows anyone with physical or ADB access to extract the app's private data using `adb backup`.

![AndroidManifest.xml open in JADX GUI](images/4.png)

---

## 🕵️ Task 4 : Sensitive String Search

**Summary :** JADX GUI's global text search was used to hunt for hardcoded sensitive data — such as API keys, passwords, or hidden URLs — that developers may have left in the source code.

Per the lab's severity grid, here is the report for the observations made.

### Observation 1 : URL Pattern Search (`http`)
* **Match found :** `http://schemas.android.com/apk/res/android`
* **Location :** `AndroidManifest.xml`, `res/layout/activity_main.xml`, `res/menu/menu_main.xml`
* **Risk Level :** Low 🟢
* **Analysis :** These are standard Android namespace declarations required by the XML schema — not data leaks. No sensitive endpoints were discovered.

![Text search for "http" across the APK in JADX](images/5.png)

---

## 🔄 Task 5 : DEX to JAR Conversion with dex2jar

**Summary :** Android bytecode is stored in `.dex` (Dalvik Executable) format, which is not directly readable by standard Java tools. This task extracts the DEX file and converts it into a `.jar` archive for use with Java decompilers.

### 1. Creating the Extraction Folder
A dedicated output folder `dex_out` was created inside the workspace.

![Creating the dex_out output directory](images/6.png)

### 2. Extracting the DEX File
PowerShell was used to open the APK as a ZIP archive and extract only the `classes.dex` file into `dex_out`.

![Extracting classes.dex from the APK](images/7.png)

![Confirming classes.dex extraction from UnCrackable-Level1.apk](images/8.png)

### 3. Converting DEX to JAR
The `dex2jar` command-line tool was then used to translate the Dalvik bytecode into standard Java bytecode:

```powershell
cd C:\APK-Analysis\dex2jar
.\d2j-dex2jar.bat "C:\APK-Analysis\dex_out\classes.dex" -o "C:\APK-Analysis\app.jar"
```

The conversion completed successfully, producing `app.jar`.

![dex2jar converting classes.dex into app.jar](images/9.png)

---

## ⚖️ Task 6 : JADX GUI vs JD-GUI — Tool Comparison

**Summary :** Two decompilation tools were compared side by side using the same target class: `sg.vantagepoint.uncrackable1.MainActivity`. JADX works directly on the APK, while JD-GUI reads the `.jar` file generated by dex2jar.

### Comparison Table

| Criteria | JADX GUI | JD-GUI |
| :--- | :--- | :--- |
| **Input Format** | Works natively on `.apk` files | Requires a `.jar` file (via dex2jar) |
| **Resource Access** | Full access to XML resources, manifest, assets | No access to Android resources — Java only |
| **Code Readability** | Better variable naming reconstruction | Tends to preserve obfuscated names (`a`, `b`, `c`) |
| **Workflow** | All-in-one: decompile & browse in one step | Two-step: convert DEX → JAR, then open in JD-GUI |

### Verdict

* **JADX GUI** is the superior tool for Android static analysis. Its native understanding of the APK format, combined with access to resources and improved readability, makes it the go-to choice for most audits.
* **JD-GUI + dex2jar** remains a valuable fallback. In cases where JADX fails to decompile heavily obfuscated or protected APKs, this pipeline can sometimes bypass those protections and yield partial source code.

---

## 📋 Task 7 : Static Analysis Mini-Report

# Static Analysis Report — UnCrackable Level 1

## A) General Information
- **Analysis Date :** March 4, 2026
- **Analyst :** Gharb (username)
- **APK Analyzed :** `UnCrackable-Level1.apk`
- **Version :** `1.0` (targetSdk: 28, minSdk: 19)
- **Source :** OWASP MSTG training application (publicly available)
- **Tools Used :** PowerShell, JADX GUI, dex2jar, JD-GUI

---

## B) Executive Summary

This static analysis uncovered **1 configuration-level vulnerability** and identified **built-in anti-tampering mechanisms** within the UnCrackable Level 1 application.

The primary concern is the enabled backup flag (`allowBackup`), which could lead to local data exposure. The application is otherwise well-configured — it requests no dangerous permissions and does not expose unencrypted network traffic.

**Overall Risk Level : Medium 🟡**

**Priority Actions :**
1. Set `android:allowBackup="false"` in the manifest to prevent ADB-based data extraction.
2. Validate any data processed by `MainActivity` against malformed or malicious Intent inputs.

---

## C) Detailed Findings

### Finding #1 — Backup Data Extraction Allowed
- **Severity :** Medium 🟡
- **Description :** The `android:allowBackup="true"` flag is set in the application manifest, enabling anyone with ADB access to extract the app's private storage.
- **Location :** `AndroidManifest.xml` → `<application>` tag
- **Potential Impact :** An attacker with physical USB access to an unlocked device could run `adb backup -noapk owasp.mstg.uncrackable1` and retrieve sensitive local data.
- **Recommended Fix :** Explicitly set `android:allowBackup="false"` for any production application that handles sensitive user data.

### Finding #2 — Anti-Debug Protections Detected
- **Severity :** Informational 🟢
- **Description :** The application actively checks whether a debugger is attached at runtime using `android.os.Debug.isDebuggerConnected()`.
- **Location :** Java source classes (identified via JADX)
- **Potential Impact :** Not a vulnerability — this is a protection measure. It raises the bar for dynamic analysis and exploitation.
- **Recommended Fix :** Good practice already in place. No action required.

### Finding #3 — MainActivity Implicitly Exported
- **Severity :** Low 🟢
- **Description :** Because `MainActivity` defines an `<intent-filter>`, Android marks it as exported by default, meaning external applications can potentially invoke it.
- **Location :** `AndroidManifest.xml` → `sg.vantagepoint.uncrackable1.MainActivity`
- **Potential Impact :** As the main launcher entry point, this is expected behavior. However, any Intent data received should be strictly validated to prevent Intent injection.
- **Recommended Fix :** Ensure all data arriving via the launch Intent is properly sanitized before processing.

---

## D) Appendix

### Permissions Declared
- *None* — no `<uses-permission>` tags present in the manifest.

### Exported Components
| Component | Type | Export Reason |
| :--- | :--- | :--- |
| `sg.vantagepoint.uncrackable1.MainActivity` | Activity | Implicit — `intent-filter` present |

---

## 🧹 Task 8 : Cleanup & Lab Wrap-Up

**Summary :** The final step involved reviewing findings, organizing deliverables, and removing temporary analysis artifacts from the workspace.

### 1. Review & Organization
* **Data Sensitivity Check :** Since this is an OWASP training app, no real credentials, tokens, or production data were handled during the audit.
* **Deliverables Archived :** The decompiled `app.jar` was stored in a `/results` subfolder for reference.

### 2. Removing Temporary Artifacts
To maintain a clean and secure environment, intermediate files were deleted after use:

```powershell
Remove-Item -Recurse -Force .\dex_out
Remove-Item .\UnCrackable-Level1.apk
```

> ✅ Workspace cleaned. Analysis complete.
