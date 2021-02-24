---
title: "CSS の疑似クラス「:target」だけを用いて脚注を半モーダル表示にする"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CSS", "Web"]
published: false
---

皆さんは Markdown で脚注を挿入した経験はありますか。実際のところ，脚注機能は Markdown のパーサー次第ですから，使ったことがない人もいらっしゃるかもしれません。ちなみに，脚注はこのように表示されます。[^footnote]

[^footnote]: 余談ですが，GitHub は Markdown の脚注表示に対応していません（2021 年 2 月現在）。

さて，脚注を含む Markdown をそれに対応したパーサーで HTML に変換すると，本文と脚注の関係が次のような形で表現されていると分かります。

#### Hugo（Goldmark）の場合

```html
<!-- 本文 -->
<p
  >脚注表示のテスト。<sup id="fnref:1"
    ><a href="#fn:1" class="footnote-ref" role="doc-noteref">1</a></sup
  ></p
>

<!-- （中略） -->

<!-- 脚注 -->
<section class="footnotes" role="doc-endnotes">
  <hr />
  <ol>
    <li id="fn:1" role="doc-endnote">
      <p>
        ここに脚注の内容が入ります。
        <a href="#fnref:1" class="footnote-backref" role="doc-backlink">↩︎</a>
      </p>
    </li>
  </ol>
</section>
```

#### Zenn の場合

```html
<!-- 本文 -->
<p
  >脚注表示のテスト。<sup class="footnote-ref"
    ><a href="#fn1" id="fnref1">[1]</a></sup
  ></p
>

<!-- （中略） -->

<!-- 脚注 -->
<section class="footnotes">
  <div class="footnotes-title">
    <img
      src="https://twemoji.maxcdn.com/2/svg/1f58b.svg"
      class="emoji footnotes-twemoji"
      loading="lazy"
      width="20"
      height="20"
    />脚注</div
  >
  <ol class="footnotes-list">
    <li id="fn1" class="footnote-item">
      <p
        >ここに脚注の内容が入ります。
        <a href="#fnref1" class="footnote-backref">↩︎</a></p
      >
    </li>
  </ol>
</section>
```

すなわち，本文中にある脚注番号が書かれたリンクをクリックすると，ページ下部にある対応箇所へジャンプする，というわけです。さらに，その脚注にある「↩︎」を押せば，本文へ戻ることもできます。また，脚注がリスト形式で表現されているのも特徴と言えるでしょう。書籍の体裁を参考にしたのでしょうか。なお，上記コードは省略箇所があるため分かりづらいですが，脚注は本文の後に続けて挿入されます。気になるのであれば，開発者ツールでこのページのソースコードを確かめてみてください。本当にそうなっています。

## リスト形式の落とし穴

ところで，この脚注の数が厖大化したらどうなるのか，先ほどのコードを基に少し考えてみましょう。既述の通り，脚注はリストで構成されていますから，当然そこに含まれる `<li>` の数は増えていきます。

