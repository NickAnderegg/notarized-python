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

Notarize the current Python signing key (found under the OpenPGP Public Keys heading on [Python's download page](https://www.python.org/downloads/)):

```bash
cas notarize -n python-signing-key --hash A035C8C19219BA821ECEA86B64E628F8D684696D
```
