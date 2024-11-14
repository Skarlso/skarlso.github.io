+++
author = "hannibal"
categories = ["Go", "Programming", "DevOps"]
date = "2015-12-08"
title = "Go Development Environment"
url = "/2015/12/08/go-development-environment/"

+++

Hello folks.

Here is a little something I've put together, since I'm doing it a lot.

[Go Development Environment](https://github.com/Skarlso/godevelopment)

If I have a project I'd like to contribute, like [GoHugo](https://gohugo.io), I have to setup a development environment, because most of the times, I'm on a Mac. And on OSX things work differently. I like to work in a Linux environment since that's what most of the projects are built on.

So here you go. Just download the files, and say **vagrant up** which will do the magic.

This sets up [vim-go](https://github.com/fatih/vim-go) with code completion given by YouCompleteMe and some go features like, fmt on save and build error highlighting.

Also sets up ctags which will give you tags and the ability to do GoTo Declaration.

Installs a bunch of utilities, and configures Go. There is an option to install docker as well. But it's ignored at the moment.

Just uncomment this line:

~~~ruby
#config.vm.provision "shell", path: "install_docker.sh"
~~~

Any questions or request, feel free to submit an Issue!

Thanks for reading!
Gergely.
