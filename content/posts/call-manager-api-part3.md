+++
title = "Using CallManager APIs for fun and profit: Part 3 - Changes"
date = "2015-06-01T10:41:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["CallManager","Automation","API","Devops","Python"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2015/06/cucm-intro-to-apis-pt3.html']
+++
In this the third part of my mini series on using the CUCM API, we start to get further into the Ju-Ju, and check web app changes via the API, before then going on to make a change via the API itself.

As before if you haven't gone through the previous two posts then please, head back to the start and we will see you back here shortly.

## Whats your motivation today?

There are two reasons why you might want to use an API in your day to day changes.

1. Validating a change
2. Making a change

APIs and scripting are a great way to ensure that whatever it is you just did, worked as you expected. First we will look at using SoapUI to ensure that what we just did worked the way we wanted it to. As an aside - this is a very common use case for SoapUI Pro, and I would strongly recommend that you consider a full Pro Licence for this product.  I get no kickbacks, nor benefits from being an advocate of it.  I just think it does the job they intended, and it can be really good to be able to define a massive heap of tests, and run all of them with a single click.  It makes your change validation process vastly quicker, and your outcomes more reliable.  as we get to the end of this post, and the series I think you'll see what I mean.

## Validating a change in SoapUI

So, lets expand on where we were at the end of the last post.  Lets use a SoapUI "TestCase" to check for a user, add the user, and then check that the user is present.

This is stretching the terminology of a test case to its extremes, but it serves a purpose for us here.

Open up your SoapUI project, and navigate to the getUser method.  Right click this and add a new request called ProveUserNotPresent.

![SoapUI: Add new request on getUser method](/img/call-manager-api-part3/soapuiAddRequestProveUserNotPresent.PNG)

Update the userid in the search criteria to match the userid we will add in a moment, and trim the returnedTags to just a few key fields to prove we added what we said we would.

Update your authentication and make sure your endpoint uri is properly defined at the top.

![SoapUI: Update the fields to limit the scope of the return](/img/call-manager-api-part3/soapuiAddRequestProveUserNotPresent-XMLFields.PNG)

Now, run the request.  You should get an error back saying the end user does not exist.

![SoapUI: ProveUserNorPresent XML Response](/img/call-manager-api-part3/soapuiAddRequestProveUserNotPresent-response.PNG)

This is ideal.  The API tells us definitively, that this userid is not in use.  Lets add this request to a new SoapUI TestCase.

Right click the request in the tree, and select add to TestCase

![SoapUI: Add request to test case](/img/call-manager-api-part3/soapuiAddRequestProveUserNotPresentToTestCase.png)

Give the TestSuite a memorable name.  Here I use AddUserTestSuite for the Suite, and AddUserCheck for the TestCase.  Accept the defaults for adding the request to the test case.

![""](/img/call-manager-api-part3/soapuiAddNewTestSuite.PNG)

![""](/img/call-manager-api-part3/soapuiAddRequestToTestCase.PNG)

Now, typically, a test case should look for a positive result, and error out on a fault.  We in this particular case want to invert that.  We want an error to be positive.  If its not then we need to bomb out before trying to add something that is obviously already there...

Open the request from within the Case, by right clicking and selecting the open in editor.

![SoapUI: Edit the request in a test case](/img/call-manager-api-part3/soapuiEditRequest.png)

Click the tab at the bottom for the Assertions, to list all the things that need to be true to mark the case as a success.

![SoapUI: Review assertions against the request](/img/call-manager-api-part3/soapuiEditRequestAssertions.PNG)

Click the small icon above the assertions to add a new one.  Select SOAP Fault from the Compliance, Status and Standards section.  Then click add

![SoapUI: Add new assertion](/img/call-manager-api-part3/soapuiEditRequestAssertions-AddSoapFault.PNG)

This now inserts a new assertion that states a SOAP Fault is a valid response.  Ignore any unknown status in the assertion pane, since that's because we havent run before. Save and close the request pane.

