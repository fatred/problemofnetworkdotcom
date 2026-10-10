---
title: "Securing your Vault instance with TLS"
date: 2026-10-10T22:30:00+02:00
author: John Howard
tags: ["vault", "tls", "certstrap"]
showFullContent: false
---

This post and [the next one](/posts/securing-vault-logins-with-client-certificates/) wrap up the Vault series, and they are where I pay back a debt. Way back in the [bootstrap](/posts/bootstrapping-hashi-vault/) post I looked at the docker setup and said I would want TLS on something "real", but that it was "a level of faff i'm not up for today". Well, it is today. We have spent a lot of posts putting things into Vault ([primitives](/posts/hashi-vault-primitives/), [python](/posts/making-use-of-vault-python/), [ansible](/posts/making-use-of-vault-ansible/), [PKI](/posts/building-vault-pki/), [audit](/posts/vault-audit-logging/), [transit](/posts/vault-transit-encryption/)), all of it over plain HTTP, which is a bit embarrassing for a secrets store.

There are two jobs to do:

* Put a proper TLS certificate on the Vault listener, so clients can verify they are talking to the right server. That's this post.
* Use certificates the other way round, to log people in, with the private key sitting on a YubiKey. That's the next one.

Both use **offline [certstrap](https://github.com/square/certstrap) CAs**. They are deliberately not the Vault PKI engine we built in the [PKI](/posts/building-vault-pki/) post. Vault cannot issue the certificate for its own listener before that listener exists, and you really don't want the thing that authenticates you to depend on itself. If Vault is down, you want to still be able to reason about who can log back in to it.

We will end up with two CAs with separate jobs:

* **Problem Of Network Infra CA** signs server certificates. Clients trust it so they can check Vault is Vault. We build it here.
* **Problem Of Network Users CA** signs certificates for people. Vault trusts it so it can check who is logging in. That one comes in the next post.

Keeping them apart is about blast radius. If the Infra CA key leaks, someone can impersonate a server, which is bad, but they cannot mint a login. If the Users CA key leaks, someone can mint logins, but clients don't trust it for servers. Neither leak gives away the other.

> Note: I am still assuming the docker compose layout from the [bootstrap](/posts/bootstrapping-hashi-vault/) post: `~/vault` on the Vault host, with `config/config.hcl` and the `file` and `logs` volumes. Everything below is a small delta on top of that.

---

## TLS For The Vault Listener

### DNS First

Before you make a single certificate, sort out the name. Pick the name your clients will use, make sure it resolves to the Vault box, and put that same name in the certificate. I'm using `vault.problemofnetwork.com` throughout; swap in your own.

```
jhow@nuc4:~$ getent hosts vault.problemofnetwork.com
192.168.99.247  vault.problemofnetwork.com
```

The reason this matters comes in two flavours. First, if you connect by IP address, the certificate will not match. We are about to issue a certificate with a single DNS name in it, and nothing else, so this is what connecting by IP looks like once TLS is on:

```
jhow@nuc4:~$ curl -sS https://192.168.99.247:8200/v1/sys/health
curl: (60) SSL: no alternative certificate subject name matches target ipv4 address '192.168.99.247'
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.
```

Second, if the name points at the wrong place, clients don't fail neatly, they just hang waiting for a connect that never completes. yubivault gives up after about 11 seconds with a dial error naming the address it tried (`dial tcp 10.62.0.1:8200: i/o timeout`), so if you see that, check the address before you go digging in certificates.

### Make The Infra CA And The Server Certificate

> WARNING: Remember this is for testing purposes people! Certstrap is designed and built to make LAB CAs that verify properly VERY easy. Use LetsEncrypt, or your internal corporate CA for real world deployments.

On whichever machine you want to keep your CA on, install certstrap. It is a Go tool, so:

```shell
go install github.com/square/certstrap@latest
```

certstrap keeps everything in a directory called `out/` under wherever you run it, which it calls the depot. I made a directory called `~/pon-infra-ca` for this and ran every command from inside it. The Users CA later gets its own directory, `~/pon-users-ca`, so the two CAs never share a depot.

First the CA. This asks for a passphrase to encrypt the CA key, and you want to set one. Then a request for the server, and a signature on that request:

```
jhow@nuc4:~/pon-infra-ca$ certstrap init --common-name "Problem Of Network Infra CA" --expires "10 years"
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Created out/Problem_Of_Network_Infra_CA.key (encrypted by passphrase)
Created out/Problem_Of_Network_Infra_CA.crt
Created out/Problem_Of_Network_Infra_CA.crl

jhow@nuc4:~/pon-infra-ca$ certstrap request-cert --common-name vault.problemofnetwork.com --domain vault.problemofnetwork.com
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Created out/vault.problemofnetwork.com.key
Created out/vault.problemofnetwork.com.csr

jhow@nuc4:~/pon-infra-ca$ certstrap sign vault.problemofnetwork.com --CA "Problem Of Network Infra CA" --expires "1 year"
Enter passphrase for CA key (empty for no passphrase): 
Created out/vault.problemofnetwork.com.crt from out/vault.problemofnetwork.com.csr signed by out/Problem_Of_Network_Infra_CA.key
```

Two passphrase prompts look similar but mean different things:

* On `init`, the passphrase protects the **CA key**. That key can sign anything, so it stays passphrase-protected, and it stays offline. Back it up, put it somewhere that isn't the Vault host, and don't leave it lying around on a server.
* On `request-cert`, I left the passphrase **empty**. That is the server key, and Vault has to read it on start-up with nobody there to type anything. A passphrase here would mean you couldn't restart Vault unattended. The protection for this key is file permissions on the Vault host, not a passphrase.

Here is what ended up in the depot:

```
jhow@nuc4:~/pon-infra-ca$ ls -l out/
total 24
-r--r--r-- 1 jhow jhow  967 Oct 10 22:06 Problem_Of_Network_Infra_CA.crl
-r--r--r-- 1 jhow jhow 1809 Oct 10 22:06 Problem_Of_Network_Infra_CA.crt
-r--r----- 1 jhow jhow 3311 Oct 10 22:06 Problem_Of_Network_Infra_CA.key
-r--r--r-- 1 jhow jhow 1598 Oct 10 22:06 vault.problemofnetwork.com.crt
-r--r--r-- 1 jhow jhow  989 Oct 10 22:06 vault.problemofnetwork.com.csr
-r--r----- 1 jhow jhow 1679 Oct 10 22:06 vault.problemofnetwork.com.key
```

The `.crt` and `.crl` files are public. The `.key` files are not, and certstrap makes them readable only by you. It's worth checking the certificate says what we think it says before it goes anywhere near Vault:

```
jhow@nuc4:~/pon-infra-ca$ openssl x509 -in out/vault.problemofnetwork.com.crt -noout -subject -issuer -dates -ext subjectAltName,extendedKeyUsage
subject=CN=vault.problemofnetwork.com
issuer=CN=Problem Of Network Infra CA
notBefore=Oct 10 19:56:42 2026 GMT
notAfter=Oct 10 20:06:41 2027 GMT
X509v3 Extended Key Usage: 
    TLS Web Server Authentication, TLS Web Client Authentication
X509v3 Subject Alternative Name: 
    DNS:vault.problemofnetwork.com

jhow@nuc4:~/pon-infra-ca$ openssl verify -CAfile out/Problem_Of_Network_Infra_CA.crt out/vault.problemofnetwork.com.crt
out/vault.problemofnetwork.com.crt: OK
```

The bit to look at is the subject alternative name. Modern clients ignore the CN for hostname checks and only look at SANs, and this one has exactly one: the DNS name we want. This is also why the IP address failed earlier. The validity is one year, so put a reminder in your calendar. A server certificate that quietly expires is a classic way to ruin a Monday.

### Put It On The Vault Host

Now get the certificate and key onto the Vault host. Following the layout from the bootstrap post, I put them in `~/vault/certs/`, renamed to something generic so the config doesn't change when you rotate the cert:

```
jhow@vault:~/vault$ mkdir -p certs
jhow@vault:~/vault$ cp vault.problemofnetwork.com.crt certs/server.crt
jhow@vault:~/vault$ cp vault.problemofnetwork.com.key certs/server.key
jhow@vault:~/vault$ cp Problem_Of_Network_Infra_CA.crt certs/ca.crt
jhow@vault:~/vault$ sudo chown 100:1000 certs/server.key
jhow@vault:~/vault$ sudo chmod 400 certs/server.key
```

I assumed the files were already copied across from the depot (scp, a USB stick, carrier pigeon). The `chown` is the same `100:1000` (vault:vault inside the image) trick from the bootstrap post, so the container can read a key that nobody else can. The CA certificate comes along too; we need it for the CLI inside the container in a moment.

Then mount that directory read-only in the compose file, next to the config volume. While we are in there, the in-container shell needs some love. `VAULT_ADDR` has to become https and use the name on the certificate, otherwise the CLI inside the container fails the same SAN check that curl did. That name also has to resolve from inside the container, and pointing it at loopback keeps the CLI talking to its own listener rather than hairpinning out through the host's published port. Finally, the CLI has to trust the Infra CA. Three small additions do it:

```yaml
services:
  vault:
    image: hashicorp/vault:2.1.0
    container_name: vault
    environment:
      VAULT_ADDR: https://vault.problemofnetwork.com:8200
      VAULT_CACERT: /vault/certs/ca.crt
    extra_hosts:
      - "vault.problemofnetwork.com:127.0.0.1"
    ports:
      - "8200:8200"
    restart: always
    volumes:
      - ./config:/vault/config:ro
      - ./certs:/vault/certs:ro
      - ./file:/vault/file
      - ./logs:/vault/logs
    command: server
```

The new bits are the `./certs:/vault/certs:ro` volume, `VAULT_CACERT` and the `extra_hosts` entry, which points the name at the container's own loopback so `docker compose exec vault vault status` works with full verification. Next, `config/config.hcl`. The listener from the bootstrap post had `tls_disable = "true"`. Swap that for the certificate and key, and tell Vault what URL it is advertised as:

```hcl
ui = true
disable_mlock = "true"
api_addr = "https://vault.problemofnetwork.com:8200"

storage "file" {
  path    = "/vault/file"
}

listener "tcp" {
  address       = "[::]:8200"
  tls_cert_file = "/vault/certs/server.crt"
  tls_key_file  = "/vault/certs/server.key"
}
```

Then restart it:

```
jhow@vault:~/vault$ docker compose up -d --force-recreate
```

> Note: a restart re-seals Vault, so you will need to do the three-of-five unseal dance again, as in the bootstrap post. That is the price of the file backend and the Shamir seal, and it is a good reminder of why you would automate this in anything real.

On your client machine, `VAULT_ADDR` is now https and uses the name:

```shell
export VAULT_ADDR=https://vault.problemofnetwork.com:8200
```

Does Vault actually think it is listening with that certificate? Rather than trust my config file, we can ask Vault what it loaded. `vault status` works over the new URL, and the sanitized config endpoint shows the listener it picked up:

```
jhow@nuc4:~$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         2.1.2
Build Date      2026-10-06T15:17:37Z
Storage Type    file
Cluster Name    vault-cluster-e619b56e
Cluster ID      96013715-71ac-d3fb-a97f-f8c5de8c59c7
HA Enabled      false

jhow@nuc4:~$ vault read -format=json sys/config/state/sanitized | jq .data.listeners
[
  {
    "config": {
      "address": "[::]:8200",
      "tls_cert_file": "/vault/certs/server.crt",
      "tls_disable": "false",
      "tls_key_file": "/vault/certs/server.key"
    },
    "type": "tcp"
  }
]
```

`tls_disable` is `false` and the cert and key paths are the ones inside the container. That is the proof.

### Trust The Infra CA On Clients

Right now, no client trusts our Infra CA, so they all refuse the connection. That is the system working as intended. Here is the failure on a Debian/Ubuntu box, the fix, and the same request working afterwards:

```
jhow@nuc4:~$ curl -sS https://vault.problemofnetwork.com:8200/v1/sys/health
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.

jhow@nuc4:~$ sudo cp Problem_Of_Network_Infra_CA.crt /usr/local/share/ca-certificates/

jhow@nuc4:~$ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt, it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.

jhow@nuc4:~$ curl -sS https://vault.problemofnetwork.com:8200/v1/sys/health | jq
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
  "replication_dr_mode": "disabled",
  "server_time_utc": 1791662875,
  "version": "2.1.2",
  "enterprise": false,
  "cluster_name": "vault-cluster-e619b56e",
  "cluster_id": "96013715-71ac-d3fb-a97f-f8c5de8c59c7",
  "echo_duration_ms": 0,
  "clock_skew_ms": 0,
  "replication_primary_canary_age_ms": 0
}
```

The `rehash` warning is a harmless quirk of the bundle file, and `1 added` is the line you care about. On macOS, the equivalent is one command, which puts the CA into the system keychain as a trusted root:

```shell
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain Problem_Of_Network_Infra_CA.crt
```

This is the public certificate only. You never copy the CA key anywhere near a client. Anything that needs to talk to Vault (CI runners, Ansible controllers, the python from the earlier posts) needs the same trust, or you point it at the CA file explicitly. It also means you can finally retire `VAULT_SKIP_VERIFY` from the env vars table in the primitives post. Good riddance.

Finally, a look at what is negotiated. `openssl s_client -brief` is the quickest way to see the TLS version, the peer certificate and, importantly, the verification result:

```
jhow@nuc4:~$ openssl s_client -brief -connect vault.problemofnetwork.com:8200 </dev/null
Connecting to 192.168.99.247
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_128_GCM_SHA256
Requested Signature Algorithms: RSA-PSS+SHA256:ECDSA+SHA256:ed25519:RSA-PSS+SHA384:RSA-PSS+SHA512:ECDSA+SHA384:ECDSA+SHA512
Peer certificate: CN=vault.problemofnetwork.com
Hash used: SHA256
Signature type: rsa_pss_rsae_sha256
Verification: OK
Negotiated TLS1.3 group: X25519MLKEM768
DONE
```

`Connecting to 192.168.99.247` shows the name resolving to the right box, TLS 1.3 is in use, and `Verification: OK` says the chain checked out against the CA we just trusted. Vault is on https, with a certificate we control.

In the [next post](/posts/securing-vault-logins-with-client-certificates/) we use certificates the other way round: a second certstrap CA for people, Vault's cert auth method, and team access by OU, with the key on a YubiKey.

Until next time, thanks for stopping by.
