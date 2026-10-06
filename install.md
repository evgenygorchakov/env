## How I install my system

### System Update

Remove unnecessary packages:

```sh
sudo dnf remove cheese rhythmbox gnome-boxesd orca gnome-contacts gnome-getting-started-docs nautilus-sendto gnome-shell-extension-* libreoffice-* gnome-characters gnome-maps gnome-photos simple-scan virtualbox-guest-additions gedit gnome-boxes gnome-tour gnome-connections mediawriter eog gnome-system-monitor baobab gnome-log gnome-calculator gnome-weather gnome-text-editor gnome-font-viewer gnome-clocks gnome-calendar evince totem snapshot cups-browsed anaconda malcontent-control loupe cockpit-bridge cockpit-system cockpit-storaged cockpit-ws cockpit-ws-selinux gnome-console
```

Run Software Center, enable Flathub and Chrome.

Add RPM Fusion (for codecs and NVIDIA driver):

```sh
sudo dnf install --nogpgcheck http://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm http://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
flatpak remote-delete fedora
```

Update system via Software Center.

Install NVIDIA driver (wait ~5 minutes for the kernel module to build before reboot):

```sh
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda libva-nvidia-driver
```

Install software:

```sh
sudo dnf install --nogpgcheck --repofrompath 'terra,https://repos.fyralabs.com/terra$releasever' terra-release
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf copr enable dusansimic/themes
sudo dnf copr enable hyperreal/better_fonts
sudo dnf install xclip micro fuse-encfs zenity borgbackup openssl ffmpegthumbnailer nss-tools mosquitto ydotool amrnb amrwb faac faad2 flac gstreamer1-plugin-libav gstreamer1-plugins-bad-freeworld gstreamer1-plugins-bad-free gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-ugly gstreamer1-plugins-ugly-free lame libdca libmad libmatroska x264 x265 xvidcore mpv ffmpeg ffmpeg-libs libheif-freeworld libheif-tools libva libva-utils mozilla-openh264 unrar 7zip speech-dispatcher speech-dispatcher-utils google-chrome-stable nodejs podman podman-compose git tig ripgrep xkill bat make difftastic zsh starship eza atuin sqlite morewaita-icon-theme nethogs fuse-sshfs logiops libgda libgda-sqlite playerctl cabextract xorg-x11-font-utils tesseract tesseract-devel tesseract-langpack-rus gdisk zbar zed
sudo rpm -ivh --nodigest --nofiledigest https://downloads.sourceforge.net/project/mscorefonts2/rpms/msttcore-fonts-installer-2.6-1.noarch.rpm
```

Set Flatpak languages:

```sh
flatpak config languages --set "en;ru"
sudo flatpak update
```

Install applications from Flatpak:

```sh
flatpak install flathub de.haeckerfelix.Fragments org.telegram.desktop org.nickvision.tubeconverter org.gnome.Loupe com.mattjakeman.ExtensionManager io.gitlab.adhami3310.Converter net.nokyan.Resources org.gnome.Calculator org.gnome.Logs org.gnome.Weather org.gnome.clocks org.gnome.Calendar org.gnome.Epiphany org.inkscape.Inkscape org.gnome.gitlab.YaLTeR.VideoTrimmer org.gnome.World.Iotas app.devsuite.Ptyxis hu.irl.cameractrls org.gnome.Snapshot org.gnome.Papers org.gimp.GIMP be.alexandervanhee.gradia com.github.PintaProject.Pinta org.gnome.font-viewer
```

Speed-up boot:

```sh
sudo systemctl disable NetworkManager-wait-online.service
```

Disable Software auto-update and notifications.

Disable file system scanning:

```sh
dconf write /org/freedesktop/tracker/miner/files/crawling-interval -2
```

### Keyboard

Switch language by Caps Lock (Caps Lock LED shows the second layout):

```sh
gsettings set org.gnome.desktop.input-sources xkb-options "['grp:caps_toggle', 'grp_led:caps']"
```

### Terminal

Set `Ctrl + C` and `Ctrl + V` for copy/paste in Ptyxis
(interrupt moves to `Ctrl + X`, it is set in `zshrc`):

```sh
flatpak run --command=gsettings app.devsuite.Ptyxis set org.gnome.Ptyxis.Shortcuts copy-clipboard '<ctrl>c'
flatpak run --command=gsettings app.devsuite.Ptyxis set org.gnome.Ptyxis.Shortcuts paste-clipboard '<ctrl>v'
```

Install zsh plugins and config:

```sh
mkdir -p ~/.local/share/history
chmod 700 ~/.local/share/history
git clone https://github.com/zsh-users/zsh-syntax-highlighting ~/.local/lib/zsh/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.local/lib/zsh/zsh-autosuggestions
git clone https://github.com/jimhester/per-directory-history ~/.local/lib/zsh/per-directory-history
curl -o ~/.zshrc https://raw.githubusercontent.com/evgenygorchakov/env/main/.zshrc
chsh -s /bin/zsh
```

Import old bash history to atuin:

```sh
atuin import bash
```

Create `/root/.zshrc`:

```sh
eval "$(starship init zsh)"
```

Install Claude Code:

```sh
curl -fsSL https://claude.ai/install.sh | bash
```

Reboot.

### GNOME

Install [Lilex](https://lilex.myrt.co).

```sh
mkdir -p ~/.local/share/fonts
# Copy variable fonts
fc-cache -f -v
gsettings set org.gnome.desktop.interface monospace-font-name "Lilex 12"
```

Disable GNOME extension version check:

```sh
gsettings set org.gnome.shell disable-extension-version-validation true
```

Install extensions from [`GNOME.md`](./GNOME.md).

Add icon theme:

```sh
gsettings set org.gnome.desktop.interface icon-theme 'MoreWaita'
```
