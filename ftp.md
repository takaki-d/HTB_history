
```
┌──(kali㉿kali)-[~]
└─$ ftp 10.129.243.180                                                                               1 ⚙
Connected to 10.129.243.180.
220 (vsFTPd 3.0.3)
Name (10.129.243.180:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04 03:25 flag.txt
226 Directory send OK.
ftp> get flag.txt
local: flag.txt remote: flag.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for flag.txt (32 bytes).
226 Transfer complete.
32 bytes received in 0.00 secs (136.4629 kB/s)
ftp> bye
221 Goodbye.
                                                                                                         
┌──(kali㉿kali)-[~]
└─$ ls                                                                                               1 ⚙
Desktop  Documents  Downloads  flag.txt  Music  Pictures  Public  starthtb  Templates  Videos
```
