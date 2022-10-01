<!--
title:   大学院生がMarkdownでスライド作りたくてQuarto触ってみた(みてる)
tags:    Markdown,Pandoc,Quarto,document,presentation
id:      e9b3a44c55e4169242f3
private: false
-->
# はじめに

本記事は、Power Pointが苦手な一人の大学院生がquartoと出会い、Document・発表スライド作りの楽しさを取り戻していく物語です。(本当は単に、公式Documentを読みながら、quartoに関するTipsや文法を備忘録を兼ねてまとめていく記事です。)

## Markdownで発表スライドを作りたい...

僕は大学での講義や研究活動における発表機会の際、Power Pointを使用して発表資料を作成していました。これは特にこだわりがあるのではなく、周りの人が全員Power Pointを使っており、それが当たり前だったからです。

最初のうちは何も気にせずPower Pointでスライドを作成していたのですが、研究活動でスクリプトをたくさん書くようになると、だんだんPower PointのGUIによる資料作成が非常に退屈で苦痛に感じてきました。
ただこのことはほぼ間違いなく個人個人の好みの問題でして、Power Pointによるスライド作成を否定しているわけではないんです：）Power Pointは誰もが直感的に操作しやすい、凄く素敵なツールだと思います...!
僕自身の場合はプログラミングが好きで楽しく感じており、かつ凝り性・心配性ゆえにPower Pointのピクセル単位でのオブジェクト調整に必要以上に時間を浪費していたので、もう嫌気が差していたんです。

脱Power Pointを志して以降、Markdown記法でのスライド作成方法をちょくちょく探していました([Marp](https://marp.app/)とか[Slidev](https://sli.dev/)とか！)。そして最近、「markdownによるDocument作成で最近イチオシなのはQuartoらしいよ！」というレコメンドをいただいたのが、僕とquartoとの出会い、そして本記事のきっかけになります！

## quartoとは？

quarto(クオルト?)とは、

> Quarto® is an open-source scientific and technical publishing system built on Pandoc
> らしいです。([公式](https://quarto.org/)より)

まあ僕は今のところ、ざっくり「Markdown記法を使ってDocumentを作り、それを.pdfだったり.htmlだったり.pptx等のスライド形式でも吐き出せるツール」の一つだと認識しています。

[公式のDocument](https://quarto.org/)が非常に丁寧で充実していると感じたので、英語の資料を読むことが苦でない方はぜひそちらを読んでみてください...!

コレ以降のSectionでは、QuartoでDocumentやスライドを作る上でのTipsや気づいた事をまとめていきます。

# Quartoの導入方法

ここに関しては、公式Documentの[Get Started](https://quarto.org/docs/get-started/)を参照。
もしくは、[@Nobukuni-Hyakutake様の記事](https://qiita.com/Nobukuni-Hyakutake/items/112a7bd5b34abd446395)でも丁寧にまとめられています。感謝...!

# QuartoによるPresentationスライド作成

## スライドのフォーマット

`.qmd`ファイルの先頭のyaml部分で、`format`オプションを以下の3つのうちどれかに指定すると、出力されるプレビューがスライド形式になるようです。

- `revealjs` — reveal.js (HTML)
- `pptx` — PowerPoint (MS Office)
- `beamer` — Beamer (LaTeX/PDF)

例えばこんな感じで。

```qmd
---
title: "demo presentation material"
author: "morinota"
format: revealjs
---
```

公式Documentが推奨するのは`revealjs`みたい。

## 各スライドの区切り方

Header を使う方法と、水平方向の罫線(`---`)を使う方法の2種類。

Header(見出し)を使ってスライドを区切る場合、lebel 1 heading(`#`)とlevel 2 heading(`##`)でスライドが区切られるみたい。

```md
---
title: "demo presentation material"
author: "morinota"
format: revealjs
---

# In the morning

## Getting up

- Turn off alarm
- Get out of bed

## Breakfast

- Eat eggs
- Drink coffee

# In the evening

## Dinner

- Eat spaghetti
- Drink wine

## Going to sleep

- Get in bed
- Count sheep
```

こんな感じで区切られる。
![2022-09-03-16-52-28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/f34fb3fe-6d6d-f5db-acd2-5116aa7005c4.png)

上の例では、各Section毎のTitleスライドとしてlevel 1 header(`#`)を、非Titleスライドとしてlevel 2 header(`##`)として使用している。
この設定はどうやら、slide-levelオプションを使ってカスタマイズできるみたい。

また、水平方向の罫線`---`を使う場合はこんな感じ。

```md
---
title: "Habits"
author: "John Doe"
format: revealjs
---

- Turn off alarm
- Get out of bed

---

- Get in bed
- Count sheepF
```

## Incremental Lists:

この機能は、何と言えばいいのか、「箇条書きの箇所が、ページ遷移時に一斉に表示されない。一つずつ表示させる。」ような機能といえば伝わるでしょうか。フェードインのアニメーションみたいな。
先頭のYAML部分で`incremental`オプションを`true`にすると、全ての箇条書きの箇所がIncremental(＝ページ遷移時に一斉に表示されない・一つずつ)になる。

```md
---
title: "demo presentation material"
author: "morinota"
format:
  revealjs:
    incremental: true
---
```

上の例だと全てのページの箇条書きがIncrementalになるが、該当する箇所の箇条書きのみを明示的にIncremental/Non-Incrementalと指定したい場合には、markdown記法の該当する箇条書きの箇所を、明示的にIncrementalクラス/Non-Incrementalクラスで囲む。

例えば以下のように。この場合、`## Getting up`ページの箇条書き箇所はIncrementalで、`## Breakfast`の箇条書き箇所はNon-Incrementalで表示される。

```md
## Getting up

::: {.incremental}

- Turn off alarm
- Get out of bed

:::

## Breakfast

::: {.nonincremental}

- Eat eggs
- Drink coffee

:::
```

## Multiple Columns
一つのスライド内でオブジェクトを横に並べて配置するには、`.columns`クラスを使ってdivコンテナを作り、その中に2つ以上の`.column`クラスを含める事で実現できます。

そして、`.column`クラスを定義する際には、`width`属性を記述しなければならないようです。
試しに`.columns`クラスで定義したコンテナ内の2つの`.column`クラスのうち、1つで`width`属性を記述しなかったところ、このMultiple Columns機能は反映されずに一列で表示される状態でした。

`md
## Multiple Columns Page

:::: {.columns}

::: {.column width="40%"}
contents...

- hoge
- hogehoge

:::

::: {.column width="60%"}
contents...

- piyo
- piyopiyo

:::

::::
`
作成されるスライドはこんな感じ

![2022-09-03-17-25-06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/77aa0b45-daff-de7c-9212-a14b1b1ce25a.png)

# おわりに

今後もQuarto公式Documentを読みながら、自分の資料作成に使いつつ、適宜追加していきたいです。
数日触ってみた感想ですが、「Marpよりも複雑なDocumentを作成可能」かつ「Slidevより記法が理解しやすい」気がしており、個人的には好感触です...!
興味を持っていただきありがとうございました：）

お互い難しくも楽しい研究者or技術者ライフを～！