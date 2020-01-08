---
title: aliBuild Package Repository
layout: main
categories: infrastructure
---

**EXPERIMENTAL**

This describes the setup of the **EXPERIMENTAL** package repository hosted on CERN S3 infrastructure.

# Essential operation guides

* [Creating the bucket](#create-bucket)
* [Updating the policy](#update-policy)

## Creating the bucket
{: #create-bucket}

Creating the bucket should not be needed unless some disaster happens. The current instructions to do so are:

* Go to the [openstack dashboard](https://openstack.cern.ch) select "Object Store > Containers" and add a new container.
* Create a set of ec2 credentials for that container, as explained on the [CERN S3 pages][clouddocs].
* Make sure with CERN/IT that the `ali-bot` S3 local user is still available.
* Make sure no expiration policy is set for the packages.
* Set the access policy to the contents of `ali-marathon/s3/alibuild-repo.json`. See [clouddocs][] how to do that.
* Verify that using the `ali-bot` access_key / secret_key you can write files.
