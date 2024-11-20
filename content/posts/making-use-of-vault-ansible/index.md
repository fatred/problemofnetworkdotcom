---
title: "Making Use of Vault: Ansible"
date: 2024-11-19T18:00:00+01:00
author: John Howard
tags: ["vault", "netdevops", "ansible", "python"]
showFullContent: false
---
# WIP!

As we come towards the end of this mini series, we talked about how to [bootstrap](/posts/bootstraping-hashi-vault/) a hashicorp vault for non-prod use, what [primitives](/posts/hashi-vault-primitives/) vault uses for secrets management, and how to talk to vault from [python](/posts/making-use-of-vault-python/). 

Here we will dig into how you can access vault content within an Ansible workflow, ensuring you never more have the pain of managing secrets with `ansible-vault`, or worse, storing them plain text in a repo somewhere.

## Vault in Ansible

Main Options: 
* HTTPS API via uri (but ... don't do this to yourself)
* Community "Hashi Vault" collection: [link](https://github.com/ansible-collections/community.hashi_vault)

### A little bootstrapping

To be able to get this to work I will stand up a small repo in the same `~/vault` path. Be sure to activate your venv as per the [python](/posts/making-use-of-vault-python/) post.

Lets install ansible and the collection

```
pip3 install git+https://github.com/ansible/ansible@stable-2.15
ansible-galaxy collection install community.hashi_vault
```

Now lets create a baseline setup that we can expand as we go.

`mkdir -p ~/vault/ansible/{group_vars,host_vars,inventory,tasks}`

lets create an inventory file in yaml for localhost

`inventory/all.yaml`:
```yaml
---
all:
  hosts:
    localhost:
```

we can quickly check this with `ansible-inventory` like so:

```
~ ansible-inventory -i inventory --list
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    },
    "ungrouped": {
        "hosts": [
            "localhost"
        ]
    }
}
```

In order to get to vault, we know we need some variables set. We can use env vars, but lets be explicit so we can use them in the HTTPS API case as well. These were based on the docs for the collection found [here](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/vault_kv2_get_lookup.html)

`group_vars/all.yml`:

```yaml
---
# vault settings
ansible_hashi_vault_addr: http://localhost:8200
ansible_hashi_vault_auth_method: token
ansible_hashi_vault_engine_mount_point: secret
# update this to the initial root token if you are using docker
ansible_hashi_vault_token: root 
ansible_hashi_vault_token_validate: true
```

finally lets create a simple playbook that we can expand as well.

`test-uri.yml`:
```yaml
---
- name: Run URI Tasks
  hosts: all
  gather_facts: no
  connection: local
  tasks:
    - name: include the uri_workflow tasks
      ansible.builtin.include_tasks:
        file: tasks/uri_workflow.yml
```


### HTTPS API use

> I will hit this extremely briefly, because I think its a terrible idea. There is ONE case where this makes sense - you have to get one small little thing out and stick it in a variable - thats it. Anythind more complicated you should do it with the collection "properly".

Because I am quite lazy, I will reuse the same examples from the primitives and python posts. There is some consistency there that helps I am sure.

Lets create a few tasks for the `uri` workflow:

`tasks/uri_workflow.yml`:
```yaml
- name: "Push secret into Vault via HTTP API"
  ansible.builtin.uri:
    url: '{{ ansible_hashi_vault_addr }}/v1/{{ ansible_hashi_vault_engine_mount_point }}/data/mycomplicatedsecret'
    method: POST
    body_format: json
    body:
      data:
        wireguard:
          keys:
            privkey: "privkey1"
            pubkey: "pubkey2"
          ip:
            outer: "100.64.0.0/30"
            inner: "100.64.255.0/31"
      options: {}
    headers:
      X-Vault-Request: 'true'
      X-Vault-Token: "{{ ansible_hashi_vault_token }}"
      Content-Type: application/json
  register: uri_put_result

- name: "show us what happened"
  debug:
    msg: "{{ uri_put_result }}"
```

which results in this: 

```
(venv) ➜  ansible ansible-playbook -i inventory test-uri.yml

PLAY [Run URI Tasks] ********************************************************************************************************************************************************************

TASK [include the uri_workflow tasks] ***************************************************************************************************************************************************
included: /Users/jhow/vault/ansible/tasks/uri_workflow.yml for localhost

TASK [Push secret into Vault via HTTP API] **********************************************************************************************************************************************
ok: [localhost]

TASK [show us what happened] ************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "ansible_facts": {
            "discovered_interpreter_python": "/Users/jhow/vault/venv/bin/python3.11"
        },
        "cache_control": "no-store",
        "changed": false,
        "connection": "close",
        "content_length": "294",
        "content_type": "application/json",
        "cookies": {},
        "cookies_string": "",
        "date": "Wed, 20 Nov 2024 14:57:31 GMT",
        "elapsed": 0,
        "failed": false,
        "json": {
            "auth": null,
            "data": {
                "created_time": "2024-11-20T14:57:31.127539Z",
                "custom_metadata": null,
                "deletion_time": "",
                "destroyed": false,
                "version": 3
            },
            "lease_duration": 0,
            "lease_id": "",
            "mount_type": "kv",
            "renewable": false,
            "request_id": "acea468b-9b11-2aa5-fb95-67e9f74be084",
            "warnings": null,
            "wrap_info": null
        },
        "msg": "OK (294 bytes)",
        "redirected": false,
        "status": 200,
        "strict_transport_security": "max-age=31536000; includeSubDomains",
        "url": "http://localhost:8200/v1/secret/data/mycomplicatedsecret",
        "warnings": [
            "Platform darwin on host localhost is using the discovered Python interpreter at /Users/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.15/reference_appendices/interpreter_discovery.html for more information."
        ]
    }
}

PLAY RECAP ******************************************************************************************************************************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Great! we pushed something into vault. Lets extend this to include fetching stuff from vault into a fact (variable)

add this to the bottom of `tasks/uri_workflow.yml`:
```yaml
- name: "fetch secret from vault"
  ansible.builtin.uri:
    url: '{{ ansible_hashi_vault_addr }}/v1/{{ ansible_hashi_vault_engine_mount_point }}/data/mycomplicatedsecret'
    method: GET
    return_content: yes
    headers:
      X-Vault-Request: 'true'
      X-Vault-Token: "{{ ansible_hashi_vault_token }}"
      Content-Type: application/json
  register: uri_get_result

- name: "show us what happened"
  debug:
    msg: "{{ uri_get_result }}"

- name: assign _just_ the secret content from the response to a fact (var)
  set_fact:
    mysecret: "{{ uri_get_result.json.data.data }}"

- name: show the secret content
  debug:
    msg: "{{ mysecret }}"
```


which when we run again (skipping the bits we already saw above):

```
TASK [fetch secret from vault] **********************************************************************************************************************************************************
ok: [localhost]

TASK [show us what happened] ************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "cache_control": "no-store",
        "changed": false,
        "connection": "close",
        "content": "{\"request_id\":\"724dbeb3-680d-54e0-d760-8a2b34c29641\",\"lease_id\":\"\",\"renewable\":false,\"lease_duration\":0,\"data\":{\"data\":{\"wireguard\":{\"ip\":{\"inner\":\"100.64.255.0/31\",\"outer\":\"100.64.0.0/30\"},\"keys\":{\"privkey\":\"privkey1\",\"pubkey\":\"pubkey2\"}}},\"metadata\":{\"created_time\":\"2024-11-20T15:05:05.452399Z\",\"custom_metadata\":null,\"deletion_time\":\"\",\"destroyed\":false,\"version\":5}},\"wrap_info\":null,\"warnings\":null,\"auth\":null,\"mount_type\":\"kv\"}\n",
        "content_length": "436",
        "content_type": "application/json",
        "cookies": {},
        "cookies_string": "",
        "date": "Wed, 20 Nov 2024 15:05:05 GMT",
        "elapsed": 0,
        "failed": false,
        "json": {
            "auth": null,
            "data": {
                "data": {
                    "wireguard": {
                        "ip": {
                            "inner": "100.64.255.0/31",
                            "outer": "100.64.0.0/30"
                        },
                        "keys": {
                            "privkey": "privkey1",
                            "pubkey": "pubkey2"
                        }
                    }
                },
                "metadata": {
                    "created_time": "2024-11-20T15:05:05.452399Z",
                    "custom_metadata": null,
                    "deletion_time": "",
                    "destroyed": false,
                    "version": 5
                }
            },
            "lease_duration": 0,
            "lease_id": "",
            "mount_type": "kv",
            "renewable": false,
            "request_id": "724dbeb3-680d-54e0-d760-8a2b34c29641",
            "warnings": null,
            "wrap_info": null
        },
        "msg": "OK (436 bytes)",
        "redirected": false,
        "status": 200,
        "strict_transport_security": "max-age=31536000; includeSubDomains",
        "url": "http://localhost:8200/v1/secret/data/mycomplicatedsecret"
    }
}

TASK [assign _just_ the secret content from the response to a fact (var)] **********************************************************************************************************************************************
ok: [localhost]

TASK [show the secret content] *************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "wireguard": {
            "ip": {
                "inner": "100.64.255.0/31",
                "outer": "100.64.0.0/30"
            },
            "keys": {
                "privkey": "privkey1",
                "pubkey": "pubkey2"
            }
        }
    }
}

PLAY RECAP ******************************************************************************************************************************************************************************
localhost                  : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

So there is the dumb way to push things in and pull things out of vault in ansible.

Lets dig into the "right" way to do it with the collection...

### The hashi-vault collection

