---
title: "右クリックメニューから『SDelete』による完全削除を実行する"
emoji: "🚮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sdelete", "windows", "powershell"]
published: true
---

『[Sysinternals File and Disk Utilities](https://docs.microsoft.com/en-us/sysinternals/downloads/file-and-disk-utilities)』の一部として提供されている，『[SDelete](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)』というデータを完全削除できるコマンドラインツールがあります。例えば，PC や HDD， USB メモリーなどを初期化・廃棄するような場合に有用でしょう。そうした用途に限らず，機密情報が含まれる特定のファイルやディレクトリーなどを消し去りたいときにも役立ちます。

さて，後者の目的で SDelete を利用する場合，その都度，削除対象の絶対パスを指定する必要が生じます。[^abs]加えて，SDelete には Bash の `rm -i` のように削除するかどうかを尋ねる機能が備わっていません。そのため，たとえ間違ったパスを入力してしまったとしても，Enter キーを押下した瞬間には復元不可能なレベルで削除されてしまいます。また，それ以外に右クリックメニューから SDelete を実行できないことも欠点として挙げられるでしょう。ただ，SDelete はコマンドラインツールですから，こればかりは致し方ありません。

[^abs]: もちろん相対パスで指定しても実行できますが，手元の環境ではなぜか時折うまくいかないことがありました。ゆえに，絶対パスで記述したほうが確実であると思われます。

しかし，そうはいってもやはり毎度コマンドを打ったり，パスをコピーして貼り付けたりするのは億劫です。何とかして上記の欠点を補えないかと思案した結果，**コンテキストメニューの「送る」から PowerShell スクリプトを実行すれば良い**という結論に達しました。[^why-powershell]本記事では，SDelete のインストール方法，スクリプトのコードを示すとともに，「送る」メニューへの登録手順についても説明いたします。

[^why-powershell]: SDelete はコマンドプロンプトからも呼び出せますので，バッチファイルで書いても同じことができそうです。この場合，後述するショートカットを作成・配置するなどの小細工を必要とせず，ファイルを直接配置するだけで済みます。

## SDelete のインストール

SDelete をインストールする手順です。

1. [SDelete のドキュメントページ](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)にアクセスする。
2. ページ上部にある「Download SDelete (x KB)」と表記されたリンクをクリックし，`SDelete.zip` をダウンロード及び解凍する。フォルダー内には，以下のファイルが含まれている。
   - `Eula.txt`
   - `sdelete.exe`
   - `sdelete64.exe`
   - `sdelete64a.exe`
3. `C:\Windows\System32` ディレクトリー直下に，`sdelete64.exe`（32-bit 版なら `sdelete.exe`）を置く。このとき，管理者権限を要求されるので留意すること。
4. PowerShell で `sdelete` と入力し，次のような表示を得られたら成功。

   ```
   SDelete v2.04 - Secure file delete
   Copyright (C) 1999-2019 Mark Russinovich
   Sysinternals - www.sysinternals.com

   usage: sdelete [-p passes] [-r] [-s] [-q] <file or directory> [...]
       sdelete [-p passes] [-z|-c [percent free]] <drive letter [...]>
       sdelete [-p passes] [-z|-c] <physical disk number>

   -c         Clean free space. Specify an option amount of space
               to leave free for use by a running system.
   -f         Force arguments containing only letters to be treated as a file/directory rather than a disk.
               Not required if the argument contains other characters (path separators or file extensions for example)
   -p         Specifies number of overwrite passes (default is 1)
   -r         Remove Read-Only attribute
   -s         Recurse subdirectories
   -z         Zero free space (good for virtual disk optimization)
   -nobanner  Do not display the startup banner and copyright message.

   Disks must not have any volumes in order to be cleaned.
   ```

## PowerShell スクリプト

コメントなし版を Gist に上げておりますので，ご希望でしたらそちらをご参照ください。

https://gist.github.com/Meiryo7743/44bf84312a4b0b43f6e79157c48bf23b

```powershell
# 削除対象を表示する
Write-Output "The following files are going to be deleted:"

for ($i = 0; $i -lt $Args.Length; $i++) {
  Write-Output $Args[$i]
}

# 確認ダイアログ（Y/n）
$title = "Confirmation"

$message = "Are you sure want to delete them with SDelete?"

$yes = New-Object System.Management.Automation.Host.ChoiceDescription "&Yes", `
  "The files are going to be deleted."

$no = New-Object System.Management.Automation.Host.ChoiceDescription "&No", `
  "Prevent the execution of deleting file."

$options = [System.Management.Automation.Host.ChoiceDescription[]]($yes, $no)

$result = $host.ui.PromptForChoice($title, $message, $options, 0)

switch ($result) {
  # 「Y」を入力または「Enter」を押下した場合
  0 {
    for ($i = 0; $i -lt $Args.Length; $i++) {
      # 削除対象が確かに存在するかチェックする
      $exists = Test-Path -Path $Args[$i]

      if ($exists) {
        # フォルダーの場合は，サブフォルダーごと消し去る
        # 「-p 5」は上書き削除を 5 回繰り返すことを意味する
        sdelete -r -s -p 5 $Args[$i]
      }
      else {
        Write-Error -Messaege "Error: the execution was cancelled because of invalided path." -Category InvalidArgument
      }
    }
  }
  # 「n」を入力した場合
  1 {
    Write-Output "The execution was cancelled by user."
  }
}

exit
```

## 「送る」への登録方法

以下の操作手順を踏むことで，いつでも右クリックメニューの「送る」から SDelete による削除ができるようになります。

1. 上記の PowerShell スクリプトを任意の場所に `.ps1` ファイルとして保存する。
2. そのファイルのショートカットを作成する。
3. ［**右クリック**］→［**プロパティー**］→［**リンク先**］の先頭に `powershell -ExecutionPolicy RemoteSigned -File ` を付加する。デフォルトの設定では PowerShell スクリプトをクリックで実行できず，一時的にセキュリティーポリシーを変更する必要があるため。
4. Win + R で「ファイル名を指定して実行」を呼び出し，`shell:sendto` と入力する。
5. フォルダーが開かれるので，そこに**手順 3 で作ったショートカット**を設置する。

注意すべき点は，手順 3 にあるとおり，PowerShell のデフォルト設定では，`.ps1` ファイルをバッチファイルのようにクリックで実行できない，ということです。解決方法はいくつかあるのですが，ここでは一番手軽なショートカットによる手法を用いました。PowerShell は Windows の様々な機能を操作できますから，悪用されるのを防ぐためにそうなっているのでしょう。

## 削除までの流れ

実際に右クリックメニューの「送る」から SDelete による削除を試してみましょう。

1. 削除したいファイルやフォルダーを選択する（複数可）。
2. ［**右クリック**］→［**送る**］→［**（ショートカット名）**］をクリックする。
3. PowerShell が立ち上がり，削除する対象が表示される。誤りがなければそのまま Enter キーを叩く。
4. SDelete によって対象の削除が行われる。処理が完了すると，PowerShell は自動的に終了する。

## 参考

<!-- textlint-disable -->

https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete

https://stoneedge.com/help/mergedProjects/pm/PM_How_to_Use_Microsoft_SysInternals_SDelete_Command.htm

https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-powershell-1.0/ff730939(v=technet.10)

https://qiita.com/tomoko523/items/df8e384d32a377381ef9

<!-- textlint-enable -->
