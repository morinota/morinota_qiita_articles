<!--
title:   初学者がCollaborative Filtering for Implicit Feedback Datasetsを読んでスクラッチ実装してみる
tags:    ALS,Python,Recommendation,implicit,matrix-factorization
id:      9bd9f6ebff13d962d75b
private: false
-->
# 概要

最近Personalized Recommendationに少しハマりつつある初学者の学生が、Collaborative Filtering for Implicit Feedback Datasetsを読み理論を理解し、Pythonのnumpyでスクラッチ実装してみました、という記事になります。

僕と同じように、「ALSによるMatrix Factorizationってこういう事なのね！あれ、でもこれはExplicitデータに対する数式だよね？じゃあImplicitデータに対してはどうやるんだ－？あれ？パッと調べても出てこない...」という状況に落ちいった方の助けになれば嬉しいです。

拙い記事かもしれませんが...シューマイ！！

# はじめに

前回、レコメンドエンジンにおける一手法の「協調フィルタリング」、の一手法である行列分解(Matrix Factorization)について自分なりにまとめました。また、行列分解のアルゴリズムの1つであるALS(Alternate Least Squares)についても理論をまとめ、Pythonのnumpyを使ってスクラッチ実装してみました。

しかし前回まとめたALSアルゴリズムによるMFは、あくまでExplicitデータに対する手法でした。

今回は、**Implicit データの評価行列に対する行列分解**について理解する為に、Hu, Korenand and Volinsky による **Collaborative Filtering for Implicit Feedback Datasets**を読み、Pythonアプローチをスクラッチ実装してみたいと思います。

ちなみに、私が現在参加しているKaggleコンペ "H&M Personalized Fashion Recommendations"ではImplicitデータが提供されています。

調べてみた所感としては、Explicitデータに対するALSとは異なり、**Implicitデータに対するALSの日本語の技術記事はかなり少ない**ように感じたので、この記事の需要もゼロではないのでは...と信じて頑張って書きます：）

# Matrix FactrizationにおけるExplicitデータとImplicitデータの違い

explicit(明示的)データは、ユーザ自身が作成した各アイテムの**直接的な(明示的な)**評価データを意味し、例としては「星1~5の評価」や「Good or Bad」ボタン等が挙げられます。
対してimplicit(暗黙的)データは、ユーザ行動の受動的な追跡に基づいて決められる、**間接的な**評価データを意味します。例としては「閲覧」や「クリック」「購入」等が挙げられます。

Matrix FactrizationにおけるExplicitデータとImplicitデータの違いは、ズバリ、評価行列内の**ゼロ要素の意味合いが異なる**事です。

Explicitデータの場合、評価行列内の**ゼロ要素は「ユーザがまだ接触(評価)していないアイテム」**として扱う事ができます。
一方でImplicitデータの場合、評価行列内の**ゼロ要素は「ユーザがまだ接触していないアイテム」と「ユーザが接触しているが嗜好に合わなかったアイテム」の両方の可能性を含む**ので、Explicit データと同様に扱う事はできません。そこで、別のアプローチが必要になります！

なので、ExplicitデータとImplicitデータに対するMatrix Factrizationでは、「評価行列(疎行列)に含まれる**ゼロ要素**をどのように処理するか」が異なるようです。

# Implicit データにおけるALSの理論

今回は Hu, Korenand and Volinsky による Collaborative Filtering for Implicit Feedback Datasetsで概説されているアプローチ（FacebookとSpotifyで使用されてるっぽい？）をまとめます。
このアプローチでは、あるユーザのあるアイテムに対する嗜好度(Preference)$p$と、その嗜好に対する信頼度(Confidence)$c$を用います。

## 嗜好度(Preference) p

Preference は以下のように設定されます。

$$
p_{ui}(r_{ui}) =
    \begin{cases}
        {1 \ (r_{ui} > 0)}\\
        {0 \ (r_{ui} =0)}
    \end{cases} \tag{1}
$$

