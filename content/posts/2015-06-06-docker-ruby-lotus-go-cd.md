+++
author = "hannibal"
categories = ["devops", "java", "Linux"]
date = "2015-06-06"
tags = ["docker", "gocd", "vagrant"]
title = "Docker + Java + Vagrant+ GO.CD"
url = "/2015/06/06/docker-ruby-lotus-go-cd/"

+++

Hello folks.

Today, I would like to write about something interesting and close to me at the moment. I'm going to setup Go.cd with Docker, and I'm going to get a Ruby Lotus app running. Let's get started.

<!--more-->

# Fluff

Now, obviously, you don't really need Go.Cd or Docker to setup a Java Gradle application, since it's dead easy. But I'm going to do it just for the heck of it.

# Setup

Okay, lets start with Vagrant. Docker's strength is coming from Linux's process isolation capabilities it's not yet properly working on OSX or Windows. You have a couple of options if you'd like to try never the less, like boot2docker, or a Tiny Linux kernel, but at that point, I think it's easier to use a VM.

#### Vagrant

So, let's start with my small Vagrantfile.

~~~ruby

# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "trusty"
  config.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box"
  config.vm.network "forwarded_port", guest: 2300, host: 2300
  config.vm.network "forwarded_port", guest: 8153, host: 8153
  config.vm.provision "shell", path: "setup.sh"
  config.vm.provider "virtualbox" do |v|
    v.memory = 8192
    v.cpus = 2
  end
end
~~~

Very simple. I'm setting up a trusty64(because docker requires 3.10 <= x) box and then doing a simple shell provision. Also, I gave it a bit juice, since go-server requires a raw power. Here is the shell script:

~~~bash

!#/bin/bash
sudo apt-get update
sudo apt-get install -y software-properties-common python-software-properties
sudo apt-get update
sudo apt-get install -y vim
sudo add-apt-repository -y "ppa:webupd8team/java"
sudo apt-get update
echo debconf shared/accepted-oracle-license-v1-1 select true | sudo debconf-set-selections &amp;&amp; echo debconf shared/accepted-oracle-license-v1-1 seen true | sudo debconf-set-selections
sudo apt-get install -y oracle-java8-installer
sudo apt-get update
wget -qO- https://get.docker.com/ | sh
route add -net 172.17.0.0 netmask 255.255.255.0 gw 172.17.42.1
~~~

The debconf at the end accepts java8's terms and conditions. And the last line installs docker in my box. This runs for a little while.

The routing on the end routes every traffic from 172.17.\*.\* to my vagrant box, which in turn I'll be able to use from my mac local, like 127.0.0.1:8153/go/home.

After a vagrant up, my box is ready to be used.

#### Docker

When that's finished, we can move on to the next part, which is writing a little Dockerfile for our image. Go.cd will require java and a couple of other things, so let's automate the installation of that so we don't have to do it by hand.

Here is a Dockerfile I came up with:

~~~Dockerfile

FROM ubuntu
MAINTAINER Skarlso

############ SETUP #############
RUN apt-get update
RUN apt-get install -y software-properties-common python-software-properties
RUN add-apt-repository -y "ppa:webupd8team/java"
RUN echo debconf shared/accepted-oracle-license-v1-1 select true | sudo debconf-set-selections &amp;&amp; echo debconf shared/accepted-oracle-license-v1-1 seen true | sudo debconf-set-selections
RUN apt-get update
RUN apt-get install -y oracle-java8-installer
RUN apt-get install -y vim
RUN apt-get install -y unzip
RUN apt-get install -y git
~~~

So, our docker images have to be setup with Java as well for go.cd which I'm taking care of here, and a little bit extra, to add vim, and unzip, which is required for dpkg later.

At this point run: **docker build -t ubuntu:go .** -> This will use the dockerfile and create the ubuntu:go image. Note the **. **at the end.

#### Go.cd

Now, I'm creating two containers. One, go-server, will be the go server, and the other, go-agent, will be the go agent.

First, go-server:

~~~

docker run -i -t --name go-server --hostname=go-server -p 8153:8153 ubuntu:go /bin/bash
wget http://download.go.cd/gocd-deb/go-server-15.1.0-1863.deb
dpkg -i go-server-15.1.0-1863.deb
~~~

Pretty straight forward, no? We forward 8153 to vagrant (which forwards it to my mac), so after we start go-server service we should be able to visit: http://127.0.0.1:8153/go/home.

Lo', and behold, go server. Let's add an agent too.

~~~
docker run -i -t --name go-agent --hostname=go-agent ubuntu:go /bin/bash
wget http://download.go.cd/gocd-deb/go-agent-15.1.0-1863.deb
dpkg -i go-agent-15.1.0-1863.deb
vim /etc/default/go-agent
GO_SERVER=172.17.0.1
service go-agent start
~~~

No need to forward anything here. And as you can see, my agent was added successfully.

All nice, and dandy. The agent is there, and I enabled it, so it's ready to work. Let's give it something to do, shall we?

# The App

I'm going to use my gradle project which is on github. This one => https://github.com/Skarlso/DataMung.git.

Very basic setup. Just check it out and then build & run tests. Easy, right?

First step in this process, define the pipeline. I'm going to keep it simple. Name the pipeline DataMunger. Group is Linux. Now, in go.cd you have to define something called, an **environment**. Environment can be anything you want, I'm going to go with Linux. You have to assign **agents** to this environment who fulfil it and the pipeline which will use that environment. More on that you can read in the go.cd documentation. This is how you would handle a pipeline which uses linux, and a windows environment at the same time.

In step one you have to define something called the **Material**. That will be the source on which the agent will work. This can be multiple, in different folders within the confines of the pipeline, or singular.

I defined my git project and tested the connection OK. Next up is the first **Stage **and the initial **Job **to perform. This, for me, will be a compile or an assemble, and later on a test run.

Now, Go is awesome in parallelising jobs. If my project would be large enough, I could have multiple jobs here. But for now, I'll use stages because they run subsequently. So, first stage, compile. Next stage, testing and archiving the results.

I added the next stage and defined the artefact. Go supports test-reports. If you define the path to a test artefact than go will parse it and create a nice report out of it.

Now, let's run it. It will probably fail on something. 😉

Well, I'll be. It worked on the first run.

And here are the test results.

# Wrap-up

Well, that's it folks. Gradle project, with vagrant, docker, and go.cd. I hope you all enjoyed reading about it as much as I did doing it.

Any questions, please feel free to ask it in the comment section below.

Cheers,
Have a nice weekend,
Gergely.
