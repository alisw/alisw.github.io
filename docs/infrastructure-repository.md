---
title: aliBuild Package Repository
layout: main
categories: infrastructure
---

## Essential operation guides

* [Creating the bucket](#creating-the-bucket)
* [Access via RClone](#accessing-via-rclone)


### Creating the bucket

Creating the bucket should not be needed unless some disaster happens. The current instructions to do so are:

* Go to the [openstack dashboard](https://openstack.cern.ch) select "Object Store > Containers" and add a new container.
* Create a set of ec2 credentials for that container, as explained on the [CERN S3 pages][clouddocs].
* Make sure with CERN/IT that the `ali-bot` S3 local user is still available.
* Make sure no expiration policy is set for the packages.
* Set the access policy to the contents of `ali-marathon/s3/alibuild-repo.json`. See [clouddocs](https://clouddocs.web.cern.ch/object_store/index.html) how to do that.
* Verify that using the `ali-bot` access_key / secret_key you can write files.

### Accessing via RClone

You can use RClone to access the repository with the following configuration:

```bash
export RCLONE_CONFIG_ALIBUILD_TYPE="s3"
export RCLONE_CONFIG_ALIBUILD_PROVIDER="Ceph"
export RCLONE_CONFIG_ALIBUILD_ENV_AUTH="false"
export RCLONE_CONFIG_ALIBUILD_ACCESS_KEY_ID=
export RCLONE_CONFIG_ALIBUILD_SECRET_ACCESS_KEY_ID=
export RCLONE_CONFIG_ALIBUILD_REGION=
export RCLONE_CONFIG_ALIBUILD_BUCKET_ACL="public-read"
export RCLONE_CONFIG_ALIBUILD_ENDPOINT=https://s3.cern.ch
export RCLONE_VERBOSE=4

rclone listremotes
rclone ls alibuild:alibuild-repo/
```

In order to actually write to the repository you need to have the valid access key / secret set. Alternatively you can have the appropriate stanza in `~/.config/rclone/rclone.conf`. 

## Content-addressed tarballs

Architectures built by aliBuild 2.0 into a content-addressed store (a `--remote-store` of the form
`reapi://...`) do not keep the tarball itself under the classic `TARS/<arch>/store/` path.
What lives there is a small stub object, which names the real payload in an HTTP header
(`x-amz-website-redirect-location`) pointing at a content-addressed blob in a separate bucket.
Note that the stub is served with HTTP 200 and not with a redirect status, so a plain `curl -L`
downloads the stub rather than following it; `aliPublish` resolves the stub with a `HEAD` request
before downloading.

The blob itself is an ordinary tarball, so relocation and modulefile generation are unaffected.
This is the reason why the download URL of a tarball may not match the historical
`TARS/<arch>/store/<hash prefix>/<hash>/` layout.