基本的に Preference は、**評価行列 r のバイナリ表現**になります。

## 信頼度(Confidence) c

またConfidenceは、$r_{ui}$の大きさを用いて、以下の様に計算されます。

$$
c_{ui} = 1 + \alpha r_{ui} \tag{2}
$$

例えばユーザがアイテムを再生、閲覧、クリックした回数が多い程、Confidenceは大きくなります。
また、1 を加えているのは、$\alpha r_{ui}$が 0 であっても、Confidenceの最小値が 0 にならないようにする為?
$\alpha$の値は、ユーザとアイテムのインタラクションが一回しかなくても、未知のデータ(ゼロ要素)よりもConfidenceが高くなる事を意味しています。(論文中では 40)

## ALS における目的関数

Implicitデータにおける行列分解Matrix Factorizationでは、上述したPreferenceとConfidenceを用いて評価行列$R$を変換している為、ALSにおける目的関数も少しExplicitデータとは異なります。

$$
\min_{x_u, y_i} \sum_{u, i} c_{ui}(p_{ui} - x_u^T y_i) + \lambda (\sum_u ||x_u||^2 + \sum_i ||y_i||^2) \tag{3}
$$

第二項は正則化項ですね。
第一項に関しては、Explicit データの場合は評価行列の要素$r_{ui}$が直接使われていた所が、Implicit データの場合は Preference（$p_{ui}(r_{ui})$）に置き換わっています。
また、各要素を Confidence で重み付けしているようですね...!
ここで、$\mathbf{X} \in \mathbb{R} ^{m \times k}$はユーザ行列、$\mathbf{Y} \in \mathbb{R} ^{n \times k}$はユーザ行列を意味します。

ALS ではこの目的関数を X と Y を交互に更新して最小化していきます。

## 更新式

ユーザ行列 X とアイテム行列 Y の各列$x_u$と$y_i$の更新式は以下のようになります。（線形回帰の最小二乗法の推定量と似てますね！たぶん目的関数を一回微分して＝０となるパラメータを求めたら、導出できるような気がします...!）

$$
x_u = (Y^T C^u Y + \lambda \mathbf{I})^{-1} Y^T C^u \cdot p(u) \tag{4}
$$

$$
y_i = (X^T C^i X + \lambda \mathbf{I})^{-1} X^T C^i \cdot p(i)
\tag{5}
$$

ここで、

- Xはユーザ行列(m×k)。$x_u$はX中のu行目を転置した列ベクトル.
- Yはアイテム行列(n×k)。$y_i$はY中のi行目を転置した列ベクトル.
- $C^u$は、ユーザuの各アイテムにおけるConfidenceが格納された、$n\times n$の対角行列。
  - 対角要素は、ユーザuのアイテムiに対するConfidenceを示す。
  - すなわち、$C^u_{ii} = c_{ui}$となる。
- $C_i$は、アイテムiの各ユーザにおけるConfidenceが格納された、$m \times m$の対角行列。
  - 対角要素は、ユーザuのアイテムiに対するConfidenceを示す。
  - すなわち、$C^i_{uu} = c_{ui}$となる。
- $p(u)$ はユーザuの全アイテムのPreferenceベクトル$\mathbb{R}^n$。
- $p(i)$はアイテムiの全ユーザのPreferenceベクトル$\mathbb{R}^m$。

## 更新式における計算量削減の工夫

更新式中の$Y^T C^u Y$と$X^T C^i X$をそれぞれ、以下の様に分解すると...

$$
X^T C^i X = X^T C^i X + X^T X - X^T X \\ = X^T  X + X^T(C^i -\mathbf{I})X
$$

$$
Y^T C^u Y = Y^T C^u Y + Y^T Y - Y^T Y \\
= Y^T Y + Y^T(C_u - \mathbf{I})Y
$$

これらを更新式に代入すると...

$$
x_u = (Y^T Y + Y^T(C_u - \mathbf{I})Y + \lambda \mathbf{I})^{-1} Y^T C^u \cdot p(u) \tag{6}
$$

