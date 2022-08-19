# Building Docker images using trusted and authenticated source code

## Introduction

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

## Authenticating and Verifying the Source Code

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
          signerID: YOUR_SIGNERID
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

## Building the Docker Image

The majority of the `build-image` job is taken up by boilerplate configuration:

```yaml
  build-image:
    name: Build and notarize Python image for Docker
    needs:
      - authenticate-source-code
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Download Python source code artifact
        uses: actions/download-artifact@v3
        with:
          name: python-${{ env.PYTHON_VERSION }}-src

      - name: Authenticate source with CAS
        uses: codenotary/cas-authenticate-asset-github-action@main
        with:
          asset: python-${{ env.PYTHON_VERSION }}.tar.xz
          signerID: YOUR_SIGNERID

      - name: Setup Docker image cache
        uses: ScribeMD/docker-cache@0.2.2
        with:
          key: docker-${{ runner.os }}-${{ hashFiles(env.PYTHON_VERSION) }}

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v3.1.1
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

In the job definition above, the `needs` key indicates that this job will only run if the `authenticate-source-code` job before it succeeds. This ensures that we won't build our Docker image using code that hasn't been either:

* Authenticated as a trusted asset with the Community Attestastion Service.
* Verified via a signature from a manually-trusted release signing key.

You might also notice in the above snippet that we're using GitHub's `download-artifact` action to retrieve the source code archive we authenticated/verified in the previous job... and then we authenticate again! There is no way for the previous job to succeed without the current source archive having been verified, but because we are downloading the previous job's artifacts from where we stashed it on GitHub's servers, this ensures the source archive hasn't been changed in any way.

When it comes to the "build and push" step for our Docker image, it runs the Dockerfile provided in this repo.

> The actual Dockerfile used in this example is a modified version of the [Dockerfile defined for the official Python container image](https://github.com/docker-library/python/blob/7b9d62e229bda6312b9f91b37ab83e33b4e34542/3.10/bullseye/Dockerfile). In particular, the lines which download and verify the Python source distribution have been removed. This functionality has been replaced by GitHub Actions, and the source code is instead added to the container via the Dockerfile command:
>
> `COPY ./python-${PYTHON_VERSION}.tar.xz .`

After our new image has been built, we can generate a Software Bill of Materials (SBOM) for it, in order to verify integrity of the image in the future:

```yaml
      - name: Notarize Docker image
        uses: codenotary/cas-notarize-docker-image-bom-github-action@main
        with:
          asset: docker://${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
          cas_api_key: ${{ secrets.CAS_API_KEY }}

      - name: Test authentication of notarized Docker image
        uses: codenotary/cas-authenticate-docker-bom-github-action@main
        with:
          asset: docker://${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
          signerID: YOUR_SIGNERID
```

As you can see from the above snippet, we end this job by authenticating our Docker image to ensure the notarization process was successful!
