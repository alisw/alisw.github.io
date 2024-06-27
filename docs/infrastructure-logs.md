---
title: Build logs
layout: main
categories: infrastructure
---

Logs for the PRs are now copied on the [CERN S3](https://clouddocs.web.cern.ch/object_store/index.html) object store.
They are accessible either via the read only web interface at:

<https://ali-ci.cern.ch/alice-build-logs/>

which is an SSO protected url exposed by machines in the `alibuild/frontend` puppet hostgroup or via the
`s3://alice-build-logs` endpoint, which provides read-write capabilities.

For the SSO access you need to be an alice member, while for the S3 endpoint, you either need to be in the `alice-vm-admin`
egroup.

# Essential operation guides

* [Creating the bucket](#creating-the-bucket)
* [Updating the policy](#updating-the-policy)
* [Accessing the logs programmatically](#accessing-the-logs-programmatically)

## Creating the bucket

Creating the bucket should not be needed unless some disaster happens. The current instructions to do so are:

* Go to the [openstack dashboard](https://openstack.cern.ch) select "Object Store > Containers" and add a new container.
* Create a set of ec2 credentials for that container, as explained on the [CERN S3 pages][clouddocs].
* Make sure with CERN/IT that the `ali-bot` S3 local user is still available.
* Set the expiration policy on the bucket to 60 days (see instructions on the [CERN S3 pages][clouddocs]) .
* Set the access policy to the contents of `ali-marathon/s3/alice-build-logs-policy.json`.
* Verify that using the `ali-bot` access_key / secret_key you can write files.

## Updating the policy

In case you need to update the S3 access permission policy, e.g. in case the frontend IP changes, you need to do so in `ali-marathon/s3/alice-build-logs-policy.json` and then apply it to the `s3://alice-build-logs`

[clouddocs]: https://clouddocs.web.cern.ch/clouddocs/

You can verify that a given machine has access to the logs by doing:

```bash
curl alice-build-logs.s3.cern.ch/test.txt
```

If you get an actual reply, rather than permission denied, it means the machine can access the logs.

## Accessing the logs programmatically

Accessing the logs programmatically can be done via any S3 enabled client, e.g. `s3cmd` (command line) or `boto3` (python). Ask usual suspects for the access key, secret. An example of how new logs can be pushed via `boto3` is at <https://github.com/alisw/ali-bot/blob/master/report-pr-errors#L175-L194>.
