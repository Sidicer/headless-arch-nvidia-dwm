# Arch Linux Headless Streaming Server (Nvidia)

Configuration to run a fully headless server for **Game** or **Video streaming** which has a remote (virtual) desktop without using a physical dummy connector for GPU or xf86-video-dummy (which doesn't use GPU to render).  
DWM used due to how light-weight it is.

Things that I run personally are:
* [qbittorrent-nox](https://archlinux.org/packages/extra/x86_64/qbittorrent-nox/) (w/ WebUI)
* [jellyfin-server](https://archlinux.org/packages/?name=jellyfin-server) (video streaming)
* [sunshine](https://aur.archlinux.org/packages/sunshine) (game streaming, remote desktop)
* [steam](https://archlinux.org/packages/multilib/x86_64/steam/) (big picture)

# Quick-start
```sh
# If yay & nvidia drivers setup already
yay -S --sudoloop --noconfirm --needed \
  dwm zsh xorg-server xorg-xinit sunshine nvidia-settings

git clone git@github.com:Sidicer/headless-arch-nvidia-dwm.git && cd headless-arch-nvidia-dwm
sudo cp nvidia-xconfig.conf /etc/X11/xorg.conf.d/10-headless.conf
cp .xinitrc ~/.xinitrc
cp .zprofile ~/.zprofile
```
How to setup NVIDIA on Arch: [My previous .dotfiles](https://github.com/sidicer/dotfiles) or [Arch Linux NVIDIA drivers installation guide](https://github.com/korvahannu/arch-nvidia-drivers-installation-guide)

---
`nvidia-xconfig.conf` should be copied to `/etc/X11/xorg.conf.d/10-headless.conf`
```sh
sudo cp nvidia-xconfig.conf /etc/X11/xorg.conf.d/10-headless.conf
```
It was generated by plugging in a monitor temporarily and exporting the X11 settings from `nvidia-settings`, but can confirm that it will work on any system without modification

---
`.xinitrc` is mainly default taken from `/etc/X11/xinit/xinitrc`, just replaced `twm, xclock, xterm` startup commands with  
```sh
nvidia-settings --load-config-only & sunshine & exec dwm
```
which loads nvidia user settings from `~/.nvidia-settings-rc`, starts `sunshine` server and `DWM`

---
`.zprofile` just has `startx` to auto-start X server.
```sh
# If not running zsh:
mv .zprofile ~/.bash_profile
```

## Steam Big Picture NVIDIA (Stuttering/Laggy) Fix:
[Original comment in Issue #8918](https://github.com/ValveSoftware/steam-for-linux/issues/8918#issuecomment-1574456384)

Modify `~/.local/share/Steam/ubuntu12_64/steam-runtime-heavy/run.sh`  
Replace last `exec "$@"` with:
```sh
export XDG_CONFIG_HOME="$HOME/.config"
# Not steamwebhelper so skip
if [[ "$1" != *steamwebhelper* ]]; then
  exec "$@"
  exit
fi

args=()

# Read blocklist from ~/.config/steam-flag-blocklist.conf
blocklisted_flags=()
while read flag; do
    blocklisted_flags+=("$flag")
done < "$XDG_CONFIG_HOME/steam-flags-blocklist.conf"

# Filter arguments using the blocklist
for arg in "$@"; do
  include_arg=true

  for blocklisted_flag in "${blocklisted_flags[@]}"; do
    if [[ "$arg" == "$blocklisted_flag" ]]; then
      include_arg=false
    fi
  done

  if $include_arg; then
    args+=("$arg")
  fi
done

# Add additional flags from ~/.config/steam-flags.conf
while read flag; do
    args+=("$flag")
done < "$XDG_CONFIG_HOME/steam-flags.conf"

# Execute
echo "${args[@]}" >> /tmp/steam-args
exec "${args[@]}"
```

Then create two configuration files: `~/.config/steam-flags-blocklist.conf` and `~/.config/steam-flags.conf`:
```sh
cat <<EOF >>~/.config/steam-flags-blocklist.conf
--disable-gpu
--disable-gpu-compositing
--use-angle=gl
--disable-smooth-scrolling
EOF
cat <<EOF >>~/.config/steam-flags.conf
--ignore-gpu-blocklist
--disable-frame-rate-limit
--enable-gpu-rasterization
--enable-features=VaapiVideoDecoder
--use-gl=desktop
--enable-zero-copy
EOF
```
