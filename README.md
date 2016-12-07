# Desktop Configuration

Current distribution: **Fedora 25**

## Data to Back Up
* `~/.gnupg/`
* `~/.password-store/`
* `~/.gitconfig`

## Machine Setup
1. Initialize a thumb drive using the [Fedora Media Writer](https://fedoraproject.org/wiki/How_to_create_and_use_Live_USB#Quickstart:_Using_Fedora_Media_Writer).
1. Boot to the USB drive.
1. Reclaim disk space. Disk encryption is good; I use [Opal](https://en.wikipedia.org/wiki/Opal_Storage_Specification) from my ThinkPad BIOS setup, but you can use [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup). I prefer Opal because GNOME Software Center updates require two reboots, and Opal can persist across reboots. 
1. Set up a single admin user (no password set for `root`).
1. Reboot into the newly installed Fedora.
1. Update Fedora using the GNOME Software Center. (A direct `dnf upgrade` can, rarely, cause issues.)
1. Configure the GNOME desktop:

        gsettings set org.gnome.desktop.interface scaling-factor 1  # Disables HiDPI scaling.
        gsettings set org.gnome.desktop.input-sources xkb-options "['caps:ctrl_modifier']"  # Use Caps Lock as Ctrl
        gsettings set org.gnome.desktop.wm.preferences num-workspaces 1  # Disable all workspace functionality.
        gsettings set org.gnome.desktop.interface clock-show-seconds true
        gsettings set org.gnome.desktop.interface clock-show-date true
        gsettings set org.gnome.desktop.peripherals.mouse natural-scroll true  # Use Apple-style natural scrolling.
        gsettings set org.gnome.shell enabled-extensions "[]"  # Disable the Fedora desktop logo.
        
        # Terminal:
        gsettings set "org.gnome.Terminal.Legacy.Keybindings:/org/gnome/terminal/legacy/keybindings/" reset-and-clear 'F12'  # Set F12 to Reset and Clear
        TPROFILE=$(gsettings get org.gnome.Terminal.ProfilesList default | tr --delete "'")
        gsettings set "org.gnome.Terminal.Legacy.Profile:/org/gnome/terminal/legacy/profiles:/:$TPROFILE/" scrollback-unlimited true  # Enable unlimited scrollback.

1. Install Google Chrome and authorize sandboxing:

        sudo dnf install https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
        sudo setsebool -P unconfined_chrome_sandbox_transition 0

1. Add RPM Fusion repositories:

        sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

1. Install other packages:

        sudo dnf install gimp htop inkscape iotop mariadb meld nano php-cli powertop quassel-client tor unbound wireshark-gnome transmission gnome-system-log fatsort nmap-frontend pass ghex composer gnome-builder u2f-hidraw-policy chromium libvirt-daemon-config-network libvirt-daemon-driver-network

1. Using Firefox, add the [Caffeine GNOME Shell extension](https://extensions.gnome.org/extension/517/caffeine/).

## Smart Cards

### Tested Hardware

* Combined reader and card:
 * [YubiKey Neo](https://www.yubico.com/products/yubikey-hardware/yubikey-neo/)
 * [YubiKey 4](https://www.yubico.com/products/yubikey-hardware/yubikey4/) and 4 Nano
* Card only:
 * [Sigilance Contact and NFC](https://www.sigilance.com/)
 * [Sigilance Contact-Only](https://www.sigilance.com/store/contact-cards/)
* Reader only:
 * [JK-A0100 Series Smartcard Keyboard](http://cherryamericas.com/product/jk-a0100eu-smartcard-keyboard/): Use `enable-pinpad-varlen` in `.gnupg/gpg-agent.conf` for secure PIN entry. The specific tested model was JK-A0100EU-2.
 * Identiv SCM SPR 532: Should work with secure PIN entry out of the box
 * Lenovo ThinkPad T560 built-in: No secure PIN entry available

### Machine Setup

1. Install packages:

        sudo dnf install ykpers pcsc-lite-ccid

1. Start and enable the smart card daemon:

        sudo systemctl enable --now pcscd

1. Enable GnuPG SSH agent support:

        echo "enable-ssh-support" >> .gnupg/gpg-agent.conf
        
1. Ensure the GnuPG Agent is available for SSH use whenever you start a Bash session:

        cat <<EOT >> ~/.bashrc
        GPG_TTY=$(tty); export GPG_TTY;
        if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
          export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
        fi
        gpg-connect-agent /bye
        EOT

### Using an Existing Smart Card

1. Complete the Machine Setup above.
1. Import any existing smart card keys:

        gpg2 --card-edit
        > fetch
        > quit
        gpg2 --card-status

1. Import any other keys:

        gpg2 --keyserver hkps.pool.sks-keyservers.net --recv-key $KEYID
        gpg2 --keyserver pgp.mit.edu --recv-key $KEYID  # A different database to try.

### Setting Up a New Smart Card

1. Complete the Machine Setup above.
1. If the smart card is a YubiKey Neo or YubiKey 4, set the card's mode:

        ykpersonalize -m86

1. Configure the card, generate a key pair, and upload the key:

        gpg2 --change-pin  # Change both the PIN (default is 123456) and the Admin PIN (default is 12345678). I use pwgen for the admin PIN.
        gpg2 --card-edit
        gpg/card> admin
        gpg/card> generate  # No off-card backup. No expiration. Back up the .rev file.
        gpg/card> quit  # GPG will then print out data, including the key fingerpring as a long, alphanumeric string.
        gpg2 --keyserver hkps://hkps.pool.sks-keyservers.net --send-keys $FINGERPRINT
        gpg2 --keyserver hkp://pgp.mit.edu --send-keys $FINGERPRINT

1. Display the public key in OpenSSH format:

        ssh-add -L

1. Optionally, export the "secret key," which will only be a stub (not the actual key, which is not obtainable):

        gpg2 --export-secret-key --armor $FINGERPRINT > $FINGERPRINT.asc  # Back this up, too.

1. After this is finished, the card should work. You should also have `$FINGERPRINT.asc` and `$FINGERPRINT.rev` backed up. Google Drive and Dropbox are fine for this backup; these files cannot be used to impersonate you.

### One-Time or Test Usage of the Agent

1. Open a terminal.
1. If you're restarted your computer since using the agent, start it:

        gpg-agent --daemon

1. In any shell where you want to use it, point OpenSSH to the GPG agent:

        export SSH_AUTH_SOCK=/run/user/$UID/gnupg/S.gpg-agent.ssh  # Or use the line shown in the output of starting the GPG agent

## Go Development

1. Install packages:

        sudo dnf install golang

1. Add Go to the path:

        cat <<EOT >> ~/.bashrc
        export GOPATH=$HOME/go
        export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
        EOT

## PHP Development

1. Install packages:

        sudo dnf install -y httpd mariadb mariadb-server php php-mysqlnd php-dbg php-cli php-bcmath php-phpass php-mbstring php-opcache php-gd php-pecl-apcu php-pecl-xdebug

1. Start and configure services:

        sudo systemctl start mariadb
        sudo mysql_secure_installation
        echo "apc.rfc1867=1" | sudo tee -a /etc/php.d/40-apcu.ini
        sudo systemctl start httpd

## Dwarf Fortress
1. Download the latest archive.
1. Install necessary packages:

        sudo dnf install -y SDL.i686 SDL_image.i686 SDL_ttf.i686 mesa-libGLU.i686 gtk2.i686 zlib.i686 openal-soft.i686 xterm python qt qt-x11 bzip2 xorg-x11-fonts-Type1

1. Launch with `startlnp`
1. Use `xterm -e` as the custom terminal command configuration.

## OpenRA

1. Add OpenRA repository:

        sudo dnf config-manager --add-repo http://download.opensuse.org/repositories/games:/openra/Fedora_24/games:openra.repo

1. Install package:

        sudo dnf install -y openra
        
## Other Resources

* [U2F Local Login](http://blog.liw.fi/posts/u2f-pam/)

