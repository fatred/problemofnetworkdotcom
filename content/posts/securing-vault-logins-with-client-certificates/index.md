---
title: "Securing Vault logins with client certificates"
date: 2026-10-11T22:30:00+02:00
author: John Howard
tags: ["vault", "pki", "certstrap", "yubikey"]
showFullContent: false
---

In the [last post](/posts/securing-your-vault-instance-with-tls/) we put a certificate from an offline certstrap CA on the Vault listener, so clients can check that Vault really is Vault. This one is the other half, and the last post in the Vault series: using certificates the other way round, to prove who *you* are, with the private key sitting on a YubiKey.

Vault's `cert` auth method lets a client present a TLS client certificate instead of a password or a token, and Vault turns that into a short-lived token with a policy attached. The user-facing half of this is done with [yubivault](https://github.com/fatred/yubivault), a small tool that logs in to Vault with a client certificate and prints a token. This post is the "why and how it fits together" version of the [PKI-SETUP.md](https://github.com/fatred/yubivault/blob/main/PKI-SETUP.md) guide in that repo.

## Why A Separate CA

One could sign user certificates with the Infra CA from the last post. Please don't. Every client machine now trusts the Infra CA, which is exactly what makes it a bad choice for logins: the thing that proves "this is a server" and the thing that proves "this is a person who can read the netops secrets" should not share a key. If they did, a leaked server CA would mint logins. Two CAs, two keys, two jobs.

> Weird comment: contrary to the server auth, where I STRONGLY recommend using LetsEncrypt or your Corpo PKI, this kind of CA based user auth, _ain't_ a terrible use for certstrap, IF (and only really IF), you don't have some existing IAM process that can mint user certs for you already. SPEAK TO YOUR IAM PEOPLE! Assuming you run with a certstrap, its now a security boundary. not only should you have save, encrypted backups of the key material, you need to PROTECT that system that holds the keymaterial. SPEAK TO YOUR SECURITY PEOPLE!

That said, lets crack on with a second certstrap CA, from a fresh directory, and again with a passphrase on the key that I'm not going to show you:

```
jhow@nuc4:~/pon-users-ca$ certstrap init --common-name "Problem Of Network Users CA" --expires "10 years"
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Created out/Problem_Of_Network_Users_CA.key (encrypted by passphrase)
Created out/Problem_Of_Network_Users_CA.crt
Created out/Problem_Of_Network_Users_CA.crl
```

## The User Makes A Key And A CSR

The person logging in creates their own private key, and sends a certificate signing request (CSR) to whoever runs the CA. The issuer never sees the private key. That is the entire point of a CSR.

In real life, the key lives on a YubiKey in PIV slot 9a, and is generated on the device so it can never be exported. These are the two commands from step 2 of PKI-SETUP.md, shown without output because I didn't use them for the demo:

```shell
ykman piv keys generate --algorithm ECCP256 9a alice.pub
ykman piv certificates request --subject "CN=alice,OU=netops" 9a alice.pub alice.csr
```

The first command creates the key on the YubiKey and writes out the public half. The second has the YubiKey sign a CSR with that key (it will ask for the PIN). Then `yubivault -yubi` uses the key on the YubiKey to log in.

For this demo I used a plain file key so that I could log in with `yubivault -local` and not need a YubiKey plugged into a lab machine, which also means this key is not hardware-protected, so don't copy that part:

```
jhow@nuc4:~$ openssl ecparam -name prime256v1 -genkey -noout -out alice.key

jhow@nuc4:~$ openssl req -new -key alice.key -subj "/OU=netops/CN=alice" -out alice.csr
```

The subject is the important bit. `CN=alice` is who the person is, and `OU=netops` is the team they are in. Vault will use the OU to decide which role applies to this certificate, and the CN to tell you who did what in the audit log.

## The Issuer Checks And Signs

This is the step with a human in it, and it's the step that matters most. **The OU in a CSR is a claim, not proof.** Alice typed `netops` into that request herself. Nothing in the CSR stops her from typing `finance`, or `vault-admins`, or anything else. Before you sign, confirm out of band (ask her manager, check the team roster, send a message on a channel you trust) that alice really is in netops. The signature is what makes the claim true.

Bring the CSR into the depot. certstrap refuses to read depot files that are writable, so the copy has to be made read-only; `chmod 600` isn't enough, it has to be `400`. Then look at the subject once more with your own eyes before signing:

