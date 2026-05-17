# LAB 17 — OWASP UnCrackable Android Level 3

> **EL YAMANI OMAYMA**

---

## Overview

This lab walks through cracking OWASP's UnCrackable Android Level 3 challenge. The app protects a secret string using root detection, tamper detection, anti-debug, anti-Frida guards, and a native XOR-obfuscated verification routine buried inside a heavily obfuscated `.so` library.

**Tools used:** `apktool` · `jadx-gui` · `Ghidra` · `apksigner` · `adb`

---

## Learning Objectives

- Decompile and patch an APK at the Smali level
- Analyze a native `.so` library with Ghidra
- Bypass anti-root, anti-debug, anti-Frida, and integrity checks
- Reverse a XOR byte-by-byte verification routine
- Recover the secret key without running Frida or any paid tool

---

## Architecture of Protections

```
MainActivity
├── verifyLibs()        → CRC check on .so and classes.dex (tamper detection)
├── onCreate()          → Root + debuggability + tampered flag checks → showDialog()
└── verify()            → Delegates to CodeCheck.check_code() [native]

libfoo.so (.init_array)
├── sub_73D0()          → Anti-Frida (/proc/self/maps scan) + ptrace anti-debug
└── FUN_001012c0()      → XOR verification logic (obfuscated with LCG + malloc chains)
```

---

## Step-by-Step

### 1. Static Analysis — JADX-GUI

Open the APK in jadx-gui and navigate to `sg.vantagepoint.uncrackable3 → MainActivity`.

<img width="816" height="375" alt="image" src="https://github.com/user-attachments/assets/e707aadc-3f58-4bc1-a144-1d08a153d432" />

<img width="935" height="443" alt="image" src="https://github.com/user-attachments/assets/d05db61a-07dc-4ae2-8ea3-cb00526efc9d" />


Key observations:
- `verifyLibs()` computes CRC of native libs and `classes.dex` via the native `baz()` function. Mismatch sets `tampered = 31337`.
- `onCreate()` calls `showDialog()` and exits if root, debugger, or tamper is detected.
- `System.loadLibrary("foo")` loads `libfoo.so`.
- The actual password check is in `CodeCheck.check_code()` — a native method.

---

### 2. Decompile with apktool

```bash
apktool d UnCrackable-Level3.apk -o uncrackable3
```

Output: `smali/` bytecode + `lib/` native libraries.

<img width="824" height="253" alt="image" src="https://github.com/user-attachments/assets/a4331292-b69c-4205-8025-52c024f6dd38" />



---

### 3. Patch Smali — Bypass Root/Tamper Dialog

File: `uncrackable3/smali/sg/vantagepoint/uncrackable3/MainActivity.smali`

Search for `showDialog` (2nd occurrence, inside `onCreate`). Replace:

```smali
# BEFORE
const-string v0, "Rooting or tampering detected."
invoke-direct {p0, v0}, Lsg/vantagepoint/uncrackable3/MainActivity;->showDialog(Ljava/lang/String;)V

# AFTER
return-void
```

Optionally, gut the `showDialog` method entirely:

```smali
.method private showDialog(Ljava/lang/String;)V
    .locals 3
    return-void
.end method
```

---

### 4. Patch Native Library — Bypass Anti-Debug / Anti-Frida

Open `libfoo.so` in Ghidra. Locate `sub_73D0` (registered in `.init_array`).

This function:
- Scans `/proc/self/maps` for `frida` or `xposed` strings
- Uses `ptrace` to detect debuggers
- Calls `goodbye()` on detection

**Patch:** Replace the first instruction of `sub_73D0` with `RET`.

```
Edit → Patch Instruction → RET
```

Export the patched `.so` and replace the original in `uncrackable3/lib/arm64-v8a/` (or `x86_64/`).

---

### 5. Rebuild, Sign, and Install

```bash
# Rebuild
apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
```

<img width="855" height="169" alt="image" src="https://github.com/user-attachments/assets/4a1dccc3-af51-4a86-9ef5-67ac8ff4e28e" />




```bash
# Install
adb uninstall owasp.mstg.uncrackable3
adb install -r UnCrackable-Level3-patched.apk
```
<img width="720" height="75" alt="image" src="https://github.com/user-attachments/assets/97e69696-b91a-433b-87ab-1889d9a5796d" />


---

### 6. Reverse the Native Verification Logic

In Ghidra, follow the call chain:

```
Java_sg_vantagepoint_uncrackable3_Check_check_code
    └── FUN_001012c0
```

`FUN_001012c0` is heavily obfuscated with a repeating LCG + `malloc` pattern (~90 iterations). This is a signature of **Tigress / O-LLVM obfuscation** — ignore it.

Focus on the **end of the function**: three `qword` writes fill a 24-byte buffer with the encoded key:

```
1d 08 11 13 0f 17 49 15  0d 00 03 19 5a 1d 13 15
08 0e 5a 00 17 08 13 14
```

The verification loop then XORs this buffer against a key derived from `DAT_00107040/41/42` and compares byte-by-byte with user input. Input must be exactly **24 bytes**.

---

### 7. Decode the Secret Key

```python
encoded = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")
xor_key = b"pizzapizzapizzapizzapizzapizza"

secret = bytes(a ^ b for a, b in zip(encoded, xor_key))
print("Secret:", secret.decode())
```

**Output:**
```
Secret: making owasp great again
```
<img width="576" height="48" alt="WIII" src="https://github.com/user-attachments/assets/c9373fd6-9cfb-40a8-8736-cfbbbfb6dee6" />


Enter this string in the app → **Success!**

<img width="168" height="308" alt="DONE" src="https://github.com/user-attachments/assets/bbed28b7-88e5-4579-a875-b9bf230ff40d" />


---

## Key Takeaways

| Technique | Purpose |
|-----------|---------|
| CRC integrity check | Detect APK modification |
| `tampered` flag via native `baz()` | Bind Java state to native layer |
| `ptrace` + `/proc/self/maps` scan | Anti-debug + Anti-Frida |
| LCG + malloc obfuscation | Hide the useful code from static analysis |
| XOR-encoded key in native buffer | Avoid plaintext secrets in memory |

**Lesson:** In obfuscated native code, skip the noise — follow the data flow into the comparison buffer, not every instruction linearly.

---

## References

- [OWASP MSTG UnCrackable Apps](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes)
- [Ghidra](https://ghidra-sre.org/)
- [apktool](https://apktool.org/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)
