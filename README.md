# Desktop Configuration

* Current distribution: **Fedora 38 Silverblue**
* Current hardware: **AMD X570 + 5900X + RX580 Desktop**, **NUC (8th Generation)**, **ThinkPad T16 Gen 1**

## Upstream Watchlist

* Migration of Fedora Silverblue to use `bootupd`
  * [Silverblue GitHub Issue](https://github.com/fedora-silverblue/issue-tracker/issues/120)
  * [Fedora Change Page](https://fedoraproject.org/wiki/Changes/FedoraSilverblueBootupd)

## Data to Back Up
* `~/.gnupg/`
* `~/.password-store/`
* `~/.gitconfig`
* `~/.pki/`

## Machine Setup

### Operating System Installation

1. Initialize a thumb drive using the [Fedora Media Writer](https://fedoraproject.org/wiki/How_to_create_and_use_Live_USB#Quickstart:_Using_Fedora_Media_Writer) using an image from [Fedora Silverblue](https://silverblue.fedoraproject.org/).
1. On ThinkPad, enable Microsoft's third-party Secure Boot CA in "BIOS."
1. Boot to the Fedora Silverblue install media.
1. Reclaim disk space. Disk encryption is good; either use [Opal](https://en.wikipedia.org/wiki/Opal_Storage_Specification) (weaker) or [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) (stronger).

### System Configuration

1. Reboot into the newly installed Fedora, enable additional repositories, and set up the first user.
1. Update Fedora using the GNOME Software Center (and reboot).
1. Switch to Flatpak for Firefox, install system-level tools and CLI utilities, and reboot:

       flatpak install flathub org.mozilla.firefox
       rpm-ostree override remove firefox firefox-langpacks
       rpm-ostree install ansible f2fs-tools gnome-boxes gnome-tweak-tool google-chrome-stable libvirt-daemon-config-network ltunify pass powertop python3-psutil steam-devices udftools

1. Configure newly installed packages and desktop environment settings:

       sudo systemctl enable --now virtnetworkd-ro.socket
       sudo systemctl enable --now bootupd.socket
       sudo bootupctl adopt-and-update
       flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
       cd ~/Downloads/
       curl https://raw.githubusercontent.com/davidstrauss/desktop-configuration/main/post_install.yml > post_install.yml
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

1. To disable Steam scaling: `Steam` -> `Settings` -> `Interface` -> `Enlarge text and icons based on monitor size (requires restart)`.

## Workarounds

* Intel laptop CPUs sometimes need "panel self refresh" or c-states altered to fix glitches:

       rpm-ostree kargs --append=i915.enable_psr=0
       rpm-ostree kargs --append=intel_idle.max_cstate=2

* `bootupctl` won't yet adopt on Silverblue. Update shim another way:

       wget https://kojipkgs.fedoraproject.org//packages/shim/15.6/2/x86_64/shim-x64-15.6-2.x86_64.rpm # https://koji.fedoraproject.org/koji/packageinfo?packageID=14502
       sudo rpm-ostree usroverlay
       sudo rpm -i --reinstall shim-*.rpm

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

1. Booting to Windows 10 should now appear as an option from the recovery menus.
1. Use the GUI boot repair tool, or [attempt it from the CLI](https://superuser.com/a/1111656).
1. Review BIOS/firmware settings to restore Fedora Linux as the default.
1. Switch to BLS and add Windows into the Linux boot menu:

       sudo grub2-switch-to-blscfg
       sudo grub2-mkconfig -o /boot/grub2/grub.cfg

## Upgrading

1. _Only if needed:_ Remove RPM Fusion repositories for current Fedora:

       rpm-ostree remove rpmfusion-free-release-$(rpm -E %fedora)-1.noarch

1. Rebase on the next release (and resolve issues with any missing packages):

       rpm-ostree rebase fedora:fedora/$(expr $(rpm -E %fedora) + 1)/x86_64/silverblue

1. _Only if needed:_ Add RPM Fusion repositories for next Fedora:

       rpm-ostree install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(expr $(rpm -E %fedora) + 1).noarch.rpm

1. Reboot.

## Mobile Phone Setup

1. Install [OpenKeychain](https://play.google.com/store/apps/details?id=org.sufficientlysecure.keychain) and [Password Store](https://play.google.com/store/apps/details?id=dev.msfjarvis.aps).
1. Generate a new SSH key.
1. Add the new key to the git server.
1. Clone the repository.
1. Copy the private key stub `EXAMPLE.asc` to the phone. If downloaded from Google Drive, it may need to get copied into another app as text and re-saved using a [one or another](https://play.google.com/store/apps/details?id=com.maskyn.fileeditor) [plain text editors](https://play.google.com/store/apps/details?id=com.rhmsoft.edit.pro). Drive likes to convert the `.asc` file to a PDF on download.
1. OpenKeychain > Import Key from File
1. Choose the `.asc` file.

## Smart Cards

### Tested Hardware

#### In Current Use

* Reader and card: YubiKey 5 and 5C

#### Previously Used and Tested

*Instructions may be out of date for these cards.*

* Reader and card: YubiKey Neo and Neo-N
* Reader and card: YubiKey 4 and 4 Nano
* Card only: [Fidesmo Dual Interface](http://shop.fidesmo.com/product/fidesmo-card-dual-interface)
* Reader only: [JK-A0100 Series Smartcard Keyboard](http://cherryamericas.com/product/jk-a0100eu-smartcard-keyboard/): Use `enable-pinpad-varlen` in `.gnupg/gpg-agent.conf` for secure PIN entry. The specific tested model was JK-A0100EU-2.
* Reader only: Identiv SCM SPR 532: Should work with secure PIN entry out of the box
* Reader only: Lenovo ThinkPad T-series built-in: No secure PIN entry available

#### Resources

* [LWN Article: A comparison of cryptographic keycards](https://lwn.net/Articles/736231/)
* [OpenKeychain Compatibility](https://github.com/open-keychain/open-keychain/wiki/Security-Tokens)

### Using an Existing Smart Card

1. Complete machine setup (above).
2. Import any existing smart card keys (that were set up according to the directions below):

       gpg2 --card-edit
       > fetch
       > quit
       gpg2 --card-status

3. Import any other keys:

       gpg2 --keyserver hkps://keys.openpgp.org --recv-key $KEYID
       gpg2 --keyserver pool.sks-keyservers.net --recv-key $KEYID  # Another database to try.
       gpg2 --keyserver pgp.mit.edu --recv-key $KEYID  # Another database to try.

4. Add trust to any necessary keys:

       gpg2 --edit-key $KEYID
       gpg> trust
       Your decision? 5
       gpg> quit

### Setting Up a New Smart Card

1. Complete machine setup (above).
1. If it's a YubiKey, enable OpenPGP:

   * For YubiKey Neo or YubiKey 4:

         sudo dnf install ykpers
         ykpersonalize -m6

   * YubiKey 5 seems to ship with OpenPGP enabled. To verify and get other information:

         sudo dnf install -y swig gcc pcsc-lite-devel python-devel  # Can be in Fedora Toolbox
         pip install --user yubikey-manager                         # Can be in Fedora Toolbox
         ykman openpgp info  # Must be outside Toolbox to use PC/SC APIs

1. Configure the card, generate a key pair, and upload the key:

       gpg2 --change-pin  # Change both the PIN (default is 123456)
                          # and the Admin PIN (default is 12345678).
                          # I use pwgen for the admin PIN.
       gpg2 --card-edit
       gpg/card> admin
       gpg/card> key-attr # On YubiKey firmware 5.2.3+, choose ECC and Curve 25519 for each option. Older keys and firmwares don't support this.
       gpg/card> generate # Perform the off-card backup (which is only a shim private key, anyway).
                          # Use a key size of 3072 for RSA on YubiKey 4. Defaults are fine for NEO and YubiKey 5.
                          # No expiration.
       gpg/card> quit  # GPG will then print out data, including the key fingerprint
                       # as a long, alphanumeric string.
       gpg2 --keyserver hkps://keys.openpgp.org --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pool.sks-keyservers.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pgp.mit.edu --send-keys $FINGERPRINT

    **Note:** Revocation certificates are backed up to `~/.gnupg/openpgp-revocs.d`

1. Display the public key in OpenSSH format:

       ssh-add -L

1. Optionally, export the "secret key," which will only be a stub (not the actual key, which is not obtainable). This is importable into OpenKeychain on Android.

       gpg2 --export-secret-key --armor $FINGERPRINT > $FINGERPRINT.asc  # Back this up, too.

1. After this is finished, the card should work. You should also have `$FINGERPRINT.asc` and `$FINGERPRINT.rev` backed up. Google Drive and Dropbox are usually fine for this backup; these files cannot be used to impersonate or decrypt, only revoke.

1. To add to Password Store:
   1. Connect one existing key that can already access passwords.
   1. Verify that the existing key is unlocked by accessing a password.
   1. Instruct Password Store to reencrypt to the desired keys:

          pass init $FINGERPRINT [...additional fingerprints...]  # Ensure existing key is connected and unlocked first.

### Testing and Troubleshooting the Setup

When there's an issue, we can narrow the problem down to an individual component or connection.

* Test that the GPG agent is running and accessible (after an attempt at use):

       systemctl --user status gpg-agent.service
       ls -l $SSH_AUTH_SOCK

       # Optionally, stop it (which will cause reinitialization on use):
       systemctl --user stop gpg-agent.service

* Test that the standard PIN counter hasn't been exhausted. It's the first number returned here:

       gpg2 --card-status|grep "PIN retry counter"

  * If the standard PIN counter has been exhausted, it's possible to unblock (using `gpg2 --card-edit` with `passwd`) as long as the third number (the mangement/admin PIN retry counter) wasn't also zero.

  * If even the management/admin PIN is exhausted, then the entire GPG module needs to be reset: [YubiKey instructions](https://www.yubico.com/support/knowledge-base/categories/articles/reset-applet-yubikey/)

* Test the GPG-to-smart card connection and key trust. The following should prompt for the regular PIN and succeed:

       FINGERPRINT=`gpg2 --card-status | grep "General key info" | grep -o "/[[:alnum:]]* " | grep -o "[[:alnum:]]*"`
       echo "test" | gpg2 --sign --armor --local-user $FINGERPRINT

* Test the OpenSSH client's connection to the GPG agent. The following should output the SSH public key:

       ssh-add -L

### One-Time or Test Usage of the Agent

1. Open a terminal.
2. If you're restarted your computer since using the agent, start it:

       gpg-agent --daemon --enable-ssh-support

3. In any shell where you want to use it, point OpenSSH to the GPG agent:

       export SSH_AUTH_SOCK=${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh  # Or use the line shown in the output of starting the GPG agent

### Revoking a Key

1. If the key isn't imported locally, follow the "Using an Existing Smart Card" steps first (but skipping the "trust" step).
2. If you don't have the revocation certificate (`.rev`) backed up but have the private key:

       gpg2 --gen-revoke --output=$FINGERPRINT.rev $FINGERPRINT

3. Import the revocation:

       gpg2 --import $FINGERPRINT.rev  # May need to remove colon before the five dashes from file.

4. Publish the revocation:

       gpg2 --keyserver hkps://keys.openpgp.org --send-keys $FINGERPRINT
       gpg2 --keyserver hkps://hkps.pool.sks-keyservers.net --send-keys $FINGERPRINT
       gpg2 --keyserver hkp://pgp.mit.edu --send-keys $FINGERPRINT

## Go Development

Configure `go get` to use SSH-based authentication:

    git config --global url.git@github.com:.insteadOf https://github.com/

## PHP Development

**This section is not yet updated for Fedora 30 Silverblue.**

1. Install packages:

       sudo dnf install -y nginx mariadb mariadb-server php php-fpm php-mysqlnd php-dbg php-cli php-bcmath php-phpass php-mbstring php-opcache php-gd php-pecl-apcu php-pecl-xdebug

2. Install, start, and configure the database:

       sudo systemctl start mariadb
       mysql_secure_installation

3. Create a directory for web projects (and enable web server access to directories of that type):

       chmod 711 ~
       mkdir ~/public_html
       sudo setsebool -P httpd_enable_homedirs 1    # Enables use of ~/public_html by nginx and PHP-FPM.
       sudo setsebool -P httpd_execmem 1            # Enables PHP's regex compilation.
       sudo setsebool -P httpd_builtin_scripting 1  # Hope to fix: https://bugzilla.redhat.com/show_bug.cgi?id=1510717
       sudo setsebool -P httpd_unified 1            # Same here.
       sudo setsebool -P httpd_enable_cgi 1         # Same here.

4. Add support for `~/public_html` to nginx using `/etc/nginx/default.d/userdir.conf`:

    ```conf
    location @drupal {
        error_log /var/log/nginx/userdir.log notice;
        #rewrite_log on;
        rewrite ^/~([^/]+)/([^/]+)(.*)\?(.*)$ /~$1/$2/index.php?q=$3&$4;
        rewrite ^/~([^/]+)/([^/]+)(/.*?)$ /~$1/$2/index.php?q=$3;
    }

    location ^~ /~ {
        error_log /var/log/nginx/userdir.log notice;

        location ~ ^/~(?<username>[^/]+)(?<path>/.+\.php)$ {
            root /home/$username/public_html;
            try_files $path =404;
            fastcgi_index  index.php;
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  $document_root$path;
            fastcgi_param  SCRIPT_NAME      /~$username$path;
            fastcgi_pass   php-fpm;
        }

        location ~ ^/~(?<username>[^/]+)(?<path>/.+?)?$ {
            root /home/$username/public_html;
            try_files $path @drupal;
        }
    }
    ```

5. Configure some PHP-related options:

       echo "apc.rfc1867=1" | sudo tee -a /etc/php.d/40-apcu.ini  # Upload progress tracking.

6. Start services for development (each time they're needed):

       sudo systemctl start mariadb php-fpm nginx

7. Use `~/public_html/$PROJECT/` as the web root, accessible via `http://localhost/~$USER/$PROJECT/`.

8. If new files with the wrong context get added, fix the selinux context:

       restorecon -R ~/public_html

## Dwarf Fortress

**This section is not yet updated for Fedora 30 Silverblue.**

1. Download the latest archive.
2. Install necessary dependencies:

       sudo dnf install -y SDL.i686 SDL_image.i686 SDL_ttf.i686 mesa-libGLU.i686 gtk2.i686 zlib.i686 openal-soft.i686 xterm python qt qt-x11 bzip2 xorg-x11-fonts-Type1

3. Launch with `startlnp`
4. Use `xterm -e` as the custom terminal command configuration.

## Stable Diffusion

### Setup

    toolbox create stable-diffusion
    toolbox enter stable-diffusion
    sudo dnf upgrade -y
    sudo dnf install -y conda python3-opencv
    git clone git@github.com:CompVis/stable-diffusion.git
    cd stable-diffusion
    conda env create -f environment.yaml
    conda activate ldm
    mkdir -p models/ldm/stable-diffusion-v1/
    curl https://huggingface.co/CompVis/stable-diffusion-v-1-4-original/blob/main/sd-v1-4.ckpt > models/ldm/stable-diffusion-v1/model.ckpt

### Operation

    cd ~/stable-diffusion
    python3 scripts/txt2img.py --prompt "a photograph of an astronaut riding a horse" --plms

## BIOS Updates

First, acquire the update. For ThinkPads, use [Lenovo's My Products tool](https://account.lenovo.com/us/en/myproducts), click on the product model (after adding yours), then Top Downloads > View All, and finally the "Bootable CD" ISO.

### USB Thumb Drive Method

1. Install the conversion utility into a Toolbox:

       toolbox enter
       sudo dnf install -y geteltorito

2. Write it to a raw drive image:

       geteltorito -o update.img downloaded.iso

3. Open the `update.img` in the Disks utility and restore it to the USB drive.
4. Boot from that drive.

### GRUB2 EFI Chainloader Method

1. **First Time:** Setup:

       rpm-ostree install syslinux p7zip p7zip-plugins  # And reboot.

       ESPUUID=`sudo grub2-probe --target=fs_uuid /boot/efi/`
       cat >> 40_custom <<EOF
       menuentry "Firmware Update" {
           insmod fat
           insmod part_gpt
           insmod chain
           insmod search_fs_uuid
           search --fs-uuid $ESPUUID
           chainloader /LenovoEFI/Boot/BootX64.efi
       }
       EOF

       sudo mv 40_custom /etc/grub.d/40_custom
       sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg

2. **Every Time:** Extract the update to appropriate places in the EFI partition:

       geteltorito -o eltorito.img downloaded.iso
       7z x -oeltorito/ eltorito.img
       sudo rsync --recursive --delete --verbose eltorito/Flash/ /boot/efi/Flash/
       sudo rsync --recursive --delete --verbose eltorito/EFI/ /boot/efi/LenovoEFI/

3. **Every Time:** Reboot and select "Firmware Update."

## Crash Reporting

* Reconfigure using `system-config-abrt`

## Other Resources

* [U2F Local Login](http://blog.liw.fi/posts/u2f-pam/)
