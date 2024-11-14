+++
author = "hannibal"
categories = ["go", "tilt", "kubernetes"]
date = "2023-02-25T01:01:00+01:00"
title = "Rapid Kubernetes Controller Development with Tilt"
url = "/2023/02/25/rapid-controller-development-with-tilt"
comments = true
+++

Welcome dear reader.

Today, we are going to dive into how to use [Tilt](https://tilt.dev) to speed up the feedback loop of developing a
Kubernetes controller. We are going to do that using an open-source project called [OCM](https://ocm.software) which has
a controller called [ocm-controller](github.com/open-component-model/ocm-controller). I'm going to walk through the
following process:

- researching tilt
- what it could do for me
- understanding the Tilt file
- trivial mapping of the developer process
- understanding Starlark
- adding more features
- tackling hot swapping
- troubleshooting

Let's dive in.

# Rapid controller development with Tilt

## Tilt

![tilt](/img/2023/02/25/tilt.png)

If you know some things about Tilt already, skip ahead to where I'll be applying its concepts to the controller [here](#development-process-of-an-controller).

### What is Tilt

Tilt is a tool we can use to automate build and deploy processes for microservices. Technically, it can handle so much
more than just a simple controller, but we are going to use this example to keep it simple for a first start.

### What can it do

It is _almost_ like a controller itself. It has an internal control loop that constantly "reconciles" the environment to
make sure it's up-to-date. I'm using quotes because it's not 100% like that. It doesn't recreate if something is
deleted. But it _does_ update them if the configuration changes. For example, if you create a secret with an incorrect
name, and change its name, there won't be two secrets, one with the correct name, and the other with an incorrect name,
but one secret with the correct name.

### Reading the documentation - where to start

![document-hell](/img/2023/02/25/documentation-hell.png)

You can start by visiting the [docs](https://docs.tilt.dev) and then jumping right into [First Look At Tilt](https://docs.tilt.dev/tutorial/index.html).

This will guide you through setting up Tilt. There is one thing I don't like here... It's starting by using a demo
application and a special command called `tilt demo`. I believe that this is not a great way to begin because you will
_never_ run this command again. I would have preferred to do a walkthrough like the one I'm going to write now. But,
this is _my_ view. Others might find the demo app more useful. Nevertheless, it's useful to read through it to understand
the basic concepts and the application (avatar generator) is pretty sweet!

#### Concepts

You can read all about the concepts [here](https://docs.tilt.dev/tiltfile_concepts.html).

TL;DR:

Tilt uses building blocks called [Resources](https://docs.tilt.dev/tiltfile_concepts.html#resources) that it manages.
A resource can be anything from a Deployment or a local file or a script or a service. It does that so it can track
them separately and display them in a shiny UI.

#### Control Loop

The control loop is described [here](https://docs.tilt.dev/tutorial/2-tilt-up.html#the-control-loop).

TL;DR:

It's a feedback loop around things that happen and things that Tilt watches. Tilt continuously watches resources and
reacts to their changes in ways that are described in the Tiltfile. Another interesting effect of this is that if a
resource is already running in your cluster it won't try to rebuild/redeploy that.

#### Tiltfile

We are going to dive deep into the Tiltfile a bit later, for now, it's the main file that describes the behavior of tilt.
It's using [Starlark](https://github.com/bazelbuild/starlark) which is a dialect of Python. Here is an example:

```python
# Define a number
number = 18

# Define a dictionary
people = {
    "Alice": 22,
    "Bob": 40,
    "Charlie": 55,
    "Dave": 14,
}

names = ", ".join(people.keys())  # Alice, Bob, Charlie, Dave

# Define a function
def greet(name):
    """Return a greeting."""
    return "Hello {}!".format(name)

greeting = greet(names)

above30 = [name for name, age in people.items() if age >= 30]

print("{} people are above 30.".format(len(above30)))

def fizz_buzz(n):
    """Print Fizz Buzz numbers from 1 to n."""
    for i in range(1, n + 1):
        s = ""
        if i % 3 == 0:
            s += "Fizz"
        if i % 5 == 0:
            s += "Buzz"
        print(s if s else i)

fizz_buzz(20)
```

This syntax is used for configuration. As you can see, it supports various Python things which give the user the ability
to do virtually anything in Tilt.

- download yaml manifests
- mutate objects
- changes deployments before applying them ( this will be important later )
- run a local script
- fetch something from a remote location

Anything is possible.

A Tiltfile has a bunch of built-in functions and APIs which are described [here](https://api.tilt.dev/).

However, I found that [THIS API](https://docs.tilt.dev/api.html) page will be your best friend together with the
extensions list [Tilt Extensions](https://github.com/tilt-dev/tilt-extensions). Tilt's functionality is extended using
extensions that are loaded using an expression like this `load('ext://restart_process', 'docker_build_with_restart')`.

During troubleshooting, reading the extension's code or documentation can often be helpful.

The [Writing your first Tiltfile](https://docs.tilt.dev/tiltfile_authoring.html) guide is pretty good though for
demystifying and understanding the dialect. It's rather basic but that's all you need to get started. So next, let's
dive into how to gradually build up a complex Tiltfile and development process.

## The development process of a controller

![yaml-hell](/img/2023/02/25/yaml-hell.png)

### Basic flow

When you are building a controller you have a couple of choices:

- run the controller locally on your machine by calling the compile manager binary
Most of the time you'd do this but by doing this, you are skipping testing in cluster communication, RBAC, service
discovery, etc. Also, it gets cumbersome to constantly port-forward other services being used by your controller to
access them.

- build a container and push it to a registry and publish the deployment with the given container image and tag
This is good if you want to test the release process of new containers. But takes a lot of time.

- use a local docker registry with kind (https://kind.sigs.k8s.io/docs/user/local-registry/) and do a cycle of constant
build/push/deploy like this

```
docker build -t localhost:5001/open-component-model/ocm-controller:v0.0.1 .
...
docker push localhost:5001/open-component-model/ocm-controller:v0.0.1
```

All of these take time. Building a docker container, even when it's reusing layers, takes time. For `ocm-controller`, on
my machine, it takes about two minutes or so. Then push and deploy takes another couple of seconds.

Tilt can help a lot in speeding up this process. We'll see about that later.

### Mapping flow to Tiltfile

![tilt-with-the-flow](/img/2023/02/25/tilt-go-with-the-flow.png)

We are going to choose the third option. Tilt has its local registry and uses unique build tags for each container to be
able to debug problems for a specific change.

As a first step, to keep it simple, we map the process of installing controller resources and deploying them using
`docker_build``.

Usually, a controller uses `kustomize`. Tilt can leverage that. We can use `kustomize` to build a single install file
that will create the deployment, the RBAX roles, and any other thing, like patches, configuration and the CRDs.

To do this, we simply run `kustomize` build ./config/default > install.yaml`. The `install.yaml` will contain everything
that can be applied to the cluster as is. Tilt can do the same thing using a very simple directive. Take a look at our
first draft of a Tiltfile:

```python
# Apply the install file to the cluster
k8s_yaml(kustomize('config/default'))

docker_build('ghcr.io/open-component-model/ocm-controller', '.')
```

By defining the `docker_build` directive we make it possible for Tilt to use a local registry. Internally, it will do its
magic and change the deployment's image to the right registry address.

In reality, this is the actual deployment that will be used:

```yaml
Image:      localhost:5001/ghcr.io_open-component-model_ocm-controller:tilt-772cb281aa593470
```

Now, we can try and test our Tiltfile. To use tilt, simply run `tilt up` next to the Tiltfile.
Press `space` and you should see your deployment starting up. Any errors will nicely be displayed on the respective
resource.

If we make any changes to any of the files now, tilt will rebuild the entire container. This is already nice because
this means that you don't have to bother with building, pushing and updating the kustomize manifest to use the local
registry. Tilt will take care of all that for us.

But wait... there is more!

### Introducing settings

For now, let's upgrade our Tilt experience with a couple of settings so any user can customize their deployment for
themselves. Starlark is just a Python dialect so we can do something like this:

```python
# -*- mode: Python -*-

kubectl_cmd = "kubectl"

# verify kubectl command exists
if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

# set defaults
settings = {
    "create_secrets": {
        "enable": True,
        "token": os.getenv("GITHUB_TOKEN", ""),
        "email": os.getenv("GITHUB_EMAIL", ""),
        "user": os.getenv("GITHUB_USER", ""),
    },
    "forward_registry": False,
    "verification_keys": {},
}

# global settings
tilt_file = "./tilt-settings.yaml" if os.path.exists("./tilt-settings.yaml") else "./tilt-settings.json"
settings.update(read_yaml(
    tilt_file,
    default = {},
))
```

Okay, _what_ is happening here? Let's break it down.

`kubectl_cmd` is just a convenient variable so we don't have to repeat the command as a string.

The next section checks whether the `kubectl` command exists. We are using a tilt built-in command called `local` to
execute a local command called `command`.

Now comes the interesting part. We define a dict. This is something akin to dict, but it's a Starlark dict. You can
check the dialect definition in the [Specification](https://github.com/bazelbuild/starlark/blob/master/spec.md).
This doc will also be your best friend. The [dict](https://github.com/bazelbuild/starlark/blob/master/spec.md#dict) can be found here.

Next, we define the variable `tilt_file` and assign it either `json` or a `yaml` extension with the name `tilt-settings`.
After this comes another built-in directive called `read_yaml`. This is a versatile directive that will produce a yaml
output from the given file. If the file is not found it will use `default` to set some default values. `update` will
merge the two produced dict results.

So, with a local file like this:

```yaml
create_secrets:
  enabled: false
```

We'll get a dict like this:

```python
settings = {
    "create_secrets": {
        "enable": False,
    },
    "forward_registry": False,
    "verification_keys": {},
}
```

We can override any values in our settings and customize the deployment.

### Adding more features

![feature-hell](/img/2023/02/25/feature-hell.png)

Let's add some more features. ocm uses [flux](https://github.com/fluxcd/flux2) for deployment and various other tasks.
To help the user, add a simple flux bootstrap OR install. Maybe the user doesn't want to bootstrap a repository but
does want flux CRDs and components installed in the cluster. The two commands we would like to replicate are:

```
flux bootstrap github --owner=skarlso --repository=ocm-flux-example --path .
```

and simply

```
flux install
```

... depending on the user's choice. First, we'll extend our settings:

```python
settings = {
    "flux": {
        "enabled": False,
        "bootstrap": True,
        "repository": os.getenv("FLUX_REPOSITORY", "podinfo-flux-example"),
        "owner": os.getenv("FLUX_OWNER", ""),
        "path": os.getenv("FLUX_PATH", "."),
    },
    "create_secrets": {
        "enable": True,
        "token": os.getenv("GITHUB_TOKEN", ""),
        "email": os.getenv("GITHUB_EMAIL", ""),
        "user": os.getenv("GITHUB_USER", ""),
    },
    "forward_registry": False,
    "verification_keys": {},
}
```

Here, we used another built-in directive with `os.getenv`. These could just be empty, but we give the option of users
to define environment properties instead of a file. Whichever is more convenient.

Now, we'll add the flux commands:

```python
flux_cmd = 'flux'

opts = settings.get("flux")
if opts.get("enabled"):
    if str(local("command -v " + flux_cmd + " || true", quiet = True)) == "":
        fail("Required command '" + flux_cmd + "' not found in PATH")

    if opts.get("bootstrap"):
        local("%s bootstrap github --owner %s --repository %s --path %s" % (flux_cmd, opts.get('owner'), opts.get('repository'), opts.get('path')))
    else:
        local(flux_cmd + " install")
```

Super easy. This feels familiar, doesn't it? It's just like when you first began using something like Ansible to code down
some deployment or translate some weird script into a higher-order definition or a DSL.

Let's go one step further and try to make this nicer to read by using functions.

```python
def bootstrap_or_install_flux():
    opts = settings.get("flux")
    if not opts.get("enabled"):
        return

    if str(local("command -v " + flux_cmd + " || true", quiet = True)) == "":
        fail("Required command '" + flux_cmd + "' not found in PATH")

    # flux bootstrap github --owner=${FLUX_OWNER} --repository=${FLUX_REPOSITORY} --path ${FLUX_PATH}
    if opts.get("bootstrap"):
        local("%s bootstrap github --owner %s --repository %s --path %s" % (flux_cmd, opts.get('owner'), opts.get('repository'), opts.get('path')))
    else:
        local(flux_cmd + " install")
```

How does our complete Tiltfile now look like, you might wonder. Let's take a look:

```python
# -*- mode: Python -*-

kubectl_cmd = "kubectl"
flux_cmd = "flux"

# verify kubectl command exists
if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

# set defaults
settings = {
    "flux": {
        "enabled": False,
        "bootstrap": True,
        "repository": os.getenv("FLUX_REPOSITORY", "podinfo-flux-example"),
        "owner": os.getenv("FLUX_OWNER", ""),
        "path": os.getenv("FLUX_PATH", "."),
    },
    "forward_registry": False,
    "verification_keys": {},
}

# global settings
tilt_file = "./tilt-settings.yaml" if os.path.exists("./tilt-settings.yaml") else "./tilt-settings.json"
settings.update(read_yaml(
    tilt_file,
    default = {},
))


def bootstrap_or_install_flux():
    opts = settings.get("flux")
    if not opts.get("enabled"):
        return

    if str(local("command -v " + flux_cmd + " || true", quiet = True)) == "":
        fail("Required command '" + flux_cmd + "' not found in PATH")

    # flux bootstrap github --owner=${FLUX_OWNER} --repository=${FLUX_REPOSITORY} --path ${FLUX_PATH}
    if opts.get("bootstrap"):
        local("%s bootstrap github --owner %s --repository %s --path %s" % (flux_cmd, opts.get('owner'), opts.get('repository'), opts.get('path')))
    else:
        local(flux_cmd + " install")


# set up the development environment

# check if flux is needed
bootstrap_or_install_flux()

# Use kustomize to build the install yaml files
k8s_yaml(kustomize('config/default'))

docker_build('ghcr.io/open-component-model/ocm-controller', '.')

# This is new thing we added here to port-forward more on this a bit later.
if settings.get('forward_registry'):
    k8s_resource('ocm-controller', extra_pod_selectors = [{'app': 'registry'}], port_forwards=5000)
```

This looks pretty nice already. But there is a lot more that we can do.

#### Adding secrets

![secrets](/img/2023/02/25/secrets.jpg)

Another thing that OCM needs ( or any other deployment really ) is creating secrets for various things and deployments.
OCM needs some docker registry credentials and some other credentials to pull things from outside repositories.

Creating secrets can be done by using an extension. The particular extension we are looking for is [secrets](https://github.com/tilt-dev/tilt-extensions/tree/master/secret).

We will use three types. A `from-file` for creating a secret that contains an SSH key. A plain key/value secret for
repository access. And a docker-registry type secret.

```python
# set defaults
settings = {
    "flux": {
        "enabled": False,
        "bootstrap": True,
        "repository": os.getenv("FLUX_REPOSITORY", "podinfo-flux-example"),
        "owner": os.getenv("FLUX_OWNER", ""),
        "path": os.getenv("FLUX_PATH", "."),
    },
    "install_unpacker": {
        "enabled": False,
        "path": "",
    },
    # Added this one
    # +++++++++++++++
    "create_secrets": {
        "enable": True,
        "token": os.getenv("GITHUB_TOKEN", ""),
        "email": os.getenv("GITHUB_EMAIL", ""),
        "user": os.getenv("GITHUB_USER", ""),
    },
    # +++++++++++++++
    "forward_registry": False,
    "verification_keys": {},
}

# ... other stuff ...

def create_secrets():
    opts = settings.get("create_secrets")
    if not opts.get("enable"):
        return

    k8s_yaml(secret_yaml_registry("regcred", "ocm-system", flags_dict = {
        'docker-server': 'ghcr.io',
        'docker-username': opts.get('user'),
        'docker-email': opts.get('email'),
        'docker-password': opts.get('token'),
    }))
    k8s_yaml(secret_from_dict("creds", "ocm-system", inputs = {
        'username' : opts.get('user'),
        'password' : opts.get('token'),
    }))

def create_verification_keys():
    keys = settings.get("verification_keys")
    if not keys:
        return

    for key, value in keys.items():
        secret_create_generic(key, 'ocm-system', from_file=value)
```

We added two new functions and an extension to the settings. Let's break this down.

`k8s_yaml(secret_yaml_registry` -> this directive creates a yaml output which we apply to the cluster. Its kubectl
equivalent would be
```
kubectl create secret docker-registry -n ocm-system regcred \
    --docker-server=ghcr.io \
    --docker-username=$GITHUB_USER \
    --docker-password=$GITHUB_TOKEN \
    --docker-email=$GITHUB_EMAIL
```

`k8s_yaml(secret_from_dict` -> same here. Equivalent kubectl command: `kubectl create secret generic creds --from-literal=username=$GITHUB_USER --from-literal=password=$GITHUB_TOKEN -n ocm-system`

Next, we have `create_verification_keys`. This is a bit trickier, but nothing too fancy. Verification keys are SSH keys
that can be used to verify components. We can have multiple keys. Therefore, the setting is a dict. We loop through the
dict and create a secret from each of those. This is what that setting could look like:

```yaml
verification_keys:
  alice: alice=./rsa.pub
  bob: bob=/Users/user/home/.ssh/bob.pub
```

All valid values. If a key is missing or the path is wrong, tilt does a dry-run first, so before applying to the cluster
there would be an error that the key cannot be found.

Phew, this was quite a lot. But now, we have secrets. Next, is the most important topic of them all...

#### Hot-Swapping

Hot-swapping is the most important ability of tilt of all of them. Instead of rebuilding the whole container it can
just rebuild the binary and switch out the running process. This is nothing new but combined with Kubernetes and
the controller development cycle will provide us with valuable time saved.

To do that though, we first, need to give Tilt some new abilities.

##### Letting Tilt build the controller

Tilt needs to manage the building of the binary for our project. Lucky for me, ocm is pretty basic in its design.
Doesn't require anything too weird. Not that tilt doesn't support _weird_, it does, with custom builds, but for me, this wasn't needed.

And I suspect for most controllers that is the case. Let's add something called a `local_resource`.

```python
local_resource(
    'manager',
    'CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/manager ./',
    deps = [
        "main.go",
        "go.mod",
        "go.sum",
        "api",
        "controllers",
        "pkg",
    ],
)
```

A `local_resource` is similar to `local`. It executes a command on the host computer. However, it also creates a resource.
Remember, that a resource is any unit of work that Tilt manages. To learn more read up [here](https://docs.tilt.dev/local_resource.html). The extra here is that whenever something changes in `deps` it will re-run the command and rebuild the resource.

Here is the API definition of [local_resource](https://docs.tilt.dev/api.html#api.local_resource).

This means that whenever something changes in `deps`, like files in API or controllers or pkg, etc, tilt rebuilds our
binary. It already monitors the component YAML files, now we also monitor the go files.

We can add any script here or anything that produces our binary.

Now, we'll modify our docker build. There are some gotcha's there.

##### Root access

The first gotcha is that Tilt requires root access to hot-swap processes. If you are doing it right, you are using a
scratch container and something like this in your deployment:

```yaml
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
```

If you aren't you really should be! First, problem is that scratch containers aren't supported by Tilt because it expects
certain commands to be in the container, like `touch`. So we add a very simple Dockerfile that only tilt will use.

```Dockerfile
FROM alpine
WORKDIR /
COPY ./bin/manager /manager

ENTRYPOINT ["/manager"]
```

And, we have to get rid of the `runAsNonRoot: true`. Or rather, set it to false. Remember we are writing code here.
So we just take the yaml, change that field, and apply it after we are finished updating the object. There is a little
doc about that that you can read: [Modifying YAML for Dev](https://docs.tilt.dev/templating.html).

Let's take a look. The basic flow is:

- read in our kustomize files
- convert it to objects
- look for our deployment and change this setting
- compile it back to yaml
- apply

We could also apply the objects separately but it's nice to do it with the install yaml because it contains the CRDs and
Kubernetes applies those first. It's a server-side apply.

```python
# Use kustomize to build the install yaml files
install = kustomize('config/default')

# Update the root security group. Tilt requires root access to update the
# running process.
objects = decode_yaml_stream(install)
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'ocm-controller':
        o['spec']['template']['spec']['securityContext']['runAsNonRoot'] = False
        break

updated_install = encode_yaml_stream(objects)

# Apply the updated yaml to the cluster.
k8s_yaml(updated_install)
```

And thus, if our kind is `Deployment` and the name is the one we are looking for `ocm-controller` we change that value
under `spec: template: spec: securityContext:`. DONE. Next!

##### With restart

Now that we have the right Dockerfile and the right deployment, let's use another extension called `restart_process`.

```python
load('ext://restart_process', 'docker_build_with_restart')
```

Load the extension and then change the `docker_build` to `docker_build_with_restart` like this:

```python
docker_build_with_restart(
    'ghcr.io/open-component-model/ocm-controller',
    '.',
    dockerfile = 'tilt.dockerfile',
    entrypoint = ['/manager'],
    only=[
      './bin',
    ],
    live_update = [
        sync('./bin/manager', '/manager'),
    ],
)
```

Let's break this down. We have two pretty straightforward things. The first is the name of the container.
The second is the context. The third is `dockerfile` our custom `alpine` container.

Fourth is `entrypoint`. Now comes the `only` part and the `live_update`. The `only` part here is **important**. This
tells Tilt to only watch that folder for changes. Since the `local_resource` is already watching for changes to the
binary the deployment only needs to care about that!

`live_update` tells Tilt what to change in the container. We copy from the local `bin/`manager` to the container `/manager`.
It can also execute a script in the container, but we don't need that here. For example, a `restart.sh`. Or, in the case
of python a `pip install -r requirements.txt` or something like that.

The rest is taken care of by a Tilt wrapper; a bash script that can swap out processes. That's what
`docker_build_with_restart` does. It wraps our container and Dockerfile into its container and runs our `entrypoint`.

#### Attaching a debugger

![debugger](/img/2023/02/25/tilt-debug.png)

> But I'm running the manager locally because I use a debugger.

I hear you! And it's possible with Tilt as well! We need to do a couple of things to achieve running delve correctly.
Luckily, once set up, it should work for the foreseeable future.

First, let's alter our settings and include a debug value:

```python
# set defaults
settings = {
    "flux": {
        "enabled": False,
        "bootstrap": True,
        "repository": os.getenv("FLUX_REPOSITORY", "podinfo-flux-example"),
        "owner": os.getenv("FLUX_OWNER", ""),
        "path": os.getenv("FLUX_PATH", "."),
    },
    "install_unpacker": {
        "enabled": False,
        "path": "",
    },
    "debug": {
        "enabled": False,
    },
    "create_secrets": {
        "enable": True,
        "token": os.getenv("GITHUB_TOKEN", ""),
        "email": os.getenv("GITHUB_EMAIL", ""),
        "user": os.getenv("GITHUB_USER", ""),
    },
    "forward_registry": False,
    "verification_keys": {},
}
```

Once enabled, for simplicity, we will forward delve on port `30000`.

Next, we need a dedicated Dockerfile that installs and runs `delve`.

```Dockerfile
FROM golang:1.20.1
WORKDIR /
COPY ./bin/manager /manager

RUN go install github.com/go-delve/delve/cmd/dlv@latest
RUN chmod +x /go/bin/dlv
RUN mv /go/bin/dlv /

EXPOSE 30000

ENTRYPOINT ["/dlv", "--listen=:30000", "--api-version=2", "--headless=true", "--continue=true", "--accept-multiclient=true", "exec", "/manager", "--"]
```

The two important settings here are `--continue` and `--accept-multiclient`. Without it, delve will stop the process and
our readiness and liveliness probes will fail. Another **super** important part is `--` at the end. Without it, our
`manager` will not get the arguments passed in by the deployment. Or rather, it will try to pass them to `delve`.

With delve in place, now we get to the funky part. We need to tell Tilt to use our debug Dockerfile and change the
`entrypoint` if debugging is enabled. Also, let's not forget that the deployment needs to port-forward `30000` otherwise
we can't access it with our debugger. Another thing we SHOULD do is, alter the Go build process and add some flags.
Usually, when debugging Go code you want the flags `-N -l`. We add that with a variable called `gcflags`.

```python
# ... other things ...
# Update the root security group. Tilt requires root access to update the
# running process.
objects = decode_yaml_stream(install)
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'ocm-controller':
        o['spec']['template']['spec']['securityContext']['runAsNonRoot'] = False
        # This is what we added. OCM has a single container so we don't need to look for a specific one.
        if settings.get('debug').get('enabled'):
            o['spec']['template']['spec']['containers'][0]['ports'] = [{'containerPort': 30000}]
        break

# ... other things ....

# Update our Go build and add conditional gcflags.
gcflags = ''
if settings.get('debug').get('enabled'):
    gcflags = '-N -l'

local_resource(
    'manager',
    "CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -gcflags '{gcflags}' -o bin/manager ./".format(gcflags=gcflags),
    deps = [
        "main.go",
        "go.mod",
        "go.sum",
        "api",
        "controllers",
        "pkg",
    ],
)

# ... other things ...

# Finally, our port-forwarding and the addition of the updated entrypoint and dockerfile.
# The `[]` syntax on entrypoint is important because we want tilt to be able to append to it.
entrypoint = ['/manager']
dockerfile = 'tilt.dockerfile'
if settings.get('debug').get('enabled'):
    k8s_resource('ocm-controller', port_forwards=[
        port_forward(30000, 30000, 'debugger'),
    ])
    entrypoint = ['/dlv', '--listen=:30000', '--api-version=2', '--continue=true', '--accept-multiclient=true', '--headless=true', 'exec', '/manager', '--']
    dockerfile = 'tilt.debug.dockerfile'


docker_build_with_restart(
    'ghcr.io/open-component-model/ocm-controller',
    '.',
    dockerfile = dockerfile,
    entrypoint = entrypoint,
    only=[
      './bin',
    ],
    live_update = [
        sync('./bin/manager', '/manager'),
    ],
)
```

Now, to attach to the debugger in GoLand follow this guide: [Attach To Remote Debugger](https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html#step-3-create-the-remote-run-debug-configuration-on-the-client-computer).

The only important piece there is to clear the **Shutdown remote debugger on disconnect** flag.

In VSCode:

```json
    {
        "name": "Connect to Container",
        "type": "go",
        "request": "attach",
        "mode": "remote",
        "remotePath": "",
        "port": 30000,
        "host": "127.0.0.1",
        "showLog": true,
        "trace": "log",
        "logOutput": "rpc"
    }
```

And that's it! We got delve working.

Let's take a look at the full Tiltfile.

```python
# -*- mode: Python -*-

kubectl_cmd = "kubectl"
flux_cmd = "flux"

# verify kubectl command exists
if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

# set defaults
settings = {
    "flux": {
        "enabled": False,
        "bootstrap": True,
        "repository": os.getenv("FLUX_REPOSITORY", "podinfo-flux-example"),
        "owner": os.getenv("FLUX_OWNER", ""),
        "path": os.getenv("FLUX_PATH", "."),
    },
    "install_unpacker": {
        "enabled": False,
        "path": "",
    },
    "debug": {
        "enabled": False,
    },
    "create_secrets": {
        "enable": True,
        "token": os.getenv("GITHUB_TOKEN", ""),
        "email": os.getenv("GITHUB_EMAIL", ""),
        "user": os.getenv("GITHUB_USER", ""),
    },
    "forward_registry": False,
    "verification_keys": {},
}

# global settings
tilt_file = "./tilt-settings.yaml" if os.path.exists("./tilt-settings.yaml") else "./tilt-settings.json"
settings.update(read_yaml(
    tilt_file,
    default = {},
))
load('ext://secret', 'secret_yaml_registry', 'secret_from_dict', 'secret_create_generic')


def bootstrap_or_install_flux():
    opts = settings.get("flux")
    if not opts.get("enabled"):
        return

    if str(local("command -v " + flux_cmd + " || true", quiet = True)) == "":
        fail("Required command '" + flux_cmd + "' not found in PATH")

    # flux bootstrap github --owner=${FLUX_OWNER} --repository=${FLUX_REPOSITORY} --path ${FLUX_PATH}
    if opts.get("bootstrap"):
        local("%s bootstrap github --owner %s --repository %s --path %s" % (flux_cmd, opts.get('owner'), opts.get('repository'), opts.get('path')))
    else:
        local(flux_cmd + " install")


def install_unpacker():
    opts = settings.get("install_unpacker")
    if not opts.get("enabled"):
        return


def create_secrets():
    opts = settings.get("create_secrets")
    if not opts.get("enable"):
        return

    k8s_yaml(secret_yaml_registry("regcred", "ocm-system", flags_dict = {
        'docker-server': 'ghcr.io',
        'docker-username': opts.get('user'),
        'docker-email': opts.get('email'),
        'docker-password': opts.get('token'),
    }))
    k8s_yaml(secret_from_dict("creds", "ocm-system", inputs = {
        'username' : opts.get('user'),
        'password' : opts.get('token'),
    }))


def create_verification_keys():
    keys = settings.get("verification_keys")
    if not keys:
        return

    for key, value in keys.items():
        secret_create_generic(key, 'ocm-system', from_file=value)

# set up the development environment

# check if flux is needed
bootstrap_or_install_flux()

# check if installing unpacker is needed
install_unpacker()

# Use kustomize to build the install yaml files
install = kustomize('config/default')

# Update the root security group. Tilt requires root access to update the
# running process.
objects = decode_yaml_stream(install)
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'ocm-controller':
        o['spec']['template']['spec']['securityContext']['runAsNonRoot'] = False
        if settings.get('debug').get('enabled'):
            o['spec']['template']['spec']['containers'][0]['ports'] = [{'containerPort': 30000}]
        break

updated_install = encode_yaml_stream(objects)

# Apply the updated yaml to the cluster.
k8s_yaml(updated_install)

# Create Secrets
create_secrets()
create_verification_keys()

load('ext://restart_process', 'docker_build_with_restart')

# enable hot reloading by doing the following:
# - locally build the whole project
# - create a docker imagine using tilt's hot-swap wrapper
# - push that container to the local tilt registry
# Once done, rebuilding now should be a lot faster since only the relevant
# binary is rebuilt and the hot swat wrapper takes care of the rest.
gcflags = ''
if settings.get('debug').get('enabled'):
    gcflags = '-N -l'

local_resource(
    'manager',
    "CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -gcflags '{gcflags}' -o bin/manager ./".format(gcflags=gcflags),
    deps = [
        "main.go",
        "go.mod",
        "go.sum",
        "api",
        "controllers",
        "pkg",
    ],
)

# Build the docker image for our controller. We use a specific Dockerfile
# since tilt can't run on a scratch container.
# `only` here is important, otherwise, the container will get updated
# on _any_ file change. We only want to monitor the binary.
# If debugging is enabled, we switch to a different docker file using
# the delve port.
entrypoint = ['/manager']
dockerfile = 'tilt.dockerfile'
if settings.get('debug').get('enabled'):
    k8s_resource('ocm-controller', port_forwards=[
        port_forward(30000, 30000, 'debugger'),
    ])
    entrypoint = ['/dlv', '--listen=:30000', '--api-version=2', '--continue=true', '--accept-multiclient=true', '--headless=true', 'exec', '/manager', '--']
    dockerfile = 'tilt.debug.dockerfile'


docker_build_with_restart(
    'ghcr.io/open-component-model/ocm-controller',
    '.',
    dockerfile = dockerfile,
    entrypoint = entrypoint,
    only=[
      './bin',
    ],
    live_update = [
        sync('./bin/manager', '/manager'),
    ],
)


if settings.get('forward_registry'):
    k8s_resource('ocm-controller', extra_pod_selectors = [{'app': 'registry'}], port_forwards=5000)
```

That last line extracts a resource that Tilt doesn't control. But we can say that it's the resource `ocm-controller` by
defining a label selector.

### Numbers

So what did we improve after ALL of this hassle? Well, let's see. Before, we needed around 2 minutes to build and deploy
then test something. With Tilt, now it's 10-15 SECONDS tops! That is a huge improvement in the feedback loop.

### When it gets annoying

Any drawbacks? Well, if you are using GoLand or save changes frequently, it will rebuild. On. Each. Save. That can be
pretty annoying. Luckily, Tilt got you covered. If you would like to pause for a second to do some rapid-fire changes,
disable the `local_resource` ( whatever you named it, for me it's `manager` ) that is the Go Build. If you do that, no
changes will be reconciled, which means, the container will not be hot-swapped either.

Looks like this:

![tilt-disabled](/img/2023/02/25/tilt-disabled.png)

Once re-enabled, you can continue your development cycle.

## Conclusion

Tilt is not easy to set up at times. Especially when you have to fiddle with a lot of access and custom-built stuff.
The syntax and view of a complex Tiltfile can be rather intimidating. Some structure adds easier ways of reading, but
sadly, this is seldom the case.

However, it can be helped. A nice Tiltfile with rightly named functions and a good structure can help a lot in readability.
Also, comments. And I can't stress this enough. Add comments! They help a lot for a new person reading the file and
understanding what is being altered why and where.

The Tiltfile as it looks like at the writing of this post can be found [here](https://github.com/open-component-model/ocm-controller/blob/95b0d839d86f643f0f10304da97bcd042e8c0dfc/Tiltfile).

I hope this helped, and good luck!

As always,

Thank you for reading,
Gergely.