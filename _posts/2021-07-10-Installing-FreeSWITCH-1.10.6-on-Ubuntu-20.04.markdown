---
layout: post
title:  "Installing FreeSWITCH 1.10.5 on Ubuntu 18.04 | Ubuntu 20.04 LTS"
date:   2020-07-10 23:18:30 +0500
categories: freeswitch voip telephony
---
    
### Introduction
FreeSWITCH is a software defined telecom stack that runs on any commodity hardware. FreeSWITCH can handle voice, video and text communication and support all popullar VoIP protocols. FreeSWITCH is flexible and modular, and can be used in any way you can imagine

This guide demonstrates how to get it install FreeSWITCH and get it up and running on a Ubuntu 20.04 LTS machine

### Prerequisites
To follow along with this guide, you need one Ubuntu 20.04 LTS server which has prerequisite packages installed and configured. In order to install required packages issue following command

###### Ubuntu 18.04 LTS
```bash
$ sudo apt install --yes build-essential pkg-config uuid-dev zlib1g-dev libjpeg-dev libsqlite3-dev libcurl4-openssl-dev \
            libpcre3-dev libspeexdsp-dev libldns-dev libedit-dev libtiff5-dev yasm libopus-dev libsndfile1-dev unzip \
            libavformat-dev libswscale-dev libavresample-dev liblua5.2-dev liblua5.2 cmake libpq-dev \
            unixodbc-dev autoconf automake ntpdate libxml2-dev libpq-dev libpq5 sngrep
```
###### Ubuntu 20.04 LTS
```bash
$ sudo apt install --yes build-essential pkg-config uuid-dev zlib1g-dev libjpeg-dev libsqlite3-dev libcurl4-openssl-dev \
            libpcre3-dev libspeexdsp-dev libldns-dev libedit-dev libtiff5-dev yasm libopus-dev libsndfile1-dev unzip \
            libavformat-dev libswscale-dev libavresample-dev liblua5.2-dev liblua5.2-0 cmake libpq-dev \
            unixodbc-dev autoconf automake ntpdate libxml2-dev libpq-dev libpq5 sngrep
```
Starting from FreeSWITCH version 1.10.4 You have to Download, Compile and install `sofia-sip` and `spandsp` libraries separately

In order to install Sofia-Sip library, you need to download the latest code from FreeSWITCH’s official Packages repository

##### Installing Sofia-Sip library
Clone the official Sofia-Sip repository into `/usr/local/src` directory

```bash
$ sudo git clone https://github.com/freeswitch/sofia-sip /usr/local/src/sofia-sip
```

Now run following commands in sequence to install the library

```bash
$ cd /usr/local/src/sofia-ip
$ sudo ./bootstrap.sh 
$ sudo ./configure
$ sudo make && make install
```
To verify if Sofia-Sip library is installed correctly in your system, run following command
```
$ ldconfig -p | grep sofia
```
If `libsofia-sip` is not installed, there will be no output. If it is installed, you will get a line for each version available.

##### Installing SpanDSP library
Clone the SpanDSP repository from FreeSWITCH packages repository into `/usr/local/src` directory
```bash
$ sudo git clone https://github.com/freeswitch/spandsp /usr/local/src/spandsp
```
Now run following commands in sequence to install the library
```bash
$ cd /usr/local/src/spandsp
$ sudo ./bootstrap.sh 
$ sudo ./configure
$ sudo make && make install
```
To verify if SpanDSP library is installed correctly in your system, run following command
```bash
$ ldconfig -p | grep spandsp
```
If `libspandsp` is not installed, there will be no output. If it is installed, you will get a line for each version available.

You are now ready to install FreeSWITCH

#### Installing FreeSWITCH
Download the FreeSWITCH 1.10.5 release file into `/usr/local/src` directory
```bash
$ sudo wget -c https://files.freeswitch.org/releases/freeswitch/freeswitch-1.10.5.-release.tar.gz -P /usr/local/src 
```
Extract the release file
```bash
$ cd /usr/local/src
$ sudo tar -zxvf freeswitch-1.10.5.-release.tar.gz
$ cd freeswitch-1.10.5.-release
````
Run the configure script
```
$ sudo ./configure 
```
> **_Note:_** FreeSWITCH uses SQLite by default for it’s core database although support for other database options Like PostgreSQL, ODBC exists. If you want to enable Postgres or ODBC support than you need to run the `./configure` script with following arguments

```bash
$ sudo ./configure --enable-core-odbc-support --enable-core-pgsql-support 
```
Now you are ready to compile and install the FreeSWITCH, run following commands in sequence
```bash
$ sudo make 
$ sudo make install 
```
To install sound and music on hold run following command
```bash
$ sudo make cd-sounds-install
$ sudo make cd-moh-install 
```
FreeSWITCH is now installed and you can confirm it by running `sudo freeswitch -nonat` from your terminal

#### Post install setup
###### Create Symlinks (Optional)
By default, FreeSWITCH will install it’s binaries and configurations in `/usr/local/bin` and `/usr/local/freeswitch`, to make them available system wide you can create following symlinks
```bash
$ sudo ln -s /usr/local/freeswitch/conf /etc/freeswitch 
$ sudo ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin/fs_cli 
$ sudo ln -s /usr/local/freeswitch/bin/freeswitch /usr/sbin/freeswitch
```
###### Create an unprivileged user
Create a unprivileged system user for running FreeSWITCH daemon
```bash
$ sudo groupadd freeswitch 
$ sudo adduser --quiet --system --home /usr/local/freeswitch --gecos 'FreeSWITCH open source softswitch' --ingroup freeswitch freeswitch --disabled-password 
$ sudo chown -R freeswitch:freeswitch /usr/local/freeswitch/ 
$ sudo chmod -R ug=rwX,o= /usr/local/freeswitch/ chmod -R u=rwx,g=rx /usr/local/freeswitch/bin/*
```
###### Running as systemd service
In order to run FreeSWITCH in background using systemctl, open `/etc/systemd/system/freeswitch.service` in your favorite editor and copy following content into it

```bash
[Unit] 
Description=FreeSWITCH open source softswitch 
Wants=network-online.target Requires=network.target local-fs.target 
After=network.target network-online.target local-fs.target 

[Service] 
; service 
Type=forking 
PIDFile=/usr/local/freeswitch/run/freeswitch.pid 
Environment="DAEMON_OPTS=-nonat" 
Environment="USER=freeswitch" 
Environment="GROUP=freeswitch" 
EnvironmentFile=-/etc/default/freeswitch 
ExecStartPre=/bin/chown -R ${USER}:${GROUP} /usr/local/freeswitch 
ExecStart=/usr/local/freeswitch/bin/freeswitch -u ${USER} -g ${GROUP} -ncwait ${DAEMON_OPTS} 
TimeoutSec=45s 
Restart=always 

[Install] 
WantedBy=multi-user.target
```
Reload the systemctl daemon

```bash
$ sudo systemctl daemon-reload
```
Start the FreeSWITCH Service

```bash
$ sudo systemctl start freeswitch
```
Check if daemon has start successfully

```bash
$ sudo systemctl status freeswitch
```
###### Command Line interface
The `fs_cli` program is a Command-Line Interface that allows a user to connect to a running FreeSWITCH™ instance. The `fs_cli` program can connect to the FreeSWITCH™ process on the local machine or on a remote system. (Network connectivity to the remote system is, of course, required.)
