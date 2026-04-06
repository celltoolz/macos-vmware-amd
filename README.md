# 🍎 macOS in VMware Workstation Pro on Windows (AMD CPU) — A Survivor's Guide

> **TL;DR:** Yes, you can run macOS on VMware on an AMD system. No, it won't be easy.
> This guide documents what actually worked — including the failures — so you don't have to spend your weekend on it.

![macOS Monterey 12.7.6 running in VMware on an AMD Windows host](images/preview__3_.webp)

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Why Monterey and Not Ventura?](#why-monterey-and-not-ventura)
- [Downloading macOS](#downloading-macos)
- [Installing QEMU](#installing-qemu)
- [Creating the VMDK](#creating-the-vmdk)
- [Preparing VMware with Auto-Unlocker](#preparing-vmware-with-auto-unlocker)
- [Creating the VM](#creating-the-vm)
- [Critical .vmx Configuration](#critical-vmx-configuration)
- [Installing macOS](#installing-macos)
- [Troubleshooting — The Ventura Experiment](#troubleshooting--the-ventura-experiment)
- [Post-Install](#post-install)

---

## Overview

This guide walks through installing **macOS Monterey 12** as a virtual machine inside **VMware Workstation Pro 17** on a **Windows 11 host with an AMD CPU**.

### Who this is for

- Developers who need macOS for UI/compatibility testing (e.g., a Python Tkinter desktop app) but don't own Apple hardware
- People who have already tried following a generic VMware/macOS guide and hit a black screen, kernel panic, or the prohibited symbol (🚫)
- Anyone on **AMD** who found that Intel-focused guides don't quite work

### What you'll end up with

A functional **macOS Monterey 12.7.6** VM running inside VMware on a Windows AMD machine, suitable for app testing and light development work.

### Host environment used in this guide

| Component | Detail |
|---|---|
| Host machine | ServeHer (Windows 11 Pro, Build 26200) |
| RAM | 32 GB |
| CPU | AMD (Ryzen series) |
| VMware Workstation Pro | 17.5.2 build-23775571 |
| Guest OS | macOS Monterey 12.7.6 |
| Guest config | 4 vCPU cores, 8 GB RAM, 80 GB disk |

![VMware Workstation Pro 17.5.2 About dialog confirming version and host details](images/1775450064999_34_vmware_version_info.png)

> ⚠️ **Legal note:** Running macOS in a VM on non-Apple hardware is technically against Apple's EULA. This guide is intended for legitimate development and testing purposes only.

> ⚠️ **Virtualization must be enabled in BIOS.** Before starting, open Task Manager → Performance tab and confirm virtualization is enabled. If not, boot into your BIOS and enable it. Without this, VMware cannot run any virtual machines.

---

## Prerequisites

Gather all of these before you start. Everything links to official sources.

| Tool | Purpose | Where to get it |
|---|---|---|
| **VMware Workstation Pro 17** | The hypervisor | [vmware.com](https://www.vmware.com/products/workstation-pro.html) — free for personal use |
| **Auto-Unlocker v2.0.x** | Patches VMware to allow macOS as a guest | [GitHub: paolo-projects/auto-unlocker](https://github.com/paolo-projects/auto-unlocker/releases) |
| **OpenCorePkg** | Contains `macrecovery.py` to download macOS | [GitHub: acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) — get the RELEASE zip |
| **QEMU for Windows** | Converts `.dmg` to `.vmdk` | [qemu.weilnetz.de](https://qemu.weilnetz.de/w64/) |
| **Python 3** | Required to run `macrecovery.py` | [python.org](https://www.python.org/downloads/) |

### Pre-flight checklist

- [ ] VMware Workstation Pro is installed and has been opened at least once
- [ ] Virtualization is enabled in BIOS (check Task Manager → Performance)
- [ ] Python 3 is on your PATH (`python --version` works in PowerShell)
- [ ] At least **80–100 GB** of free disk space available
- [ ] Auto-Unlocker downloaded but **not yet run** (we'll do that after closing VMware)

---

## Why Monterey and Not Ventura?

If you're here because Ventura didn't work, this section is for you.

macOS on VMware requires spoofing the CPU to look like Intel hardware (via CPUID values in the `.vmx` file). This works — but **macOS Ventura (13) introduced stricter kernel-level CPU validation** that CPUID spoofing cannot fully satisfy on AMD hardware.

The failure isn't obvious because Ventura doesn't fail immediately. It gets through the installer, runs the progress bar to completion, and then kernel panics on the first real reboot. When you add CPUID spoof values to try to fix it, the failure mode changes to the prohibited symbol (🚫) — a *different* and harder failure. We documented this entire sequence. See [Troubleshooting](#troubleshooting--the-ventura-experiment) for the full story.

**macOS Monterey (12)** predates the stricter validation and is stable on AMD VMware installs with the correct CPUID configuration.

| macOS Version | AMD VMware Compatibility | Notes |
|---|---|---|
| Monterey (12) | ✅ Reliable | **Use this.** Stable with correct CPUID spoof. Tested: 12.7.6. |
| Ventura (13) | ⚠️ Unreliable | Installs, then kernel panics on first reboot. Not worth the fight. |
| Sonoma (14) | ❌ Very difficult | Not recommended for AMD VMware. |
| Big Sur (11) | ✅ OK | Works, but older. Monterey is the better choice. |

---

## Downloading macOS

We use **`macrecovery.py`** from the OpenCorePkg project to download a genuine macOS recovery image directly from Apple's servers. This is preferred over random ISOs — the integrity is verified and the source is Apple's own CDN.

### Step 1: Extract OpenCorePkg and navigate to macrecovery

Download the latest **RELEASE** zip from [GitHub: acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases), extract it, and navigate into:

```
OpenCorePkg-X.X.X-RELEASE\Utilities\macrecovery\
```

You'll see `macrecovery.py`, `boards.json`, `recovery_urls.txt`, and a few other files.

![OpenCorePkg macrecovery folder — Shift+right-click to open PowerShell here](images/1775450064997_24_monterey_macrecovery_command.png)

### Step 2: Open PowerShell in this folder

Hold **Shift** and right-click inside the Explorer window (not on a file), then select **"Open PowerShell window here"** or **"Open in Terminal"**.

### Step 3: Download macOS Monterey

Run the following command:

```powershell
py macrecovery.py -b Mac-FFE5EF870D7BA81A -m 00000000000000000 download
```

> **Board ID `Mac-FFE5EF870D7BA81A` targets macOS Monterey 12.** Do not change it if you want Monterey. If you want a different version, look up its board ID in the [OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/installer-guide/windows-install.html#downloading-macos).

The script will download two files and verify the image against Apple's chunklist:

![macrecovery.py download complete — BaseSystem.dmg verified against chunklist](images/1775450064998_25_monterey_downloading.png)

When it finishes you'll see:
- `com.apple.recovery.boot\BaseSystem.dmg` (~623 MB)
- `com.apple.recovery.boot\BaseSystem.chunklist`

### Step 4: Copy BaseSystem.dmg to Documents

Cut or copy the `BaseSystem.dmg` file out of the `com.apple.recovery.boot` subfolder and paste it directly into your **Documents** folder (`C:\Users\<username>\Documents\`). The QEMU conversion step will run from there.

![Documents folder with BaseSystem.dmg ready for conversion (638,371 KB)](images/1775450064998_27_basesystem_dmg_in_documents.png)

---

## Installing QEMU

QEMU is an open-source disk image utility. We only need one tool from it: `qemu-img.exe`, which converts the `.dmg` to a `.vmdk`.

### Step 1: Download QEMU for Windows

Go to [qemu.weilnetz.de/w64/](https://qemu.weilnetz.de/w64/) and download the latest `qemu-w64-setup-XXXXXXXX.exe` installer. Ignore the year-named folders.

### Step 2: Run the installer

Run the installer. If Windows SmartScreen blocks it, click **Run Anyway**. If a file-write error appears during installation, click **Retry**.

QEMU installs to `C:\Program Files\qemu\`.

![QEMU installer language selection](images/1775450064998_28_qemu_install_components.png)

![QEMU setup completing — click Finish](images/1775450064998_29_qemu_install_complete.png)

> 💡 You don't need to add QEMU to your PATH — we'll call it with its full path in the next step.

---

## Creating the VMDK

VMware cannot use a `.dmg` file directly. We need to convert `BaseSystem.dmg` to a `.vmdk` using QEMU.

### Step 1: Open PowerShell in your Documents folder

Navigate to `C:\Users\<username>\Documents\` in Explorer. Hold **Shift** and right-click the empty area of the window, then select **"Open PowerShell window here"**.

### Step 2: Run the conversion

```powershell
& "C:\Program Files\qemu\qemu-img.exe" convert -O vmdk -o compat6 BaseSystem.dmg recovery.vmdk
```

> **Note the exact syntax:** This uses PowerShell's `&` call operator to invoke the executable with its full path. The output file is named `recovery.vmdk`. The `-o compat6` flag ensures VMware compatibility.

![PowerShell: qemu-img convert command running — the prompt hangs here while it works](images/1775450064999_31_qemu_convert_command.png)

The command produces no output while running. When it returns to the prompt with no error message, it succeeded.

![PowerShell: convert complete — returned to prompt with no errors](images/1775450064999_32_qemu_convert_success.png)

### Step 3: Verify the VMDK was created

Back in Explorer, your Documents folder should now show `recovery.vmdk` at approximately **1.72–1.8 GB**.

![Documents folder showing recovery.vmdk (1,809,216 KB) alongside BaseSystem.dmg](images/1775450064999_33_recovery_vmdk_created.png)

---

## Preparing VMware with Auto-Unlocker

By default, VMware Workstation does not allow macOS as a guest OS. **Auto-Unlocker** patches VMware's binaries to add macOS support.

> ⚠️ **Fully close VMware before running Auto-Unlocker.** Check Task Manager to make sure no `vmware.exe` processes are running.

### Step 1: Download and extract Auto-Unlocker

Download the latest release from [GitHub: paolo-projects/auto-unlocker](https://github.com/paolo-projects/auto-unlocker/releases). Extract the zip — you'll see `Unlocker.exe`, a log file location, `backup/`, and `tools/` folders.

![VMware Workstation Pro 17 home screen — click Create a New Virtual Machine](images/1775450065000_36_vmware_home_screen.png)

### Step 2: Run Unlocker.exe

Double-click `Unlocker.exe`. The GUI will open and auto-detect your VMware installation paths.

![Auto-Unlocker extracted folder — Unlocker.exe ready to run](images/1775450065000_37_auto_unlocker_gui.png)

Verify the VMware install location and X64 location fields are populated, then click **Patch**. The tool will download required macOS tools and patch VMware's executables. When it finishes, close the window.

### Step 3: Verify the patch worked

Open VMware Workstation. When creating a new VM, you should now see **Apple Mac OS X** as a guest OS option. If you don't see it, close VMware, re-run Unlocker as administrator, and try again.

---

## Creating the VM

With VMware patched, we can now create the virtual machine. The wizard has quite a few screens — follow each step exactly.

### Step 1: Start the wizard

Open VMware. From the home screen, click **Create a New Virtual Machine**.

![Auto-Unlocker v2.0.0 GUI with VMware paths auto-filled and Patch button ready](images/1775450065000_38_vmware_create_new_vm.png)

### Step 2: Choose Custom configuration

Select **Custom (advanced)** and click Next.

![New VM Wizard — Custom (advanced) selected](images/1775450065000_39_wizard_custom_config.png)

### Step 3: Hardware compatibility

Select **Workstation 17.5.x** from the dropdown. Do not downgrade compatibility.

![Wizard: hardware compatibility set to Workstation 17.5.x](images/1775450064997_40_wizard_hardware_compatibility.png)

### Step 4: Guest OS Installation

Select **"I will install the operating system later."** This skips the disc/ISO selection — we'll attach the recovery disk at the disk selection step below.

![Wizard: I will install the operating system later](images/41_wizard_install_os_later.png)

### Step 5: Guest OS type

Select **Apple Mac OS X** as the guest OS, and choose **macOS 12** from the Version dropdown.

> If you don't see Apple Mac OS X in the list, Auto-Unlocker didn't patch successfully. Go back and re-run it.

![Wizard: Apple Mac OS X selected, version macOS 12](images/42_wizard_select_macos12.png)

### Step 6: Name and location

Name the VM (e.g., `macOS 12`) and note the location. The `.vmx` file we'll edit later lives here.

![Wizard: VM named macOS 12, saved to Documents\Virtual Machines\macOS 12](images/43_wizard_name_macos12.png)

### Step 7: Processor configuration

- **Number of processors:** `1`
- **Number of cores per processor:** `4`

> Don't assign too many cores. macOS VMs on AMD can be unstable with excessive vCPU counts. 4 is a stable, tested value.

![Wizard: 1 processor, 4 cores per processor — 4 total](images/44_wizard_1_processor_4_cores.png)

### Step 8: Memory

Set to **8096 MB (8 GB)**. The minimum is 4 GB, but 8 GB gives a noticeably smoother experience.

![Wizard: 8096 MB memory allocated](images/45_wizard_memory_8gb.png)

### Step 9: Network type

Select **Use network address translation (NAT)**.

![Wizard: NAT networking selected](images/46_wizard_nat_networking.png)

### Step 10: I/O Controller and Disk Type

Accept the defaults on the I/O Controller page. On the **Select a Disk Type** page, leave **SATA** selected (it's recommended — do not change to NVMe).

### Step 11: Select a Disk — attach the recovery VMDK

On the **Select a Disk** page, choose **"Use an existing virtual disk"**.

![Wizard: Use an existing virtual disk selected](images/47_wizard_use_existing_disk.png)

Click **Next**, then **Browse** and navigate to `C:\Users\<username>\Documents\recovery.vmdk`.

![Wizard: recovery.vmdk path entered in the existing disk field](images/48_wizard_browse_recovery_vmdk.png)

VMware will ask if you want to convert the disk to a newer format. Click **Keep Existing Format**.

![VMware: Convert to newer format dialog — click Keep Existing Format](images/49_wizard_keep_existing_format.png)

### Step 12: Review and finish

Review the summary. You should see:
- Hard Disk: `Existing disk C:\Users\...\Documents\recovery.vmdk`
- Memory: 8096 MB
- Network: NAT
- 4 CPU cores

![Wizard: summary showing macOS 12 VM config with recovery.vmdk](images/50_wizard_vm_summary.png)

Click **Finish**. **Do not power on the VM yet.**

### Step 13: Add the 80 GB install disk

The recovery VMDK is the installer/boot disk. macOS needs a separate disk to actually install onto. We'll add it now via VM Settings.

In VMware, right-click the new VM and select **Settings**. The Hardware tab shows the current devices.

![VM Settings after creation — recovery VMDK visible as 3 GB Hard Disk (SATA)](images/51_macos12_vm_created.png)

Click **Add...** → Select **Hard Disk** → Click **Next** → Select **SATA** (recommended).

![Add Hardware Wizard: SATA disk type selected](images/52_add_hardware_hard_disk.png)

Select **Create a new virtual disk**.

![Add Hardware Wizard: Create a new virtual disk](images/53_add_disk_create_new.png)

Set the size to **80 GB** (VMware's recommended size for macOS 12). Select **Store virtual disk as a single file**.

![Add Hardware Wizard: 80.0 GB, Store as single file](images/54_add_disk_80gb_capacity.png)

Accept the default filename and click **Finish**. The VM now has two SATA disks: the `recovery.vmdk` boot disk and a blank 80 GB install destination.

### Step 14: Disable 3D graphics acceleration

Still in VM Settings → **Display** → uncheck **"Accelerate 3D graphics"**. This can cause instability on AMD during installation.

---

## Critical .vmx Configuration

This section is what separates a successful AMD boot from hours of kernel panics. The `.vmx` is VMware's plain-text config file for the VM. We need to add entries that spoof the CPU identity and tune behavior for AMD.

### Finding the .vmx file

Navigate to your VM's folder — for example:
```
C:\Users\Alex\Documents\Virtual Machines\macOS 12\macOS 12.vmx
```

Open it in a proper text editor. **Use Notepad++ or VS Code.** Do not use standard Notepad — it can corrupt line endings and mangle quotes into "smart quotes" that VMware cannot read.

> ⚠️ The VM must be **powered off** before editing. Save and close before powering on.

### The complete AMD config block

Add the following lines to the **end** of your `.vmx` file:

```ini
# Required: disables VMware's SMC version check — without this, macOS will not boot at all
smc.version = "0"

# AMD CPUID spoof — encodes "GenuineIntel" in the vendor string registers
# macOS checks for this string; AMD CPUs return "AuthenticAMD" which macOS rejects
cpuid.0.eax = "0000:0000:0000:0000:0000:0000:0000:1011"
cpuid.0.ebx = "0111:0101:0110:1110:0110:0101:0100:0111"
cpuid.0.ecx = "0110:1100:0110:0101:0111:0100:0110:1110"
cpuid.0.edx = "0100:1001:0110:0101:0110:1110:0110:1001"

# Prevents AMD-specific MCE behavior from triggering spurious kernel panics
mce.enable = "FALSE"

# Disables side-channel mitigations that conflict with macOS CPU assumptions on AMD
ulm.disableMitigations = "TRUE"
```

For Intel you can use these CPUIDs:

```ini
# Spoofs Intel Core i7 (Ivy Bridge) family/model/stepping and feature flags
cpuid.1.eax = "0000:0000:0000:0001:0000:0110:0111:0001"
cpuid.1.ebx = "0000:0010:0000:0001:0000:1000:0000:0000"
cpuid.1.ecx = "1000:0010:1001:1000:0010:0010:0000:0011"
cpuid.1.edx = "0000:0111:1000:1011:1111:1011:1111:1111"
```

You can verify the file looks right by checking the bottom section in your editor. Here's how the actual working `.vmx` looks, showing `numvcpus`, the CPUID block, and the final lines:

![VMX file in Notepad — numvcpus=4 and CPUID lines visible with search highlighting](images/1775450064997_21_vmx_numvcpus_4.png)

### What each setting does

| Setting | Why it's needed |
|---|---|
| `smc.version = "0"` | Disables VMware's System Management Controller version check. Without this, macOS fails immediately at boot with no useful error. |
| `cpuid.0.*` | Encodes the string `GenuineIntel` in ASCII across the four 32-bit vendor registers. macOS checks for this exact string at boot and refuses to continue on AMD without it. |
| `cpuid.1.*` | Spoofs the CPU's family, model, stepping, and feature flags to match an Intel Ivy Bridge Core i7. macOS uses these to gate hardware feature usage. |
| `mce.enable = "FALSE"` | AMD's Machine Check Exception behavior differs from Intel's. Without this, macOS can trigger spurious kernel panics on AMD from MCE activity. |
| `ulm.disableMitigations = "TRUE"` | Disables Spectre/Meltdown CPU mitigations in the hypervisor layer. These interfere with macOS's Intel-specific CPU assumptions and cause instability on AMD. |

### Side channel mitigations dialog

After adding `ulm.disableMitigations = "TRUE"`, VMware may show this warning on first boot:

![VMware side channel mitigations warning dialog — click OK and proceed](images/06_side_channel_mitigations_dialog.png)

This is expected and informational. Click **OK**. The mitigations are intentionally disabled because they cause problems for macOS on AMD. For a local dev/test VM, this is an acceptable tradeoff.

### If your .vmx edits keep disappearing

VMware can overwrite your `.vmx` when you open VM Settings. To make edits permanent, add them through the UI instead:

**VM → Settings → Options → Advanced → Configuration Parameters**

Add each key-value pair there — these survive Settings saves.

---

## Installing macOS

### Step 1: Boot the VM

Power on the VM. You'll see the VMware boot screen, then the Apple logo with a progress bar.

![Apple logo with progress bar — Monterey recovery booting for the first time](images/13_apple_logo_booting.png)

> ⏱️ The first boot from the recovery VMDK takes 3–5 minutes to reach the installer. Don't assume it's frozen unless nothing has changed for more than 10 minutes.

### Step 2: The macOS Monterey installer loads

After the progress bar completes, the macOS installer will appear — first loading installation information, then showing the familiar Monterey logo.

![macOS Monterey installer loading — "Loading Installation Information..."](images/preview__4_.webp)

### Step 3: Select language

Choose your language and click the arrow to continue.

![macOS Monterey language selection — English selected](images/preview__5_.webp)

### Step 4: Open Disk Utility and format the install disk — DO THIS FIRST

**Do not click "Install macOS Monterey" yet.** The 80 GB disk you added is unformatted, and macOS cannot install to it without APFS formatting first.

From the menu bar, go to **Utilities → Disk Utility** (or it may appear as one of the options on the main screen).

In Disk Utility, select the **VMware Virtual SATA Hard Drive Media** — the 85+ GB uninitialized disk (not the 2.8 GB macOS Base System, which is the recovery VMDK).

![Disk Utility showing the uninitialized 85.9 GB VMware virtual disk selected](images/08_disk_utility_select_drive.png)

Click **Erase** and configure:
- **Name:** anything you like (this guide used `osx`)
- **Format:** `APFS`
- **Scheme:** `GUID Partition Map`

Click **Erase**. When done, you'll see the APFS container creation log and "Operation successful."

![Disk Utility: APFS erase complete — "Operation successful" with APFS container details](images/09_disk_utility_erase_complete.png)

> ⚠️ **APFS is required.** macOS Monterey will not install to HFS+ and may fail silently if the disk format is wrong. Always use APFS + GUID Partition Map.

Close Disk Utility.

### Step 5: Install macOS

Click **Install macOS Monterey** and then **Continue**. Accept the license agreement.

On the disk selection screen, you'll see your freshly formatted disk alongside the macOS Base System. Select your formatted disk — the 85 GB one, **not** macOS Base System.

![macOS Monterey installer disk selection — "osx" (85.69 GB) selected as the install target](images/preview.webp)

Click **Continue**. The installer copies files and the VM reboots automatically — this is normal.

![macOS installation in progress — "About 29 minutes remaining" during the second boot phase](images/preview__1_.webp)

> ⏱️ Total install time is typically **30–60 minutes** including automatic reboots. The progress bar resets during phase transitions — this is normal and not a sign of failure.

### Step 6: macOS Setup Assistant

After installation, the VM boots into the Setup Assistant — country/region, keyboard, Apple ID, and account creation.

![macOS Monterey Setup Assistant — Select Your Country or Region](images/preview__2_.webp)

You can skip Apple ID sign-in for a dev/test environment. Create a local account, finish setup, and you'll reach the desktop.

### Step 7: You made it

![macOS Monterey 12.7.6 desktop — running on AMD via VMware on Windows 11](images/preview__3_.webp)

**macOS Monterey is running on your AMD Windows machine.** Take a snapshot before doing anything else.

Verify the install by clicking **Apple menu → About This Mac**:

![About This Mac — macOS Monterey Version 12.7.6, 4×3.62 GHz, 8 GB RAM, Startup Disk: osx](images/Screenshot_2026-04-05_214224.png)

---

## Troubleshooting — The Ventura Experiment

This section documents the complete Ventura attempt — what we tried, in what order, and exactly why we stopped. If you're here at 2am Googling "macOS Ventura AMD VMware kernel panic", this is for you.

---

### The Ventura attempt — full timeline

**1. Downloaded Ventura and set up the VM using the same process.**

The Ventura board ID (from the source guide, for reference):
```powershell
py macrecovery.py -b Mac-27AD2F918AE68F61 -m 00000000000000000 download
```

Setup went fine. The VM booted.

**2. Ventura reached the language selection screen.**

The first surprise: Ventura didn't immediately fail. It loaded the installer.

![Ventura language selection — the installer loaded successfully on first boot](images/01_ventura_language_selection.png)

**3. The installation ran and appeared to succeed.**

Ventura installed onto the APFS-formatted disk. The progress bar completed. "About 12 minutes remaining" counted down. The file copy finished.

![Ventura installing — "macOS Ventura will be installed on the disk OSX", About 12 minutes remaining](images/02_ventura_installing.png)

**4. The VM rebooted for phase 2 of installation. Kernel panic.**

After the first automatic reboot, the VM hit a kernel panic immediately — the multilingual dark screen reading "Your computer restarted because of a problem."

![Kernel panic after Ventura installation completes — first reboot fails on AMD](images/03_troubleshoot_kernel_panic.png)

The VM looped: boot → kernel panic → boot → kernel panic. No error code. No useful output.

**5. We tried Disk Utility First Aid.**

Booted back into recovery and ran First Aid on the installed volume — maybe filesystem corruption from the panic?

![Disk Utility with installed Ventura volume selected — about to run First Aid](images/16_disk_utility_first_aid.png)

![First Aid running on the Ventura volume](images/17_first_aid_running.png)

![First Aid complete — Operation successful. Volume was healthy.](images/18_first_aid_complete.png)

First Aid reported no problems. The volume was fine. The kernel was the problem.

**6. We tried the `bless` command to force the boot target.**

From Terminal in recovery:

```bash
bless --mount /Volumes/OSX --setBoot
```

![Recovery Terminal — bless --mount /Volumes/OSX --setBoot being typed](images/19_bless_command_terminal.png)

![bless returned to prompt with no error — Unix success (no output = no error)](images/20_bless_command_success.png)

`bless` succeeded. Rebooted. Kernel panic. The issue was the kernel itself, not the boot target.

**7. We added CPUID spoof values to the `.vmx` and tried again.**

Following AMD VMware guides online, we added the full CPUID block to the `.vmx` file.

![VMX file in Notepad++ with CPUID lines added and highlighted in blue](images/04_vmx_cpuid_lines.png)

**8. It got worse. The prohibited symbol appeared.**

With CPUID spoofing in place, the kernel panic was replaced by the prohibited symbol — a black screen with 🚫 and `support.apple.com/mac/startup`.

![Prohibited symbol — black screen with circle-slash after CPUID spoof was added to Ventura](images/05_troubleshoot_prohibited_symbol.png)

This is a **different** and harder failure. The CPUID spoof passed the CPU vendor string check (`GenuineIntel`), but Ventura's kernel reached a deeper feature validation that AMD cannot satisfy — and failed earlier and harder than before.

**9. VMware began generating additional errors.**

![VMware side channel mitigations warning dialog](images/06_side_channel_mitigations_dialog.png)

![SATA device connection error — "Cannot connect the virtual device sata0:1"](images/07_sata_cdrom_dialog.png)

The SATA error (sata0:1 can't connect) is a secondary symptom — VMware is trying to reconnect the CD-ROM device that was present in the original config. Click **No** and it goes away.

**10. We pivoted to Monterey.**

After exhausting the standard fixes, the diagnosis was clear: Ventura's XNU kernel performs CPU validation that CPUID spoofing cannot satisfy on AMD under VMware 17. This isn't a misconfiguration — it's a compatibility wall. The same CPUID values that work on Monterey don't help Ventura because Ventura validates something deeper.

**Monterey worked on the first try with the same config.**

---

### Other common issues

#### 🚫 Prohibited symbol (not Ventura-related)

If you see the prohibited symbol on Monterey, check:
1. `smc.version = "0"` is in the `.vmx`
2. All CPUID lines are present and have no typos
3. The `.vmx` wasn't overwritten by VMware after you edited it — use **Configuration Parameters** in VM Settings to make edits permanent

#### ⛔ "Cannot connect the virtual device sata0:1"

![SATA device connection error dialog](images/07_sata_cdrom_dialog.png)

Click **No**. This appears when VMware tries to connect the CD-ROM (sata0:1) that's no longer needed after installation. After a successful Monterey install, you can go to VM Settings and remove the CD/DVD device entirely.

#### 🔄 Freezes during installation at "X minutes remaining"

Wait. macOS progress bar estimates are wildly inaccurate and the bar can appear frozen for many minutes during phase transitions. Give it at least 15 minutes before assuming it's actually stuck.

#### 💾 "No bootable device" after installation

Boot from the recovery VMDK, open Terminal in recovery, and run:

```bash
# Check your volume name first
diskutil list

# Then bless it (replace "osx" with your volume name)
bless --mount /Volumes/osx --setBoot
```

---

## Post-Install

### 📸 Take a snapshot immediately

Before installing anything:

**VM → Snapshot → Take Snapshot**

Name it `Fresh Install — Pre-Tools`. This is your safe rollback point.

### 🔧 Install VMware Tools

VMware Tools enables proper display scaling, clipboard sharing between host and guest, and better performance.

1. In VMware, go to **VM → Install VMware Tools**
2. A disk image will mount inside macOS
3. Open it and run the **Install VMware Tools** package
4. Approve the system extension in **System Preferences → Security & Privacy**
5. Restart the VM

After VMware Tools installs, the display will scale to a proper resolution and clipboard sync will work.

### 🖥️ Set a useful resolution

**System Preferences → Displays** — set 1920×1080 or 1440×900 depending on your host monitor.

### 🐍 Python and Tkinter for app testing

macOS Monterey includes Python 3 via Xcode Command Line Tools:

```bash
xcode-select --install
```

For a Tkinter app, you'll also want Homebrew and the proper Python build:

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Python with Tk support
brew install python-tk
```

### 🔐 Login screen

After setup you'll have a proper macOS login screen with your local account:

![macOS Monterey login screen — local account ready](images/Screenshot_2026-04-05_214117.png)

### 💾 Snapshot strategy

- `Fresh Install — Pre-Tools` — right after first boot, before anything
- `VMware Tools Installed` — after tools are working and display scales properly
- `Dev Environment Ready` — after Python, Homebrew, and dependencies are installed
- `Before Testing Session` — before running your app tests each session

---

## Summary

| Step | Key point |
|---|---|
| Use Monterey, not Ventura | Ventura installs, then kernel panics on first reboot on AMD. Not fixable with CPUID alone. |
| QEMU command syntax matters | Use the full PowerShell path: `& "C:\Program Files\qemu\qemu-img.exe" convert -O vmdk -o compat6 BaseSystem.dmg recovery.vmdk` |
| Output is `recovery.vmdk` | Not `BaseSystem.vmdk`. The filename matters when you browse for it in the wizard. |
| `smc.version = "0"` is non-negotiable | Without it, the VM won't boot at all. |
| CPUID spoof is non-negotiable | Without it, macOS sees AMD and refuses to continue. |
| `mce.enable = "FALSE"` matters on AMD | Prevents spurious kernel panics from AMD MCE behavior mismatches. |
| Format the disk FIRST in Disk Utility | Do this before clicking Install. Use APFS + GUID Partition Map. |
| Adding CPUID to Ventura makes it worse | It changes the failure from kernel panic to prohibited symbol. Both are dead ends. |
| Take a snapshot before anything | Snapshots are cheap. Recovery from a bad install is not. |

---

## Resources

- [OpenCorePkg releases](https://github.com/acidanthera/OpenCorePkg/releases) — source of `macrecovery.py`
- [Auto-Unlocker](https://github.com/paolo-projects/auto-unlocker) — VMware macOS patcher
- [QEMU for Windows](https://qemu.weilnetz.de/w64/) — disk image conversion tool
- [Original guide this was adapted from](https://bluebubbles.app/docs/server/advanced/macos-virtualization/running-a-macos-vm/deploying-macos-in-vmware-on-windows-full-guide) — bluebubbles-docs by BlueBubbles
- [AMD OSX community](https://amd-osx.com/) — community knowledge for AMD macOS

---

*Guide written based on a real installation attempt on Windows 11 Pro / AMD Ryzen / VMware Workstation Pro 17.5.2. All screenshots captured live during the process — including every failure. Adapted from the [BlueBubbles deployment guide](https://bluebubbles.app/docs/server/advanced/macos-virtualization/running-a-macos-vm/deploying-macos-in-vmware-on-windows-full-guide) with AMD-specific fixes, a corrected QEMU command, and a documented Ventura troubleshooting sequence added.*