![SoapUI: Updated assertions](/img/call-manager-api-part3/soapuiEditRequestAssertions-complete.PNG)

We can now run our test case and providing we don't have a use of that name defined, the test case will finish successfully.

![SoapUI: First run of test case successful](/img/call-manager-api-part3/soapuiTestCaseSuccess.PNG)

Great.  Lets now go an add that user in through the addUser method.

## Delivering a change via SoapUI

This is the fun part.  Again I remind you not to do this against a productive system. I will not be held responsible if you do something wrong here!

Just as before, right Click the addUser Method in the SoapUI tree, and select New Request. Name this AddUserMinimal (we are going to supply the least amount of info possible).

Set the username as per the account we validated in the past command, and add a few key fields.  Don't forget to check your request URI and the Auth!

![SoapUI: addUser request XML](/img/call-manager-api-part3/soapuiAddUserRequest.PNG)

Now for the paranoid out there, feel free to run this command through standard request process, and then go back and delete it via the web interface.  Here is what that looked like for me:

![SoapUI: addUser return from API - the guid for the new user](/img/call-manager-api-part3/soapuiAddUserResponse.PNG)

![CUCM GUI: New End User visible](/img/call-manager-api-part3/guiNewEndUserAppears.PNG)

In the GUI I then delete it do that we can run the test case end to end.

![CUCM GUI: New End User deleted](/img/call-manager-api-part3/guiNewEndUserDelete.PNG)

So we know our API call works.  Lets add that task to the Test Case.  Again, right click the AddUserMinimal Request and add it to the test case

![SoapUI: Insert AddUserMinimal Request to TestCase](/img/call-manager-api-part3/soapuiInsertAddUserMinimaltoTestCase.png)

Add it to our existing test case.

![""](/img/call-manager-api-part3/soapuiInsertAddUserMinimaltoTestCase-dialog.png)

Take the default settings again

![""](/img/call-manager-api-part3/soapuiInsertAddUserMinimaltoTestCase-settings.png)

Open the assertions tab and on top of the default validation we get a SOAP message back, add a check for "Not SOAP Fault"

![""](/img/call-manager-api-part3/soapuiInsertAddUserMinimaltoTestCase-assertions.png)

So now we have a check for a valid SOAP response, and that it wasn't a fault.  Again, ignore any unknown status - that sorts itself out once we run it for the first time.

Now before we run the case, we want to validate our user was added without checking the GUI.  Lets clone our existing ProveUserNotPresent request, and make it check that the user is present now...

![SoapUI: Clone](/img/call-manager-api-part3/soapuiCloneGetUserRequest.png)

Call it ProveUserPresent

![SoapUI: Rename Request for test case](/img/call-manager-api-part3/soapuiCloneGetUserRequest-naming.PNG)

Add the new request to the same test case, with the default settings

![SoapUI: Add request to test case](/img/call-manager-api-part3/soapuiAddProveUserPresentToTestCase.png)

This time we want to check for a valid response and that the response is not a fault.

![SoapUI: Update assertions](/img/call-manager-api-part3/soapuiProveUserPresentAssertions.PNG)

We can now run our test case end to end. Click the play button at the top of the test case

![SoapUI: Complete test case execution successful](/img/call-manager-api-part3/soapuiFullTestCaseSuccess.PNG)

Success!!!

If we head to the GUI we can prove that its there too...

![CUCM GUI: New Use visible in CUCM](/img/call-manager-api-part3/guiNewEndUserValidated.PNG)

Now just for poops and giggles if we try and run the same suite again immediately, it should fail...

![SoapUI: Test case fails due to existing account](/img/call-manager-api-part3/soapuiFullTestCaseFail.PNG)

So there we have it.  SoapUI as a change control mechanism.  But its still very clunky. Editing all that stuff is hard work.  Next up in Part 4 we look at how we can do that in python...
