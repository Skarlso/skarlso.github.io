+++
author = "hannibal"
categories = ["Go", "Wercker"]
date = "2016-03-09"
title = "Wercker Fixed"
url = "/2016/03/09/wercker-fixed"

+++

Hi Folks.

So Wercker was not working. After a minor modification it seems to be okay now. The config file needed for it to work looks like this:

~~~bash
box: golang
build:
    steps:
        - arjen/hugo-build:
            theme: redlounge
deploy:
    steps:
        - install-packages:
            packages: git
        - leipert/git-push:
            gh_oauth: $GIT_TOKEN
            repo: skarlso/skarlso.github.io
            branch: master
            basedir: public
~~~

The modification is the box type to *golang* and removed *ssh-client* from *packages*.

Thanks,
Gergely.
