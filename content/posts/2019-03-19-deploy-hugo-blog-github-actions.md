+++
author = "hannibal"
categories = ["hugo", "github", "actions"]
date = "2019-03-19T22:01:00+01:00"
title = "Deploy a Hugo Blog Github Actions"
url = "/2019/03/19/deploy-hugo-blog-github-actions"

+++

# Intro

Hi folks.

Today I thought I show you how you can use [Github Actions](https://github.com/features/actions) to deploy a hugo based blog like this one.

Let's dive in.

# Actions

What are actions? If you read the above linked document they are basically steps performed in containers based on some events that happened with your repository. Events can be such as pushing, creating a PR or creating/closing an issue etc.

We need an even on a push.

Actions is in beta right now so much of the documentation has some gaps but they are fairly okay. I recommend reading through this one carefully: [Developer Guide](https://developer.github.com/actions/). This describes for example accessing the environment. That is important because we will need to access the generated content from one action in the next action.

# Dockerfile

Each action requires a Dockerfile which will be used to create a container to run this particular action in. The Dockerfile uses LABELS to mark a container. It is recommended to create an ENTRYPOINT in the Dockerfile that can work with CMDs passed in from the action.

For example my pusher container has the ability to push into any repository thanks to using arguments for the entrypoint.sh script.

We'll see that later on.

# Blog actions

Let's look at the two actions in detail which we'll be using.

## Builder

First, we need to build the blog. This is accomplished pretty much the same as I wrote earlier in the travis blog part but with a little extra information.

This is the Dockerfile:

~~~Dockerfile
FROM golang:latest

LABEL "name"="Hugo Builder"
LABEL "maintainer"="Gergely Brautigam <gergely@gergelybrautigam.com>"
LABEL "version"="0.1.0"

LABEL "com.github.actions.name"="Go Builder"
LABEL "com.github.actions.description"="Build a hugo blog"
LABEL "com.github.actions.icon"="package"
LABEL "com.github.actions.color"="#E0EBF5"

RUN \
  apt-get update && \
  apt-get install -y ca-certificates openssl git && \
  update-ca-certificates && \
  rm -rf /var/lib/apt

RUN go get github.com/gohugoio/hugo

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
~~~

Pretty simple. The entrypoint script looks like this:

~~~bash
#!/bin/bash

set -e
set -x
set -o pipefail

if [[ -z "$GITHUB_WORKSPACE" ]]; then
  echo "Set the GITHUB_WORKSPACE env variable."
  exit 1
fi

if [[ -z "$GITHUB_REPOSITORY" ]]; then
  echo "Set the GITHUB_REPOSITORY env variable."
  exit 1
fi

root_path="$GITHUB_WORKSPACE"
echo "Root path is: ${root_path}"
blog_path="$GITHUB_WORKSPACE/.blog"
echo "Blog path is: ${blog_path}"
mkdir -p "$blog_path"
mkdir -p "$root_path"
cd "$root_path"
echo "Preparing to build blog"
hugo --theme hermit
echo "Building is done. Copying over generated files"
cp -R public/* "$blog_path"/
echo "Copy is done."

exit 0
~~~

The interesting parts here are `GITHUB_WORKSPACE` and `GITHUB_REPOSITORY`. The workspace is where the repository is located at.

This is the place where we will copy our built blog files. Since this is a mount basically on the local build machine the next action which comes along will see the folder `.blog`. This is how we pass artifacts between actions.

This action can be found here: [Hugo Blog Builder Action](https://github.com/Skarlso/blog-builder).

## Publisher

Once the building finishes successfully we can push it to the new location.

Dockerfile is similar to the one above in every regard. Except for the name and that it doesn't need Hugo and the command...

~~~Dockerfile
FROM golang:latest

LABEL "name"="Hugo Pusher"
LABEL "maintainer"="Gergely Brautigam <gergely@gergelybrautigam.com>"
LABEL "version"="0.1.0"

LABEL "com.github.actions.name"="Go Pusher"
LABEL "com.github.actions.description"="Push a hugo blog"
LABEL "com.github.actions.icon"="package"
LABEL "com.github.actions.color"="#E0EBF5"

RUN \
  apt-get update && \
  apt-get install -y ca-certificates openssl git && \
  update-ca-certificates && \
  rm -rf /var/lib/apt

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["Skarlso/skarlso.github.io.git"]
~~~

Why do we require the CMD? Let's take a look at the script.

~~~bash
#!/bin/bash

set -e
set -x

if [[ -z "$GITHUB_WORKSPACE" ]]; then
  echo "Set the GITHUB_WORKSPACE env variable."
  exit 1
fi

if [[ -z "$GITHUB_REPOSITORY" ]]; then
  echo "Set the GITHUB_REPOSITORY env variable."
  exit 1
fi

setup_git() {
  repo=$1
  git config --global user.email "bot@github.com"
  git config --global user.name "Github Actions"
  git init
  echo "Starting to clone blog repository"
  git remote add origin https://"${PUSH_TOKEN}"@github.com/"${repo}" > /dev/null 2>&1
  git pull origin master
  echo "Cloning is done"
  ls -l
}

commit_website_files() {
  git add .
  git commit -am "Github Action Build ${GITHUB_SHA}"
}

upload_files() {
  git push --quiet --set-upstream origin master
}

echo "Beginning publishing workflow"
repo=$1
if [ -z "${repo}" ]; then
    echo "Repo must be defined."
    exit 1
fi
echo "Using repository ${repo} to push to"
mkdir /opt/publish && cd /opt/publish
blog_path="$GITHUB_WORKSPACE/.blog"
echo "Blog is located at: ${blog_path}"
ls -l "${blog_path}"
echo "Setting up git"
setup_git "${repo}"
cp -R "${blog_path}"/* .
echo "Copied over generated content from blog path. Committing."
commit_website_files
echo "Committed. Pushing."
upload_files
echo "All done."
exit 0
~~~

Now this is a lot more involved. I'm leaving as many echos in here as possible for esae of debugging.

The interesting part in here is the `repo=$1`. This is why we need CMD specified. But this is what makes this Action a bit more flexible too. It can push anywhere it has access to.

This action can be found here: [Hugo Blog Publisher Action](https://github.com/Skarlso/blog-publisher).

## The Workflow file

How does this all fit together? You have to create a workflow file which looks something like this:

~~~
workflow "Publish Blog" {
  on = "push"
  resolves = ["blog-publisher"]
}

action "blog-builder" {
  uses = "skarlso/blog-builder@master"
  secrets = ["GITHUB_TOKEN"]
}

action "blog-publisher" {
  uses = "skarlso/blog-publisher@master"
  needs = ["blog-builder"]
  secrets = ["GITHUB_TOKEN", "PUSH_TOKEN"]
}
~~~

This is located in your repositroy under `.github/main.workdflow`. Notice the secrets. GITHUB_TOKEN is created for you by Github. This is a basic token which lets you access the github API. But it can't be used for pushing code. Thus, we need another token. This can be defined under your repository / settings / secrets. Once you have a token, add a new secret called PUSH_TOKEN and... done.

Everything should be ready to go.

## Location of the actions

Now, I read the doc and should have been possible to have these actions in the repositroy itself. However, I faced some problems with that setup so I ended up having actions in their respectice repository. That's why `uses` is set up to be `skarlso/<action-name>@branch`.

# Conclusion

On a push now the blog is built and published. If a step fails it won't be published. It's actually a lot faster than my travis build was.

Thank you for reading,

Gergely.
