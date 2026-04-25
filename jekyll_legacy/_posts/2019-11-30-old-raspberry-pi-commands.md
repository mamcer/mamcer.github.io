---
layout: post
title: Old Raspberry Pi commands
subtitle: Tested on a Raspberry Pi 2 B
---

Back in 2015 I was in United States on vacations an decided to bought a Raspberry Pi Canakit. The latest model  at that moment  was 2 B and the possibilities were infite: learn bash, play movies in a non-smart TV, remote play music, task automation

The reality was a little different. It ended mostly resting in a drawer of my desk until I decided to sell it in 2017.. Maybe lots of you share a similar story

But it was not completely unused. I have implemented a .NET mono POC to remotely play music using SignalR to make it realtime [https://github.com/mamcer/usignalr-raspberry](https://github.com/mamcer/usignalr-raspberry) and some other quick little POCs

Two days ago I found some txt files with notes in an old private git repository and decided to share them as Gists and then in this blog post

## Keep Program running after a ssh session is closed

Install screen 

    sudo apt-get install screen

Run it: 

    screen 

Commands:

* Ctrl + A + C:  Create a new SCREEN session
* Ctrl + A + N:  Switch to the Next screen session
* Ctrl + A + P:  Switch to the Previous screen session
* Ctrl + A + D:  Detaches a screen session (without killing the * processes in it - they continue)

Close a session: 

    exit

To see open sessions: 

    screen -ls

To switch to an open session type: 

    screen -r 2494.pts-0.raspberrypi (where 2494.pts-0.raspberrypi is the session name)

You can run a program in a screen session and it will continue running after you close your ssh session

## Mount Windows share

Install dpkg
    
    dpkg -s cifs-utils

if it is not installed: 
    
    sudo apt-get install cifs-utils

Create shared folder home
    
    sudo mkdir -p /media/[mc]/[public]

Add an entry in fstab
    
    sudo nano /etc/fstab

add the following line:

    //[192.168.0.148]/[public] /media/[mc]/[public] cifs guest,uid=[1000],gid=[1000],iocharset=utf8 0 0

> guest is basically telling the network drive that it’s a public share, and you don’t need a password to access it (not to confuse with username)  
> uid=1000 makes the Pi user with this id the owner of the mounted share, allowing them to rename files  
> gid=1000 is the same as uid but for the user’s group  
> iocharset=utf8 allows access to files with names in non-English languages.

To know your uid and gid for a specific user you can run: 

    id [username]

Mount

    sudo mount -a

With Credentials

if you need credentials you can configure: 

    //[192.168.0.148]/[public] /media/[mc]/[public] cifs username=[username],password=[password],uid=[1000],gid=[1000],iocharset=utf8 0 0

> you can use a credentials file to make it more secure

source: [http://geeks.noeit.com/mount-an-smb-network-drive-on-raspberry-pi](http://geeks.noeit.com/mount-an-smb-network-drive-on-raspberry-pi)

## Play video from terminal

Using omxplayer

Example:

    pi@raspberrypi ~ $ omxplayer -r -b -o local -l 00:12:00 /media/[flash-drive]/videos/my_video.mkv 

> -b : black background  
> -o : audio output local (3.5 jack)  
> -l : start from 00 hours 12 min 00 sec

Then you can use:

* "+" to increase volume  
* "-" to decrease volume  
* "s" to enable/disable subtitles
* Left and Up arrow to jump time
* "q" to quit

Hopefully they are still current and someone find them useful