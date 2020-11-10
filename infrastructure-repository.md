---
title: aliBuild Package Repository
layout: main
categories: infrastructure
---

# Essential operation guides

* [Creating the bucket](#create-bucket)
* [Access via RClone](#access-rclone)


## Creating the bucket
{: #create-bucket}

Creating the bucket should not be needed unless some disaster happens. The current instructions to do so are:

* Go to the [openstack dashboard](https://openstack.cern.ch) select "Object Store > Containers" and add a new container.
* Create a set of ec2 credentials for that container, as explained on the [CERN S3 pages][clouddocs].
* Make sure with CERN/IT that the `ali-bot` S3 local user is still available.
* Make sure no expiration policy is set for the packages.
* Set the access policy to the contents of `ali-marathon/s3/alibuild-repo.json`. See [clouddocs](https://clouddocs.web.cern.ch/object_store/README.html) how to do that.
* Verify that using the `ali-bot` access_key / secret_key you can write files.

## Accessing via RClone
{: #access-rclone }

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
