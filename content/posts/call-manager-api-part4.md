+++
title = "Using CallManager APIs for fun and profit: Part 4 - Python End to End"
date = "2015-06-29T00:41:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["CallManager","Automation","API","Devops","Python",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
So by now you can use SoapUI and Python to read info out, and can utilise SoapUI Test Cases to validate, and deliver changes to the CallManager system.  Again, please do ensure you have reviewed parts 1-3 before entering into this post.  Its entirely contextualised within the series.

Up until now what we have been doing is quite focused on a single object being added or sense checked.  In this, the final post in this mini series we will be looking at an end to end python script that can be used to add a new end user to the system, with pre and post validation.

As you will see in the final script, we have had to jump forward significantly in our python engagement in order to deliver on the goal of mimicking the test case we defined in SoapUI. You will find use of programming constructs such as if statements, functions, arguments and return codes.  

This will potentially confuse people new to the programming world, so before we dive in too deep, I have decided to start the post off with something I call a "train of thought" script.

Whenever I start out on a new script, I tend to look at a problem and how I might solve it manually.  I then embark on a process of bashing that out into code pretty much as a single linear script.  This is both highly dangerous, and highly effective.  It gets you a working proof of concept, which is good. It also gets you two problems, where previously you would have had one!  The next steps from that, is to review the code fully, and look where you can pull stretches of processing out into functions. Once you then have the functions working in isolation using the python cli, you can then re-do your script, including endless comments and a tidy layout to support it.

Any developer will tell you this is a terrible idea.  I am not a developer.  This works for me - YMMV.

It may not have escaped your notice that there was a significant gap between this post and the last 3.  Part of this was Work commitments like Cisco Live US and the inevitable catchup of 2 weeks out of the office.  The other part was the not insignificant pain I had ham-fisting this script together.  I wanted to keep it simple, whilst also keeping it as close to the SoapUI work as possible.  Apologies for that delay...

## Coding as the Crow Flies

This for me, tends to be a process run in parallel.  I will have one screen running vim with the script itself.  I then have another screen running python cli, so that I can type/paste in commands one a time, and explore my options both input and output.  Anything that passes the cli gets added to the script in vim.  I then regularly insert print statements on return/object creation tasks in the python script and run it to ensure I have consistency.  Be mindful that occasionally you will have to head to the CUCM GUI to clean up changes made during iterative development!

So, we want to start with the same old top half that imports the modules, sets up the environment, then connects to the CUCM server.

Once we have a proxy object running and logged in, we have a new toy to play with - the factory.

## Generating Request objects via the suds factory

Back in SoapUI you will recall that we had a bunch of pre-canned requests under every single method, called "Request1".  That is where SoapUI read the wsdl, spotted there was some definitions for what a request can look like, and build you one ready to use.  The factory does exactly the same thing.  What we need to do is call the factory.create() method, using an argument of the request type we need.

```python
new_user_request = cucm_server.factory.create("ns0:AddUserReq")
```

This creates a new object called new_user_req containing all the elements allowed for the GetUser method.

We can then update that with the values we want to use, and then fire that to the same method we used in SoapUI.

This is the equivalent to the Request1 objects created in SoapUI...  You should see that we have all the same information in the Suds Factory object, as we have in the SoapUI requests.  Take a moment to have a look at that and bank the technique.  As you expand your work, you could find yourself without access to SoapUI, and this gives you a chance to ensure that you can enumerate the objects properly.

Anywhere, where were we...

We want to check the user doesn't already exist, add it, then check it added successfully.

## The script

...also available on github as a gist

