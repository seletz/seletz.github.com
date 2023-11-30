---
layout: post
title: "Creating Vagrant Base Boxes With veewee"
date: 2012-01-17 08:26
comments: true
tags: tools testing
---

In this post I show how to install and use [veewee] [1] to create a new
base box for [vagrant] [2].  Then I'll show how to use this new base box to
create a [vagrant] [2] setup.

<!-- more -->

Installing to a ruby rvm gemset
-------------------------------


Disclaimer: I'm **completely** new to `rvm` and all.  So please bear with
me.

I've used the [rvm documentation] [3] to get this going (btw, awesome domain name).

So it seems that `rvm` is some kind of ruby's `virtualenv`.  Let's setup a
new environment using ruby 1.9.2, and then create a `gemset` which we'll
use to install [veewee] [1] with :

```bash
$ rvm use 1.9.2
$ rvm gemset create veewee
$ rvm 1.9.2@veewee
$ gem install veewee
$ which vagrant
/Users/seletz/.rvm/gems/ruby-1.9.2-p290@veewee192/bin/vagrant
```

**EDIT: the first version of this post had an error in the above
statements.  Thanks, oliver!**

So, whenever I want to use [vagrant] [2] with the [veewee] [1] extension, I need to
do a:

```bash
$ rvm 1.9.2@veewee
```

OK, fine.

Testing
-------

[veewee] [1] promises that it can build [vagrant] [2] base boxes from *scratch*
using templates.  Let's test this and select a *ubuntu 11.04 64 bit
"natty"* template:

```bash
$ vagrant basebox define 'myubuntubox' 'ubuntu-11.04-server-amd64'
$ tree
.
└── definitions
    └── myubuntubox
        ├── definition.rb
        ├── postinstall.sh
        └── preseed.cfg

2 directories, 3 files
```

[veewee] [1] created a directory `definitions` for us, which is basically a
copy of the template we selected.  We *could* change the definitions, but
we'll do that later -- I'll just want to see if the basic process works.

Now let's actually *build* the VM image which will eventually become our
new [vagrant] [2] base box.  We did not change the definitions, and we did'nt
specify a *ISO* image, and thus [veewee] [1] will download it to a `iso`
directory for us:

```bash
$ vagrant basebox build 'myubuntubox'
```

It's quite spectacular -- the image will be booted and [veewee] [1] will use
VBoxManage to send keystrokes to the VM -- defining initial boot parameters
etc.  The net result is, that `veewe` will do a complete *headless*
install of the specified template (a ubuntu "natty" 64 bit server image in
this test) and install all the dependencies needed for the image to act as
a [vagrant] [2] box.

Let's verify the created image using the `verify` command:

```bash
$ vagrant basebox validate 'myubuntubox'
...
Failing Scenarios:
cucumber /Users/seletz/.rvm/gems/ruby-1.9.2-p290@veewee192/gems/veewee-0.2.2/lib/veewee/../../validation/vagrant.feature:33 # Scenario: Checking chef

7 scenarios (1 failed, 6 passed)
21 steps (1 failed, 20 passed)
0m2.462s
```

Cool -- they use ruby specs to actually test the image.  I've got one
failing spec regarding `chef` -- I don't plan to se `chef` for now,
so I'll ignore this.

Up until now, all we have is a VirtualBox VM.  Now let's export the VM
to an actual [vagrant] [2] box file:

```bash
$ vagrant basebox export 'myubuntubox'
Vagrant requires the box to be shutdown, before it can export
Sudo also needs to work for user vagrant
Performing a clean shutdown now.
Executing command: sudo shutdown -P now
...
Machine myubuntubox is powered off cleanly
Executing vagrant voodoo:
vagrant package --base 'myubuntubox' --output 'myubuntubox.box'

To import it into vagrant type:
vagrant box add 'myubuntubox' 'myubuntubox.box'

To use it:
vagrant init 'myubuntubox'
vagrant up
vagrant ssh
```
    
Cool, it did create a box file:

