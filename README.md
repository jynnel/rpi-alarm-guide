# Setting up Arch Linux ARM on a Raspberry Pi 4 Model B
This post details my experiences setting up Arch Linux ARM on a Raspberry Pi 4 Model B with the official 7" touch screen display. The results are a very capable miniature computer running Arch with a touch screen and the ability to attach external displays via the Pi 4's mini HDMI ports. Other features include clear audio over the 3.5mm jack, bluetooth audio (a USB adapter is recommended), capable OpenGL drivers, and more.

Here's the basic hardware I'm using:
* Raspberry Pi 4 Model B 4GB
* Raspberry Pi 7" Touch Screen Display
* SmartiPi Touch 2 case (has to be model 2 for the Pi 4)

This guide assumes you are starting with another Arch install for inital setup. I recommend getting it booting and doing some simple configuration before installing the board into the case. Once you're ready to put it together, follow the guide here: https://smarticase.com/pages/smartipi-touch-2-setup-1

### Installing
To start, you will want to load an SD card with Arch Linux ARM. To do that, you can follow the installation instructions here: https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4

### Further Setup
You can either insert the SD card on the Raspberry Pi and follow the instructions from there, running the `pacman-key` commands directly on the system, or you can simply use `arch-chroot` to chroot into the SD card and configure the system from your primary machine. But you can't just chroot into an ARM system. To do this you will need to install `qemu-arm-static` from the AUR and use the PKGBUILD in the [binfmt-manager repo on github](https://github.com/mikkeloscar/packages/tree/master/binfmt-manager) to install `binfmt-manager`.

Once you have both installed, assuming you have your SD card mounted to /mnt/, you can run the following commands to chroot in (you only need to run the first one once):
```
sudo binfmt_manager register arm,armeb
sudo arch-chroot /mnt qemu-arm-static /bin/bash
```
If you'd rather SSH into the system while it's running on the Pi, look up the Pi's IP address on your router and use a command similar to `ssh alarm@192.168.1.180`, replacing the correct local IP address. The shell might not work correctly—backspace might not be working, nor will you be able to run basic commands—but this was fixed for me by entering `export TERM=xterm` during the SSH session.

You can also use `scp` to transfer files to the Pi from your other machine. For example:
```
scp /path/to/file alarm@192.168.1.180:/home/alarm
```

So, if you haven't already, you'll want to run these commands:
```
pacman-key --init
pacman-key --populate archlinuxarm
pacman -Syu
```

## Configuring the System
### Setting Time
Find your correct timezone in `/usr/share/zoneinfo`. Then run these commands (replacing your timezone):
```
timedatectl set-ntp true
timedatectl set-timezone "America/Los_Angeles"
```

### Basic Tools
To start you will want the `git` and `base-devel` packages. Run: `pacman -S git base-devel`.

### Extend the Life of your SD Card
A basic configuration to your `/etc/fstab` would be to change `defaults` to `defaults,noatime`. `noatime` also induces `nodiratime`, so there's no need to add both. This turns off writing access times to files and directories.

### Sync Stuf
You'll want a number of packages to make the system run X11, have a window manager, etc. Here's my basic list.
```
pacman -S git base-devel
pacman -S xorg-server xf86-video-fbdev xorg-xrefresh mesa
pacman -S xf86-input-evdev xorg-xinput
pacman -S i3-wm i3blocks i3status dmenu termite htop
pacman -S chromium onboard ttf-dejavu
pacman -S lightdm-gtk-greeter
```

### Sudo
To make `sudo` work on the `alarm` account, enter these commands:
```
groupadd sudoers
usermod -a -G sudoers alarm
```
If you must, it isn't complicated to set the `alarm` account to not require a password on `sudo`.
```nano /etc/sudoers.d/myOverrides```
Add the line:
```alarm ALL=NOPASSWD:ALL```. Now you're living on the wild side.

### AUR Helper
`yay` is the one to use these days.
```
git clone https://aur.archlinux.org/yay.git
cd yay
mkpkg -si
```
`-si` syncs deps and installs the package. If you make it `-sri`, it will remove build dependencies for you. You can also run `yay -c` to clean out orphan build dependencies, for example if you cancel during compile.

