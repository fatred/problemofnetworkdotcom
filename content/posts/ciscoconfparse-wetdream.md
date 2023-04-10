+++
title = "Ciscoconfparse Wetdream"
date = "2015-15-30T03:07:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Tools","Automation","Devops","Python",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
Ick,  I know.

Python has long been the language of choice for engineers looking to make their day go that little bit quicker or easier.  With deepening skill levels, more and more complex repetitive tasks can be disected and segmented into functions and reuable code, such that a competent scripting engineer can go from blank page to automated process in a matter of hours in most cases.  It is for this reason that I sit here to write this - an advocacy for ALL Cisco engineers to down tools and spend however long you need to get good at this.

## Stop doing work and learn scripting? What are you high?

No. Unless you mean caffeinated, at which point yes.

Recently I found myself working with a satellite team of semi skilled guys in a far far away place. They needed us to tweak switchport settings on a 2960x stack as part of a slide migration from static vlan access ports to dot1x enabled dynamic vlan mappings.

We had a process whereby there was a spreadsheet with a port mapping on, and they would edit that highlight the edits, save to sharepoint and send us a ticket to process the changes.  We would complete the change, edit the spready to "commit" that change and close the ticket.  Pretty standard balance of audit history and inefficiency.  The issue would come when they would call up with an urgent request, which we would complete there and then, but despite our protestations, they would never bother to revisit the sheet and retrospectively request everything. Next thing you know they put in a regular request the proper way, and the document is wrong. FAIL!

In reality I'm not that fussed about the sheet, and the audit trail is more for info sharing/documentary evidence than it is for anything official.

So, what if I could automate the generation of the sheet every night?  No sending it back and forth, and we are free to make the change as needed on demand (just fire in a ticket).

Enter Paramiko (SSH Client), CiscoConfParse (obvs), and openpyxl (Excel Library).

