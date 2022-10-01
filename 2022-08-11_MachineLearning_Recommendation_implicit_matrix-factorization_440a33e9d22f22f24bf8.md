<!--
title:   implicit-feedback MFにおけるALSの高速化に関して - 「Fast Matrix Factrization for Online Recommendation with Implicit Feedback」を読んで- 
tags:    MachineLearning,Recommendation,implicit,matrix-factorization,推薦システム
id:      440a33e9d22f22f24bf8
private: false
-->
# 1. はじめに

まず始めに本記事は、「Fast Matrix Factrization for Online Recommendation with Implicit Feedback」という論文を読み、内容を自分なりに噛み砕きつつまとめた＋実装してみたものになります。

この論文を読み始めた背景としてはいくつか理由があるのですが、主要な理由の一つが、この論文を読む事で私の感じた課題の解法が得られる、気がした事なんです。

というのも、私は一ヶ月程前からConvMF(Convolutional Matrix Factorization for Document Context-Aware Recommendation)の論文を読んで実装してみているのですが、その際にALSによるMatrix Factirizationの**ハイパーパラメータである$k$(latent vectorの次元数)を大きくすると、パラメータ推定の時間がめちゃめちゃ長くなってしまう**な...と感じていました。
$k$はMFの表現力を高める為に重要なハイパーパラメータなので、ある程度大きくすべき(ex. $k = 100 ~ 200 ?$)だし...。その一方で、新しく観測されたデータを使ってコンスタントにモデルを更新していくスピードも、推薦すべきアイテムに鮮度があるような推薦システムにおいては特に重要そうです...！
そしてConvMFでは、CNN(Convolutional Neural Network)のパートと、ALSによるMF(Matrix Factirization)のパートを交互に学習させるのですが、「CNNパートの学習はGPUを使うなり色々高速化の工夫がありそうだけど、MFパートの学習はどのような工夫をしたら高速化できるんだろう...」と思っていたところに、この論文と出会い、よし読もうとなった訳なんです。
(ちなみConvMFの論文ではExplicit Feedback、本論文ではImplicit Feedbackを扱っています。でもまあ、どちらの論文の手法も、もう一方の種類のデータにも応用できそうな気がします...!)

ちなみに本記事の作成者である私は、非情報系(理系＆MLや統計解析は研究で使ったりする！)の博士学生です。KaggleのPersonalized Recommendationコンペへの参加をきっかけに推薦システムに魅力を感じ、以降、ちょくちょく独学で論文や技術書を読んだり実装してみたりしています。暖かい目で見ていただきつつ、なにか気になった点があればぜひコメントいただけますと嬉しいです：）

# 2. Matrix Factrization(MF)に関する前置き

## 2.1. Implicit feedbackに対するMFの概要

まず以下に、基本的な表記法をまとめておきます。以下のsectionでは、この表記法に従って記述していきます。適宜、自分の感想が挟まっています...!

- $R \in \mathbb{R}^{M \times N}$:ユーザとアイテムのInteraction Matrix。
  - Explicitデータの場合はRating Matrixって呼ぶ?Implicit feedbackの場合はInteraction Matrix?
  - でも以前読んだ Hu et al. (Implicit feedback に対するALSによるMFの元祖(?)の論文)でもRating Matrixだったような気もする。...
- $M$:the number of users
- $N$:the number of items
- $\mathcal{R}$: the set of user-item pairs whose values are non-zero.
- $u, i$: indices of a user and an item.
- $\mathbf{p}_u$: the user latent vector for $u$
- $\mathcal{R}_{u}$: the set of items that are interacted by $u$.
- $\mathbf{q}_i$: the item latent vector for $u$
- $\mathcal{R}_{i}$: the set of users that are interacted by $i$.
- $P \in \mathbb{R}^{M\times K}$: the latent factor matrix for users.
- $Q \in \mathbb{R}^{N\times K}$: the latent factor matrix for items.

ざっくり言うと、「Interaction(Rating) Matrix $R$を元データとして、user latent matrix $P$とitem latent matrix $Q$を推定し、それらの行列積で得られる$\hat{R}$を用いてユーザのアイテムに対するPreference(嗜好度合い) を評価する」のがMatrix Factrizationですね！

```math
\hat{r}_{ui} = <\mathbf{p}_u, \mathbf{q}_i> = \mathbf{p}_u^T \mathbf{q}_i
\tag{1}
```

そしてImplicit feedbackに対するモデルパラメータ($P$と$Q$)を推定する為に、Hu et al. (確かこれが前に読んだやつ！)は、Interaction Matrix行列$R$の各予測に**重み（＝たしか、Hu et al.では信頼度Confidenceって呼んでたっけ...?**）を関連付ける、**weighted regression function**を(目的関数として！！)導入しています。

```math
J = \sum_{u=1}^M \sum_{i=1}^N w_{ui}(r_{ui} - \hat{r}_{ui})^2
+ \lambda(\sum_{u=1}^M ||\mathbf{p}_u||^2
+ \sum_{i=1}^N ||\mathbf{q}_i||^2)
\tag{2}
```

