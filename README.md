# RootKeep

RootKeep is a **fully independent Android root framework** — your own, built from scratch, with **no Magisk** anywhere in the stack. It grants and gatekeeps root on a **Samsung Galaxy S9 (SM-G960F / Exynos 9810 / starlte / Android 10)**.

Unlike Magisk (which uses a systemless overlay), RootKeep is its own custom kernel + init shim + root daemon + manager app. Root is delivered entirely by code you control.

---

## What's in this folder

| File | What it is |
|---|---|
| `README.md` | This guide. |
| `RootKeep-app-v0.7.apk` | The RootKeep manager app (`com.rootkeep`, v0.5-dev). Install on the phone. |
| `Margrinakernel-selfheal-v3.tar.md5` | **Known-good boot image.** Flash this to get root. Your safe fallback. |

Both `.tar.md5` files are **Odin AP** images and contain **only the `boot` partition** (kernel + ramdisk). They do **not** wipe your data.

---

## How it works (the short version)

The device is **system-as-root**: the `/` filesystem *is* the SYSTEM partition — there is **no separate "root" partition**. The boot image carries three things stitched together:

1. **Margrinakernel** — a custom kernel built from Samsung's open source (4.9.118). Its SELinux denial path is patched to be **globally permissive**, and Samsung's security blockers (DEFEX, PROCA, the RKP/UH suite) are disabled. This is what makes independent root possible — the stock kernel is enforcing and *ignores* SELinux "permissive" domains, so a custom kernel is mandatory.
2. **rkinit** — a tiny first-stage `/init` shim. It launches the root daemon, then hands off to the real Samsung init (kept as `/init.real`) so the phone boots normally.
3. **sud** — the root daemon. It listens on a private socket and is the **gatekeeper**: every app that wants root must be approved by you in the RootKeep app. It identifies callers by the kernel (not by anything the caller claims), so it can't be spoofed.

The **app** talks to the daemon to show root status, grant/deny root per app, install the `su` command, and flash partitions.

---

## Quick start

> You need: a Windows PC with **Odin3**, the Samsung USB driver, and this folder.
> Your bootloader must be **unlocked** (it already is — `orange` state, Knox tripped).

### 1. Flash the boot image
1. Power the phone **off**.
2. Enter **Download mode**: hold **Volume Down + Bixby**, then plug in USB, then press **Volume Up**.
3. Open **Odin3**. The top-left box shows **ID:COM** highlighted = phone detected.
4. Click **AP** → select **`Margrinakernel-selfheal-v3.tar.md5`**.
5. Leave **BL / CP / CSC / USERDATA empty**. Options → **Auto Reboot ON**, **Re-Partition OFF**.
6. Click **Start**. Wait for **PASS!** The phone reboots.

> ⚠️ **Only ever flash ONE boot image to AP.** `Margrinakernel-boot` and `Margrinakernel-selfheal-v2` are *alternatives*, not layers — flashing one replaces the other. Never flash old `RootKeep-shim-TEST-v*.tar` or `vendor-permissive` files: they use the stock kernel and will break root.

### 2. Install the app
Copy `RootKeep-app-v0.7.apk` to the phone and tap it (allow "install unknown apps"), **or** over adb:

```
adb install -r RootKeep-app-v0.7.apk
```

### 3. Verify root
After boot, open RootKeep — Home should show **ROOT STATUS: Active**. Or from a PC:

```
adb shell "uname -r; su -c id"
```

Expect `4.9.118-Margrinakernel` and `uid=0(root)`.

---

## Using the app

- **Home** — live root status, device/boot state, and the linked-image badge.
- **Superuser** — the gatekeeper list. Every app is shown with **Allow / Deny**. Tapping **Allow** on *"Shell (adb / rksu)"* is what gives **ADB** root. Grant other apps the same way. Standard "root checker" apps will only see root **after** you Allow them here — that's the security model, not a bug.
- **Flash** — write a `.img` or Odin `.tar` to a partition, as root. **It auto-detects the target partition from the file** (e.g. `recovery.img` → RECOVERY, `boot.img` → BOOT), resolves partition names regardless of case, refuses an image larger than the partition, and streams a live log.
- **Settings** — live device facts, plus **Install su** (puts `su` in `/system/bin` + `/system/xbin`) and a **setuid** toggle.

### su in a terminal / adb
Once ADB is allowed in the Superuser tab and `su` is installed:

```
adb shell su -c id          # -> uid=0(root)
```

---

## Recovery (if something goes wrong)

- **A boot image hangs at the Samsung logo** → you're *not* bricked. Re-enter Download mode and flash **`Margrinakernel-selfheal-v3.tar.md5`** (the known-good one) to AP. No data loss.
- Root works but **can't grant apps** right after a clean flash → the daemon's config dir was wiped. Fix over adb (no root needed — `shell` owns `/data/local/tmp`):