$$
y_i = (X^T  X + X^T(C^i -\mathbf{I})X + \lambda \mathbf{I})^{-1} X^T C^i \cdot p(i) \tag{7}
$$

ここで、更新式中の$Y^T Y$と$X^T X$は**ユーザインデックス u とアイテムインデックス i に依存しない**為、**事前に計算**する事ができ、**計算量を削減**する事ができます。

上記の2つの更新式を交互に繰り返し計算する事で、$P \approx X ^T \cdot Y = \hat{P}$を満たすようなアイテム行列とユーザ行列を推定する事ができます。
(Explicit データの場合は評価行列を直接近似するような$R \approx X ^T \cdot Y = \hat{R}$を推定していましたよね。)

どうやらImplicitデータにおけるALSを用いたMatrix Factorizationでは、**Preference 値を用いて評価行列を Binary 値に変換**する事と、**目的関数において Confidence 値を用いて各要素の重み付け**をしている点が、Explicitデータの場合と異なるようです。

## 推定後のレコメンド方法について

ALS によってユーザ行列とアイテム行列を推定した後、以下の式で、あるユーザuのあるアイテムiに対するPreference(=すなわち、嗜好度合い)の推定値$\hat{p_{ui}}$を得る事ができます。

$$
\hat{p_{ui}} = \mathbf{x}_u^T \cdot \mathbf{y}_i \tag{8}
$$

レコメンドの際は、$\hat{P}$ のユーザuの行においてPreference値が高いk個のアイテム、をユーザuに推薦する事になるようです。

## 予測結果(レコメンド結果)の解釈性

良いレコメンデーションには、「**なぜその商品をユーザに推薦したのか**」という簡単な説明や解釈ができる事が重要なようです。

- ユーザーのシステムに対する信頼と、推薦を正しい視点で見る能力を向上させるのに役立つ
- さらに、システムのデバッグや予期せぬ動作の原因を突き止めるためにも有用

しかし、今回のような行列分解の手法(潜在因子モデル, Latest factor model)では、過去のユーザの行動はすべて**潜在変数を通して抽象化**されます。それにより**過去のユーザの行動と出力される推薦文の間に直接的な関係がなくなってしまう**ので、説明が困難になります。

ただ、このALSモデルではある程度説明可能性を得る事ができる、と筆者は述べています。以下、本手法の解釈性(説明性)についてまとめていきます。

その鍵は、式(4)$\mathbf{x}_u = (Y^T C^u Y + \lambda \mathbf{I})^{-1} Y^T C^u \cdot p(u)$を用いてユーザベクトル$\mathbf{x}_{u}$を置き換えることです。
$x_u$を上式で置き換えると、ユーザ u のアイテム i に対するPreferenceの推定値$\hat{p_{ui}}$は、

$$
\hat{p_{ui}} = \mathbf{x}_u^T \mathbf{y}_i \\
= \mathbf{y}_i^T \mathbf{x}_u
= \mathbf{y}_i^T (Y^T C^u Y + \lambda \mathbf{I})^{-1} Y^T C^u \cdot p(u)
$$

となります。

上記の表現は、以下の新しい表記法$W_u$と$s_{ij}^u$を導入する事で簡略化する事ができます。

上式における、f×f 行列$(Y^T C^u Y + \lambda \mathbf{I})^{-1}$(f は潜在変数の数)を$W_u$とすると、これは**ユーザ u に関連する重み付け行列**と見なす事ができます。

この場合、**ユーザ u の視点からの、アイテム i と j の重み付き類似度**$s_{ij}^u$は、

$$
s_{ij}^u = y_{i}^T W_u y_j
$$

と表されます。

**この新しい表記法を用いると、ユーザ u のアイテム i に対するPreferenceの推定値**は、

$$
\hat{p_{ui}} = \sum_{j:r_{u,j}>0}{s_{ij}^u c_{uj}}
$$

**で書き換える**ことができ、実はこれにより推定値pを解釈しやすくなるようです。

