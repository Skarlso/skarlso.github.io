+++
author = "hannibal"
categories = ["kubernetes"]
date = "2019-09-21T21:01:00+01:00"
title = "Using a Kubernetes based Cluster for Various Services with auto HTTPS"
url = "/2019/09/21/kubernetes-cluster"
comments = true
+++

# Intro

Hi folks.

Today, I would like to show you how my infrastructure is deployed and managed. Spoiler alert, I'm using Kubernetes to do that.

I know... What a twist!

Let's get to it.

# What

![kube-architecture](/img/hosting/kube-architecture.png)

What services am I running exactly? Here is a list I'm running at the time of this writing:

- Athens Go Proxy
- Gitea
- The Lounge (IRC bouncer)
- Two CronJobs
    - Fork Updater
    - IDLE RPG online checker
- My WebSite (gergelybrautigam.com)
- Monitoring

And it's really simple to add more.

# Where

My cluster is deployed at DigitalOcean using two droplets each 1vCPU and 2GB RAM.

![kube-on-digitalocean](/img/hosting/kube-on-digitalocean.png)

# What Not

This isn't going to be a production grade cluster. What I don't include in here:

## RBAC for various services and users

Since I'm the only user of my cluster I didn't create any kind of access limits / users or such. You are free to create them though. The only role based auth that's going on is for Prometheus.

I'm not using any third party things which require access to the API.

## Resource limitation

I'm the sole user of my things. I'm not really scaling my gitea up or down based on usage and as such, I did not define things like:

- Resource limits
- Nodes with certain capabilities
- Affinities and Taints -- which means, everything can run anywhere

## Readiness Liveliness

Most of by deploys and services don't have these except for Athens.

# How

Okay, with that out of the way, let's get into the hows of things...

# Beginning

The most important thing that you need to do in order to use Kubernetes is Containerizing all the things.

![containers](/img/hosting/containers.png)

Since Kubernetes is a container orchestration tool, without containers it's pretty useless.

As a driver, I'm going to use Docker. Kubernetes can use anything that's OCI compatible, which means if you would like to use runc as a container engine, you can do that. I'd like to keep my sanity though.

## Example

![fork-updater](/img/hosting/fork-updater.png)

To show you what I mean... I have a cronjob which is running every month. It gathers all my forks on github and updates them with the latest from their parents. This a small ruby script located here: [Fork Updater](https://gist.github.com/Skarlso/fd5bd5971a78a5fa9760b31683de690e). How do we run this? It requires two things. First, a token. We pass that currently as an environment property. It could be in a file in a vault or a secret mounted in as a file it doesn't matter. Currently, it's an environment property. The second thing is more subtle.

I'm pushing the changes back into my remote forks. I'm doing this via SSH. So, we need a key in there too. How we'll get that in there, I'll talk about later when we are talking about how to set this cron job up. For now though, the container needs to look for a key in a specific location because we don't want to over-mount `/root/.ssh/` and we also don't want to use an initContainer to copy over an SSH key (because it's mounted in as a symlink (but that's a different issue all together)). Also, we certianly do NOT want to have a key in the container.

To achieve this, we simply set up a `config` file for SSH like this one:

```
Host github.com
    IdentityFile /etc/secret/id_rsa
```

`/etc/secret` will be the destination of the ssh key we create.

And we also need to have a known_hosts file, otherwise git clone will complain. We also bake this into the container. Why? Why not generate that on the fly? Because I want it to fail in case there is something wrong or there is a MIMA going on etc.

All this translated into a Dockerfile looks like this:

```Dockerfile
# We are using alpine for a minimalistic image
FROM alpine:latest

RUN apk --no-cache add ca-certificates
RUN apk update
# Openssh is needed for the SSH command
RUN apk --no-cache add ruby vim curl git build-base ruby-dev openssh openssh-client

# Setup dependencies for the fork ruby file
RUN gem install octokit logger multi_json json --no-ri --no-rdoc

RUN mkdir /data
WORKDIR /data
# Setup some data about the committer
RUN git config --global user.name "Fork Updater"
RUN git config --global user.email "email@email.com"
RUN mkdir -p /root/.ssh
# Get the host config for github.com
RUN ssh-keyscan github.com >> /root/.ssh/known_hosts

# Setup the SSH config
COPY ./config /root/.ssh
COPY ./fork_updater.rb .
CMD ["ruby", "/data/fork_updater.rb"]
```

