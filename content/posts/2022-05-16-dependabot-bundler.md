+++
author = "hannibal"
categories = ["go"]
date = "2022-05-16T01:01:00+01:00"
title = "Dependabot Bundler"
url = "/2022/05/16/dependabot-bundler"
comments = true
+++

Hello Dear readers.

Today, I would like to write about a small pet project of mine called [Dependabot Bundler](https://github.com/Skarlso/dependabot-bundler).

We've all been here:

![prs](/img/2022/05/16/multiple_prs.png)

You would like to review these individually, of course, however, having to individually merge them is quite the pain.

Isn't it a lot better to do this instead?

![prs](/img/2022/05/16/single_pr.png)

A lot nicer and more comfortable, isn't it?

This is done by running dependabot-bundler with the following command:

```
./bundler --token ${GITHUB_TOKEN} --repo eksctl --owner weaveworks --author-name Skarlso --author-email gergely@weave.works --labels skip-release-notes
```

The command has to be executed from the root folder of the project.

That's all for today.

Thank you for reading!
Gergely.
