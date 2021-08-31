### 0. はじめに
Hack the box のウォークスルー（CTFでいうWrite Up）の英語版はたくさん存在するが，
日本語はあまり見かけないため，今回から日本語で記載．

### 1. ポートスキャン
TCPのポートでサービスが何を動いているかをnmapによって調査．
（nmapは速度は遅いが，--scriptオプションがかなり優秀）
```
$ sudo nmap -A 10.10.10.28
```
オプションは適時変更する．
今回は，22番（SSH）と80番（HTTP）が動いていることが分かる．
SSHはマシン管理者の整備用であると思われるため，
80番を中心に見ていく．

### 2. Webアプリケーションへの攻撃
Webページの脆弱性診断をするために，Burp Suiteを使用する．
Burp Suiteの設定はFox Proxyを利用すると簡単に設定，変更ができる．
詳しくは以下を参照してほしい．
Burp SuiteのTarget->Site mapから

```
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.10.10.28/
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.28/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/07/20 11:25:29 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 276]
/.hta                 (Status: 403) [Size: 276]
/.htpasswd            (Status: 403) [Size: 276]
/css                  (Status: 301) [Size: 308] [--> http://10.10.10.28/css/]
/fonts                (Status: 301) [Size: 310] [--> http://10.10.10.28/fonts/]
/images               (Status: 301) [Size: 311] [--> http://10.10.10.28/images/]
/index.php            (Status: 200) [Size: 10932]                               
/js                   (Status: 301) [Size: 307] [--> http://10.10.10.28/js/]    
/server-status        (Status: 403) [Size: 276]                                 
/themes               (Status: 301) [Size: 311] [--> http://10.10.10.28/themes/]
/uploads              (Status: 301) [Size: 312] [--> http://10.10.10.28/uploads/]
                                                                                 
===============================================================
2021/07/20 11:26:48 Finished
===============================================================
```  
