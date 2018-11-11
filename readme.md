# Linux on the Gigabyte Aero 15

## Introduction
Random and miscellaneous information about installing/running Linux on the Gigabyte Aero 15.
Currently using Linux Mint 19 Cinnamon.
Information should be universally applicable across the various Aero 15 generations and models (I'm running an Aero 15W v8 which has an i7-8750H and an nVidia GTX 1060).

## Updates
Further testing required with nVidia 390.77 driver. I'm considering using Timeshift to go back to the old driver, but if this driver properly turns off the dGPU and allows the laptop to get the promised 9-hour battery life...

## Installation

### Installer doesn't boot/freezes
Follow instructions from [Solving freezes during the boot sequence](https://www.linuxmint.com/rel_tara_cinnamon.php).

Summary:
* Wait for GRUB menu to appear
* Ensuring that the Linux Mint option is selected, press the E key
* Replace `quiet splash` with `nomodeset`
* Press Ctrl-X or F10 to continue boot

### Post-installation
Install the proprietary nVidia drivers (in Ubuntu/Linux Mint, you can do that via the Driver Manager GUI). 

## What works
* CPU Power Management
* Display (1080p@144Hz)
	* Backlight control works via software, not via Fn-keys
* Bluetooth
	* The dedicated Bluetooth LED doesn't seem to ever turn off though.
* Audio
	* Speakers, 3.5mm jack, HDMI Audio (see below)
* Keyboard
	* Only sleep and volume Fn-keys work, none of the rest do.
		* They aren't detected by `acpi_listen` either, wonder what we can do to get them working...
	* Keyboard RGB lighting remembers your last configuration from Windows.
		* [WIP Linux tool to configure keyboard lighting](https://github.com/martin31821/fusion-kbd-controller)
* USB-C Lightning port
	* HDMI seems to work with an adapter.

## What can be made to work
### Hardware video acceleration within web browser
(This is a general Linux thing, not particular to this laptop.)
There's a patched version of Chromium with VAAPI support maintained by Saikrishna Arcot. [Link to the PPA](https://launchpad.net/~saiarcot895/+archive/ubuntu/chromium-beta)

Follow steps on [Linux Uprising blog](https://www.linuxuprising.com/2018/08/how-to-enable-hardware-accelerated.html).

Summary:
```
sudo add-apt-repository ppa:saiarcot895/chromium-dev
sudo apt-get update
sudo apt install chromium-browser

sudo apt install i965-va-driver # Intel iGPU
sudo apt-get install vdpau-va-driver # nVidia dGPU
```
Open the patched version of Chromium, navigate to `chrome://flags/#enable-accelerated-video`, enable and restart.

Note: There should be no need to install h264ify browser extension to force YouTube to use H.264, since both Kaby Lake and Coffee Lake iGPUs have hardware VP9 decoding support. However, seems like that's not necessarily true anymore after updating to nVidia drivers 390.77, because if you have `prime-select nvidia` to turn on the dGPU, it will try to use the dGPU to decode the video. And nVidia's Pascal GPUs don't seem to support VP9 hardware decode on Linux...?
```
$ vainfo
libva info: VA-API version 1.1.0
libva info: va_getDriverName() returns 0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/nvidia_drv_video.so
libva info: Found init function __vaDriverInit_1_0
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.1 (libva 2.1.0)
vainfo: Driver version: Splitted-Desktop Systems VDPAU backend for VA-API - 0.7.4
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            :	VAEntrypointVLD
      VAProfileMPEG2Main              :	VAEntrypointVLD
      VAProfileMPEG4Simple            :	VAEntrypointVLD
      VAProfileMPEG4AdvancedSimple    :	VAEntrypointVLD
      <unknown profile>               :	VAEntrypointVLD
      VAProfileH264Main               :	VAEntrypointVLD
      VAProfileH264High               :	VAEntrypointVLD
      VAProfileVC1Simple              :	VAEntrypointVLD
      VAProfileVC1Main                :	VAEntrypointVLD
      VAProfileVC1Advanced            :	VAEntrypointVLD
```
This means if you're using the HDMI output you don't get hardware VP9 decode acceleration...T_T.

### HDMI Output
The Aero 15's HDMI port is hardwired to the dGPU. Thus (as of nVidia 390.77), you need to **enable the nVidia dGPU** to use the HDMI port.

Note: The 390.48 nVidia driver seemed to allow HDMI output even when the nVidia dGPU was disabled. This "broke" after I updated to 390.77, and troubleshooting led me to discover that HDMI output wasn't really broken, but I had to enable the dGPU. It makes sense because the HDMI port is wired to the dGPU - even on Windows, there are reports of users having the dGPU active while using HDMI output. So, I assume the old drivers did not _really_ turn off the dGPU...

### HDMI Audio
Follow instructions from [this post](https://devtalk.nvidia.com/default/topic/1024022/linux/gtx-1060-no-audio-over-hdmi-only-hda-intel-detected-azalia/post/5230494/#5230494) (credit to user_0462)

Summary: 
```
git clone https://github.com/hhfeuer/nvhda
cd nvhda
make
sudo make install
echo nvhda | sudo tee -a /etc/initramfs-tools/modules
echo "options nvhda load_state=1" | sudo tee /etc/modprobe.d/nvhda.conf
sudo update-initramfs -u
```
## What doesn't
* Ambient light sensor

## What's wonky
* Weird behaviour when turning off the laptop display
	* [Found video showing the problem](https://streamable.com/wsuz5)
* Battery life is fairly bad (3-4hr)
	* Not tested with nVidia 390.77 drivers yet

## Misc QoL stuff

### Reduce heat/fan noise/performance:
`echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo` as root.

### Run laptop monitor at 60Hz instead of 144Hz
I don't believe this really makes a big difference to be honest.

`xrandr --output eDP-1 --rate 60 --mode 1920x1080`
