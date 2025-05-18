+++
author = "hannibal"
categories = ["kubernetes", "go", "crds"]
date = "2025-05-18T01:01:00+01:00"
title = "MFA generator in External Secrets Operator"
url = "/2025/05/18/mfa-generator-in-eso"
comments = true
+++

# MFA Generator in External Secrets Operator

Today, I'm happy to announce that [external-secrets-operator](https://github.com/external-secrets/external-secrets) will have an MFA / TOTP token generator once
[this](https://github.com/external-secrets/external-secrets/pull/4790) PR is merged and released. This opens up some exciting new features.

For example, imagine having an AWS session that requires MFA, or you have an Azure flow that requires a TOTP token. Now, you can achieve that using automation
and an MFA generated in a secret.

Just define a generator:
```yaml
apiVersion: generators.external-secrets.io/v1alpha1
kind: MFA
metadata:
  name: mfa-generator
spec:
  secret:
    key: secret
    name: secret
```

Where the secret is something like this:
```yaml
apiVersion: v1
data:
  secret: NDdFT1dZVTNISFdaVk1HRUNHWjNKQ1NETVpMV0JZUllXTVM2RVNSRDVNVVdNM0NVWVNCM0I0TlVDUUNCTkw0Sg==
kind: Secret
metadata:
  name: secret
type: Opaque
```

And then add an ExternalSecrets object:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mfa-eso
spec:
  refreshInterval: "30s"
  target:
    name: mfa-token
  dataFrom:
  - sourceRef:
      generatorRef:
        apiVersion: generators.external-secrets.io/v1alpha1
        kind: MFA
        name: mfa-generator
```

Here are some images that set up this MFA device on an AWS account:

![mfa-1.png](/img/2025/05/mfa-1.png)
![mfa-1.png](/img/2025/05/mfa-2.png)

To show the secret that needs to be provided to the MFA generator, simply click on Show secret key.
![mfa-show-secret.png](/img/2025/05/mfa-show-secret.png).

Sometimes a provider will require some additional configs. For example, a longer token ( longer than 6 digits ) or a longer time ( default is 30 seconds ) or a different encoding scheme ( default from the RFC is SHA1 ).

These all can be configured with the following fields:

| Key        | Default  | Description                                                                                                    |
|------------|----------|----------------------------------------------------------------------------------------------------------------|
| length     | 6        | Digit length of the generated code. Some providers allow larger tokens.                                        |
| timePeriod | 30       | Number of seconds the code can be valid. This is provider specific, usually it's 30 seconds                    |
| secret     | empty    | This is a secret ref pointing to the seed secret                                                               |
| algorithm  | sha1     | Algorithm for encoding. The RFC defines SHA1, though a provider will set it to SHA256 or SHA512 sometimes      |

Have fun!