# Desktop Configuration

Current version: Fedora 25

## Data to Back Up
* `~/.gnupg/`
* `~/.password-store/`
* `~/.gitconfig`

## Machine Setup
1. Initialize a thumb drive using the [Fedora Media Writer](https://fedoraproject.org/wiki/How_to_create_and_use_Live_USB#Quickstart:_Using_Fedora_Media_Writer).
1. Install, reclaiming space as necessary.
1. Update Fedora using the GNOME Software Center.
1. Configure the GNOME desktop:

        gsettings set org.gnome.desktop.interface scaling-factor 1
        gsettings set org.gnome.desktop.input-sources xkb-options "['caps:ctrl_modifier']"
        gsettings set org.gnome.desktop.wm.preferences num-workspaces 1
        gsettings set org.gnome.desktop.interface clock-show-seconds true
        gsettings set org.gnome.desktop.interface clock-show-date true
        gsettings set org.gnome.desktop.peripherals.mouse natural-scroll true
        gsettings set org.gnome.shell enabled-extensions "[]"
        gsettings set "org.gnome.Terminal.Legacy.Keybindings:/org/gnome/terminal/legacy/keybindings/" reset-and-clear 'F12'
        TPROFILE=$(gsettings get org.gnome.Terminal.ProfilesList default | tr --delete "'")
        gsettings set "org.gnome.Terminal.Legacy.Profile:/org/gnome/terminal/legacy/profiles:/:$TPROFILE/" scrollback-unlimited true

1. Install Google Chrome and authorize sandboxing:

        sudo dnf install https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
        sudo setsebool -P unconfined_chrome_sandbox_transition 0

1. Add RPM Fusion repositories:

       sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

1. Install OpenRA:

       sudo dnf config-manager --add-repo http://download.opensuse.org/repositories/games:/openra/Fedora_24/games:openra.repo
       sudo dnf install -y openra

1. Install other packages:

        sudo dnf install gimp htop inkscape iotop mariadb meld nano php-cli powertop quassel-client tor unbound wireshark-gnome transmission gnome-system-log fatsort nmap-frontend golang pass ghex composer gnome-builder u2f-hidraw-policy chromium libvirt-daemon-config-network libvirt-daemon-driver-network

1. Add smart card support.
1. Add Go paths to ~/.bashrc:

        # User specific aliases and functions
        export GOPATH=$HOME/go
        export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

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
