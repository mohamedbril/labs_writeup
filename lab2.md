# Android Security Lab — Rooting & Verified Boot

---

## Scope

| Field | Value |
|---|---|
| **App + Version** | app-debug.apk (à compléter) |
| **Environment** | AVD (Android Virtual Device) — émulateur de laboratoire |
| **Objective** | Comprendre le rooting Android et ses impacts sur la sécurité système |
| **Data** | Fictives |
| **Network** | Test isolé |

---

## Step 1 — Clean AVD Setup

Start a fresh AVD from Android Studio → Device Manager. Prefer API 29+ for modern security mechanisms.

Verify detection:

```bash
adb devices
```

> **Important:** Never reuse a previously tested AVD. Residual apps or configs will compromise your results.

---

## Step 2 — Root the AVD

Rooting grants elevated privileges on the device, allowing access to normally protected system areas.

### Launch with writable system

```bash
emulator -avd NOM_AVD -writable-system
```

### Enable root & remount

```bash
adb root       # Restarts ADB daemon with root privileges
adb remount    # Remounts /system as read-write
```

### Verify root state

```bash
adb shell id                                    # Expect: uid=0(root)
adb shell getprop ro.boot.verifiedbootstate     # Expect: orange or yellow
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"                            # Test su availability
```

**Interpretation:**
- `uid=0(root)` → root confirmed
- `orange` / `yellow` → system integrity no longer guaranteed
- `green` → signed, unmodified image

### Optional — Disable verity

```bash
adb disable-verity
adb reboot
adb remount
```

Verity is a filesystem integrity mechanism. Disabling it allows modifications but removes the integrity guarantee.

### Collect logs

```bash
adb logcat -d | tail -n 200 > logcat_root_check.txt
```

> Always keep logs during security tests. They document your actions and help reproduce or debug findings.

---

## Step 3 — Install & Launch the Test App

```bash
adb install app-debug.apk
```

Confirm the app launches and a basic scenario is reachable. Note the version in the report.

---

## Step 4 — Define 3 Repeatable Scenarios

| # | Scenario | Steps |
|---|---|---|
| 1 | Open home screen | Launch app, observe main UI |
| 2 | Search an item | Tap search, enter fixed keyword, observe results |
| 3 | Open a detail view | Tap an item, observe detail screen |

> Use fixed inputs (no random data). Include a screenshot per step.

---

## Step 5 — Android Security Model (Summary)

Source: [source.android.com/docs/security](https://source.android.com/docs/security)

Android security relies on layered protections:

1. **App sandboxing** — each application is isolated from others at the OS level
2. **Permission model** — explicit access control to sensitive resources (camera, location, contacts…)
3. **System integrity** — kernel and platform protections prevent unauthorized modification

Rooting bypasses all three layers simultaneously.

---

## Step 6 — Verified Boot

Source: [source.android.com/docs/security/features/verifiedboot](https://source.android.com/docs/security/features/verifiedboot)

### What is Verified Boot?

Verified Boot ensures that the system that boots is exactly the one intended by the manufacturer — with no unauthorized modifications. If anything has been altered, the device alerts the user or refuses to boot entirely.

### Chain of Trust (2 lines)

A chain of trust is a sequence of verifications where each component validates the authenticity of the next before trusting it. Starting from immutable hardware (ROM), each stage — bootloader, kernel, system — is cryptographically verified before execution.

### Why is boot integrity critical?

If the boot process is compromised, every security mechanism above it can be bypassed. A rootkit at the bootloader level is invisible to the OS and all applications running on top of it.

### Check on AVD

```bash
adb shell getprop ro.boot.verifiedbootstate
```

| Color | Meaning |
|---|---|
| `green` | System verified and unmodified |
| `yellow` / `orange` | Modified but functional (custom key or unlocked) |
| `red` | Integrity compromised — do not trust |

---

## Step 7 — Android Verified Boot (AVB)

Source: [source.android.com/docs/security/features/verifiedboot/avb](https://source.android.com/docs/security/features/verifiedboot/avb)

AVB (version 2.0) extends Verified Boot with a more flexible and modern integrity model. It adds:

- **Hashtree verification** on each partition (system, vendor, boot…)
- **Rollback protection** — prevents downgrading to older firmware versions that may carry known vulnerabilities
- **VBMeta structure** — a signed metadata structure that chains all partition digests together

### Check AVB state (lab device with unlocked fastboot)

```bash
fastboot getvar avb_boot_state
fastboot oem device-info
```

> For a temporary demo without flashing:
> ```bash
> fastboot boot magisk_patched.img
> ```
> **Warning:** Never flash or manipulate a personal device. These operations can permanently brick hardware and void warranties.

---

## Observations & Notes

| Check | Result | Notes |
|---|---|---|
| `adb shell id` | | |
| `ro.boot.verifiedbootstate` | | |
| `ro.boot.vbmeta.device_state` | | |
| App installs correctly | | |
| Scenario 1 reproducible | | |
| Scenario 2 reproducible | | |
| Scenario 3 reproducible | | |

---

## Conclusion

This lab demonstrates the practical impact of rooting an Android Virtual Device. By escalating privileges through `adb root` and disabling verity, the system's integrity guarantees are removed — evidenced by the boot state shifting from `green` to `orange`. The three-layer security model (sandboxing, permissions, integrity) depends on the boot chain remaining uncompromised. AVB and Verified Boot are the first line of defense; once bypassed, upper-layer controls lose their trustworthiness.
