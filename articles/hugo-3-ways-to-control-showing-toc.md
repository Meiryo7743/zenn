---
title: "Hugo で目次の表示を制御する 3 つの方法"
emoji: "📄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hugo"]
published: false
---

[Hugo](https://gohugo.io) は，[v0.6.0](https://gohugo.io/news/0.60.0-relnotes/) からデフォルトの Markdown パーサーがそれまでの『[Blackfriday](https://github.com/russross/blackfriday)』から『[Goldmark](https://github.com/yuin/goldmark)』へ変わりました。[^hugo-version]これにより，Goldmark の持つ柔軟性を生かし，[Markdown の出力形式をカスタマイズ](https://gohugo.io/getting-started/configuration-markup/)できるようになりました。

[^hugo-version]: 2021 年 3 月 18 日現在の最新版は [v0.81.0](https://gohugo.io/news/0.81.0-relnotes/) です。

その一方で，目次（`.TableOfContents`）の出力については，いささか微妙な挙動になってしまったことも事実です。

Blackfriday では [with 構文](https://gohugo.io/functions/with/)を用いることで，見出しの有無に応じてその表示を制御できます。これは，見出しがない場合の `.TableOfContents` の値が空っぽだからです。

```go
{{/* Blackfriday で目次を自動表示する */}}

{{- with .TableOfContents -}}
  {{- . -}}
{{- end -}}
```

ところが，新しい Markdown パーサーではそうはいきません。上記コードのやり方では，以下の HTML が必ず挿入されてしまいます。

```html
<nav id="TableOfContents"></nav>
```

目次の出力機能を利用する際は，そのままの状態ではなく，HTML で目次をラップしたり，CSS を適用したり……といったカスタマイズを施すでしょう。特に，目次をアコーディオン化している場合，空っぽの状態だとクリックしても何も起こらず，少々間抜けなことになってしまいます。

先述の Markdown の出力カスタマイズ機能の追加然り，Hugo 的には Goldmark を全力で推したい意図が伺えます。そこで，Goldmark でも見出しの有無に応じて目次を表示できるようにする方法を考えてみます。

## その 1：CSS

手っ取り早い方法は CSS を適用することです。先ほど示したように，空っぽな目次では `TableOfContents` という ID が振られた `<nav>` 要素の空タグが生成されます。そのため，[`:empty` セレクター](https://developer.mozilla.org/en-US/docs/Web/CSS/:empty)を組み合わせれば容易にこれを隠せます。

```css
#TableOfContents:empty {
  display: none !important;
}
```

### 長所

- 簡単に実装できる。
- 特に，`#TableOfContents` に対して直接 CSS を適用している場合に有用である。

### 短所

- `#TableOfContents` の親要素は非表示にできない。
- HTML のコード自体は残ったままである。

## その 2：Front matter

front matter に目次表示に関するパラメーターをセットする手段も考えられます。

```md
---
toc: true # 目次を表示する
---

# 見出し

テキスト
```

```go
{{/* front matter にある toc の値が true のときに目次を表示する */}}

{{- with .Page.Params.toc -}}
  {{- $.TableOfContents -}}
{{- end -}}
```

### 長所

- Blackfriday のときと同様の処理が行える。
- 副次的な効果として，目次を任意で出力できるようになる（見出しがある場合でも意図的に非表示にできる）。

### 短所

- 目次の制御を手動でする破目になり，面倒である。
- 全 Markdown ファイルの front matter にこれを追加する必要がある。

## その 3：文字列の比較

以下のコードは `.TableOfContents` の値が，見出しが皆無な場合のそれと一致するかどうかを判定する方法です。つまるところ，不一致の場合にだけ目次を吐き出す処理を行うようにするわけです。

```go
{{- $TOC_EMPTY := "<nav id=\"TableOfContents\"></nav>" -}}

{{- if ne .TableOfContents $TOC_EMPTY -}}
    {{- .TableOfContents -}}
{{- end -}}
```

### 長所

- 構文こそ異なるが，Blackfriday のときと全く同じ処理が行える。
- 見出しの有無に応じて自動的に目次を出力してくれる。

### 短所

- 特になし。

## 終わりに

Hugo で目次の制御法を 3 つご紹介しました。それぞれ特徴を持っていますが，個人的におススメしたいのは一番最後の方法です。調べた限りでは，このやり方に言及しているページは見つかりませんでしたが，もっとも良い手段だと考えています。