```
jhow@nuc4:~/pon-users-ca$ cp ~/alice.csr out/alice.csr

jhow@nuc4:~/pon-users-ca$ chmod 400 out/alice.csr

jhow@nuc4:~/pon-users-ca$ openssl req -in out/alice.csr -noout -subject
subject=OU=netops, CN=alice

jhow@nuc4:~/pon-users-ca$ certstrap sign alice --CA "Problem Of Network Users CA" --expires "1 year"
Enter passphrase for CA key (empty for no passphrase): 
Created out/alice.crt from out/alice.csr signed by out/Problem_Of_Network_Users_CA.key

jhow@nuc4:~/pon-users-ca$ openssl x509 -in out/alice.crt -noout -subject -issuer -dates
subject=OU=netops, CN=alice
issuer=CN=Problem Of Network Users CA
notBefore=Oct 10 19:58:43 2026 GMT
notAfter=Oct 10 20:08:42 2027 GMT
```

That `out/alice.crt` goes back to Alice. It is public, so email is fine. Leave the copy in the depot too, because `certstrap revoke` looks certificates up there later. With a YubiKey, Alice would now import it into slot 9a with `ykman piv certificates import --verify 9a alice.crt`, which checks the certificate matches the key before writing it.

## Configuring Vault

Now the Vault side. This needs an admin token, and fittingly mine comes from my own YubiKey: I log in with `yubivault -yubi` against the `admin` role from PKI-SETUP.md, which hands me a short-lived token with an admin policy. I'm not showing that token, obviously. First, check the cert auth method is there:

```
jhow@nuc4:~/pon-users-ca$ vault auth list
Path         Type        Accessor                  Description                Version
----         ----        --------                  -----------                -------
cert/        cert        auth_cert_b6fcaf66        n/a                        n/a
token/       token       auth_token_daec4935       token based credentials    n/a
userpass/    userpass    auth_userpass_977778b8    n/a                        n/a
```

`cert/` is already enabled on my instance. On a fresh Vault you'd do `vault auth enable -path=cert cert` first, and that's the only extra step.

Next, a policy. I'm treating policies as a team's group policy, so this one is for netops, and it gives them create/read/update/delete on one corner of the `network-automation` KV mount from the PKI post, and nothing else:

```
jhow@nuc4:~/pon-users-ca$ vault policy write netops - <<'EOF'
# netops team: read/write their own corner of the network-automation kv
path "network-automation/data/netops/*" {
  capabilities = ["create", "read", "update", "patch", "delete", "list"]
}
path "network-automation/metadata/netops/*" {
  capabilities = ["read", "list", "delete"]
}
EOF
Success! Uploaded policy: netops
```

Remember that kv v2 has `data/` and `metadata/` in its real API paths, which is why there are two stanzas. The CLI hides that from you, the policy doesn't.

Then the role, which is the piece that ties a certificate to a policy:

```
jhow@nuc4:~/pon-users-ca$ vault write auth/cert/certs/netops \
    display_name=netops \
    certificate=@out/Problem_Of_Network_Users_CA.crt \
    allowed_organizational_units=netops \
    token_policies=netops \
    token_ttl=1h \
    token_max_ttl=8h
Success! Data written to: auth/cert/certs/netops
```

There is a lot packed into that:

* `certificate` is the Users CA certificate. Any certificate that CA signed *could* match this role, subject to the constraints.
* `allowed_organizational_units=netops` is the constraint. Any CN is accepted, but at least one OU on the certificate must match. A certificate with `OU=finance` and `OU=netops` gets in. One with no OU, or only `finance`, does not.
* `token_policies=netops` is what you get when it matches.
* `token_ttl=1h` and `token_max_ttl=8h` mean the token lasts an hour, and can be renewed up to eight hours in total.

The pattern is one role per team, each pointing at that team's policy. Adding a person to netops means issuing them a certificate with `OU=netops`. Nothing changes in Vault. For a new team, write a new policy and a new role with that team's OU. Without the OU constraint, every certificate the CA ever signs gets the role, which is not what you want.

Here is the role read back. The certificate field dumps the whole CA, so I've trimmed it:

```
jhow@nuc4:~/pon-users-ca$ vault read auth/cert/certs/netops
Key                             Value
---                             -----
alias_metadata                  map[]
allowed_common_names            <nil>
allowed_dns_sans                <nil>
allowed_email_sans              <nil>
allowed_metadata_extensions     <nil>
allowed_names                   <nil>
allowed_organizational_units    [netops]
allowed_organizations           <nil>
allowed_uri_sans                <nil>
certificate                     -----BEGIN CERTIFICATE-----
…
-----END CERTIFICATE-----
display_name                    netops
ocsp_ca_certificates            n/a
ocsp_enabled                    false
ocsp_fail_open                  false
ocsp_max_retries                4
ocsp_query_all_servers          false
ocsp_servers_override           <nil>
ocsp_this_update_max_age        0
required_extensions             <nil>
token_bound_cidrs               []
token_explicit_max_ttl          0s
token_max_ttl                   8h
token_no_default_policy         false
token_num_uses                  0
token_period                    0s
token_policies                  [netops]
token_ttl                       1h
token_type                      default
```

Last bit of setup: something for the team to read. With the admin token, I wrote a demo secret into the netops corner:

```
jhow@nuc4:~/pon-users-ca$ vault kv put network-automation/netops/demo owner=netops note="hello from the netops team"
=========== Secret Path ===========
network-automation/data/netops/demo

======= Metadata =======
Key                Value
---                -----
created_time       2026-10-10T20:08:51.307226446Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

## Alice Logs In

Over to Alice's machine. She has `alice.key` and the `alice.crt` that came back from the issuer. yubivault reads a config file from `~/.yubivault`, so she puts the files and the config there:

```
jhow@nuc4:~$ mkdir -p ~/.yubivault && chmod 700 ~/.yubivault

jhow@nuc4:~$ cp alice.crt alice.key ~/.yubivault/ && chmod 600 ~/.yubivault/alice.key

jhow@nuc4:~$ cat ~/.yubivault/config.yml
vaultAddr: https://vault.problemofnetwork.com:8200
certAuthMount: cert
certAuthName: netops
certAuthPemFile: alice.crt
certAuthKeyFile: alice.key
```

`certAuthName` is the role name, `netops`. In the YubiKey version of this config you'd have the OpenSC module path and the PIV serial in there instead of the two file names, and you'd run `yubivault -yubi`. For the demo, `yubivault -local` prints a token on stdout, which I put straight into `VAULT_TOKEN` so the normal `vault` CLI picks it up:

```
jhow@nuc4:~$ export VAULT_TOKEN="$(yubivault -local)"

jhow@nuc4:~$ vault token lookup
Key                 Value
---                 -----
accessor            6dDIDTuNGCb1ikYZH8talxRW
creation_time       1791662942
creation_ttl        1h
display_name        cert-netops
entity_id           505f15f0-fb46-8cf0-23c3-a55874b172c4
expire_time         2026-10-10T21:09:02.729928424Z
explicit_max_ttl    0s
id                  hvs.<redacted>
issue_time          2026-10-10T20:09:02.729937576Z
meta                map[authority_key_id:d0:05:9d:bb:63:e9:8e:30:28:a7:5f:6c:50:bf:2a:88:5d:33:90:de cert_name:netops common_name:alice serial_number:33426732528195777887456428177611582076 subject_key_id:9d:b0:0f:e4:1d:64:1a:1d:ee:3d:05:c7:b7:f2:f3:4f:8d:98:42:9e]
num_uses            0
orphan              true
path                auth/cert/login
policies            [default netops]
renewable           true
ttl                 59m59s
type                service
```

Three things worth pointing at:

* `policies [default netops]`: the token got the `netops` policy from the role, plus `default`.
* `display_name cert-netops`: Vault names the token after the auth method and role.
* `common_name:alice` in `meta`: the CN from the certificate. The role admits anyone with `OU=netops`, but the token (and so the audit log from the [audit](/posts/vault-audit-logging/) post) still says exactly *which* person it was. That's the other half of using one role per team.

The token is also `orphan true`, with a one-hour TTL. Now to see what that policy lets her do. She can read the team secret:

```
jhow@nuc4:~$ vault kv get network-automation/netops/demo
=========== Secret Path ===========
network-automation/data/netops/demo

======= Metadata =======
Key                Value
---                -----
created_time       2026-10-10T20:08:51.307226446Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

==== Data ====
Key      Value
---      -----
note     hello from the netops team
owner    netops
```

But go anywhere else, even somewhere nearby, and she is refused. `device-certs` is a perfectly real secret in the same mount, which the netops policy doesn't cover. Listing policies is an admin thing, so that is denied too:

```
jhow@nuc4:~$ vault kv get network-automation/device-certs
Error reading network-automation/data/device-certs: Error making API request.

URL: GET https://vault.problemofnetwork.com:8200/v1/network-automation/data/device-certs
Code: 403. Errors:

* 1 error occurred:
	* permission denied



jhow@nuc4:~$ vault policy list
Error listing policies: Error making API request.

URL: GET https://vault.problemofnetwork.com:8200/v1/sys/policies/acl?list=true
Code: 403. Errors:

* 1 error occurred:
	* permission denied


```

That is least privilege working: the same certificate that opens the team's secrets does nothing else.

## Eve From Finance Is Refused

A role that lets everyone in isn't a role, it's a door. So let's try to walk through it as someone who shouldn't. Eve works in finance, and the issuer (correctly, after checking) signed her a perfectly valid certificate from the same Users CA, but with `OU=finance`:

```
jhow@nuc4:~$ openssl x509 -in ~/.yubivault/eve.crt -noout -subject
subject=OU=finance, CN=eve

jhow@nuc4:~$ yubivault -local
2026/10/10 22:09:11 400 Bad Request: failed to match all constraints for this login certificate

jhow@nuc4:~$ echo $?
1
```

Her certificate chains to the CA Vault trusts, so the TLS handshake is fine, but it fails the role's `allowed_organizational_units` constraint, so Vault refuses to mint a token. yubivault exits non-zero, so a script using it will stop. Note that nothing about Eve's certificate was "wrong". The CA is the same. The constraint on the role is what separates the teams, which is why the issuer's check before signing matters too: a certificate with the wrong OU on it would have been accepted by the wrong role.

## Renewal Needs The Certificate Again

Tokens from the cert method are renewable, but with a catch. Cert-auth tokens can only be renewed by presenting the client certificate again. A bare `vault token renew`, with just the token and no certificate, fails:

```
jhow@nuc4:~$ vault token renew
Error renewing token: Error making API request.

URL: PUT https://vault.problemofnetwork.com:8200/v1/auth/token/renew-self
Code: 400. Errors:

* client certificate must be supplied
```

The `vault` CLI isn't presenting Alice's certificate, so Vault won't renew. You can see why that's reasonable: a stolen token on its own can't be kept alive indefinitely. The fix is to not bother with renewal. Just run yubivault again, get a fresh token, and carry on. If you are on a YubiKey it will ask for the PIN (and a touch, if you set a touch policy when you generated the key).

The role's TTLs are the safety net here. An hour for the token and eight in total means a leaked token is useful to an attacker for an hour at the most unless they also have the certificate and its key, and with a YubiKey they can't have the key. That's a much better position than a long-lived static token sitting in an env file.

## Revocation And Renewal Of The Certificates Themselves

Certificates expire (ours are a year) and people leave. Both are handled on the CA side. To renew, the issuer moves the old certificate aside and signs the same CSR again. To remove someone, the issuer revokes the certificate with `certstrap revoke` and uploads the updated CRL to Vault with `vault write auth/cert/crls/...`. Vault only knows about the revocation list you last gave it, so you repeat the upload after every revocation. Tokens already issued stay valid until they expire unless you revoke them too, which is another reason for the short TTLs. The exact commands, including the renewal dance, are in the Renewal and Revocation sections of [PKI-SETUP.md](https://github.com/fatred/yubivault/blob/main/PKI-SETUP.md), and I'd rather you follow the guide than copy a half-remembered version from a blog.

---

## Wrapping Up The Series

That is the end of the Vault series. We started with a [dev server](/posts/bootstrapping-hashi-vault/) and a pile of secrets, moved on to [primitives](/posts/hashi-vault-primitives/), pushed it through [python](/posts/making-use-of-vault-python/) and [ansible](/posts/making-use-of-vault-ansible/), built a [PKI](/posts/building-vault-pki/) for devices, looked at [TLS for network appliances](/posts/vault-tls-with-network-appliances/), [audit logging](/posts/vault-audit-logging/) and [transit encryption](/posts/vault-transit-encryption/), and finally got the front door properly locked, with a server that proves who it is and users that prove who they are.

It's still a lab. A real deployment wants a proper cluster, an auto-unseal story and a plan for those offline CA keys that doesn't involve my home directory. But every piece here is the same piece you'd use in anger.

The [yubivault](https://github.com/fatred/yubivault) repo has the tool and the [PKI-SETUP.md](https://github.com/fatred/yubivault/blob/main/PKI-SETUP.md) guide, including the YubiKey specifics (PIN changes, touch policies, import) that I only brushed past. Issues and pull requests are welcome.

If you made it this far, thanks for sticking with the whole series.

Until next time, thanks for stopping by, and toodleoo. :D
