# SonicPad Multiple USB Cameras Instructions
The Sonic Pad is a great device and an easy way to get Klipper up and running. Unfortunatelly some of the standard capabilities of Klipper on Raspberry PI are missing (deliberately? don't know)... One of them is the ability to run multiple UBS cameras.

Since the January 2023 update, Creality has provided us access to SSH and root accounts, this enables a number of possibilities, like [installing Obico](https://github.com/wavezcs/SonicPadObico).

One thing that I was still missing was the ability to run multiple cameras (one for Obico, one for Timelapse). The problem is that _mjpg-streamer_ is "missing" and any changes to webcam.txt gets completely ignored...

Creality encapsulated the _mjpg-streamer_ inside a command called _rtsp_demo_, which always connect to /dev/video0 and streams it at 1280x720 10fps, regardless of how many cameras you have connected and any configuration of the webcam.txt.

I managed to install _mjpg-streamer_ into the Sonic Pad so you can manually configure all the USB cameras you have connected. Below are instructions to get it going. You will need to use root account in order to complete these steps. Please be careful as anytime you are using root, there is a chance you could brick the device, if you can't boot into it. Creality has not provided a process to completely flash the device.

**This guide was written for folks with prior experience using linux. Only follow this guide if you are comfortable editing files, navigating linux filesystems, and can do your own troubleshooting. This worked for me, but I can't guarantee it will work for you.**

>If you do run into some trouble, you can restore the device by running the following command:
>>/usr/share/script/recovery.sh all

This guide was setup for Sonic Pad firmware 'V1.0.6.46.25  09 Mar. 2023'

The following are the high level steps:

1. SSH to the device using root account
2. Download and install Entware
3. Install mjpeg-streamer
4. Test / Configure your cameras
5. Set to Run at Boot

Let's do it!

## 1. SSH to the device using root account

  NOTE: The latest update of the sonic pad will allow you to retrieve the root password. Take a note of it as you will need below.

  ### 1.1 Grab your root password
  - From your Sonic Pad screen, go to Configure -> Other Settings -> Advanced Options -> Root Account
  - Read the warning, tick the box and click "Next Step"
  - The root password will be shown on the screen.
  
  ### 1.2 SSH in as root
  ```
  ssh -oHostKeyAlgorithms=+ssh-rsa root@<your-ip/hostname>
  ```
  >Password: <from_your_sonicpad>

## 2. Download and install Entware

  ### 2.1 Get Entware install script.
  ```
  mkdir /mnt/UDISK/entware
  cd /mnt/UDISK/entware
  wget http://bin.entware.net/aarch64-k3.10/installer/generic.sh
  ```

  ### 2.2 Install Entware
  ```
  chmod 755 generic.sh
  ./generic.sh
  ```

## 3. Install mjpg-streamer

  ### 3.1 Use Entware opkg to download the packages
  ```
  /opt/bin/opkg install mjpg-streamer mjpg-streamer-input-uvc mjpg-streamer-output-http coreutils-yes
  ```
  
## 4. Test / Configure your cameras

  ### 4.1 Stop _rtsp_demo process_, so you can start your own _mjpg-streamer_ instances.

  ```
  /etc/init.d/rtsp_server stop
  ```
  This will stop streaming your existing camera.
 
  ### 4.2 Manually run  _mjpg-streamer_ for each of your cameras
  Below is the sample of the command line, with explanation of each of the parameters:
  ```
  /opt/bin/mjpg_streamer --input "/opt/lib/mjpg-streamer/input_uvc.so -d [camera_device] -f [frame_rate] -r [resolution] [-y]" --output "/opt/lib/mjpg-streamer/output_http.so -p [port] -w /www/webcam"
  ```
  Valid options:
  - camera_device: location of the USB Camera device. eg.:
    - /dev/video0
    - /dev/video1
    - etc.
  - frame_rate: Any number that will indicate how many frames per second. The lower the less resources it will consume. eg.:
    - 10
    - 15
    - 30
  - resolution: width x height in pixels. Again, the lower the resolution, the less resources it will consume. eg:
    - 640x480
    - 1280x720
    - 1920x1080
  - -y: some cameras stream in a different format (YUV), if you're not getting anything without _-y_, try adding it to your command.
  - port: which port the USB camera will stream to. eg:
    - 8080 (maps to /webcam)
    - 8081 (maps to /webcam2)
    - 8082 (maps to /webcam3)
    - 8083 (maps to /webcam4)
  
  This is what I use for my primary camera:
  ```
  /opt/bin/mjpg_streamer --input "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -f 15 -r 1920x1080" --output "/opt/lib/mjpg-streamer/output_http.so -p 8080 -w /www/webcam"
  ```
  
  And this is for my secondary camera:
  ```
  /opt/bin/mjpg_streamer --input "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video1 -f 10 -r 1280x720 -y" --output "/opt/lib/mjpg-streamer/output_http.so -p 8081 -w /www/webcam"
  ```
  Notice that I use a lower resolution and frame rate for my Obico camera, and it requires the use of the -y (YUV format).
  
  After starting the streaming, go to your browser and try accessing your Sonic Pad IP webcam stream for each of your cameras (one at a time, according to the port you're using.
  ```
  http://<your_sonipad_ip>/webcam?action=stream
  http://<your_sonipad_ip>/webcam2?action=stream
  ```

  To end the streaming, go back to your SSH and use ctrl-c. Make sure you take note of each successful command per camera.
  
## 5. Run at Boot

  Now that you managed to stream all your cameras individually, we can set them up to run together, at boot.
  
  ### 5.1 Edit your start up script
  
  ```
  vi /etc/rc.local
  ```
  
  Include the _rstp_demo stop_, your successful _mjpg-streamer_ commands followed by _--background_ just above _exit 0_. Something like this:
  
  ```
  /etc/init.d/rtsp_server stop
  /opt/bin/mjpg_streamer --input "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video0 -f 15 -r 1920x1080" --output "/opt/lib/mjpg-streamer/output_http.so -p 8080 -w /www/webcam" --background
  /opt/bin/mjpg_streamer --input "/opt/lib/mjpg-streamer/input_uvc.so -d /dev/video1 -y -f 10 -r 1280x720" --output "/opt/lib/mjpg-streamer/output_http.so -p 8081 -w /www/webcam" --background
  exit 0
  ```
  
  Note: The _--background_ makes the command to run as a deamon.
  
  ### 5.2 Reboot
  Simply turn your Sonic Pad off and on again. After it reboots you should be able to access both cameras simultaneouly using their respective webcam folders:
  ```
  http://<your_sonipad_ip>/webcam?action=stream
  http://<your_sonipad_ip>/webcam2?action=stream
  ```
  Now go your Fluidd or Mainsail interface, add them to the webpage and you're good to go!
