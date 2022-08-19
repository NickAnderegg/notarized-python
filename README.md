# CodeNotary Work Sample

## Prompt

Describe how to use github actions to notarize and create a bom for docker, and for build artifacts.

Resources:
* https://github.com/codenotary/cas-notarize-docker-image-bom-github-action
* https://github.com/codenotary/cas-authenticate-docker-bom-github-action
* https://github.com/codenotary/cas-notarize-asset-github-action

The action should only succeed if the source code has been notarized by your signerID, therefore a combination with the authenticate action is required: https://github.com/codenotary/cas-authenticate-asset-github-action

## Tutorial

* Verify signing key with signingID
* If code isn't notarized:
  * Check with signing key.
  * Notarize code.
* If code is notarized:
  * Authenticate.

Then use source to build Docker image.

---

This tutorial will demonstrate how to create a trusted Docker image for Python using GitHub Actions with CodeNotary's Community Attestastion Service.

Let's begin by setting up our GitHub Workflow for authenticating and building our Python image. We define some boilerplate content, as well as some environment variables:

```yaml
name: Build Python image

on:
  pull_request:
  push:
    branches:
      - main

env:
  PYTHON_GPG_KEY: "A035C8C19219BA821ECEA86B64E628F8D684696D"
  PYTHON_VERSION: "3.10.6"
  REGISTRY: ghcr.io
  IMAGE_NAME: YOUR_USERNAME/notarized-python
```

In the snippet above, we define the GPG key used to sign Python source code releases as the `PYTHON_GPG_KEY` environment variable. To be able to authenticate this key for future Python releases, locally notarize the current Python signing key fingerprint using the `cas` CLI (found under the OpenPGP Public Keys heading on [Python's download page](https://www.python.org/downloads/)):

```bash
cas notarize -n python-signing-key --hash A035C8C19219BA821ECEA86B64E628F8D684696D
```

Our first job, `authenticate-signing-key`, will define steps to authenticate Python's signing key fingerprint before doing anything else:

```yaml
jobs:
  authenticate-signing-key:
    name: Authenticate current Python signing key
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate key with CAS
        uses: codenotary/cas-authenticate-asset-github-action@main
        with:
          asset: --hash=${{ env.PYTHON_GPG_KEY }}
          signerID: YOUR_SIGNERID
```

The snippet above uses [CodeNotary's `cas-authenticate-asset-github-action`](https://github.com/codenotary/cas-authenticate-asset-github-action) to authenticate the signing key fingerprint defined in the `PYTHON_GPG_KEY` envrionment variable. This ensures that if there is a change in Python release signing keys, this workflow will not be able to continue unless the hash (fingerprint) of the new signing key is explicitly trusted!

The second job we need to define is a bit more complex, so let's break it down, step-by-step.

### Step 1: Download Source

**Download the compressed tarball** of the Python source code (and its signature file).

```yaml
  authenticate-source-code:
    name: Authenticate Python source code
    needs:
      - authenticate-signing-key
    runs-on: ubuntu-latest
    steps:
      - name: Get Python source code and verify with GPG
        run: |
          set -eux;
          curl -sL -o python-${PYTHON_VERSION}.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz";
          curl -sL -o python-${PYTHON_VERSION}.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc";
```

### Step 2: Attempt Authentication

**Attempt to authenticate source archive with CAS.**
  * If authentication is successful (that is, if this file has already been notarized with the specified signerID), skip to Step 5.
  * If authentication fails (that is, the file has not been notarized), the job moves on to the next step.

```yaml
      - name: Authenticate source with CAS
        id: auth-source
        continue-on-error: true
        uses: codenotary/cas-authenticate-asset-github-action@main
        with:
          asset: python-${{ env.PYTHON_VERSION }}.tar.xz
          signerID: nick@anderegg.io
```

### Step 3: Verify Source Signature

**Verify source code signature with release signing key.** This ensures that the code we are downloading hasn't been tampered with after it was signed.
  * If signature verification is sucessful, the job moves on to the next step.
  * If signature verification fails, the job fails, because the signature doesn't match the downloaded source archive.

```yaml
      - name: Verify source with GPG key
        id: verify-source
        if: steps.auth-source.outcome == 'failure'
        run: |
          set -eux;

          export GNUPGHOME="$(mktemp -d)"

          gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$PYTHON_GPG_KEY";
          gpg --batch --verify python-${PYTHON_VERSION}.tar.xz.asc python-${PYTHON_VERSION}.tar.xz;
          command -v gpgconf > /dev/null && gpgconf --kill all || :;
          rm -rf "$GNUPGHOME" python-${PYTHON_VERSION}.tar.xz.asc;
```

### Step 4: Notarize Source Code

**Notarize Python source code archive** to generate a record for authentication in the future. Because we've verified the signature of the source archive with the signing key we've manually trusted, we can notarize this version of the Python source code for future authentication.

```yaml
      - name: Notarize Python source code
        if: steps.verify-source.outcome == 'success'
        uses: codenotary/cas-notarize-asset-github-action@main
        with:
          asset: python-${{ env.PYTHON_VERSION }}.tar.xz
          cas_api_key: ${{ secrets.CAS_API_KEY }}
```

### Step 5: Save Source Code Artifact

**Upload source code as workflow artifact.** We want to pass this source code archive to the next job for use in building our Docker image.

```yaml
      - name: Upload Python source for next job
        uses: actions/upload-artifact@v3
        with:
          name: python-${{ env.PYTHON_VERSION }}-src
          path: python-${{ env.PYTHON_VERSION }}.tar.xz
```