```python
import suds
from suds.client import Client
import ssl
import logging
import sys


# ignore ssl pain - python 2.7.9+ only
ssl._create_default_https_context = ssl._create_unverified_context

# set suds logging to critical only
logging.getLogger('suds.client').setLevel(logging.CRITICAL)

# if we need to see the soap envelopes for debugging, we can uncomment the below...
#handler = logging.StreamHandler(sys.stderr)
#logger = logging.getLogger('suds.transport.http')
#logger.setLevel(logging.DEBUG), handler.setLevel(logging.DEBUG)
#logger.addHandler(handler)

# define a function to convert the suds requests to a dict for return to AXL 
def sudsToDict(data):
    return dict([(str(key),val) for key,val in data])

# setup creds for auth
username = 'admin'
passwd = 'well_good_pass()rd'

# setup access to the wdsl and the pub
# 
# NOTE: this has localhost:8000 for the wsdl.  
# the wsdl isnt available on the server (why i will never know) 
# 
# two choices:
# 1) host the wsdl online somewhere you know, and refer to that
# 2) in another terminal browse to the AXLToolkit/ folder 
#    run 'python -m SimpleHTTPServer' in that dir.  
#    that spins up a mini web server you can read the wsdl from...
wsdl_url = 'http://localhost:8000/schema/10.5/AXLAPI.wsdl'
service_url = 'https://10.31.74.30/axl/'

# Create an object to communicate with the API called cucm_server.  
# this is how we pull and push data into CUCM via XML.

cucm_server = Client(wsdl_url, location = service_url, username=username, password=passwd)

# read some data in.
#
print "Checking for user: "
try:
        user_check = cucm_server.service.listUser({'userid':'New_End_User'},{'firstName':'','lastName':'','userid':'','mailid':'','department':'','userLocale':''})

except suds.WebFault, err:
        print err
        exit()

# Check to see if our request was an error or a success...
if len(user_check[0]) > 0:
        print "User already exists!"
        exit()
else:
        print "User not found... Continue..."


# we only get here when we had a success...
#
# build a new request object with factory

print "Creating user request"
new_user_request = cucm_server.factory.create("ns0:AddUserReq")


# set the username that we need...

print "Updating User request"
new_user_request.user.userid = "New_End_User"
new_user_request.user.firstName = "API Generated"
new_user_request.user.lastName = "End User"
new_user_request.user.department = 'API'
new_user_request.user.password = 'cisco'
new_user_request.user.pin = '08520'
new_user_request.user.mailid = 'meh@meh.com'


#delete the ones we dont need...
del new_user_request.user.authenticationType
del new_user_request.user.accountType
del new_user_request.user.convertUserAccount
del new_user_request.user.ipccExtension
del new_user_request.user.nameDialing
del new_user_request.user.userIdentity
del new_user_request.user.ldapDirectoryName
del new_user_request.user.calendarPresence
del new_user_request.user.userProfile
del new_user_request.user.selfService
del new_user_request.user.extensionsInfo
del new_user_request.user.pagerNumber
del new_user_request.user.homeNumber
del new_user_request.user.mobileNumber
del new_user_request.user.title
del new_user_request.user.telephoneNumber
del new_user_request.user.directoryUri
del new_user_request.user.lineAppearanceAssociationForPresences
del new_user_request.user.serviceProfile
del new_user_request.user.imAndPresenceEnable
del new_user_request.user.homeCluster
del new_user_request.user.customUserFields
del new_user_request.user.mlppPassword
del new_user_request.user.numericUserId
del new_user_request.user.patternPrecedence
del new_user_request.user.ctiControlledDeviceProfiles
del new_user_request.user.enableEmcc
del new_user_request.user.primaryDevice
del new_user_request.user.pinCredentials
del new_user_request.user.passwordCredentials
del new_user_request.user.remoteDestinationLimit
del new_user_request.user.maxDeskPickupWaitTime
del new_user_request.user.enableMobileVoiceAccess
del new_user_request.user.enableMobility
del new_user_request.user.subscribeCallingSearchSpaceName
del new_user_request.user.presenceGroupName
del new_user_request.user.defaultProfile
del new_user_request.user.phoneProfiles
del new_user_request.user.digestCredentials
del new_user_request.user.enableCti
del new_user_request.user.associatedGroups
del new_user_request.user.associatedPc
del new_user_request.user.primaryExtension
del new_user_request.user.associatedDevices
del new_user_request.user.userLocale
del new_user_request.user.manager

# convert that to a dict
new_user_req_dict = sudsToDict(new_user_request)

# push that user object back into call manager with addUser method
print "submitting new user request"
try:
        add_new_user_results = cucm_server.service.addUser(**new_user_req_dict)

except suds.Webfault, err:
        print err

        exit()

# we only get where when we had success
#
print "pulling new user data back..."
try:
        user_check2 = cucm_server.service.listUser({'userid':'New_End_User'},{'firstName':'','lastName':'','userid':'','mailid':'','department':'','userLocale':''})

except suds.WebFault, err:
        print err

        exit()

# Check to see if our request was an error or a success...
if len(user_check2[0]) > 0:
        print "User created successfully!"
        print user_check2[0]

else:
        print "FAIL WHALE!!!"
        print user_check2[0]
        exit()
```