ここで、

- $w_{ui}$: the weight of entry $r_{ui}$. and we use $W=[w_{ui}]_{M\times N}$
- $\lambda$: the strength of regularizationをコントロールするハイパーパラメータ。

Implicit FeedbackのMFでは、通常、欠落したentriesには$r_{ui}$の値はゼロだが、**$w_{ui}$の重みはゼロでないものが割り当てられ**、どちらも性能にとって重要であることに注意、との事です。

## 2.2. ALS(Alternating Least Square)による目的関数の最適化(パラメータ推定)

Alternating Least Square(ALS)は、MFを最適化するための一般的なアプローチです。
ALSでは「他のパラメータを固定した状態で、一つのパラメータを最適化する」事を交互に繰り返し行います。つまり今回の場合は、目的関数$J$に対して、$P$と$Q$を交互に最適化していきます。
(更に細かく言えば、全てのuser latent vector $p_u$とitem latent vector $q_i$に対して、他のパラメータを固定して一つずつ最適化していきます！)

そして、user latent vector $p_u$に関して$J$を最小化することは、以下の目的関数$J_u$最小化することと等しいです。

```math
J_u = ||W^u (r_u - Q \mathbf{p}_u)||^2 + \lambda ||\mathbf{p}_u||^2
```

ここで、

- $W^u$は、$N \times N$のDiagonal matrix(対角行列)。
  - つまりその対角要素$W_{ii}^u = w_{ui}$

そして上の$J_u$を最小にするのは、一階微分=0となるような$p_u$です。

```math
\frac{\partial J_u}{\partial p_u}
= 2 Q^T W^u Q p_u - 2Q^T W^u r_u + 2\lambda p_u = 0 \\
\Rightarrow p_u = (Q^T W^u Q + \lambda I)^{-1} Q^T W^u r_u
\tag{3}
```

ここで、

- $I$は単位行列(Identity Martix)。

同じ手順に従って、$q_i$の解も得る事ができます。

### 2.2.1. ALSの最適化の計算効率に関するIssue

どうやらALSにおけるTime Complexityの重要なIssueは、「**user latent vector及びitem latent vectorのパラメータ更新をする為に、$K \times K$行列の逆行列の計算が避けられない**」ことにあるようです。
ここでの”$K \times K$行列の逆行列”というのは、上の式(3)の$(Q^T W^u Q + \lambda I)$のことですね。

**逆行列の計算はExpensiveな処理**であり、通常、Time Complexityは$O(K^3)$と仮定されるんですね。
その為、ある１ユーザのuser latent vectorを更新するには、時間$O(K^3 + N K^2)$を要します。
したがって、全てのモデルパラメータ(user latent matrix & item latent matrix)を一回更新する為に必要なTime Complexityは、$O((M+N)K^3 + MNK^2)$となります。
明らかに、この計算量の多さは、数百万のuserとitem、数十億のInteractionが存在し得る大規模データでこのアルゴリズムを実行する事を非現実的なモノにしている、ようです。
なるほどなるほど、逆行列の計算がボトルネックなのか...!

### 2.2.2. Uniform weightingを適用する事による高速化

Hu et alでは、高い計算量を削減する為に、missing entries(=Rating matrixのゼロ要素)に対してUniform weightingを適用しています。なわち、Rating Matrixの全てのゼロ要素は、同じ重み$w_0$を持つと仮定したって事ですね。

この工夫を使うと、user latent vectorの更新式(式3)の中の$Q^T W^u Q$は以下のように変形できます。

```math
Q^T W^u Q = w_0 Q^T Q + Q^T (W^u - W^0)Q \tag{4}
```

ここで、

- $W^0$ : 全ての対角要素が$w0$である対角行列(diagonal matrix)

そして$Q^T Q$は、任意の$u$とは独立なので、全てのuser latent vectorを更新する前に事前計算が可能です。
$W^u - W^0$が$|R_u|$個の非ゼロ要素しか持たない事を考慮すると、式(4)を$O(|\mathcal{R}_u| K^2)$で計算する事が可能になります。
$K^2$にかかる値がMから$|\mathcal{R}_u|$に変わって、ちょっと高速化されましたね！
（”ちょっと”と言いつつ、通常Interaction Matrixはスパース性が高いので、結構大きな変化ではありますね！）

＝＞したがって、ALSにおける全てのパラメータ一回更新にかかるtime Complexityは、$O((M + N)K^3 + |\mathcal{R}|K^2)$に低減されます！

それでも、逆行列の計算部分$O((M + N)K^3)$項は、$(M + N)K \geq |R|$の場合に主要なコストとなり得ます。
(**すなわちlatent vectorの次元数$K$が多くなればなるほど**...!!!**$K^3$だからグングン増えちゃうんですね...！**)

