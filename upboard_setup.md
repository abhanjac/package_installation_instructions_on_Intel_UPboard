# Ubuntu 16.04 Install on Upboard
* References:
  * [Intel UP Board](https://click.intel.com/aaeon-up-board.html)
  * [UP Community Wiki](https://wiki.up-community.org/Ubuntu)
  
# Prepare Bootable USB Drive
1. Format USB drive (10 - 15 minutes)
* Insert USB drive into Laptop/PC
* Erase & Format USB drive
  * **Windows 10**
    * Right-click USB drive and select **Format**
  * **Mac**
    * Open **Disk Utility**
    * Select the USB drive and click **Erase**
    * Set new format to ExFAT and leave other options as defaults
* Navigate to the [Ubuntu Old-Releases Archive](http://old-releases.ubuntu.com/releases/16.04.3/)
  * Download [ubuntu-16.04.3-desktop-amd64.iso](http://old-releases.ubuntu.com/releases/16.04.3/ubuntu-16.04.3-desktop-amd64.iso)
  * Download and install [Etcher](https://etcher.io/)
  * Insert USB Drive
  * Open Etcher
  * Click **Select Image** and browse to the ***.iso** downloaded above
  * Select the USB Drive
  * Click **Flash**

## Install Ubuntu on UP Board
  [Partitioning Reference](https://askubuntu.com/questions/343268/how-to-use-manual-partitioning-during-installation)
1. Plug Bootable USB Drive into UP Board and power it up
  * The following devices will need to be connected to the UP Board:
    * Keyboard, Mouse, Monitor, Ethernet cable (internet access)
  * If Ubuntu is already installed, hold F7 during power up and select the USB drive as the boot device
    * e.g. `UEFI: hp v220w 1100, Partition 1`
* Select `Install Ubuntu` after the **"UP bridge the gap"** splash screen
* Welcome screen
  * Select `English` and Continue
* Preparing to Install Ubuntu:
  * Select `Install third-party software for graphics and Wi-Fi hardware, Flash, MP3 and other media`
  * DO NOT Select `Download updates while installing Ubuntu`
* Popup: "Force UEFI Installation?"
  * Select "Continue in UEFI mode" to overwrite the existing "BIOS Compatibility mode"
* Installation Type
  * Select "Something else"
  * Click "New Partition Table.."
    * Select "free space" under /dev/mmcblk0
    * Create EFI System Partition
      * Click `+` to create a partition
        * Size: `200 MB`
        * Type: `Primary`
        * Location: `Beginning of this space`
        * Use as: `EFI System Partition`
        * Click `OK`
    * Select `free space` at bottom of `/dev/mmcblk0` (~31072 MB)
    * Create swap partition
      * Click `+` to create a partition
        * Size: `1024 MB`
        * Type: `Primary`
        * Location: `Beginning of this space`
        * Use as: `swap area`
        * Click `OK`
    * Select `free space` at bottom of `/dev/mmcblk0` (~30047 MB)
    * Create root partition
      * Click `+` to create a partition
        * Size: `30046` (pre-populated, default)
        * Type: `Logical`
        * Location: `Beginning of this space`
        * Use as: `Ext4 journaling file system` (default)
        * Mount point: `/`
        * Click `OK`
    * Device for boot loader installation: `/dev/mmcblk0`
    * Click `Install Now` then `Continue`
* Where are you?
  * Select timezone: `Indianapolis`
* Keyboard Layout: keep defaults (English)
* Who are you? 
  * name: `upboard`
  * computer's name: `upboard-UP-CHT01`
  * username: `upboard`
  * password: `upboard`
  * Select `Require my password to log in`
* Click `Install Now`
* Possible Issues:
  * Error: 'grup-efi-amd64-signed' package failed to install into / target/. Without the GRUB boot loader, the installed system will not boot.
    * Clicked OK
    * Installer crashed
    * [Bug report](https://bugs.launchpad.net/ubuntu/+source/ubiquity/+bug/1766945)
      * Explains need for `EFI System Partition`
  * System RESTART
    * ```
      efi: requested map not found.
      ESRT header is not in the memory map
      ```
  * Power OFF (and disconnect HDMI) then restart and try again

## Install Ubuntu kernel for UP from PPA
1. Add repository & update
  * `sudo add-apt-repository ppa:ubilinux/up`
  * `sudo apt update`
* Remove all generic installed kernels
  * `sudo apt autoremove --purge 'linux-.*generic'`
  * **IMPORTANT NOTE:** A prompt will appear asking to `Abort kernel removal?`
    * Select: `<No>`
    * Make sure to Install the UP kernel on the following line before rebooting!
* Install custom UP kernel (with headers)
  * ```
    sudo apt install linux-image-generic-hwe-16.04-upboard
    sudo apt install linux-headers-generic-hwe-16.04-upboard 
    ```
* Install updates and reboot (do NOT upgrade to Ubuntu 18.04)
  * `sudo apt dist-upgrade -y`
  * `sudo reboot`
  
## Set up Basic Applications/Environment
1. Enable root login
  * `sudo passwd root`
  * Enter new UNIX password: `upboard`
* Add user to dialout group
  * `sudo usermod -a -G dialout upboard`
  * `sudo reboot`
  
* Install basic applications
  * ```
  sudo apt update
  sudo apt install openssh-server git cmake kate chrony screen unzip tightvncserver v4l-utils htop curl
  ```
* Configure **Chrony**
  * This application will synchronize the clocks between multiple machines on the same network. This is necessary since the Time class in ROS uses the system clock for timing, and the system clock may differ between two machines, even if they are both using the same ntp server. Thus making it impossible to accurately compare timestamps between topics sent from different machines.
  * References: [1](https://chrony.tuxfamily.org/documentation.html), [2](https://wiki.archlinux.org/index.php/Chrony), [3](http://cmumrsdproject.wikispaces.com/file/view/ROS_MultipleMachines_Wiki_namank_fi_dshroff.pdf), [4](http://wiki.ros.org/ROS/NetworkSetup), [5](https://github.com/pr2-debs/pr2/blob/master/pr2-chrony/root/unionfs/overlay/etc/chrony/chrony.conf), [6](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-16-04)
  * To check the time difference between machines, use:
    * `ntpdate -q <IP address of other machine>` 
  * Turn off **timesyncd**
    * `sudo timedatectl set-ntp no` 
  * Configure Chrony (make these changes in `/etc/chrony/chrony.conf`)
    * Add Purdue’s ntp servers
      * ```
      sudo sed -i 's/pool 2.debian.pool.ntp.org offline iburst$/pool 2.debian.pool.ntp.org offline iburst\nserver ntp3.itap.purdue.edu offline iburst\nserver ntp4.itap.purdue.edu offline iburst/' /etc/chrony/chrony.conf
      ```
  * Set up chrony server for other devices
    * ```
    sudo sed -i 's/#local stratum 10$/local stratum 10/' /etc/chrony/chrony.conf
    sudo sed -i 's~#allow ::/0 (allow access by any IPv6 node)$~#allow ::/0 (allow access by any IPv6 node)\nallow 192.168~' /etc/chrony/chrony.conf
    ```
  * Allow for step time changes at startup (for initial time setup)
    * ```
    su
    echo -e “# In first three updates step the system clock instead of slew\n# if the adjustment is larger than 10 seconds\nmakestep 10 3” | tee -a /etc/chrony/chrony.conf > /dev/null
    exit
    ```
  * Set timezone
    * Get the exact name of the desired timezone (`America/Indiana/Indianapolis`)
      * `timedatectl list-timezones`
    * `sudo timedatectl set-timezone America/Indiana/Indianapolis`
    * `date`
  * Restart chrony to apply changes
    * `sudo systemctl restart chrony`
* Configure **Kate**
  * Top Menu --> Settings --> Configure Kate
    * Application
      * Sessions --> Check `Load last used session`
    * Editor Component
      * Appearance --> Borders --> Check `Show line numbers`
      * Fonts & Colors --> Font --> `Size: 10`
      * Editing --> General --> Check `Show static word wrap marker`
      * Editing --> General --> Check `Enable automatic brackets`
      * Editing --> Indentation --> `Tab width: 4 characters`
      * Editing --> Indentation --> `Keep extra spaces`
      * Open/Save --> Modes & Filetypes
        * Select Filetype: `Markup/XML`
        * Add `;*.launch` to 'File extensions'
          * NOTE: Be careful not to add a space after the `;`
  * Click `Apply` then `OK`
  * Set Kate as default file editor (instead of GEdit)
    * ```
    sudo sed -i 's/gedit.desktop/kate.desktop/g' /etc/gnome/defaults.list
    sudo cp /usr/share/applications/gedit.desktop /usr/share/applications/kate.desktop
    sudo sed -i 's/gedit/kate/g' /usr/share/applications/kate.desktop
    sudo sed -i 's/=Text\ Editor/=Kate/g' /usr/share/applications/kate.desktop
    sudo reboot (to apply changes)
    ```
* Configure **Terminal**
  * Top Menu --> Edit --> Profile Preferences
    * General --> Initial terminal size: **80 x 20**
    * Scrolling --> Check **Unlimited**
* Configure **File Manager**
  * Open File Browser, then select Edit --> Preferences
    * Views --> View new folders using: **List View**
    * Views --> List View Defaults: zoom level: `33%`
* Configure window size of **Compiz File Browser**
  * `cd ~/.config/compiz-1/compizconfig`
  * Add **size** setting in Compiz config file
    * `nano config`
    * Add `size = 800, 100` at bottom of file

## Set up XFCE Light-weight Desktop Environment
1. Install XFCE (needed for TightVNC to work)
  * `sudo apt install xfce4`
* To switch environments
  * Log out of Ubuntu
    * Select `Log Out` from Setting menu (top right corner of screen)
  * Click Ubuntu icon at right of username and select `XFCE Session`
  * Log in
* To Uninstall:
  * sudo apt purge xfce4
  * sudo apt autoremove

## Configure Autologin and VNC server
1. **Automatic Login**
  * Add the following lines to `/etc/lightdm/lightdm.conf`
  ```
  [SeatDefaults]
  autologin-user=upboard
  autologin-user-timeout=0
  ```
* Setup **TightVNCServer**
  * References: [1](https://ubuntu-mate.community/t/accessing-rpi2-remotely/3074/7), [2](https://askubuntu.com/questions/506580/how-to-configure-a-password-for-lightdm-vnc-connection)
  * Set up the vnc password: `vncpasswd`
    * Password: **upboard**
    * View-only password: **No**
  * Create Xresources file
    * touch ~/.Xresources
  * Configure the VNC server to work with Ubuntu XFCE desktop
    * Make a backup copy of `~/.vnc/xstartup` as `~/.vnc/xstartup.bak`
    * Paste the following into `~/.vnc/xstartup`
      * 
      ```
        #!/bin/sh
        # ONLY use these "unset" lines if NOT using shell function "tvnc()"
        #unset SESSION_MANAGER
        #unset DBUS_SESSION_BUS_ADDRESS
        
        xrdb $HOME/.Xresources
        xsetroot -solid grey
        export XKL_XMODMAP_DISABLE=1
        startxfce4 &
      ```
    * Create shell function `tvnc()` to start vncserver
      * Add the function to `~/.bashrc`
      * ```
        echo -e "\n# Start TightVNC Server\nfunction tvnc() {\
        \n  env -u SESSION_MANAGER -u DBUS_SESSION_BUS_ADDRESS vncserver\n}"\
        >> ~/.bashrc 
        source ~/.bashrc
        ```
  * To run the server, type: `tvnc` in the shell
    * This will output:
    `New 'X' desktop is upboard-UP-CHT01:1`
    * Calling `tvnc` again will start new servers at `:2`, `:3`, etc.
    * View server at `<IP Address>:5901`
      * **NOTE:** `5901` is for `:1`, `5902` for `:2`, etc.
    * [Viewer application (Java)](https://www.tightvnc.com/download/2.8.3/tvnjviewer-2.8.3-bin-gnugpl.zip)
  * To kill the server: `vncserver -kill :1`
    * **NOTE:** `:1` above indicates index number (could be `:1`, `:2`...etc.)
* Disable suspend
  * System Settings (Top right corner) --> Brightness & Lock:
    * Turn screen off when inactive for: `Never`
    * Lock screen after: `10 minutes`
    * UNCHECK `Require my password when waking from suspend`
    
## Install Python Packages
1. [**Pip**](https://pip.pypa.io/en/stable/installing/)
  * ```
  curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
  sudo python get-pip.py
  ```
* **PySerial** (for Python access to serial ports)
  * `sudo python -m pip install pyserial`

## Install Driver for Netgear AC600 (A6100) USB-WiFi adapter
1. Download the driver
  * ```
  cd ~ 
  git clone https://github.com/scrivy/rtl8812AU_8821AU_linux.git
  cd rtl8812AU_8821AU_linux
  ```
* Build and install
  * ```
  make -j4
  sudo make install 
  sudo modprobe rtl8812au 
  ```
* Test Driver
  * Plug in WiFi adapter
  * Select LabRouter5 from Network dropdown and connect
  
## Install Driver for Netgear N150 (WNA1100) USB-WiFi adapter
  * [Reference](https://wiki.debian.org/ath9k_htc)
  
## Install Kinect Driver (OpenKinect/libfreenect)
1. Install the driver: `sudo apt install freenect`
* Download the udev rules
  ```
  cd /etc/udev/rules.d/
  sudo wget https://github.com/OpenKinect/libfreenect/blob/master/platform/linux/udev/51-kinect.rules
  ```
* OPTION 2: Download & Install SensorKinect Source
  ```
  cd ~/
  git clone https://github.com/OpenKinect/libfreenect
  cd libfreenect
  mkdir build
  cd build
  cmake -L -DCMAKE_BUILD_PYTHON=ON ..
  make -j4
  sudo make install
  sudo ldconfig /usr/local/lib
  ```

## Install OpenCV 3.4 (~1 hr)
  * Reference: [OpenCV Docs](https://docs.opencv.org/3.4.3/d7/d9f/tutorial_linux_install.html)
  * Full list of [CMake Options](https://github.com/opencv/opencv/blob/master/CMakeLists.txt#L198)

1. Install Dependencies
  ```
  sudo apt install build-essential cmake git libgtk2.0-dev pkg-config libavcodec-dev \
       libavformat-dev libswscale-dev python-dev python-numpy libtbb2 libtbb-dev \
       libjpeg-dev libjasper-dev libdc1394-22-dev checkinstall yasm libxine2-dev \
       libv4l-dev libav-tools unzip libdc1394-22 libpng12-dev libtiff5-dev tar dtrx \
       libopencore-amrnb-dev libopencore-amrwb-dev libtheora-dev libvorbis-dev \
       libxvidcore-dev x264 libgstreamer0.10-dev libgstreamer-plugins-base0.10-dev \
       libqt4-dev libmp3lame-dev
  ```
* Download OpenCV and OpenCV contrib source code
  ```
  cd ~
  mkdir OpenCV
  cd OpenCV
  git clone -b 3.4 --single-branch https://github.com/opencv/opencv.git
  git clone -b 3.4 --single-branch https://github.com/opencv/opencv_contrib.git
  ```
  
* Build from source (~40 minutes)
  ```
  cd ~/OpenCV/opencv
  mkdir release
  cd release
  sudo cmake -D CMAKE_BUILD_TYPE=RELEASE -D BUILD_NEW_PYTHON_SUPPORT=ON -D INSTALL_C_EXAMPLES=ON -D INSTALL_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_GTK=ON -D WITH_QT=ON -D WITH_GSTREAMER_0_10=ON -D WITH_TBB=ON -D BUILD_TBB=ON -D WITH_OPENGL-ES=ON -D WITH_V4L=ON -D CMAKE_INSTALL_PREFIX=/usr/local -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules ..
  sudo make -j4
  sudo make install
  ```
* Create config file (if it doesn't exist)
  * `sudo nano /etc/ld.so.conf.d/opencv.conf`
  * Add `/usr/local/lib/` to the file
  * `sudo ldconfig`
* Test installation with Python
  ```
  python
  import cv2
  recognizer = cv2.face.LBPHFaceRecognizer_create()
  detector = cv2.CascadeClassifier("haarcascade_frontalface_default.xml")
  ```
* Remove OpenCV source to free up storage
  * `sudo rm -r ~/OpenCV`

## Install ROS Kinetic (~20 minutes)
  * Reference: [ROS Wiki](http://wiki.ros.org/kinetic/Installation/Ubuntu)

1. Set up
  ```
  sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
  sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key 421C365BD9FF1F717815A3895523BAEEB01FA116
  sudo apt update
  ```
  
* Install ROS desktop version and additional packages
  ```
  sudo apt install ros-kinetic-desktop-full
  sudo apt install python-rosdep
  sudo rosdep init
  rosdep update
  sudo chown -R upboard:upboard /opt/ros
  sudo apt install python-rosinstall ros-kinetic-mavros ros-kinetic-vision-opencv ros-kinetic-image-pipeline
  sudo apt autoremove
  ```
  
* Create Catkin workspace
  ```
  source /opt/ros/kinetic/setup.bash
  mkdir -p ~/ros_ws/src
  cd ~/ros_ws/
  catkin_make
  ```
  
* Set up ROS Environment (auto-load in bashrc)
  ```
  su
  echo "# ROS Settings" | tee -a /home/upboard/.bashrc > /dev/null
  echo "source /opt/ros/kinetic/setup.bash" | tee -a /home/upboard/.bashrc > /dev/null
  echo "source /home/upboard/ros_ws/devel/setup.bash" | tee -a /home/upboard/.bashrc > /dev/null
  echo "export ROS_MASTER_URI=http://localhost:11311" | tee -a /home/upboard/.bashrc > /dev/null
  echo "#export ROS_MASTER_URI=http://192.168.1.88:11311" | tee -a /home/upboard/.bashrc > /dev/null
  echo "#export ROS_IP=192.168.1.106" | tee -a /home/upboard/.bashrc > /dev/null
  exit
  ```
  
* Set up for Turtlebot 2
  * Reference: [Turtlebot ROS Wiki](http://wiki.ros.org/turtlebot/Tutorials/indigo/Turtlebot%20Installation)
  * Install Turtlebot packages
    ```
    sudo apt install ros-kinetic-turtlebot ros-kinetic-turtlebot-apps \
      ros-kinetic-turtlebot-interactions ros-kinetic-turtlebot-simulator \
      ros-kinetic-kobuki-ftdi ros-kinetic-ar-track-alvar-msgs
    ```
  * Prepare **udev rules** for the Kobuki base
    ```
    . /opt/ros/kinetic/setup.bash
    rosrun kobuki_ftdi create_udev_rules
    ```
    
  * Identify the Kinect (or R200) as the 3D sensor
    * Add the following ABOVE the **"source .../setup.bash"** lines in **~/.bashrc**
    * ```
      export TURTLEBOT_3D_SENSOR=kinect
      #export TURTLEBOT_3D_SENSOR=r200
      ```
      
* Set up for 3D Mapping and Path-Planning
  * Install [OctoMap](https://www.ros.org/wiki/octomap_server)
    * `sudo apt install ros-kinetic-octomap-server`
  * Install [rtabmap_ros](http://wiki.ros.org/rtabmap_ros)
    * `sudo apt install ros-kinetic-rtabmap-ros`

* Create a custom package
  * ```
  cd ~/ros_ws/src/
  catkin_create_pkg my_package rospy roscpp std_msgs geometry_msgs sensor_msgs cv_bridge 
  ```
* Build only one or more specified package(s)
  * List packages to build, separated by ';'
    * `catkin_make -DCATKIN_WHITELIST_PACKAGES="pkg1;pkg2"`
      
## Install librealsense (for Intel RealSense D435)
  * Reference: [librealsense GitHub](https://github.com/IntelRealSense/librealsense/blob/v1.12.1/doc/installation.md)
  * Note: To make the D435 work, the **librealsense** driver and the **pyrealsense** python package must both be installed.

1. Install Dependencies
```
sudo apt update
sudo apt install libssl-dev libusb-1.0-0-dev pkg-config libgtk-3-dev libglfw3-dev cython
```
* Update CMake to latest version
  * [Download](https://cmake.org/download/)
    * [cmake-3.12.3.tar.gz](https://cmake.org/files/v3.12/cmake-3.12.3.tar.gz)
  * [Install](https://cmake.org/install/)
    * ```
    tar -xzvf cmake-3.12.3.tar.gz
    cd cmake-3.12.0
    ./bootstrap -- -DCMAKE_BUILD_TYPE:STRING=Release
    make -j4
    sudo make install
    ```
* Download source code from Github
  * Option 1
  ```
  cd ~
  git clone https://github.com/IntelRealSense/librealsense.git
  cd librealsense
  git checkout tags/v2.16.1
  ```
  * Option 2
    * Copy `./librealsense-2.16.1` folder from SPIRA server
    * `cd ~/librealsense-2.16.1`

* Add the **udev rules** for the RealSense
```
sudo cp config/99-realsense-libusb.rules /etc/udev/rules.d/ 
sudo udevadm control --reload-rules && udevadm trigger 
```
* Apply kernel patch
  * Modify the following file: `./scripts/patch-realsense-ubuntu-lts.sh`
    * Comment out line 23: `#sudo apt-get install linux-headers-generic build-essential git`
    * **IMPORTANT:** the appropriate linux headers for UP board should have already been installed, see step 3 under **Install Ubuntu kernel for UP from PPA** above
  * Run the script
    * `./scripts/patch-realsense-ubuntu-lts.sh`
    * **NOTE**: If a `Permission denied` error occurs, may need to run `sudo chmod +x scripts/patch-realsense-ubuntu-lts.sh`
  
* Build and Install
```
mkdir build
cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=true -DBUILD_GRAPHICAL_EXAMPLES=false -DBUILD_PYTHON_BINDINGS=true -DPYTHON_EXECUTABLE=/usr/bin/python2.7
make -j4
sudo make install 
```
  * **NOTE:** to uninstall, use: `sudo make uninstall && make clean`
  
* Update the **PYTHONPATH** env. variable (add path to pyrealsense lib)
  * Add the following line to **~/.bashrc**
  * `export PYTHONPATH=$PYTHONPATH:/usr/local/lib`
  
* Verify install 
  * Plug in D435 camera
  * dmesg log should indicate a new uvcvideo driver has been registered
  * `sudo dmesg | tail -n 50`

* Test **pyrealsense** Python package (built with librealsense)
  * ```
  cd ~/librealsense/wrappers/python/examples
  python opencv_viewer_example.py
  ```
  * Note: If the program crashes or the pipeline fails, try replugging the D435

## Install RealSense ROS Package
* Download source from Github
  * ```
  cd ~/ros_ws/src
  git clone https://github.com/intel-ros/realsense.git
  cd realsense
  git checkout tags/2.1.0
  cd ~/ros_ws
  catkin_make install
  ```
  
  * NOTE: if the system gets hung up, try to reduce RAM usage with:
    * `catkin_make -j2 install`
    
## Update D435 firmware using Intel DFU Tool (As Needed)
  * [Instructions](https://www.intel.com/content/dam/support/us/en/documents/emerging-technologies/intel-realsense-technology/Linux-RealSense-D400-DFU-Guide.pdf)
  
1. Add Intel server to list of repositories:
  * `echo 'deb http://realsense-hw-public.s3.amazonaws.com/Debian/apt-repo xenial main' | sudo tee /etc/apt/sources.list.d/realsensepublic.list`
* Register the servers public key:
  * `sudo apt-key adv --keyserver keys.gnupg.net --recv-key 6F3EFCDE`
* Refresh the list of repositories and packages available:
  * `sudo apt update`
* Install the intel-realsense-dfu package:
  * `sudo apt-get install intel-realsense-dfu*`
* Download latest D400 series firmware .bin file (Signed Binary):
  * [Click on this link:](https://downloadcenter.intel.com/download/27522/Latest-Firmwarefor-Intel-RealSense-D400-Product-Family?v=t)
    * https://downloadcenter.intel.com/download/27522/Latest-Firmwarefor-Intel-RealSense-D400-Product-Family?v=t
* Plug in D400 Series camera to host USB3.1 port. Check serial # and bus#. type:
  * Type: `lsusb`
  * Notice “Intel Corp.” bus and device numbers; DFU tool uses these
values to identify Intel® RealSense™ D400 series camera. 
* Upgrade D400 Series Camera Firmware with Linux DFU Tool:
  * Type command:
    * (This command specifies bus #, device #, -f flag to force upgrade, and –i flag for complete system path to downloaded FW.bin file.)
    * `intel-realsense-dfu –b 002 –d 002 –f –i /home/upboard/Signed_Image_UVC_5_10_6_0.bin`
* Tool will begin upgrade process, and notify when upgrade is complete.
* Firmware Check:
  * Check firmware with command:
    * `intel-realsense-dfu –p`
    * **NOTE:** Firmware version was `5.9.14` until updated to `5.10.6` on Oct. 16
  
## Set up D435 to work in ROS with Turtlebot 2
  * **NOTE:** Source files located in `./ROS/`
  * Set `TURTLEBOT_3D_SENSOR=d435` in `~/.bashrc`
  * Copy the D435 URDF file and create kobuki reference to D435 urdf
    * ```
    cp ~/ros_ws/src/realsense/realsense2_camera/urdf/_d435.urdf.xacro /opt/ros/kinetic/share/turtlebot_description/urdf/sensors/d435.urdf.xacro
    cd /opt/ros/kinetic/share/turtlebot_description/robots
    cp kobuki_hexagons_r200.urdf.xacro kobuki_hexagons_d435.urdf.xacro
    nano kobuki_hexagons_d435.urdf.xacro
    ```
  * Modify the newly created kobuki urdf file as follows (note r200-->d435):
    * ```xml
    <!-- 
        ...
        - 3d Sensor : d435
    -->
    <xacro:include filename="$(find turtlebot_description)/urdf/sensors/d435.urdf.xacro" />
    <sensor_d435  parent="base_link">
        <origin xyz="0 0 0.55" rpy="0 0 0" /> <!-- D435 mounted ~55cm above ground -->
    </sensor_d435>
    ```
  * Copy camera launch files from `turtlebot_bringup` package
    * ```
    roscd turtlebot_bringup/launch
    cp 3dsensor.launch ~/ros_ws/src/d435_test/launch/d435_start.launch
    cd includes/3dsensor/
    cp r200.launch.xml d435.launch.xml
    ```
  * Modify `~/ros_ws/src/d435_test/launch/d435_start.launch`
    * Rename the registered (aligned) depth topic
      * ```xml
      <!-- Factory-calibrated depth registration -->
      <arg name="depth_registration"  default="true"/>
      <arg name="depth_aligned"       default="$(arg depth_registration)"/>
      <arg if="$(arg depth_aligned)"     name="depth" value="aligned_depth_to_color/image_raw" />  
      <arg unless="$(arg depth_aligned)" name="depth" value="depth/image_rect_raw" />
      ```
    * Disable unnecessary processing
      * ```xml
      <arg name="ir_processing"                   default="false"/>
      <arg name="depth_processing"                default="false"/>
      <arg name="disparity_processing"            default="false"/>
      <arg name="disparity_registered_processing" default="false"/>
      ```
    * Rename the camera nodelet manager based on realsense2_camera package
      * **NOTE** manager name is defined in `realsense2_camera/launch/includes/nodelet.launch.xml`
      * ```xml
      <node pkg="nodelet" type="nodelet" name="depthimage_to_laserscan" args="load depthimage_to_laserscan/DepthImageToLaserScanNodelet $(arg camera)/realsense2_camera_manager" output="screen">`
      ```
    * Reduce min/max range of sensor for laser scan conversion
      * ```xml
      <param name="range_min" value="0.25"/>
      <param name="range_max" value="5.00"/>
      ``` 
    * Change the laserscan depth topic remapping
      * ```xml
      <remap from="image" to="$(arg camera)/$(arg depth)"/>
      <remap from="scan" to="$(arg scan_topic)"/>
      <!-- Somehow topics here get prefixed by "$(arg camera)" when not inside an app namespace,
         so in this case "$(arg scan_topic)" must provide an absolute topic name (issue #88).
         Probably is a bug in the nodelet manager: https://github.com/ros/nodelet_core/issues/7 -->
      <remap from="/camera/image" to="$(arg camera)/$(arg depth)"/>
      <remap from="$(arg camera)/scan" to="$(arg scan_topic)"/>
      ```
      
  * Modify `.../turtlebot_bringup/launch/includes/3dsensor/d435.launch.xml`
    * Change include line from `r200_nodelet_rgbd.launch` to:
      * FROM: `<include file="$(find realsense_camera)/launch/r200_nodelet_rgbd.launch">`
      * TO: `<include file="$(find realsense2_camera)/launch/rs_msral.launch">`
    * Comment out `camera_type`, `publish_tf`, and `num_worker_threads` arguments
      * ```xml
      <!-- <arg name="camera_type"                     value="D435" />
      <arg name="publish_tf"                           value="$(arg publish_tf)" />
      <arg name="num_worker_threads"                   value="$(arg num_worker_threads)" />
      -->
      ```
      
  * Copy demo launch files from realsense2_camera package
    * ```
    cd ~/ros_ws/src/realsense/realsense2_camera/launch
    cp rs_rgbd.launch rs_msral.launch
    ```
  * Modify the `rs_msral.launch` file
    * Change the file to match the following:
      * ```xml
      <arg name="enable_fisheye"        default="false"/>
      <arg name="enable_infra1"         default="false"/>
      <arg name="enable_infra2"         default="false"/>
      <arg name="enable_imu"            default="false"/>
      <arg name="enable_pointcloud"     default="false"/>
      <arg name="depth_width"           default="848"/>
      <arg name="infra1_width"          default="848"/>
      <arg name="infra2_width"          default="848"/>
      <arg name="color_width"           default="848"/>
      <arg name="depth_registered_pub"  default="depth_registered" unless="$(arg align_depth)" />"
      <arg name="depth_registered_pub"  default="aligned_depth_to_color" if="$(arg align_depth)" />"
      ```
        * NOTE: [Unresolved Bug when using enable_pointcloud:true](https://github.com/intel-ros/realsense/issues/489)
          * If enable_pointcloud:false, then pointcloud topic is `/camera/aligned_depth_to_color/points`, but if it is false, then pointcloud topic is `/camera/depth/color/points`, and I assume that the former is software generated and the latter is hardware-generated?
    
  * Set up RViz to visualize data
    * `rosrun rviz rviz`
    * Global Options --> Fixed Frame: `camera_link`
    * Add the following Displays (click `Add` at bottom left of screen)
      * PointCloud2
        * Topic: `/camera/aligned_depth_to_color/points`
      * Image: Color
        * Topic: `/camera/color/image_raw`
      * Image: Depth
        * Topic: `/camera/aligned_depth_to_color/image_raw`
      * LaserScan
        * Topic: `/scan`

* Troubleshooting camera errors/warnings
  * Errors/Warnings when running `roslaunch realsense2_camera rs_*.launch`
    1. `(backend-v4l2.cpp:1013) Frames didn't arrived within 5 seconds`
      * [Known Issue](https://github.com/IntelRealSense/librealsense/issues/1086)
      * Need to unplug and replug camera
      * Seems to happen every time after ROS device launcher terminates
    * `(sensor.cpp:338) Unregistered Media formats : [ UYVY ]; Supported: [ ]`
      * Not sure the cause, but should be able to ignore
    * ```
    WARNING... (types.cpp:57) get_xu(id=11) failed! Last Error: Input/output error`
    [WARN]... Reconfigure callback failed with exception get_xu(id=11) failed! Last Error: Input/output error:
    ```
      * Seems to mostly occur when disable `infra1` and `infra2`, but can ignore
    * `[ERROR]...An error has occurred during frame callback: null pointer passed for argument "frame_ref"`
      * Happens when try to view `/camera/aligned_depth_to_color/image_raw` when `enable_pointcloud=true`
        * Work-around, set `enable_pointcloud=false` in `rs_msral.launch` and rely on software-registered pointcloud
    * Optimal image resolution for D435
      * [848x480](https://github.com/intel-ros/realsense/issues/397)
    * Aligning depth to color is slow (~7 Hz) UNRESOLVED
      * [Issue #2321](https://github.com/IntelRealSense/librealsense/issues/2321)
      * [Issue #2376](https://github.com/IntelRealSense/librealsense/issues/2376)
    * Dropped frames after bad camera reset
      * [Custom librealsense branch](https://github.com/icarpis/realsense/commit/292e7f8204aa1ab03633a0b161e47ccdbdb69bc4) with hardware reset built-in
      

## Install Additional Python Packages
1. **SciKitLearn**
  * Install dependencies
    ```
    sudo apt install python-numpy python-scipy python-matplotlib ipython \
    ipython-notebook python-pandas python-sympy python-nose
    ```
    
  * Install Sklearn
    * `sudo pip install -U scikit-learn`
    
* **SciKitImage**
  * Install dependencies
    ```
    sudo apt install python-matplotlib python-numpy python-pil python-scipy \
    build-essential cython
    ```
  * Install Skimage
    * `sudo pip install -U scikit-image`
    
* **QRcode**
  * Install qrcode
    * `sudo pip install qrcode`
    
* **Zbarlight**
  * Install dependencies
    * `sudo apt install libzbar0 libzbar-dev`
    
  * Install zbarlight
    ```
    git clone https://github.com/Polyconseil/zbarlight.git
    cd zbarlight
    sudo python setup.py install
    ```
    
* **Dronekit**
  * Install dependencies
    * (These are optional, needed only if the installation fails.)
    ```
    sudo apt install python-dev libxml2-dev libxslt1-dev zlib1g-dev build-essential \
         autoconf libtool pkg-config python-opengl python-imaging python-pyrex \
         python-pyside.qtopengl idle-python2.7 qt4-dev-tools qt4-designer libqtgui4 \
         libqtcore4 libqt4-xml libqt4-test libqt4-script libqt4-network libqt4-dbus \
         python-qt4 python-qt4-gl libgle3 python-dev libssl-dev
    sudo pip install future greenlet gevent
    ```
  * Install dronekit
    ```
    git clone https://github.com/dronekit/dronekit-python.git
    cd dronekit-python/
    sudo python setup.py install
    ```
* To uninstall pip packages, use:
  * `python setup.py install --record files.txt`
  * `files.txt` will list all files created by `setup.py install`, delete these files

## Install Pytorch
  * Reference: [Pytorch github](https://github.com/pytorch/pytorch#from-source)
  * Reference to install libffi-dev: [libffi-dev](https://github.com/markwal/OctoPrint-PolarCloud/issues/19)

1. Install Dependencies
  ```
  sudo apt install python-numpy cmake gcc libopenblas-dev cython libatlas-dev m4 \
                   libblas-dev libffi-dev
  sudo pip install pyyaml typing cffi
  sudo pip install -U pip setuptools
  sudo pip install leveldb==0.18
  ```
* Download Pytorch from source
  Go to [pytorch.org](pytorch.org)
  Select the options **Pytorch Build**: Stable, **OS**: Linux, **Package**: Source, **Language**: Python 2.7, **CUDA**: None
  Go to the github page suggested by **Run this command**.
  The most updated instructions are here.
  
  ```
  git clone --recursive https://github.com/pytorch/pytorch.git
  ```
* Build from source (~60 minutes)
  ```
  cd pytorch
  sudo git submodule update --init
  export NO_CUDA=1
  export NO_DISTRIBUTED=1
  sudo -E MAX_JOBS=2 python setup.py install
  ```
  If this installation fails (may fail even at 99%) because of an error like the following:
  ```
  ../../lib/libtorch.so.1: undefined reference to `dlclose'
  ../../lib/libtorch.so.1: undefined reference to `dlsym'
  ../../lib/libtorch.so.1: undefined reference to `dlopen'
  ../../lib/libtorch.so.1: undefined reference to `dlerror'
  ```
  Then first clean the last build, and use the following command for installation.
  ```
  sudo python setup.py clean
  sudo -E MAX_JOBS=2 LDFLAGS="-Wl,--no-as-needed -ldl" python setup.py install
  ```
* Install torchvision
  (In separate directory, not inside pytorch source directory)
  ```
  cd ~
  git clone https://github.com/pytorch/vision.git
  cd vision
  sudo python setup.py install
  ```
* Test installation with Python
  (Get out of the source pytorch directory first)
  ```
  cd ~
  python
  import torch
  x = torch.rand(5, 3)
  print(x)
  ```

## General udev rule setup
  1. Obtain device info
    * References: [Debian Wiki: udev](https://wiki.debian.org/udev), [Overview](http://reactivated.net/writing_udev_rules.html#udevinfo)
    * `udevadm info --attribute-walk --name /dev/<device_name>`
  * Place udev rule files in: `/etc/udev/rules.d/`
    * Example file: `99-usb-serial.rules` 
    * ```
    # 3.3/5V FTDI Adapter
    SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="AI04XENV", SYMLINK+="ttyFTDI3_5"
    # 5V FTDI Adapter
    SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="AK05C4L0", SYMLINK+="ttyFTDI5"
    ```
    
## Video4Linux Webcam Driver Usage
* Show available options for webcam 
  * `v4l2-ctl --list-formats-ext`  (shows options, with associated fps)
* Set video output format
  * `v4l2-ctl --set-fmt-video=width=1920,height=1080,pixelformat=YUYV`
* Show video capture options
  * `v4l2-ctl --help-vidcap (shows video capture options)`
* Automatically configure webcam on plug-in
  * Create udev rule in `/etc/udev/rules.d/99-webcam.rules`
  * `udevadm info --attribute-walk --name /dev/video0`
  * `SUBSYSTEM=="video4linux", SUBSYSTEMS=="usb", ATTRS{idVendor}=="05a3", ATTRS{idProduct}=="9230", PROGRAM="/usr/bin/v4l2-ctl --set-fmt-video=width=640,height=480,pixelformat=MJPG --device /dev/%k"`
    
## Set up rtabmap_ros for use with Turtlebot
* Copy and open demo launch file in nano
  * ```
  roscd rtabmap_ros
  cp demo_turtlebot_mapping.launch ~/ros_ws/src/d435_test/launch/rtab_d435.launch
  roscd d435_test/launch
  nano rtab_d435.launch
  ```
* Modify the following lines:
  * ```xml
  <arg     if="$(arg simulation)" name="rgb_topic"   default="/camera/color/image_raw"/>
  <arg unless="$(arg simulation)" name="rgb_topic"   default="/camera/color/image_rect_color"/>
  <arg unless="$(arg simulation)" name="depth_topic" default="/camera/aligned_depth_to_color/image_raw"/>
  <arg name="camera_info_topic" default="/camera/color/camera_info"/>
  <include unless="$(arg simulation)" file="$(find d435_test)/launch/d435_start.launch"/>
  ```

## Set up robot_localization
* Install ROS package
  * `sudo apt install ros-kinetic-robot-localization`
  
## Install ORB-SLAM 2
  * Reference: [https://github.com/raulmur/ORB_SLAM2](https://github.com/raulmur/ORB_SLAM2)
  
1. Install Dependencies
  * Pangolin
    * Reference: [GitHub-Pangolin](https://github.com/stevenlovegrove/Pangolin)
    * Dependencies:
      * `sudo apt install libglew-dev`
      * `sudo apt-get install ffmpeg libavcodec-dev libavutil-dev libavformat-dev libswscale-dev libavdevice-dev`
    * ```
    cd ~
    git clone https://github.com/stevenlovegrove/Pangolin.git
    cd Pangolin
    mkdir build
    cd build
    cmake ..
    cmake --build .
    ```
* Clone the repository & install
  * ```
  cd ~
  git clone https://github.com/raulmur/ORB_SLAM2.git ORB_SLAM2
  cd ORB_SLAM2
  chmod +x build.sh
  ```
  
  * Modify the `ORB_SLAM2/Examples/ROS/ORB_SLAM2/CMakeList.txt` file to include boost_system libs
    * ```
    ...
    ${PROJECT_SOURCE_DIR}/../../../lib/libORB_SLAM2.so
    -lboost_system
    )
    ...
    ```
      * (Add on line 58)
    
  * Modify `build.sh` to prevent RAM exhaustion (use 3 cores only)
    * `make -j3`
  * Build the code
    * `./build.sh`
* Build the ROS Examples
  * Add the ORB SLAM2 path to ROS environment
    * ```
    nano ~/.bashrc
    export ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:/home/upboard/ORB_SLAM2/Examples/ROS
    ```
    
  * Modify `build_ros.sh` to prevent RAM exhaustion (use 3 cores only)
    * `make -j3`
    
  * Execute the `build_ros.sh` script
    * ```
    source ~/.bashrc
    cd ~/ORB_SLAM2/
    chmod +x build_ros.sh
    ./build_ros.sh
    ```
    
* Calibrate the camera to be used:
  * Reference: [camera_calibration](http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration)
  
## Install Ceres Solver
  * Reference: [ceres-solver.org](http://ceres-solver.org/installation.html#linux)
  
1. Install Dependencies
  * ```
  sudo apt install libgoogle-glog-dev
  sudo apt install libatlas-base-dev
  sudo apt install libeigen3-dev
  sudo add-apt-repository ppa:bzindovic/suitesparse-bugfix-1319687
  sudo apt update
  sudo apt install libsuitesparse-dev
  ```
* Build, Test \& Install
  * ```
  tar zxf ceres-solver-1.14.0.tar.gz
  mkdir ceres-bin
  cd ceres-bin
  cmake ../ceres-solver-1.14.0
  make -j4
  make test
  sudo make install
  ```  
