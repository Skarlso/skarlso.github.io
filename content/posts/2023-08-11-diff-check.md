+++
author = "hannibal"
categories = ["ci"]
date = "2023-08-11T01:01:00+01:00"
title = "Diff check and Manifest generation in GitHub Actions"
url = "/2023/08/11/diff-check"
comments = true
+++

# Diff check and manifest generation GitHub Actions

For Go projects it's crucial that you don't forget to run `go mod tidy` from time to time. Combine that with a project
that includes Kubernetes controllers and the other thing people tend to forget is running `make manifest && make generate`.

To check for these I added a small GitHub action that looks like this:

```yaml
name: Check for diff after manifest and generated targets

on:
  pull_request: {}

jobs:
  diff-check-manifests:
    name: Check for diff
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      with:
        fetch-depth: 0
    - name: Make manifests && generate
      run: |
        make manifests && make generate
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
    - name: go mod tidy
      run: |
        go mod tidy
    - name: Check for diff
      run: |
        git diff --exit-code --shortstat
```

Small but effective and saved my behind a couple times now.

That is all.