WOAH NELLY!!!! Whats all that noise!?!

Lets pick that apart section by section...

I'll ignore the top bit, just note that we added "import sys" to allow us to use sys.exit() to easily bomb out of the script.

Next, in order to keep noise to a minimum, we set the logging for the suds client to critical items only.  This ensures we only get stuff on the screen when we really need it.  We also have commented options for enabling DEBUG level logging, which throws up the exact XML in and out of the service.  This can be particularly handy when you need more details as to what is going wrong.

```python
# set suds logging to critical only
logging.getLogger('suds.client').setLevel(logging.CRITICAL)

# if we need to see the soap envelopes for debugging, we can uncomment the below...
#handler = logging.StreamHandler(sys.stderr)
#logger = logging.getLogger('suds.transport.http')
#logger.setLevel(logging.DEBUG), handler.setLevel(logging.DEBUG)
#logger.addHandler(handler)
```

Now, we have something new.  Due to the way the suds service handled the submission of generated objects, we have to be a bit creative about how we post our stuff back to CUCM. Some conversations on the internet say its all suds fault, others argue that its the AXL service definition. Tit for tat is worthless for me, I need solutions.  what this little function does, is convert the suds object into a python style dict (dictionary) collection. Essentially each name value pair is extracted and put into the 'name':'value' notation, which we can then submit as an argument to the get/list/update/remove functions.

```python
# define a function to convert the suds requests to a dict for return to AXL 
def sudsToDict(data):
    return dict([(str(key),val) for key,val in data])
```

I think this is probably where I lost the most time in this process. I was pushing back the full object, and getting errors about the returnedTags field in my response.  Having gone though the debug process i could clearly see that my entire request was being stuffed into he returned tags section of the XML, with none of the required search/select information.  Try it yourself if you like.  Its annoying as hell.  This method works now, so I urge you to just use that instead.

Creds are obvious.  Again, this is not Prod code so meh, but if you go there, pull this out to a prompt yeah?

I also think we have done enough on the whole setup access to the service and then connect in part 2.

Now, this is the first extension of Python... The try/except operations. What this essentially does is tell python to execute a block of code, and if an error occurs, rather than just shout and die, to take the error, and do something intelligent with it, without exiting explicitly.

```python
# read some data in.
#
print "Checking for user: "
try:
        user_check = cucm_server.service.listUser({'userid':'New_End_User'},{'firstName':'','lastName':'','userid':'','mailid':'','department':'','userLocale':''})

except suds.WebFault, err:
        print err
        exit()
```

My try/except here attempts to run that same block of code we had in part 2.  Lookup a user with id 'New_End_User', and return a specific set of fields in the reply.

In the event it completes that successfully, it just carries on to the next section of code after the indented stuff in the except loop.  It skips right past that section basically.  If however, that fails, the except is configured to capture the contents of the suds.WebFault output, stores it in an object called err, and then runs the code in the indented block below.  My code prints that error out, then exits. Not much real benefit over the default operation of python I guess, but I really wanted to show that rather critical feature off.  Its stock best practice, and you will certainly need it in the future.

