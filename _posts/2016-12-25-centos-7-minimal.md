---
layout: post
title: CentOS 7 Minimal
subtitle: Configuration
---

# CentOS 7 Minimal Installation Configuration

Based on: [http://www.tecmint.com/things-to-do-after-minimal-rhel-centos-7-installation/]()

A small subset of steps I was executing after a CentOS 7 minimal installation. In this case it was to configure different Bamboo, Crucible and JIRA test server instances.


## Static IP

Install ifconfig utility

    sudo yum install net-tools

Show IP address

    ip addr show

Edit ifcfg file

    sudo vi /etc/sysconfig/network-scripts/ifcfg-enp0s3

For example:

    IPADDR="10.0.64.2"
    GATEWAY="10.0.64.1"
    DNS1="8.8.8.8"
    DNS2="8.8.4.4"

Restart network

    service network restart

Test configuration

    ip addr show
    ping -c4 google.com

## Set Hostname

    sudo vi /etc/hostname

you have to logout and login again

## Update or Upgrade CentOS

    sudo yum update && yum upgrade

## Install and Configure SSH Server

    ssh -V

Use secure protocol

    sudo vi /etc/ssh/ssh_config

uncomment Protocol 2,1 and delete 1. Final result:

    Protocol 2

Disable ssh root login

    sudo vi /etc/ssh/sshd_config

Uncomment PermitRootLogin and configure it with no. Final result

    PermitRootLogin no

Restart ssh service

systemctl restart sshd.service

## Install wget

    sudo yum install wget

## Add User

    adduser [username]

    passwd [username]

    visudo

    [username] ALL=(ALL) ALL
