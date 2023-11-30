---
layout: post
title: "Creating a RabbitMQ test setup with vagrant"
date: 2012-01-18 16:00
comments: true
published: yes
tags: tools testing rabbitmq
---

In this post I'll provide a writeup on how I created a test setup for
a project I'm working on.  The project uses [RabbitMQ] [1] to distribute
tasks to a worker, and the production system is a cluster of Apache/Tomcat
machines.

I'll use [vagrant] [2] to create two test machines, one which runs [RabbitMQ] [1], the other
one will run the actual worker code.

<!-- more -->

Preparation
-----------

I'll have the [vagrant] [2] files in a subdirectory `vagrant` in my git repository.  I'll
add the `Vagrantfile` and all provisioning scriptzs there::

```bash
    $ mkdir vagrant
```

The `Vagrantfile` looks like this::

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby ts=2 sw=2 expandtab:

Vagrant::Config.run do |config|

  box = "ubuntu-natty-64"
  config.vm.define :rabbitmq do |rabbit_config|
    #rabbit_config.vm.boot_mode = :gui
    rabbit_config.vm.box = box
    rabbit_config.vm.network("192.168.64.10")
    rabbit_config.vm.provision :shell, :path => "rabbitmq.sh"
    rabbit_config.vm.customize do |vm|
      vm.name = "nexiles - rabbitmq"
    end
  end

  config.vm.define :worker do |worker_config|
    worker_config.vm.box = box
    #worker_config.vm.boot_mode = :gui
    worker_config.vm.network("192.168.64.20")
    worker_config.vm.provision :shell, :path => "worker.sh"
    worker_config.vm.share_folder("src", "/src", "~/develop/nexiles/worker/src")
    worker_config.vm.customize do |vm|
      vm.name = "nexiles - worker"
    end
  end

end
```

The configuration sets the machines up, such that:

- both have a IP to talk to eachother
- on the rabbitmq machine I've used the shell provisioner to install
  [RabbitMQ] [1] 2.7.1
- on the worker machine I've used the shell provisioner to create a python
  development environment in /opt
- share my source folder on the worker

Note: For some reason I wanted to have all the [vagrant] [2] stuff in a separate directory.  I
**could** have put the Vagrantfile in the top-level directory and thus saved me the additional
shared folder.  Oh well.

Here are the provisioning shell scripts::

```bash
cat >> /etc/apt/sources.list <<EOT
deb http://www.rabbitmq.com/debian/ testing main
EOT

wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
apt-key add rabbitmq-signing-key-public.asc

apt-get update

apt-get install -q -y screen htop vim curl wget
apt-get install -q -y rabbitmq-server

# RabbitMQ Plugins
service rabbitmq-server stop
rabbitmq-plugins enable rabbitmq_management
rabbitmq-plugins enable rabbitmq_jsonrpc
service rabbitmq-server start

rabbitmq-plugins list


#wget http://www.rabbitmq.com/releases/rabbitmq-server/v2.7.1/rabbitmq-server_2.7.1-1_all.deb
#dpkg install rabbitmq-server_2.7.1-1_all.deb
```

```ruby
export NX_HOME=/opt/nexiles

apt-get install -q -y screen htop curl wget vim

mkdir -p $NX_HOME/{bin,downloads}
cat > /etc/profile.d/nexiles.sh << EOT
export NX_HOME=$NX_HOME
test -f "\$NX_HOME/env.sh" && source \$NX_HOME/env.sh
EOT

# python environment
apt-get install -q -y python2.6
cd $NX_HOME
export PYTHON=/usr/bin/python2.6
export PIP=/usr/local/bin/pip
curl -s http://peak.telecommunity.com/dist/ez_setup.py > $NX_HOME/downloads/ez_setup.py
$PYTHON downloads/ez_setup.py -U

easy_install-2.6 pip
$PIP install virtualenv

virtualenv -p /usr/bin/python2.6 --clear $NX_HOME

. $NX_HOME/bin/activate
pip install requests
pip install kombu==1.5.1

# fix permissions
chown -R vagrant $NX_HOME
```

Usage
-----

To use the system I now just need to do a::

```bash
    $ vagrant up
```

And I'm set.


[1]: http://www.rabbitmq.com/ 		"RabbitMQ"
[2]: http://vagrantup.com/ 		"vagrant"
[3]: https://www.virtualbox.org/	"VirtualBox"

