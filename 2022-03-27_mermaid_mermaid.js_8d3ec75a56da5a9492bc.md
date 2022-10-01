<!--
title:   グラフィカルモデルをmermaid.jsを使って良い感じに書きたいんだけど...
tags:    mermaid,mermaid.js,グラフィカルモデル,ベイズ推定
id:      8d3ec75a56da5a9492bc
private: false
-->
# はじめに
最近、Bayesian Approachによる回帰モデルを活用する機会が有り、その際にグラフィカルモデルを描きたいと思ったんですが、どんなツールで書くべきか迷ってしまいました。
一方で世の中にはmermaid.jsというものがあり、「最近githubでmermaid記法を使えるようになった！」という話題をtwitterで目にしたので、「mermaid.jsを使えば楽にグラフィカルモデルを書けるのでは？」という考えに至りました。
本記事では、グラフィカルモデルをmermaid.jsで描く為に使えそうなtipsをまとめました：）
# グラフィカルモデルって例えばこんな感じ
例えばこんな感じのグラフィカルモデルを書きたいんです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/581af383-8bed-17e9-668b-84307e019830.png)

# mermaid.js でノードの形を変更する
括弧を使い別ける事で、ノードの形を変更できるようです。

```mermaid
graph TD
    a
    b[テキスト入りノード]
    c(丸括弧形ノード)
    d((円形ノード))
    e>非対称形ノード]
    f{ひし形ノード}
```

こんな感じになるようです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/a1c4e2aa-7f0e-fd75-f2a2-666723c45476.png)

なお、グラフィカルモデルにおいては、基本的には円形ノードを使っていきます。

# mermaid.js でノードの色を変更する

CSS の記法(?)が使えるようです。

```mermaid
graph LR
    start(x)-->stop(y)
    style start fill:#f9f
    style stop fill:#ccf
```

こんな感じになります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/693a8960-9b12-9c9b-da30-2154f1903635.png)

また、**ClassDef を使う事で、複数のノードに同一のテーマを適用できる**ようですね。`ノード名:::クラス名`と記述します。

なおグラフィカルモデルにおいては、観測されていない確率変数は塗りつぶさず、観測されている確率変数は塗りつぶして表します。なので、ClassDef で観測変数と見観測変数のスタイルをそれぞれ定義して、使い分けたいです。
(ちなみに既知の定数は塗りつぶした点で表します。)
なので例えば、

```mermaid
graph LR
    classDef observed fill:white
    classDef non-observed fill:#EEEEEEE
    x((X)):::non-observed -->y((y)):::observed
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/3d170fd2-8420-0470-c157-6035ede967f2.png)


# 繰り返しのノードを枠で囲む

サブグラフを用いれば、特定のノードを枠で囲める様ですね。
例えばこんな風に。

```mermaid
graph LR
    classDef observed fill:white
    classDef non-observed fill:#DDDDDD
    subgraph a[N]
        x((x)):::non-observed --> y((y)):::observed
    end
    z((z)):::non-observed --> y
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1697279/8d41ed61-eceb-5538-9f1a-2158006fb9df.png)

# おわりに
とりあえずこれで何とか、グラフィカルモデルっぽい図は書けそうな気がします。(気がするだけで、実際には決して最適解ではないと思います...!)
皆さんはグラフィカルモデルをどうやって書いてるのでしょうか？オススメのツール等があればぜひコメント等で教えていただければ嬉しいです：）
draw.io等で手書きすべきなのかー。
# 参考
- 【目的無しの泥臭調査⑤】mermaid.jsの記法を覚えて、楽しく図を描く。
  - https://qiita.com/t_o_d/items/ac5b04419252f768a535
- Mermaid のテーマ・スタイルの変更方法
  - https://zenn.dev/junkawa/articles/zenn-mermaidjs-theme-config
- CSS カラーコード
  - http://www.netyasun.com/home/color.html