---
title: Setup an AliEn VOBOX
layout: page
categories: infrastructure
---

# Register the VOBOX

An AliEn VOBOX has first to be registered to the AliEn LDAP. AliEn
administrators can do that with two pieces of information:

* the desired site name (_e.g._ `CERN_MYSITE`), the fully qualified host name
* of the VOBOX (_e.g._ `mysitevobox.cern.ch`)

A site certificate and an associated private key will be created.


# Store credentials in Vault

First create a policy:

    echo 'path "secret/alienvoboxes/mysitevobox/*" { policy = "read" }' | ./vault policy-write mysitevobox -

This policy allows reading the content of all secrets under
`secret/alienvoboxes/mysitevobox`. Now we create a token valid one week:

    vault token-create -policy mysitevobox -ttl 168h

The token can be renewed using the `vault token-renew` command.

When obtaining the certificate and key in PKCS12 format, export it in two PEM
files. Certificate:

    openssl pkcs12 -clcerts -nokeys -in ~/mysitevobox.p12 -out usercert.pem

Password-protected key:

    openssl pkcs12 -nocerts -in ~/mysitevobox.p12 -out userkey_enc.pem

Now unprotect the key:

    openssl rsa -in userkey_enc.pem -out userkey.pem

Save the certificate in Vault. We will use the following keys:

* `secret/mysitevobox/host_cert`
* `secret/mysitevobox/host_privkey`

We can use the following command (and then paste the secret to stdin):

    vault write secret/mysitevobox/host_cert value="`cat`"

Alternatively we can read it from a file:

    vault write secret/mysitevobox/host_cert value=@usercert.pem


# Run the Ansible configuration

Our configuration is stored on Ansible. To run it, by limiting the run only to
the AliEn VOBOXes, do - from the private configuration folder:

    ansible-playbook site.yml -i inventory/ -e vault_token=<valid_vault_token> -l alienvoboxes

A valid Vault token must be provided: secrets are stored there and not in the
configuration repository.
