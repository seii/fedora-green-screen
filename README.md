# Setting up a "GREEN SCREEN" for live video in Fedora
Have you ever seen a "GREEN SCREEN" and wondered if you could use that effect yourself? I had that thought for a long time, and always thought it was complicated, or required special colours or backgrounds. In truth, it's actually so simple, I'm now using the idea with my laptop webcam. This guide will be written for Linux, and the Fedora distribution in particular, but much of the approach should work across Linux distributions with minor command adaptations.

### Pre-requisites
- Fedora 30 or higher (lower versions might work, but I didn't test any)
- An internet connection
- Root access to your Linux installation (via `sudo` or otherwise)
- Some kind of video capture tool installed on / attached to your Linux system (I use a webcam)
- Basic working knowledge of how to clone Git repositories

### Colours Preparation
1. Pick a colour that will be your chroma key. I recommend a colour that you don't typically wear or otherwise use in your shots... unless you want your clothes or other objects to be the "screen"!
1. Take a picture or screenshot of that colour with the video device you have chosen. This will help to ensure that, whatever the colour actually is, you can verify what your video device will "see".
1. Use any graphics editor with a colour selection tool ([GIMP](https://www.gimp.org/) works well) to load the screenshot you have taken
1. Use the colour selection tool in your editor to get the RGB or HTML representation of that colour (ideally, get both!)

### Installing OBS Studio
1. First, you will want to install [Open Broadcaster Software Studio](https://obsproject.com/download) (OBS Studio). How to do this varies across operating systems and even between Linux distributions, but for Fedora we'll use the RPM Fusion "free" repository.
1. In a terminal, enter the following command to add the RPM Fusion "free" repository to your system: `sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm` <sup id="add-free-repo">[1](#footnote1)</sup>
1. In a terminal, run `sudo dnf update` to get the latest from the RPM Fusion "free" repository
1. In a terminal, run `sudo dnf install obs-studio` to install OBS Studio

At this point, you should have a working install of OBS Studio. Yay! If your only goal was to save the output of your green screen to a file, or to livestream it to one of the various internet sites that OBS Studio already supports, then jump straight to [OBS Studio Chroma Key Setup](#obs-studio-chroma-key-setup). However, if you want to direct your chroma-key'd output into another program that only accepts video devices, you will need a "loopback" function. Luckily, someone has already created the necessary software: [v4l2loopback](https://github.com/umlaeute/v4l2loopback).

### Install Kernel Headers
v4l2loopback is a module for the Linux kernel. In order to use it, you will build it and add it as a Linux kernel module. There's nothing wrong with that, and it's not a scary process. However, it does require the development headers for the kernel to be installed, which your system may or may not have setup already.

1. In a terminal, run the command `uname -a` to see your currently running kernel version. (It will look something like `5.0.17-300.fc30.x86_64`)
1. In a terminal, run the command `dnf list --installed kernel-devel` to see if you have kernel headers installed
1. If the `kernel-devel` package is not installed, run `sudo dnf install kernel-devel` to install it

### Installing v4l2loopback
1. Create a directory for cloning repositories from GitHub (called `$GIT_DIR` from here onwards)
1. Change into the `$GIT_DIR` using `cd`
1. Clone the [v4l2loopback](https://github.com/umlaeute/v4l2loopback) repository
1. Change into the v4l2loopback directory
1. Inside the v4l2loopback directory, run the command `make`. This should succeed without needing to use `sudo`. If there are any errors about dependencies (there shouldn't be), you can search for the dependency using `dnf search $DEPENDENCY_NAME` and install it
1. Once `make` succeeds, run `sudo make install && sudo depmod -a` to install the module and update the kernel's dependencies to be aware of it
1. In order to load the module, run the command `sudo modprobe v4l2loopback`. This should succeed (i.e. no output is generated)
1. In order to test that the module has been loaded, run the command `lsmod | grep v4l2loopback`. The command should have output, displaying the module
1. By default v4l2loopback would be installed and available, but wouldn't be loaded by the system after a reboot. In order to change this, run the command `echo 'v4l2loopback' | sudo tee -a /etc/modules-load.d/v4l2loopback.conf`

### (Optional) Automatically Rebuild v4l2loopback When Kernel Changes
Kernel modules are built against one specific version of the kernel - the one which was running when they were built. However, especially with Fedora and other fast-moving distros, the kernel version may change quite often as newer updates are installed. In order to keep up with this, we can use Dynamic Kernel Module Support, or [DKMS](https://en.wikipedia.org/wiki/Dynamic_Kernel_Module_Support). DKMS will automatically rebuild the v4l2loopback module against a new kernel version whenever one is installed.

1. Navigate to the directory where you cloned v4l2loopback in a terminal
1. In order to verify the version of v4l2loopback, run the command `cat dkms.conf | grep PACKAGE_VERSION` (the version was `0.12.2` when this guide was written). Remember this version for the next step
1. Run the command `sudo cp -R ./ /usr/src/v4l2loopback-$PACKAGE_VERSION/`, inserting the version from the above step instead of `$PACKAGE_VERSION`. (Example: `sudo cp -R ./ /usr/src/v4l2loopback-0.12.2/`)
1. Run the command `sudo dkms add v4l2loopback/$PACKAGE_VERSION`. (Example: `sudo dkms add v4l2loopback/0.12.2`)

### Installing obs-v4l2sink
We now have a way to create loopback video devices in the Linux kernel, but OBS Studio doesn't know how to use them. For that, we will turn to another Github repository: [obs-v4l2sink](https://github.com/CatxFish/obs-v4l2sink). obs-v4l2sink needs to be built against the source of OBS Studio.

1. Navigate to $GIT_DIR and clone the [obs-v4l2sink](https://github.com/CatxFish/obs-v4l2sink) repository
1. Clone the [obsstudio](https://github.com/obsproject/obs-studio) repository using the command `git clone --recursive https://github.com/obsproject/obs-studio.git`
1. Make sure that the `qt` development library is installed by running the command `sudo dnf install qt5-qtbase-devel`
1. Navigate to the `obs-v4l2sink` directory
1. Prepare to make the plugin by running the command `cmake -D LIBOBS_INCLUDE_DIR="../obs-studio/cmake" -D LIBOBS_LIB=/usr/lib64/libobs.so.0" -D CMAKE_INSTALL_PREFIX=/usr .`
1. Actually make it by running the command `make -j4`
1. Install the plugin by running the command `sudo make install`
1. The plugin has been installed to `/usr/lib/obs-plugins`, but Fedora uses the location `/usr/lib64/obs-plugins`. Fix this with a soft link by running the command `sudo ln -s /usr/lib/obs-plugins/v4l2sink.so /usr/lib64/obs-plugins/v4l2sink.so`
1. To verify that the plugin is installed, open OBS Studio and select the "Tools" menu. If there is an entry for "V4L2 Video Output", the plugin is installed successfully

### OBS Studio Chroma Key Setup
OBS Studio should already have detected your video capture device, and if it's active you should be seeing yourself (or whatever the device is pointed at) as soon as you launch the program. Now, we need to add chroma keying as a filter to the video device's input.

1. Launch OBS Studio
1. In the "Sources" box near the lower left corner, right-click your video capture device and select the "Filters" option
1. In the bottom left of the Filters screen is an "Effect Filters" box. Click the "+" button and select "Chroma Key"
1. Select the new "Chroma Key" option in Effect Filters, if it isn't selected already
1. Next to the "Key Colour" field is a button named "Select colour". Click it
1. In the new "Key Colour" popup, enter the colour (in RGB or HTML values) that you collected in the [Colours Preparation](#colour-preparation) step. Click "OK"
1. The area which matches this colour may now be a different colour, or display artifacts or "cloudiness". For now, ignore this and click "Close" on the "Filters" dialog
1. In the "Sources" box, click the "+" and select "Image"
1. In the "Create/Select Source" dialog, select "Create new" and title it "Test Image". Click "OK"
1. In the "Sources" box, you should now have your video capture device and your "Test Image". If things have gone smoothly, you will see your selected test image instead of the areas which match the chroma key colour you have already set up
1. If the chroma keying is rough or doesn't match as you'd like, right-click on the video source and once again select "Filters" > "Effect Filters" > "Chroma Key". From here you can change the settings in real time until they match what you want

### Streaming To v4l2loopback
Now that everything is in place, we can use OBS Studio and the "obs-v4l2sink" plugin to make our computer think we have another video device - a video device which happens to be displaying our chroma-key'd input.

1. In OBS Studio, select "Tools" > "V4L2 Video Output"
1. The "Path to V4L2 Device" field should be auto-populated. On my system this value was correct, but if you have multiple `/dev/video*` devices you may need to change it for best results
1. Click "Start"
1. In the program you wish to use your chroma keying for, select the fake device (It was called "Dummy video device" on my system)
1. Have fun streaming! (Don't forget to "Stop" the device in OBS Studio when you're done)


##### Footnote:
<b id="footnote1">[1]</b> [RPM Fusion: Command Line Setup using rpm](https://rpmfusion.org/Configuration) [↩](#add-free-repo)
