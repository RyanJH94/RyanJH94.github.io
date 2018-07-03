---
layout: post
title: Cracking Domain Passwords
subtitle: Using Hashcat on Windows
tags: [Penetration Testing, Hashcat]
comments: true
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

Fire up another command prompt and change directory to where your Hashcat executable is stored. From here, you can run

~~~
hashcat64.exe --help
~~~

to view all switches and supported hash types in Hashcat.

In this case, we're going to use a dictionary and rule based attack against NTLM hashes. This means we want an attack mode of 0 and a hash type of 1000. Lets start to build that Hashcat command.

~~~
hashcat64.exe -a 0 -m 1000
~~~

The above calls Hashcat and tells it that you want to use the "Straight" attack mode againt NTLM hashes, but we haven't told it what hashes, dictionaries, or rules to use yet. We'll append these next, and because our hash file contains usernames, we'll need to tell Hashcat that. Otherwise, you'll get Hash Length Exception errors.

In the below command, "--username" tells Hashcat to expect usernames in the input hashes. "-o" followed by a file path tells Hashcat to output its findings to file. "C:\Your\File\Path\Hashes.txt" is the path to the hashes we extracted earlier. "C:\Dictionary\File\Path\rockyou.txt" is the path to your dictionary, in this case we're using rockyou which is a prebuilt dictionary available online. Finally, "-r" followed by a file path tell Hashcat to use a rule to modify each line of the dictionary. In this case, we're using Dive which is included with Hashcat.
~~~
hashcat64.exe -a 0 -m 1000 --username -o C:\Your\File\Path\Output.txt C:\Your\File\Path\Hashes.txt C:\Dictionary\File\Path\rockyou.txt -r .\rules\dive.rule
~~~

If Hashcat reports it may take a while to run through your dictionary and rule, you can append "-w 3" or "-w 4"; this tells Hashcat to use more resource (-w 4 may make your machine unusable until Hashcat has finished).

Once Hashcat is finished, you'll find a file in your output path containing hashes and passwords in a hash:password format, but all the usernames are gone! To get all passwords in a format of username:hash:password, we'll need to rerurn Hashcat again; don't worry it won't need to recrack everything as it remembers what it's already cracked (it stores these hashes in a 'pot file').

This time, run:

~~~
hashcat64.exe -a 0 -m 1000 --username --show -o C:\Your\File\Path\OutputWithUsername.txt C:\Your\File\Path\Hashes.txt C:\Dictionary\File\Path\rockyou.txt -r .\rules\dive.rule
~~~

Note the change here is the "--show".

That's it. You now have usernames and passwords for weak passwords in your domain! 

If you didn't crack as many as you'd thought, try editing your ditcionary to include things like:

* Days of the week
* Months of the year
* Seasons
* Local town/city names
* Local sports teams names
* Your company name

Happy cracking!

-Ryan
