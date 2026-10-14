---
title: ALICE Docker images
layout: main
categories: infrastructure
---

Most of our [CI Jobs][ci-jobs] running on Nomad are containerized via Docker to
ensure reproducibility between builds.

The docker image definitions are available in
[alisw/docks](https://github.com/alisw/docks), and the CERN docker registry is
<https://registry.cern.ch/>

## Rebuilding a Docker image

If an image definition has changed, it must be rebuilt and pushed to the proper
registry for Nomad to use it in new job allocations.

### Packer-defined images

Newer images have a `packer.json` file, which allows them to be built using
Hashicorp packer

There's a [GitHub Action
here](https://github.com/alisw/docks/actions/workflows/push-docker-image.yml)
that will rebuild and push the images for you.

In case you prefer to do it manually:

```bash
brew install packer # > 0.10.0
cd <image-name>
packer build packer.json
docker push registry.cern.ch/alisw/<image-name>
```

### Dockerfile-defined images

Older images without a `packer.json` can be built with:

```bash
docker build -t alisw/<image-name> <image-name>
```

## Conventions

The CI uses an image named `<arch>-builder` where `<arch>` is the architecture of the
image. `aliBuild` derives the image name from the architecture it is asked to build for, so the
image name must match the format exactly: for instance, building for `slc10_x86-64` pulls
`registry.cern.ch/alisw/slc10-builder`.

The code that derives the image name is
[in aliBuild](https://github.com/alisw/alibuild/blob/master/alibuild_helpers/utilities.py)
(see `default_builder_image`).

[ci-jobs]: https://github.com/alisw/ci-jobs
[packer]: https://www.packer.io/

## Reverting a broken image

Our registry keeps previously built images, in case a new release has an issue you can just re-tag the previous image as `latest`, and go back to a working state

For example, these are the last builds for `slc9-gpu-builder`: [slc9-gpu-builder artifacts-tab](https://registry.cern.ch/harbor/projects/3529/repositories/slc9-gpu-builder/artifacts-tab)
