---
layout: post
title:  "Installing Odoo 15 Community Edition on Ubuntu 20.04 LTS"
date:   2020-02-26 01:33:01 +0500
categories: odoo odoo-development python erp crm
---

## Introduction 

This tutorial demonstrate how you can setup a simple SDN network, using the Faucet controller and Open vSwitch

The goal of the tutorial is to demonstrate Open vSwitch and Faucet in an end-to-end way, that is, to show how it works from the Faucet controller configuration at the top, through the OpenFlow flow table, to the datapath processing. Along the way

## Environment 
* Ubuntu 20.04
  
## Preparing for Odoo installation

This section explain how to setup Ubuntu system for the purpose of installing Odoo:

First of all, Clone the Odoo repostory on local system 

```bash
mkdir -p /odoo/odoo15
sudo git clone --depth 1 --branch 15.0 https://www.github.com/odoo/odoo /odoo/odoo15
```

### Create System User
Create a system user for running Odoo process

```bash
sudo adduser --system --home=/opt/odoo --group odoo
```

### Create Log directory
Crate a directory for Odoo logs
```bash       
sudo mkdir /var/log/odoo
```

### Change permissions
Set owner of the Odoo directories to `odoo` by issuing following command:

```bash
sudo chown -R odoo:odoo /var/log/odoo
sudo chown -R odoo:odoo /odoo/odoo15 

```
## Install PostgreSQL Server

Perform following steps to install postgresql server

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt-get update

sudo apt-get -y install postgresql
```

Create the database user:

```bash
sudo su - postgres -c "createuser -s odoo"
```

## Install System Dependencies
Install python and various other development packages using following command

```bash
apt install -y python3 \
python3-pip \
build-essential \
wget \ 
python3-dev \
python3-venv \
python3-wheel \
libxslt-dev \
libzip-dev \
libldap2-dev \
libsasl2-dev \
python3-setuptools \
node-less \
libjpeg-dev \
gdebi \
python3-venv \
python-is-python3 \
libxml2-dev \
libxslt1-dev \
libffi-dev \
libpq-dev
```

## Create a virtual environment (VENV) for Odoo App

Create a virtual environment for installing python packages required by Odoo

```bash
cd /odoo

python -m venv venv
```

Activate it by running following command:

```bash
source /odoo/venv/bin/activate
```

And then install pip packages using following commands

```bash
pip install -r /odoo/odoo15/requirements.txt
```

## Create Odoo Server Config
Create the master configuration using following steps:

```bash

sudo cat <<EOT >> /odoo/server.conf
[options]
admin_passwd = admin
xmlrpc_port = 8069
logfile = /var/log/odoo/odoo-server.log
addons_path=/odoo/odoo15/addons
EOT

sudo chown odoo:odoo /odoo/server.conf
sudo chmod 640 /odoo/server.conf 
```


## Start Odoo Server

```bash
sudo su - odoo -s /bin/bash
source /odoo/venv/bin/activate
/odoo/odoo15/odoo-bin -c /odoo/server.conf --http-interface=0.0.0.0
```