---
layout: post
title: Old Hardware Debian 9
subtitle: Golang development
---

This post have at least two years in draft mode. Does not have anything special, a short story and a list of command hope someone will find useful

Back in 2019 I found an old laptop in my house [LG S1](https://www.notebookcheck.net/Review-LG-S1-Pro-Notebook.2916.0.html) (Intel Core 2 duo from 2006) battery long dead but I conected it to AC power and it started. Originally shipped with Windows XP (and Vista Ready :)) Nostalgia + willigness to learn Golang => this post

I replaced the old 120GB 5400RPM hard drive with a brand new SSD drive put a Debian 9 DVD and it returned to the living

Here are the list of commands and tools I installed in a XFCE Debian 9.1 environment

## install sudo

    su
    apt install sudo
    adduser [username] sudo

logout
  
## set time date

    timedatectl
    timedatectl set-timezone America/Argentina/Buenos_Aires

## upgrade

    sudo apt update
    sudo apt upgrade

## show kernel version

    uname -a

## install htop

    sudo apt install htop

## install vim

    sudo apt install vim

## install git

    sudo apt install git

## install go

download Go (latest current version 1.13.4)

    tar -C /usr/local -xzf go1.13.4.linux-amd64.tar.gz

add go to path

    vim ~/.profile
    export PATH=$PATH:/usr/local/go/bin

    source ~/.profile

test your installation

    go version

multiple go paths

    vim ~/.profile
    export GOPATH=/home/mario/go:/home/mario/projects/go

[https://golang.org/doc/code.html](https://golang.org/doc/code.html)

## install docker

[https://docs.docker.com/install/linux/docker-ce/debian/#install-docker-ce](https://docs.docker.com/install/linux/docker-ce/debian/#install-docker-ce)

setup repository
    
    sudo apt update

install packages

    sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common

add docker GPG key

    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

setup stable repository

    sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/debian \
    $(lsb_release -cs) \
    stable"

update apt

    sudo apt update

install latest version of docker 

    sudo apt install docker-ce docker-ce-cli containerd.io

test installation

    sudo docker run hello-world

add your user to docker group (no need for sudo to run docker commands)

    sudo usermod -aG docker $USER

## visual studio code

Go to [https://code.visualstudio.com/](https://code.visualstudio.com/)
download .deb package (latest current version 1.4)

    sudo dpkg -i code_1.40.2-1574694120_amd64.deb

## extra: appearance

[https://www.gnome-look.org/p/1214931/](https://www.gnome-look.org/p/1214931/)

my selected option was: Flat-Remix-GTK-Blue-Dark-Solid

## extra: configure firefox

[https://markosaric.com/firefox/](https://markosaric.com/firefox/)  

## extra: connect android phone	

install jmtpfs

    apt-get install jmtpfs
    mkdir ~/nokia
    jmtpfs ~/nokia

## final thoughts

With only 2 gb of ram memory sometimes it struggle to run VS Code and for example a Web browser at the same time.. 
You can replace VSCode with a less resource hungry option like Vim and a plugin like [vim-go](https://github.com/fatih/vim-go) for example, but in my case at least I'm not that kind of power user yet

It was fun and really useful, something that I like about these environments that today end up being restricted by their resources is that they force you to look for economic alternatives and to be focused on a specific task