しかし**latent vectorの次元数$K$の大きさは、より良い汎化性、ひいてはより良い予測性能につながる為、非常に重要なハイパーパラメータ**ですから、なるべく$k$を小さくはしたくない。なので上記の手法によって高速化されたALSを持ってしても、大規模データでの実行はまだなかなか厳しい...。

加えて、Uniform weightingの仮定は実際のアプリケーションでは通常無効であり、モデルの予測性能を劣化させるらしく...。なので、Uniform Weightingに依存しない、効率的なImplicit MF法を設計すべき、と主張されています。

読んでいて思ったのですが、この工夫はImplicit feedbackに対するALSゆえの工夫ですね...!
確かExplicit Feedbackに対するALSの場合は、非ゼロ要素のみを用いて目的関数を作ってた気がするので！

# 3. Generic Element-wise ALS Learner

ここから、これまでのALSアプローチのボトルネックになっていた課題「uesr latent vectorとitem latent vectorの更新の際に発生する逆行列計算」をどうにかする工夫に入っていきます。

この課題への解決策として、「ベクトルレベルではなく、要素(Element)レベルでパラメータを最適化する」というのが、Element-wise ALS(eALS)の考えになります。（逆行列の計算を不要にしようって考え??）
なるほど...。これまでは各latent vectorに対して更新式を順番に適用していたところを、一つ解像度を細かくして、latent vectorの各要素に対して順番にパラメータ更新していこうってことか...!

eALSを実現するには「latent vector (user & item)の各要素(=論文ではlatent factorと呼んでいます)のパラメータ更新式」が必要です。それを導出する為に、まず目的関数である式（２）の微分を求めます。

```math
\frac{\partial J}{\partial p_{uf}}
= -2 \sum_{i=1}^N (r_{ui} - \hat{r}_{ui}^f)
+ 2 p_{uf} \sum_{i=1}^N w_{ui} q_{if}^2 + 2 \lambda p_{uf}
```

ここで、

- $\hat{r}_{ui}^f = \hat{r}_{ui} - p_{uf} q_{if}$.
  - 言い換えれば、ある一つのlatent factor (添字$f$)を含めない場合の予測値$\hat{r}$みたいな感じ...??

この微分式=0とすると、$p_{uf}$の解が得られます。

```math
p_{uf} = \frac{
    \sum_{i=1}^N (r_{ui} - \hat{r}_{ui}^f) w_{ui} q_{if}
}{
    \sum_{i=1}^N w_{ui} q_{if}^2 + \lambda
}
\tag{5}
```

上式と同様の手順で、item latent factorの解析解も得ることができます。

```math
q_{if} = \frac{
    \sum_{u=1}^M (r_{ui} - \hat{r}_{ui}^f) w_{ui} p_{uf}
}{
    \sum_{u=1}^N w_{ui} p_{uf}^2 + \lambda
}
\tag{6}
```

これで各latent factorを最適化するclosed-formな解法が与えられたので、ALSは共同最適(a joint optimum)に達するまで、全てのモデルパラメータに対して更新を繰り返し実行することができそうですね。
注目すべきは、更新式に「$K\times K$行列の逆行列計算」がなくなっている点ですね...!これがelement-wiseでパラメータ更新することのモチベーションですし！

## 3.1. Time Complexity

上述したように、要素レベルでパラメータ最適化を行う事で、サイズの大きい(expensiveな)行列の**逆行列の計算を回避**する事ができました。

＝＞これにより、一回の反復(=MFの全てのパラメータの一回更新)にかかる
Time Complexityは$O(M N K^2)$となり、$O(K^3)$の項を排除することで直接的にALSを高速化できます。

さらに、$\hat{r}_{ui}$を事前計算する事で、$\hat{r}_{ui}^f$を$O(K)$ではなく$O(1)$で計算する事ができます。
＝＞そのため、**一回の反復(=MFの全てのパラメータの一回更新)にかかる
Time Complexityは$O(M N K)$となり**、全てのuser × itemの評価値$\hat{r}_{ui}$を予測するのと同じになります。

# 4. おわりに

「implicit feedbackに対するALS手法の前置き」と「eALSによる高速化」の内容だけで結構長くなってしまったので、ここで一旦記事を分けようと思います。

このelement-wiseのパラメータ更新の工夫は、Explicit feedbackに対するALSにも適用できそうですね...!
ConvMF(Convolutional Matrix Factorization for Document Context-Aware Recommendation)の場合はどうなんでしょうか...。
少なくともuser latent vector側の更新式は通常のALSと同じなので適用できるとして、item latent vectorの更新式にはCNNの項が入ってくるので...、うーん。
でもまあ、結局は他のパラメータを固定して一つのパラメータを順番に最適化していく点は同じなので、item latent vector側にもelement-wiseなパラメータ更新を適用できそうな気がします。

次の記事では、論文のMain partである、eALSによって高速化された & Online学習が可能なMatrix Factrizationについて理論面をまとめ、試しに実装してみます...!
読んでいただきありがとうございました：）
何か指摘や感想がありましたら、ぜひコメントしていただければ嬉しいです：）