上式より、潜在因子モデル(行列分解によるレコメンド手法)は、

- **過去の行動($r_{uj}>0$)の線形関数として嗜好度合Preferenceを予測**し、**アイテム-アイテムの類似性によって重み付される線形モデル**

と解釈する事ができます。

それぞれの過去の行動($r_{uj}>0$、すなわち評価行列内のユーザ u の非ゼロ因子)は、予測される$p_{ui}$を形成する際にその類似度を足し合わせるので、**その固有の寄与度合いを分離する事ができます**。
すなわち、

- **どの過去の行動が予測値$p_{ui}$に最も寄与しているか解釈できる**！

そして、**最も寄与度の高い$r_{uj}>0$(=過去の行動)が、レコメンド結果の背景となる主要な説明**とする事ができます。

更に、個々の$r_{uj}>0$の寄与度$s_{ij}^u c_{uj}$を、ユーザuとアイテムjとの関係の重要度$c_{uj}$と、対象アイテムiとjの関係性$s_{ij}^u$の**2つの根拠に分離して考える事もできますね**

この解釈方法は、**アイテムベースの近傍モデルとよく似ており**、推定された予測値の説明性・解釈性を高め得る、とのことでした。
また、この手法におけるアイテム間の類似度$s_{ij}^u$は、**「２つのアイテムがどの程度似ているか」の感じ方は、各ユーザに応じて大なり小なり異なる**事を反映しており、通常のアイテムベースの近傍モデルよりもユーザの感性を適当に再現し得るモデル、といえそう...??

# スクラッチ実装

論文後半はまだ良く理解できていませんが、とりあえず理論は分かったのでアルゴリズムを実装していきます。

とりあえずサンプルの評価行列として、適当に以下のNumpy.Arrayを用意します。
ここで、サンプル評価行列の各行がユーザ、各列がアイテム、各要素はトランザクションの回数(例えば、アイテムの購入回数)を想定しています。

```python
# サンプルの評価行列を生成(行がユーザ、列がアイテム、各要素は購入回数を意味する)
R = np.array([
    [5, 3, 0, 1],
    [4, 0, 0, 1],
    [1, 1, 0, 5],
    [1, 0, 0, 4],
    [0, 1, 5, 4],
])
```

続いて、評価行列RからPreference値の行列Pを生成する関数を定義しておきます。

```python
def _get_preference_all_element(R: np.ndarray) -> np.ndarray:
    """評価行列Rを入力とし、Preferenceの実測値の行列Pを計算する関数。

    Parameters
    ----------
    R : np.ndarray
        評価行列R

    Returns
    -------
    np.ndarray
        Preferenceの実測値の行列P
    """
    # 各要素に対して条件分岐処理
    P = np.where(R > 0, 1, 0)
    return P

```

同様に、評価行列RからConfidence値の行列Cを生成する関数を定義します。

```python
def _get_confidence_all_element(R: np.ndarray, alpha:int) -> np.ndarray:
    """評価行列Rと\alphaを入力とし、Confidence値の行列Cを計算する関数

    Parameters
    ----------
    R : np.ndarray
        評価行列R
    alpha : int
        Confidence値の計算式におけるハイパーパラメータα。

    Returns
    -------
    np.ndarray
        Confidence値の行列C
    """
    # 各要素に対して四則演算
    C = R * alpha + 1
    return C
```

次に、ALSにおける目的関数(誤差関数)の値を計算する関数を定義していきます。

$$
\sum_{u, i} c_{ui}(p_{ui} - x_u^T y_i) + \lambda (\sum_u ||x_u||^2 + \sum_i ||y_i||^2) \tag{3}
$$

1つ目に、目的関数内のΣの中身1つ1つ、更に正確にはConfidence値による重み付けをする前の$(p_{ui} - x_u^T y_i)$を計算する関数を定義します。

