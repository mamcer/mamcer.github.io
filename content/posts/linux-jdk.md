---
layout: post
title: JDK linux install
subtitle: basic jdk install on linux
---

# JDK

Tested on a CentOS 7 minimal installation

## Install

The current latest stable version is 1.8.0_77

    cd /opt/
    wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u77-b03/jdk-8u77-linux-x64.tar.gz"
    tar xzf jdk-8u77-linux-x64.tar.gz

    cd /opt/jdk1.8.0_77/
    alternatives --install /usr/bin/java java /opt/jdk1.8.0_77/bin/java 2
    alternatives --config java

## Set environment variable

add a `jdk.sh` file in `/etc/profile.d/`

With the following content

    export JAVA_HOME=/opt/jdk1.8.0_77
    export PATH=${PATH}:/opt/jdk1.8.0_77/bin 
    
Logout

    gnome-session-quit
    
## Test 

Test the installation by executing

    java -version