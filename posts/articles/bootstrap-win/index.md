---
title: "Windows PC のセットアップを楽にする"
date: "2024-04-14T18:00:29+09:00"
description: winget をはじめとするパッケージ管理ツールと少しのスクリプトでセットアップを楽に。
---

## 背景

新しく PC を購入した際、最初に自分が普段使っているアプリケーションをまとめてインストールすると思いますが、
PC を高頻度で買い替える場合、このインストール作業が非常に大変だと思います。

そこで、この作業を楽にするため、自動化スクリプトのようなものを PowerShell で書いてみます。

---

## 手動でインストールする手順

自動化スクリプトを作成するにあたり、まずは手動での手順をおさらいします。

### パッケージ管理ツールをインストールする

Windows に用意されているパッケージ管理ツールには、
[winget](https://learn.microsoft.com/ja-jp/windows/package-manager/winget/) や
[chocolatey](https://chocolatey.org/)、
[scoop](https://scoop.sh/) などがあります。

winget に関しては私の PC では標準でインストールされていました。

winget がインストールされていることを確認するには、
PowerShell で `winget --version` を実行してバージョン番号が出ることを確認します。

```text
PS C:\Users\somebody> winget --version
v1.7.10861
```

chocolatey や scoop は公式サイトのインストール方法を参考にします。

- chocolatey

```powershell
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

- scoop

```powershell
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```

### 必要なアプリケーションを検索・インストールする

#### winget

winget の場合、`winget search ${キーワード}` で検索し、ID を控えておきます。
例えば Zoom を検索した場合は `Zoom.Zoom` になります。

```text
PS C:\Users\somebody> winget search zoom
名前                           ID                                            バージョン    一致      ソース
------------------------------------------------------------------------------------------------------------
Zoom - One Platform to Connect XP99J3KP4XZ4VV                                Unknown                 msstore
Crosshair Zoom                 9P2ZTH30XXGL                                  Unknown                 msstore
Zoom Player                    XP88X30WJ519GR                                Unknown                 msstore
Zoom                           Zoom.Zoom                                     5.17.11.34827           winget
Zoom Outlook Plugin            Zoom.ZoomOutlookPlugin                        6.0.0         Tag: zoom winget
Fotophire Photo Maximizer      Wondershare.Fotophire.Maximizer               1.3.1         Tag: zoom winget
AeroZoom Beta                  wandersick.AeroZoom                           4.0.2         Tag: zoom winget
Virtual Magnifying Glass       VirtualMagnifyingGlass.VirtualMagnifyingGlass 3.6           Tag: zoom winget
Final2x                        Tohrusky.Final2x                              1.2.0         Tag: zoom winget
Resizer                        den4b.Resizer                                 2.1.0.0       Tag: zoom winget
Zoom Rooms                     Zoom.ZoomRooms                                5.17.8                  winget
ZoomIt                         Microsoft.Sysinternals.ZoomIt                 8.01                    winget
```

Zoom をインストールするには `winget install --exact --id Zoom.Zoom` を実行します。

#### chocolatey

chocolatey の場合は [パッケージ一覧](https://community.chocolatey.org/packages) で検索するか、
`choco search ${キーワード}` で検索します。

```text
PS C:\Users\somebody> choco search zoom
Chocolatey v2.2.2
（前略）
zoom 5.17.11.34827 [Approved]
（後略）
```

インストールは `choco install zoom` です。
chocolatey の場合、アプリケーションのインストール時に要求される管理者権限が自動では要求されないため、PowerShell を管理者権限で起動しておく必要があります。

#### scoop

scoop の場合は [トップページ](https://scoop.sh/) で検索するか、
`scoop search ${キーワード}` で検索します。

scoop ではソフトウェアがいくつかのバケット（リポジトリのようなもの）に分かれており、バケットの追加とアプリケーションの追加をそれぞれ行う必要があります。 
`git-chglog` は `main` バケットに属しているようです。

```text
PS C:\Users\somebody> scoop search git-chglog
Results from local buckets...

Name       Version Source Binaries
----       ------- ------ --------
git-chglog 0.15.4  main
```

インストールするには以下のコマンドを実行します。

```powershell
scoop bucket add main
scoop install main/git-chglog
```

{{< alert type="info" >}}
scoop のパッケージは main バケット以外に様々なバケットが存在するため、パッケージ名とバケット名を CSV でまとめた方がいいかもしれません。
ここではインストールしたいパッケージがすべて `main` バケットであると仮定して進めます。
{{</ alert >}}

---

## 自動スクリプトの作成

### PowerShell のスクリプトを実行できるようにする

PowerShell のスクリプトである .ps1 ファイルを実行するには、実行ポリシーの設定が必要です。
PowerShell を開き、`Get-ExecutionPolicy` を実行して `Restricted` と表示される場合はスクリプトの実行が制限されています。
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` を実行してスクリプトの実行を許可します。

### スクリプトを記述する

ここからスクリプトの記述となります。
chocolatey で管理者権限が必要になるため、最初に管理者権限を要求するようにしておくと便利です。

```powershell
if (!([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole("Administrators"))
{
    Start-Process powershell.exe "-File `"$PSCommandPath`"" -Verb RunAs
    exit
}
```

#### winget

検索しておいた ID をテキストファイルにまとめておき、そのテキストファイルを読み込んで 1 行ずつ処理する方法でやってみます。

参考までに、テキストファイルの例を記載します。
ファイル名は `winget_dependencies.txt` としておきます。

```text
7zip.7zip
Docker.DockerDesktop
FiloSottile.mkcert
GnuPG.Gpg4win
JetBrains.Toolbox
Microsoft.VisualStudioCode
SlackTechnologies.Slack
Sony.MusicCenter
Zoom.Zoom
```

まずはテキストファイルを読み込み、文字列の配列にして変数に格納します。

```powershell
$dependencies = [string[]](Get-Content -Path "${PSScriptRoot}\winget_dependencies.txt" | Select-Object)
```

各パッケージに対して、インストールされているかどうかを確認し、インストールされていない場合はインストール、インストールされている場合はアップグレードします。
インストール済みパッケージを検索する `winget list` コマンドは、パッケージが見つからない場合に結果が `False` となるため、
`$?` を利用した判定を行います。

```powershell
foreach ($dependency in $dependencies)
{
    winget list --exact --id $dependency
    if ($?)
    {
        winget upgrade --exact --id $dependency
    }
    else
    {
        winget install --exact --id $dependency
    }
}
```

#### chocolatey

chocolatey の場合、`choco list` でローカルにインストールされているパッケージを検索できますが、
インストールされている場合とされていない場合の区別をするため、
`--limit-output` オプションを付けることでインストールされている場合のみコンソールへの出力が発生するようにします。

winget と同じようにテキストファイルと foreach 句まで書き、各パッケージの処理は以下のようになるイメージです。

```powershell
if (choco list --limit-output --exact $dependency)
{
    choco upgrade -y $dependency
}
else
{
    choco install -y $dependency
}
```

#### scoop

scoop の場合はバケットの検索とパッケージの検索をそれぞれ行います。

バケットは `scoop bucket list` で検索するとハッシュテーブルの形で取得できます。

```text
Name   Source                                   Updated             Manifests
----   ------                                   -------             ---------
extras https://github.com/ScoopInstaller/Extras 2024/04/14 18:56:11      2008
main   https://github.com/ScoopInstaller/Main   2024/04/14 19:33:46      1313
```

ハッシュテーブルの Name 属性にバケット名が含まれているかどうかで、バケットが存在するかを判定します。
この後のパッケージ更新で、バケットに更新があると自動更新されます。
その前に `scoop update` でバケットをまとめて更新しておくとログ出力が整理されるので個人的におすすめです。

```powershell
scoop update
$buckets = scoop bucket list
if ("main" in $buckets.name)
{
    scoop bucket add main
}
```

パッケージは `scoop list` で検索するとハッシュテーブルの形で取得できます。

```text
Name                   Version Source Updated             Info
----                   ------- ------ -------             ----
git-chglog             0.15.1  main   2022-04-02 08:33:01
```

スクリプトとしては、以下のように書きます。

```powershell
$installed = scoop list
if ($dependency -in $installed.name)
{
    scoop update $dependency
}
else
{
    scoop install $dependency
}
```

### 完成品

`install.ps1`

```powershell
if (!([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole("Administrators"))
{
    Start-Process powershell.exe "-File `"$PSCommandPath`"" -Verb RunAs
    exit
}

# winget
$dependencies = [string[]](Get-Content -Path "${PSScriptRoot}\winget_dependencies.txt" | Select-Object)
foreach ($dependency in $dependencies)
{
    winget list --exact --id $dependency
    if ($?)
    {
        winget upgrade --exact --id $dependency
    }
    else
    {
        winget install --exact --id $dependency
    }
}

# chocolatey
$dependencies = [string[]](Get-Content -Path "${PSScriptRoot}\choco_dependencies.txt" | Select-Object)
foreach ($dependency in $dependencies)
{
    if (choco list --limit-output --exact $dependency)
    {
        choco upgrade -y $dependency
    }
    else
    {
        choco install -y $dependency
    }
}

# scoop
scoop update
$buckets = scoop bucket list
if ("main" in $buckets.name)
{
    scoop bucket add main
}

$dependencies = [string[]](Get-Content -Path "${PSScriptRoot}\scoop_dependencies.txt" | Select-Object)
$installed = scoop list
foreach ($dependency in $dependencies)
{
    if ($dependency -in $installed.name)
    {
        scoop update $dependency
    }
    else
    {
        scoop install $dependency
    }
}
```

完成したら、PowerShell で `.\install.ps1` と入力して実行するか、
エクスプローラで install.ps1 を右クリックして「PowerShell で実行」をクリックすれば実行されます。

今回のスクリプトには chocolatey や scoop 自身のインストールを含めていませんが、
パッケージと同じように、chocolatey や scoop が存在しない場合はインストールするなどの分岐を記述すれば正常に動くものと思われます。

---

## まとめ

いかがでしたでしょうか。

今回はインストールを単に自動化するだけでなく、アップデートも考慮してスクリプトを作成したので、
まっさらの状態からインストールする場合、アップデートする場合、パッケージを追加する場合など、あらゆる状況で同じスクリプトが使えます。
（削除する場合のみ非対応・・・ :cry:）

また、このようにアプリケーションのインストールを一部自動化することで、環境構築にかかる時間が短縮されるのはもちろんのこと、
大量の新入社員向けに大量の PC をセットアップする際に情シスの負担を軽減することにも繋がるので、積極的に導入したいですね。
