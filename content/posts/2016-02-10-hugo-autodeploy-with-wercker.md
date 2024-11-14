+++
author = "hannibal"
categories = ["Hugo", "Wercker"]
date = "2016-02-10"
title = "Hugo Autodeploy with Wercker and Github - Pages"
url = "/2016/02/10/hugo-autodeploy-with-wercker"

+++

There already is a nice tutorial on how to create github-pages with Hugo [Here](https://gohugo.io/tutorials/github-pages-blog/) if you prefer deplying your pages to a different branch on the same repo. There is also a post about Wercker and Hugo [Here](https://gohugo.io/tutorials/automated-deployments/) deploying pages to said separate branch.

However, I took an easier approach on the matter with a completely separate branch for my blog source and my compiled github pages.

This blog sits here: https://github.com/Skarlso/skarlso.github.io. In order to deploy to it, I just have to commit a new blog post to this repository: [Blog Source](https://github.com/Skarlso/blogsource). After that, [Wercker](http://wercker.com/) takes care of the rest. It builds my blog, and pushes the generated pages to my blog's repository to the master branch without creating the gh-pages branch.

The Wercker yml for that looks like this:

~~~yaml
box: debian
build:
    steps:
        - arjen/hugo-build:
            theme: redlounge
deploy:
    steps:
        - install-packages:
            packages: git ssh-client
        - leipert/git-push:
            gh_oauth: $GIT_TOKEN
            repo: skarlso/skarlso.github.io
            branch: master
            basedir: public
~~~

Pretty easy. The $GIT_TOKEN is a variable set-up on Wercker containing a restricted token which is only good for pushing. And note that you have to use an explicit package name with git-push or else Wercker will not find that step. Hugo-build will build my blog with a simple command using redlounge theme.

And that's it. No other setup is necessary and no new branch will be made. Any questions, please feel free to leave a comment.

Thanks for reading!

Gergely.
