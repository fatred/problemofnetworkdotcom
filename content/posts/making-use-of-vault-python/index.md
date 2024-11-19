---
title: "Making Use of Vault: Python"
date: 2024-11-19T09:00:00+01:00
author: John Howard
tags: ["vault", "netdevops", "ansible", "python"]
showFullContent: false
---

It's remarkably easy to get sucked into hardcoding things that probably should live outside your code. 

It is clear to many of us that storing secrets anywhere that isn't vault (or something like it), is a terrible practice. It is also true that the best laid plans of mice and men _aft gan aglais_. 

In other words, the problem is rarely that we don't want to do secure coding, its that we lack the time, talent, or awareness to do this right. Don't dwell on that too much either - its just how the world works. 

Here I want to share some snippets that break down the barrier to entry at least for some, in the hope that you don't have to waste as much time as I did, figuring out how this stuff works.

> If you want to get into the examples below, then I recommend you follow the [bootstrap post](/posts/bootstrapping-hashi-vault/) to get a vault setup that matches the below examples. If you want to understand more about the primitives, thats is covered [here](/posts/hashi-vault-primitives/).

---

## Vault in Python

Here we cover the use case of once-shot scripting and/or something more structured like Temporal or Stackstorm.

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


### The hvac library

In Vault land, HVAC is nothing to do with aircon.

#### making your env

Lets start small. in your shell visit the same ~/vault folder that we may or may not have created for the vault server:

`mkdir ~/vault ; cd ~/vault`

In here, lets create a python venv. If you don't have python installed, where did you expect this document to take you? I fear you may be lost. I would suggest that you actually use a specific version, like `python3.11`, but for simplicity, lets just use the symlink

`python3 -m venv venv && source venv/bin/activate`

We can now install hvac with pip

`pip3 install hvac`

Lets start simple, and use the create_or_update_secrets function to submit the objects that may or may not exist from the primitives post. If they don't exist, they will be created, if they do, they will be updated to match your _exact_ spec.

```python
from hvac import Client

# where is our vault server?
VAULT_ADDR: str = "http://localhost:8200"
# replace with your initial vault token if you are using docker
VAULT_TOKEN: str = "root"

# lets make an instance of the hvac cault client
vault = Client(url=VAULT_ADDR, token=VAULT_TOKEN)

# lets check if this token is valid and the server is accessible
if not vault.is_authenticated:
    raise RuntimeError("Vault setup failed.")
else:
    print("vault is authenticated!")

# great! lets make a dictionary that represents what we want to update
complex_data: dict = {
    "wireguard": {
        "ip": {"inner": "100.64.255.0/31", "outer": "100.64.0.0/30"},
        "keys": {"privkey": "privkey1", "pubkey": "pubkey2"},
    }
}

# and lets describe where we will be storing this
mount_point = "secret"
secret_path = "mycomplexsecret"

# lets fire the request in that will create_or_update that secret.
results = vault.secrets.kv.v2.create_or_update_secret(
    mount_point=mount_point, path=secret_path, secret=complex_data
)
print(results)
```

when we run this on an empty vault, we will get something like this, where we can see the `results['data']['version'] = 1`. 

```
{'request_id': '9678d3c7-186c-d37b-1a28-a94a274625d0', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'created_time': '2024-11-19T20:54:59.25745Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}, 'wrap_info': None, 'warnings': None, 'auth': None, 'mount_type': 'kv'}
```

If we push this to a vault instance with this data already in situ, the only thing that is different is the version number increased, and the "created_time" now reports the time you submitted that "update":

```
{'request_id': '07fada74-18de-eea2-0c04-bd41c498a2f3', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'created_time': '2024-11-19T20:59:20.087158Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 2}, 'wrap_info': None, 'warnings': None, 'auth': None, 'mount_type': 'kv'}
```

This is a handy thing:

Firstly, you can pull the data before, send an update, and if the version increases, you have a successful commit. 

Secondly, you can choose to make your client/service/app aware of version numbers, and use this to ensure you always get that version of a secret (like using a git commit hash or a tag). Add the following to the bottom of that file for example: 