```python
def _get_error_each_element(p_ui:float, x_u:np.ndarray, y_i: np.ndarray) -> float:
    """Preference行列内のある1要素p_uiの実測値と推定値の差を計算する関数

    Parameters
    ----------
    p_ui : float
        Preference行列内の各要素p_ui
    x_u : np.ndarray
        ユーザuのユーザベクトル。すなわちユーザ行列Xのu列目(列ベクトル)
    y_i : np.ndarray
        アイテムiのアイテムベクトル。すなわちアイテム行列yのi列目(列ベクトル)

    Returns
    -------
    float
        Preference行列内のある1要素p_uiの実測値と推定値の差。
    """
    return (p_ui - np.dot(x_u, y_i))
```

続いて2つ目に、目的関数におけるΣの外側、というか「各要素の誤差を計算してConfidenceで重みづけする処理」を行列内の全要素に適用する関数を定義します。

```python
def _get_error_all_element(P: np.ndarray, C: np.ndarray, X: np.ndarray, Y: np.ndarray, beta: float) -> float:
    """implicitデータに対するALSの行列分解における、目的関数の値を計算する関数。

    Parameters
    ----------
    P : np.ndarray
        評価行列の各要素r_uiから算出された嗜好度p_uiの行列(ユーザ×アイテム).
    C : np.ndarray
        評価行列の各要素r_uiから算出された信頼度c_uiの行列(ユーザ×アイテム).
    X : np.ndarray
        ユーザ行列(潜在変数×ユーザ)
    Y : np.ndarray
        アイテム行列(潜在変数×アイテム)
    beta : float
        L2正則化における罰則項のハイパーパラメータlambda

    Returns
    -------
    float
        implicitデータに対するALSの行列分解における、目的関数の値。
    """

    # 誤差関数の初期値
    error = 0.0
    # 2重(各アイテム、各ユーザ)のforループでXの各要素に対して処理を実行する.
    for u in range(len(P)):
        for i in range(len(P[u])):
            # 誤差をpow関数で2乗して、信頼度c_uiで重み付して、足し合わせ
            error += C[u][i] * \
                pow(_get_error_each_element(P[u][i], X[u, :], Y[i, :]), 2)

    # 全ての要素を足し終えたら、L2正則化項を追加
    error += beta/2.0 * (np.linalg.norm(x=X, ord=2) +
                         np.linalg.norm(x=Y, ord=2))

    return error
```

Preference値、Confidence値の計算、ALSにおける目的関数の計算の関数を定義できたので、最後にパラメータ更新の処理を定義します。
一応、以下の、計算量を減らしたVer.の更新式にしました笑

$$
x_u = (Y^T Y + Y^T(C_u - \mathbf{I})Y + \lambda \mathbf{I})^{-1} Y^T C^u \cdot p(u) \tag{6}
$$

$$
y_i = (X^T  X + X^T(C^i -\mathbf{I})X + \lambda \mathbf{I})^{-1} X^T C^i \cdot p(i) \tag{7}
$$

