---
title: "Google Apps Script でアクセスカウンターを作る"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas", "web"]
published: false
---

その昔，多くのウェブサイトでアクセスカウンターが設置されていました。サイトにどれだけのアクセスがあったのかを計測するためです。しかしながら，[Google アナリティクス](https://marketingplatform.google.com/about/analytics/)のようなアクセス解析ツールが普及したり，そもそも CGI を使う機会が減ったり……といった背景からなのでしょうか，現在ではあまり見かけない代物です。[^ggr]もちろん，今でも CGI 環境の構築さえすれば，当然のことながら自作のカウンターを設置できます。

[^ggr]: それでも「[アクセスカウンター 無料](https://www.google.com/search?hl=en&q=%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%AB%E3%82%A6%E3%83%B3%E3%82%BF%E3%83%BC%20%E7%84%A1%E6%96%99)」と Google で検索すれば，アクセスカウンターを無料で提供してくれるサービスが数多く出てきます。今もなお，一定の需要はあるのかもしれません。

しかし，技術が発展した現代，アクセスカウンターの設置にあたって CGI は不要です。本記事では，[Google Apps Script](https://developers.google.com/apps-script) を用いた実装を試みます。なお，本文中に登場するコードは全て GitHub で公開しておりますので，よろしければご覧ください。[^source]

[^source]: 本当は TypeScript で書いたコードを公開しようと思ったのですが，わざわざ npm モジュールをインストールするのが億劫で断念しました。`.gs` ファイルは，拡張子を `.js` にしていますので，Clasp を使ってコマンドラインから直接アップロードすることは一応可能です。

https://github.com/Meiryo7743/web-counter

## 実装

今回作るアクセスカウンターは，以下の機能を持っています。

- **アクセス数の表示**：これまでのアクセス回数を `1234` のような文字列で表示する。これをしてアクセスカウンターたらしめるわけで，大変重要な機能。
- **キリ番判定**：アクセス数が `1000` や `1111` のような数値である場合，特別な表示にする。[^kiri]
- **複数ページに対応**：[Properties Service](https://developers.google.com/apps-script/guides/properties) を利用して，複数のカウンターを作れるようにする。例えば，サイト全体のアクセス数とは別にトップページへのアクセス数を計測したい……というような場合に有用。

[^kiri]: 一般に，キリ番には回文数や `123456789` のような値なども含まれます（[参考](https://kotobank.jp/word/%E3%82%AD%E3%83%AA%E7%95%AA-188422)）。ただ，それらを判定する処理も実装すると，コードが煩雑になりそうだったので潔く諦めました。

また，その大雑把な仕組みは次の通りです。

1. `<iframe>` から Web app として公開したスクリプトを呼び出す。
2. カウント処理が実行される。
3. 実行結果が HTML として返ってくる。
4. 実行結果が `<iframe>` 内に表示される。

## コード

アクセスカウンターの実装に必要なコードです。[Google Apps Script の新規プロジェクトを作成](https://script.new)し，以下の内容をコピー&ペーストするだけで出来上がります。なお，コメントなし版を GitHub に上げておりますので，ご所望でしたらぜひご利用ください。

### `doGet.gs`

```js:doGet.gs
const doGet = (e) => {
  const template = HtmlService.createTemplateFromFile("index.html");
  template.webCounter = webCounter(e.parameter.url);

  const output = template
    .evaluate()
    .addMetaTag("viewport", "width=device-width, initial-scale=1.0")
    .setTitle("Web Counter")
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);

  return output;
};
```

`doGet()` は，アクセス時に自動実行される特別な関数です。受け取った URL パラメーターを基に `webCounter()` を実行します。そして，その実行結果を HTML で返すという仕組みです。

### 設定（`config.gs`）

```js:config.gs
const config = () => {
  // 設置するアクセスカウンターの URL リスト。適当なものに書き換えてください
  const ALLOW_LIST = ["https://example.com"];

  const scriptProps = PropertiesService.getScriptProperties();
  scriptProps.setProperty("ALLOW_LIST", JSON.stringify(ALLOW_LIST));
};
```

`ALLOW_LIST` に登録された識別子が URL パラメーターで指定されたときだけカウント処理が走るようスクリプトプロパティーを設定します。`{"ALLOW_LIST": "["https://example.com"]"}` という形式で格納されています。

**識別子は設置場所の URL にすることを想定しています**（例：`https://example.com`，`https://example.com/dir/`）。複数のアクセスカウンターを置く際は，その数だけ識別子となる URL を配列内に追加してください。

なお，識別子に URL 以外の文字列を設定したい場合は，後述の `webCounter.gs` 内にある `isUrl()` の処理を削除してください。

### カウント処理（`webCounter.gs`，`index.html`）

```js:webCounter.gs
// 引数が URL かを判定
const isUrl = (url) => {
  const REGEXP = /^https?:\/\/([A-zd-]+.)+[A-z]+\/?(.+)*/g;
  return typeof url === "string" && REGEXP.test(url);
};

// キリ番判定
const isLuckyNumber = (integer) => {
  const REGEXP = /^(\d)\1+$|^\d0+$/g;
  return typeof integer === "string" && REGEXP.test(integer);
};

// カウント処理
const count = (integer) => {
  return typeof integer === "string" ? `${Number(integer) + 1}` : "1";
};

const webCounter = (url) => {
  const scriptProps = PropertiesService.getScriptProperties();
  const allowList = scriptProps.getProperty("ALLOW_LIST");

  // 引数が許可リスト内の URL に該当する場合にのみ，カウント処理が走る
  if (isUrl(url) && allowList.includes(url)) {
    const result = count(scriptProps.getProperty(url));
    scriptProps.setProperty(url, result);

    // アクセス数を返す。キリ番なら☆を両側に添える
    return isLuckyNumber(result) ? `☆${result}☆` : result;
  }

  // エラーメッセージを返す
  return Math.PI.toString();
};
```

アクセスカウンターの処理部分です。URL パラメーターの値が，前述の `ALLOW_LIST` プロパティー内のいずれかに該当する場合にのみアクセス数のカウントを行う仕組みです。アクセス数はスクリプトプロパティーに `{"https://example.com": "1234"}` の形式で保存されます。

万一，URL パラメーターが不正な値であった場合，デフォルトではエラーメッセージとして円周率 $\pi$ の近似値を返します。特に理由はありませんので，お好きな形に変更を加えてください。[^exp]

[^exp]: 例えば， $\pi$ ではなくネイピア数 $e$ を出力したい場合，`Math.exp(1).toString()` で $e$ の近似値が出ます。

```html:index.html
<!DOCTYPE html>
<html>
  <head>
    <!-- 適当なスタイルに書き換えてください -->
    <style>
      *,
      *::before,
      *::after {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
      }
      .web-counter {
        font-family: monospace;
        line-height: 1;
        text-align: center;
      }
    </style>
  </head>
  <body>
    <div class="web-counter"><?= webCounter ?></div>
  </body>
</html>
```

こちらは実行結果の表示部です。このままだと大変寂しい表示ですから，どうぞご自由に CSS を書き換えてお好みのデザインにしてください。

## 設置

ここからは Web app として公開し，`<iframe>` から呼び出すまでの流れを説明していきます。

### `config()` の実行

まず一番最初にすべきことは `config()` の実行です。`config.gs` を開き，エディター上部にある「Run」ボタンから，この関数を実行しましょう。そうしないと，`ALLOW_LIST` プロパティーの値が空っぽのままで，アクセスカウンターが一向に使えません。

![](https://storage.googleapis.com/zenn-user-upload/4hf57f6i7w43qphf1vmvsw4fp5fw) _`config()` の実行により，`ALLOW_LIST` という名のスクリプトプロパティーが登録される_

### デプロイ

1. エディターの右上にある「Deploy」ボタンを押下し，「**New deployment**」を選択。\
   ![](https://storage.googleapis.com/zenn-user-upload/06dfbf5u5qrtwavllhn8fxdykytm)
2. 「Select type」の右隣の歯車アイコンから，「**Web app**」を選ぶ。\
   ![](https://storage.googleapis.com/zenn-user-upload/a6uuno3wfdbi5wds132m8jtnrvjj)
3. 「Description」はテキトーに入力してください。「Execute as」は「**Me（Gmail アドレス）**」を，「Who has access」は「**Anyone**」を選択します。その後，下部にある「Deploy」ボタンを押しましょう。\
   ![](https://storage.googleapis.com/zenn-user-upload/eh8p3ie9m0zqt52h5ky2qsuqfamf)
4. これでデプロイが完了しました。「Web app」の下に表示されている文字列が公開ページの URL ですのでコピーしましょう。\
   ![](https://storage.googleapis.com/zenn-user-upload/5iuw8u95rzo3ese1pw44t67ohgd3)

### 表示の確認

先ほどの**手順 4** でコピーした公開ページの URL にアクセスします。すると，エラーメッセージ（$\pi$ の近似値）が表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/zl9s0trj4uv1m9tepbo0xizestid) _URL パラメーターがないため，エラーになる_

ここで，`?url={ALLOW_LIST の URL}` を末尾に付けてアクセスすると，下図のように「1」と表示されるでしょう。また，ページを更新すれば，そのたびに数字が 1 ずつ増えていきます。

![](https://storage.googleapis.com/zenn-user-upload/sp7rybdfjeovbvnjdqifp6qz2lqb) _このように表示されたら成功_

![](https://storage.googleapis.com/zenn-user-upload/326yge63vywxgit4qqhkcqfqb6ys) _キリ番に該当するときは，特別な表示になる_

### 埋め込み

さて，`<iframe>` による埋め込み表示でこのカウンターを呼び出してみましょう。設置したいページに，以下の HTML コードを挿入してください。`{DEPLOYMENT_ID}` には「Deployment ID」を，`{PARAMETER}` には `ALLOW_LIST` の URL を 1 つ当てはめます。

```HTML
<iframe frameborder="0" loading="lazy" scrolling="no" src="https://script.google.com/macros/s/{DEPLOYMENT_ID}/exec?url={PARAMETER}" title="アクセスカウンター"></iframe>
```

今回は以下の HTML ファイル（`embed.html`）を作成し，検証しました。

```html:embed.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>アクセスカウンターの埋め込み表示テスト</title>
    <style>
      body {
        background: black;
        block-size: 100vh;
        display: flex;
        margin: 0;
      }
      iframe {
        block-size: 1rem;
        border: none;
        inline-size: 10rem;
        margin: auto;
        overflow: hidden;
      }
    </style>
  </head>
  <body>
    <iframe
      frameborder="0"
      loading="lazy"
      scrolling="no"
      src="https://script.google.com/macros/s/{DEPLOY_ID}/exec?url={PARAMETER}"
      title="アクセスカウンター"
    ></iframe>
  </body>
</html>
```

上記ファイルをブラウザーで開くと，このような表示になります。

![](https://storage.googleapis.com/zenn-user-upload/0bbxb5i80wi0sek9rllf4ko4uqa7) _`<iframe>` からアクセスカウンターを呼び出した際の表示_

## 欠点

実を言うと Google Apps Script で実装したこのカウンターには，いくつかの**欠点**が存在します。

第一に，**アクセス数を表示するためには JavaScript を有効にする必要がある**という点です。試しにブラウザーの JavaScript を無効にした状態で公開ページを開いてみてください。まっさらなページが表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/a9yk7nxvafxf0ycc95pcb867qo76) _JavaScript が無効だと，一面に真っ白なページだけが表示される_

![](https://storage.googleapis.com/zenn-user-upload/37pusbxnw5tud4sq9t0dxgltpvro) _`<iframe>` で呼び出しても同様に数字が表示されない_

公開ページのソースコードを読むと分かるのですが，どうやらページ自体はクライアントの JavaScript で描画する仕組みのようです。とはいえど，カウント処理自体はサーバーサイドで実行されますので，アクセス数の計測には影響を及ぼしません。あくまでもクライアント側から実行結果が見えないというだけですので，実際のところ，それほど気にする必要はないでしょう。

第二に，**カウントの制限が一切不可能である**ということが挙げられます。例えば，「同一 IP アドレスから短時間に複数回のアクセスがあったらカウントの増加を抑制する」といった設定は Google Apps Script の仕様上，行えません。[^restrict]

[^restrict]: [Google Apps Script のドキュメント](https://developers.google.com/apps-script/overview)をざっとしか読んておりませんので，間違っていればご指摘いただきたく存じます。

そのほか，**ページの読み込み完了までの時間が幾ばくか長くなる**ことを確認しています。もともと `<iframe>` に遅延読み込みをさせていますが，それを加味しても何だか遅いような気がするのです。試しに Lighthouse で `embed.html` のパフォーマンスを計測してみたものの，特に指摘を受けなかったので，実は大した影響はないのかもしれませんが……。

## 終わりに

Google Apps Script を使い始めたばかりなのですが，色々なことができて楽しいですね。以前は，古臭い JavaScript の構文でコードをゴリゴリ書く必要がありましたが，2020 年 2 月頃から [V8 ランタイムによる実行](https://developers.google.com/apps-script/guides/v8-runtime)が可能になりました。これにより，ES6 の構文でコードを書けるようになっています。さらに，同年 12 月には[スクリプトエディターのアップデート](https://workspaceupdates.googleblog.com/2020/12/google-apps-script-ide-better-code-editing.html)も行われ，Visual Studio Code のような IDE に生まれ変わりました。このように，今の Google Apps Script は，以前と比べ大変書きやすくなっていますので，興味のある人はぜひ一度試してみてはどうでしょうか。

最後に，冒頭でも述べましたが，アクセスカウンターは，サイトにどれだけのアクセスがあったのかを把握するために設置するものです。しかし，**今現在，アクセス数を正確に計測したいのであれば，Google アナリティクスなどの解析ツールを導入するのが賢明でしょう**。本記事で作ったアクセスカウンターは，あくまでもお遊び程度のものとして捉えていただきたく存じます。
