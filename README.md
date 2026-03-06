# Android APK Reverse Engineering — Lab Writeup

**Course:** Mobile Application Security  
**Target:** UnCrackable-Level1.apk  
**Date:** March 4–6, 2026  
**Author:** Gharb

---

## What This Lab Is About

Mobile applications ship as `.apk` bundles. Even without running them, you can learn a surprising amount just by picking apart the package. This writeup walks through the complete process I followed: from setting up a controlled environment, to decompiling Java bytecode, to documenting every security-relevant finding.

No emulator. No device. Pure static analysis.

---

## Tools I Used

| Tool | Purpose |
|------|---------|
| PowerShell | File inspection, hashing, ZIP parsing |
| JADX GUI | APK decompilation and manifest analysis |
| dex2jar | Dalvik bytecode → Java JAR conversion |
| JD-GUI | Java source browsing from the converted JAR |

---

## Step 1 — Building the Lab Environment

I started by creating a dedicated folder at `C:\APK-Analysis` and dropping the APK inside it. Working in an isolated directory keeps forensic artifacts separate from the rest of the system — a basic but important habit.

**Confirming the file is what it claims to be**

APKs are ZIP files with a different extension. The quickest way to verify this is to check the magic bytes at offset 0. A valid ZIP always starts with `50 4B` in hex, which maps to `PK` in ASCII — the signature left by Phil Katz when he designed the format.

I ran a hex dump via PowerShell and confirmed the header matched. Screenshot below:


<img width="1200" height="279" alt="Screenshot 2026-03-04 152123" src="https://github.com/user-attachments/assets/4c683637-3b42-46e0-986b-d54d6c1012e8" />



**Peeking inside without extracting**

Using `System.IO.Compression.ZipFile` directly in PowerShell, I listed the first 20 entries without touching the disk:

```
AndroidManifest.xml
META-INF/CERT.RSA
META-INF/CERT.SF
META-INF/MANIFEST.MF
classes.dex
res/layout/activity_main.xml
res/menu/menu_main.xml
res/mipmap-*/ic_launcher.png  (multiple densities)
resources.arsc
```

**Hashing the file**

Before touching anything else, I locked in a SHA-256 fingerprint. If anything changes during the lab, the hash will catch it.

```
SHA-256: 1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21
```


<img width="1727" height="636" alt="Screenshot 2026-03-04 153134" src="https://github.com/user-attachments/assets/a7fa60b4-dcd5-4cd3-9c7c-a798815aebfa" />


---

## Step 2 — Confirming APK Acquisition

Quick sanity check before moving forward. The file was sitting in `C:\APK-Analysis`, dated 3/4/2026, weighing in at 66 KB (66,651 bytes exactly).

Source: OWASP Mobile Security Testing Guide — UnCrackable Level 1. It's a deliberately vulnerable app built for security training.


<img width="1379" height="552" alt="Screenshot 2026-03-04 151740" src="https://github.com/user-attachments/assets/171a216a-ab28-47cb-a78a-9e0775c6e698" />

---

## Step 3 — Tearing Apart the Manifest

The `AndroidManifest.xml` is the first file I always read. It's the app's ID card — package name, SDK targets, declared components, permissions, and security flags. Everything the system needs to know before launching the app lives here.

I opened the APK in JADX GUI and navigated directly to the manifest.

**Package identity**

```xml
package="owasp.mstg.uncrackable1"
android:versionName="1.0"
android:minSdkVersion="19"
android:targetSdkVersion="28"
```

So this app supports anything from Android 4.4 KitKat onwards, targeting Android 9 Pie behavior.

**Permissions**

Zero. Not a single `<uses-permission>` tag anywhere. The app doesn't ask for camera, location, contacts, storage — nothing. That's unusual but not suspicious for a standalone challenge app.

**Components and attack surface**

Only one activity was declared:

```
sg.vantagepoint.uncrackable1.MainActivity
```

It has an `<intent-filter>` with `android.intent.action.MAIN` and `android.intent.category.LAUNCHER`. Because of that filter, Android treats this activity as implicitly exported — meaning other apps on the device can send intents to it directly.

**Security flags worth noting**

- `android:debuggable` — not set → ✅ good
- `android:usesCleartextTraffic` — not set → ✅ good  
- `android:allowBackup="true"` — **present** → ⚠️ risk

The backup flag is the one finding here. With `allowBackup` enabled, anyone with ADB access can dump the app's private data directory without root:

```bash
adb backup -noapk owasp.mstg.uncrackable1
```

For a training app this is acceptable. In production it's a data leakage vector.

<img width="1321" height="739" alt="Screenshot 2026-03-04 153629" src="https://github.com/user-attachments/assets/fad9229f-ed0a-495e-9c91-6ae231e8adb2" />


---

## Step 4 — Hunting for Hardcoded Secrets

Developers sometimes leave sensitive strings baked into the binary — API keys, tokens, internal URLs, debug credentials. JADX's text search covers classes, methods, fields, code, resources, and comments all at once.

**Search #1 — "http"**

Three hits, all pointing to the same thing:

```
http://schemas.android.com/apk/res/android
```

This appears in `AndroidManifest.xml`, `activity_main.xml`, and `menu_main.xml`. It's the standard Android XML namespace URI — not a live endpoint, not a leak. Every Android project includes this string automatically.

Risk rating: **none**.

