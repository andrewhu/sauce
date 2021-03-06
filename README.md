## sauce
configs, dotfiles, and post-installation reminders

[Server configs](server) \
[Wallpapers](wallpapers)

#### [Use iGPU for display](https://askubuntu.com/questions/1061551/)
First check iGPU
```
$ lspci  | grep VGA
00:02.0 VGA compatible controller: Intel Corporation Device 3e92
01:00.0 VGA compatible controller: NVIDIA Corporation GP104 (rev a1)
```
Then `/etc/X11/xorg.conf`:
```
Section "Device"
    Identifier      "intel"
    Driver          "intel"
    BusId           "PCI:0:2:0"
EndSection

Section "Screen"
    Identifier      "intel"
    Device          "intel"
EndSection
```

#### Vim config: `~/.vimrc`
```
set tabstop=4
set shiftwidth=4
set expandtab
set autoindent

syntax on
set showtabline=4
set number
```

#### `~/.inputrc`
```
set completion-ignore-case On
```

#### Configure swap space
```
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile 
sudo mkswap /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
`/etc/sysctl.conf`:
```
vm.swappiness=10 # Only use swap when absolutely necessary
vm.vfs_cache_pressure=50
```

#### Disable mouse acceleration 
```
apt install dconf-editor
org -> gnome -> desktop -> peripherals -> mouse
accel-profile = 'flat'
```

#### Remap mouse buttons: `~/.xsessionrc`
```
mouse_id=$(xinput | grep "Logitech M585/M590" -m 1 | sed 's/^.*id=\([0-9]*\)[ \t].*$/\1/')
xinput set-button-map $mouse_id 1 2 3 4 5 6 7 2 9 10 11 12 13 14 15 16 17 18 19 20
```