Now then, if the code is still running at this point, then the call worked. In the SoapUI test case, we checked the system returned data. SO we need to check our returned info and if its got data in cry foul and quit, or otherwise carry on, since we know its not already there.  In programming, this is called if/else.  There are many extensions of this and alternatives such as case. Think of it conversationally, if this is true, then do this, otherwise do something else.  Note that you are welcome to just to an if this then do this, without an else.  I prefer to keep it obvious, and an else in the code from the start gives you obvious options.

```python
# Check to see if our request was an error or a success...
if len(user_check[0]) > 0:
        print "User already exists!"
        exit()
else:
        print "User not found... Continue..."
```

So we use a built in function in Python to check the character length of the returned info.  If nothing was found, then nothing will be in the return: len(user_check[0]) = 0

However, if len(user_check) is greater than 0, we must have had data back, meaning our search matched an existing user.  Therefore, in our true section (the top bit of indented code), we say the user already exists, and we quit the script.  If we don't get anything back, that is skipped entirely, and we end up in the else section, which tells us we are good to go, and then script continues on as normal.

So, before we go back to CUCM and make the user, we should try and follow some standards when creating the new object.  Lets use the factory to build a fully specified user object, set some values, then trim out the things we are not setting, before pushing it back in.  The next section is HUGE. This is because the User object in CUCM is unsurprisingly complex. I did look at just pushing the whole thing back in, but there are nested sections within the factory generated object, that if named, MUST have values set for them.  I didn't want that, and I didn't need that.  Do, i use the bulletin del operator to strip down the completed object.  This is UHHHHHGLY.  I will latterly build a clever function to prune the object prior to return.  For now, this is it.

```python
# build a new request object with factory

print "Creating user request"
new_user_request = cucm_server.factory.create("ns0:AddUserReq")

# set the username that we need...

print "Updating User request"
new_user_request.user.userid = "New_End_User"
new_user_request.user.firstName = "API Generated"
new_user_request.user.lastName = "End User"
new_user_request.user.department = 'API'
new_user_request.user.password = 'cisco'
new_user_request.user.pin = '08520'
new_user_request.user.mailid = 'meh@meh.com'

#delete the ones we dont need...
del new_user_request.user.authenticationType
del new_user_request.user.accountType
del new_user_request.user.convertUserAccount
del new_user_request.user.ipccExtension
del new_user_request.user.nameDialing
del new_user_request.user.userIdentity
del new_user_request.user.ldapDirectoryName
del new_user_request.user.calendarPresence
del new_user_request.user.userProfile
del new_user_request.user.selfService
del new_user_request.user.extensionsInfo
del new_user_request.user.pagerNumber
del new_user_request.user.homeNumber
del new_user_request.user.mobileNumber
del new_user_request.user.title
del new_user_request.user.telephoneNumber
del new_user_request.user.directoryUri
del new_user_request.user.lineAppearanceAssociationForPresences
del new_user_request.user.serviceProfile
del new_user_request.user.imAndPresenceEnable
del new_user_request.user.homeCluster
del new_user_request.user.customUserFields
del new_user_request.user.mlppPassword
del new_user_request.user.numericUserId
del new_user_request.user.patternPrecedence
del new_user_request.user.ctiControlledDeviceProfiles
del new_user_request.user.enableEmcc
del new_user_request.user.primaryDevice
del new_user_request.user.pinCredentials
del new_user_request.user.passwordCredentials
del new_user_request.user.remoteDestinationLimit
del new_user_request.user.maxDeskPickupWaitTime
del new_user_request.user.enableMobileVoiceAccess
del new_user_request.user.enableMobility
del new_user_request.user.subscribeCallingSearchSpaceName
del new_user_request.user.presenceGroupName
del new_user_request.user.defaultProfile
del new_user_request.user.phoneProfiles
del new_user_request.user.digestCredentials
del new_user_request.user.enableCti
del new_user_request.user.associatedGroups
del new_user_request.user.associatedPc
del new_user_request.user.primaryExtension
del new_user_request.user.associatedDevices
del new_user_request.user.userLocale
del new_user_request.user.manager
```

