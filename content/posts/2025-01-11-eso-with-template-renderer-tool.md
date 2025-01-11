+++
author = "hannibal"
categories = ["external-secrets-operator", "golang"]
date = "2025-01-11T01:01:00+01:00"
title = "External Secrets Operator template rendering tool"
url = "/2025/01/11/eso-with-template-renderer-tool"
comments = true
+++

# External Secrets Operator template rendering tool

Once this[^1] pull request is merged, ESO will be extended with a tool that has the ability to render
templates in an object.

Let's step back a little... what are templates even? But... what is ESO even?

Okay, so... ESO is _external-secrets-operator_[^2]. It's Kubernetes operator than can sync secrets between an external
provider, like AWS Parameter/Secret Store, and a cluster. This is bi-directional. Meaning ESO can sync the secret back
as well using something called a `PushSecret`[^3]. The secret will then be pushed to the provider. This way, secrets can
be backed up or straight Generated[^4] to provide ephemeral access to certain resources.

Now, there are two main parts to ESO. `ExternalSecret`[^5] and `PushSecret`. ES is used to sync a secret from a provider
to the cluster. Both objects support templating. This templating is similar to helm templating and it's using plain
Go templating[^6] as executing engine. It also provides certain functions[^7] that you can use in the templates.

This has notoriously been difficult to explain to users. And testing has been even harder because in order to test
a template you have to:
- create a cluster
- install ESO
- setup a provider
- create a store ( with potentially set up auth )
- create a secret
- create a PS|ES
- create a source secret
- wait for it to reconcile
- try to figure out what went wrong if the secret doesn't even appear

And I already lost you half-way through this list. But now, in order to make this whole process a bit more user-friendly
we are going to provide a command that works like this:

Given this `PushSecret`:
```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: example-push-secret-with-template
spec:
  refreshInterval: 10s
  secretStoreRefs:
    - name: secret-store-name
      kind: SecretStore
  selector:
    secret:
      name: git-sync-secret
  template:
    engineVersion: v2
    data:
      token: "{{ .token | toString | upper }} was templated"
  data:
    - match:
        secretKey: token
        remoteRef:
          remoteKey: git-sync-secret-copy-templated
          property: token
```

and this secret data source:
```yaml
token: dG9rZW4=
```

running the `render-template` command produces the following output:
```
➜ ./bin/external-secrets render-template --source-templated-object template-test/push-secret.yaml --source-secret-data-file template-test/secret.yaml
data:
  token: VE9LRU4gd2FzIHRlbXBsYXRlZA==
metadata:
  creationTimestamp: null

➜ echo -n "VE9LRU4gd2FzIHRlbXBsYXRlZA==" | base64 -d
TOKEN was templated⏎
```

No need for a cluster. No need for a provider or a store. No need for ESO. It is all offline.

It's plain and simple to use and will provide additional debugging options to any user dealing with templating issues.
Now, you can see directly what the result of a template is given specific data from a secret.

[^1]: https://github.com/external-secrets/external-secrets/pull/4277
[^2]: https://external-secrets.io/latest/
[^3]: https://external-secrets.io/latest/api/pushsecret/
[^4]: https://external-secrets.io/latest/guides/generator/
[^5]: https://external-secrets.io/latest/api/externalsecret/
[^6]: https://external-secrets.io/latest/guides/templating/
[^7]: https://external-secrets.io/latest/guides/templating/#helper-functions
