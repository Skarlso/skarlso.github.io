+++
author = "hannibal"
categories = ["kubernetes", "cert-manager"]
date = "2023-10-25T01:01:00+01:00"
title = "Self-Signed locally trusted certificates with cert-manager"
url = "/2023/10/25/self-signed-locally-trusted-certificates-with-cert-manager"
comments = true
+++

# Self-Signed locally trusted certificates with cert-manager

We are going to discuss how to set up a Kubernetes environment where components can run using HTTPS without pain.

## Premise

Usually, people either generate certificates outside the cluster using either openssl, or mkcert, then mount them in or
use those as seeds for further generation. This poses a number of problems during testing and distribution of these
certificates. And then, switching to production, it proves that local certs will either no longer work or pose even
more problems in getting them properly distributed again.

## Proposition

Start with cert-manager from the begin. This will give the flexibility needed to move into production immediately, and
the ability to set up a local testing environment quickly and efficiently. I will demonstrate how to achieve this with
minimal pain and no interaction needed aside from a password confirm when starting the test. I will also show how to set
all this up using a Github Action for e2e testing on CI environment.

## Breakdown

First, let's take a look at two deployments to demonstrate this set up. The first deployment we'll be using is a plain
docker registry. It requires some certs to be mounted in so it can run using HTTPS.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          env:
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
            - name: REGISTRY_HTTP_TLS_CERTIFICATE
              value: "/certs/cert.pem"
            - name: REGISTRY_HTTP_TLS_KEY
              value: "/certs/key.pem"
            - name: REGISTRY_HTTP_TLS_CLIENTCAS_0
              value: "/certs/ca.pem"
          volumeMounts:
            - mountPath: "/certs"
              name: "root-certs"
      volumes:
        - name: registry
          emptyDir: {}
        - name: "root-certs"
          secret:
            secretName: "root-certs"
            items:
              - key: "tls.crt"
                path: "cert.pem"
              - key: "tls.key"
                path: "key.pem"
              - key: "ca.crt"
                path: "ca.pem"
```

Nothing too fancy.

The second pod will be a plain ubuntu that runs a curl to this service to verify that it is indeed HTTPS.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl-container
        image: curlimages/curl:latest
        command: ["curl", "https://registry.default.svc.cluster.local:5000"]
```

Now, let's assume that the registry already has the certificate, this deployment now would fail with something like this:

```
kubectl logs "$(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=curl)"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Because the pod is lacking the root certificate for our registry. So far so good. Now, let's see how we can make this
work automatically.

### Setting up

First, we need to configure cert-manager to create a root certificate for us. Now, cert-manager creates root certificates
by the means of marking a certificate as a CA. This, sometimes, does not work well with certain browsers. For example,
Firefox doesn't like certificates that are also root CAs. I'm willing to accept this sacrifice in the test environment.

To prime cert-manager to generate a root certificate for us, all we need to apply this combo:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-selfsigned-ca # deprecated but still requires in iOS environment
  secretName: root-secret
  subject: # needed later for local trust store
    organizations:
      - example.com
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: root-secret
```

We are going to start to put this into a setup script at this point. This script will work for us to prime a test
cluster.

```bash
#!/usr/bin/env bash

# cleanup
rm -fr rootCA.pem

# install cert-manager with a specific version.
CERT_MANAGER_VERSION=${CERT_MANAGER_VERSION:-v1.13.1}

# if the manifest hasn't been downloaded yet, download it.
if [ ! -e 'cert-manager.yaml' ]; then
  echo "fetching cert-manager manifest for version ${CERT_MANAGER_VERSION}"
  curl -L https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml -o cert-manager.yaml
fi

# create a test cluster -- use any configuration here for kind
kind create cluster --name=e2e-test-cluster

echo -n 'installing cert-manager'
# apply the downloaded manifest then wait for all the deployments to become ready.
kubectl apply -f cert-manager.yaml
kubectl wait --for=condition=Available=True Deployment/cert-manager -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-webhook -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-cainjector -n cert-manager --timeout=60s
echo 'done'

# apply the cluster_issuer combination that was defined above this.
echo -n 'applying root certificate issuer'
kubectl apply -f cluster_issuer.yaml
echo 'done'

# wait for the certificate to be generated.
echo -n 'waiting for root certificate to be generated...'
kubectl wait --for=condition=Ready=true Certificate/mpas-bootstrap-certificate -n cert-manager --timeout=60s
echo 'done'
```