```python
my_version1 = vault.secrets.kv.v2.read_secret_version(mount_point=mount_point, path=secret_path, raise_on_deleted_version=False, version=1)
print(f"v1: {my_version1}")

my_version2 = vault.secrets.kv.v2.read_secret_version(mount_point=mount_point, path=secret_path, raise_on_deleted_version=False, version=1)
print(f"v2: {my_version2}")
```

...we will see that the version number and the timestamp are in fact different.

```
>>> my_version1 = vault.secrets.kv.v2.read_secret_version(mount_point=mount_point, path=secret_path, raise_on_deleted_version=False, version=1)
>>> print(f"v1: {my_version1}")
v1: {'request_id': '4c4bd94c-c83e-03b0-ca05-0b458b99b33c', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'data': {'wireguard': {'ip': {'inner': '100.64.255.0/31', 'outer': '100.64.0.0/30'}, 'keys': {'privkey': 'privkey1', 'pubkey': 'pubkey2'}}}, 'metadata': {'created_time': '2024-11-19T20:54:59.25745Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}}, 'wrap_info': None, 'warnings': None, 'auth': None, 'mount_type': 'kv'}
>>>
>>> my_version2 = vault.secrets.kv.v2.read_secret_version(mount_point=mount_point, path=secret_path, raise_on_deleted_version=False, version=1)
>>> print(f"v2: {my_version2}")
v2: {'request_id': 'ae4d335c-97a8-0a9b-764f-14b29038c9ca', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'data': {'wireguard': {'ip': {'inner': '100.64.255.0/31', 'outer': '100.64.0.0/30'}, 'keys': {'privkey': 'privkey1', 'pubkey': 'pubkey2'}}}, 'metadata': {'created_time': '2024-11-19T20:54:59.25745Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}}, 'wrap_info': None, 'warnings': None, 'auth': None, 'mount_type': 'kv'}
```

Again, this is kinda ugly and we have a bunch of metadata in the mix. If we just want that data, we can trim the results with dictionary references:

```
>>> my_version1 = vault.secrets.kv.v2.read_secret_version(mount_point=mount_point, path=secret_path, raise_on_deleted_version=False, version=1)
>>> print(f"v1: {my_version1['data']['data']}")
v1: {'wireguard': {'ip': {'inner': '100.64.255.0/31', 'outer': '100.64.0.0/30'}, 'keys': {'privkey': 'privkey1', 'pubkey': 'pubkey2'}}}
```

Why is is `my_version1['data']['data']`? No idea sorry.

## Bonus content: mini-library

