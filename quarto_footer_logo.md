<!--
title:   Quarto(Reveal.js)でスライドのロゴとフッターをいい感じに設定できたメモ
tags:    Quarto
id:      c2a37c409f513a5aba0e
private: false
-->


# はじめに

僕は大学院生なのですが、研究報告や輪講等のスライド作成において脱PowerPointを目指し、最近Quarto(Reveal.js)でのスライド作りを試みています。
「一体なぜそんな事を...」と思われる方もいるかもしれません。僕の脱パワポのモチベーションに関しては[前記事](https://qiita.com/morinota/items/e9b3a44c55e4169242f3)の導入パートに記述しているので、知りたい場合はぜひそちらを読んでいただけると幸いです:)

さて本記事の内容ですが、Quarto(Reveal.js)でのスライド作りにおいて、**ロゴとフッターをいい感じに設定する**際に調べた事や詰まった事をまとめたものになります。
「なぜロゴとフッターを細かく設定する必要があるのか」という事なんですが、研究発表の際には研究室テンプレートの使用が推奨されていまして、そのテンプレートには大学のロゴなり研究室ホームページのURLなりが記載されているんです。そのテンプレートをQuarto(Reveal.js)でもできる限り再現しておきたいな、という事なんです...!

まだQuartoはリリースされて日が浅いからか、日本語の記事やDocumentが多くありません。また、ロゴとフッターの細かい指定方法については、少なくともQuartoの公式Documentでは書かれていない(?)ので結構苦労したんです...。なので本記事が、少しでも脱PowerPoint同志の役に立てば幸いです：）

# まず完成イメージ!

まず初めに、「こんな感じのスライドを作りたいんだ！」という完成イメージを共有しておきますね。
今回は以下のような感じのスライド作成を目指します。
![](https://pbs.twimg.com/media/FdrIaYtUAAA7ZFq?format=jpg&name=medium)

左上に大学ロゴ(今回は試しにN大学のロゴを使用してみました)、フッターは左詰めでURLを表示しinline colorは緑ですね。
また、スライドの背景はとりあえず何も考えず黒にしています。
(本当はLinear Gradientにしたいのですが、まだ設定方法にたどり着いておらず...。もしご存知の方がいましたらぜひコメント等で教えていただけますと嬉しいです...!)

# まずQuarto(Reveal.js)でスライドを作る.

本記事のメインパートは**ロゴとフッターの設定**なので、とりあえずスクリプトだけ...。
気になった方は[Quartoの公式Document](https://quarto.org/)や[前記事](https://qiita.com/morinota/items/e9b3a44c55e4169242f3)等を読んでいただければと思います。

```md:presentation.qmd
---
format:
    revealjs:
        logo: logo_nagoya_university.png
        footer: "⇒ hogehoge/hoge/hogehogehoge/"
---

# Piyo Piyopiyo
-hugahuga huga-

モーリタ
```

`yaml`部分で`reveal.js`オプション以下に`logo`オプションと`footer`オプションを記述すると、各スライドの下部にフッターテキストとロゴを含めることができます。デフォルトだと以下のようになりますね。

![](https://pbs.twimg.com/media/Fd-0TrFUUAA5Xf_?format=jpg&name=900x900)

ここから、ロゴのサイズ＆位置、フッターテキストのalignを中央寄せから左寄せに、またスライド背景とフッターの背景色を指定していきます...!

# いざカスタム！

デフォルトの出力結果に上述した変更を加える為に、Customized Themeを作成し適用していきます。

## Customized Themeの作り方

Quartoでは現状以下の11個のbuild-in themeを利用可能です。`reveal.js`オプションの下の階層に`theme`オプションを指定すればOKです。

- `beige`
- `blood`
- `dark`
- `default`
- `league`
- `moon`
- `night`
- `serif`
- `simple`
- `sky`
- `solarized`

そして、ユーザ自身のCustomized Themeも使用する事ができるようです。
その際には、独自に`Sass`テーマファイルを作成する必要があります。
ここでCSSをほとんど書いた経験のない僕は、`Sass`って何？？となってしまいました...。

軽くGoogleしてみた感じ、Sass(Syntactically Awesome StyleSheet)は、**CSSを拡張したメタ言語**(=ある言語について何らかの記述をするための言語)らしいです。CSSをより効率的にコーディングできるようにした言語、みたいな感じみたいです...!
(まあそもそも僕はCSSをほとんど書いたことがないので、あまりイメージつかないのですが笑)

(以下を参考にさせていただきました...!)

- [Black Lives Matter](https://sass-lang.com/)
- [これからはcssはSassで書こう。](https://qiita.com/m0nch1/items/b01c966dd2273e3f6abe)

`.sass`記法(インデントで依存関係を表す。Pythonっぽい?)と`.scss`記法(`{}`で依存関係を表す。CSSの書き方)の２種類があるらしく、今回は後者の`.scss`記法を使用してみます。

```css:custom.scss
/*-- scss:defaults --*/
// 以下でフォント、色、ボーダーなどに影響する変数を定義する

/*-- scss:rules --*/
// 以下でCSSルールを作成する

.reveal .slide-logo {
    // ロゴの設定
}

.reveal .footer {
    // フッターの設定
}

```

- `.scss`ファイルのコードの意味：
  - `/*-- scss:defaults --*/` は、フォント、色、ボーダーなどに影響する変数を定義するために使用します
    - 変数は `$` で書き始める。
    - [カスタマイズ可能な変数の一覧](https://quarto.org/docs/presentations/revealjs/themes.html#sass-variables)
  - `/*-- scss:rules --*/` は、CSSルールを作成するために使用します。

作成した`custom.scss`を`.qmd`ファイルと同じ階層に保存し、`.qmd`側での`theme`オプションを以下のように指定します。

```yaml
---
format:
  revealjs:
    theme: [default, custom.scss]
---
```

## 実際にCustomized Themeを作っていく

カスタマイズ目的(ロゴのサイズ＆位置、フッターテキストのalignを中央寄せから左寄せに、またスライド背景とフッターの背景色を指定)に合わせて、`custom.scss`ファイルに追記していきます。

まず`/*-- scss:defaults --*/`にて、スライド全体の背景色を指定します。
`$body-bg`変数にカラーコードを代入する事で指定できました○

```css:custom.scss
/*-- scss:defaults --*/

$body-bg: #191919;

// 略
```

続いて、`/*-- scss:rules --*/`にて、ロゴとフッターの設定を記述していきます。

/_-- scss:rules --_/にて、`.reveal .slide-logo `ブロック内にロゴの設定を、`.reveal .footer `ブロック内にフッターの設定を記述していきます。
`.reveal .slide-logo `ブロックにはロゴの位置(`top`, `left`)、ロゴサイズ(`max-height`,`height`, `width`)を指定しています。
`.reveal .footer `ブロックにはフッターのフォントサイズ(`font-size`)、左寄せ(`text-align`)、テキストの背景色(`background-color`)を指定しています。

```css:custom.scss
// ロゴの設定
.reveal .slide-logo {
  // position
  top: 0;
  left: 12px;
  // size
  max-height: 3rem;
  height: 100%;
  width: auto;
}

// フッターの設定
.reveal .footer {
  font-size: 0.35em;
  text-align: left;
  background-color: #48fa42;
}

```

記述した`custom.scss`を`presentation.qmd`ファイル内で指定し、カスタマイズを反映させてみます。

![](https://pbs.twimg.com/media/Fd-9QvFVUAI21Al?format=jpg&name=900x900)

スライド背景色、ロゴの位置、フッターテキストの背景色の変更が想定通りに反映されています。
一方で、ロゴのサイズとフッターテキストの左寄せが反映されていません...!

## `!important`フラグを指定する！

実はロゴのサイズ`max-height`とフッターテキストの`text-align`は、別の場所でもデフォルト値が指定されている(?)らしく、単にCustomized Themeで値を指定しただけでは変更が反映されないようです。

実は、僕はここの対応にかなり詰まってしまいました。最終的には[GithubのIssues](https://github.com/quarto-dev/quarto-cli/issues/746)にて解決策と出会う事ができました。先人たちに感謝...!

解決策ですが、Customized Themeファイル内にて`!important`フラグを指定する事でした！
この`!important`フラグによって、指定した内容が「どのような場合でも使用するルール」としてみなされるようです。

最終的には、以下のように`custom.scss`ファイルを修正しました。

```css
/*-- scss:defaults --*/

$body-bg: #191919;
$body-color: #fff;
$link-color: #42affa;

/*-- scss:rules --*/

.reveal .slide blockquote {
  border-left: 3px solid $text-muted;
  padding-left: 0.5em;
}
// !importantを使用して、どのような場合でも使用するルールとしてフラグを立てる。
// ロゴの設定
.reveal .slide-logo {
  // position
  top: 0;
  left: 12px;
  // size
  max-height: 3rem !important;
  // max-height: 3rem;
  height: 100%;
  width: auto;
}

// フッターの設定
.reveal .footer {
  font-size: 0.35em;
  text-align: left !important;
  // text-align: left;
  background-color: #48fa42;
}
```

結果として以下のように、完成イメージのスライドが作成できました...!
`!important`フラグを追記した事で、左上のロゴサイズとフッターテキストの左寄せの変更が想定通りに反映されています。

![](https://pbs.twimg.com/media/FdrIaYtUAAA7ZFq?format=jpg&name=medium)

# 終わりに

本記事では、Quarto(Reveal.js)でスライドのロゴとフッターをいい感じに(=想定どおりに)設定した際に詰まった事をメモとして投稿しました。
これで脱PowerPoint、Quartoによる快適なスライド作成に少し近づいた気がします...!
最後までお読みいただきありがとうございました：）