```python
def matrix_factorization_implicit(R: np.ndarray, len_of_latest_variable: int, steps: int = 5000, lr: float = 0.0002, alpha: int = 40, beta: float = 0.02, threshold: float = 0.001) -> Tuple[np.ndarray, np.ndarray]:
    """implicitデータに対する、ALSによる行列分解を実行する関数

    Parameters
    ----------
    R : np.ndarray
        実際に観測された評価行列.
    len_of_latest_variable : int
        Matrix Factorizationにおける潜在変数の数。
    steps : int, optional
        パラメータ更新を実行する回数, by default 5000:int
    lr : _type_, optional
        勾配降下法の学習率, by default 0.0002:float
    alpha : _type_, optional
        Confidence値の計算におけるハイパーパラメータ(元論文では40), by default 40:int
    beta : _type_, optional
        L2正則化における罰則項のハイパーパラメータlambda, by default 0.02:float
    threshold : _type_, optional
        学習を終了するかどうかを判定する、目的関数の値の閾, by default 0.001:float

    Returns
    -------
    Tuple[np.ndarray, np.ndarray]
        分解されたユーザ行列W(潜在変数×ユーザ)とアイテム行列H(潜在変数×アイテム)のタプル。
    """

    # ユニークなユーザ数mとアイテム数nを取得
    m = len(R)
    n = len(R[0])

    # WとHの初期値を設定
    X = np.random.rand(m, len_of_latest_variable)  # m * k行列
    Y = np.random.rand(n, len_of_latest_variable)  # n * k行列

    # 嗜好度行列Pを取得
    P = _get_preference_all_element(R=R)

    # 信頼度行列Cを取得
    C = _get_confidence_all_element(R=R, alpha=alpha)
    # 誤差関数の値の初期値
    error_value = 100

    # パラメータ更新
    for step in tqdm(range(steps)):
        # 更新式において、事前に計算できる値を計算
        Y_T_Y = np.dot(Y.T, Y)  # -.(k×k行列)
        X_T_X = np.dot(X.T, X)  # -.(k×k行列)

        # まずはユーザ行列を更新
        for u in range(m):
            # C_uを作成(n×nの対角行列)
            C_u = np.diag(C[u])
            # p_uを作成(一次元配列=>列ベクトル)
            p_u = P[u].reshape(-1, 1)
            # 更新式
            # x_u = np.linalg.inv(Y.T @ C_u @ Y + np.eye(len_of_latest_variable)
            #                     * beta) @ Y.T @ C_u @ p_u
            # 更新式(計算量節約ver.)
            x_u = np.linalg.inv(Y_T_Y + Y.T @ (C_u - np.eye(n)) @ Y +
                                np.eye(len_of_latest_variable) * beta) @ Y.T @ C_u @ p_u
            X[u:u+1, :] = x_u.T

            del C_u, p_u, x_u

        # 続いてアイテム行列を更新
        for i in range(n):
            # C_i を作成
            C_i = np.diag(C[:, i])
            # p_iを作成(一次元配列=>列ベクトル)
            p_i = P[:, i].reshape(-1, 1)

            # 更新式
            # y_i = np.linalg.inv(
            #     X.T @ C_i @ X + np.eye(len_of_latest_variable)*beta) @ X.T @ C_i @ p_i

            # 更新式(計算量節約ver.)
            y_i = np.linalg.inv(X_T_X + X.T @ (C_i - np.eye(m)) @ X +
                                np.eye(len_of_latest_variable) * beta) @ X.T @ C_i @ p_i
            Y[i:i+1, :] = y_i.T

            del C_i, p_i, y_i

        # 更新後の目的関数(誤差関数)の値を確認
        error_value = _get_error_all_element(P, C, X, Y, beta)
        # 十分に評価行列Xを近似できていれば、パラメータ更新終了！
        if error_value < threshold:
            break

    print(error_value)
    return X, Y
```

最後に、長いですがコードの全体像を記述しておきます。