```bash
$ ll -h
total 338M
drwxr-xr-x 3 seletz staff  102 Jan 16 18:45 definitions
drwxr-xr-x 3 seletz staff  102 Jan 16 18:48 iso
-rw-r--r-- 1 seletz staff 338M Jan 16 20:20 myubuntubox.box
$ file myubuntubox.box
myubuntubox.box: POSIX tar archive
$ tar tvf myubuntubox.box
-rw-------  0 501    20  353481216 16 Jan 20:20 box-disk1.vmdk
-rw-------  0 501    20        121 16 Jan 20:20 box.mf
-rw-------  0 501    20      13271 16 Jan 20:19 box.ovf
-rw-r--r--  0 501    20        505 16 Jan 20:20 Vagrantfile
```
    
See? the box file is actually a tarball of the exported VM, the root disk
and some meta data.

But we *still* can't use that file.  We first need to *add* the box to our
vagrant setup.  We could upload that box file to a local HTTP server or
something like that.  The `vagrant add` command takes a URL or a path.

OK, let's add the box, so that we can use it to create [vagrant] [2] machines:

```bash
$ vagrant box add 'myubuntubox' 'myubuntubox.box'
[vagrant] Downloading with Vagrant::Downloaders::File...
[vagrant] Copying box to temporary location...
[vagrant] Extracting box...
[vagrant] Verifying box...
[vagrant] Cleaning up downloaded box...
$ tree -h --du ~/.vagrant.d
/Users/seletz/.vagrant.d
├── [979M]  boxes
│   ├── [383M]  base
│   │   ├── [ 505]  Vagrantfile
│   │   ├── [383M]  box-disk1.vmdk
│   │   ├── [ 121]  box.mf
│   │   └── [ 14K]  box.ovf
│   ├── [259M]  lucid32
│   │   ├── [ 505]  Vagrantfile
│   │   ├── [259M]  box-disk1.vmdk
│   │   ├── [ 121]  box.mf
│   │   └── [ 13K]  box.ovf
│   └── [337M]  myubuntubox
│       ├── [ 505]  Vagrantfile
│       ├── [337M]  box-disk1.vmdk
│       ├── [ 121]  box.mf
│       └── [ 13K]  box.ovf
├── [  68]  logs
└── [ 102]  tmp
    └── [   0]  vagrant.lock

 979M used in 6 directories, 13 files
```

OK, all that did was to untar the tarball to `~/.vagrant.d/boxes/myubuntubox`.

Let's create a test vagrant setup in the `tst` directory:

```bash
$ mkdir tst
$ vagrant init myubuntubox
  create  Vagrantfile
$ vagrant up
[default] Importing base box 'myubuntubox'...
[default] Matching MAC address for NAT networking...
[default] Clearing any previously set forwarded ports...
[default] Forwarding ports...
[default] -- ssh: 22 => 2222 (adapter 1)
[default] Creating shared folders metadata...
[default] Running any VM customizations...
[default] Booting VM...
[default] Waiting for VM to boot. This can take a few minutes.
[default] VM booted and ready for use!
[default] Mounting shared folders...
[default] -- v-root: /vagrant
```
    
Wohooo!  Look's like we're in business!

[vagrant] [2] created a *new* VM based on the *myubuntubox*, fired it up,
configured a host-only network device, and configured port-forwarding such
that we have SSH access.  Other than the port `2222` being forwarded the
machine is completely isolated.  The VM is also started in *headless* mode.

All the above is specified in a configuration file `Vagrantfile`, which the
command `vagrant init` did create:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby ts=2 sw=2 expandtab:

Vagrant::Config.run do |config|
  config.vm.box = "myubuntubox"
end
```
    
OK, I lied -- the actuall file is quite a bit longer -- but I just removed
*comments*.

Now let'ls login, shall we? :

```bash
$ vagrant ssh
Welcome to Ubuntu 11.04 (GNU/Linux 2.6.38-8-server x86_64)

 * Documentation:  http://www.ubuntu.com/server/doc
