---
layout: post
title: Bamboo Ubuntu Install
subtitle: Installation steps
---

# Bamboo Ubuntu

## Initial Notes

I have installed it on my laptop so I have skipped some steps like create a specific bamboo user, etc.

## Create Database

If you don't have a database you can follow the guidelines described in requirements/postgres or requirements/sql-server

> Remember to create an application database and user. For example: bamboodb and bamboouser

## Create Installation Directory

    cd /opt
    mkdir atlassian
    cd atlassian
    wget https://www.atlassian.com/software/bamboo/downloads/binary/atlassian-bamboo-5.10.3.tar.gz
    tar zxvf atlassian-bamboo-5.10.3.tar.gz
    cd ../..
    ln -s atlassian/atlassian-bamboo-5.10.3 bamboo
    chown [user] atlassian

## Set Bamboo Home

    vi atlassian-bamboo/WEB-INF/classes/bamboo-init.properties

Uncommment bamboo.home with a value like: bamboo.home=/opt/atlassian/app-data/bamboo-home

Create service file

As root, create the file /etc/init.d/bamboo with the following content:

    #!/bin/sh
    set -e
    ### BEGIN INIT INFO
    # Provides: bamboo
    # Required-Start: $local_fs $remote_fs $network $time
    # Required-Stop: $local_fs $remote_fs $network $time
    # Should-Start: $syslog
    # Should-Stop: $syslog
    # Default-Start: 2 3 4 5
    # Default-Stop: 0 1 6
    # Short-Description: Atlassian Bamboo Server
    ### END INIT INFO
    # INIT Script
    ######################################

    # Define some variables
    # Name of app ( bamboo, Confluence, etc )
    APP=bamboo
    # Name of the user to run as
    USER=mario.moreno
    # Location of application's bin directory
    BASE=/opt/bamboo

    case "$1" in
    # Start command
    start)
        echo "Starting $APP"
        /bin/su - $USER -c "export BAMBOO_HOME=${BAMBOO_HOME}; $BASE/bin/startup.sh &> /dev/null"
        ;;
    # Stop command
    stop)
        echo "Stopping $APP"
        /bin/su - $USER -c "$BASE/bin/shutdown.sh &> /dev/null"
        echo "$APP stopped successfully"
        ;;
    # Restart command
    restart)
            $0 stop
            sleep 5
            $0 start
            ;;
    *)
        echo "Usage: /etc/init.d/$APP {start|restart|stop}"
        exit 1
        ;;
    esac

    exit 0

Add access rights:

    chmod \+x /etc/init.d/bamboo

Add symlinks

    sudo update-rc.d bamboo defaults
    
## Troubleshooting

- You can check the logs in [bamboo-install-dir]/logs
- Test if the user have correct access rights to folders
- Run bamboo manually [bamboo-install-dir]bin/start-bamboo.sh

## Configure Instance

Navigate to http://localhost:8085

Configure with an external database: jdbc:postgresql://localhost:5432/bamboodb