+++
author = "hannibal"
categories = ["tilt", "kubernetes", "operators"]
date = "2025-08-02T01:01:00+01:00"
title = "Tilt.dev with server side apply"
url = "/2025/08/02/tilt-with-server-side-apply"
comments = true
+++

# Tilt.dev with server-side apply

Hey folks. This will be a quick one. Recently, [ESO](https://external-secrets.io/latest/) reached the CRD limit
of 256KB which means that every time there is an apply of the CRDs it needs to be a [Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/). This isn't something new.

However, [Tilt](https://tilt.dev/) doesn't really have a clear documentation in this case. And since ESO is using Tilt, or at least
I'm using Tilt with ESO, I also had to fix tilt. A little bit of back-and-forth with the documentation and I decided to use
[k8s_custom_deploy](https://docs.tilt.dev/api.html#api.k8s_custom_deploy).

However (2), I'm also using some manipulations before applying the YAML file so it's basically all in memory at
that point. I can't use stdin for the apply command either because the CRD is too large. And so, the only viable option
really is to write it out into a temp file and apply it from there. Thus, the fix looks like this:

```python
objects = decode_yaml_stream(read_file('bin/deploy/manifests/external-secrets.yaml'))
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') in ['external-secrets-cert-controller', 'external-secrets', 'external-secrets-webhook']:
        o['spec']['template']['spec']['containers'][0]['securityContext'] = {'runAsNonRoot': False, 'readOnlyRootFilesystem': False}
        o['spec']['template']['spec']['containers'][0]['imagePullPolicy'] = 'Always'
        if settings.get('debug').get('enabled') and o.get('metadata').get('name') == 'external-secrets':
            o['spec']['template']['spec']['containers'][0]['ports'] = [{'containerPort': 30000}]


updated_install = encode_yaml_stream(objects)

# Apply the updated yaml to the cluster.
# Create the directory and write the file
local('mkdir -p .tilt-tmp')
local('cat > .tilt-tmp/external-secrets-modified.yaml', stdin=updated_install)

# Now use k8s_custom_deploy to apply it
k8s_custom_deploy(
    'external-secrets',
    apply_cmd='kubectl apply --server-side -f .tilt-tmp/external-secrets-modified.yaml -o yaml',
    delete_cmd='kubectl delete --ignore-not-found -f .tilt-tmp/external-secrets-modified.yaml',
    deps=['bin/deploy/manifests/external-secrets.yaml'],
    image_deps=['oci.external-secrets.io/external-secrets/external-secrets']
)
```

Previously was:

```python
objects = decode_yaml_stream(read_file('bin/deploy/manifests/external-secrets.yaml'))
for o in objects:
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') in ['external-secrets-cert-controller', 'external-secrets', 'external-secrets-webhook']:
        o['spec']['template']['spec']['containers'][0]['securityContext'] = {'runAsNonRoot': False, 'readOnlyRootFilesystem': False}
        o['spec']['template']['spec']['containers'][0]['imagePullPolicy'] = 'Always'
        if settings.get('debug').get('enabled') and o.get('metadata').get('name') == 'external-secrets':
            o['spec']['template']['spec']['containers'][0]['ports'] = [{'containerPort': 30000}]


updated_install = encode_yaml_stream(objects)

# Apply the updated yaml to the cluster.
k8s_yaml(updated_install, allow_duplicates = True)
```

And that's it.

I hope this helps someone if the SEO gods will it.

Thanks for reading,
Gergely.