New release 'oneiric' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Mon Jan 16 19:01:18 2012 from 10.0.2.2
vagrant@myubuntubox:~$ sudo id
uid=0(root) gid=0(root) groups=0(root)
vagrant@myubuntubox:~$ mount
/dev/mapper/myubuntubox-root on / type ext4 (rw,errors=remount-ro)
proc on /proc type proc (rw,noexec,nosuid,nodev)
none on /sys type sysfs (rw,noexec,nosuid,nodev)
fusectl on /sys/fs/fuse/connections type fusectl (rw)
none on /sys/kernel/debug type debugfs (rw)
none on /sys/kernel/security type securityfs (rw)
none on /dev type devtmpfs (rw,mode=0755)
none on /dev/pts type devpts (rw,noexec,nosuid,gid=5,mode=0620)
none on /dev/shm type tmpfs (rw,nosuid,nodev)
none on /var/run type tmpfs (rw,nosuid,mode=0755)
none on /var/lock type tmpfs (rw,noexec,nosuid,nodev)
/dev/sda1 on /boot type ext2 (rw)
v-root on /vagrant type vboxsf (uid=1000,gid=1000,rw)
```

Wow.  Sweet.

A quite few things are of interest here:

- we're able to ssh in w/o specifying a password.  [vagrant] [2] uses
  public key authentication, and per default uses an private key which
  lies inside the [vagrant] [2] gem.

- we're able to do a `sudo` w/o specifying a password.

- There's a `/vagrant` directory mounted.

The [vagrant] [2] mount point is actually a VirtualBox shared folder, and is
connected to our local `tst` folder, the one which contains the generated
`Vagrantfile`!  This will come in quite handy later when we're start to
provision things.

Now lets look at resuming starting, stopping and so on:

```bash
$ vagrant suspend
[default] Saving VM state and suspending execution...
$ vagrant resume
[default] Resuming suspended VM...
[default] Booting VM...
[default] Waiting for VM to boot. This can take a few minutes.
[default] VM booted and ready for use!
$ vagrant halt
[default] Attempting graceful shutdown of linux...
$ vagrant up
[default] VM already created. Booting if it's not already running...
[default] Clearing any previously set forwarded ports...
[default] Forwarding ports...
[default] -- ssh: 22 => 2222 (adapter 1)
[default] Cleaning previously set shared folders...
[default] Creating shared folders metadata...
[default] Running any VM customizations...
[default] Booting VM...
[default] Waiting for VM to boot. This can take a few minutes.
[default] VM booted and ready for use!
[default] Mounting shared folders...
[default] -- v-root: /vagrant
```

OK, let's recap.

- Using [veewee] [1] one can create a [vagrant] [2] base box with very minimal
  effort.  All the gory details are handled for us:

  - downloading the ISO file

  - A headless install of the base box

  - Installation of all the [vagrant] [2] dependencies into that VM

  - Installation of the VirtualBox Guest Additions in the correct, matching
    version.

  - we can choose amongst a vast number of templates (64 templates as of
    2012-01-16)

    **There are even WINDOWS templates**

  - we can probably customize and tweak templates

- Using [vagrant] [2] we can create *new* machines at will, with very minimal
  configuration

- We only ever need to check in the `Vagrantfile` to our source repository,
  and all developers can just do a `vagrant up` and have their machines for
  testing up and running

Not bad for a first test.  Not bad at all.

Mutli-VM setup
--------------

OK, now let's use the new base box and create a custom `Vagrantfile` which
specifies *two* VMs.  We'll have to name them differently, we'll use
`rabbitmq` and `worker` (that's what I want to have in the end: a machine
for RabbitMQ and a machine for a custom worker which uses RabbitMQ).

We'll create a new directory for this test:

```bash
$ mkdir multivm
$ cd multivm
```

The `Vagrantfile` is simple:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby ts=2 sw=2 expandtab:

Vagrant::Config.run do |config|
  config.vm.define :rabbitmq do |rabbitmq|
    rabbitmq.vm.box = "myubuntubox"
    rabbitmq.vm.network "192.168.50.10"
  end

  config.vm.define :worker do |worker|
    worker.vm.box = "myubuntubox"
    worker.vm.network "192.168.50.11"
  end
end
```

We have configured two machines using the same 'myubuntubox' base box.  We
also have configured a **additional** network device on both such that
they're able to communicate with eachother.

Fire it up:

