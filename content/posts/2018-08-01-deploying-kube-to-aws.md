+++
author = "hannibal"
categories = ["Go", "Kubernetes", "Distributed"]
date = "2018-07-04T08:01:00+01:00"
title = "Deploying the Distributed Face-Recognition Kubernetes App to AWS"
url = "/2018/07/04/learning-rust-with-totp-cli"
description = "Distributed AWS"
featured = "kube-aws.jpg"
featuredalt = "Kubernetes"
featuredpath = "date"
linktitle = ""
draft = true
+++

# Intro

Today, we are going to continue with the Distributed Face-recognition application that we talked about in [this]() previous post.

# Steps

[kops](https://kubernetes.io/docs/setup/custom-cloud/kops/) -- setting up kops

- Create domain under route53.
- Create a hosted zone. // look into private hosted zones. Do public hosted zones cost money?
- dev.skarlso-kube.com. / $0.50 / 25 domains / month.
- skarlso-kube-bucket
- export KOPS_STATE_STORE=s3://dev.skarlso-kube-bucket.com
- Build cluster config: `kops create cluster --zones=us-east-1c useast1.dev.skarlso-kube.com.`
> Note the `.` at the end.

```bash
kops create cluster ${NAME} \
 --dns private  \
 --topology private \
 --api-loadbalancer-type internal \
 --zones ${AZs} \
 --cloud aws \
 --vpc ${VPC} \
 --network-cidr ${NETWORK_CIDR} \
 --networking calico \
 --state ${KOPS_STATE_STORE} \
 --ssh-public-key ./kops-key.pub
 --bastion
```


I'm not going to setup a DNS for this, I'm going to use the `gossip` type of discovery as it's way easier to play around with.

>Note: If you are using Kops 1.6.2 or later, then DNS configuration is optional. Instead, a gossip-based cluster can be easily created. The only requirement to trigger this is to have the cluster name end with .k8s.local. If a gossip-based cluster is created then you can skip this section.

https://github.com/kubernetes/kops/blob/master/docs/aws.md

<!-- Using a private DNS:

skarlso-kube.com vpc-d032d4b8 | eu-central-1 -->

For DNS based approach I would have to create a DNS and create a hosted zone for it and change the name server to the.


But I'm going to use the new `gossip` approach because that is more open to play around with and costs no money at all. And this is a tutorial. Most of the people in tech industry probably own a domain or two, but I'm trying to reach as many people as possible here.

Also, the DNS way is already very well walked where gossip is experimental.

And writing yet another tutorial on DNS is boring.

`aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops`

# Gossip

`.k8s.local` => `dev.skarlso.k8s.local`

`kops create cluster --zones=eu-central-1a --name=dev.skarlso.k8s.local`

**NOTE**: This does not create AWS resources yet. It just creates the configuration for a cluster.

In order to not repeat ourselfes let's put this name into a variable $NAME.

export EDITOR=vim

`kops edit cluster $NAME`.

Actually create the cluster:

```bash
kops update cluster ${NAME} --yes
```

We should see something like:

```bash
kops has set your kubectl context to dev.skarlso.k8s.local
```

And then we can run:

```bash
kops validate cluster # to validate the cluster
```

To check out the nodes:

```bash
kubectl get nodes --show-labels
```

Kops launches a couple of things, which might cost money though. Not much if you want to play around but it will launch 3 ec2 instaces and a load balancer.

After a while, we should see something like this:

```bash
❯ kops validate cluster
Using cluster from kubectl context: dev.skarlso.k8s.local

Validating cluster dev.skarlso.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-eu-central-1a	Master	m3.medium	1	1	eu-central-1a
nodes			Node	t2.medium	2	2	eu-central-1a

NODE STATUS
NAME						ROLE	READY
ip-172-20-32-222.eu-central-1.compute.internal	node	True
ip-172-20-41-18.eu-central-1.compute.internal	node	True
ip-172-20-52-237.eu-central-1.compute.internal	master	True

Your cluster dev.skarlso.k8s.local is ready
```

Now, for a test ride, lets try to deploy one of the services on this cluster which is public facing.

For example the receiver.

Note that for now the receiver is a NodePort. In a critical application that probabaly would be `LoadBalancer`. In fact we'll try that shortly.

For now, we will run:

```bash
kubectl apply -f receiver.yaml
```

I've got a CreateContainerConfigError on the receiver, but the NSQ works fine. Let's see what the problem is.

We can troubleshoot this error by running the following commands:

```bash
❯ kubectl get pods
NAME                                  READY     STATUS                       RESTARTS   AGE
nsqd-584954c44c-kwqdn                 1/1       Running                      0          1m
receiver-deployment-5bd6ff874-vdlhl   0/1       CreateContainerConfigError   0          1m
```

Then:

```bash
kubectl describe pods receiver-deployment-5bd6ff874-vdlhl
```

Which told me amongst other things, this:

```bash
...
  Warning  Failed                 4m (x10 over 6m)  kubelet, ip-172-20-32-222.eu-central-1.compute.internal  Error: secrets "kube-face-secret" not found
...
```

Ah, yeah. I didn't create my secret first...

Let's do that now:

```bash
kubectl create -f secret.yaml
```

I included it into te repo. The db password is password123. :P Okay, now that the secret is created, let's try launching receiver again.

```bash
kubectl apply -f receiver.yaml
```

Let's look at the progress with:

```bash
❯ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
nsqd-584954c44c-dhbxx                 1/1       Running   0          6s
receiver-deployment-5bd6ff874-58nvj   1/1       Running   0          6s
```

Alright. The receiver is running! Let's checkout how we can reach it. Note this bit:

> By default Kops does not configure the EC2 instances to allows NodePort traffic from outside.

Thus we have to go in and manually open the port to check if our service is running correctly.

After editing the security group manually we need to find out on what node this service is running on currently.

To do that we do:

```bash
kubectl describe pods receiver-deployment-5bd6ff874-58nvj
```

It's running on `Node:           ip-172-20-32-222.eu-central-1.compute.internal/172.20.32.222`.

After doing a describe nods for that node, we get the external ip: `ExternalIP:   18.185.6.137`. Thus:

```bash
❯ curl -d '{"path":"unknown4456.jpg"}' http://18.185.6.137:8000/images/post
got paths: {Paths:[]}
```

This works! Now. We need to do some adjustments here in order for this to work properly and to really scale up the parts of the application which we think might be under heavy load. Let's get to it then!

First off let's change receiver to `LoadBalancer` and see if we can reach it through the balancer's IP. The balancer that kops made is for the master and not the nodes. So we will need to create another balancer for our receiver service. If we want two pods of the service to run and be balanced, we'll have to create one more.

If I would have a proper dns behind it I could access it.
