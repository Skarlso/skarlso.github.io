+++
author = "hannibal"
categories = ["crd", "kubernetes", "golang"]
date = "2024-08-21T01:01:00+01:00"
title = "Update your CRDs with confidence"
url = "/2024/08/21/update-your-crds-with-confidence"
comments = true
+++

# Update your CRDs with confidence

Hello.

I would like to write about a release for crd-to-sample-yaml[^1]. It's the release version v0.8.0[^2].

This version brings with it a feature to test the validity of your CRD changes.

It means that if you change your CRD it will test if the changes do not break working samples of that version.

This is achieved by a helm unittest type of YAML based test scenarios and snapshot generation.

It looks something like this:

```yaml
suite: test crd bootstrap
template: crd-bootstrap/crds/bootstrap_crd.yaml # should point to a CRD that is regularly updated like in a helm chart.
tests:
  - it: matches bootstrap crds correctly
    asserts:
      - matchSnapshot:
          # this will generate one snapshot / CRD version and match all of them to the right version of the CRD
          path: sample-tests/__snapshots__
      - matchSnapshot:
          path: sample-tests/__snapshots__
          # generates a yaml file
          minimal: true
  - it: matches some custom stuff
    asserts:
      - matchString:
          apiVersion: v1alpha1 # this will match this exact version only from the list of versions in the CRD
          kind: Bootstrap
          spec:
            source:
              url:
                url: https://github.com/Skarlso/test
```

It can be run by pointing cty at it with a command like `cty test tests/crds`.

More about this can be read on the project's readme here[^3].

Enjoy, and I hope it's helpful.

Thanks!

[^1]: https://github.com/Skarlso/crd-to-sample-yaml/
[^2]: https://github.com/Skarlso/crd-to-sample-yaml/releases/tag/v0.8.0
[^3]: https://github.com/Skarlso/crd-to-sample-yaml/blob/858e82c2284f9237432c7bc2f312883d414b66fe/crd-testing-README.md