### Autologin
My system automatically logs into an X session with `lightdm` and a passwordless user called `touch` (without `sudo` privileges).
```
useradd touch
passwd -d touch
usermod -a -G autologin touch
usermod -a -G input touch
systemctl enable lightdm.service
```
To actually log in automatically you need to edit `/etc/lightdm/lightdm.conf`. Under the Seat section, uncomment the `autologin-user=` line and set it to `autologin-user=touch`.

LightDM reads the `~/.xprofile` file for the user logging in. Use this to launch applications and run commands at startup.

If you've copied config files from a backup into `/home/touch`, you'll want to run `chown -R touch /home/touch`.

### PAM Timeouts
I was having a problem where I kept getting logged out automatically. After checking `journalctl` I found it was due to PAM user timeouts on the `lightdm` user. To fix this, run: `sudo chage -E -l lightdm`. There's probably a better solution, but this works for now.

### Networking
I use NetworkManager because it has an easy-to-use applet.
```
pacman -S network-manager-applet
systemctl enable NetworkManager.service
```
However, with this I wasn't getting access to most webpages over WiFi. I had to edit `/etc/nsswitch.conf` and in the line that starts with `hosts:`, change `resolve` to `dns`.

### Audio
`nano /boot/config.txt`, add: `dtparam=audio=on` and `audio_pwm_mode=2`. You may want to sync `pulseaudio-alsa` as well. For me, this wasn't enough; audio quality was poor, full of static and crackling. You'll want to edit `/etc/pulse/default.pa` and change the line that reads `load-module module-(u)dev-detect` to `load-module module-udev-detect tsched=0`.

### Bluetooth
The Raspberry Pi has Bluetooth onboard, but I couldn't get good audio output over this device. Nonetheless, to set up Bluetooth you will need to sync a number of packages.
```
pacman -S bluez bluez-libs bluez-utils blueman pulseaudio-bluetooth pavucontrol
yay -S pi-bluetooth
```
You may need to add a service file to run `btattach` so your adapter becomes active.
`/etc/systemd/system/rpi-btattach.service`
```
[Unit]
Description=Start btattach, needed for Bluetooth capabilities.

[Service]
Type=simple
# a delay is needed
ExecStartPre=/bin/sleep 1s
ExecStart=/usr/bin/btattach -B /dev/ttyAMA0 -P bcm -S 3000000

[Install]
WantedBy=multi-user.target
```
Then:
```
systemctl start rpi-btattach.service
systemctl enable rpi-btattach.service
```
However, I highly recommend using a USB adapter if you're going to be sending audio. 'Plugable' makes good one that worked out of the box for my desktop running Arch as well as on the Pi (with Bluetooth drivers installed).

### GTK3 Touch Screen Compatibility
There's a bug in GTK+ 3 that causes the touch screen tap events to get interpreted as stylus hover events, so you can't click on sub-menus. This affects `nm-applet`, such that you won't be able to connect to most WiFi networks using the system tray icon menu. Thankfully, someone made a fix. Read about it and download the script in the comments here: https://gitlab.gnome.org/GNOME/gtk/issues/945

Edit the script as instructed (in the script itself) and run it on startup if you wish. I tried setting up a systemd service, but found that the script worked for me without root privileges, so I just added it to my `~/.xprofile`.

### Xscreensaver Power Options
`pacman -S xscreensaver`, run `xscreensaver-demo`, configure it to "blank only," and navigate to the power options. You can configure it to turn off the display when the screensaver activates. You'll want to add `xscreensaver -nosplash &` to your `~/.xprofile`.

However, I found that after enabling the OpenGL drivers (see somewhere below) that xscreensaver no longer turned off the touch display, but whatever was connected to the HDMI port (even if nothing was connected). To solve this I created a Python script that runs on startup. Making a systemd service might be cleaner, but I don't mind having a cluttered `~/.xprofile`.
```
import subprocess, sys
from time import sleep

def execute(command):
    process = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)

    # Poll process for new output until finished
    while True:
        nextline = process.stdout.readline().decode()
	
        if nextline == '' and process.poll() is not None:
            break
        elif nextline.startswith("BLANK"):
            subprocess.run("sh -c '/bin/echo 1 > /sys/class/backlight/rpi_backlight/bl_power'", shell=True)
        elif nextline.startswith("UNBLANK"):
            subprocess.run("sh -c '/bin/echo 0 > /sys/class/backlight/rpi_backlight/bl_power'", shell=True)

        sys.stdout.write(nextline)
        sys.stdout.flush()

        sleep(0.25)

    output = process.communicate()[0]
    exitCode = process.returncode

    if (exitCode == 0):
        return output
    else:
        raise ProcessException(command, exitCode, output)

execute("xscreensaver-command --watch")
```

