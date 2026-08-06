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
1. Add third-party repositories and install system-level tools and CLI utilities, then reboot:

       sudo cp brave-browser.repo google-chrome.repo vscode.repo /etc/yum.repos.d/
       rpm-ostree install ansible brave-browser code dbus-tools gh gnome-boxes gnome-tweaks google-chrome-stable libguestfs-tools libvirt-daemon-kvm podman-compose qemu-kvm steam-devices virt-install virt-manager

1. Enable the libvirt socket and install a polkit rule so members of `wheel` can manage libvirt without an auth prompt (the unix socket is already world-rw on Fedora, so polkit is the only gate; no group membership is needed):

       sudo systemctl enable --now libvirtd.socket
       sudo cp libvirt-wheel.rules /etc/polkit-1/rules.d/49-libvirt-wheel.rules

1. Configure newly installed packages and desktop environment settings:
       cd ~/Projects/desktop-configuration/
       ansible-playbook --check -vvv local.yml  # Optional Very Verbose Dry Run
       ansible-playbook local.yml

1. Disable the GNOME Keyring password (redundant with LUKS on a single-user system): open **Passwords and Keys** (installed by the playbook), right-click the **Login** keyring, select **Change Password**, enter the current password, and leave the new password blank.

1. Authenticate the GitHub CLI so cloning and pushing over HTTPS works without a manual token prompt. Choose **GitHub.com**, **HTTPS** as the protocol, and **Login with a web browser**; `gh` registers itself as git's credential helper, so subsequent `git clone https://github.com/...` commands authenticate automatically:

       gh auth login

1. To disable Steam scaling: `Steam` -> `Settings` -> `Interface` -> `Scale text and icons to match monitor settings`.

## LUKS Unlock with TPM2 + PIN

After installing with LUKS encryption, enroll the TPM2 chip so the disk can be unlocked with a PIN instead of a full passphrase. The existing passphrase is kept as a fallback.

### Initial Setup

1. Enroll TPM2 with PIN:

       LUKS_DEVICE=$(sudo blkid --match-token TYPE=crypto_LUKS -o device)
       sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 --tpm2-with-pin=yes "$LUKS_DEVICE"

1. Add `tpm2-device=auto` to the options for the LUKS device in `/etc/crypttab` and regenerate the initramfs to include the crypttab change:

       sudo sed -i 's/discard$/discard,tpm2-device=auto/' /etc/crypttab
       rpm-ostree initramfs-etc --track=/etc/crypttab

1. Reboot. The system should now prompt for the TPM2 PIN instead of the full passphrase.

### Re-Enrolling After BIOS/Secure Boot Changes

BIOS updates, Secure Boot key changes, or shim updates will change PCR 7 values, causing TPM unlock to fail. The system will fall back to the full LUKS passphrase. To re-enroll:

       LUKS_DEVICE=$(sudo blkid --match-token TYPE=crypto_LUKS -o device)
       sudo systemd-cryptenroll --wipe-slot=tpm2 --tpm2-device=auto --tpm2-pcrs=7 --tpm2-with-pin=yes "$LUKS_DEVICE"

## WWAN / Mobile Broadband (ThinkPad)

The ThinkPad T16 Gen 1 has an Intel XMM7560 (Fibocom L860-GL) LTE modem using the `iosm` kernel driver.

1. Verify the modem is detected by ModemManager:

       mmcli -L

1. If the modem is listed but not connected, check its status:

       mmcli -m $(mmcli -L | grep -oP '/Modem/\K\d+')

1. Configure the mobile broadband connection in **GNOME Settings** under **Network**. The `mobile-broadband-provider-info` package allows GNOME to auto-detect the carrier APN from the SIM card.

## Wireguard VPN Setup

       sudo nmcli connection import type wireguard file "$filename"

## Flipper Zero

### USB Access (udev)

