---
layout: post
title: Bamboo Linux Agent Installation
subtitle:  Agent installation on CentOS
---

On this post I will describe my experience installing a Bamboo agent on a Linux server (CentOS based) 

## Requirements

### JDK

It's strongly recommended to install the same JDK version as the Bamboo server. You can follow my post about [how to install the JDK in a CentOS server](https://mamcer.github.io/2016-12-24-linux-jdk/).

In my case the configuration is:

    java version "1.8.0_51"
    Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
    Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)

### Agent JAR

The next step is to download the agent JAR from our Bamboo server. It can be found in: 

Administration > Overview > Agents > Install Remote Agent

[https://[bamboo-server]/admin/agent/addRemoteAgent.action]()

Download Remote Agent JAR option. Next you can copy the jar file to our server through scp for example.

## User Creation

It's very recommended to create a specific user for the agent. In a peak of originality we will call it `bamboo-agent`

As sudo:

    adduser bamboo-agent
    passwd bamboo-agent

login as the bamboo-agent user

    su - bamboo-agent

> Another way to download the remote agent jar is logged in as the bamboo-agent user use wget or curl to download the file : [https://[bamboo-server]/agentServer/agentInstaller/atlassian-bamboo-agent-installer-[bamboo-server-version].jar]()

## Run the Agent

You can run the agent on the server by executing:

    java -jar atlassian-bamboo-agent-installer-[bamboo-server-version].jar https://[bamboo-server]/agentServer/ &

> In a ssh session you can append & at the end to leave the process running in background

There are different additional agent options. You can review the [official bamboo documentation](https://confluence.atlassian.com/bamboo/additional-remote-agent-options-436044733.html) for an extensive list of available options.

## Docker (Only)

To avoid the need of sudo to run docker on the agent as part of a Bamboo job execution we can add the `docker` group and include our `bamboo-agent` user to it.

    sudo groupadd docker
    sudo gpasswd -a bamboo-agent docker
    sudo service docker restart

## Open Firewall Ports

In our Bamboo server we have to explicitly approve the agent ip and different ports in the firewall. This step is necessary first to allow the Bamboo server to see our agent server and next for different monitoring activities like the heartbeat or the way the Bamboo server determine if an agent is still online.

    /sbin/iptables -L
    vi /etc/sysconfig/iptables 

Add the following lines:

    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 8085 -s [bamboo-agent-ip] -j ACCEPT
    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 54663 -s [bamboo-agent-ip] -j ACCEPT
    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -s [bamboo-agent-ip] -j ACCEPT

then we can execute the following commands to apply our changes:

    sudo /sbin/service iptables restart
    /sbin/iptables -L

## Approve the Agent

In the Bamboo server as an administrator go to Agents > Agent authentication

You should find our recently executed agent ready to be approved. We have to explicitly approve the agent in order to see it in the list of online agents.

In a matter of seconds we should see our remote agent in the list of online agents.

## Proposed Directory Structure

Bamboo agent files:

    /home/bamboo-agent/
        /bamboo-home/
            [bamboo-agent-home-directory]

Additional tools location:

    /opt/
        bamboo/
            7z/
            maven3/
            sonar-scanner/
            etc…
