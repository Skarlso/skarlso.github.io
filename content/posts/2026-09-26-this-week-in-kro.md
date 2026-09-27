+++
author = "hannibal"
categories = ["kro", "golang", "open-source"]
date = "2026-09-26T01:01:00+01:00"
title = "This week in Kro"
url = "/2026/09/26/this-week-in-kro"
comments = true
+++

I'm going to start to create a hand-curated list of interesting things that has been and
will be happening with [Kro](https://github.com/kubernetes-sigs/kro).

This will be notable pull requests that have been merged. Interesting blogs that might be of relevance.
Information on what we are planning to do next and all kinds of news related to Kro.

Let's dive in to the first episode of... This week in Kro!

_Read this in old-timey anchor news voice._

Today, I'm happy to announce to you that [v0.10.0.rc0](https://github.com/kubernetes-sigs/kro/releases/tag/v0.10.0-rc.0) has been cut!
Yes! You heard right! Get it while it's hot! It contains a lot of goodies so hang on to your butts while we go through the most interesting ones.

This release contains the most massive update the RGD engine, and it contains several fixes that have been long plaguing the community.
Fixes such as, invalid long `node-id`s and `instance-name` labels. You can use as long as a node-id that you want without having to
worry about the maximum 63 character label length. Further, RGD schema changes have gotten a beefing up with more validation rules. This
one might catch you if you don't pay attention! But most important of all... Graph CRD has landed!

_[KREP-024](https://github.com/kubernetes-sigs/kro/pull/1302) has been implemented_! It was a herculean effort to get it in over the course of
several weeks of test runs, reviews and fixes. It is one of the most tested paths in the entire code base, and yet! There
might be still dragons lurking in it (as the number of "fix" PRs show), so proceed with caution. Today, you need a flag to enable it. Please read the [Documentation](https://kro.run/next/docs/concepts/graph/overview) carefully (It is available by selecting `main` version on the Kro website).

Please check out the Release page for more concrete changes linked up above.

**Some up-coming interesting additions that people are working on are in no particular order:**

- [KREP-025 CEL Time library](https://github.com/kubernetes-sigs/kro/pull/1376) is getting lots of thoughts accumulated and is bound to be an interesting addition for
  people who would _hate_ to be woken up on a sunday morning by a failed certificate rotation attempt.
- [KREP-021 Kro CLI](https://github.com/kubernetes-sigs/kro/pull/1421) has been revived and is being pushed forward.
- [Deletion Policy for individual resources](https://github.com/kubernetes-sigs/kro/pull/1445) has been created and is under review. The issues has been talked about
  and tentatively accepted during a community call, but, naming being one of the hardest things in software, how we call the policy is up for debate.
- [Cycle detection optimization](https://github.com/kubernetes-sigs/kro/pull/1449) this one _might_ look tiny, but it actually increases the performance of cycle detection
  by a _LOT_.
- [CEL Anywhere](https://github.com/kubernetes-sigs/kro/pull/1403) I'm sure everyone thought at some point "Hey, why can't I use a CEL function there?" right? Well, now you **can**.
- [Allow referring to CRDs that do not exist yet](https://github.com/kubernetes-sigs/kro/issues/1243) is an issue that pops up in various forms like `defer` for schemas or a more ergonomic way
  of referencing a CRD. This has been solved by `Graphs`! Yet another reason why you should check them out.

---

**In other news:**

[Jesse Butler](https://github.com/jlbutler) mentioned in last community meeting that he would like to re-visit the usage of KREPs in the Kro ecosystem. This came from the problem that he believe KREPs
got a bit out of hand and people use them for bike-shedding ideas around a concept and thus the pull request and the discussions explode quickly. They, however, seem
to be no longer so useful after a while and the idea gets abandoned because it gets nowhere and the person who opened the KREP gets discouraged from engaging further.

This seems like a good time to review this idea and figure out how to better handle architectural decisions and ideas.

---

Eddie Moya, keeps updating his amazing set of complex RDG examples to provide the community with a nice set of not-so-basic-usage scenarios over at his [Codeberg Repository for Gitea RGD](https://codeberg.org/eddiemoya/gitea-rgd).

---

And this is all for this week folks! Hope to see you again next time.

Have a lovely rest of the Sunday.

Gergely.
