+++
author = "hannibal"
categories = ["go", "tilt", "kubernetes"]
date = "2023-03-12T01:01:00+01:00"
title = "Painless controller testing with e2e-framework and tilt"
url = "/2023/03/12/painless-controller-testing-with-e2e-framework-tilt"
comments = true
+++

Welcome dear reader.

When [last](https://skarlso.github.io/2023/02/25/rapid-controller-development-with-tilt/) we met, we talked a lot about setting up Tilt
for rapid controller development.

Now, let's see how powerful Tilt can be once we bring it together with Kubernetes' [e2e-framework](https://github.com/kubernetes-sigs/e2e-framework).

# Controller E2E Framework

I'd like to present my [controller-e2e-framework](https://github.com/controller-e2e-framework) which brings Tilt and e2e-framework together to easily write and run
tests for controllers that work together.

This framework can be used to integration test or e2e test controllers that work together. They set up some kind of
ref connection between certain objects and perform some operation on said object.

The result should contain the operation of both (or many) controllers.

The linked organization contains three main repositories. Let's walk through them.

## Test 1 Controller

Test 1 controller is a simple controller that accepts a basic object like this:

```yaml
apiVersion: delivery.controller-e2e-framework/v1alpha1
kind: Controller
metadata:
  name: controller-sample
spec:
  ref:
    kind: Responder
    apiVersion: delivery.controller-e2e-framework/v1alpha1
    name: test-responder
```

The `ref` is the interface between test-2-controller and test-1-controller. Its simple operation involves fetching this
reference, whatever the kind is, adding a label to it, and patching that object back to Kubernetes.

The label that it adds is `controller-e2e-framework.controlled = true`.

This repository contains another main ingredient. A simple Tilt file that does the following. It builds the project and
sets up all necessary yaml objects like the CRD definition, RBAC, any patches that might exist and the deployment.

This is important to note for various reasons. First, if we would build a testing image instead of using Tilt, we would
flood the package manager with a bunch of test imagines. Or, always set the `latest` image, in which case, we would lose
the ability to debug any problems. Further, we would have to have some kind of way to download the manifest files that
correspond to that version. If doing local development, that would be a hassle or we would have to have a checked-out
source of the manifest files anyways.

Thus, we bring together all of this with Tilt which uses a local registry and the files from the checked-out source.

This simple Tiltfile looks like this:

```python
# -*- mode: Python -*-

kubectl_cmd = "kubectl"

# verify kubectl command exists
if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

# Use kustomize to build the install yaml files
install = kustomize('config/default')

# Update the root security group. Tilt requires root access to update the
# running process.
objects = decode_yaml_stream(install)
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'test-1-controller':
        o['spec']['template']['spec']['securityContext']['runAsNonRoot'] = False
        break

updated_install = encode_yaml_stream(objects)

# Apply the updated yaml to the cluster.
# Allow duplicates so the e2e test can include this tilt file with other tilt files
# setting up the same namespace.
k8s_yaml(updated_install, allow_duplicates = True)

load('ext://restart_process', 'docker_build_with_restart')

# enable hot reloading by doing the following:
# - locally build the whole project
# - create a docker imagine using tilt's hot-swap wrapper
# - push that container to the local tilt registry
# Once done, rebuilding now should be a lot faster since only the relevant
# binary is rebuilt and the hot swat wrapper takes care of the rest.
local_resource(
    'test-1-controller-binary',
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

# Build the docker image for our controller. We use a specific Dockerfile
# since tilt can't run on a scratch container.
# `only` here is important, otherwise, the container will get updated
# on _any_ file change. We only want to monitor the binary.
# If debugging is enabled, we switch to a different docker file using
# the delve port.
entrypoint = ['/manager']
dockerfile = 'tilt.dockerfile'
docker_build_with_restart(
    'ghcr.io/controller-e2e-framework/test-1-controller',
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

Basic things, plus the modification of the root securityContext so Tilt can hot-swap the binary.

## Test 2 Controller

Test 2 controller looks at a responder object and once it detects the above label on the object, it will set a status
flag called `Acquired` to `true`. Also, a simple operation just to demo how these two controllers work together.

This repository has the same Tiltfile as test-1-controller.

## E2E Framework

Now comes the main menu. The heart of the thing.

First, we can determine the flow. We would like to apply two files. `controller-sample.yaml` and `responder-sample.yaml`.
This will kick-start the whole process of reconciliation of the responder object.

Once these two exists and we verified that they exist, we will wait for the label to be applied and the `Acquired` flag
to be set to true.

Let's look at the architecture of this whole thing.

![architecture](/img/2023/03/12/architecture.png)

What happens here is as follows... e2e-framework has a `TestMain` function defined.

```go
func TestMain(m *testing.M) {
	cfg, _ := envconf.NewFromFlags()
	testEnv = env.NewWithConfig(cfg)
	kindClusterName = envconf.RandomName("test-controller-e2e", 32)
	namespace = envconf.RandomName("namespace", 16)

	testEnv.Setup(
		envfuncs.CreateKindCluster(kindClusterName),
		envfuncs.CreateNamespace(namespace),
		shared.RunTiltForControllers("test-1-controller", "test-2-controller"),
	)

	testEnv.Finish(
		envfuncs.DeleteNamespace(namespace),
		envfuncs.DestroyKindCluster(kindClusterName),
	)

	os.Exit(testEnv.Run(m))
}
```

This function has a `Setup` and a `Teardown` phase. During this phase, a kind cluster is created and `tilt ci` is
executed to bring up a cluster and run tilt. But what Tiltfile does it run? Well... it imports all the Tiltfiles of the
defined controllers.

This part:

```go
shared.RunTiltForControllers("test-1-controller", "test-2-controller"),
```

But where are they? They have to be checked-out next to this project. Then, the framework will do a `cd `..` like a
thing and look for these folders until it either finds them or reaches the root.

`tilt ci` is super convenient because it will run to completion or fail if it doesn't start up for whatever reason. Thus
we can be sure that we set up RBAC and controller settings correctly. Something that we can't test through pure unit
testing.

Once we have a cluster and tilt the test starts. Here we have steps that are either a `Setup` an `Assess` or a
`Teardown`.

We can extract a bunch of steps here to make it easy to just plug together tests using common steps.

Like, setting up a scheme, looking for objects, checking object state, applying test data, etc.

In this test we do something like this:

```go
func TestControllerApply(t *testing.T) {
	t.Log("running component version apply")

	feature := features.New("Custom Controller").
		Setup(setup.AddSchemeAndNamespace(cv1.AddToScheme, namespace)).
		Setup(setup.AddSchemeAndNamespace(rv1.AddToScheme, namespace)).
		Setup(setup.ApplyTestData(v1alpha1.AddToScheme, namespace, "*")).
		Assess("check if controller was created", assess.ResourceWasCreated("controller-sample", namespace, &cv1.Controller{})).
		Assess("check if responder was created", assess.ResourceWasCreated("responder-sample", namespace, &rv1.Responder{})).
		Assess("wait for responder condition to be true", func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
        ...
```

We have then waiters or waiting steps for certain conditions and once everything is tested and done, we remove all
objects in a `Teardown` so the next test doesn't collide with anything.

```go
		Teardown(func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
			t.Helper()
			t.Log("teardown")

			// remove test resources before exiting
			r, err := resources.New(cfg.Client().RESTConfig())
			if err != nil {
				t.Fatal(err)
			}

			if err := decoder.DecodeEachFile(ctx, os.DirFS("./testdata"), "*",
				decoder.DeleteHandler(r),           // try to DELETE objects after decoding
				decoder.MutateNamespace(namespace), // inject a namespace into decoded objects, before calling DeleteHandler
			); err != nil {
				t.Fatal(err)
			}

			t.Log("teardown done")

			return ctx
		}).Feature()
```

This is super convenient because we tested apply, scheme, delete and finalizers, everything in one go.

And the **BEST** part about all this is that the ONLY thing the user needs to run is _just_ `go test ./...`.

You can funk it up if you wish to only run a single suite with `go test -v ./test/controller`.

## CI

To set all this up in a CI that runs it continuously or some kind of trigger just set it up like this:

```yaml
name: e2e

on:
  workflow_dispatch: {}
  schedule:
    - cron: 0 0 * * 1 # every Monday at 00:00

permissions:
  contents: read # for actions/checkout to fetch code

jobs:
  kind-linux:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout this repo
        uses: actions/checkout@v3
        with:
          path: e2e-framework
      - name: Checkout test-1-framework
        uses: actions/checkout@v3
        with:
          repository: 'controller-e2e-framework/test-1-controller'
          path: test-1-controller
     - name: Checkout test-2-controller
       uses: actions/checkout@v3
       with:
         repository: 'controller-e2e-framework/test-2-controller'
         path: test-2-controller
      - name: Setup Go
        uses: actions/setup-go@v3
        with:
          go-version-file: '${{ github.workspace }}/e2e-framework/go.mod'
      - name: Restore Go cache
        uses: actions/cache@v3
        with:
          path: /home/runner/work/_temp/_github_home/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-
      - name: Run tests
        run: cd e2e-framework && go test ./...
```

_Note_ that you need to have checkout access to all repositories involved.

## Conclusion

I find this testing amazingly simple. It lets me focus on the tests and I **know** that I will run the same thing in CI
and my _local___ environment. And I know that any modification I'll do in code will immediately be tested without me
having to either run something weird with exported environment properties such as **TEST_IMAGE** or a dedicated test
runner with some weird parameters. CI runs the same thing I'm running on my local environment. And that's `go test ./...`.

Thank you for reading.

And as always, have a nice day.

Gergely.