```python
from typing import Tuple
import numpy as np
from numpy import ndarray
from tqdm import tqdm


def _get_preference_all_element(R: np.ndarray) -> np.ndarray:
    """評価行列Rを入力とし、Preferenceの実測値の行列Pを計算する関数。

    Parameters
    ----------
    R : np.ndarray
        評価行列R

    Returns
    -------
    np.ndarray
        Preferenceの実測値の行列P
    """
    # 各要素に対して条件分岐処理
    P = np.where(R > 0, 1, 0)
    return P


def _get_confidence_all_element(R: np.ndarray, alpha: int) -> np.ndarray:
    """評価行列Rと\alphaを入力とし、Confidence値の行列Cを計算する関数

    Parameters
    ----------
    R : np.ndarray
        評価行列R
    alpha : int
        Confidence値の計算式におけるハイパーパラメータα。

    Returns
    -------
    np.ndarray
        Confidence値の行列C
    """
    # 各要素に対して四則演算
    C = R * alpha + 1
    return C


def _get_error_each_element(p_ui: float, x_u: np.ndarray, y_i: np.ndarray) -> float:
    """Preference行列内のある1要素p_uiの実測値と推定値の差を計算する関数

    Parameters
    ----------
    p_ui : float
        Preference行列内の各要素p_ui
    x_u : np.ndarray
        ユーザuのユーザベクトル。すなわちユーザ行列Xのu列目(列ベクトル)
    y_i : np.ndarray
        アイテムiのアイテムベクトル。すなわちアイテム行列yのi列目(列ベクトル)

    Returns
    -------
    float
        Preference行列内のある1要素p_uiの実測値と推定値の差。
    """
    return (p_ui - np.dot(x_u, y_i))


def _get_error_all_element(P: np.ndarray, C: np.ndarray, X: np.ndarray, Y: np.ndarray, beta: float) -> float:
    """implicitデータに対するALSの行列分解における、目的関数の値を計算する関数。

    Parameters
    ----------
    P : np.ndarray
        評価行列の各要素r_uiから算出された嗜好度p_uiの行列(ユーザ×アイテム).
    C : np.ndarray
        評価行列の各要素r_uiから算出された信頼度c_uiの行列(ユーザ×アイテム).
    X : np.ndarray
        ユーザ行列(潜在変数×ユーザ)
    Y : np.ndarray
        アイテム行列(潜在変数×アイテム)
    beta : float
        L2正則化における罰則項のハイパーパラメータlambda

    Returns
    -------
    float
        implicitデータに対するALSの行列分解における、目的関数の値。
    """

    # 誤差関数の初期値
    error = 0.0
    # 2重(各アイテム、各ユーザ)のforループでXの各要素に対して処理を実行する.
    for u in range(len(P)):
        for i in range(len(P[u])):
            # 誤差をpow関数で2乗して、信頼度c_uiで重み付して、足し合わせ
            error += C[u][i] * \
                pow(_get_error_each_element(P[u][i], X[u, :], Y[i, :]), 2)

    # 全ての要素を足し終えたら、L2正則化項を追加
    error += beta/2.0 * (np.linalg.norm(x=X, ord=2) +
                         np.linalg.norm(x=Y, ord=2))

    return error


def matrix_factorization_implicit(R: np.ndarray, len_of_latest_variable: int, steps: int = 5000, lr: float = 0.0002, alpha: int = 40, beta: float = 0.02, threshold: float = 0.001) -> Tuple[np.ndarray, np.ndarray]:
    """implicitデータに対する、ALSによる行列分解を実行する関数

    Parameters
    ----------
    R : np.ndarray
        実際に観測された評価行列.
    len_of_latest_variable : int
        Matrix Factorizationにおける潜在変数の数。
    steps : int, optional
        パラメータ更新を実行する回数, by default 5000:int
    lr : _type_, optional
        勾配降下法の学習率, by default 0.0002:float
    alpha : _type_, optional
        Confidence値の計算におけるハイパーパラメータ(元論文では40), by default 40:int
    beta : _type_, optional
        L2正則化における罰則項のハイパーパラメータlambda, by default 0.02:float
    threshold : _type_, optional
        学習を終了するかどうかを判定する、目的関数の値の閾, by default 0.001:float

    Returns
    -------
    Tuple[np.ndarray, np.ndarray]
        分解されたユーザ行列W(潜在変数×ユーザ)とアイテム行列H(潜在変数×アイテム)のタプル。
    """

    # ユニークなユーザ数mとアイテム数nを取得
    m = len(R)
    n = len(R[0])

    # WとHの初期値を設定
    X = np.random.rand(m, len_of_latest_variable)  # m * k行列
    Y = np.random.rand(n, len_of_latest_variable)  # n * k行列

    # 嗜好度行列Pを取得
    P = _get_preference_all_element(R=R)

    # 信頼度行列Cを取得
    C = _get_confidence_all_element(R=R, alpha=alpha)
    # 誤差関数の値の初期値
    error_value = 100

    # パラメータ更新
    for step in tqdm(range(steps)):
        # 更新式において、事前に計算できる値を計算
        Y_T_Y = np.dot(Y.T, Y)  # -.(k×k行列)
        X_T_X = np.dot(X.T, X)  # -.(k×k行列)

        # まずはユーザ行列を更新
        for u in range(m):
            # C_uを作成(n×nの対角行列)
            C_u = np.diag(C[u])
            # p_uを作成(一次元配列=>列ベクトル)
            p_u = P[u].reshape(-1, 1)
            # 更新式
            # x_u = np.linalg.inv(Y.T @ C_u @ Y + np.eye(len_of_latest_variable)
            #                     * beta) @ Y.T @ C_u @ p_u
            # 更新式(計算量節約ver.)
            x_u = np.linalg.inv(Y_T_Y + Y.T @ (C_u - np.eye(n)) @ Y +
                                np.eye(len_of_latest_variable) * beta) @ Y.T @ C_u @ p_u
            X[u:u+1, :] = x_u.T

            del C_u, p_u, x_u

        # 続いてアイテム行列を更新
        for i in range(n):
            # C_i を作成
            C_i = np.diag(C[:, i])
            # p_iを作成(一次元配列=>列ベクトル)
            p_i = P[:, i].reshape(-1, 1)

            # 更新式
            # y_i = np.linalg.inv(
            #     X.T @ C_i @ X + np.eye(len_of_latest_variable)*beta) @ X.T @ C_i @ p_i

            # 更新式(計算量節約ver.)
            y_i = np.linalg.inv(X_T_X + X.T @ (C_i - np.eye(m)) @ X +
                                np.eye(len_of_latest_variable) * beta) @ X.T @ C_i @ p_i
            Y[i:i+1, :] = y_i.T

            del C_i, p_i, y_i

        # 更新後の目的関数(誤差関数)の値を確認
        error_value = _get_error_all_element(P, C, X, Y, beta)
        # 十分に評価行列Xを近似できていれば、パラメータ更新終了！
        if error_value < threshold:
            break

    print(error_value)
    return X, Y


def main():
    # サンプルの評価行列を生成(行がユーザ、列がアイテム、各要素は購入回数を意味する)
    R = np.array([
        [5, 3, 0, 1],
        [4, 0, 0, 1],
        [1, 1, 0, 5],
        [1, 0, 0, 4],
        [0, 1, 5, 4],
    ]
    )

    # 行列分解
    X_hat, Y_hat = matrix_factorization_implicit(
        R, len_of_latest_variable=3, steps=10)
    # P\hatを生成.
    P_hat = np.dot(a=X_hat, b=Y_hat.T)
    print(P_hat)


if __name__ == '__main__':
    main()

```

