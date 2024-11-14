+++
author = "hannibal"
categories = ["helm", "cert-manager"]
date = "2024-07-02T01:01:00+01:00"
title = "Using cert-manager as a subchart for your own Helm Chart"
url = "/2024/07/02/using-cert-manager-as-a-subchart-with-helm"
comments = true
+++

# Using cert-manager as a subchart

Hello.

In today's post, I would like to show how to set up cert-manager as a subchart. Not only that,
but also, how to add a `Job` to make sure that cert-manager is installed and its webhook is
up and running so it can process any `Issuers` that are going to be applied with your own chart.

Let's get to it.

## Defining a subchart

To define a subchart, simply add it as a dependency in your `Charts.yaml` like this:

```yaml

dependencies:
  - name: cert-manager
    version: v1.14.5
    repository: https://charts.jetstack.io
    condition: cert-manager.enabled
```

Easy. Now, notice the condition. This value indicates that it needs to be set in order to install cert-manager.
That happens by defining this in your values file:

```yaml
cert-manager:
  enabled: false
  namespace: cert-manager
  installCRDs: true
  fullnameOverride: "cert-manager" # this is needed for the certificate issuer to not throw an unknown authority error
  nameOverride: "cert-manager" # needed because otherwise it will call it `certManager`
```

Once these values are defined, now comes the hard part.

## Post-install, Post-upgrade

With Helm chart what you want is to make sure all things are up and running before continuing on. This is sometimes
not easy to determine for helm especially if the deployed resource gives a thumbs-up even before everything is actually
running. That can happen if Readiness isn't define properly.

With cert-manager it happens that cert-manager is up and running, yet when it tries to apply the next thing in your chart
which is an Issuer, it will error that cert-manager webhook could not be reached.

To work around that problem, helm introduced chart hooks[^1]. They allow you to control when then next thing should run.

This is somewhat helpful, but we need even more control. We need to be able to make sure cert-manager is up and running.
We can do that by running the following `wait` commands with `kubectl`.

```
kubectl wait --for=condition=Available=True Deployment/cert-manager -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-webhook -n cert-manager --timeout=60s
kubectl wait --for=condition=Available=True Deployment/cert-manager-cainjector -n cert-manager --timeout=60s
```

But we need to be able to do that DURING `helm install`. That's where the fun begins.

## Jobs

To run a command during install we can utilize a `Job`. How? By creating a `Job` deployment and putting it
into the templates so it's applied. Like this:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: wait-for-cert-manager
  namespace: cert-manager
  labels:
    app: {{ .Release.Name }}-wait-for-cert-manager
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      serviceAccountName: wait-for-cert-manager-sa
      containers:
        - name: wait-for-cert-manager
          image: bitnami/kubectl:latest
          command:
            - /bin/sh
            - -c
            - |
              kubectl wait --for=condition=Available=True Deployment/cert-manager -n cert-manager --timeout=60s
              kubectl wait --for=condition=Available=True Deployment/cert-manager-webhook -n cert-manager --timeout=60s
              kubectl wait --for=condition=Available=True Deployment/cert-manager-cainjector -n cert-manager --timeout=60s
      restartPolicy: OnFailure
```

However, we only want to do this if cert-manager is enabled so we put a little templating in there:

```yaml
{{- if index .Values "cert-manager" "enabled" }}
apiVersion: batch/v1
kind: Job
metadata:
  name: wait-for-cert-manager
  namespace: cert-manager
  labels:
    app: {{ .Release.Name }}-wait-for-cert-manager
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      serviceAccountName: wait-for-cert-manager-sa
      containers:
        - name: wait-for-cert-manager
          image: bitnami/kubectl:latest
          command:
            - /bin/sh
            - -c
            - |
              kubectl wait --for=condition=Available=True Deployment/cert-manager -n cert-manager --timeout=60s
              kubectl wait --for=condition=Available=True Deployment/cert-manager-webhook -n cert-manager --timeout=60s
              kubectl wait --for=condition=Available=True Deployment/cert-manager-cainjector -n cert-manager --timeout=60s
      restartPolicy: OnFailure
{{- end}}
```

Now, what else we need are some roles and service account to run this with.

```yaml
{{- if index .Values "cert-manager" "enabled" }}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: wait-for-cert-manager-role
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
{{- end}}
```

```yaml
{{- if index .Values "cert-manager" "enabled" }}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: wait-for-cert-manager-rolebinding
subjects:
  - kind: ServiceAccount
    name: wait-for-cert-manager-sa
    namespace: cert-manager
roleRef:
  kind: ClusterRole
  name: wait-for-cert-manager-role
  apiGroup: rbac.authorization.k8s.io
{{- end}}
```

```yaml
{{- if index .Values "cert-manager" "enabled" }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wait-for-cert-manager-sa
  namespace: cert-manager
{{- end}}
```

Put these into the template section too and run `helm install`.

```yaml
{{- if index .Values "cert-manager" "enabled" }}
```

This is using `index` because the value in values.yaml is `cert-manager` with a hyphen.

## Conclusion

Once these values are set, you should be able to run helm install and have your Issuer run correctly AFTER
cert-manager is up and running properly.

A sample implementation of all of this can be found in this[^2] repository.

Thank you for reading!


[^1]: https://helm.sh/docs/topics/charts_hooks/
[^2]: https://github.com/open-component-model/ocm-controller/tree/6042a15dcb62942275aa310ac6c2383edb87ae12/deploy/templates