<img width="977" height="608" alt="Screenshot 2026-03-06 160317" src="https://github.com/user-attachments/assets/36c20060-dc56-4cc7-8614-68ff87c87525" />


No sensitive URLs, credentials, or tokens were found in this APK.

---

## Step 5 — Extracting and Converting the DEX Bytecode

Android compiles Java into Dalvik Executable (`.dex`) format rather than standard JVM bytecode. To use classic Java decompilers like JD-GUI, the DEX needs to be translated first.

**Extracting classes.dex**

I created an output directory and used PowerShell's ZIP API to pull only the DEX file out of the APK:

```powershell
mkdir dex_out
$zip = [System.IO.Compression.ZipFile]::OpenRead("C:\APK-Analysis\UnCrackable-Level1.apk")
$zip.Entries | Where-Object { $_.Name -like "classes*.dex" } | ForEach-Object {
    [System.IO.Compression.ZipFileExtensions]::ExtractToFile($_, "C:\APK-Analysis\dex_out\$($_.Name)", $true)
}
$zip.Dispose()
```

Result: `classes.dex` (5,528 bytes) landed in `dex_out`.

<img width="951" height="291" alt="Screenshot 2026-03-06 160552" src="https://github.com/user-attachments/assets/2392be05-c8fa-4edb-9ba3-9b700ecf5748" />


<img width="1053" height="349" alt="Screenshot 2026-03-06 160623" src="https://github.com/user-attachments/assets/ac649ecd-687a-41cb-bbde-3f307a92aaea" />


<img width="1346" height="480" alt="Screenshot 2026-03-06 160922" src="https://github.com/user-attachments/assets/f1b9a852-7016-46dd-bdb9-8bb25ac60eb3" />


**Running dex2jar**

```powershell
cd C:\APK-Analysis\dex2jar
.\d2j-dex2jar.bat "C:\APK-Analysis\dex_out\classes.dex" -o "C:\APK-Analysis\app.jar"
```

Output: `classes.dex -> C:\APK-Analysis\app.jar` — conversion successful.

<img width="1049" height="89" alt="Screenshot 2026-03-06 161358" src="https://github.com/user-attachments/assets/2f6445e8-2808-4cc0-80a4-bff7a28061fe" />


---

## Step 6 — JADX vs JD-GUI: Which One Wins?

Both tools reconstruct Java source from compiled bytecode. But they're not equals.

**Where JADX stands out**

JADX ingests APKs natively. It decodes binary XML, reconstructs resource references, deobfuscates where it can, and presents everything — manifest, resources, and code — inside a single unified browser. For Android work, this is the gold standard.

**Where JD-GUI fits in**

JD-GUI is a general-purpose Java decompiler. It knows nothing about Android-specific formats. Feed it a JAR and it shows you classes — nothing more. Renamed variables stay renamed. Resource IDs stay as integers. You lose context.

**When to reach for dex2jar + JD-GUI**

Sometimes JADX chokes. Heavily protected APKs with custom class loaders, multi-dex setups with non-standard merging, or aggressive obfuscation can cause JADX to fail partially or entirely. In those situations, dex2jar often manages to convert the DEX regardless, and JD-GUI can at least get you partial source to work with.

**Bottom line**

Use JADX first, every time. Fall back to the dex2jar → JD-GUI pipeline when JADX hits a wall.

---

## Step 7 — Security Assessment Summary

### Application Profile

| Field | Value |
|-------|-------|
| Package | `owasp.mstg.uncrackable1` |
| Version | 1.0 |
| SDK Range | 19 – 28 |
| Permissions | None |
| Network Traffic | N/A (no network calls detected) |
| Exported Components | MainActivity (implicit) |

### Vulnerability Table

| # | Finding | Severity | Location |
|---|---------|----------|----------|
| 1 | `android:allowBackup="true"` enables ADB data extraction | Medium | `AndroidManifest.xml` |
| 2 | Anti-debug checks present (`isDebuggerConnected`) | Info | Java source (MainActivity) |
| 3 | MainActivity implicitly exported via intent-filter | Low | `AndroidManifest.xml` |

### Finding Details

**[MEDIUM] Backup Enabled**  
Any user with a USB cable and developer mode enabled can dump the app's `/data/data/` directory. For apps storing tokens, session data, or locally cached credentials, this is a real attack path. Fix: `android:allowBackup="false"`.

**[INFO] Anti-Debug Logic**  
The app detects debugger attachment at runtime. This is a protection, not a flaw. It signals the developer was aware of runtime analysis and took steps against it — which is exactly what you'd expect from a crackme challenge.

**[LOW] Exported Entry Point**  
`MainActivity` is reachable by external intents because it handles `MAIN/LAUNCHER`. Standard behavior for a launcher activity. No data processing from incoming intents was identified, so the practical risk here is minimal.

### Recommendations

1. Set `android:allowBackup="false"` before any production deployment
2. Review any Intent extras parsed by `MainActivity` for injection risks
3. Current configuration (no permissions, no cleartext traffic, no debug flag) is solid — keep it

---

## Step 8 — Cleanup

After wrapping up the analysis, intermediate files were cleared:

```powershell
Remove-Item -Recurse -Force .\dex_out
Remove-Item .\UnCrackable-Level1.apk
```

The converted `app.jar` was archived under `\results` for future reference. No sensitive data was handled during this lab — the target is a public training application.

---

*Lab completed. All findings documented. Environment cleaned.*
