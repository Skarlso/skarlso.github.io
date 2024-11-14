+++
author = "hannibal"
categories = ["github-actions", "certificates", "go"]
date = "2023-07-04T01:01:00+01:00"
title = "How to add a self-signed certificate to the GitHub action runner"
url = "/2023/07/04/how-to-add-self-signed-certificates-to-github-action-runner-using-mkcert"
comments = true
+++

# Adding a certificate to a GitHub runner

Imagine having a project where you have a server that you would like to run with TLS. Let's say, you want to run a
Docker registry in a cluster using TLS. You need the generated certificate's root certificate in the trust store of the
GitHub action runner.

This is simple with [mkcert](https://github.com/FiloSottile/mkcert).

The action is simple:

```yaml
name: tests

on:
  pull_request:
    paths-ignore:
      - 'CODE_OF_CONDUCT.md'
      - 'README.md'
      - 'Contributing.md'
  workflow_call:

  push:
    branches:
      - main

permissions:
  contents: read # for actions/checkout to fetch code

jobs:
  run-test-suite:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Go
        uses: actions/setup-go@v3
        with:
          go-version-file: '${{ github.workspace }}/go.mod'
      - name: Restore Go cache
        uses: actions/cache@v3
        with:
          path: /home/runner/work/_temp/_github_home/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-
      - name: Run e2e
        run: make e2e

```

This is nothing fancy. The fancy thing is coming from the `make e2e` part.

First, we make sure we have mkcert:

```Makefile
MKCERT ?= $(LOCALBIN)/mkcert
MKCERT_VERSION ?= v1.4.4
...
.PHONY: mkcert
mkcert: $(MKCERT)
$(MKCERT): $(LOCALBIN)
	curl -L "https://github.com/FiloSottile/mkcert/releases/download/$(MKCERT_VERSION)/mkcert-$(MKCERT_VERSION)-$(UNAME)-amd64" -o $(LOCALBIN)/mkcert
	chmod +x $(LOCALBIN)/mkcert
```

Then, let's add a target which we'll call before the test is called:

```Makefile
.PHONY: generate-developer-certs
generate-developer-certs: mkcert
	./hack/create_developer_certificate_secrets.sh
```

Lastly, put this as a dependency on our test target:

```Makefile
.PHONY: e2e
e2e: generate-developer-certs test-summary-tool ## Runs e2e tests
	$(GOTESTSUM) --format testname -- -count=1 -tags=e2e ./e2e
```

That's all the setup we need in the GitHub action workflow part. Now, let's take a look at the magic:

```bash
#!/usr/bin/env bash

# You can ignore this.
path=$(pwd)

if [[ "${path}" == *hack* ]]; then
  echo "This script is intended to be executed from the project root."

  exit 1
fi

if [ "$(kubectl get secret -n ocm-system registry-certs)" ]; then
  echo "secret already exist, no need to re-run"

  exit 0
fi

echo "generating developer certificates and kubernetes secrets"

# Set up certificate paths
CAROOT=./hack/certs ./bin/mkcert -install
certPath="./hack/certs/cert.pem"
keyPath="./hack/certs/key.pem"
rootCAPath="./hack/certs/rootCA.pem"

echo "updating root certificate"

sudo cat "${rootCAPath}" | sudo tee -a /etc/ssl/certs/ca-certificates.crt || echo "failed to append to ca-certificates. Ignoring the failure"

if [ ! -e "${certPath}" ] && [ ! -e "${keyPath}" ]; then
  echo -n "certificates not found, generating..."

  CAROOT=./hack/certs ./bin/mkcert -cert-file ./hack/certs/cert.pem -key-file ./hack/certs/key.pem registry.ocm-system.svc.cluster.local localhost 127.0.0.1 ::1

  echo "done"
else
  echo "certificates found, will not re-generate"
fi

# You can ignore this.
echo -n "creating secret..."
kubectl create secret generic \
  -n ocm-system registry-certs \
  --from-file=caFile="${rootCAPath}" \
  --from-file=certFile="${certPath}" \
  --from-file=keyFile="${keyPath}" \
  --dry-run=client -o yaml > ./hack/certs/registry_certs_secret.yaml


```

Let's break this down line by line:

1-15: Ignore this part. It's just checking some things before running.
19: This is calling `mkcert` but notice the `CAROOT` setting. This will output the root certificate to that location.
27: Now, this is the important bit. Here, since we are using `Ubuntu` machine type, we append our certificate to the
system's root certificates. There are other ways to do this but none of them will work properly on GitHub actions or
requires way too much fiddling. This is a heck of a lot easier to execute.
29-37: If the certificates do not exist it will create them. This, again, is using the configured ROOT location. This
is done so that when this script is running locally, it doesn't continue re-generating the certificates constantly.
39-45: Here, we create the secret which will contain our certificates.

So that's the script.

## How to automatically create developer certificates

To do this automatically we use a combination of Makefile targets and Tilt. Tilt configures the mount and Makefile target
calls `mkcert` and the script and then runs the e2e tests locally using `Kind`.

```python
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'ocm-controller':
        print('updating ocm-controller deployment to add generated certificates')
        o['spec']['template']['spec']['volumes'] = [{'name': 'root-certificate', 'secret': {'secretName': 'registry-certs', 'items': [{'key': 'caFile', 'path': 'ca.pem'}]}}]
        o['spec']['template']['spec']['containers'][0]['volumeMounts'] = [{'mountPath': '/certs', 'name': 'root-certificate'}]
```

The name can be made configurable with an environment property, but here we just ignore that in the dev environment.

### Adding the root certificate to the container

To add the root certificate to the container in which our controller is running, we use an entrypoint.sh script.
We get the root certificate that is mounted into the container and then append that rootCA to the system CA. When done
we launch our manager.

```bash
#!/usr/bin/env sh

rootCA=${REGISTRY_ROOT_CERTIFICATE:-/certs/ca.pem}

if [ ! -e "${rootCA}" ]; then
  echo "warning... root certificate at location ${rootCA} not found."

  exec "$@"
fi

echo "updating root certificate with provided certificate..."
tee -a /etc/ssl/certs/ca-certificates.crt < "${rootCA}"

echo "done."

exec "$@"
```

That's all we need. With this, our controller will now understand our self-signed certificate or any certificate that is
provided to the controller if it exists.

## Conclusion

This is it. We have all the things we need for a self-signer certificate to seamlessly work in dev and in CI environment.
The whole project with all the files can be found here: [ocm-controller](https://github.com/open-component-model/ocm-controller).

Thank you for reading!

Gergely.