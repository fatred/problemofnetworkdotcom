+++
title = "Doing YANG Wrong: Part 4 - Config stores"
date = "2020-07-01T13:57:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = []
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2020/07/doing-yang-wrong-part-4-config-stores.html']
+++
## Part 4: Config stores

As we head down this rabbit hole, we start to get ever closer to something useful, but ever more deep into the weeds of NETCONF/YANG. For the uninitiated, config stores are a place where you can put config chunks either singularly, or in aggregate over a series of netconf pushes, to generate a new config that you will apply in one hit. In the SP world you might want to setup some interfaces, some BGP and then some overlay like an MPLS service. You may not want to do all this in one script since you might not need all of those things in one script. better to make a script per thing, and then call the ones your CMDB thinks it needs, place them all in a candidate config, validate that as a whole and then push that in one go. If anything fails, you can then throw it all away and not touch the operational running config.  At this point juniper folks are shrugging their shoulders - this is very old hat to them, but for enterprise people, particularly the type who lived on the CLI for years, this is unimaginable.

There be dragons here still however. There is invariably only one candidate config store per device, and there is always the chance that someone did something and their script didnt clear out in error handling, or the chance that some other script or person hops onboard whilst your session is in progress. Either way, you could end up with a poisoned candidate and a mess.

To approach this risk, there are two things to help treat this. First, is config locking. When you start a session it's a good idea to lock the target datastore (candidate), and the indented datastore (running). This prevents any out of band changes to the running place, and no inflight changes from other agents on the candidate config. All scripts should have error handling in from the start to discard config and release locks.

So. Lets tickle our script a bit. We need to add support for the config candidates. All this is handled in our function send_to_device()

```python
def send_to_device(**kwargs):
    rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>'

    with manager.connect_ssh(host=kwargs['dev'], port=830, username=kwargs['user'], password=kwargs['password'], hostkey_verify=False) as m:
        # we must handle errors nicely so always work in a try loop
        try:
            assert(":candidate" in m.server_capabilities)
            # lock running manually
            m.lock(target='running')
            # use a magic context manager for locking candidate whilst we work. it unlocks it as we exit the with: frame
            with m.locked(target='candidate'):
                m.discard_changes()
                m.edit_config(target='candidate', config=rpc_body)
                m.commit()
                print('Successfully configured IP on {}'.format(kwargs['dev']))
        # spit the error out
        except Exception as e:
            print('Failed to configure interface: {}'.format(e))
            m.discard_changes()

        # regardless of error or success, unlock running store.
        finally:
            m.unlock(target='running')
```

You'll see we added stuff into our try loop, added a discard_changes into our exception loop and a finally section to always unlock the running config as we said we would. Lets give it a spin.

```shell
    python3 ./models/interface/push_inside_if.py
    Failed to configure interface: 
```

Lame. The fact we have no errors here makes me think we have actually got raised by the Assert failing (Exception != AssertionError). Lets add a new handler for this.

```python
def send_to_device(**kwargs):
    rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>'
    with manager.connect_ssh(host=kwargs['dev'], port=830, username=kwargs['user'], password=kwargs['password'], hostkey_verify=False) as m:
        # we must handle errors nicely so always work in a try loop
        try:
            assert(":candidate" in m.server_capabilities)
            # lock running manually
            m.lock(target='running')
            # use a magic context manager for locking candidate whilst we work. it unlocks it as we exit the with: frame
            with m.locked(target='candidate'):
                m.discard_changes()
                m.edit_config(target='candidate', config=rpc_body)
                m.commit()
                print('Successfully configured IP on {}'.format(kwargs['dev']))
        # spit the error out
        except AssertionError:
            print('Go and enable candidate configs pls')
            m.discard_changes()
        except Exception as e:
            print('Failed to configure interface: {}'.format(e))
            m.discard_changes()
        # regardless of error or success, unlock running store.
        finally:
            m.unlock(target='running')
```

```shell
    python3 ./models/interface/push_inside_if.py
    Go and enable candidate configs pls
    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 20, in send_to_device
        assert(":candidate" in m.server_capabilities)
    AssertionError

    During handling of the above exception, another exception occurred:

    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 32, in send_to_device
        m.discard_changes()
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/manager.py", line 231, in execute
        return cls(self._session,
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/rpc.py", line 292, in __init__
        self._assert(cap)
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/rpc.py", line 367, in _assert
        raise MissingCapabilityError('Server does not support [%s]' % capability)
    ncclient.operations.errors.MissingCapabilityError: Server does not support [:candidate]

    During handling of the above exception, another exception occurred:

    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 64, in <module>
        send_to_device(dev=device_ip, user=username, password=password, py_obj=ocintmodel.interfaces)
      File "./models/interface/push_inside_if.py", line 38, in send_to_device
        m.unlock(target='running')
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/manager.py", line 231, in execute
        return cls(self._session,
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/lock.py", line 49, in request
        return self._request(node)
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/rpc.py", line 348, in _request
        raise self._reply.error
    ncclient.operations.rpc.RPCError: {'type': 'protocol', 'tag': 'access-denied', 'severity': 'error', 'info': None, 'path': None, 'message': None}
```

Puke.

As we dig over that, we can see we were right, and the AssertionError wasn't being picked up, and now it is, but then after that it barfs everywhere with additional exceptions.

Looks like the first one is the `m.discard_changes()` in the AssertionError handler and the second is the `m.unlock('running')` in the finally block.

The first one is sorta obvious, if you don't have support for candidates, you didn't even try to lock the config, cos that assert is first. We can delete that.