```bash
$ vagrant up
[rabbitmq] Importing base box 'myubuntubox'...
[rabbitmq] Preparing host only network...
[rabbitmq] Creating new host only network for environment...
[rabbitmq] Matching MAC address for NAT networking...
[rabbitmq] Clearing any previously set forwarded ports...
[rabbitmq] Forwarding ports...
[rabbitmq] -- ssh: 22 => 2222 (adapter 1)
[rabbitmq] Creating shared folders metadata...
[rabbitmq] Running any VM customizations...
[rabbitmq] Booting VM...
[rabbitmq] Waiting for VM to boot. This can take a few minutes.
[rabbitmq] VM booted and ready for use!
[rabbitmq] Enabling host only network...
[rabbitmq] Mounting shared folders...
[rabbitmq] -- v-root: /vagrant
[worker] Fixed port collision 'ssh'. Now on port 2200.
[worker] Importing base box 'myubuntubox'...
[worker] Preparing host only network...
[worker] Matching MAC address for NAT networking...
[worker] Clearing any previously set forwarded ports...
[worker] Forwarding ports...
[worker] -- ssh: 22 => 2200 (adapter 1)
[worker] Creating shared folders metadata...
[worker] Running any VM customizations...
[worker] Booting VM...
[worker] Waiting for VM to boot. This can take a few minutes.
[worker] VM booted and ready for use!
[worker] Enabling host only network...
[worker] Mounting shared folders...
[worker] -- v-root: /vagrant
```

We can now ssh in to each machine:

```bash
$ vagrant ssh rabbitmq
...
$ vagrant ssh worker
...
```

OK, but what about installing some base packages after a `vagrant up`?
Well, that's what *provisioning* is for.


Provisioning
------------

OK, let's use the [vagrant] [2] shell provisiooning to install `RabbitMQ`.  For
this, all we need to do is to create a shell script and place it next to
the `Vagrantfile` and tell [vagrant] [2] about it.  When doing a `vagrant up`,
the script will be executed and we're set.

This is the script:

```bash
$ cat rabbitmq.sh
apt-get install -q -y rabbitmq-server
apt-get install -q -y iotop htop
```

And this is the only change to the `Vagrantfile` from the previous step:

```diff
diff --git a/Vagrantfile b/Vagrantfile
index b0c5528..77ec500 100644
--- a/Vagrantfile
+++ b/Vagrantfile
@@ -5,6 +5,7 @@ Vagrant::Config.run do |config|
   config.vm.define :rabbitmq do |rabbitmq|
     rabbitmq.vm.box = "myubuntubox"
     rabbitmq.vm.network "192.168.50.10"
+    rabbitmq.vm.provision :shell, :path => "rabbitmq.sh"
   end
```

Now actually run the provisioning step.  Note that the VM is already
running:

```bash
$ vagrant provision rabbitmq
[rabbitmq] Running provisioner: Vagrant::Provisioners::Shell...
[rabbitmq] stdin: is not a tty
[rabbitmq] Reading package lists...
[rabbitmq] 
[rabbitmq] Building dependency tree...
[rabbitmq] 
[rabbitmq] Reading state information...
[rabbitmq] 
...
[rabbitmq] Processing triggers for man-db ...
[rabbitmq] Setting up htop (0.9-2) ...
[rabbitmq] Setting up iotop (0.4-2) ...
[rabbitmq] Processing triggers for python-support ...
```

That's really all there is to it!

Recap:

- We were able to install a multi-machine, isolated VM setup for test and
  development using 15 lines of configuration

- The *provisioning* step is just telling [vagrant] [2] the name of the shell
  script to execute

- The scripts used in provisioning were located next to the `Vagrantfile`
  (i.e *not* inside the VM), and thus can be checked in.

Conclusion
----------

I think [vagrant] [2] is great.  And [veewee] [1] allows for hassle-free base boxes
to be set up, so no more downloading from untrusted sources!

I'll definitely continue to integrate these systems in my task to get a
more complete test setup, including a headless CI setup with multiple
machines.  Together with something like `phantom.js` we can even create
screen shots in headless mode, such that we can peek at them errors ...

[1]: http://github.com/jedi4ever/veewee 			 "veewee"
[2]: http://www.vagrantup.com 					 "vagrant"
[3]: https://rvm.beginrescueend.com/                             "rvm documentation"