Updating the firmware over USB needs udev rules granting the local user access to the serial and DFU interfaces. This covers qFlipper / the Momentum web updater (the Flipper's STM32) and flashing the WiFi dev board (its ESP32-S2). The dev board rule also tells ModemManager to leave the port alone — otherwise it grabs the `ttyACM` device and the web flasher fails with "Failed to open serial port."

1. Write the rules file:

       sudo tee /etc/udev/rules.d/42-flipperzero.rules > /dev/null <<'EOF'
       # Flipper Zero serial port
       SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="5740", ATTRS{manufacturer}=="Flipper Devices Inc.", TAG+="uaccess"

       # Flipper Zero DFU
       SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="df11", ATTRS{manufacturer}=="STMicroelectronics", TAG+="uaccess"

       # Flipper Zero WiFi dev board (ESP32-S2): serial access + keep ModemManager off it
       SUBSYSTEMS=="usb", ATTRS{idVendor}=="303a", ATTRS{idProduct}=="0002", ATTRS{manufacturer}=="Espressif", TAG+="uaccess", ENV{ID_MM_DEVICE_IGNORE}="1"
       EOF

1. Reload the rules, then unplug and replug the device so the rule applies:

       sudo udevadm control --reload-rules && sudo udevadm trigger

### Momentum Firmware

[Momentum](https://momentum-fw.dev/) is the custom firmware used here. The web updater is the simplest install:

1. Close qFlipper — only one tool may access the Flipper at a time.
1. Connect the Flipper over USB and open <https://momentum-fw.dev/update> in a WebSerial browser (Chrome or Edge; Firefox and Safari are unsupported).
1. Follow the prompts and wait for the Flipper to reboot into the new firmware.

Firmware lives in flash while user data is on the SD card, so updating never wipes your files. (qFlipper alternative: copy the release folder into `SD/update`, then run `Update` from the Flipper's file menu.)

### WiFi Dev Board (Marauder)

[Marauder](https://github.com/justcallmekoko/ESP32Marauder) is the most useful firmware for the official ESP32-S2 WiFi dev board (Wi-Fi scanning, sniffing, deauth). Detach the board from the Flipper and flash it standalone through its own USB-C port; the Flipper takes no part in flashing. Reattach it afterward to run the Marauder app.

The case's two plungers are unlabeled but press through to the board's **BOOT** and **RESET** buttons (silkscreened `B0`/`RST` underneath). To tell them apart, tap one while the board is plugged in: **RESET** reboots it (LED blinks, the serial device re-enumerates), while **BOOT** alone does nothing. (Holding BOOT alone for ~10 seconds resets the board's settings to default.)

1. Put the board into bootloader mode: hold **BOOT**, press and release **RESET**, wait ~5 seconds, then release **BOOT**.
1. In Chrome or Edge, open <https://fzeeflasher.com>, click **Connect**, and select the board's serial device.
1. Choose **Flipper Dev Board** as the board, **Marauder** firmware, version **latest**, then click **PROGRAM**.
1. Press **RESET** once when flashing completes.

Launch the **[WiFi] Marauder** app on the Flipper with the board attached to use it.

## Dev Containers with Podman

VSCode runs on the host (the `code` RPM), **not** as a Flatpak — the Dev Containers extension shells out to the container CLI and bind-mounts the workspace, both of which the Flatpak sandbox breaks. The playbook sets `"dev.containers.dockerPath": "podman"` in the VSCode user settings so the extension drives rootless Podman directly. Caveats when using `.devcontainer`:

* **File ownership / UID mapping:** if files created inside the container show as owned by the wrong user, add `--userns=keep-id` to the project's `devcontainer.json`:

       "runArgs": ["--userns=keep-id"]

* **SELinux bind mounts:** Silverblue runs SELinux enforcing. If a custom mount hits "permission denied," append `:Z` to it for per-container relabeling (the workspace mount is handled automatically).

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

## SSH with a FIDO2 Token (YubiKey)

### Installing the Provisioning Tools

Toolbox containers are privileged and bind-mount both `/dev` and `/run/pcscd/pcscd.comm`, so `ykman` can reach the token's FIDO2 (hidraw) and PIV (CCID) interfaces without extra flags:

       toolbox create   # only if no container exists yet
       toolbox enter
       sudo dnf install -y yubikey-manager fido2-tools

Run `ykman` inside the toolbox; run `ssh-keygen` and `ssh-add` on the host.

### Using the OpenSSH Agent Instead of GCR

GNOME's default agent (`gcr-ssh-agent`) supports neither FIDO2 `-sk` keys nor PKCS#11. The playbook masks it, enables `ssh-agent.socket`, and writes `~/.config/environment.d/ssh-agent.conf`; log out and back in, then confirm `$SSH_AUTH_SOCK` no longer contains `gcr`.

Terminal `ssh` prompts for the PIN on the tty. For GUI clients (VSCode) to prompt, the host also needs an askpass helper, which must be layered rather than put in a toolbox: `rpm-ostree install openssh-askpass`.

### Wiping the Token and Setting a FIDO2 PIN

A FIDO2 reset erases **all** resident credentials and the PIN — every site registered for passwordless login must be re-enrolled. The token only accepts a reset within ~5 seconds of being plugged in, and it requires a touch:

       ykman fido reset          # unplug and replug first, then touch when it blinks
       ykman fido access change-pin --new-pin <pin>
       ykman fido info

The PIN may be 4–63 characters. Three wrong attempts in a row force a replug; eight total attempts lock the FIDO2 applet until another reset.

### Generating the Key

Run this on the host. `-O resident` stores the handle on the token so it can be recovered on another machine, and `-O application=ssh:` namespaces it so multiple resident keys can coexist:

       ssh-keygen -t ed25519-sk -O resident -O application=ssh:fedora -C "fedora@$(hostname)"

Add `-O verify-required` if you want the PIN demanded on every use, accepting that it cannot be cached. Older tokens (YubiKey firmware before 5.2.3) lack Ed25519 support — use `-t ecdsa-sk` there.

### Loading a Resident Key on a New Machine

       ssh-keygen -K          # writes the key handle into the current directory
       ssh-add -K             # loads resident keys straight into the agent

### Testing

       ssh-add -L
       ssh -T git@github.com

## SSH with YubiKey PIV (Cached PIN)

This is the path that gives a session-long unlock. Slot `9a` defaults to a `ONCE` PIN policy, meaning the PIN is required once per session with the card; `ssh-agent` holds that session open, so one prompt covers every subsequent connection. `pcscd.socket` is enabled by default on Fedora and `opensc-pkcs11.so` ships in the base image, so no host layering is needed.

1. Reset and configure the PIV applet (in the toolbox). This wipes all PIV keys and certificates and restores the default PIN `123456` and PUK `12345678`, which should then be changed:

       ykman piv reset
       ykman piv access change-pin
       ykman piv access change-puk
       ykman piv access change-management-key --generate --protect

1. Generate a key in slot `9a`. `--touch-policy CACHED` requires a touch at most once per 15 seconds; use `NEVER` for no touch at all:

       ykman piv keys generate --algorithm ECCP384 --pin-policy ONCE --touch-policy CACHED 9a /tmp/9a.pem

1. OpenSC only exposes slots that hold a certificate, so generate a self-signed one:

       ykman piv certificates generate --subject "CN=SSH" 9a /tmp/9a.pem

1. Export the SSH public key for `authorized_keys` and GitHub (run on the host):

       ssh-keygen -D /usr/lib64/opensc-pkcs11.so -e

1. Load it into the agent. The PIN is requested once here and cached until the token is unplugged or the agent restarts:

       ssh-add -s /usr/lib64/opensc-pkcs11.so
       ssh-add -l

To make it automatic, add `PKCS11Provider /usr/lib64/opensc-pkcs11.so` to `~/.ssh/config`.

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
