---
layout: post
title: "WSL2 configuration"
date: 2023-08-04 12:00:00 +0800
categories: Windows
---
Setup WSL2 Ubuntu 22.04 development environment with GUI and audio

## GUI
2 GUI options:
- WSLg by Microsoft(recommended)
- VcXsrv X server

### WSLg (recommended)
C:\Users\<UserName>\.wslconfig

```ini
[wsl2]
;enable WSLg
guiApplications=true

memory=10GB
swap=8GB
```

C:\Users\<UserName>\.wslgconfig

```ini
[system-distro-env]
;scale 2x
WESTON_RDP_DEBUG_DESKTOP_SCALING_FACTOR=200
```

### VcXsrv
First of all, download and install VcXsrv.

Write a PowerShell script to start VcXsrv and WSL: C:\Users\<UserName>\start-wsl.ps1

```powershell
# Start "vcxsrv" if it's not running
if (Get-Process -Name vcxsrv -ErrorAction SilentlyContinue) {
    Write-Host "VcXsrv is running."
} else {
    Write-Host "VcXsrv is not running. Starting"
    Start-Process -FilePath "C:\Program Files\VcXsrv\vcxsrv.exe" -ArgumentList "-ac -multiwindow -clipboard -noprimary -wgl"
}
# Start windows terminal
wt
```

Create a shortcut on desktop to start it: `C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe -command "& '%userprofile%\start-wsl.ps1'"`. Remember to change Windows terminal settings and set WSL linux distro as the default profile.

Double-click the shortcut to start wsl terminal. In .bashrc, add:

```bash
# VcXsrv
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2; exit;}'):0.0
export LIBGL_ALWAYS_INDIRECT=1

# Scale UI(causes problem with WSLg)
export GDK_SCALE=2
```

Run `source ~/.bashrc` and try start a GUI app, such as: xfce4-terminal. Note that VcXsrv needs to be running before starting WSL.

## Fcitx
Install fcitx5: `sudo apt install fcitx5 fcitx5-chinese-addons`

In ~/.bashrc, add

```
# Fcitx
export INPUT_METHOD=fcitx
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
if ! pgrep dbus-launch &> /dev/null; then
  dbus-launch --sh-syntax --exit-with-session > /dev/null
fi
if ! pgrep fcitx5 &> /dev/null; then
  daemonize -e /tmp/fcitx5.log -o /tmp/fcitx5.log -p /tmp/fcitx5.pid -l /tmp/fcitx5.pid -a /usr/bin/fcitx5 --disable=wayland
fi
```

fcitx5 will be automatically started everytime you start WSL.

## PulseAudio
Download PulseAudio for windows: https://www.freedesktop.org/wiki/Software/PulseAudio/Ports/Windows/Support/. Extract it to C:\pulseaudio

Create a config.pa file at C:\pulseaudio\etc\pulse\config.pa

```
load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1;172.16.0.0/12
load-module module-esound-protocol-tcp auth-ip-acl=127.0.0.1;172.16.0.0/12
load-module module-waveout sink_name=output source_name=input record=0
```

Download nssm from https://nssm.cc/download. Extract and copy nssm.exe to C:\pulseaudio\bin\nssm.exe. Open a cmd or PowerShell at C:\pulseaudio\bin\, and run `nssm.exe install PulseAudio` for cmd, or `.\nssm.exe install PulseAudio` for PowerShell. In the dialogue, set:

* Path: C:\pulseaudio\bin\pulseaudio.exe
* Startup directory: C:\pulseaudio\bin
* Arguments: -F C:\pulseaudio\etc\pulse\config.pa --exit-idle-time=-1
* (optional) set a display name in "Details" tab

Click "Install" to install PulseAudio as a windows service. Win + R and run services.msc to review it. Now pulseaudio will automatically start on each windows launch.

In .bashrc, add:

```bash
# PulseAudio
export HOST_IP="$(ip route |awk '/^default/{print $3}')"
export PULSE_SERVER="tcp:$HOST_IP"
```
