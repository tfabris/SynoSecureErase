Securely Erasing a USB Disk Attached to a Synology NAS
======================================================
&copy; 2025 by Tony Fabris


Problem
-------

I have several large-capacity SATA disk drives that I want to securely erase, and I also have a USB3 dock into which I can plug SATA drives. I don't want to quick-format these drives, I want to overwrite the entire disk so that data recovery tools won't easily work on these drives. However, that takes a long time; if I plug the drive into my laptop and format it from there, the laptop has to sit and wait for the job to be done before I can take it anywhere. I don't want to be tied down to the formatting process.

Solution
--------

I am using my Synology NAS to do the job (a device which is already intended to just sit there and run forever), so that my laptop and I can walk away.

### The Catch

Synology's DSM interface doesn't give you a GUI option to securely erase an attached USB drive. So this procedure documents some shell commands you can do on the Synology, to overwrite the entire disk partition with zeroes.

### Notes

In my solution, I'm simply filling the disk with zeroes in a single pass. This is enough security for me, for my current purposes. You could modify the commands to do multiple fills, or to use random numbers if you like, but that will use more CPU power and take more time.

I don't know if this procedure will work on SSDs or USB sticks. I've only tried it on winchester-style spinning rust so far. I know that SSDs often have their own special secure erase features, so you might be better off using those.

I'm using DSM version 6.2 on my Synology, so the procedure might differ if you are using a later version.

### Drawbacks

For large disks, it can still take a long time to run. In my case, some of my larger disks (multiple terabytes) are taking days to erase.

Though this solution tries to use the Linux "nice" command to prevent it from hogging the CPU, in my experience this sometimes still makes the Synology slow down a little bit. Specific errors that I saw were that my Time Machine backups to the Synology would intermittently fail, and the Surveillance Station "Live Broadcast" feature glitched sometimes when streaming to YouTube. I'm not sure if this slowness was because of the CPU processing of the "dd" command, or if it's because of contention on the system bus from all the USB I/O that's happening. Also, it didn't seem to happen every time, so the disk drive itself might be a factor. Not sure.

Also, this process has a size limit of about 16 terabytes. That's because the EXT4 file system has a 16 terabyte file size limit. This is OK for my current purposes, since I don't yet have any disks quite that big yet, but I'm starting to get close. If I get something bigger, I'll need to improve this procedure.


Steps
-----

### Temporarily disable any timed reboot features, if any

  This step might not apply to you, but it's important to mention this. I have scripts which reboot my network routers and my Synology NAS in the wee hours of the night, to ensure that they are continuously stable during daytime use. For example, in my Synology Task Scheduler, I must temporarily disable my tasks "PowerOff task x" and "PowerOn task x". This allows the Synology to run these disk erasures overnight without stopping. If you have this sort of thing running on your NAS, you'll likely know about it because you're the one who set it up.

### Ensure that the "SynoCli Network Tools" package is installed on your Synology

  From the Synology DSM web interface:

  - Open the "Package Center", and press its "Settings" button in the upper-right corner. In the "Package Sources" tab, click "Add". Give it a Name of "SynoCommunity" and a Location of "https://packages.synocommunity.com/", and then confirm through its prompts.
  - In the Package Center, select the "Community" tab, and ensure the package called "SynoCli Network Tools" is installed on your NAS.
  - This will install the "screen" command, which we'll need later.

### Attach your disk to the Synology, using a USB attachment of some kind

  My USB attachment is a USB3 drive caddy, it's a "Plugable" brand, model USB3-SATA-UASP1. It's several years old and I'm sure you can find better ones these days. Attach it to the fastest port you can. In my case, that's a USB3 port on the synology. Mine has a USB2 port on the front of the unit and two USB3 ports on the back of the unit. Make sure to use a USB cable that is capable of the fastest speeds, in my case that was a USB3-specific cable with a special connector on the "B" end which fits my caddy.

### Identify the attached disk

  From the Synology DSM web interface:

  - Open "Control Panel", navigate to "External Devices", and select the USB disk drive that you plugged in.
  - Expand the details of the disk drive by pressing the ⋁ symbol.
  - **Ensure that it looks like the correct device and size.** If there is no USB disk drive found, then the disk drive may be bad, or your USB interface or cable may be bad. If it doesn't look like the correct size, maybe you're looking at the wrong disk, though keep in mind that the External Devices screen will list a lower number than the marketing-speak that's printed on the disk drive label, for example, my "1TB" drive is actually just 916GB in the External Devices screen. In any case, if things don't look correct, stop and troubleshoot this if needed. You don't want to erase the wrong disk!

### Quick-format the attached disk

  While still in the "External Devices" panel:

  - With the correct USB drive selected, press the "Format" button, choose the formatting option "Entire Disk", and file system type of "EXT4". Confirm this and wait for this quick formatting process to complete.
  - Though this doesn't **securely** wipe the disk, it ensures the disk has only one partition, the partition is the correct type to accept large files, and the partition is empty. Now the empty partition can be completely filled with the erasure file we are going to create.