### Backlight Control
For backlight control, I went with the python script [rpi-backlight](https://github.com/linusg/rpi-backlight). I just installed it with pip, for simplicity's sake. `pacman -S pip` and `pip3 install rpi-backlight`. In order for it to work, you need to add a permissions file. `su alarm` (if you're logged into `touch`) and run this command:
```
sudo echo 'SUBSYSTEM=="backlight",RUN+="/bin/chmod 666 /sys/class/backlight/%k/brightness /sys/class/backlight/%k/bl_power"' | sudo tee -a /etc/udev/rules.d/backlight-permissions.rules
```
Then, you can adjust the brightness of the touch display by running, for example:
```
rpi-backlight --set-brightness 30 --duration 0.2
```

### Turn off Backlight on Poweroff
It's kind of a hack, but if you want to run a script at shutdown, you can add it to `/lib/systemd/system/system-shutdown`. For example, here's my `/lib/systemd/system/rpi-display-poweroff`:
```
#!/bin/bash
bin/sleep 5
bin/sh -c '/bin/echo 1 > /sys/class/backlight/rpi_backlight/bl_power'
```
I added the `sleep 5` so I could see some of the shutdown messages before the display turned off. With this script, you still miss the final few messages, but at least there's still something.

### Enabling OpenGL
On Arch Linux ARM, you don't want to run the default `raspi-config`, or `rpi-update` for that matter (I bricked my system doing that). Thankfully, there's one on the AUR stripped down for Arch.
```
yay -S raspi-config
```
Run it and, under Advanced, you can enable the OpenGL driver. I chose "fake KMS," because I think I read "full KMS" doesn't work for the touch screen display. I haven't tried it yet. If you plan to output to external HDMI displays, you may also want to disable overscan, also under Advanced. Proceed to Finish to save the settings and get a prompt to reboot.

This will enable you to connect external HDMI monitors and use them at the same time as the touch display, but you may find that touch inputs now go to the external monitor and not the touch screen. This is an easy fix. Use `xrandr` to list your displays, and get the name of the touch display. It should be the 800x480 one. Then use `xinput --list` to list your input devices. The touch interface should be called `FT5406 memory based driver`.

Then, for example, run this command (which I simply added to my `~/.xprofile`—a hack, I know):
```
xinput —map-to-output ‘FT5406 memory based driver’ DSI-1
```

### Mirror or Switch to HDMI
If you want to disable the touch display and output to HDMI instead, edit `/boot/config.txt` and add the line `ignore_lcd=1` and reboot. Just make sure you have another display to connect, or another computer to SSH in or edit the config with if it doesn't work out. Remove the line to return output to the touch display.

Want to use both at the same time, but find the OpenGL driver too buggy? You can mirror the touch display to an attached HDMI display. Edit `/boot/config.txt` and add `max_framebuffers=2`. Then you want to `git clone https://github.com/AndrewFromMelbourne/raspi2fb` somewhere; I put such things in `~/pkg/`. Once you've made and installed it, you can add the @.service file in the repo to systemd and start it when you want to mirror your display. You may want to add `--fps 60` or somesuch to the `ExecStart` line for smoother mirroring.

## Finally
Don't forget to perform `pacman -Syu` regularly, especially before syncing new packages.

Congratulations! You now have a tiny, very capable computer. Hopefully you didn't spend too much money on it, like me.

![photo of a raspberry pi with a touch screen display](https://i.imgur.com/P9IhieD.jpg)

### Bonus: Installing FTE Quake World
```
yay -S ftequake
(wait for it to build and install)
fteqw-gl -basedir /path/to/folder/containing/id1
```
