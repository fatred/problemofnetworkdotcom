---
title: "Making Use of Vault: Python"
date: 2024-11-19T09:42:22+01:00
---
# WIP!

It's remarkably easy to get sucked into hardcoding things that probably should live outside your code. 

Recently I found myself trying to clean up a lot of bad behaviour in our repos, not least of all, secrets management. Here we will cover a few basics that show how to use Hashicorp Vault in more places. 

> If you want to get into the examples below, then I recommend you follow the [bootstrap post](/posts/bootstrapping-hashi-vault/) to get a vault setup that matches the below examples. If you want to understand more about the primitives, thats is covered [here](/posts/hashi-vault-primitives/). The git repo can also be found [here](github.com/fatred/practical-vault-for-netops.git)

---

## Vault in Python

Here we cover the use case of once-shot scripting and something more structured like Temporal or Stackstorm.

Main Options: 
* HTTPS API via requests library (easy, but clunky)
* Python hvac library (easy, pythonic, but seems quite "engineered")

### HTTPS API use

Long story short you want the requests library. Here we are literally reproducing the work we did with `curl` in primitives, but in `requests` instead.

```python
import requests

headers = {
    'X-Vault-Request': 'true',
    # change this to your initial root token if you are using my docker vault...
    'X-Vault-Token': 'root',
    'Content-Type': 'application/json',
    'Accept': 'application/json',
}

json_data = {
    'data': {
        'wireguard': {
            'keys': {
                'privkey': 'privkey1',
                'pubkey': 'pubkey2',
            },
            'ip': {
                'outer': '100.64.0.0/30',
                'inner': '100.64.255.0/31',
            },
        },
    },
    'options': {},
}

response = requests.put('http://localhost:8200/v1/secret/data/mycomplicatedsecret', headers=headers, json=json_data)

print(response.json())
```
When we run this, assuming nothing failed, you will get a json reply like the following: 
```
{'request_id': '1bec0773-68b9-4a90-b254-fc511cd3cfef', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'created_time': '2024-11-19T13:50:32.015432Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 3}, 'wrap_info': None, 'warnings': None, 'auth': None, 'mount_type': 'kv'}
```

And so now lets pull it back into an object.

```python
import requests

headers = {
    'X-Vault-Request': 'true',
    'X-Vault-Token': 'root',
    'Content-Type': 'application/json',
    'Accept': 'application/json',
}

response = requests.get('http://localhost:8200/v1/secret/data/mycomplicatedsecret', headers=headers)

print(response.json())
```

here the response comes back as a json block, and there is a bit of "fluff" that comes with it. Lets cheat, and use the fancier version of print from the rich package

```python

>>> from rich import print
>>> print(response.json())
{
    'request_id': 'cf17b7ce-cda7-e8d7-7370-ca757698fd19',
    'lease_id': '',
    'renewable': False,
    'lease_duration': 0,
    'data': {
        'data': {'wireguard': {'ip': {'inner': '100.64.255.0/31', 'outer': '100.64.0.0/30'}, 'keys': {'privkey': 'privkey1', 'pubkey': 'pubkey2'}}},
        'metadata': {'created_time': '2024-11-19T13:50:32.015432Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 3}
    },
    'wrap_info': None,
    'warnings': None,
    'auth': None,
    'mount_type': 'kv'
}
```

So you see, the bit we want is in a dictionary nested further down...

```
>>> print(response.json()['data']['data'])

{'wireguard': {'ip': {'inner': '100.64.255.0/31', 'outer': '100.64.0.0/30'}, 'keys': {'privkey': 'privkey1', 'pubkey': 'pubkey2'}}}
```

The HTTP API is a perfectly decent way to interact with vault. I can imagine that a lot of these workflow orchestrators that dont offer SDKs/native bindings, would have HTTP API support. If you can't talk to vault with an SDK, this is a very effective method.



## Vault in Ansible

Main Options: 
* HTTPS API via uri (but...don't do this to yourself)
* Community "Hashi Vault" collection: [link](https://github.com/ansible-collections/community.hashi_vault)

### HTTPS API use

> I will hit this extremely briefly, because I think its a terrible idea. There is ONE case where this makes sense - you have to get one small little thing out and stick it in a variable - thats it. Anythind more complicated you should do it with the collection "properly".

