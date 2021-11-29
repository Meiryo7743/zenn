---
title: "右クリックメニューから『SDelete』による完全削除を実行する"
emoji: "🚮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sdelete", "windows", "powershell"]
published: true
---

[Sysinternals File and Disk Utilities](https://docs.microsoft.com/en-us/sysinternals/downloads/file-and-disk-utilities) の一部として提供されている，[SDelete](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete) というデータを完全削除できるコマンドラインツールがあります。これは HDD や USB メモリーなどを初期化・廃棄する際に有用でしょう。また，そうした用途に限らず，機微情報が含まれる特定のファイルやディレクトリーなどを消し去りたいときにも役立ちます。

さて，後者の目的で SDelete を利用する場合，その都度，削除対象の**絶対パス**を指定する必要があります。[^abs]加えて，SDelete には Bash の `rm -i` のように削除前するかどうかを実行前に尋ねてくれる機能が備わっていません。そのため，誤って削除対象とは異なるパスを入力した場合でも，Enter キーを押下した瞬間には復元不可能なレベルで削除してしまいます。また，それ以外に右クリックメニューから直接実行できないことも欠点として挙げられるでしょう。[^cui]

[^abs]: もちろん相対パスで指定しても実行できますが，手元の環境では，なぜか時折うまく実行されないことがありました。したがって，絶対パスで記述したほうが確実でしょう。
[^cui]: SDelete はコマンドラインツールですから，こればかりは致し方ないとも捉えられます。

毎回コマンドを打ったり，パスをコピーして貼り付けたりするのは，やはり億劫なものです。どうにかして上記の欠点を補えないかと思案した結果，**コンテキストメニューの「送る」から PowerShell スクリプトを実行すれば良い**という結論に達しました。[^why-powershell]本記事では，SDelete のインストール方法とその PowerShell スクリプトのコードを示すとともに，「送る」メニューへの登録手順についても説明いたします。

[^why-powershell]: SDelete はコマンドプロンプトからも呼び出せますので，バッチファイルでも同様に可能です。この場合，[後述する小細工](#「送る」への登録)を必要とせず，バッチファイルを直接配置するだけで済みます。

## SDelete のインストール

SDelete をインストールする手順です。

1. [SDelete のドキュメントページ](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)にアクセスする。
2. ページ上部にある「**Download SDelete (x KB)**」と表記されたリンクをクリックし，`SDelete.zip` をダウンロード及び解凍する。内部には以下のファイルが含まれている。
   - `Eula.txt`
   - `sdelete.exe`
   - `sdelete64.exe`
   - `sdelete64a.exe`
3. `C:\Windows\System32` 直下に，`sdelete64.exe`（32-bit 版なら `sdelete.exe`）を置く。このとき，管理者権限を要求されるので留意すること。
4. PowerShell で `sdelete` と打ち，次のような表示を得られたら成功。

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

Gist にもコードを置いてあります。

@[card](https://gist.github.com/Meiryo7743/44bf84312a4b0b43f6e79157c48bf23b)

```powershell
# 削除対象の絶対パスを表示する
# 対象数が 0 の場合は何もせずに終了する
switch ($Args.Length) {
  0 {
    Write-Host "No item is selected."
    exit
  }
  1 {
    Write-Host "The following item will be deleted:"
    Write-Host "$($Args)"
  }
  Default {
    Write-Host "The following items will be deleted:"
    foreach ($item in $Args) {
      Write-Host "$($item)"
    }
  }
}

# 削除を実行するか尋ねる
$title = "WARNING!"
$message = "Once you start this operation, you cannot restore them all. Are you sure you want to delete them with SDelete?"
$options = @(
  New-Object System.Management.Automation.Host.ChoiceDescription(
    "&Yes",
    "Delete."
  )
  New-Object System.Management.Automation.Host.ChoiceDescription(
    "&No",
    "Cancel."
  )
)
$choice = $host.ui.PromptForChoice(
  $title,
  $message,
  $options,
  1
)
switch ($choice) {
  # [Y]: Delete
  0 {
    foreach ($item in $Args) {
      sdelete -nobanner -r -s -p 5 $item
    }
  }

  # [N]: Cancel (default)
  1 {
    Write-Host "You cancelled this operation."
  }
}
```

## 「送る」への登録

以下の操作手順を踏むことで，いつでも右クリックメニューの「送る」から SDelete による削除が可能です。

1. 上記の PowerShell スクリプトを任意の場所に `.ps1` ファイルとして保存する。
2. 保存したファイルのショートカットを作成する。
3. ［**右クリック**］→［**プロパティー**］→［**リンク先**］の先頭に `powershell -ExecutionPolicy RemoteSigned -File ` を付加する。デフォルトの設定では PowerShell スクリプトをクリックで実行できず，一時的にセキュリティーポリシーを変更する必要があるため。
4. Win + R で「ファイル名を指定して実行」を呼び出し，`shell:sendto` と入力する。
5. 表示されたディレクトリー内に**手順 3 で作ったショートカット**を設置する。

注意すべき点は，**手順 3** にあるとおり，PowerShell のデフォルト設定では，`.ps1` ファイルをバッチファイルのようにクリックで実行できないことです。解決方法はいくつかあるのですが，ここでは一番手軽なショートカットによる手法を用いました。PowerShell は Windows の様々な機能にアクセスできますから，悪用されるのを防ぐためにこうなっているのでしょう。

## 削除までの流れ

実際に右クリックメニューの「送る」から SDelete による削除を試してみましょう。

1. 削除したいファイルやディレクトリーを選択する（複数可）。
2. ［**右クリック**］→［**送る**］→［**（ショートカット名）**］をクリックする。
3. PowerShell が立ち上がり，削除する対象と確認ダイアログが表示される。誤りがなければ `y` と入力し，Enter キーを叩く。
4. SDelete による削除が行われる。処理が完了すると，PowerShell は自動的に終了する。

## 参考

<!-- textlint-disable -->

@[card](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)

@[card](https://stoneedge.com/help/mergedProjects/pm/PM_How_to_Use_Microsoft_SysInternals_SDelete_Command.htm)

@[card](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-powershell-1.0/ff730939(v=technet.10))

@[card](https://qiita.com/tomoko523/items/df8e384d32a377381ef9)

<!-- textlint-enable -->
