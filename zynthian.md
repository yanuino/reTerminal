
# Overview

This document cover the process of installing Zynthian in a [reTerminal](https://www.seeedstudio.com/ReTerminal-with-CM4-p-4904.html). The reTerminal is a Raspberry Pi All in One Board: Powered by RPi CM4 32GB, integrated with IPS multi-touch screen, dual-band Wi-Fi & Bluetooth, pre-installed compatible Linux system.

![Image: SeeedStudio reTerminal (c)](reterminal.jpg)

# Zynthian installation

Follow the instructions of the offical Zynthian web site [Zynthian.org Software](https://zynthian.org/#software) and the [Wiki](https://wiki.zynthian.org/index.php/Zynthian_Wiki_Home)

## Entering 'flash mode'

To flash the operating system into the reTerminal eMMC, you need to boot it in a special mode: [Flash Raspberry Pi OS/ 64-bit Ubuntu OS or Other OS to eMMC](https://wiki.seeedstudio.com/reTerminal/#flash-raspberry-pi-os-64-bit-ubuntu-os-or-other-os-to-emmc)

In short: remove the back cover and the heatsink, and flip down the boot switch inside. Then, after installing the drivers in your PC/Mac, connect the reTerminal to your host using the USB-C cable. If everything is ok, your host will see it like an USB drive.

Flash the Zynthian image, disconnect the reTerminal from your computer and switch back the boot switch to normal mode.

# Installing reTerminal drivers

**You need network conection and a HDMI display connected to the microHDMI port could help a lot** Read [Accessing_Zynthian_from_your_computer](https://wiki.zynthian.org/index.php/Accessing_Zynthian_from_your_computer) from the Zynthian Wiki.

Power up the reTerminal and wait a couple of time. And then, connect ot ssh.

```shell
ssh root@zynthian.local
```

Once you are inside the reTerminal, it's time to download and install the needed drivers. Read the reTerminal WiKi: [Install reTerminal drivers after flashing new Raspberry Pi OS/ Ubuntu OS or Other OS](https://wiki.seeedstudio.com/reTerminal/#install-reterminal-drivers-after-flashing-new-raspberry-pi-os-ubuntu-os-or-other-os):

```shell
git clone --depth 1 https://github.com/Seeed-Studio/seeed-linux-dtoverlays
```

Now, enter in that directory and run the installation script:

```shell
cd seeed-linux-dtoverlays
./scripts/reTerminal.sh --keep-kernel
```

This will install some packages, and compile the kernel modules needed for the hardware to work.

---

When the script finishes, it will instruct you to reboot, and we obediently do so. If everything goes well, after rebooting the display will show the Zynthian logo while it tries to start. But it will restart again and again, and the reason is the sound card. When you entered in the SSH terminal, you probably saw something like this:

![Zynthian SSH screen](zynthian-ssh-screen.png)

As you can see, the selected soundcard is `RBPi Headphones`, but the CM4 (and thus the reTerminal) does not have such a thing. We need to change it, so open the web configuration interface of Zynthian (http://zynthian.local) in your browser, and go to `Hardware > Audio`.

![Alt text](webconf-choose-audio.png)

There, for now, select `Dummy Device` and click `Save`. It will ask you to reboot, but if you do, you will loose the screen again. Why? Because when you pres `Save`, Zynthian modifies the file `/boot/config.txt`, which was also changed by the driver installation of reTerminal (changes that will be lost). So, before rebooting, go to `Hardware > Options`, click on `Advanced view`, and put the following line in the `Custom Config` area:

```
dtoverlay=dwc2,dr_mode=host
dtparam=ant2
disable_splash=1
ignore_lcd=1
dtoverlay=vc4-kms-v3d-pi4
dtoverlay=i2c3,pins_4_5
gpio=13=pu
dtoverlay=reTerminal,tp_rotate=0
```

Now click `Save` again, and reboot (you may need to reboot from the SSH terminal, as the Zynthian UI is not working properly yet). Then, after the reboot, you must see the Zynthian UI in the reTerminal screen. **Congratulations!** :)

Next thing to do: fix the touch screen panel.

## Fixing the display
From terminal (putty or command prompt)
type this — DISPLAY=:0 xrandr
(this helps with findinng out which DSI the display is connected. Should be the same for all 5 inch Reterminal but just in case things change)

Create this path sudo nano /usr/share/X11/xorg.conf.d/90-monitor.conf

Paste the text below
```
Section “Monitor”
Identifier “DSI-1”
# This identifier would be the same as the name of the connector printed by xrandr
# for example “DVI-I-1 connected primary” means that the identifier is “DVI-I-1”
# another example “Unknown19-1 connected primary” some GPIO screens identify as Unknown19

Option "Rotate" "right"
# Valid rotation options are normal,inverted,left,right

Option "PreferredMode" "720x1280"
# May be necesary if you are not getting your prefered resolution.
EndSection
```

Setting the correct width and height (1280 720) in Zynthian could help

## Fixing the touch screen

Before doing anything else, it's a good idea to update the software to the latest release. You can do it using the web interface. Go to `Software > Update` and click on the button `Update Software`. Wait until finishes. This will update the Zynthian software, but not the system packages. If you also want to upgrade the system, open an SSH terminal and run (remember to reboot when finished):

```shell
apt update
apt upgrade -y
```

Ok, ready for fixing the touch screen. The problem is simple: the X asis is inverted. And the fix is also very simple. Connect to reTerminal using SSH, and open the file `/usr/share/X11/xorg.conf.d/40-libinput.conf`. Using `nano`, the command would be:

```shell
nano /usr/share/X11/xorg.conf.d/40-libinput.conf
```

There, look for the section `InputClass` with the Identifier `libinput touchscreen catchall`, and in that section, add the following option: `Option "InvertX" "1"`. The whole section would be as follows:

```
Section "InputClass"
        Identifier "libinput touchscreen catchall"
        MatchIsTouchscreen "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"
        Option "InvertX" "1"
EndSection
```

Save the file (press `Ctrl + X`, then `Y`, then `Enter`), and restart the Zynthian service:

```shell
systemctl restart zynthian
```

Press the mute button to check if the touch screen is working. Or press in the upper left corner to open the main menu. If it works, **give yourself a pat on the back!** ;)

More stranger things: with the TP working, if I print its xinput properties, it says that the device is not enabled (even though it is working):
```
xinput list-props seeed-tp
Device 'seeed-tp':
	Device Enabled (145):	0
[...]
```
Then, I can ‘break’ it touching the lower Y axis part, and it stops working, as always. BUT, I can enable it again, using:
```
xinput enable seeed-tp
xinput list-props seeed-tp
Device 'seeed-tp':
	Device Enabled (145):	1
[...]
```
And then, it works again, and I can use the whole screen! :joy: It won’t stop working until the next reboot, when the device is disabled again. I need to check why is it disabled in the first instance, but at least, the problem is easy to workaround.

UPDATE: For those wondering, I ended up adding the following systemd service to fix the problem:
```
# cat /etc/systemd/system/xinput-enable-tp.service 
[Unit]
Description=Run xinput to enable seeed-tp device
After=zynthian.service

[Service]
Environment=DISPLAY=:0
Environment=XAUTHORITY=/root/.Xauthority
Type=oneshot
ExecStart=/bin/bash -c 'while ! $(xset q > /dev/null 2>&1); do sleep 2; done; sleep 5; xinput enable seeed-tp'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
It waits until Zynthian and Xorg are running, and then, after 5 seconds, it does the xinput trick. It’s not the most elegant solution, but as X is handled by Zynthian, and I don’t want to loose the hack on the next update, I couldn’t come up with a better solution :sweat_smile:

Next step? The sound system.


## Fixing the sound system

As we said earlier, this device **does not have a soundcard** (as far as I know), so I will use an external USB card. The problem with these cards is that they may be enumerated differently on every boot. So, for instance, if you instruct Zynthian to use the device number 3 (because the USB sound card now has the id number 3), and on next reboot it gets the number 2, Zynthian will not be able to start. How to fix it? Well, there are many ways. What I've choosen to do here is to setup ALSA (the sound system) to give any USB sound card a fixed number. If you only have one, this fix may work for you too.

How to do it? Again, connect to reTerminal using SSH, and create the file `/etc/modprobe.d/alsa-base.conf`. With nano, the command would be:

```shell
nano /etc/modprobe.d/alsa-base.conf
```

There, enter the following line (I've choosen the index number 5, you can pick up the number of your preference):

```
options snd_usb_audio index=5
```

Save the file (press `Ctrl + X`, then `Y`, then `Enter`), and connect your USB device. To check that it received the number that you set, run `aplay -l`, and check the name/number of your cards. In my case, I'm using a Hercules DJControl Starlight as sound interface, so that command says:

```shell
**** List of PLAYBACK Hardware Devices ****
card 0: Dummy [Dummy], device 0: Dummy PCM [Dummy PCM]
[...]
card 5: Starlight [DJControl Starlight], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

Index 5, good. Now, you can go again to the Zynthian web configuration interface, to `Hardware > Audio`. There, change the `Dummy Device` that we put earlier, and set it to `Custom Device`. Check the `Advanced View` box, and observe the field `Jackd Options`. There, by default you will find this option: `-d hw:0`. Change it with the index that you put in the `alsa-base.conf` file. I used 5, so the option would be, in my case: `-d hw:5`. Press `Save` and, as instructed, reboot.

Once rebooted, you can check that the audio is working. Using the touch screen, press the upper left corner to enter the main menu. There, go to `Admin` and tap in `Test Audio` to verify that it works. You should hear a pretty special song... :D


## Using reTerminal buttons

The reTerminal has 5 physical buttons: 4 in the front, labelled F1, F2, F3 and a green one with a circle; and one more in the upper side. These buttons are mapped by the driver as the keys 'A', 'S', 'D', 'F' (for the front buttons) and SLEEP (for the upper side one), respectively. I'm perfectly fine with the ASDF keys, but not that much with the SLEEP key, as it could not be mapped inside Zynthian.

Let's change it for something that Zynthian recognizes. I've choosen a key that can be used withouot modifiers (i.e. Shift, Control or Alt), but that is rarely used (so that, in case you connect a keyboard, you don't fire that function by accident): Scroll Lock. To change it, open an SSH console to your reTerminal, and enter in the driver directory:

```shell
cd drivers/seeed-linux-dtoverlays/overlays/rpi/
```

Open (with nano) the file called `reTerminal-overlay.dts`, and go to the section called `fragment@7` (nearly the end of the file). There, you can see the GPIO keys mapping. In the case named `pwr_btn`, change its `linux,code` value for the selected KEY code. In my case, Scroll Lock is 70, so the whole block would look like this:

```c++
    pwr_btn: pwr_btn {
        debounce-interval = <5>;
        linux,code = <70>; /* PWR_KEY, KEY_SCR_LOCK */
        label = "power_key";
        gpios = <&gpio 13 1>;
        gpio-key,wakeup;
        autorepeat;
    };
```

Save the file, and then re-run the installation script:

```shell
cd ~/drivers/seeed-linux-dtoverlays/
scripts/reTerminal.sh
```

> **NOTE:** After installing the drivers, the script will have changed the file `/boot/config.txt` (just like Zynthian did before), adding some options that we don't need. You can discard the changes going to the Zynthian web interface, to `Hardware > Options`, and pressing `Save`, whithout changing anything else.

Now, you can reboot as instructed. Once the system is up again, go to the web interface and bind these keys to something useful for you. In my case, F1 is 'Back', F2 is 'Up', F3 is 'Down', green button is 'Enter' and the upper side button is 'Power Off'. The mapping is something like this:

![Alt text](key-mappings.png)


# Troubleshooting

* When using an external USB sound card, you may find that jack does not work properly: the command `jack_wait -c` does not return `running`, and the Zynthian UI shows an error message. Sometimes, this may be fixed disabling the sound card input port. Open the webconf tool, and go to `Hardware > Audio`. There, in the *Jackd Options* field, add `-P` after `-d alsa`. Reboot to test if it worked.


# References

### Zynthian

* [Zynthian](https://zynthian.org/)

### reTerminal
* [Seeed Studio shop: reTerminal](https://www.seeedstudio.com/ReTerminal-with-CM4-p-4904.html)
* [reTerminal Wiki](https://wiki.seeedstudio.com/reTerminal/)
* [reTerminal: installing an OS to the eMMC](https://wiki.seeedstudio.com/reTerminal/#flash-raspberry-pi-os-64-bit-ubuntu-os-or-other-os-to-emmc)
