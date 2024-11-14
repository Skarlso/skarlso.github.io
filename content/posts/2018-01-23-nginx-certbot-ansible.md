+++
author = "hannibal"
categories = ["Ansible", "Nginx", "Certbot"]
date = "2018-01-23T22:34:00+01:00"
title = "Ansible + Nginx + LetsEncrypt + Wiki + Nagios"
url = "/2018/01/23/nginx-certbot-ansible"
description = ""
featured = "ansible.svg"
featuredalt = "Ansible"
featuredpath = "date"
linktitle = ""
+++

# Intro

Hi folks.

Today, I would like demonstrate how to use [Ansible](https://www.ansible.com/) in order to construct a server hosting multiple HTTPS domains with [Nginx](https://www.nginx.com/) and [LetsEncrypt](https://letsencrypt.org/). Are you ready? Let's dive in.

## TL;DR

![playbook](/img/ansible.svg)

## What you will need

There is really only one thing you need in order for this to work and that is Ansible. If you would like to run local tests without a remote server, than you will need [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org/wiki/Downloads). But those two are optional.

## What We Are Going To Set Up

The setup is as follows:

### Nagios

We are going to have a Nagios with a custom check for pending security updates. That will run under nagios.example.com.

### Hugo Website

The main web site is going to be a basic [Hugo](https://gohugo.io/) site. Hugo is a static Go based web site generator. This Blog is run by it.

We are also going to setup [NoIP](https://www.noip.com/) which will provide the DNS for the sites.

### Wiki

The wiki is a plain, basic [DokuWiki](https://www.dokuwiki.org/dokuwiki#).

### HTTPS + Nginx

And all the above will be hosted by Nginx with HTTPS provided by letsencrypt. We are going to set all these up with Ansible on top so it will be idempotent.

### Repository

All of the playbooks and the whole thing together can be viewed here: [Github Ansible Server Setup](https://github.com/Skarlso/ansible-server-setup).

## Ansible

I won't be writing everything down to the basics about Ansible. For that you will need to go and read its documentation. But I will provide ample of clarification for using what I'll be using.

### Some Basics

Ansible is a configuration management tool which, unlike chef or puppet, isn't master - slave based. It's using SSH to run a set of instructions on a target machine. The instructions are written in yaml files and look something like this:

~~~yaml
---
# tasks file for ssh
- name: Copy sshd_config
  copy: content="{{sshd_config}}" dest=/etc/ssh/sshd_config
  notify:
  - SSHD Restart
~~~

This is a basic Task which copies over an `sshd_config` file overwriting the one already being there. It can execute in priviliged mode if root password is provided or the user has sudo rights.

It works from so called `hosts` files where the server details are described. This is how a basic host file would look like:

~~~bash
[local]
127.0.0.1

[webserver1]
1.23.4.5
~~~

Ansible will use these settings to try and access the server. To test if the connection is working, you can send a `ping` task like this:

~~~bash
ansible all -m ping
~~~

Ansible uses `variables` for things that change. They are defined under each task's subfolder called `vars`. Please feel free to change the varialbes there to your liking.

### SSH Access

You can either define SSH information per host or per group or globally. In this example I have it under the groups wars called
`webserver1` like this (vars.yaml):

~~~yaml
---
# SSH sudo keys and pass
ansible_become_pass: '{{vault_ansible_become_pass}}'
ansible_ssh_port: '{{vault_ansible_ssh_port}}'
ansible_ssh_user: '{{vault_ansible_ssh_user}}'
ansible_ssh_private_key_file: '{{vault_ansible_ssh_private_key_file}}'
home_dir: /root
~~~

### Further reading

Further readings are:

* [Servers For Hackers](https://serversforhackers.com/c/an-ansible-tutorial)
* [Ansible docs](http://docs.ansible.com/ansible/latest/intro_getting_started.html)

### Vault

The vault is the place where we can keep secure information. This file is called `vault` and usually lives under either `group_vars` or `host_vars`. The preference is up to you.

This file is encrypted using a password you specify. You can have the vault password stored in the following ways:

* Store it on a secure drive which is encrypted and only mounted when the playbook is executed
* Store it on [Keybase](https://keybase.io)
* Store it on an encrypted S3 bucket
* Store it in a file next to the playbook which is never commited into source control

Either way, in the end, ansible will look for a file called `.vault_password` for when it's trying to decrypt the file. You can
define a different file in the `ansible.cfg` file using the `vault_password_file` option.

You can create a vault like this:

~~~bash
ansible-vault create vault
~~~

If you are following along, you are going to need these variables in the vault:

~~~yaml
vault_ansible_become_pass: <your_sudo_password> # if applicable
vault_ansible_ssh_user: <ssh_user>
vault_ansible_ssh_private_key_file: /Users/user/.ssh/ida_rsa
vault_nagios_password: supersecurenagiosadminpassword
vault_nagios_username: nagiosadmin
vault_noip_username: youruser@gmail.com
vault_noip_password: "SuperSecureNoIPPassword"
vault_nginx_user: <localuser>
~~~

You can always edit the vault later on with:

~~~bash
ansible-vault edit group_vars/webserver1/vault --vault-password-file=.vault_pass
~~~

### Tasks

The following are a collection of tasks which execute in order. The end task, which is letsencrypt, relies on all the hosts being present and configured under Nginx. Otherwise it will throw an error that the host you are trying to configure HTTPS for, isn't defined.

#### No-IP

I'm choosing No-ip as a DNS provider because it's cheap and the sync tool is easy to automate. To automate the CLI of No-IP, I'm using a package called `expect`. This looks something like this:

~~~bash
cd {{home_dir}}
wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
mkdir -p noip
tar zxf noip-duc-linux.tar.gz -C noip
cd noip/*
make

/usr/bin/expect <<END_SCRIPT
spawn make install
expect "Please enter the login/email*" { send "{{noip_username}}\r" }
expect "Please enter the password for user*" { send "{{noip_password}}\r" }
expect {
    "Do you wish to have them all updated*" {
        send "y"
        exp_continue
    }
}
expect "Please enter an update interval*" { send "30\r" }
expect "Do you wish to run something at successful update*" {send "N" }
END_SCRIPT
~~~

The interesting part is the command running expect. Basically, it's expecting some kind of output which is outlined there. And has canned answers for those which it `send`s to the waiting command.

#### To Util or Not To Util

So, there are small tasks, like installing vim and wget and such which could warrant the existance of a `utils` task. Utils task would install the packages that are used as convinience and don't really relate to a singe task.

Yet I settled for the following. Each of my tasks has a dependency part. The given tasks takes care of all the packages it needs so they can be executed on their own as well as in unison.

This looks like this:

~~~yaml
# Install dependencies
- name: Install dependencies
  apt: pkg="{{item}}" state=installed
  with_items:
    - "{{deps}}"
~~~

For which the `deps` variable is defined as follows:

~~~yaml
# Defined dependencies for letsencrypt task.
deps: ['git', 'python-dev', 'build-essential', 'libpython-dev', 'libpython2.7', 'augeas-lenses', 'libaugeas0', 'libffi-dev', 'libssl-dev', 'python-virtualenv', 'python3-virtualenv', 'virtualenv']
~~~

This is much cleaner. And if a task is no longer needed, it's dependencies will no longer be needed either in most of the cases.

#### Nagios

I'm using Nagios 4 which is a real pain in the butt to install. Luckily, thanks to Ansiblei, I only ever had to figure it out once. Now I have a script for that. Installing Nagios demands several, smaller components to be installed. Thus our task uses import from outside tasks like this:

~~~yaml
- name: Install Nagios
  block:
    - include: create_users.yml # creates the Nagios user
    - include: install_dependencies.yml # installs Nagios dependencies
    - include: core_install.yml # Installs Nagios Core
    - include: plugin_install.yml # Installs Nagios Plugins
    - include: create_htpasswd.yml # Creates a password for Nagios' admin user
    - include: setup_custom_check.yml # Adds a custom check which is to check how many security updates are pending
  when: st.stat.exists == False
~~~

The `when` is a check for a variable created by a file check.

~~~yaml
- stat:
    path: /usr/local/nagios/bin/nagios
  register: st
~~~

It checks if Nagios is installed or not. If yes, skip.

I'm not going to paste in here all the subtasks because that would be huge. You can check those out in the repository under Nagios.

#### Hugo

Hugo is easy to install. Its sole requirement is Go. To install hugo you simply run `apt-get install hugo`. Setting up the
site for me was just checking out the git repo and than execute hugo from the root folder like this:

~~~bash
hugo server --bind=127.0.0.1 --port=8080 --baseUrl=https://example.com --appendPort=false --logFile hugo.log --verboseLog --verbose -v &
~~~

#### Wiki

I used DokuWiki because it's a file based wiki so installation is basically just downloading the archive, extracting it and done. The only thing that's needed for it, is php-fpm to run it and a few php modules which I'll outline in the ansible playbook.

The VHOST file for DokuWiki is provided by them and looks like this:

~~~bash
server {
    server_name   {{ wiki_server_name }};
    root {{ wiki_root }};
    index index.php index.html index.htm;
    client_max_body_size 2M;
    client_body_buffer_size 128k;
    location / {
        index doku.php;
        try_files $uri $uri/ @dokuwiki;
    }
    location @dokuwiki {
        rewrite ^/_media/(.*) /lib/exe/fetch.php?media=$1 last;
        rewrite ^/_detail/(.*) /lib/exe/detail.php?media=$1 last;
        rewrite ^/_export/([^/]+)/(.*) /doku.php?do=export_$1&id=$2 last;
        rewrite ^/(.*) /doku.php?id=$1 last;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    location ~ /\.ht {
        deny all;
    }
    location ~ /(data|conf|bin|inc)/ {
        deny all;
    }
}
~~~

#### Nginx

Nginx install is through apt as well. Here, however, there is a bit of magic going on with templates. The templates provide the
vhost files for the three hosts we will be running. This looks as follows:

~~~yaml
- name: Install vhosts
  block:
    - template: src=01_example.com.j2 dest=/etc/nginx/vhosts/01_example.com
      notify:
      - Restart Nginx
    - template: src=02_wiki.example.com.j2 dest=/etc/nginx/vhosts/02_wiki_example.com
      notify:
      - Restart Nginx
    - template: src=03_nagios.example.com.j2 dest=/etc/nginx/vhosts/03_nagios.example.com
      notify:
      - Restart Nginx
~~~

Now, you might be wondering what `notify` is? It's basically a handler that gets notified to restart nginx. The great part about
it is that it does this only once, even if it was called multiple times. The handler looks like this:

~~~yaml
- name: Restart Nginx
  service:
    name: nginx
    state: restarted
~~~

And lives under `handlers` sub-folder.

With this, Nginx is done and should be providing our sites under plain HTTP.

#### LetsEncrypt

Now comes the part where we enable HTTPS for all these three domains. Which is as follows:

* example.com
* wiki.example.com
* nagios.example.com

This is actually quiet simple now-a-days with `certbot-auto`. In fact, it will insert the configurations we need all by itself.
The only thing for us to do is to specify what domains we have and what our challenge would be. Also, we have to pass in some
variables for `certbot-auto` to run in a non-interactive mode. This looks as follows:

~~~yaml
- name: Generate Certificate for Domains
  shell: ./certbot-auto --authenticator standalone --installer nginx -d '{{ domain_example }}' -d '{{ domain_wiki }}' -d '{{ domain_nagios }}' --email example@gmail.com --agree-tos -n --no-verify-ssl --pre-hook "sudo systemctl stop nginx" --post-hook "sudo systemctl start nginx" --redirect
  args:
    chdir: /opt/letsencrypt
~~~

And that's that. The interesting and required part here is the `pre-hook` and `post-hook`. Without those it wouldn't work because
the ports that certbot is performing the challenge on would be taken already. This stops nginx, performs the challenge and
generates the certs, and starts nginx again. Also note `--redirect`. This will force HTTPS on the sites and disables plain HTTP.

If all went well our sites should contain information like this:

~~~bash
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/example.com-0001/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/example.com-0001/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
~~~

### Test Run using Vagrant

If you don't want to run all this on a live server to test out, you can do either of these two things:

* Use a remote dedicated test server
* Use a local virtual machine with Vagrant

Here, I'm giving you an option for the later.

It's possible for most of the things to be tested on a local Vagrant machine. Most of the time a Vagrant box is enough to test out installing things. A sample Vagrant file looks like this:

~~~ruby
# encoding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :
# Box / OS
VAGRANT_BOX = 'ubuntu/xenial64'

VM_NAME = 'ansible-practice'

Vagrant.configure(2) do |config|
  # Vagrant box from Hashicorp
  config.vm.box = VAGRANT_BOX
  # Actual machine name
  config.vm.hostname = VM_NAME
  # Set VM name in Virtualbox
  config.vm.provider 'virtualbox' do |v|
    v.name = VM_NAME
    v.memory = 2048
  end
  # Ansible provision
  config.vm.provision 'ansible_local' do |ansible|
    ansible.limit = 'all'
    ansible.inventory_path = 'hosts'
    ansible.playbook = 'local.yml'
  end
end
~~~

This interesting part here is the ansible provision section. It's running a version of Ansible that is called `ansible_local`. It's local, becuase it will be only on the VirtualBox. Meaning, you don't have to have Ansible installed to test it on a vagrant box. Neat, huh?

To test your playbook, simply run `vagrant up` and you should see the provisioning happening.

## Room for improvement

And that should be all. Note that this setup isn't quiet enterprise ready. I would add the following things:

### Tests and Checks

A ton of tests and checks if the commands that we are using are actually successful or not. If they aren't make them report the failure.

### Multiple Domains

If you happen to have a ton of domain names to set up, this will not be the most effective way. Right now letsencrypt creates a
single certificate file for those three domains with `-d` and that's not what you want with potentially hundreds of domains.

In that case, have a list to go through with `with_items`. Note that you'll have to restart nginx on each line, because you don't
want one of them fail and stop the process entirely. Rather have a few fail but the rest still work.

# Conclusion

That's it folks. Have fun setting up servers all over the place and enjoy the power of nginx and letsencrypt and not having to
worry about adding another server into the bunch.

Thank you for reading,
Gergely.