![](https://storage.googleapis.com/zenn-user-upload/pdopaiecck9kttg8zuo3vw2747tm) _Hugo で脚注が 20 個あるページを生成した結果_

![](https://storage.googleapis.com/zenn-user-upload/w67da7lyslcl4ltq6v2r3tf42lfx) _Zenn で脚注が 20 個ある場合はこのような表示になる_

上記画像はその例ですが，どのように捉えたでしょうか。項目数が多く少しごちゃごちゃしていますね。幸いにも注釈の文字数が少ないため，著しい読みづらさこそ感じませんが，だからといって，いつでもこうとは限りません。それとは別に，本文へ戻ろうと思い，結果として別の脚注にある矢印を押してしまう可能性もあり得そうです。

## 「モーダルウィンドウ」という選択肢

そこで，より良い脚注の表示方法の 1 つとして，モーダルウィンドウによるアプローチを考えてみます。本文の脚注リンクを押下すると，対応する注釈だけがモーダルウィンドウで表示される，という仕組みです。実際に，Wikipedia のモバイル版サイトがそのような実装をしています。

![](https://storage.googleapis.com/zenn-user-upload/p23q3c8q9r57v4o4ywu9ujnolgwk) _[Wikipedia モバイル版](https://ja.m.wikipedia.org/wiki/%E3%82%A6%E3%82%A3%E3%82%AD%E3%83%9A%E3%83%87%E3%82%A3%E3%82%A2)で注釈をクリックすると，画面下部にモーダルウィンドウが現れる_

このような別表示にすることで，他の脚注と読み間違えたり，本文の違うところへ遷移したりする可能性を排除できます。ただ，そのためには JavaScript を必要としたり，併せて HTML 自体を改修したりしなければなりません。場合によっては，少々骨が折れることになりそうです。

ところが，わざわざそんなことをせずとも，CSS を 20 行程度書くだけでこれを実現できるのです。百聞は一見に如かずと言いますから，今すぐブラウザーの開発者ツールを開き，以下の CSS を貼り付けて脚注を参照してみましょう。[^result]

[^result]: 例の CSS を貼り付けたのなら，この脚注がモーダルウィンドウ内に表示されているはずです。

```css
/* Zenn の脚注をモーダルウィンドウ内に表示する */
.footnote-item:target {
  background: var(--c-base-bg);
  bottom: 0;
  box-shadow: 0 0 0 100vmax rgba(0, 0, 0, 0.5);
  inline-size: 100%;
  left: 0;
  padding: 60px !important;
  position: fixed;
  z-index: 10;
}
.footnote-item:target .footnote-backref {
  block-size: 200%;
  inline-size: 100%;
  left: 0;
  position: fixed;
  top: -50%;
  z-index: -10;
}
```

すると，下記の画像のように脚注が出てくるはずです。

<!-- 画像 -->

この状態で暗部をクリックするとウィンドウが閉じられ，元通りになります。

## `:target` の秘める可能性

なぜ，あの CSS だけでモーダルウィンドウ表示ができるのでしょうか。そのカギは `:target` という疑似クラスにあります。

### `:target` とは何か

[MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/:target) によると，`:target` というは次の役割を持つ疑似クラスであると説明されています。

> The :target CSS pseudo-class represents a unique element (the target element) with an id matching the URL's fragment.
>
> 拙訳）CSS の `:target` 疑似クラスは，URL フラグメントの ID と一致する唯一の要素（対象要素）を表す。

また，`:target` は CSS3 のセレクターであり，[ほとんどのブラウザーでサポートされています](https://caniuse.com/mdn-css_selectors_target)。

![](https://storage.googleapis.com/zenn-user-upload/d104txpmeh1z1kevr4hozngryaos) _あの IE でさえ `:target` に対応している_

### 仕組み

先ほどの脚注［2］を **アドレスバーに注目しつつ**，もう一度クリックしてみてください。URL の末尾に **`#fn:2`** があったはずです。

ここで，冒頭にある Zenn の脚注部分の HTML コードと先の CSS を再度示します。

```html
<!-- 本文 -->
<p
  >脚注表示のテスト。<sup class="footnote-ref"
    ><a href="#fn1" id="fnref1">[1]</a></sup
  ></p
>

<!-- （中略） -->

<!-- 脚注 -->
<section class="footnotes">
  <div class="footnotes-title">
    <img
      src="https://twemoji.maxcdn.com/2/svg/1f58b.svg"
      class="emoji footnotes-twemoji"
      loading="lazy"
      width="20"
      height="20"
    />脚注</div
  >
  <ol class="footnotes-list">
    <li id="fn1" class="footnote-item">
      <p
        >ここに脚注の内容が入ります。
        <a href="#fnref1" class="footnote-backref">↩︎</a></p
      >
    </li>
  </ol>
</section>
```

```css
/* Zenn の脚注をモーダルウィンドウ内に表示する */
.footnote-item:target {
  background: var(--c-base-bg);
  bottom: 0;
  box-shadow: 0 0 0 100vmax rgba(0, 0, 0, 0.5);
  inline-size: 100%;
  left: 0;
  padding: 60px !important;
  position: fixed;
  z-index: 10;
}
.footnote-item:target .footnote-backref {
  block-size: 200%;
  inline-size: 100%;
  left: 0;
  position: fixed;
  top: -50%;
  z-index: -10;
}
```

各々の脚注項目（`.footnote-item`）には固有の ID が与えられており，必ず本文へ戻るためのリンク（`.footnote-backref`）を子要素として含んでいます。同様に，本文中の脚注番号とそこへのリンクを含む要素（`.footnote-ref`）にも脚注側から戻れるよう，固有の ID が振られています。

本文中にある脚注番号（`.footnote-ref > a`）をクリックすると，その脚注の URL フラグメントを参照します。その結果，当該の `.footnote-item` が `:target` 疑似クラスにマッチするわけです。反対に `.footnote-backref` をクリックする（脚注から本文へ戻る）際には別の ID を参照するため，脚注は `:target` の対象とみなされなくなります。ゆえに，上記の CSS を適用するだけで脚注のモーダルウィンドウの表示・非表示ができたのです。

### 既知の問題

ただ，CSS の `:target` 疑似クラスを用いたモーダルウィンドウには，少し厄介な点があります。自分が見つけた範囲では次の通りです。もしも妙案がありましたら，ぜひともお教えください。

- 表示中は CSS だけで `<body>` のスクロールを無効化できない。ただし，将来的に [`:target-within`](https://developer.mozilla.org/en-US/docs/Web/CSS/:target-within) が主要ブラウザーで実装された際は，その限りでないと思われる。
- ウィンドウを閉じるリンクの判定が，脚注の子要素以外の全域に及んでしまう。つまり，**ウィンドウの余白部分をクリックしても閉じられる**。
- 本文へ戻る際の挙動がややぎこちない。一応，`scroll-behavior: smooth;` を `<html>` に適用することで対応可能。

## 終わりに

終わりに，事の発端は，MDN の『[CSS reference](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference)』を眺めていたときのことでした。そこで `:target` の存在をたまたま知り，その面白さと多くのブラウザーがサポートしていることから，本記事を書いた次第です。
