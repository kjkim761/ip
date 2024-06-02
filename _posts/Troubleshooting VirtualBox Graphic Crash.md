# Troubleshooting VirtualBox Graphic Crash

I am using VirtualBox to play other OS on windows. I wanted to play Linux guest OS (Ubuntu). But VirtualBox kept crashing.

Through numerous trials and errors, I found a way to resolve this problem. 

First, I installed older version of VirtualBox(6.10).
<br>In the display setting, set Graphic Controller to VMSVGA and uncheck "Enable 3D Acceleration" and increase Video Momory to 24MB.
<br>After installing guest OS, "insert Guest Additions CD Image" and run autorun.sh script.

That's all. Finally it works fine.

Tip: It's always recommended to use "Snapshot" feature of VirtualBox for restoring.