That's it. Now our updater is containerized and ready to be deployed as a cronjob on a kube cluster. Oh, and we also need to create the SSH key like this:

```
kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa
```

# Before we Begin

There are two things we will need though to set up for our cluster before we even begin adding the first service. And that's an ingress with a load-balancer and cert-manager.

# Cert-Manager

Now, you have the option of installing cert-manager via helm, or via the provided kube config yaml file. I **STRONGLY** recommend using the config yaml file because the upgrading process with helm is a hell of a lot dirtier / failure prone than simply applying a new yaml file with a different version in it.

Either way, to install cert-manager follow this simple guide: [Cert-manager Install Manual](https://docs.cert-manager.io/en/latest/getting-started/install/kubernetes.html#installing-with-regular-manifests).

# Ingress

An Ingress is a must. This is the component which manages external access to the services which we will define. Like a proxy before your http server. This component will handle the hostname based routing between our services.

I'm using nginx ingress, although there are a couple of implementations out there.

To install nginx ingress, follow their guide here: [Installing Nginx-Ingress](https://kubernetes.github.io/ingress-nginx/deploy/).

# From Easy to Complicated

Alright. Now that we have the prereqs out of the way, let's get our hands dirty. I'll start with the easiest of them all, my web site, and then will progress towards the more complicated ones, like Gitea and Athens, which require a lot more fiddling and have more moving parts.

## My Website

The site, located here: [Gergely's Domain](https://gergelybrautigam.com); is a really simple, static, [Hugo](https://gohugo.io) based website. It contains nothing fancy, no real Javascript magic, has a simple list of things I've done and who I am.

It's powered / served by an nginx instance running on port 9090 define by a very simple Dockerfile:

```Dockerfile
FROM golang:latest as builder
RUN apt-get update && apt install -y git make vim hugo
RUN mkdir -p /opt/website
RUN git clone https://github.com/Skarlso/gergelybrautigam /opt/website
WORKDIR /opt/website
RUN make

FROM nginx:latest
RUN mkdir -p /var/www/html/gergelybrautigam
WORKDIR /var/www/html/gergelybrautigam
COPY --from=builder /opt/website/public .
COPY 01_gergelybrautigam /etc/nginx/sites-available/
RUN mkdir -p /etc/nginx/sites-enabled/
RUN ln -s /etc/nginx/sites-available/01_gergelybrautigam /etc/nginx/sites-enabled/01_gergelybrautigam
```

Easy as goblin pie. Nginx has a command set like this `CMD ["nginx", "-g", "daemon off;"]` and exposes port 80.

### The deployment

In order to deploy this in the cluster, I created a deployment as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gb-deployment
  namespace: gergely-brautigam
  labels:
    app: gergelybrautigam
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gergelybrautigam
  template:
    metadata:
      labels:
        app: gergelybrautigam
      annotations:
          prometheus.io/scrape: 'true'
          prometheus.io/port:   '9090'
    spec:
      containers:
      - name: gergelybrautigam
        image: skarlso/gergelybrautigam:v0.0.26
        ports:
        - containerPort: 9090
```

The metadata section defines information about the deployment. It's name is gb-deployment. The namespace in which this sits is called gergely-brautigam and it has some labels to it so Prometheus monitoring tool can discover the pod.

It's running a single replica, has a bunch of more metadata and template settings, and finally the container spec which defines the image, and the exposed container port on which the application is running.

Now we need a service to expose this deployment.

### The service

The service is also simple. It looks like this:

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: gergely-brautigam
  name: gb-service
spec:
  selector:
    app: gergelybrautigam
  ports:
  - port: 80
    targetPort: 9090
```

Again, nothing fancy here, just a simple service exposing a port to a different port on the front-end side. This service will be picked up by our previously created routing facility.

### Ingress

Now that we have the service we need to expose it to the domain. I have the domain gergelybrautigam.com and I already pointed it at the LoadBalancer's IP which was created by the nginx ingress controller.

We only want one LoadBalancer, but we have multiple hostnames. We can achieve that by creating an Ingress resource in the namespace our service is in like this:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: gergely-brautigam
  name: gergely-brautigam-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/cluster-issuer: "letsencrypt-prod"
    certmanager.k8s.io/acme-challenge-type: http01
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - gergelybrautigam.com
    secretName: gergelybrautigam-tls
  rules:
  - host: gergelybrautigam.com
    http:
      paths:
      - backend:
          serviceName: gb-service
          servicePort: 80
```

Remember, we already have the nginx ingress resource in the default namespace when we installed the controller previously. That is the main entrypoint. We are taking advantage of the rewrite-target annotation. That is our key to success `nginx.ingress.kubernetes.io/rewrite-target: /`. The rest is basic routing. We'll have something like this in the other namespace to.

And with that, our website is done and it should be working under HTTPS. Cert-manager should have picked it up and generated a certificate for it. Let's check.

Running `k get certs -n gergely-brautigam` you should see something like this:

```
 $ k get certs -n gergely-brautigam
NAME                   READY   SECRET                 AGE
gergelybrautigam-tls   True    gergelybrautigam-tls   86d
```

If there is a problem, just describe the cert resource and look for the generated challenge and if it was successful or not. The challenge contains mostly good error messages.

## IRC bouncer

That wasn't too bad, right? Let's do something a bit more complex this time. We are going to deploy [The lounge](https://github.com/thelounge/thelounge) irc bouncer.

It's actually quite easy to do but can be daunting to look at at first.

![easy](/img/hosting/the-climb.png)

### The container

Lucky for us, the bouncer already provides a container located here: [The Lounge Docker Hub](https://hub.docker.com/r/thelounge/thelounge/).

We just need two things. To expose the port 9000 and to give it something called a PersistentVolume. What's a persistent volume? Well, look it up here: [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

TL;DR: We need to preserve data. Containers are ephemeral in nature. Meaning if there is a problem we usually just delete the pod. Which means that all data will be lost. But we need persistence in this case because we'll have user data and user information which we would like to persist between pods. That's what a volume is for.

It will be mounted into the pod so we can point the bouncer to use that location for data management.

### PVC

With that, this is how our PersistentVolumeClaim will look like:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: powerhouse
  name: do-storage-irc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: do-block-storage
```

DigitalOcean provides a block storage implementation for this claim so we use that storage class `do-block-storage`.

### Deployment

With that, this is how the deployment will look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: powerhouse
  name: irc-app
  labels:
    app: irc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: irc
  template:
    metadata:
      labels:
        app: irc
        app.kubernetes.io/name: irc
        app.kubernetes.io/instance: irc
    spec:
      containers:
      - name: irc
        image: thelounge/thelounge:3.1.1
        ports:
        - containerPort: 9000
          name: irc-http
        volumeMounts:
        - mountPath: /var/opt/thelounge
          subPath: thelounge
          name: irc-data
          readOnly: false
      volumes:
        - name: irc-data
          persistentVolumeClaim:
            claimName: do-storage-irc
```

Short and sweet. The important bits are the labels, those are used by cert-manager and ingress to find the right deployment, and the `volumeMounts`. We mount into the /var/opt/thelounge folder because that's the main configuration location. The subPath is important for a correct mounting.

### The service

Alright, with the deployment in place, let's take a look at the service.

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: powerhouse
  name: irc
  labels:
    app: irc
    app.kubernetes.io/name: irc
    app.kubernetes.io/instance: irc
spec:
  selector:
    app: irc
    app.kubernetes.io/name: irc
    app.kubernetes.io/instance: irc
  ports:
  - name: http
    port: 9000
    targetPort: irc-http
```

Again, very boring stuff. Boring is good. Boring is predictable. We expose port 9000 to the named targetPort called irc-http which we defined in the above deployment.

Now, I have a domain in which these things are running, let's call it `powerhouse.com` (because I'm tired of example.com). And I have multiple services in this namespace too, so I'll call the namespace here, powerhouse and put this irc service in there. This also means that the ingress resource for this namespace will contain a couple more routings, because my powerhouse namespace will also contain my gitea and Athens proxy installation.

We can, however, take a peak at the ingress resource here and now... because I hate suspense.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: powerhouse
  name: powerhouse-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/cluster-issuer: "letsencrypt-prod"
    certmanager.k8s.io/acme-challenge-type: http01
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - irc.powerhouse.com
    secretName: irc-powerhouse-tls
  - hosts:
    - gitea.powerhouse.com
    secretName: gitea-powerhouse-tls
  - hosts:
    - athens.powerhouse.com
    secretName: athens-powerhouse-tls
  rules:
  - host: irc.powerhouse.com
    http:
      paths:
      - backend:
          serviceName: irc
          servicePort: 9000
        path: /
  - host: gitea.powerhouse.com
    http:
      paths:
      - backend:
          serviceName: gitea
          servicePort: 3000
        path: /
  - host: athens.powerhouse.com
    http:
      paths:
      - backend:
          serviceName: athens-service
          servicePort: 80
        path: /
```

We can see that I have multiple paths pointing to three different subdomains with different ports. These ports will be routed to by nginx ingress. Meaning you **DO NOT OPEN THESE ON YOUR LOADBALANCER**. These will all be accessible from 443/HTTPS. Expect for gitea's SSH port later on.

With these in place, cert-manager should pick it up and provide a certificate for it.

### Side track -- debugging

If there is a problem and we can't reach TheLounge we need to debug. I use the following tool to access Kubernetes resources: [K9S](https://github.com/derailed/k9s). It's a neat CLI tool to look at kube resources in an interactive way and not having to type in a bunch of commands. Never the less, I will also paste those in here.

To look at the pods that should have been created, type:

```
k get pods -n powerhouse
```

Should see something like this:

```
NAME                          READY   STATUS    RESTARTS   AGE
athens-app-857749c59c-lmjnb   1/1     Running   0          6d3h
gitea-app-6974fb995b-pn2vv    1/1     Running   0          9d
gitea-db-59758fbcd9-4562c     1/1     Running   0          9d
irc-app-5f87688f98-dqsvb      1/1     Running   0          9d
```

You can see that my other services are running fine. And there is IRC as well. Now if there would be any kind of problem we could access the Pods information be describing the pod with:

```
k describe pod/irc-app-5f87688f98-dqsvb -n powerhouse
```

Which will provide a bunch of information about the pod. But the pod could be absolutely fine, yet our service could be down. (We didn't define any liveliness or readiness probs after all).

We can verify that by taking a peak in the container (also, check if our mounting was successful). Since this is just a container, exec works similar to docker exec.

```
 $ k exec -it irc-app-5f87688f98-dqsvb -n powerhouse /bin/bash
root@irc-app-5f87688f98-dqsvb:/#
```

Should give us a prompt. We can now look at logs, check out the configuration folder etc.

In k9s you would simply select the right namespace, select the pod, hit `d` for describe or `s` for shell. Done.

## Gitea

Now, we have IRC running. Let's try deploying [Gitea](https://gitea.io/en-us/). This takes a tiny bit more fiddling though.

### Requirements

Gitea requires the following things to be present:

- The gitea app configuration file (this can be done via environment properties though)
- A DB
- A PersistentVolume
- SSH Port for SSH based git clones instead of simple https

#### DB

We shall begin with the simplest of them, the DB. At this point we could go with the DigitalOcean managed Postgres installation, but I didn't want to put that on the bill as well. So I choose to simply put my DB into a container and deploy it within the cluster.

This is actually quite simple. The DB will be a separate deployment / application in the same namespace as the Gitea app. It will also contain a network policy, since the DB doesn't need internet access and the internet shouldn't be able to access it.

In fact the only thing that should be able to access the DB is the Gitea application itself which we will be able to restrict via the usage of... Labels!

##### Deployment

But first, take a look at the deployment of a Postgres 11 pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: powerhouse
  name: gitea-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea-db
  template:
    metadata:
      name: gitea-db
      labels:
        app: gitea-db
    spec:
      containers:
      - name: postgres
        image: postgres:11
        env:
          - name: POSTGRES_USER
            value: gitea
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: gitea-db-password
                key: password
          - name: POSTGRES_DB
            value: gitea
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          subPath: data # important so it gets mounted properly
          name: git-db-data
      volumes:
        - name: git-db-data
          persistentVolumeClaim:
            claimName: do-storage-gitea-db
```

Okay, there are a lot of things going on here, but the three things we need to note are the following:

```yaml
      labels:
        app: gitea-db
```

Our network policy will look for this label to identify the pods which fell under its rule.

```yaml
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: gitea-db-password
                key: password
```

The database password will come from a secret.

```yaml
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          subPath: data # important so it gets mounted properly
          name: git-db-data
      volumes:
        - name: git-db-data
          persistentVolumeClaim:
            claimName: do-storage-gitea-db
```

We also need a persistent volume otherwise the data will be lost on each pod restart.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: powerhouse
  name: do-storage-gitea-db
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: do-block-storage
```

##### Service

We also need a Service so Gitea will be able to reach it. This isn't public though so a NodePort is enough with no clusterIP.

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: powerhouse
  name: gitea-db-service
spec:
  ports:
  - port: 5432
  selector:
    app: gitea-db
  clusterIP: None
```

In order to reach this DB we can use a URL like this now from the Gitea app: `gitea-db-service.powerhouse.svc.cluster.local:5432`.

##### NetworkPolicy

We want the Gitea app to be able to reach it. Which means in-out to the Gitea app and nothing else.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gitea-db-network-policy
  namespace: powehouse
spec:
  podSelector:
    matchLabels:
      app: gitea-db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: gitea
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: gitea
    ports:
    - protocol: TCP
      port: 5432
```

We can test this now by exec-ing into the Pod of the DB deployment and trying to ping google.com for example. It should be denied. Yet later, when we deploy our Gitea app, that should be able to talk to the DB instance.

##### Secret

Finally, we have a Secret which contains our db password base64 encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: cronohub
  name: gitea-db-password
type: Opaque
data:
  password: Z2l0ZWE=
```

That says password123. To get it, you can run something like `echo -n "password123" | base64`.

#### Gitea App ini

Huh, with that done, we can go on with the application ini file. This can be configured via environment properties but once you get over a dozen configuration entries, it's just easier to use an app.ini. My app ini is large, so I won't post it here. I could mount it in as a file, but that proved to be difficult or not work at all properly because Gitea is running under a different user than root. Also, once the mount happened the fact the gitea was trying to write into it caused problems. Mounting as a different user didn't work out either, so I'm using an [InitContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) to do the job. They are there for that reason. And it was actually a hell of a lot simpler than doing file mounting.

The app.ini is defined as a ConfigMap like this:

```
kubectl create configmap gitea-app-ini --from-file=app.ini --namespace powerhouse
```

This was done from the folder where my app.ini was residing.

#### Deployment

Now comes the big gun. The Gitea deployment file. This is how it looks like in all its glory:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: cronohub
  name: gitea-app
  labels:
    app: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
        app.kubernetes.io/name: gitea
        app.kubernetes.io/instance: gitea
    spec:
      initContainers:
      - name: init-disk
        image: busybox:latest
        command:
        - /bin/chown
        - 1000:1000 # we set the gid and uid of the user for gitea.
        - /data
        volumeMounts:
        - name: git-data
          mountPath: "/data"
          readOnly: false
      - name: init-app-ini
        image: busybox:latest
        command: ['sh', '-c', 'mkdir -p /data/gitea/conf/; cp /data/app.ini /data/gitea/conf']
        volumeMounts:
        - name: git-data
          mountPath: "/data"
          readOnly: false
        - name: gitea-app-ini-conf
          mountPath: /data/app.ini
          subPath: app.ini
          readOnly: false
      containers:
      - name: gitea
        image: gitea/gitea:1.9.2
        env:
          - name: DB_PASSWD
            valueFrom:
              secretKeyRef:
                name: gitea-db-password
                key: password
          - name: DB_TYPE
            valueFrom:
              configMapKeyRef:
                name: gitea-config-map
                key: DB_TYPE
          - name: DB_HOST
            valueFrom:
              configMapKeyRef:
                name: gitea-config-map
                key: DB_HOST
          - name: DB_NAME
            valueFrom:
              configMapKeyRef:
                name: gitea-config-map
                key: DB_NAME
          - name: DB_USER
            valueFrom:
              configMapKeyRef:
                name: gitea-config-map
                key: DB_USER
        ports:
        - containerPort: 3000
          name: gitea-http
        - containerPort: 22
          name: gitea-ssh
        volumeMounts:
        - mountPath: /data
          name: git-data
          readOnly: false
      volumes:
        - name: git-data
          persistentVolumeClaim:
            claimName: do-storage-gitea
        - name: gitea-app-ini-conf
          configMap:
            name: gitea-app-ini

```

The important bit is the initContainer section. What's happening here? We mount the app.ini file to the init container under /data. The awesome part about the initContainer is that the real container will have access to the file system the init container created.

So we take that file, fix the permissions on it and copy it to the right location under `/data/gitea/conf` for the Gitea app to work with.

Done!

And the configMap is simple:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: powerhouse
  name: gitea-config-map
data:
  APP_COLOR: blue
  APP_MOD: prod
  DB_TYPE: postgres
  DB_HOST: "gitea-db-service.cronohub.svc.cluster.local:5432"
  DB_NAME: gitea
  DB_USER: gitea
```

#### SSH

Normally, Ingress only allows HTTP based traffic control. But what would an ingress be without also regular TCP based routing?

But it's not trivial. Nginx Ingress provides a documentation for this here: [Exposing TCP and UDP services](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/). What does that mean in practice?

You see we are also exposing port 22 on the container:

```yaml
        - containerPort: 22
          name: gitea-ssh
```

I choose to differentiate my SSH port for Gitea from port 22 because that's just cumbersome to get done right. Gitea provides an explanation on how to do port 22 forwarding in a docker container with a custom git command which forwards commands to the container itself. This is all just plain too much to worry about.

I have this in the app.ini:

```ini
SSH_PORT         = <port of my choosing>
```

And then this in the Service definition:

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: powerhouse
  name: gitea
  labels:
    app: gitea
    app.kubernetes.io/name: gitea
    app.kubernetes.io/instance: gitea
spec:
  selector:
    app: gitea
    app.kubernetes.io/name: gitea
    app.kubernetes.io/instance: gitea
  ports:
  - name: http
    port: 3000
    targetPort: gitea-http
  - name: ssh
    port: <port of my choosing>
    targetPort: gitea-ssh
    protocol: TCP
```

And then, we edit the nginx-controller deployment like this:

```
kubectl edit deployment.apps/nginx-ingress-controller
```

And add this line `--tcp-services-configmap=cronohub/gitea-ssh-service` to the container's args field:

```yaml
      containers:
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=default/nginx-ingress-default-backend
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=default/nginx-ingress-controller
        - --tcp-services-configmap=powerhouse/gitea-ssh-service
```

One more thing is that we have to open that port on the load balancer as well to get traffic to it. To that end, edit the nginx ingress service as well:

```
kubectl edit services/nginx-ingress-controller
```

And add the exposed port:

```yaml
  - name: ssh
    port: <port of my choosing>
    protocol: TCP
    targetPort: <port of my choosing>
```

There will probably be a nodePort section in there on the other ports. Ignore that for your change.

Also, if you are doing the nginx installation by hand, just add this or save the yaml file from those deployments like this:

```
kubectl get service/nginx-ingress-controller -o yaml > nginx-ingress-controller.yaml
```

So you can deploy / modify it later on.

#### Finished Gitea

And with that, visit `gitea.powerhouse.com` and it should work including HTTPS and SSH!

You can now clone repositories like this: `git clone ssh://git@gitea.powerhouse.com:1234/user/awesome_project.git` after you created your user.

User creation is done by using the gitea admin CLI tool described here: [Gitea Documentation](https://docs.gitea.io/en-us/command-line/).

It is important to note that we don't use `latest` anywhere. It's just not good if you are trying to update a service later on. We could set ImagePolicy to AlwaysPull but that's just not a good thing to do if you have a 2 gig image. Always use version and policy `imagePullPolicy: IfNotPresent` to save yourself some bandwidth.

## Idle Checker

![idle-checker](/img/hosting/idle-checker.png)

Let's create a last resource, then we'll call it a day.

The idle RPG is a cool little game that you play by... not playing. At all. If you play, you get penalties. Here is a cool resource to start: [Idle RPG](https://idlerpg.lolhosting.net). It looks something like this:

```
21:56 <@IdleBot> Verily I say unto thee, the Heavens have burst forth, and the blessed hand of God carried ganome 0 days, 03:52:11 toward level 45.
21:56 <@IdleBot> ganome reaches next level in 0 days, 01:49:16.
22:02 <@IdleBot> himuraken, the level 77 Mage Of BitFlips, is now online from nickname himuraken. Next level in 11 days, 10:35:53.
22:14 <@IdleBot> Nechayev, Sundance, and simple [2011/2347] have team battled HeavyPodda, Sixbierehomme, and L [1417/2717] and won! 0 days, 06:14:54 is removed from their clocks.
22:18 <@IdleBot> canton7 saw an episode of Ally McBeal. This terrible calamity has slowed them 0 days, 05:10:53 from level 85.
22:18 <@IdleBot> canton7 reaches next level in 2 days, 00:21:36.
22:26 <@IdleBot> Tor [4/1142] has challenged Brainiac [232/817] in combat and lost! 3 days, 23:06:05 is added to Tor's clock.
22:26 <@IdleBot> Tor reaches next level in 39 days, 23:39:35.
```

It could happen that by some misfortune the bouncer gets restarted and it doesn't log you back in. Or you simply just lose connection and you don't re-connect. That is unacceptable because the point is to be present. Otherwise you don't play. So you need an early warning in case you are offline. Luckily, IdleRPG provides an XML based endpoint to get which contains your status.

From there, I'm using mailgun with a registered domain to send me an email in case my status is offline. For that, here is a small Go program [IdleRPG Checker Go Code](https://gist.github.com/Skarlso/318ddd6f8d71dbda8fbbd1a908fdb159).

To put that into a Docker container, here is a Dockerfile:

```Dockerfile
FROM golang:latest as build
RUN go get -v github.com/sirupsen/logrus && \
    go get -v github.com/mailgun/mailgun-go
COPY ./main.go /code/
WORKDIR /code
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /idlerpg-checker .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
COPY --from=build /idlerpg-checker /idlerpg-checker
RUN echo "v0.0.1" >> .version
ENTRYPOINT ["/idlerpg-checker"]
```

And the corresponding cronjob resource definition:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: idle-checker
  namespace: idle-checker
spec:
  schedule: "*/20 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: idle-checker
            image: skarlso/idle-checker
            imagePullPolicy: IfNotPresent
            env:
              - name:  MG_API_KEY
                valueFrom:
                  secretKeyRef:
                    name:  idle-rpg-secret
                    key:  MG_API_KEY
              - name:  MG_DOMAIN
                valueFrom:
                  secretKeyRef:
                    name:  idle-rpg-secret
                    key:  MG_DOMAIN
            args: ['-username', 'username', '-email', 'user@powerhouse.com']
          restartPolicy: OnFailure
```

Aaaand, the secret for the API key:

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: idle-checker
  name: idle-rpg-secret
type: Opaque
data:
  MG_API_KEY: asdf=
  MG_DOMAIN: asdf==

```

Done. Huh. This will run every 20 minutes and check if the user with username `username` is online. If not, it will send an email to the given email address. Your levels are safe.

# Closing words

Phew. This has been quite the ride. The post is now really long, so I will split the rest out into a Part 2. That is, Athens and Monitoring.

Thank you for reading this far!

Cheers,
Gergely.