---
title: "Vault Audit Logging"
date: 2025-01-12T23:07:17+01:00
---

I thought i was done with this series, but there are a few loose ends that I think we can clear up pretty quickly. The most important of which is Audit Logging, because what is the point of a secure secrets tool if you don't track who does what (or most importantly, _fails_ to do what) with it. Lets jump in!

## Enabling audit logs

Enabling audit logging requires you to tell the vault server that it should use one of the audit "device types" that it offers with the required parameters. The system will then send audit events to _all_ defined devices, and will consider the audit entry a success when at least one confirms the write.

If all the configured devices are inaccessible (socket fail?), or inoperable (disk full?) then vault will refuse to accept any requests. More on the blocking device semantics can be found [here](https://developer.hashicorp.com/vault/tutorials/monitoring/blocked-audit-devices).

TL:DR, you should probably log to file, using a dedicated LVM path, and you should always log off box using logstash/vector/rsyslog or similar. Logging to two diverse off box locations is probably the best answer for secure environments.

When doing so, you should _always_ consider that one may not have all the logs and the "union" of these logs is the whole truth. This is why two remote systems with identical control planes will make it easier to deduplicate logs later.

### The file device

This is remarkably easy. Make sure your environment vars are right and you either have the root token or one with policies for audit, then simply run:

`vault audit enable file file_path=/vault/logs/audit.log`

where the `file_path` reflects a location that works for you. If you are still using my demo repo, this path here is the logs folder within the docker repo, as mounted inside the container. You should immediately be able to see (via sudo), log lines appearing in that file.

### The socket device

The socket device is just as easy, but there is one step you should do before. Since the absense of the device _could_ lead to blocking if its the only audit device in use, you should ensure the logging service is up and listening _BEFORE_ you configure it in vault.

First, lets use `nc` to ensure we can see this socket in question:

```
nc -vw2 192.168.99.252:8000
Connection to 192.168.99.252 8000 port [tcp/*] succeeded!
```

> If that times out after 2 seconds, fix that before you proceed!

Now we setup the socket

`vault audit enable socket address=192.168.99.252:8000 socket_type=tcp`

We should now start to see logs appearing on the logging platform of choice.

### The (lack of) data schema

Here we have an example from the file device

```json
{
  "auth": {
    "accessor": "hmac-sha256:6fc3809aa4009064f8431de7042ad2659eb4e07f3e6b60e821dc3fe4141de344",
    "client_token": "hmac-sha256:60c07b5f07b544ea93ecb13b4e8219d0ac34c36989fe8293b5ef28bec8ef3182",
    "display_name": "root",
    "policies": [
      "root"
    ],
    "policy_results": {
      "allowed": true,
      "granting_policies": [
        {
          "type": ""
        },
        {
          "name": "root",
          "namespace_id": "root",
          "type": "acl"
        }
      ]
    },
    "token_policies": [
      "root"
    ],
    "token_issue_time": "2024-11-18T16:19:28Z",
    "token_type": "service"
  },
  "request": {
    "client_id": "0DHqvq2D77kL2/JTPSZkTMJbkFVmUu0TzMi0jiXcFy8=",
    "client_token": "hmac-sha256:60c07b5f07b544ea93ecb13b4e8219d0ac34c36989fe8293b5ef28bec8ef3182",
    "client_token_accessor": "hmac-sha256:6fc3809aa4009064f8431de7042ad2659eb4e07f3e6b60e821dc3fe4141de344",
    "id": "3a9e1103-8624-37e0-a5d2-ac3404a05f64",
    "mount_accessor": "kv_d906bfe9",
    "mount_class": "secret",
    "mount_point": "secret/",
    "mount_running_version": "v0.20.0+builtin",
    "mount_type": "kv",
    "namespace": {
      "id": "root"
    },
    "operation": "read",
    "path": "secret/data/mysimplesecret",
    "remote_address": "172.22.0.1",
    "remote_port": 52446
  },
  "response": {
    "data": {
      "data": {
        "password": "hmac-sha256:fb07ae054bb6f2560e1a03b3bc6486ef7a8b868848db7c265d796c4f4e25d09c",
        "username": "hmac-sha256:08fe172405cd9d81b7e8f28783764b2f942b5a0146aca12bc7edc2abd08b6f59"
      },
      "metadata": {
        "created_time": "hmac-sha256:ce97fad5f72d1f15130c53c9ba2a683b87b9114f1ede09b5e285742437bdcc2d",
        "custom_metadata": null,
        "deletion_time": "hmac-sha256:e08b387af763558841d5877cf1d4643d871689912d4c58c6e31a890b96bcb295",
        "destroyed": false,
        "version": 1
      }
    },
    "mount_accessor": "kv_d906bfe9",
    "mount_class": "secret",
    "mount_point": "secret/",
    "mount_running_plugin_version": "v0.20.0+builtin",
    "mount_type": "kv"
  },
  "time": "2025-01-12T21:55:07.656951011Z",
  "type": "response"
}
```

First, lets clear one thing up, there is a lot of metadata in here, but actually, the secrets are all hashed before logging, which _should_ mean that the content is safe to store in a central store. 

I say _should_ because there are a couple of caveats [in the documentation](https://developer.hashicorp.com/vault/docs/audit#sensitive-information)

> Currently, only strings that come from JSON or returned in JSON are HMAC'd. Other data types, like integers, booleans, and so on, are passed through in plaintext. We recommend that all sensitive data be provided as string values inside all JSON sent to Vault (i.e., that integer values are provided in quotes).

In our example, we have clean k/v data which marshals into JSON cleanly, so its hashed. Phew!

Now when you want to send JSON to something like Elastic, one would typically provide a JSON Schema to the engine so that the data is stored efficently in the backend. When you read the [documentation](https://developer.hashicorp.com/vault/docs/audit#format), we are reminded that the format is simply auth/request/response.

Within these keys, there is some reliable structure, and some that depends on the data you have in your request (the auth type and the engine you used) and some that varies with the response (the actual dictionary structure your secret uses)

Worry not, elastic has an [integration](https://www.elastic.co/guide/en/integrations/current/hashicorp_vault.html) in the standard packages, so best you just use that!

## Leveraging the log output

Now since we have the log data exported as JSON, it will not require complex regex and grok, and should be easy to search already.

lets start with a few examples taken from the [hashicorp docs](https://developer.hashicorp.com/vault/tutorials/monitoring/query-audit-device-logs)

> I exported the logfile location as per the note in the docs like so: `export AUDIT_LOG_FILE=logs/audit.log`

```shell
jhow@vault:~/vault$ sudo jq -n '[inputs | {DisplayName: .auth.display_name | select(. != null)} ] | group_by(.DisplayName) | map({DisplayName: .[0].DisplayName, Count: length})  | .[]' $AUDIT_LOG_FILE 
{
  "DisplayName": "root",
  "Count": 9
}

jhow@vault:~/vault$ sudo jq -n '[inputs | {Operation: .request.operation} ] | group_by(.Operation) | map({Operation: .[0].Operation, Count: length}) | .[]' $AUDIT_LOG_FILE 
{
  "Operation": "list",
  "Count": 2
}
{
  "Operation": "read",
  "Count": 6
}
{
  "Operation": "update",
  "Count": 2
}
```

Now lets look at the examples for "badness". Here I will make am attempt to list a non-existant kv store called `secret` (ours is called `secrets`) 

```shell
jhow@vault:~/vault$ vault kv list secrets
Error making API request.

URL: GET http://localhost:8200/v1/sys/internal/ui/mounts/secrets
Code: 403. Errors:

* preflight capability check returned 403, please ensure client's policies grant access to path "secrets/"
```

Now if we look for the errors:

```shell
jhow@vault:~/vault$ sudo jq -n '[inputs | {Errors: .error} ] | group_by(.Errors) | map({Errors: .[0].Errors, Count: length}) | sort_by(-.Count) | .[]' $AUDIT_LOG_FILE
{
  "Errors": null,
  "Count": 15
}
{
  "Errors": "permission denied",
  "Count": 1
}
```

So, I already hear the questions - why log that with a False Positive!?!

Well, in security terms, vault has a client with a token and that client is requesting a secrets path that _is not in their policy_, and so it replies with a "permission denied". In other words, vault isn't "trying" to access that resource, and then getting rejected. Its actually looking at the policy assigned to the token, establishing that the requested secret store is not covered there, and blindly replies with "no". The fact the secret store in question doesnt exist, is irrelevant. 

Consider that if vault did the request to the backend, and then somehow replied back differently (knowing or unknowingly), you will be providing the requesting client with at the very least, timing information that could become useful in a hands on scenario.

## wrapping up

This was a much shorter post that the previous ones, because the purpose here was to ensure that people have the audit logging enabled. Most of this post could be TL:DRd as links to the hashi docs lets be honest. 

I hope that you found it useful however, and until next time. Toodleoo ;)