### Use shell commands to securely erase all empty space on the drive

  The quick format did not securely erase it yet. The Synology has no option to securely erase a USB disk from the GUI, so you must use shell commands to do the job. Details:

  - Open a shell prompt on your computer and SSH into your Synology NAS (using your administrator password):

         ssh admin@mySynologyName.local

  - Go into sudo shell mode, because many of the below commands require elevation (you'll need to enter the password again).

         sudo -s

  - Learn the system name of the USB disk device:

         mount | grep usbshare

    Note: You must do this every time you plug in a new disk because the name might change. My output looked similar to this:

         /dev/sdr1 on /volumeUSB1/usbshare type ext4 (...)

    That means:

         "/volumeUSB1" <- The USB input.
         "/usbshare"   <- The shared drive partition on the USB drive.
         "/dev/sdr"    <- System disk drive name ("sd" for SATA drive, "r" for drive letter R)
         "/dev/sdr1"   <- Partition name on that drive ("1" for the first partition).

  ------------------------------------------------------------------------

  > [!IMPORTANT]
  > Your results may have different names! Such as "sds" instead of "sdr", or "volumeUSB2" instead of "volumeUSB1". Remember the names, whatever they turn out to be, and replace those names in all of the following commands with the correct ones. Failing to replace the names correctly will result in unexpected errors, or worse, erasing a disk that you didn't mean to erase!

  ------------------------------------------------------------------------

  - Check that the disk drive size is the expected correct size:

         fdisk -l | grep sdr

    The output looks something like this. Ensure its size is the expected size:

         Disk /dev/sdr: 149.1 GiB, 160041885696 bytes, 312581808 sectors
         /dev/sdr1         256 312576704 312576449  149G 83 Linux

  - Fill the partition with zero bytes.

    - The command below uses "screen" to launch a background screen session, so that you can exit your SSH shell without terminating the running process.
    - The command below uses "nice -n xx" to prevent dd from interfering with the NAS's other processes. -n 0 is default normal priority, -n 19 is maximum niceness. Resist the temptation to lower the nice value to something less nice. Doing so won't make it much speedier, yet it would mess up other processes on the Synology.
    - The command below uses "dd" to directly copy an infinite stream of zeroes to a file named "zerofile.dat" on the USB drive. It will continue doing this until it runs out of disk space or gets some other kind of error. **Ensure that you enter the correct name of the USB drive that was learned in the steps above!**

            screen nice -n 19 dd if=/dev/zero of=/volumeUSB1/usbshare/zerofile.dat status=progress

  ------------------------------------------------------------------------

  > [!NOTE]
  > Resist the temptation to change the command to write zeroes directly to the parent drive (such as /dev/sdr1 in my example) instead of a data file. Writing directly to the drive will mess up the partition's file system and prevent you from being able to monitor the progress via the External Devices control panel. I have also noticed it causes more negative effects on the rest of the Synology system, not sure why.

  ------------------------------------------------------------------------

### Monitor progress

  After executing the command, a line of text showing the progress will appear, and will continuously update the screen session with its progress, thanks to the "status=progress" parameter. Though it doesn't show a progress bar, you can see the number of gigabytes copied so far, and its speed.

  - You can close and terminate your SSH window now, and dd will keep running in the background, thanks to the "screen" command we used.

  - You can monitor the progress via the Synology control panel. Under "External Devices" if you expand the disk drive details, you can see the "Used/Total Size" value changing. Once the "Used" number equals the "Total Size" number, the drive is done being filled.

  - At any time during the erasure, you can optionally reconnect via SSH, to monitor the progress of the dd process:

         ssh admin@mySynologyName.local   (log back in)
         sudo -s                          (become the super user that launched dd above)
     
    Here, you can optionally do any of the following:

         top                          (list processes, and find the PID of the dd command)
         q                            (quit the top command)
         renice -n 19 -p 16572        (re-prioritize the niceness of the dd process, using the PID found in "top")
         screen -ls                   (list existing screen sessions)
         screen -r                    (resume the screen session and watch its progress)

    Remember: To list or resume your screen session, you have to be running as the same user that launched the screen session, which is why the "sudo -s" is needed.

    If you enter "screen -ls" and it responds with something like "No Sockets found in /tmp/uscreens/S-admin", or you enter "screen -r" and it says "There is no screen to be resumed", then it thinks that the screen session doesn't exist. It means that one of these things has happened:
      - Maybe you didn't do "sudo -s" successfully before starting the dd process.
      - Maybe you didn't do "sudo -s" successfully after logging back in.
      - The dd process has ended, either because it successfully filled the disk, or maybe because of some other error. 
      - You can confirm whether the dd process has ended by looking at the output of the "top" command.


Finishing
---------

When it's done, ensure that it wrote the correct number of bytes to fill the entire disk. You can see this by looking at the Synology web interface, under "Control Panel", "External Devices". You can also see the "zerofile.dat" file in the "File Station" app.

### Final steps

- In the Control Panel, do another quick format on the disk drive, so that it's left with an empty, usable file system.

- Remember to eject the drive from the Control Panel ("Eject" button in "External Devices" screen) before physically removing the USB disk.

- Remember to re-enable your timed reboot features, if any. For example in Task Scheduler, I had to re-enable my disabled "PowerOff task x" and "PowerOn task x" entries.


References:
-----------

- https://community.synology.com/enu/forum/17/post/90560
- https://www.themtparty.com/secure-erase-synology-nas-usb/
- https://confluence.atlassian.com/bbkb/how-do-i-keep-a-linux-shell-runner-online-after-i-log-out-of-a-remote-ssh-session-1312162785.html
