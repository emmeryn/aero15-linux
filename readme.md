# Linux on the Gigabyte Aero 15

## Introduction
Random and miscellaneous information about installing/running Linux on the Gigabyte Aero 15.
Currently using Linux Mint 19 Cinnamon.
Information should be universally applicable across the various Aero 15 generations and models (I'm running an Aero 15W v8 which has an i7-8750H and an nVidia GTX 1060).

## Updates
* 20190413: If you're experiencing intermittent system freezes, update the kernel to 4.18.0. I haven't had a single system freeze since the kernel update. To do that, open Update Manager (for Linux Mint), go to View -> Linux Kernels -> 4.18, select a kernel and click Install. Reboot.

* 20181111: 390.77 driver is a mixed bag.
	* Pros:
		* V-Sync works properly on HDMI output
		* HDMI Audio only works with 390.77
	* Cons:
		* nVidia card is the default render device - was unable to figure out how to tell chromium-vaapi to use the iGPU for video acceleration. In fact `/dev/dri/renderD128` (iGPU on this laptop) device seems to be hardcoded in the chromium-vaapi patch so I don't think it's a problem with chromium-vaapi. Rather, `vainfo` (and other vaapi-dependent apps?) defaulted to the dGPU. I could get `vainfo` to display the iGPU's decoding capabilities by running `vainfo --display drm --device /dev/dri/renderD128`, but [setting the `LIBVA_DRIVER_NAME` environment variable](https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Configuring_VA-API) to `i965`and running `vainfo` did not resolve the error.
		* the nVidia dGPU does not expose VP9 decode capability in VDPAU. In theory it does expose H.264 decode but somehow couldn't get it to work in chromium-vaapi.
		* Battery life on `prime-select intel` mode is 3hr+, no improvement.
	* The best compromise would probably be to use a USB-C to HDMI dongle with the new driver and setting `prime-select intel`. You get Intel's hardware accelerated decode capabilities and HDMI audio support immediately. When you need gaming performance or working V-Sync, use the HDMI port on the laptop. I do find it really lame to use a dongle on a laptop with so much I/O though :(
	* Who do I even ask/how do I help to resolve these issues?
	* I've gone back to the old driver via Timeshift for now.

## Installation

### Installer doesn't boot/freezes
Follow instructions from [Solving freezes during the boot sequence](https://www.linuxmint.com/rel_tara_cinnamon.php).

Summary:
* Wait for GRUB menu to appear
* Ensuring that the Linux option is selected (usually the first option), press the E key
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
Update: If you're on `prime-select nvidia`, you'll get the above. If you're on `prime-select intel`, you'll get VP9 hardware decode support.

### HDMI Output
The Aero 15's HDMI port is hardwired to the dGPU. Thus (as of nVidia 390.77), you need to **enable the nVidia dGPU** to use the HDMI port.

Note: The 390.48 nVidia driver seemed to allow HDMI output even when the nVidia dGPU was disabled. This "broke" after I updated to 390.77, and troubleshooting led me to discover that HDMI output wasn't really broken, but I had to enable the dGPU. It makes sense because the HDMI port is wired to the dGPU - even on Windows, there are reports of users having the dGPU active while using HDMI output. So, I assume the old drivers did not _really_ turn off the dGPU...

### HDMI Audio
Only works on `prime-select nvidia`.
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
* V-Sync on iGPU (slight UI/video tearing)

## What's wonky
* Weird behaviour when turning off the laptop display
	* [Found video showing the problem](https://streamable.com/wsuz5)
* Battery life is fairly bad (3hr)

## Misc QoL stuff

### Reduce heat/fan noise/performance:
`echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo` as root.

### Run laptop monitor at 60Hz instead of 144Hz
I don't believe this really makes a big difference to be honest.

`xrandr --output eDP-1 --rate 60 --mode 1920x1080`