# おわりに

途中、更新式中の逆行列にする箇所を見逃して実装したり、手間取りましたが無事にスクラッチ実装する事ができました。
恐らくですが実際の現場における評価行列は、100万行×10万列のような巨大な疎行列なので、numpy.Arrayのような通常の行列格納方式ではなく、COO形式やCSR形式等の疎行列専用の格納方式を用いないと計算量が膨大になってしまうと思います。
ですがやはり、数式を追いながらスクラッチで実装すると、手法に対する理解が深まる気がしますね。

最後までお読みいただき、本当にありがとうございました：）

最後になりますが、僕と同じように、「ALSによるMatrix Factorizationってこういう事なのね！あれ、でもこれはExplicitデータに対する数式だよね？じゃあImplicitデータに対してはどうやるんだ－？あれ？パッと調べても出てこない...」という状況に落ちいった方の助けになれば嬉しいです。

シューマイ！！

# 参考
- 「Matrix Factorizationとは」(Explicitデータに対するALSのスクラッチ実装をされていて、関数の定義方法等を参考にさせていただきました！)
    - https://qiita.com/ysekky/items/c81ff24da0390a74fc6c

- Collaborative Filtering for Implicit Feedback Datasets(上述した手法の論文です！)
  - http://yifanhu.net/PUB/cf.pdf