The full code can be found in my ![basic-scripts](https://github.com/fatred/basic-scripts/blob/master/get-switch-map.py) repo on github.

_Note: this script relies on the following non-standard imports to work. Make sure you install them with pip/easy_install before use..._

* paramiko
* ciscoconfparse
* argparse
* getpass
* openpyxl

Now this code takes a few liberties.  I use functions, but I KNOW they could be segmented further, and probably classed.  I'm no expert, and this works for me.  Feel free to branch and submit me pull requests if you can do it better ;)

The basic flow of the script is as follows:

1. Check what args the user provided - ensure thye have the address of the switch as a minimum, and then gather whatever else was provided behind the -- tags and assign them appropriately for global use.  Prompt for required missing extras.
2. Use the SSH details to try and access the switch
3. When connected, disable output paging for the session, and do a sh run to gather all data into a buffer.
4. parse buffer using the CiscoConfParse module.  This uses various forms of intelligence to group like minded config sections together for searching/manipulation and the creation of rollout/rollback scripts
5. report some summary data to the end user to prove the script is talking to the switch properly
6. create a new workbook object and label up its first worksheet 
7. install the row headers
8. extract all the gigabit ethernet ports and store them in a list of config objects.
9. iterate through each port in turn
   1.  extract all the dot1x ports from the list, and update the spreadsheet adding some pertinent data to appropriate columns in a specific row for that port
   2.  extract all the static ports from the list, and update the spreadsheet adding some pertinent data to appropriate columns in the specific row for that port
   3.  extract all the trunk ports from the list, and update the spreadsheet adding some pertinent data to appropriate columns in a specific row for that port
10. report some interface data back to the end user, then save the worksheet to disk

I will start with the get_switch_conf function, since this is the most reusable section of code I have. Most if not all of that is generic enough to lift and shift to any other script that uses SSH to gather base configs.

`get_switch_conf():`
To drive this function we need some values sent in for working; the IP, username and password for the SSH session.  We also have a debug option if we want to get into the weeds and watch the session in near real time.

The function starts by setting up a paramiko SSH client.  This is in two stages;

* the preconnection requirements are set to auto add any ssh key to our users ssh keystore without checking. In a trusted, closed environment that's probably fine. The Paramiko docs are good for talking you through the other options.  we then tell the object holding the preconditions to connect to our switch.
* once connected, we then build a new object to hold the shell provided by the SSH client connection

Now we are connected, we need to tweak our session to ensure that a "sh run" doesn't get paged with the -- MORE -- annoyances, and a program trying to guess if it needs to press spacebar for the next page.providing we don't "copy run start" the session we are on, we are fine - the terminal length 0 command will not persist between connection attempts.

We are now in a position to send in a "sh run" and extract the regular config as a multiline string and save it to an object in our script.  At this point, we can log out of the switch and not worry about hogging vty lines.

the real magic is the last few lines of that function:

```python
    # read the return buffer into an object
    output = remote_conn.recv(256000)

    # convert object to a string for parsing
    conf_string = str(output).splitlines()

    cisco_conf = CiscoConfParse(conf_string, factory=True)

    switch_hostname = cisco_conf.find_objects(r"^hostname")[0].text[9:]
    switch_version = cisco_conf.find_objects(r"^version")[0].text[-4:]
    print "[*] Config Parsed: %s | %s" % (switch_hostname,switch_version)

    # return that as CiscoConfParse obj
    return cisco_conf 
```

first we have to get the data back from the switch, then we flatten that to a single string and then insert newlines to help it parse properly.  we then tell the CiscoConfParse module to contextualize the entire config file.

in the penultimate 3 lines we show off a bit, parsing through the config file and yanking out just a couple of info nuggets and formatting them in a nice way for the user to see and validate that we are doing a good job. Once we are all done, we hand a single object back to the rest of the script that can be simply and quickly queried.

`add_value_to_cell():`
this one is a quick one liner that takes a sheet object, and then inserts or replcaes a value in a cell referenced at the intersection of a given row/cell.

the excel bit
This is a funny beast.  In one way, I'm amazed how easy it was to get great results fast with this, but in another, it took a little while to figure out how I could iterate through the switchports in list order, whilst ensuring that I only hit a cell once with the correct data.

```python
    # make a new sheet
    spreadsheet = openpyxl.Workbook() 

    sheet = spreadsheet.active
    sheet.title = 'Port Map' 

    # add headings
    sheet.cell(row=1,column=1).value = "interface"
    sheet.cell(row=1,column=2).value = "dot1x|static|trunk"
    sheet.cell(row=1,column=3).value = "port desc"
    sheet.cell(row=1,column=4).value = "access vlan|trunk vlans"
```

So we made a new sheet, picked the first sheet (tab), and then set its title appropriately

We then add headings into row 1 for each of the 4 columns we will be populating from the ioscfg sections.

We then need to limit our work to just the gigabitEthernet ports.

```python
    # get all the interface object
    interfaces = switch_conf.find_objects(r'^interface GigabitEthernet') 
```

now we have a list object called interfaces, each object within can be manipulated by the CiscoConfParse module.  We need to iterate through this list and extract some key information for our spready, which we then plonk into the correct columns for the appropriate port.

Now this one is a bit of a chunky paste...

```python
    # populate the spreadsheet
    for interface in interfaces:
    # increment the total intf count
    total_intf_count+=1

    # get the layout correct
    switch = interface.ordinal_list[0]
    port = interface.ordinal_list[2]

    # check if its a dot1x port
    if interface.re_search_children(r'dot1x'):
    # increment the dot1x intf count
    dot1x_intf_count+=1

    if debug:
    print "[D] Interface %s is dot1x enabled" % interface.name
    name = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 1, interface.name)
    type = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 2, "dot1x")
    desc = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 3, interface.description)
    acc_vl = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 4, interface.access_vlan)

    else:
    name = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 1, interface.name)
    if interface.access_vlan == 0:
    # increment trunk intf counter
    trunk_intf_count+=1

    if debug:
    print "[D] Interface %s is trunk port" % interface.name
    type = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 2, "trunk")
    desc = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 3, interface.description)
    all_vl = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 4, "all")
    else:
    # increment access port counter
    static_intf_count+=1
    if debug:
    print "[D] Interface %s is a static port" % interface.name
    type = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 2, "static")
    desc = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 3, interface.description)
    acc_vl = add_value_to_cell(sheet, (54+port if switch == 2 else 2+port), 4, interface.access_vlan)
```

Starting at the top, we start a loop that takes each interface in the list one at a time and perform the same operations on them.  To keep track of the interfaces we have, we add 1 to the total_interf_count variable.
the switch and port variables are used to ensure we enter the data into the spreadsheet in the right place.  ordinal_list returns 3 integers, matching the interface e.g. GigabitEthernet[1]/[0]/[1] would give switch = 1 port = 1 (I don't need the ordinal_list[1] value for module - the middle digit).
we then split ports into two major types - those with dot1x enabled, and those without.  in the without camp i then have two sub types - access and trunk.
if the port is a dot1x port, then i add the interface name, "dot1x, description and access_vlan id (guest vlan id basically) to the cell appropriate to that port.
the way we identify which row to use, depends on two things.  the port number obviously, and whether that port is in switch 1 or 2.  if its in switch 1 then i start in row 3 for port 1, and go up to row 54 (52 ports in a 1G only 2960x), for switch 2, following on directly behind switch 1, its row 53 onward. Stop and think about that for a while, and run the script - you will see what I mean.

if the port is not dot1x enabled, then we look at the current access vlan id for the port.  If that is set to 0, then we know it is a trunk port.  if it is a non zero int, then we have an access port.  We repeat the same exercise as in dot1x land, but enter in the values relevant for that port.

Once all this iteration is over, we use the state we generated in the loops to report on how many of each port we have, and then save the file to disk. if no specific file name is given, then we use the ip and append "_switchport_map.xlsx

this is then scheduled to run every morning at 3am, upon which the output is committed to Sharepoint.

I am now looking to extend this further into a webapp, that with role based access, and approval mechanisms, provides a self service portal for the non-skilled users on site, to administer their switch in house, without risking other key assets, and without costing time and effort for upskilling users who wont need the skills regularly once the project is complete.
