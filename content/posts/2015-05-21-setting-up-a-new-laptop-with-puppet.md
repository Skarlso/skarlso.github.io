+++
author = "hannibal"
categories = ["devops"]
date = "2015-05-21"
tags = ["puppet"]
title = "Setting up a new Laptop with Puppet"
url = "/2015/05/21/setting-up-a-new-laptop-with-puppet/"

+++

Hello folks.

So, some of you know <a href="https://puppetlabs.com/" target="_blank">puppet</a>, some of you don't. Puppet is a configuration management system. It's quite awesome. I like working with it. One of the benefits of puppet is, that I never, ever, EVER have to setup a new laptop from scratch, EVER again.

I'm writing a puppet manifest file which sets up my new laptop to my liking. I will improve it as I go along. Here is version 1.0.

~~~ruby

# include apt

class base::basics {
        $packages = ['git', 'subversion', 'mc', 'vim', 'maven', 'gradle']

        exec { "update":
                command => "/usr/bin/apt-get update",
        }

        package { $packages:
                ensure => installed,
                require => Exec["update"],
        }

}

class base::skype {
        exec { "add-arc":
                command => "/usr/bin/dpkg --add-architecture i386",
        }

        exec { "add-repo-skype":
                command => "/usr/bin/add-apt-repository \"deb http://archive.canonical.com/ \$(lsb_release -sc) partner\"",
                require => Exec['add-arc'],
        }

        exec { "update-and-install":
                command => "/usr/bin/apt-get update && /usr/bin/apt-get install skype",
                require => Exec['add-repo-skype'],
        }
}

class base::java8 {
        # Automatically does an update afterwards
        # apt::ppa { 'ppa:webupd8team/java': }
        exec { "add-repo-java":
                command => "/usr/bin/add-apt-repository -y \"ppa:webupd8team/java\" && /usr/bin/apt-get update"
        }

        exec { "set-accept":
                command => "/bin/echo /usr/bin/debconf shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections && /bin/echo /usr/bin/debconf shared/accepted-oracle-license-v1-1 seen true | sudo /usr/bin/debconf-set-selections",
                require => Exec['add-repo-java'],
        }

        exec { "install":
                command => "/usr/bin/apt-get install -y oracle-java8-installer",
                require => Exec['set-accept'],
        }

        exec { "setup_home":
                command => "/bin/echo \"export JDK18_HOME=/usr/lib/jvm/java-8-oracle/\" >> /etc/environment",
                require => Exec['install'],
        }
}

include base::basics
include base::skype
include base::java8
~~~

I'll improve upon it as I go, and you can check it out later from my git repo. I removed the parts which required extra libraries for now, as I want it to run without the need of getting extra stuff installed. I might automate that part as well later on.

EDIT: <a href="https://github.com/Skarlso/puppet/blob/master/manifests/base_setup.pp" target="_blank">https://github.com/Skarlso/puppet/blob/master/manifests/base_setup.pp</a>

Have fun.

Gergely.