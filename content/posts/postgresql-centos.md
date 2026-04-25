---
layout: post
title: PostgreSQL Centos 7
---

Installation friendly for a JIRA or Bamboo installation afterwards. 

## Install

    sudo yum install postgresql-server postgresql-contrib

## Init

    sudo postgresql-setup initdb

## Password authentication

By default, PostgreSQL does not allow password authentication.

    sudo vi /var/lib/pgsql/data/pg_hba.conf

Find the lines that looks like this, near the bottom of the file:

    pg_hba.conf excerpt (original)
    host    all             all             127.0.0.1/32            ident
    host    all             all             ::1/128                 ident

Then replace "ident" with "md5", so they look like this:

pg_hba.conf excerpt (updated)

    # IPv4 local connections:
    
    host    all             all             127.0.0.1/32            md5  
    host    all             all             all                     trust   
    
    # IPv6 local connections:
    
    host    all             all             ::1/128                 md5  

Now start and enable PostgreSQL:

    sudo systemctl start postgresql
    sudo systemctl enable postgresql

## Testing the connection

    sudo netstat -plunt

It should appear a line like: 

    tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      1295/postgres   

The installation procedure created a user account called postgres

    sudo -i -u postgres
    psql
    \q (to exit)

## Remote Connections

You will need to configure your `data/postgresql.conf` and `data/pg_hba.conf` files to accept remote TCP connections from your Bamboo server's IP address.

    sudo -i -u postgres
    psql
    SHOW config_file;

It will show the path to the config file, like: `/var/lib/pgsql/data/postgresql.conf`

    sudo vi /var/lib/pgsql/data/postgresql.conf

## Create a Database (Bamboo for example)

    sudo -s -H -u postgres
    createuser -S -d -r -P -E bamboouser
    createdb -O bamboouser bamboo

Testing the connection

    sudo adduser bamboouser
    sudo -i -u bamboouser
    psql -d bamboo

## Links

[https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-7]()
[https://confluence.atlassian.com/bamboo/postgresql-289276816.html]()