Now, when running our e2e test from a Makefile for example, we can easily use this script as a primer. Consider the
following target for a Go project:

```Makefile
.PHONY: e2e
e2e: prime-test-cluster ## Runs e2e tests
	go test -count=1 -tags=e2e ./e2e

.PHONY: prime-test-cluster
prime-test-cluster:
	./prime_test_cluster.sh
```

Simple Makefile dependency so it primes the test cluster first, than executes our e2e tests.

We aren't done yet. We do have a certificate root, but we don't have a certificate. We could use the rootCA, but I
would rather create a new certificate with the rootCA so each has its own access. To generate a new certificate we apply
a `Certificate`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: curl-certs
spec:
  secretName: curl-certs
  dnsNames:
    - registry.default.svc.cluster.local
    - localhost
  ipAddresses:
    - 127.0.0.1
    - ::1
  privateKey:
    algorithm: RSA
    encoding: PKCS8
    size: 2048
  issuerRef:
    name: my-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

With this, we have our certificate and we can mount it into our CURL deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl-container
        image: curlimages/curl:latest
        env:
          - name: CURL_CA_BUNDLE
            value: "/etc/ssl/certs/registry-root.pem"
        command: ["curl", "https://registry.default.svc.cluster.local:5000"]
        volumeMounts:
        - mountPath: "/etc/ssl/certs/registry-root.pem"
          subPath: "registry-root.pem"
          name: "certificates"
      volumes:
      - name: "certificates"
        secret:
          secretName: "curl-certs"
          items:
            - key: "ca.crt"
              path: "registry-root.pem"
```

A small trick in case of this curl container we do here is using `CURL_CA_BUNDLER` env. But normally, running something
like alpine, or the likes, you would install `ca-certificate` package. For `gcr.io/distroless/static:nonroot` this is
not needed at all. The certificate will be loaded automatically after start.

Running this deployment now should result in a passing container:

```
curl-test-6589849f47-9f57h                               1/1     Completed          0                4s
```

This is all fine and good, however, what if you would like to have access to the container using https from localhost?
Maybe you want to prime some test data, push some data into the registry and you don't want to use `insecure`. There
are a number of reasons to not use insecure. And we don't want an if case in our code to detect if we are running in a
test environment and inject stuff into our product code either.

Don't worry. [mkcert](https://github.com/FiloSottile/mkcert) got you covered.

### Trusting the certificate locally using mkcert

What we get from priming this cluster is that we have access to the certificate that we bootstrapped. We can download
and decode that rootCA and use `mkcert` to install it locally. We'll put in some guards to make sure mkcert is installed
and then run it in the prime script to install the download rootCA into the local trust stores.

Altered Makefile:

```Makefile
## Location to install dependencies to
LOCALBIN ?= $(shell pwd)/bin
$(LOCALBIN):
	mkdir -p $(LOCALBIN)

MKCERT ?= $(LOCALBIN)/mkcert
MKCERT_VERSION ?= v1.4.4

## Make sure mkcert exists
.PHONY: mkcert
mkcert: $(MKCERT)
$(MKCERT): $(LOCALBIN)
	curl -L "https://github.com/FiloSottile/mkcert/releases/download/$(MKCERT_VERSION)/mkcert-$(MKCERT_VERSION)-$(UNAME)-amd64" -o $(LOCALBIN)/mkcert
	chmod +x $(LOCALBIN)/mkcert

