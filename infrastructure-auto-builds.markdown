---
title: Setting up automated release builds
layout: main
categories: infrastructure
---

This document describes how to automatically start build jobs for a project on
Jenkins when a new release is tagged on GitHub.

## Set up the GitHub workflow

The build is started by the `.github/workflows/release.yml` file in the
repository that should be built. This file requests a VM, installs Kerberos to
authenticate with Jenkins, and then starts real build task there. Here is a
template:

```yaml
name: Release

on:
  release:
    types: [published]

jobs:
  build_release:
    runs-on: ubuntu-18.04
    steps:
    - name: Install Kerberos
      run: |
        sudo DEBIAN_FRONTEND=noninteractive apt-get install -y krb5-user
        cat << \EOF > krb5.conf
        ${{ "{{" }}secrets.KRB5CONF}}
        EOF
        grep rdns krb5.conf
        sudo mv -f krb5.conf /etc/krb5.conf

    - name: Trigger release in jenkins
      run: |
        echo ${{ "{{" }}github.event.release.tag_name}} | grep -e "prod-20[0-9][0-9][0-1][0-9]-[0-9][0-9]"
        echo ${{ "{{" }}secrets.JENKINS_BOT_PASS}} | kinit ${{ "{{" }}secrets.PRINCIPAL}}
        curl -X POST -k --negotiate -u : ${{ "{{" }}secrets.API_URL}} -H 'Content-Type: application/x-www-form-urlencoded' -d 'PROJECT_TAG=${{ "{{" }}github.event.release.tag_name}}'
        klist
        kdestroy
```

### Secrets

GitHub secrets can be referenced in the build configuration using
`${{ "{{" }}secrets.SECRET_NAME}}` syntax. The template script above needs the following
secrets set:

- `KRB5CONF`: the system configuration file for Kerberos. The important thing
  about this is the following setting: `[libdefaults] rdns = false`
- `PRINCIPAL` and `JENKINS_BOT_PASS`: the username and password of the Jenkins
  build user
- `API_URL`: a Jenkins URL that the Kerberos token should be sent to in order to
  start the build

It seems easiest to set a single Kerberos configuration, Jenkins username and
password for every project, but the API URL differs for each project. The first
three secrets are therefore organisation-wide on GitHub, while the last is
repository-specific.

## Jenkins job

The automatic build is triggered on Jenkins by the `curl` command in the
template above. A separate Jenkins configuration must exist for each project to
be built. This should take a single parameter (`PROJECT_TAG` in the template
above) specifying the Git tag name or commit hash to build from. The
project-specific Jenkins configuration then invokes the general build task,
specifying project-specific values.
