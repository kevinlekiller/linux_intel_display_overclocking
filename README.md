# linux_intel_monitor_overclocking
Guide for overclocking monitors on Linux with Intel GPUs.
--------

###Disclaimer

Use this information at your own risk. I do not claim responsability for any damage you might cause to your equipment, yourself or others.

###Intro

As you may already know, you can not overclock your monitor with xrandr with an Intel GPU (at least on Haswell or Broadwell GPU's I've tried).

This guide will try to help you overclock your monitor without xrandr directly.

It helps if you know the monitor can be overclocked to the frequency you desire before starting (using an Nvidia card for example, or a Windows operating system, since the Intel drivers on windows allow monitor overclocking, I'm not sure if AMD supports overclocking monitors on linux). Otherwise this might require considerable trial and error and be very frustrating.

###Software required:

* [xrandr](http://www.x.org/wiki/Projects/XRandR/) This should be already installed on GNU/Linux distros.
* [gnu coreutils](https://www.gnu.org/software/coreutils/coreutils.html) This should be already installed on GNU/Linux distros.
* [wine](https://www.winehq.org/) This will be used to install a Windows edid editor, since linux edid editors are uncommon.
* [AW Edid Editor](http://www.analogway.com/en/products/software-and-tools/aw-edid-editor/#dl) I will use this edid editor throughout the guide, it might be possible to use an alternative edid editor, although the guide might not be as simple to use. This edid editor installs without any issues in wine.  
* [QT](https://www.qt.io/) This might not be required, I did see some warnings about missing certain QT5 images when running AW Edid Editor.
* [gcc](https://gcc.gnu.org/) This is used to compile cvt12 from source.
* [cvt12](https://github.com/kevinlekiller/cvt_modeline_calculator_12) See below for instructions.

###Guide:

#####Compiling cvt12.

Optional reading: [Article](https://en.wikipedia.org/wiki/Coordinated_Video_Timings#Reduced_blanking) on wikipedia about reduced blanking.

We will compile a modified version of cvt to get access to CVT v1.2 reduced blanking timings. Most distributions use cvt from x.org which only supports CVT 1.1 currently, CVT 1.1 does not allow reduced blanking on any refresh rates other than multiples of 60hz.

Reduced blanking is for LCD monitors. If you use a CRT monitor, do not use reduced blanking.  
By using reduced blanking we can get a higher overclock when using HDMI 1.x, since HDMI 1.x has a pixel clock limit of about 165Mhz.

For example, without reduced blanking 72hz has about 210Mhz pixel clock. With reduced blanking, this drops to 160Mhz.

Download and compile cvt12:    
`$ cd ~/ && wget https://raw.githubusercontent.com/kevinlekiller/cvt_modeline_calculator_12/master/cvt12.c && gcc cvt12.c -O2 -o cvt12 -lm -Wall`

![cvt12](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/cvt12_compile.png)

#####Find your GPU.

`$ lspci | grep -Pi 'intel.*graphics'`

![GPU](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_GPU.png)

We are interested in the following: `00:02.0`

#####Find the port the monitor is connected to:

`$ xrandr | grep -Pio '.*?\sconnected'`

![port](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_port.png)

We are interested in: `HDMI1`

#####Find the edid file using the above information:

`$ sudo find /sys/devices/ -name edid`

![find_edid](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/find_edid.png)

We are interested in:  
`/sys/devices/pci0000:00/0000:00:02.0/drm/card0/card0-HDMI-A-1/edid`

Note the `00:02.0` and `HDMI-A-1`, which correspond to the information we got in the above steps.  
Remember the `HDMI-A-1`, which we will require later on.

#####Copy the edid to your home folder:

`$ sudo cp /sys/devices/pci0000:00/0000:00:02.0/drm/card0/card0-HDMI-A-1/edid ~/edid.bin`

#####Print the edid:

`$ cat ~/edid.bin && echo ""`

![print_edid](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/print_edid.png)

You will see mostly unintelligible text, you might see the monitor's model number.  
You can use this to confirm this is indeed the edid file for the monitor.  
If not, we can confirm in one of the following steps.

#####Get a new modeline using cvt12.

The parameters are: screen\_width screen\_height refresh_rate.

-b uses CVT v1.2 reduced blanking timings - remove this if you use a CRT monitor.  
If your LCD is old, you might have to use -r instead, which uses CVT v1.1 reduced blanking timings.  
If it is VERY old (10+ years), then do not use -r or -b, as most of those old LCD's do not support reduces blanking.

Be aware that **HDMI 1.x has a limit of 165Mhz pixel clock**, make sure you use displayport if you need to go over 165MHz, or try underclocking to 48hz if your goal is to watch movies / tv.

If you want to get a refresh rate that works well for watching movies or tv, pass the -o argument.  
The -o argument will for example, convert 72hz to 71.928hz, so when you watch 23.976fps content it will play smoothly.

`$ cd ~/ && ./cvt12 1920 1080 72 -b`  

![cvt_readout](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/cvt_readout.png)

We are looking for this: `Modeline "1920x1080_72.00_rb"  167.28  1920 1968 2000 2080  1080 1103 1108 1117 +hsync -vsync`

Strip anything after `"1920x1080` in the quotes, so it looks like this: `Modeline "1920x1080"  167.28  1920 1968 2000 2080  1080 1103 1108 1117 +hsync -vsync`

#####Convert the modeline into timings.

Head to [this page](http://www.epanorama.net/faq/vga2rgb/calc.html)

If that webpage is down, I've backed it up [here](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/backups/Video%20timing%20calculator.html), download it (right-click save as) and open it in your browser.

Paste in the modeline from the above step into the box over the `Import Modeline` button.

Click `Import Modeline`.

Keep this web page open, we will need it in the next steps.

![video_timing_calculator](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/video_timing_calculator.png)

#####Open the edid file in AW Edid Editor.

You can start the program and open the file manually or start it from the shell with the file as a parameter:  

`$ wine ~/.wine/drive_c/Program\ Files\ \(x86\)/ANALOG\ WAY/AW\ EDID\ Editor/AWEDIDEditor.exe ~/edid.bin`

You might be able to confirm this is the edid for your monitor by using the information on the top left of the GUI in the Standard Data tab or in the Detailed Data tab, look for Display Product Name.

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

Save the file _**by doing File->Save As, NOT FILE->SAVE, File->Save does not function correctly in wine**_. Call the new file edid_custom.bin and exit AW Edid Editor.

![detailed_timings](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/detailed_timings.png)

#####Make linux use the modified edid file on boot.

You can make linux use the modified edid file when starting.

Copy the edid file to an appropriate folder:

`$ sudo mkdir -p /usr/lib/firmware/edid/ && sudo cp ~/edid_custom.bin /usr/lib/firmware/edid/edid_custom.bin`

Note: Change `HDMI-A-1` for what we found in an earlier step.

If you don't use grub, see [this page](https://wiki.archlinux.org/index.php/Kernel_parameters), you need to add `drm_kms_helper.edid_firmware=HDMI-A-1:edid/edid_custom.bin` to your kernel parameter.

With grub you can do it the following way (change nano to the editor of your choice):

`$ sudo nano /etc/default/grub`

Look for a line similar to this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`

Add the following to the end: `drm_kms_helper.edid_firmware=HDMI-A-1:edid/edid_custom.bin`

It should now look like this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash drm_kms_helper.edid_firmware=HDMI-A-1:edid/edid_custom.bin"`

![grub_cfg](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/grub_cfg.png)

Now you can regenerate your grub.cfg file:

`$ sudo grub-mkconfig -o /boot/grub/grub.cfg`

![grub_refresh](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/grub_refresh.png)

#####(Optional) Write the edid to the monitor.

This step is not recommended. Most people reading this should skip to the next step.

Some monitors allow writing edid information back into it.  
The advantage to this is it will survive operating system re-installations, since the information will be inside the monitor.

On most monitors that do support it, you need to enter "service mode", you will have to research this for your particular monitor. Some also require the monitor to be not displaying anything.

Writing the edid file to the monitor can be dangerous, be forewarned.

You can use the [edid-rw](https://github.com/bulletmark/edid-rw) tool.

Please read README and install the pre-requisite software on that page before using.

First you should try finding which number the edid is in.

`$ sudo ./edid-rw 0 | edid-decode`

Increment 0 until you see the edid data that corresponds to your monitor.

Once you see it, you can write your custom edid file changing 0 for the number you found above.

`$ sudo ./edid-rw -w 0 < ~/edid_custom.bin`

You can verify it was written by reading it again:

`$ sudo ./edid-rw 0 | edid-decode`

#####Reboot and confirm refresh rate.

Reboot the computer. You can do this in the GUI or in the shell like this:

`$ sudo reboot`

Confirm your refresh rate using xrandr.

`$ xrandr -q`

For example, I see 71.92 is my refresh rate, which confirms it worked (I did not use the timings above, hence why you don't see 72hz).

Your monitor OSD might have the info also.

![xrandr_query](https://raw.githubusercontent.com/kevinlekiller/linux_intel_monitor_overclocking/images/xrandr_query.png)

###Conclusion

As noted at the start of the guide, if you do not have a Nvidia or Windows PC to test different refresh rates, this can be painful, as you will need to continuously try editing the edid file until you find something that works.

Hopefully this guide has helped you in some way.