```
adb shell mkdir -p /data/local/tmp/rk
```

Then grant apps in the Superuser tab as normal. (The `selfheal-v2` boot does this automatically.)

---

## Flash order (full sequence)

Each line says **which Odin slot** the file goes in. Golden rule: **one Odin operation per step, and only the listed file(s) loaded** — never mix steps, and never put two boot images in AP at once.

1. **Stock Android 10 firmware** — *only if you want a clean slate.* Load the 4 firmware files in one Odin flash: `BL_…` → **BL**, `AP_…` → **AP**, `CP_…` → **CP**, `HOME_CSC_…` → **CSC** (use **HOME_CSC** to keep data; plain **CSC** wipes it). Start → PASS → boot into Android once to confirm stock is healthy. *(Skip entirely if the phone already boots Android 10.)*
2. **Root — `Margrinakernel-selfheal-v3.tar.md5`** → **AP** *(this file ALONE, nothing in BL/CP/CSC)*. Start → PASS → reboot. If **"Verification failed"** appears, see the next section.
3. **Install the app — `RootKeep-app-v0.7.apk`** *(not a flash)* — tap it on the phone, or `adb install -r RootKeep-app-v0.7.apk`.
4. **Grant / bootstrap** *(not a flash)* — open RootKeep → **Superuser** → **Allow** on *"Shell (adb / rksu)"* for ADB root. If grants don't save right after a clean flash: `adb shell mkdir -p /data/local/tmp/rk` first.
5. **(Optional) TWRP** — `twrp-…-recovery.img.tar` → **AP**, or via the app's **Flash** tab (auto-targets RECOVERY). Independent of root; only if you want custom recovery.

If the phone already boots Android 10, you only need **2 → 3 → 4**.

---

## "Verification failed" — what it means and how to avoid losing data

> *"Unable to restart your device. The integrity verification has failed. You need to reset your device to factory default settings. This will erase all your data."*

This is **not** a skippable prompt — it's the bootloader's encryption gate. On this S9 (Android 10, **file-based encryption**), the keys that decrypt `/data` are tied to the **verified-boot state**. When the running boot image doesn't match the state `/data` was encrypted under, the phone literally **cannot decrypt `/data`**, so it offers only **RESET** (which wipes `/data` to clear the mismatch). There is no data-preserving "ignore" — the data genuinely can't be opened.

**Key fact:** after a wipe, `/data` is re-encrypted **on the next boot, under whichever boot image is running then.** So moving `/data` from the stock state to the Margrinakernel state costs **exactly one** wipe — the goal is to not wipe *twice*.

| What you do | Result |
|---|---|
| Reset → **boot into stock Android** → *then* flash Margrinakernel | `/data` re-encrypts under **stock** → Margrinakernel mismatches → **Verification failed again** ❌ (two wipes for nothing) |
| Reset → **go straight to Download mode, do NOT boot the OS** → flash Margrinakernel → boot | `/data` encrypts fresh under **Margrinakernel** → **no screen** ✅ |
| Flash Margrinakernel → screen appears → tap **RESET** | reboots straight into Margrinakernel, `/data` set up fresh → clean, and it won't return ✅ |

**Recommended:** just flash Margrinakernel and tap **RESET** when the screen appears — that's the one required wipe, and normal reboots afterward won't show it again.

**Rescue without wiping (worth a try first):** re-flash the **exact** boot image the phone last booted successfully with (usually `Margrinakernel-selfheal-v3.tar.md5`). If `/data` is still encrypted under that state, it boots with no data loss. If it still fails, `/data` is tied to a different state and the reset is unavoidable.

**Stop it forever:** flash **TWRP**, run `multidisabler-samsung-3.1-keep…zip`, and format data once. That leaves `/data` **unencrypted**, so this mismatch can never trigger again on any boot flash — one format up front, no dance afterward.

---

## Notes & caveats

- **`Margrinakernel-selfheal-v2`** installs `su` and repairs its config *after boot* (triggered by the app on first launch), avoiding the boot-time operation that could hang the phone. It is **statically verified but flash it only when you're ready to test** — keep the known-good image as your fallback.
- Knox is tripped (`0x1`) and the bootloader is unlocked — expected for any custom-kernel setup; not reversible.
- The app's *"linked image · verified"* badge on Home is a cosmetic checksum of a bundled file, **not** what provides root.
- This is a personal, from-scratch project for one specific device. Don't flash these images on any other phone.

---

*RootKeep — independent Android root for the Galaxy S9. Kernel · init shim · root daemon · manager app, all your own.*
