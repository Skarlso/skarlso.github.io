+++
author = "hannibal"
categories = ["kubernetes"]
date = "2019-10-01T21:01:00+01:00"
title = "How I killed my entire Kubernetes cluster"
url = "/2019/10/01/killing-kubernetes-cluster"
comments = true
+++

# Intro

One morning I woke up and tried to access my gitea just to find that it wasn't running.

![dead kube](/img/kube_dead.png)

I checked my cluster and found that the whole thing was dead as meat. I quickly jumped in and ran `k get pods -A` to see what's
going on. None of my services worked.

What immediately struck my eye was a 100+ pods of my fork_updater cronjob. The fork_updater cronjob which runs once a month, looks
like this:

~~~yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: fork-updater
  namespace: fork-updater
spec:
  schedule: "* * 1 * *"
  jobTemplate:
    spec:
      template:
        spec:
          volumes:
          - name: fork-updater-ssh-key
            secret:
              secretName: fork-updater-ssh-key
              defaultMode: 256 # yaml spec does not support octal mode
          containers:
          - name: fork-updater
            imagePullPolicy: IfNotPresent
            image: skarlso/repo-updater:1.0.4
            env:
              - name:  GIT_TOKEN
                valueFrom:
                  secretKeyRef:
                    name:  fork-updater-secret
                    key:  GIT_TOKEN
            volumeMounts:
            - name: fork-updater-ssh-key
              mountPath: "/etc/secret"
              readOnly: true
          restartPolicy: OnFailure
~~~

Inherently there is nothing wrong with this at first glance. But on a second glance, the problem is `restartPolicy: Always`.
For whatever the reason, the cronjob died when it started up. The restart policy then... restarted the cronjob, which failed again
really fast. Then it scheduled a new one and a new one and a new one... and I had 100+ containers pending and running and
creating.

At that point the cluster was basically DDOSd into oblivion. Once the other resources started to die ( since this was a private
cluster and I didn't bother to set up restrictions on resources ) the cronjob hogged even more and it basically blocked everything
else from being able to run. It overwhelmed the scheduler.

Lovevly that.

This is how you could potentionally kill a cluster which doesn't have any resource limits and restrictions set up.

Gergely.
