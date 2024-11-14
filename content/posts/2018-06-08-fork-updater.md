+++
author = "hannibal"
categories = ["Ruby", "Git"]
date = "2018-06-08T08:01:00+01:00"
title = "Keep your git forks updated all the time"
url = "/2018/06/08/fork-updater"
description = "Keep your git forks updated"
featured = "git_updater.png"
featuredalt = "Octokit"
featuredpath = "date"
linktitle = ""
+++

Hi folks.

Today's is a quick tip for keeping your forks updated.

If you are like me, and have at least a 100 forks in your repository because:
    * You would like to contribute at some point
    * Save it for yourself because you are afraid that it disappears
    * Would like to make modifications for your own benefit
    * Whatever the reason

...then you probably have a lot of trouble keeping them updated and making sure you always see the latest change.

Upstream can change a lot especially if it's a busy repository.

Fret not. Help is here. This little ruby script will solve your troubles:

~~~ruby

#!/usr/bin/env ruby

require 'octokit'
require 'logger'

@logger = Logger.new("output.log")

def update_fork(repo)
  repo_name = repo.name
  # clone the repository -- octokit doesn't provide this feature as it's a github api library
  @logger.info("cloning into #{repo.ssh_url}")
  system("git clone #{repo.ssh_url} #{repo_name}")
  # setup upstream for updating
  @logger.info("setup upstream to #{repo.parent.ssh_url}")
  system("cd #{repo_name} && git remote add upstream #{repo.parent.ssh_url}")
  # do the update
  @logger.info("doing the update with push")
  system("cd #{repo_name} && git fetch upstream && git rebase upstream/master && git push origin")
ensure
  # ensure that the folder is cleaned up
  @logger.info("cleanup: removing the repository folder")
  system("rm -fr #{repo_name}")
end

client = Octokit::Client.new(:access_token => ENV['GIT_TOKEN'], per_page: 100)
repos = client.repos({}, query: {type: 'owner', sort: 'asc'})

# Go through all the pages and add them to the list of repositories.
repos.concat(client.last_response.rels[:next].get.data)

repos = repos.select{ |r| r.fork }

@logger.info("going to update '#{repos.length}' repositories")

repos.each do |repo|
  # get the repositories information
  @logger.info("updating #{repo.name}")
  r = client.repository(repo.id)
  update_fork(r)
end
~~~

This script is also available as a Gist located [here](https://gist.github.com/Skarlso/fd5bd5971a78a5fa9760b31683de690e).

Put this into a cron job, or a Jenkins job on a schedule and you should be good to go.

Note two things:
First: `ENV['GIT_TOKEN']` this should be set to a token which you can acquire by navigating to
[tokens](https://github.com/settings/tokens). Add a token which has `repo` access.

Second: Obviously this script will push to your local repository. So wherever you run this, make sure git is set-up and can push
to your repository via SSH. This script is using `ssh_url` for the repositories. It won't ask for a username or a password.

That's it. Enjoy and keep updating.

Thanks for reading

Gergely.