.PHONY: e2e
e2e: prime-test-cluster ## Runs e2e tests
	go test -count=1 -tags=e2e ./e2e

.PHONY: prime-test-cluster
prime-test-cluster: mkcert ## mkcert added as a dependency
	./prime_test_cluster.sh
```

Then we add a tiny bit at the end of our primer:

```bash
#!/usr/bin/env bash

# cleanup
rm -fr rootCA.pem

# install cert-manager with a specific version.
CERT_MANAGER_VERSION=${CERT_MANAGER_VERSION:-v1.13.1}

# if the manifest hasn't been downloaded yet, download it.
if [ ! -e 'cert-manager.yaml' ]; then
  echo "fetching cert-manager manifest for version ${CERT_MANAGER_VERSION}"
  curl -L https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml -o cert-manager.yaml
fi

# create a test cluster -- use any configuration here for kind
kind create cluster --name=e2e-test-cluster

echo -n 'installing cert-manager'
# apply the downloaded manifest then wait for all the deployments to become ready.
kubectl apply -f cert-manager.yaml
kubectl wait --for=condition=Available=True Deployment/cert-manager -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-webhook -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-cainjector -n cert-manager --timeout=60s
echo 'done'

# apply the cluster_issuer combination that was defined above this.
echo -n 'applying root certificate issuer'
kubectl apply -f cluster_issuer.yaml
echo 'done'

# wait for the certificate to be generated.
echo -n 'waiting for root certificate to be generated...'
kubectl wait --for=condition=Ready=true Certificate/mpas-bootstrap-certificate -n cert-manager --timeout=60s
echo 'done'

#### New segment from here ####

# download the certificate and decode it using base64.
kubectl get secret ocm-registry-tls-certs -n cert-manager -o jsonpath="{.data['tls\.crt']}" | base64 -d > rootCA.pem
echo -n 'installing root certificate into local trust store...'
CAROOT=. ./bin/mkcert -install
rootCAPath="./rootCA.pem"

# if the local environment as a ca-certificates store, append our certificate to it. This will come in handy later
# when using a github action to set this all up.
if [ -e '/etc/ssl/certs/ca-certificates.crt' ]; then
  echo "updating root certificate"
  sudo cat "${rootCAPath}" | sudo tee -a /etc/ssl/certs/ca-certificates.crt || echo "failed to append to ca-certificates. Ignoring the failure"
fi

echo 'done'
```

And this is it. Running this script will prompt the user for a root password so mkcert can install the downloaded rootCA.
The name is important here `rootCA.pem`. This is the name mkcert expects to find when running `-install`.

## Github Action

All that remains now is to put this all together so it can run in a github action:

```yaml
name: e2e

on:
  pull_request:
    paths-ignore:
      - 'CODE_OF_CONDUCT.md'
      - 'README.md'
      - 'Contributing.md'
    branches:
      - main

permissions:
  contents: read # for actions/checkout to fetch code

jobs:
  run-e2e-suite:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version-file: '${{ github.workspace }}/go.mod'
      - name: Restore Go cache
        uses: actions/cache@v3
        with:
          path: /home/runner/work/_temp/_github_home/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-
      - name: Setup Kubernetes
        uses: helm/kind-action@v1.5.0
        with:
          install_only: true
      - name: Run E2E tests
        id: e2e-tests
        run: make e2e-verbose
```

That's it! Because of our little hack at the end of our script in prime, it will update the ca-certificate with the root
certificate on an Ubuntu environment. If you are running a different environment, you might have to put this root ca
elsewhere. That depends on your configuration.

## Conclusion

We've seen how to use cert-manager to easily generate a root certificate in cluster. This will alieviate the point of
having to:

- locally generate them
- create secret(s) for them
- rotate certificates when they expire
- deal with switching code between development and prod environments
- using insecure

We are now using cert-manager which is an industry standard for generating certificates. This setup will also make it a
lot easier to switch to a prod environment since we already installed cert-manager and we already have a means of
mounting and distributing certificates amongst pods that need them.