In my work with [Temporal](https://temporal.io), I found that I needed some little utility modules that would host some re-usable things and simplify my interactions with tools like vault. This is by no means complete, but it should get you moving if you are on the same path that I am.

first, a test script:

`use_vault.py`:
```python
rom vault_services import (
    quick_vault_auth_from_env,
    export_vault_client_token,
    quick_vault_auth_from_token,
    fetch_kv,
    SecretRequest,
)

env_client = quick_vault_auth_from_env()

env_client_token = export_vault_client_token(env_client)

token_client = quick_vault_auth_from_token(token=env_client_token)

my_request_on_env_client = SecretRequest(
    vault_token=env_client_token,
    secret_mount="secret",
    secret_path="mysimplesecret",
    secret_name="username",
)

env_client_response = fetch_kv(secret_request=my_request_on_env_client)

print(
    f"you asked for {my_request_on_env_client.model_dump()['secret_name']} and the result was {env_client_response.model_dump()['value']}"
)
```

and here is the library file that you put in the same folder to abstract these functions away...

`vault_services.py`:
```python
from hvac import Client
from os import environ
from pydantic import BaseModel

# constants that _could_ come from env
# vault token
if not environ.get("VAULT_TOKEN"):
    VAULT_TOKEN: str = "root"
else:
    VAULT_TOKEN: str = environ.get("VAULT_TOKEN")
# vault address
if not environ.get("VAULT_ADDR"):
    VAULT_ADDR = "http://127.0.0.1:8200"
else:
    VAULT_ADDR: str = environ.get("VAULT_ADDR")
# vault tls true/false
if not environ.get("VAULT_INSECURE"):
    VAULT_INSECURE: True
else:
    VAULT_INSECURE: bool = environ.get("VAULT_INSECURE")

# constants that are in the script only
VAULT_TOKEN_MIN_TTL = 30


class SecretRequest(BaseModel):
    vault_token: str
    secret_mount: str
    secret_path: str
    secret_name: str


class SecretResponse(BaseModel):
    key: str
    value: str


def quick_vault_auth_from_env() -> Client:
    """
    Leverage an env var and login to vault quickly

    Raises:
        RuntimeError: Vault is sad.

    Returns:
        hvac.Client: Authenticated Vault client
    """
    # spin an instance of the client
    vault_client = Client(url=VAULT_ADDR, token=VAULT_TOKEN, verify=not VAULT_INSECURE)
    # bypass for dev token
    if VAULT_TOKEN == "root":
        pass
    else:
        # check if the token is valid and alive
        if not _check_vault_token_ttl_ok(vault_client):
            # we have less than 30s - elaborate.
            if not _check_vault_token_expired(vault_client):
                # we are not expired, but we have less than 30s - renew
                _renew_vault_token(vault_client)
            else:
                raise Exception("Token is already expired!")
        else:
            # we have > 30s of validity - carry on.
            pass
    # if we got here, we have a valid token with > 30s of life; done.
    return vault_client


def quick_vault_auth_from_token(token: str) -> Client:
    """
    Leverage a provided token and login to vault quickly

    Raises:
        RuntimeError: Vault is sad.

    Returns:
        hvac.Client: Authenticated Vault client
    """
    # spin an instance of the client
    vault_client = Client(url=VAULT_ADDR, token=token, verify=not VAULT_INSECURE)
    # bypass for dev token
    if token == "root":
        pass
    else:
        # check if the token is valid and alive
        if not _check_vault_token_ttl_ok(vault_client):
            # we have less than 30s - elaborate.
            if not _check_vault_token_expired(vault_client):
                # we are not expired, but we have less than 30s - renew
                _renew_vault_token(vault_client)
            else:
                raise Exception("Token is already expired!")
        else:
            # we have > 30s of validity - carry on.
            pass
    # if we got here, we have a valid token with > 30s of life; done.
    return vault_client


def export_vault_client_token(vault_client: Client) -> str:
    # take the vault client and return just the token as a string
    return vault_client.token


def _check_vault_token_ttl_ok(vault_client: Client) -> bool:
    # test its actually authenticated
    if vault_client.is_authenticated():
        current_token = vault_client.auth.token.lookup_self()
        return current_token["data"]["ttl"] > VAULT_TOKEN_MIN_TTL
    else:
        return False


def _check_vault_token_expired(vault_client: Client) -> bool:
    # test its actually authenticated
    if vault_client.is_authenticated():
        current_token = vault_client.auth.token.lookup_self()
        return current_token["data"]["ttl"] > 0
    else:
        return False


def _renew_vault_token(vault_client: Client) -> bool:
    # test its actually authenticated
    if vault_client.is_authenticated():
        vault_client.auth.token.renew_self()
        return True
    else:
        return False


def fetch_kv(secret_request: SecretRequest) -> SecretResponse:
    vault_client = quick_vault_auth_from_token(
        token=secret_request.model_dump()["vault_token"]
    )
    # get our secret block from the store
    secrets = vault_client.secrets.kv.v2.read_secret_version(
        mount_point=secret_request.model_dump()["secret_mount"],
        path=secret_request.model_dump()["secret_path"],
    )
    # check if our named secret is there or not
    if secrets["data"]["data"].get(secret_request.model_dump()["secret_name"]) is None:
        raise Exception(
            f"Failed to retrieve token {secret_request.model_dump()['secret_name']} from mount point {secret_request.model_dump()['secret_mount']} and path {secret_request.model_dump()['secret_path']}"
        )

    return SecretResponse(
        key=secret_request.model_dump()["secret_name"],
        value=secrets["data"]["data"][secret_request.model_dump()["secret_name"]],
    )
```

As you can see, there is a lot of utility to hiding the repetitve work away in a utility module, and then having clean, simple code that calls these _wherever you need them_ later down the line. This is the "Dont repeat yourself (DRY)" principle in action. 

Hopefully this was educational for you, and if you want more, hit me up on the socials somewhere and let me know what's good or not about it.

Until next time... toodeloo