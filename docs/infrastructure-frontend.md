---
title: Build Infrastructure Frontend
layout: main
categories: infrastructure
---

## Frontend setup

The ALICE build infrastructure is exposed via SSO.

Due to limitations of the SSO protocol this happens via single machine which
runs apache and does the reverse proxying to the actual service.

The machine is setup in CERN/IT puppet + OpenStack facility in the hostgroup
`alibuild/frontend`.

## Disaster recovering

### Starting the frontend

The quick recipe to restart the frontend is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "ALICE Release Testing")

- To spawn a machine you need to use the `ai-bs-vm` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<alibuild-frontendXX>

      ai-bs-vm -g alibuild/frontend                  \
               --cc7                                 \
               --nova-sshkey alibuild                \
               --nova-flavor m2.large                \
               --landb-mainuser alice-agile-admin    \
               --landb-responsible alice-agile-admin \
               $MACHINE_NAME
- Once you have the frontend created, you also need to grant it read-only
  permission to some CERN S3 bucket we use for storing logs. The policy for
  it can be found in the [ali-marathon](https://gitlab.cern.ch/AliceDevOps/ali-marathon)
  repository. The required policies are:
  
  * `s3/alice-build-logs-policy.json`
  
  and they need to have the right Ip Address registered there.

### Enabling / disabling one host in the load balancing

Machines in the `alibuild/frontend` hostgroup participate in a load balanced DNS alias. In order to do so they must be in roger state `production`. To do so:

```
roger update --app_alarmed false --appstate production --message 'Fully operational' <hostname>
```

to do an intervention on them:

```
roger update --app_alarmed true --appstate intervention --message '<Some log message>' <hostname>
puppet agent -t -v
```

You can check their load balanced score with:

```
/usr/local/sbin/lbclient -d TRACE
```

## CERN Single Sign-On (SSO) authentication

Some web applications use Apache's OIDC support to authenticate with CERN SSO. Apache then sets [various `OIDC_CLAIM_*` headers][headers] on the forwarded requests.

See also [the CERN SSO documentation][cern-sso].

[headers]: https://auth.docs.cern.ch/user-documentation/oidc/config/
[cern-sso]: https://auth.docs.cern.ch/applications/application-configuration/

### Adding a new application

Applications must be configured on the CERN SSO side through the [Application Portal][app-portal] and on the ALICE side though our Puppet-generated Apache configuration, specifically the file `it-puppet-hostgroup-alibuild/data/hostgroup/alibuild/frontend.yaml`.

1. [Add the application on the Application Portal][add-app]
   - Configure the client ID of the application in `it-puppet-hostgroup-alibuild/data/hostgroup/alibuild/frontend.yaml`, using `oidc_client_id: <client ID>`
   - Store the generated client secret in Teigi as `<app name>-oidc-client-secret`
   - Generate an OIDC crypto passphrase, e.g. using `tr -dc '[:alnum:].+_-' < /dev/urandom | head -c 24; echo` and save it in Teigi as `<app name>-oidc-crypto-passphrase`
2. [Add an OIDC (not SAML) registration to the new application][sso-registration]
   - Make sure you use `https://<app name>.cern.ch/oidc-redirect` as the redirect URI. Apache intercepts the `/oidc-redirect` path.
3. Optionally, configure any required roles, e.g. `alice-member`:
   - [Add this role as documented here][roles] and map any required e-groups (e.g. `alice-member`) to it
   - Configure `oidc_require_role: <role identifier>` for the application in `it-puppet-hostgroup-alibuild/data/hostgroup/alibuild/frontend.yaml`

[app-portal]: https://application-portal.web.cern.ch/
[add-app]: https://auth.docs.cern.ch/applications/adding-application/
[sso-registration]: https://auth.docs.cern.ch/applications/sso-registration/
[roles]: https://auth.docs.cern.ch/applications/role-based-permissions/
