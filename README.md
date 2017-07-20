# Intro

The purpose of this guide is to get you started with your own Bro and ELK system. These are the steps I followed to set up my own demo system for the presentation.

## Prereqs

You'll need to have a server with two network interfaces, and a switch with a mirror port to monitor. 
You can also use a tap if you have one with a single output, or an aggregator. 
Install Ubuntu Server 16.04, preferably the 64 bit edition.
[Download the ISO here](https://www.ubuntu.com/download/server)
I did an all defaults install, with the exception of adding the SSH server package.
Connect your mirror port to your second network interface and bring it up with `ifconfig $INTERFACE up` (Sub in your interface name for $INTERFACE)
I assume you're root for all of this. Use `sudo su` get root after you log in.

## Bro

### Download Bro

Download and extract the latest version of Bro.

```bash
wget https://www.bro.org/downloads/bro-2.5.1.tar.gz
tar xvf bro-2.5.1.tar.gz
cd bro-2.5.1
```

### Install Deps

Install the packages we need to build bro.

```bash
apt-get install -y cmake make gcc g++ flex bison libpcap-dev libssl-dev python-dev swig zlib1g-dev
```

### Build & Install

This step will take a while. Go strech your legs during the make step.

```bash
./configure
make
make install
```

### Configure Bro

You'll need to set your interface in Set interface in `/usr/local/bro/etc/node.cfg`

You'll also need to add `@load tuning/json-logs` to the end of `/usr/local/bro/share/bro/site/local.bro`

### Start Bro
```bash
/usr/local/bro/bin/broctl deploy
```

## The Elastic Stack

### Install Java

```bash
apt-get install -y openjdk-8-jre-headless
```

### Install ElasticSearch, Logstash, Kibana

#### Install their signing key
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
```

### Install Everything

```bash
echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-5.x.list
apt-get update && apt-get install -y elasticsearch logstash kibana
```

### Configure Kibana

You'll need to configure Kibana to listen on all interfaces by adding or changing this line `server.host: "0.0.0.0"` in `/etc/kibana/kibana.yml`

### Start Services

```bash
service elasticsearch Start
service kibana start
```

### Configure Logstash

Put the following code into `/etc/logstash/conf.d/logstash.conf`

```
input {
  file {
    start_position => 'beginning'
    path => '/usr/local/bro/logs/current/*.log'
    codec => 'json'
  }
}

filter {
  date {
    match => ['ts', 'UNIX']
  }
  geoip {
    source => 'id.orig_h'
    target => 'geo-source'
  }
  geoip {
    source => 'id.resp_h'
    target => 'geo-dest'
  }
}

output {
  elasticsearch {
  }
}
```

### Start Logstash

```bash
service logstash start
```

### Get started!

Connect to `http://server.ip:5601/`

The first time you connect you'll need to configure your index pattern. If everything was set up correctly you'll just need to accept the defaults and hit `Create`
