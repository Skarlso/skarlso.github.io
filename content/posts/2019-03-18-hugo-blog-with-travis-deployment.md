+++
author = "hannibal"
categories = ["hugo", "travis"]
date = "2019-03-18T21:01:00+01:00"
title = "Deploy a Hugo Blog with Travis on Git Push"
url = "/2019/03/18/hugo-blog-with-travis-deployment"

+++

# Intro

Hi folks.

I've been using the Hugo build for wercker for a long time now. Recent problems occurred though where I did not understand at first
what the problem was. It was quite difficult to debug since I did not have too much insight on the wercker build itself. Turned
out that I deleted the GITHUB token that the process was using. However, the error message was telling me that a function failed
to load some other function. Which was totally unrelated.

Thus, I thought that I'm going to shift away from this outside medium to a different one that I'm already familiar with and have
greater control over.

Hence, Travis. Incidentally, since I will no longer be dependend on a third party component (which was the image wercker was
using), I'll be able to switch away from this build platform easily. For example, to CircleCI.

I'm using github pages, but without the whole convoluted submodule init, different branch stuff. I find that that simply adds unnecessary complexity to the whole thing. I'm keeping the source and the website in a different repository.

The steps are simple:

* Get the source
* Generate the content locally using `hugo`
* Setup Git
* Get the source for the generated web site
* Copy in the newly generated code
* Push the code up to git

Sounds simple... In fact it's so simple, it's three files.

### Travis

The travis modification is such:

~~~yaml
language: go

install:
    - go get github.com/gohugoio/hugo
    - sudo apt-get install -y git

script:
    - .travis/build.sh

after_success:
    - cd ${TRAVIS_BUILD_DIR} && .travis/push.sh
~~~

Easy, it's a standard Go based travis file. There are two things here which stand out. The `scripts` section and the
`after_success` section. Why `after_success`? Because if we made a mistake, we don't want to destroy the website. Thus we only
push in case build.sh was successful.

### Building

In this light, building the blog is simple. In fact the whole script is such:

~~~bash
#!/bin/sh

set -e
set -x

mkdir /opt/blog
git clone --recurse-submodules https://github.com/Skarlso/blogsource.git /opt/app
echo Build started on `date`
cd /opt/app
hugo --theme hermit
cp -R public/* /opt/blog
~~~

For clone, `--recurse-submodules` is required because the theme is a submodule. Once this script runs successfull,
we can push the new version of the site.

### Pushing

Pushing is a bit more involved. There are four steps involved in this process.

#### Setup git

First, we set up git to use some specific name so we know where the push came from.

~~~bash
setup_git() {
  git config --global user.email "travis@travis-ci.org"
  git config --global user.name "Travis CI"
  git init
  git remote add origin https://${GITHUB_TOKEN}@github.com/Skarlso/skarlso.github.io.git > /dev/null 2>&1
  git pull origin master
}
~~~

Github token is a secret environment property. We also pull the blog source in this step.

#### Copy

Then copy everything from the built site's public folder (which was we already copied to /opt/blog) to this folder.

~~~bash
cp -R /opt/blog/* .
~~~

#### Commit the changes

~~~bash
commit_website_files() {
  git add .
  git commit -am "Travis build: $TRAVIS_BUILD_NUMBER"
}
~~~

This is extracted for clarity.

#### Pushing the changes

~~~bash
upload_files() {
  git push --quiet --set-upstream origin master
}
~~~

The script in it's entirety here: [push.sh](#push-sh)

And that's it. The site is changed and updated. This can be executed in any environment and the only requirement is hugo and git being present. If you still prefer the branch method of Github pages, this is easily altered to checkout the right branch and push the changes from there.

No dependency on anything. Just how I like my build processes.

Thanks for reading!

Gergely.

# Appendix

## push.sh

~~~bash
#!/bin/sh

set -e
set -x

setup_git() {
  git config --global user.email "travis@travis-ci.org"
  git config --global user.name "Travis CI"
  git init
  git remote add origin https://${GITHUB_TOKEN}@github.com/Skarlso/skarlso.github.io.git > /dev/null 2>&1
  git pull origin master
}

commit_website_files() {
  git add .
  git commit -am "Travis build: $TRAVIS_BUILD_NUMBER"
}

upload_files() {
  git push --quiet --set-upstream origin master
}

mkdir /opt/publish && cd /opt/publish
setup_git
cp -R /opt/blog/* .
commit_website_files
upload_files
~~~