If you are following along on the CLI (you should be), you can now do a cheeky little print new_user_request to see the final object that we want to return to CUCM to add the new user.  As you go on to make your own toys, you will soon learn which ones you want and which ones you don't.  Consider that this is built against the 10.5 schema too.  You will almost certainly have to faff about with the things you prune and require across versions.

Now we have an object, what we need to do is convert that to a dict for return to AXL. I cheat and just extent the object name with an _dict.  It helps when debugging and makes things clearer to third parties reading my code (hopefully!)

```python
# convert that to a dict
new_user_req_dict = sudsToDict(new_user_request)
```

Again, if you're following along, try a print(new_user_req_dict) and see what you get.

We are now ready to send this back into CUCM.  As before, we will use a try/except loop to ensure that all our external communication has error handling (like for example the connection or server goes down - its just polite :D)

```python
# push that user object back into call manager with addUser method
print "submitting new user request"
try:
        add_new_user_results = cucm_server.service.addUser(**new_user_req_dict)

except suds.Webfault, err:
        print err
        exit()
```

You'll notice that I used `.addUser(**new_user_req_dict)` to refer to the object we created.  The `**object` notation means dump out each item in the object as an argument one at a time. Look at the output of the `print(new_user_req_dict)`, look at the listUser arguments from earlier and try `print(**new_user_req_dict)`.  You'll see that basically we used the `**object` notation to save us the effort of typing out the details manually, like we had to on the listUser args.  TIME SAVED ACHIEVEMENT COMPLETE!

If this works, then the add_new_user_results should contain a GUID, which is the new user unique id.  if it fails, we get an error and the script exits. We could do some fancy work with the return data, but at this point, we are on a roll.  Lets press on with the next check - to ensure the user was added correctly.

So essentially, we are repeating the lookup from earlier, only this time, we want to get a confirmed result, rather then an empty one.

```python
# we only get where when we had success
#
print "pulling new user data back..."
try:
        user_check2 = cucm_server.service.listUser({'userid':'New_End_User'},{'firstName':'','lastName':'','userid':'','mailid':'','department':'','userLocale':''})

except suds.WebFault, err:
        print err
        exit()

# Check to see if our request was an error or a success...
if len(user_check2[0]) > 0:
        print "User created successfully!"
        print user_check2[0]

else:
        print "FAIL WHALE!!!"
        print user_check2[0]
        exit()
```

Same MO - try/except each external interaction, and all we did was make the true output (the first indented section, report something successful, and the else section (i.e. we can't find the user) report FAIL.

## ONE UGLY ASS SCRIPT

Coding as the crow flies is a great way to learn you way around a language and an interface. For me, this script works really well, but its HIDEOUS.  it has limited error handling, only validates the least amount of detail, and again, only in the function of the change, not the detail.  It does however, fit the brief or replicating our SoapUI functional tests.  

When working your way through this, your milage will undoubtedly vary depending on the Schema versions you are working on, and the system you have access to.  I recommend spinning up a 60 day trial using your Cisco supplied media under your support contract. If you have neither, then consider playing with the Sandbox labs here: https://developer.cisco.com/site/devnet/sandbox/ - there is a Collaboration 10.5 lab there, where you get a CUCM Pub/Sub, and a CUPS box that you can reserve and play with for 60 days I think.

With a little feedback from colleagues I have will take this onto the next step and start extracting and extending the functionality into a more audacious script - for full automation of our Leavers and Starters. Soon enough i will post a bonus part 5 that expands on how i extracted repeated operations into new functions, as well as prompting the end user for valued info like the server location and the creds, alongside the obvious new user details.

Watch this space.
