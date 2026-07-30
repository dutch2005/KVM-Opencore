# KVM-Opencore

This is a fork of [Leoyzen's OpenCore image](https://github.com/leoyzen/KVM-Opencore) for QEMU/KVM, 
which I've extended to add a build system for automatically buliding all of the required files from 
sourcecode, and to keep up with the latest OpenCore changes.

It is currently tested to boot macOS Catalina, Big Sur, Monterey, and up to Sequoia/Tahoe (with specific AMD patches), but will likely also boot older 
versions of macOS.

Although the images offered here should work on all QEMU/KVM distributions, I specifically build
and test these for my Proxmox Hackintosh guide here:

https://www.nicksherlock.com/2021/10/installing-macos-12-monterey-on-proxmox-7/

Note that although the images in the Releases have filenames like OpenCore-v15.iso, these aren't 
real ISOs, but rather raw hard disk images, and need to be booted as such. (They're just using .iso
extensions so they'll appear in Proxmox's disk image picker).

## 🍏 macOS Sequoia & Tahoe (macOS 15) on AMD Proxmox
When attempting to upgrade to macOS Tahoe using older OpenCore builds (0.9.x) and the standard `AMD_Vanilla` patches on an AMD Proxmox host, the VM will permanently freeze at `#[EB|LOG:EXITBS:START]`. This is because Apple refactored `_cpuid_set_generic_info` in the Tahoe kernel, causing standard `wrmsr`/`rdmsr` spoofing patches to fail.

To boot the Tahoe kernel on AMD, you must spoof the hypervisor presence (`hv_vmm_present`) instead of modifying CPU registers.

### Step 1: Update your config.plist
Using [ProperTree](https://github.com/corpnewt/ProperTree) or any plist editor, mount your EFI partition and edit `EFI/OC/config.plist` with the following changes:

1. **Boot Arguments:** Navigate to `NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82 -> boot-args` and append `-lilubetaall` to force Lilu to load on the unreleased OS.
2. **SMBIOS:** Navigate to `PlatformInfo -> Generic -> SystemProductName` and change it to `iMac20,1`. (Using MacPro models like `MacPro7,1` triggers memory panics during Tahoe boot on Proxmox).
3. **Kernel Patches:** Navigate to `Kernel -> Patch` and inject the following two patches to replace hibernation signatures with hypervisor spoofing signatures:

**Patch 1:**
* **Identifier:** `kernel`, **Count:** `1`, **Limit/Skip:** `0`, **Arch:** `x86_64`
* **Find:** `68696265726E61746568696472656164790068696265726E617465636F756E7400`
* **Replace:** `68696265726E61746568696472656164790068765F766D6D5F70726573656E7400`

**Patch 2:**
* **Identifier:** `kernel`, **Count:** `1`, **Limit/Skip:** `0`, **Arch:** `x86_64`
* **Find:** `626F6F742073657373696F6E20555549440068765F766D6D5F70726573656E7400`
* **Replace:** `626F6F742073657373696F6E20555549440068696265726E617465636F756E7400`

### Step 2: Update OpenCore & Reset NVRAM
1. Ensure your OpenCore engine is updated to `1.0.5` or newer. If you are downloading a pre-built `.iso` from this repository, stop your VM and flash it to your boot disk (e.g., `dd if=OpenCore.iso of=/dev/zvol/rpool/data/vm-XXXX-disk-X`).
2. Boot your Proxmox VM. 
3. At the OpenCore boot picker, press the **Spacebar** to reveal hidden options.
4. Select **Reset NVRAM** and press Enter. This ensures your new SMBIOS and `boot-args` are properly loaded.

### Step 3: Trigger the macOS Upgrade
Once booted into your existing macOS installation, you can upgrade to Tahoe using either method:
* **UI Method:** Open `System Settings -> General -> Software Update` and click "Upgrade Now".
* **Terminal Method:** Open Terminal and run `sudo softwareupdate -i -a -R` to automatically download, stage, and reboot into the installer.

*Note: The VM will reboot 2 to 3 times during the installation. Ensure OpenCore selects the `macOS Installer` partition during these reboots.*

### Step 4: Bypassing the Tahoe Setup Assistant via SSH (Optional)
Tahoe may lock you out of the desktop with an aggressive "Setup Assistant" asking for Apple ID, iCloud Analytics, and FileVault configurations after the upgrade. To bypass this remotely, SSH into the Mac and run:

```bash
defaults write com.apple.SetupAssistant DidSeeCloudSetup -bool TRUE && \
defaults write com.apple.SetupAssistant DidSeePrivacy -bool TRUE && \
defaults write com.apple.SetupAssistant DidSeeSiriSetup -bool TRUE && \
defaults write com.apple.SetupAssistant DidSeeSyncSetup2 -bool TRUE && \
defaults write com.apple.SetupAssistant DidSeeTouchIDSetup -bool TRUE && \
defaults write com.apple.SetupAssistant DidSeeFileVaultSetup -bool TRUE && \
killall "Setup Assistant"
```
