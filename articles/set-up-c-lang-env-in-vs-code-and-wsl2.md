---
title: "VS Code & WSL 2 で C 言語の開発環境をサクッと構築する"
emoji: "⚙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["C", "C言語", "vscode", "wsl2"]
published: false
---

[WSL 2（Windows Subsystem for Linux）](https://learn.microsoft.com/en-us/windows/wsl/)からインストールした Ubuntu 上で，**VS Code の [Tasks 機能](https://code.visualstudio.com/docs/editor/tasks)を使って C 言語のコードを書いて簡単にコンパイルとデバッグが可能な環境**を作り上げます。

:::message alert

**WSL 2 は Windows 固有の機能**です。MacOS をはじめとする Windows 以外の OS では実現できません。

:::

## ターゲット

- VS Code で C 言語のコードを書きたい人
- 既に別の環境で C 言語のコードを書いているが，VS Code と WSL 2 による環境へと移行しようか検討している人

いずれの場合も，例えば「C 言語のコードは大学の授業でしか書く機会がない」といった，さほど本格的（？）な開発環境は求めていないケースを想定しています。

## 導入手順

### 1. VS Code をインストールする

:::message

既に VS Code をインストール済みの場合は，この手順を読み飛ばし，次の「[WSL 2 から Ubuntu をインストールする](#2.-wsl-2-から-ubuntu-をインストールする)」にお進みください。

:::

C 言語のコードを書くエディターとして「[VS Code](https://code.visualstudio.com/)」を使います。公式サイトの「**Download for Windows Stable Build**」をクリックし，インストーラーをダウンロードしましょう。

![](https://storage.googleapis.com/zenn-user-upload/7d9961b688c5-20221203.png)

あとはダウンロードしたインストーラーを実行し，画面指示に従って操作するだけで VS Code のインストール作業は完了です。

### 2. WSL 2 から Ubuntu をインストールする

:::message

既に WSL 2 上で Ubuntu をインストール済みの場合は，この手順を読み飛ばし，次の「[VS Code に拡張機能をインストールする](#3.-vs-code-に拡張機能をインストールする)」にお進みください。

:::

続いて，WSL 2 上に Ubuntu をインストールして行きましょう。PowerShell（もしくはコマンドプロンプト）を起動し，以下のコマンドを実行してください。

```
wsl --install -d Ubuntu
```

そのまま気長に待ち続けると，やがて Ubuntu のウィンドウが出現するはずです。

![](https://storage.googleapis.com/zenn-user-upload/8698ac462836-20221203.png)

このあと，Ubuntu のウィンドウの指示に従って，ユーザー名とパスワードを設定します。最終的に以下の画像のような状態になれば，WSL 2 と Ubuntu のセットアップは完了です。

![](https://storage.googleapis.com/zenn-user-upload/25c10d8a5dfa-20221203.png)

### 3. VS Code に拡張機能をインストールする

:::message

ここでは，VS Code に「Remote Development」と「C/C++ Extension Pack」をインストールしていきます。既にこの 2 つがインストール済みの場合は，この手順を読み飛ばし，次の「[Ubuntu に GCC と GDB をインストールする](#4.-ubuntu-に-gcc-と-gdb-をインストールする)」にお進みください。

:::

WSL 2 上の Ubuntu で VS Code を使うにあたって，「[Remove Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)」は欠かせません。「**Install**」ボタンをクリックしてインストールしましょう。

![](https://storage.googleapis.com/zenn-user-upload/2c9b2c13d4a7-20221203.png)

インストールが無事に成功すると，次のような画面が現れるはずです。後述の「C/C++ Extension Pack」をインストールする準備のため，「**Open the Menu**」ボタンをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/01d2d4cc24ec-20221203.png)

「**New: WSL Window**」を選択し，クリックします。そうすると，新しく VS Code のウィンドウが立ち上がります。

![](https://storage.googleapis.com/zenn-user-upload/fbb0bdd7a594-20221203.png)

続いて，**先ほど開いた WSL 側のウィンドウ**[^wsl-window]から「[C/C++ Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools-extension-pack)」を追加します。先と同じく「**Install in WSL: Ubuntu**」を押下すれば良いです。

[^wsl-window]: 左下のステータスバーに「**WSL: Ubuntu**」と表示されているウィンドウ

![「C/C++ Extension Pack」を追加するときの画面](https://storage.googleapis.com/zenn-user-upload/a660dd19de32-20221203.png)

この拡張機能が追加されるまで 5 分ほどの時間を要します。気長に待ちましょう。しばらくして，下のような状態になっていれば無事完了です。

![「C/C++ Extension Pack」を無事に追加できたときの画面](https://storage.googleapis.com/zenn-user-upload/08f20b9fc5e3-20221203.png)

### 4. Ubuntu に GCC と GDB をインストールする

:::message

既に WSL 2 上の Ubuntu で GCC と GDB の両方がインストール済みの場合，この手順を読み飛ばし，次の「[ワークスペースを作成し，実際に C 言語のコードを実行してみる](#5.-ワークスペースを作成し，実際に-c-言語のコードを実行してみる)」にお進みください。

:::

[GCC](https://gcc.gnu.org/)（コンパイラー）と [GDB](https://www.sourceware.org/gdb/)（デバッガー）の両方が WSL 2 上の Ubuntu にないと，VS Code の Tasks 機能がエラーとなり動いてくれません。そのため，以下のコマンドを実行して両者のインストールを行います。なお，実行時にパスワードを要求されたときは，**WSL 2 の Ubuntu で設定したユーザーパスワード**を入力してください。

```
sudo apt install gcc gdb
```

### 5. ワークスペースを作成し，実際に C 言語のコードを実行してみる

VS Code で C 言語のコードを書くためのワークスペース（作業場所）を作成します。今回は，例としてホームディレクトリーの直下に `example` という名のディレクトリーを作り，これをワークスペースとします。具体的には，以下のコマンドをそのまま実行するだけです。

```
cd && mkdir test && code $_
```

すると，`example` をワークスペースとした状態で VS Code が起動します。もしも下の画像のような画面が出た場合は，青いボタンを押しましょう。[^vscode-restrict-mode]

[^vscode-restrict-mode]: VS Code では，セキュリティー保護の観点から，拡張機能やデバッグ機能がデフォルトで無効化される「[Restrict Mode](https://code.visualstudio.com/docs/editor/workspace-trust)」という機能が存在します。本記事で作成したワークスペースは，ユーザーが自らの手で作ったものであり，なおかつ拡張機能とデバッグ機能を必要とするため，Restrict Mode を無効化（信頼するに設定）しているわけです。

![](https://storage.googleapis.com/zenn-user-upload/983a0f033475-20221203.png)

続いて，以下のプログラム `example.c` を作成します。コンパイルして実行すると，画面に `Hello, world!` と表示されるだけのシンプルなプログラムです。ご自身の好みに応じて，コードは自由に書き換えてください。[^example-c]

[^example-c]: 例えば，円周率 $\pi$ やネイピア数 $\mathrm{e}$ の近似値を表示するコードに変えても良いかもしれませんね！

```c:example.c
#include <stdio.h>

int main(void)
{
  printf("Hello, world!\n");
  return 0;
}
```

`example.c` を開いた状態のまま `F5` キーを押下します。出てきたメニューから，「**C/C++: gcc build and debug active file**（下の画像では最下段にある選択肢）」を選択してください。

![](https://storage.googleapis.com/zenn-user-upload/b94f166f98b9-20221203.png)

すると，**`example.c` のコンパイルと，生成されたバイナリファイル `example` が自動で実行されます**。これが VS Code の **Tasks 機能**です。わざわざコマンドを手打ちする手間が省けて大変便利ですね！

![](https://storage.googleapis.com/zenn-user-upload/e52551e0b631-20221203.png)

ただ，上の画像のように実行結果が表示されておらず，本当に正しく動作したのか分かりません。実行結果を表示するには，「Terminal」タブをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/9a5979774042-20221203.png)

間違いなく `Hello, world!` と表示されています。**ここまで確認できれば，一連の導入手順は全て無事に完了です！**

## （補足）`tasks.json` をカスタマイズする

手順 5 の `example.c` を Tasks 機能でコンパイルすると，`example` という拡張子なしのバイナリファイルが生成されました。通常，GCC でコンパイルすると生成されたバイナリファイルには `.out` という拡張子が付与されます。Tasks 機能でも，ワークスペース直下に自動生成された `.vscode/tasks.json`（Tasks 機能の設定ファイル）を編集すれば，同様のことを実現可能です。

```diff json:tasks.json
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: gcc build active file",
            "command": "/usr/bin/gcc",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
-               "${fileDirname}/${fileBasenameNoExtension}"
+               "${fileDirname}/${fileBasenameNoExtension}.out"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "Task generated by Debugger."
        }
    ],
    "version": "2.0.0"
}
```

上記のように修正した後，再度 `example.c` を Tasks 機能でコンパイルすると，`example.out` が代わりに生成されることが分かります。

![](https://storage.googleapis.com/zenn-user-upload/3fe542c60c28-20221203.png)