The second is in the finally: block which always runs, is failing because we don't support it still.

My first thought was to wrap the m.unlock in another try block, with an except: pass concept, but that's a hack that could hurt you later. What I did instead was two try blocks, one for the candidate check and one for the config changes. If we don't support candidates, we bomb out don't try to do any config.

```python
def send_to_device(**kwargs):
    rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>'
    with manager.connect_ssh(host=kwargs['dev'], port=830, username=kwargs['user'], password=kwargs['password'], hostkey_verify=False) as m:
        try:
            candidate_supported = True
            assert(":candidate" in m.server_capabilities)
        except AssertionError:
            print('Go and enable candidate configs pls')
            candidate_supported = False
        
        if candidate_supported:
            # we must handle errors nicely so always work in a try loop
            try:
                # lock running manually
                m.lock(target='running')
                # use a magic context manager for locking candidate whilst we work. it unlocks it as we exit the with: frame
                with m.locked(target='candidate'):
                    m.discard_changes()
                    m.edit_config(target='candidate', config=rpc_body)
                    m.commit()
                    print('Successfully configured IP on {}'.format(kwargs['dev']))
            # spit the error out
            except Exception as e:
                print('Failed to configure interface: {}'.format(e))
                m.discard_changes()
            # regardless of error or success, unlock running store.
            finally:
                m.unlock(target='running')
```

Try again:

```shell
    python3 ./models/interface/push_inside_if.py
    Go and enable candidate configs pls
```

Nice and clean. Now where were we? Ah yes. We need to actually enable the candidate datastore on the router...

```shell
    netconf-yang feature candidate-datastore
```

and try again...

```shell
    python3 ./models/interface/push_inside_if.py
    Failed to configure interface: the configuration database is locked by session 5 syncfromdaemon tcp (system from 127.0.0.1) on since 2020-07-01 10:19:28
       IOS-XE YANG Infrastructure
    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 31, in send_to_device
        with m.locked(target='candidate'):
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/lock.py", line 67, in __enter__
        Lock(self.session, self.device_handler, raise_mode=RaiseMode.ERRORS).request(self.target)
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/lock.py", line 35, in request
        return self._request(node)
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/rpc.py", line 348, in _request
        raise self._reply.error
    ncclient.operations.rpc.RPCError: the configuration database is locked by session 5 syncfromdaemon tcp (system from 127.0.0.1) on since 2020-07-01 10:19:28
       IOS-XE YANG Infrastructure

    During handling of the above exception, another exception occurred:

    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 68, in <module>
        send_to_device(dev=device_ip, user=username, password=password, py_obj=ocintmodel.interfaces)
      File "./models/interface/push_inside_if.py", line 39, in send_to_device
        m.discard_changes()
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/manager.py", line 231, in execute
        return cls(self._session,
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/edit.py", line 192, in request
        return self._request(new_ele("discard-changes"))
      File "/home/jhow/.local/lib/python3.8/site-packages/ncclient/operations/rpc.py", line 348, in _request
        raise self._reply.error
    ncclient.operations.rpc.RPCError: the configuration database is locked by session 5 syncfromdaemon tcp (system from 127.0.0.1) on since 2020-07-01 10:19:28
       IOS-XE YANG Infrastructure
```

Lol? One thing that could be happening is that the candidate config is a copy of running, and if we lock running, it cant make a copy of running for the candidate store? like a race condition if you will... Lets move the running lock into the with loop for locking candidate:

```python
def send_to_device(**kwargs):
    rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>'
    with manager.connect_ssh(host=kwargs['dev'], port=830, username=kwargs['user'], password=kwargs['password'], hostkey_verify=False) as m:
        try:
            candidate_supported = True
            assert(":candidate" in m.server_capabilities)
        except AssertionError:
            print('Go and enable candidate configs pls')
            candidate_supported = False
        if candidate_supported:
            # we must handle errors nicely so always work in a try loop
            try:
                # use a magic context manager for locking candidate whilst we work. it unlocks it as we exit the with: frame
                with m.locked(target='candidate'):
                    # lock running manually as well
                    m.lock(target='running')
                    # scrap any unknown pending changes in that buffer
                    m.discard_changes()
                    # apply our edits
                    m.edit_config(target='candidate', config=rpc_body)
                    # commit them
                    m.commit()
                    # tell us
                    print('Successfully configured IP on {}'.format(kwargs['dev']))
            # spit the error out
            except Exception as e:
                print('Failed to configure interface: {}'.format(e))
                m.discard_changes()
            # regardless of error or success, unlock running store.
            finally:
                m.unlock(target='running')
```

and try again:

```shell
    python3 ./models/interface/push_inside_if.py
    Failed to configure interface: illegal reference /oc-if:interfaces/interface[name='GigabitEthernet4']/subinterfaces/subinterface[index='0']/oc-ip:ipv4/addresses/address[ip='109.109.109.2']/ip
```

So we are back where we were at the end of the last attempt, and config candidates aren't helping us.

Where to next then? Google doesn't have a lot to be honest, but there is this forum post about an illegal reference in the OC BGP Model against the CSV1000v. Parsing the response from the Cisco guy a bit, I'm left with two paths to follow.

* Version incompatibility with my IOS-XE (16.9.5) and the OC model here.
* The missing Cisco IOS-XE deviations from the OC Interfaces model need to be here.

I will start with option 2, since its easy enough to proove, although I expect I might have to upgrade anyways. That said, it's better to fix this in situ than blindly upgrade with no visibility of other bugs I might hit outside of YANG...
