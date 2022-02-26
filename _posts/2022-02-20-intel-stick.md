---
layout: post
title: Intel Stick
subtitle: remote execution
---

- stick
- project
- future


Intel Stick

On my last vacation, I took the opportunity to spend more time with my family, go to the beach, read some books and resume old software projects (and start new ones)

On of those technological adventures include the opportunity to buy at a very good price an used Intel Stick

![intel-stick](../img/2022-02-20-intel-stick/01-intel-stick.jpg)

I have to recognize that I didn't know about their existence until I saw a picture in the marketplace and start to read about their [technical specifications](https://ark.intel.com/content/www/us/en/ark/products/86612/intel-compute-stick-stck1a32wfc.html). They seemed fairly good at release date (Q1 2015) but for 2022 they seem limited at least

Basically an Intel(R) Atom(TM) CPU  Z3735F @ 1.33GHz with 2GB RAM and 32GB of flash storage. About connectivity, two USB ports and one Micro SD card is all you have

![intel-stick-ports](../img/2022-02-20-intel-stick/02-intel-stick-ports.jpg)

The project: Install a Linux flavor and remotely play videos and pictures on a TV

## Configuration

It was configured with dual boot, Windows 10 home 32 bit and Android 4.x. It clearly struggle to run Windows in an acceptable performance level. Android runs well but it looks incredible old like in fact 10 years old :)

Also it is noisier than I expected. But in any case, it still amazes me to see products like this. A complete mini computer in such a small size

Next step: install Linux

I wish I had spent more time on this point. Research I mean. That 32 bits Windows was not only because of the 2GB of ram memory. But also because of the bios, In my case many unknown details here but it seemed by hardware boot time had to be in 32 bits

My first option was Debian 11. Thinking in a Debian XFCE desktop, spoiler alert: I didn't get that far.. I successfully loaded the installer, formatted the drive and start the minimal installation. Everything looked good until it wasn't able to detect the Wifi. After some time trying different options I ended with a terminal installation with no network. Not was intended and it clearly exceed by far my limited Linux knowledge to find a solution to the missing Wifi

Then I tried another distros like XUbuntu (64 bit) with no success. It not even recognize the drive as bootable

Forced me to read, what I should have done from the beginning. I found the portal [https://linuxiumcomau.blogspot.com](https://linuxiumcomau.blogspot.com/2017/06/customizing-ubuntu-isos-documentation.html) and following some clearly explained steps I was able to generate an image of xubuntu 21.10 that would boot

Steps that worked for me: 

download isorespin_sh (http://url.linuxium.com.au/isorespin_sh)

    sudo mv isorespin.sh /usr/local/bin
    sudo chmod 755 /usr/local/bin/isorespin.sh

    sudo apt -y install p7zip-full bc klibc-utils iproute2 genisoimage dosfstools  
    sudo apt -y install squashfs-tools rsync unzip wget findutils xorriso bsdutils
        
Download a 64 bit linux flavor, for example: xubuntu-21.10-desktop-amd64.iso
        
	isorespin.sh -i xubuntu-21.10-desktop-amd64.iso --atom

It will generate a linuxium-xubuntu-21.10-desktop-amd64.iso image

On Windows and using [Rufus](https://rufus.ie/en/) create a booteable flash drive with "dd" mode using the newly created image
> Why Windows? at least for me on linux didn't work for me generate a booteable drive recognized by the stick

The installation took time but a success

![intel-stick-desktop](../img/2022-02-20-intel-stick/03-intel-stick-desktop.jpg)

Install openssh-server and start playing remotely with the computer



sudo umount /dev/<USB device>*
sudo dd if=<respun ISO> of=/dev/<USB device> bs=4M

## hardware

Intel(R) Atom(TM) CPU  Z3735F @ 1.33GHz
2GB RAM
32GB flash drive

xubuntu

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=21.10
DISTRIB_CODENAME=impish
DISTRIB_DESCRIPTION="Ubuntu 21.10"


## play movie

			cmd = exec.Command("mplayer", "-fs", "-vo", "xv", "-ao", "alsa:device=hdmi", "-lavdopts", "threads=4", p, "&")
			cmd.Stdout = os.Stdout
			if err := cmd.Run(); err != nil {
				fmt.Println("Error: ", err)
			}

    export DISPLAY=:0
    mplayer -fs -vo xv -ao alsa:device=hdmi [video-file]

    mplayer -fs -lavdopts threads=4 [video-file]

[https://github.com/mamcer/nmovies/blob/master/main.go](https://github.com/mamcer/nmovies/blob/master/main.go)

> previous test trough ssh

## images

    sudo apt install feh
    feh -F ./Pictures/563139.jpg

