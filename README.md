# linux_intel_monitor_overclocking
Guide for overclocking monitors on Linux with Intel GPUs.
--------

###Disclaimer

Use this information at your own risk. I do not claim responsability for any damage you might cause to your equipment, yourself or others.

###Intro

As you may already know, you can not overclock your monitor with xrandr with an Intel GPU (at least on Haswell or Broadwell GPU's I've tried).

This guide will try to help you overclock your monitor without xrandr directly.

It helps if you know the monitor can be overclocked to the frequency you desire before starting (using an Nvidia card for example, or a Windows operating system, since the Intel drivers on windows allow monitor overclocking, I'm not sure if AMD supports overclocking monitors on linux).  
Otherwise this might require considerable trial and error and be very frustrating.

###Software required:

* [xrandr](http://www.x.org/wiki/Projects/XRandR/)
* [gnu coreutils](https://www.gnu.org/software/coreutils/coreutils.html)
* [wine](https://www.winehq.org/) For an edid editor.
* [AW Edid Editor](http://www.analogway.com/en/products/software-and-tools/aw-edid-editor/#dl) Or another decent edid editor, if you find one. This one installs fine in wine.
* [QT](https://www.qt.io/) AW Edid Editor possibly requires QT5, it seems to work on QT4 with some warnings. It might work without QT, I can't say for sure.
* [gcc](https://gcc.gnu.org/) For compiling cvt
* [cvt](http://www.uruk.org/~erich/projects/cvt/cvt.c) See below for instructions/reasoning.

###Guide:

#####Compiling cvt.

If you do not care about using reduced blanking, or you want to use gtf, then ignore this and use cvt or gtf that comes with your distro.

cvt on my distro is compiled to only support multiples of 60hz for reduced blanking (which is fine according to cvt 1.1 spec, but going out of the spec works fine and I'm fine with it).

You can read [here](https://en.wikipedia.org/wiki/Coordinated_Video_Timings#Reduced_blanking) about reduced blanking.

I require reduced blanking to keep my pixel clock low, cvt without reduced blanking and gtf at 1920x1080@72hz have pixel clocks of 200+mhz (since I use HDMI, I'm limited to 165mhz or so - although I can go up to 180 or so before my monitor refuses to turn on).

Download and compile cvt:

`$ cd ~/ && wget http://www.uruk.org/~erich/projects/cvt/cvt.c && gcc cvt.c -O2 -o cvt -lm -Wall`

![cvt](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/cvt_compile.png)

(Optional) You can also edit cvt.c, changing the CLOCK\_STEP constant to something much lower (0.0000001 for example) to improve the precision, otherwise you might get like 71.99hz instead of 72hz. Recompile after changing CLOCK_STEP.

#####Find your GPU.

`$ lspci | grep -Pi 'intel.*graphics'`

![GPU](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_GPU.png)

We are interested in the following: `00:02.0`

#####Find the port the monitor is connected to:

`$ xrandr | grep -Pio '.*?\sconnected'`

![port](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_port.png)

#####Find the edid file using the above information:

`$ sudo find /sys/devices/ -name edid`

![find_edid](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_edid.png)

We are interested in:  
`/sys/devices/pci0000:00/0000:00:02.0/drm/card0/card0-HDMI-A-1/edid`

#####Copy the edid to your home folder:

`$ sudo cp /sys/devices/pci0000:00/0000:00:02.0/drm/card0/card0-HDMI-A-1/edid ~/edid.bin`

#####Print the edid:

`$ cat ~/edid.bin && echo ""`

![print_edid](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/print_edid.png)

You will see mostly unintelligible text, you might see the monitor's model number.  
You can use this to confirm this is indeed the edid file for the monitor.  
If not, we can confirm in one of the following steps.

#####Get a new modeline using cvt.

Run cvt with the screen width, screen height, refresh rate, -r if you need reduced blanking. You can alternatively use gtf if you know what you want/are doing.

`$ cd ~/ && ./cvt 1920 1080 72 -r`

![cvt_readout](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/cvt_readout.png)

We are looking for this: `Modeline "1920x1080_72.00_rb"  167.25  1920 1968 2000 2080  1080 1083 1088 1117  +HSync -Vsync`

#####Convert the modeline into timings.

Head to [this page](http://www.epanorama.net/faq/vga2rgb/calc.html)

Paste in the modeline above, in the box over the Import Modeline button.

Click Import Modeline. Keep this web page open.

![video_timing_calculator](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/video_timing_calculator.png)

#####Open the edid file in AW Edid Editor.

You can start the program and open the file or start it from the shell:  

`$ wine ~/.wine/drive_c/Program\ Files\ \(x86\)/ANALOG\ WAY/AW\ EDID\ Editor/AWEDIDEditor.exe ~/edid.bin`

You might be able to confirm this is your monitors edid by using the information on the top left of the GUI in the Standard Data tab or in the Detailed Data tab, look for Display Product Name.

#####Change the "Standard Timings".

In the Standard Data tab, change the Standard Timings (on the right) Refresh Rate to the desired refresh rate for the screen resolution you want.

For example, here I set 1920x1080 to 72hz.

![standard_timings](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/standard_timings.png)

#####Change the "Prefered Timings".

Head to the Detailed Data tab.

If you see the "Preferred Timing Block" and it is the resolution you are using (1920x1080 for example), continue this step, otherwise, skip to the next step.

Using the information we got on the Video timing calculator website, we can input this data.

To use the data on the website in the edid editor program I've written the following: (left is program, right is website)

Pixel Clock      -> Pixel clock frequency _ MHz  
Interlaced       -> Interlace  
H. Active Pixels -> Resolution _ pixels  
H. Blank         -> (you must calculate the following: Front porch _ pixels + Sync pulse _ pixels + Back porch _ pixels)  
H. Front Porch   -> Front porch _ pixels  
H. Sync Width    -> Sync pulse _ pixels  
H. Image Size    -> This is wideness in millimeters of your monitor, you shouldn't change this.  
H. Border        -> You shouldn't change this, this changes where the pixels start on the monitor.  
V. Active Pixels -> Resolution _ lines  
V. Blank         -> (you must calculate the following: Front porch _ lines + Sync pulse _ lines + Back porch _ lines)  
V. Front Porch   -> Front porch _ lines  
V. Sync Width    -> Sync pulse _ lines  
V. Image Size    -> This is height in millimeters of your monitor, you shouldn't change this.  
V. Border        -> You shouldn't change this, this changes where the pixels start on the monitor.  
V.Sync Polarity  -> If Sync pulse _ lines, polarity _ * is 1 , put a checkmark, otherwise remove the checkmark.  
H.Sync Polarity  -> If Sync pulse _ pixels, polarity _ * is 1 , put a checkmark, otherwise remove the checkmark.

![prefered_timings](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/preferred_timings.png)

#####Change the "Detailed Timings".

Head to the "CEA Extension" tab.

On the right, under the "Detailed Timing Blocks" section, click on the "n#" tab until you find your wanted resolution.

Change all these settings with the information in the previous step.

**MAKE SURE YOU FILE->SAVE_AS AND NOT FILE->SAVE**, doing file->save, the AW edid editor does not seem to overwrite the file. Save it as edid_custom.bin then exit the program.

![detailed_timings](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/detailed_timings.png)

#####Make linux use the modified edid file on boot.

You can make linux use the modified edid file when starting.

Copy the edid file to an appropriate folder:

`$ sudo mkdir -p /usr/lib/firmware/edid/ && sudo cp ~/edid_custom.bin /usr/lib/firmware/edid/edid_custom.bin`

With grub you can do it the following way (change nano to the editor of your choice):

`$ sudo nano /etc/default/grub`

Look for a line similar to this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`

Add the following to the end (change HDMI-A-1 based on what we found near the start of the guide): `drm_kms_helper.edid_firmware=HDMI-A-1:edid/edid_custom.bin`

It should now look like this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash drm_kms_helper.edid_firmware=HDMI-A-1:edid/edid_custom.bin"`

![grub_cfg](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/grub_cfg.png)

Now you can regenerate your grub.cfg file:

`$ sudo grub-mkconfig -o /boot/grub/grub.cfg`

[grub_refresh](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/grub_refresh.png)

If you don't use grub, see [this page](https://wiki.archlinux.org/index.php/Kernel_parameters).

#####(Optional) Write the edid to the monitor.

This is an optional, non recommended step.

Some monitors allow writing edid information back into it.

On most monitors that do support it, you need to enter "service mode", you will have to research this for your particular monitor.

Writing the edid file to the monitor can be dangerous, so I do not recommended it.

You can use the [edid-rw](https://github.com/bulletmark/edid-rw) tool.

Please read README and install the pre-requisite software on that page before using.

First you should try finding which number the edid is in.

`$ sudo ./edid-rw 0 | edid-decode`

Increment 0 until you see the edid.

Once you see it, you can write your custom edid file changing 0 for the number you found above.

`$ sudo ./edid-rw -w 0 < ~/edid_custom.bin`

#####Reboot and confirm refresh rate.

Reboot the computer.

`$ sudo reboot`

Confirm your refresh rate using xrandr.

`$ xrandr -q`

For example, I see 71.92 is my refresh rate, which confirms it worked.

Your monitor OSD might have the info also.

![xrandr_query](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/xrandr_query.png)

###Conclusion

As noted at the start of the guide, if you do not have a Nvidia or Windows PC to test different refresh rates, this can be painful, as you will need to continuously try editing the edid file until you find something that works.
