- stick
- project
- future


Intel Stick

On my last vacation, I took the opportunity to spend more time with my family, go to the beach, read some books, resume old software projects (and start new ones)

On of those technological adventures include the opportunity to buy at a good price an used Intel Stick. I have to recognize that I didn't know about their existence until I saw a picture in the marketplace and start to read about their [technical specifications](https://ark.intel.com/content/www/us/en/ark/products/86612/intel-compute-stick-stck1a32wfc.html). They seemed fairly good at release date (Q1 2015) but for 2022 they seem limited at least

Basically an Intel(R) Atom(TM) CPU  Z3735F @ 1.33GHz with 2GB RAM and 32GB of flash storage. About connectivity, two USB ports and one Micro SD card is all you have. In any case, I am still impressed to see a complete mini computer in such a small size

![]()

The project: remotely play videos and pictures on a TV

## Configuration


It was configure with dual boot, Windows 10 home 32 bit and Android 4.x

Noisier than I expected...

[https://www.intel.com/content/www/us/en/products/sku/86612/intel-compute-stick-stck1a32wfc/downloads.html](https://www.intel.com/content/www/us/en/products/sku/86612/intel-compute-stick-stck1a32wfc/downloads.html)

STCK1A32WFC

[https://linuxiumcomau.blogspot.com/2017/06/customizing-ubuntu-isos-documentation.html](https://linuxiumcomau.blogspot.com/2017/06/customizing-ubuntu-isos-documentation.html)

download isorespin_sh

http://url.linuxium.com.au/isorespin_sh

	sudo mv isorespin.sh /usr/local/bin
        sudo chmod 755 /usr/local/bin/isorespin.sh

        sudo apt -y install p7zip-full bc klibc-utils iproute2 genisoimage dosfstools  
        sudo apt -y install squashfs-tools rsync unzip wget findutils xorriso bsdutils
        
download 64 bit image, for example: xubuntu-21.10-desktop-amd64.iso
        
	isorespin.sh -i xubuntu-21.10-desktop-amd64.iso --atom

it will generate a linuxium-xubuntu-21.10-desktop-amd64.iso image

On Windows and using Rufus create booteable flash drive with "dd" mode

> on linux didn't work to me this last part

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

