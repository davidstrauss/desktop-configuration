# Desktop Configuration

* Current distribution: **Fedora 43 Silverblue**
* Current hardware: **AMD X570 + 5900X + RX580 Desktop**, **ThinkPad T16 Gen 1 (Intel)**

## Data to Back Up
* `~/.gitconfig`
* `~/.var/app/com.valvesoftware.Steam/.local/share/Steam/steamapps/common/`
* `~/Documents/`
* `~/Pictures/`
* `~/Projects/`
* `/etc/NetworkManager/system-connections/`

## Machine Setup

### Operating System Installation

1. Initialize a thumb drive using the [Fedora Media Writer](https://fedoraproject.org/wiki/How_to_create_and_use_Live_USB#Quickstart:_Using_Fedora_Media_Writer) using an image from [Fedora Silverblue](https://silverblue.fedoraproject.org/).
1. On ThinkPad, enable Microsoft's third-party Secure Boot CA in "BIOS."
1. Boot to the Fedora Silverblue install media.
1. Reclaim disk space. Disk encryption is good; either use [Opal](https://en.wikipedia.org/wiki/Opal_Storage_Specification) (weaker) or [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) (stronger).

### System Configuration

1. Reboot into the newly installed Fedora, enable additional repositories, and set up the first user.
1. Update Fedora using the GNOME Software Center (and reboot).
1. Install system-level tools and CLI utilities, and reboot:

       rpm-ostree install ansible code gnome-boxes gnome-tweaks steam-devices

1. Configure newly installed packages and desktop environment settings:

       sudo cp vscode.repo /etc/yum.repos.d/
       cd ~/Projects/desktop-configuration/
       ansible-playbook --check -vvv post_install.yml  # Optional Very Verbose Dry Run
       ansible-playbook post_install.yml  # Many dconf configs seem to fail unless already correctly set.

1. Configure git (if not restoring `~/.gitconfig`):

       git config --global user.name "David Strauss"
       git config --global user.email name@example.com
       git config --global init.defaultBranch main
       git config --global color.ui auto

1. Set battery charging thresholds (on laptop):

       echo 10 | sudo tee /sys/class/power_supply/BAT0/charge_start_threshold
       echo 90 | sudo tee /sys/class/power_supply/BAT0/charge_stop_threshold
       #Configuring thresholds for the second battery doesn't seem to work yet.
       #echo 10 | sudo tee /sys/class/power_supply/BAT1/charge_start_threshold
       #echo 90 | sudo tee /sys/class/power_supply/BAT1/charge_stop_threshold

1. To disable Steam scaling: `Steam` -> `Settings` -> `Interface` -> `Scale text and icons to match monitor settings`.

## LUKS Unlock with TPM2 + PIN

After installing with LUKS encryption, enroll the TPM2 chip so the disk can be unlocked with a PIN instead of a full passphrase. The existing passphrase is kept as a fallback.

### Initial Setup

1. Enroll TPM2 with PIN (PCR 7 covers Secure Boot state):

       sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 --tpm2-with-pin=yes $(blkid --match-tag TYPE=crypto_LUKS -o device)

1. Add `tpm2-device=auto` to the options for the LUKS device in `/etc/crypttab`.

1. Regenerate the initramfs to include the crypttab change:

       rpm-ostree initramfs-etc --track=/etc/crypttab

1. Reboot. The system should now prompt for the TPM2 PIN instead of the full passphrase.

### Re-Enrolling After BIOS/Secure Boot Changes

BIOS updates, Secure Boot key changes, or shim updates will change PCR 7 values, causing TPM unlock to fail. The system will fall back to the full LUKS passphrase. To re-enroll:

       sudo systemd-cryptenroll --wipe-slot=tpm2 --tpm2-device=auto --tpm2-pcrs=7 --tpm2-with-pin=yes $(blkid --match-tag TYPE=crypto_LUKS -o device)

## Wireguard VPN Setup

       sudo nmcli connection import type wireguard file "$filename"

## Workarounds

* Intel laptop CPUs sometimes need "panel self refresh" or c-states altered to fix glitches:

       rpm-ostree kargs --append=i915.enable_psr=0
       rpm-ostree kargs --append=intel_idle.max_cstate=2

* Missing Flatpak icons (untested fix):

       sudo gtk-update-icon-cache -f /var/lib/flatpak/exports/share/icons/hicolor/
       sudo gtk4-update-icon-cache -f /var/lib/flatpak/exports/share/icons/hicolor/

## Coexistence with Windows

After a complete wipe of the EFI partition, Windows won't have its required resources to boot.

1. Boot from Windows install media (F8 for the boot menu on Asus boards and F12 on ThinkPad).
1. Use `diskpart` to assign a drive letter (like `G`) to the EFI partition (which should be labeled `System`).
1. Restore boot files:

       G:\EFI
       bootrec /rebuildbcd

1. Booting to Windows should now appear as an option from the recovery menus.
1. Use the GUI boot repair tool, or [attempt it from the CLI](https://superuser.com/a/1111656).
1. Review BIOS/firmware settings to restore Fedora Linux as the default.

## Upgrading

1. _Only if needed:_ Remove RPM Fusion repositories for current Fedora:

       rpm-ostree remove rpmfusion-free-release-$(rpm -E %fedora)-1.noarch

1. Rebase on the next release (and resolve issues with any missing packages):

       rpm-ostree rebase fedora:fedora/$(expr $(rpm -E %fedora) + 1)/x86_64/silverblue

1. _Only if needed:_ Add RPM Fusion repositories for next Fedora:

       rpm-ostree install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(expr $(rpm -E %fedora) + 1).noarch.rpm

1. Reboot.

## SSH with FIDO2

### Generating an SSH Key

       ssh-keygen -t ed25519-sk -O resident -O application=ssh:

### Loading a Resident Key on a New Machine

       ssh-keygen -K

### Testing

       ssh-add -L

## OpenMW

1. Install the Flatpak:

       flatpak install flathub org.openmw.OpenMW

1. Download the "backup" file from GOG.
1. Extract the backup:

       mkdir morrowind
       mv setup_tes_morrowind_goty_2.0.0.7.exe morrowind/
       cd morrowind
       innoextract setup_tes_morrowind_goty_2.0.0.7.exe
       mv app/Data\ Files/* ~/.var/app/org.openmw.OpenMW/data/openmw/data/
