---
layout: post
title: PostgreSQL Ubuntu
---

PostgreSQL Ubuntu installation. Default port 5432.

## Install

    sudo apt-get update 
    sudo apt-get install postgresql postgresql-contrib
  
## Test Installation
   
    sudo -i -u postgres
    psql  
    \quit

## Example: Create Bamboo Database

    sudo -s -H -u postgres
    createuser -S -d -r -P -E bamboouser
    createdb -O bamboouser bamboodb

Testing the connection

    sudo adduser bamboouser
    sudo -i -u bamboouser
    psql -d bamboodb

## Links
[https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04]()
[https://confluence.atlassian.com/bamboo/postgresql-289276816.html]()