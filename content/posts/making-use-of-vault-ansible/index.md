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

# Vault in Ansible

Main Options: 
* HTTPS API via uri (but ... don't do this to yourself)
* Community "Hashi Vault" collection: [link](https://github.com/ansible-collections/community.hashi_vault)

## A little bootstrapping

To be able to get this to work I will stand up a small repo in the same `~/vault` path. Be sure to activate your venv as per the [python](/posts/making-use-of-vault-python/) post.

Lets install ansible and the collection

```
pip3 install git+https://github.com/ansible/ansible@stable-2.17
ansible-galaxy collection install ansible.utils    # required in the last example 
ansible-galaxy collection install community.hashi_vault   # required in the second set of tasks
pip3 install hvac    # needed to make the vault collection work!
pip3 install netaddr   # needed to make the utils collection work!
```

Now lets create a baseline setup that we can expand as we go.

`mkdir -p ~/vault/ansible/{group_vars,host_vars,inventory,tasks,templates}`

lets create an inventory file in yaml for localhost

`inventory/all.yaml`:
```yaml
---
all:
controller:
  hosts:
    localhost:
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

We can quickly check the inventory and the group_vars applied correctly with `ansible-inventory -i inventory --list` like so:

```
{
    "_meta": {
        "hostvars": {
            "localhost": {
                "ansible_hashi_vault_addr": "http://localhost:8200",
                "ansible_hashi_vault_auth_method": "token",
                "ansible_hashi_vault_engine_mount_point": "secret",
                "ansible_hashi_vault_token": "hvs.trdFUmCJ00eTm6FuWq3NrmBR",
                "ansible_hashi_vault_token_validate": true
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "controller"
        ]
    },
    "controller": {
        "hosts": [
            "localhost"
        ]
    }
}
```


finally lets create a simple playbook that we can expand as well.

`test-uri.yml`:
```yaml
---
- name: Run URI Tasks
  hosts: controller
  gather_facts: no
  connection: local
  tasks:
    - name: include the uri_workflow tasks
      ansible.builtin.include_tasks:
        file: tasks/uri_workflow.yml
```


## HTTPS API use

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

running `ansible-playbook -i inventory test-uri.yml` results in this: 

```
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
- name: "fetch secret from vault with the http api directly"
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

Why is it dumb? Well I guess it actually _isn't_ but for example; that janky pattern of uri, register, pivot, use is not fabulous is it. With the collection you can use a sort of filter expression to pull things in from vault and set_fact in one line. 

Lets dig into the "right" way to do it with the collection...

## The hashi-vault collection

The collection includes [a bunch](https://docs.ansible.com/ansible/devel/collections/community/hashi_vault/#plugin-index) of modules, filters and lookup plugins that simplify the process of getting things out of vault and into ansible. 

### Modules

When you peruse the list of Modules in the collection, it starts with a long list of vault_database ones. These are there to help application developers remove hardcoded credentials for database client authentication from their apps, and instead have them brokered via vault. The application gets a token, can talk to vault, get the credentials for that database, and then use them to log their client into the DB. This is brilliant, but not in our scope today. REad more from hashicorp on the feature [here](https://developer.hashicorp.com/vault/docs/secrets/databases).

We then see the kv modules, which we will return to in a moment.

Finally we see a few more generic modules for list/login/read/write, a token_create which we will skip, and finally a very interesting pki_generate_certificate, which we will certainly revisit in another post.

### Filter plugins

This is sort of meh. It allows you to export a token quickly from a session that we created with our global vars. I won't be covering that.

### Lookup plugins

This is probably the most useful set of objects, since it allows up to pull vault content directly into facts (vars).

For the core of this post we will focus on the `hashi_vault` lookup, which is what I typically use to pull vars to ansible. I can't really remember if all these other plugins existed when I started, and I expect this one is only there for backwards compatibility, but lets keep it simple!

### Our simple example

Copying the same activities we had before, lets fetch the `mycomplicatedsecret` from vault using the filter plugin, and assign it directly to a fact, for use elsewhere.

Make a new file in the playbook called `tasks/module_workflow.yml`:

```yaml
- name: "fetch secret from vault with the module"
  ansible.builtin.set_fact:
    module_get_result: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/mycomplicatedsecret') }}"

- name: "show us what happened"
  debug:
    msg: "{{ module_get_result }}"
```

now lets fork the uri playbook and pick this task instead:

```bash
cp test-uri.yml test-module.yml
```

update that file to look like this: 

```yaml
- name: Run Module Tasks
  hosts: controller
  gather_facts: no
  connection: local
  tasks:
    - name: include the module_workflow tasks
      ansible.builtin.include_tasks:
        file: tasks/module_workflow.yml
```

Which when run with `ansible-playbook -i inventory test-module.yml` results in this: 

```

PLAY [Run Module Tasks] ****************************************************************************************************************************************************************************************

TASK [include the module_workflow tasks] ***********************************************************************************************************************************************************************
included: /home/jhow/vault/ansible/tasks/module_workflow.yml for localhost

TASK [fetch secret from vault with the module] *****************************************************************************************************************************************************************
ok: [localhost]

TASK [show us what happened] ***********************************************************************************************************************************************************************************
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

PLAY RECAP *****************************************************************************************************************************************************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Thats a heck of a lot easier isn't it. ;)

## Migrate a secret from inventory variables to vault

Lets assume for a second, you have some variables set into the inventory, and you really dont want to leave them there. Here is a simple example of how you could shift a pair of wireguard keys out of host vars and into vault, WITHOUT editing your playbook.

### bootstrap

Here are some files that we need to make the ugly way "work". To prevent the need to create some target VMs, we will use a local path on the ansible host for the outputs of our templates. 

> This is not a method people use in the real world, so don't be weirded out by it. This is a learning exercise, so DON'T straight up copy the code. apply the relevant parts to your use case ;)

Create the two hosts in your inventory under a new group called `wireguard`:

`vi inventory/all.yaml`

```yaml
---
all:
controller:
  hosts:
    localhost:
wireguard:
  hosts:
    wireguard1:
    wireguard2:
```

Test the inventory works again with the `ansible-inventory -i inventory --list` command:

> note that you can add the group name at the end of the command to graph the hosts/vars that group if you prefer... `ansible-inventory -i inventory --graph --vars wireguard`

```json
{
    "_meta": {
        "hostvars": {
            "localhost": {
                "ansible_hashi_vault_addr": "http://localhost:8200",
                "ansible_hashi_vault_auth_method": "token",
                "ansible_hashi_vault_engine_mount_point": "secret",
                "ansible_hashi_vault_token": "hvs.trdFUmCJ00eTm6FuWq3NrmBR",
                "ansible_hashi_vault_token_validate": true
            },
            "wireguard1": {
                "ansible_hashi_vault_addr": "http://localhost:8200",
                "ansible_hashi_vault_auth_method": "token",
                "ansible_hashi_vault_engine_mount_point": "secret",
                "ansible_hashi_vault_token": "hvs.trdFUmCJ00eTm6FuWq3NrmBR",
                "ansible_hashi_vault_token_validate": true
            },
            "wireguard2": {
                "ansible_hashi_vault_addr": "http://localhost:8200",
                "ansible_hashi_vault_auth_method": "token",
                "ansible_hashi_vault_engine_mount_point": "secret",
                "ansible_hashi_vault_token": "hvs.trdFUmCJ00eTm6FuWq3NrmBR",
                "ansible_hashi_vault_token_validate": true
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "controller",
            "wireguard"
        ]
    },
    "controller": {
        "hosts": [
            "localhost"
        ]
    },
    "wireguard": {
        "hosts": [
            "wireguard1",
            "wireguard2"
        ]
    }
}
```

now lets add some public/private keys into the host_vars for the two new hosts:

`vi host_vars/wireguard1.yml`
```yaml
---
ipam:
  inner: "100.64.255.1/30"
  outer: "100.64.1.1"
keychain:
  privkey: "wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM="
  pubkey: "EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4="
peer: wireguard2
```

`vi host_vars/wireguard2.yml`
```yaml
---
ipam:
  inner: "100.64.255.2/30"
  outer: "100.64.2.1"
keychain:
  privkey: "6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0="
  pubkey: "qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do="
peer: wireguard1
```

again lets test the vars work (using the graph this time) `ansible-inventory -i inventory --graph --vars wireguard`:

```
@wireguard:
  |--wireguard1
  |  |--{ansible_hashi_vault_addr = http://localhost:8200}
  |  |--{ansible_hashi_vault_auth_method = token}
  |  |--{ansible_hashi_vault_engine_mount_point = secret}
  |  |--{ansible_hashi_vault_token = hvs.trdFUmCJ00eTm6FuWq3NrmBR}
  |  |--{ansible_hashi_vault_token_validate = True}
  |  |--{ipam = {'inner': '100.64.255.1/30', 'outer': '100.64.1.1'}}
  |  |--{keychain = {'privkey': 'wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=', 'pubkey': 'EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4='}}
  |  |--{peer = wireguard2}
  |--wireguard2
  |  |--{ansible_hashi_vault_addr = http://localhost:8200}
  |  |--{ansible_hashi_vault_auth_method = token}
  |  |--{ansible_hashi_vault_engine_mount_point = secret}
  |  |--{ansible_hashi_vault_token = hvs.trdFUmCJ00eTm6FuWq3NrmBR}
  |  |--{ansible_hashi_vault_token_validate = True}
  |  |--{ipam = {'inner': '100.64.255.2/30', 'outer': '100.64.2.1'}}
  |  |--{keychain = {'privkey': '6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=', 'pubkey': 'qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do='}}
  |  |--{peer = wireguard1}
```

now lets make a jinja2 template that will render the wg0.conf file for each wireguard instance.

`vi templates/wg0.conf.j2`

```jinja2
[Interface]
Address = {{ ipam.inner }}
ListenPort = 51820
PrivateKey = {{ keychain.privkey }}

[Peer]
PublicKey = {{ hostvars[peer]['keychain']['pubkey'] }}
AllowedIPs = {{ hostvars[peer]['ipam']['outer'] -}}/32
```

now we need a task to render the template out to the localhost for testing purposes.

`vi tasks/render_wg_conf.yml`

```yaml
- name: create template files in /tmp
  template: 
    src: wg0.conf.j2
    dest: "/tmp/wg0-{{ inventory_hostname }}.conf"
  delegate_to: localhost
```

and lets make a playbook

`vi test-wireguard.yml`

```yaml
---
- name: Run Wireguard Tasks
  hosts: wireguard
  gather_facts: no
  connection: local
  tasks:
    - name: include the render_wg_conf tasks
      ansible.builtin.include_tasks:
        file: tasks/render_wg_conf.yml
```

and so when we run this with `ansible-playbook -i inventory test-wireguard.yml` we will see it spit out two files in /tmp:

```

PLAY [Run Wireguard Tasks] *************************************************************************************************************************************************************************************

TASK [include the render_wg_conf tasks] ************************************************************************************************************************************************************************
included: /home/jhow/vault/ansible/tasks/render_wg_conf.yml for wireguard1, wireguard2

TASK [create template files in /tmp] ***************************************************************************************************************************************************************************
[WARNING]: The value 'False' is not a valid IP address or network, passing this value to ipaddr filter might result in breaking change in future.
[WARNING]: Platform linux on host wireguard1 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [wireguard1 -> localhost]
[WARNING]: Platform linux on host wireguard2 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [wireguard2 -> localhost]

PLAY RECAP *****************************************************************************************************************************************************************************************************
wireguard1                 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
wireguard2                 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

which if we look at the content, we have some "valid" wireguard configs:

`cat /tmp/wg0-wireguard1.conf`:

```
[Interface]
Address = 100.64.255.1/30
ListenPort = 51820
PrivateKey = wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=

[Peer]
PublicKey = qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do=
AllowedIPs = 100.64.2.1/32
``` 

`cat /tmp/wg0-wireguard2.conf`: 
```
[Interface]
Address = 100.64.255.2/30
ListenPort = 51820
PrivateKey = 6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=

[Peer]
PublicKey = EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4=
AllowedIPs = 100.64.1.1/32
```

Great!

### moving that key material to vault

...without updating the playbook.

#### what is actually secret here?

If we look at the host and group vars, the only part that is genuinely secret is the keys. lets push the keys into vault via the cli first.

wireguard1: `vault kv put -mount="secret" "wireguard1" privkey="wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=" pubkey="EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4="`

wireguard2: `vault kv put -mount="secret" "wireguard2" privkey="6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=" pubkey="qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do="`

#### how to we set a secret _per host_?

the set_fact uses a per host run, so we can set the host var with a connection_local override.

add the following to the top of `tasks/render_wg_conf.yml`

```yaml
- name: get data from vault
  set_fact:
    keychain: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/{{ inventory_hostname }}') }}"

- name: debug
  debug:
    msg: "{{ keychain }}"
```

Then comment out the `keychain` in the `host_vars/wireguard1` and `host_vars/wireguard2`

`vi host_vars/wireguard1.yml`:

```yaml
---
ipam:
  inner: "100.64.255.1/30"
  outer: "100.64.1.1"
    #keychain:
    #  privkey: "wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM="
    #  pubkey: "EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4="
peer: "wireguard2"
```

`vi host_vars/wireguard2.yml`:

```yaml
---
ipam:
  inner: "100.64.255.2/30"
  outer: "100.64.2.1"
    #keychain:
    #  privkey: "6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0="
    #  pubkey: "qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do="
peer: "wireguard1"
```

if we now run the playbook, it _should_ say NOTHING needed updating (`ok` result, not `changed`):

`ansible-playbook -i inventory test-wireguard.yml`

```

PLAY [Run Wireguard Tasks] *************************************************************************************************************************************************************************************

TASK [include the render_wg_conf tasks] ************************************************************************************************************************************************************************
included: /home/jhow/vault/ansible/tasks/render_wg_conf.yml for wireguard1, wireguard2

TASK [get data from vault] *************************************************************************************************************************************************************************************
ok: [wireguard1]
ok: [wireguard2]

TASK [debug] ***************************************************************************************************************************************************************************************************
ok: [wireguard1] => {
    "msg": {
        "privkey": "wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=",
        "pubkey": "EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4="
    }
}
ok: [wireguard2] => {
    "msg": {
        "privkey": "6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=",
        "pubkey": "qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do="
    }
}

TASK [create template files in /tmp] ***************************************************************************************************************************************************************************
[WARNING]: Platform linux on host wireguard1 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
ok: [wireguard1 -> localhost]
[WARNING]: Platform linux on host wireguard2 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
ok: [wireguard2 -> localhost]

PLAY RECAP *****************************************************************************************************************************************************************************************************
wireguard1                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
wireguard2                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

and to be double sure, lets delete the config files, and the keys in the host vars entirely:

delete the outputs...
`rm /tmp/wg0-wireguard*`

remove the keys from the host_vars
`vi host_vars/wireguard1.yml`:
```yaml
---
ipam:
  inner: "100.64.255.1/30"
  outer: "100.64.1.1"
peer: "wireguard2"
```

`vi host_vars/wireguard2.yml`:

```yaml
---
ipam:
  inner: "100.64.255.2/30"
  outer: "100.64.2.1"
peer: "wireguard1"
```

run it again:
`ansible-playbook -i inventory test-wireguard.yml`

```
PLAY [Run Wireguard Tasks] *************************************************************************************************************************************************************************************

TASK [include the render_wg_conf tasks] ************************************************************************************************************************************************************************
included: /home/jhow/vault/ansible/tasks/render_wg_conf.yml for wireguard1, wireguard2

TASK [get data from vault] *************************************************************************************************************************************************************************************
ok: [wireguard1]
ok: [wireguard2]

TASK [debug] ***************************************************************************************************************************************************************************************************
ok: [wireguard1] => {
    "msg": {
        "privkey": "wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=",
        "pubkey": "EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4="
    }
}
ok: [wireguard2] => {
    "msg": {
        "privkey": "6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=",
        "pubkey": "qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do="
    }
}

TASK [create template files in /tmp] ***************************************************************************************************************************************************************************
[WARNING]: Platform linux on host wireguard1 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [wireguard1 -> localhost]
[WARNING]: Platform linux on host wireguard2 is using the discovered Python interpreter at /home/jhow/vault/venv/bin/python3.11, but future installation of another Python interpreter could change the meaning
of that path. See https://docs.ansible.com/ansible-core/2.17/reference_appendices/interpreter_discovery.html for more information.
changed: [wireguard2 -> localhost]

PLAY RECAP *****************************************************************************************************************************************************************************************************
wireguard1                 : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
wireguard2                 : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

and compare the content of the configs to the keys we got from vault

`cat /tmp/wg0-wireguard1.conf`:

```
[Interface]
Address = 100.64.255.1/30
ListenPort = 51820
PrivateKey = wJGwzAUgZiTL071BTd8yM9GTMEaPqa4rWVuU/BqnlnM=

[Peer]
PublicKey = qUzoU2XMB+W0EUn0vkUyUF0o7BUiGrQIdYSaokGr2Do=
AllowedIPs = 100.64.2.1/32
```

`cat /tmp/wg0-wireguard2.conf`:

```
[Interface]
Address = 100.64.255.2/30
ListenPort = 51820
PrivateKey = 6FDJPPefeCWkMEiHudfE2P3pnBai3bIPnfcBqG8BgF0=

[Peer]
PublicKey = EI2JFnpywLB7BhfyhQsePcGmmLQhYLVp2sdNZ5Lc8E4=
AllowedIPs = 100.64.1.1/32
```

## wrapping is all up

So eventually we get to the end of the demo. Hopefully you can see that vault is not actually that hard, and getting secrets out of your repo is a big deal, especially when it comes to making your git repo simpler to publish and to share. 

The only last thing I would recommend is that you do is _take the `ansible_hashi_vault_token` out of the repo_ and use the `VAULT_TOKEN` env var instead. You then have _no_ secret material in your repo.

In a follow up post I will look at some of the more production use cases. But, for now, thanks for joining me on this little adventure, and let me know if there are any changes you would like to see. 

Until next time! peace out. :D