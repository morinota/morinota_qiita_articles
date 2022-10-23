<!--
title: [空間統計学] MoranのI統計量ををPythonのpysalで算出してみる
tags: Python
-->

# はじめに

私は普段、空間データの一種である地理情報データを扱った分析をしています。しかしこれまで、空間的自己相関を考慮した分析を行っておりませんでした。空間に基づくデータを扱う上で、こういった空間データ独自の性質を考慮して分析する事は必須である、もしくは、必須でなくとも考慮する事で更に有用な示唆が得られるかもしれない、と考え、空間統計学(空間計量経済学)を勉強し始めました。そのOutputの場の一環として、Pythonによる空間統計解析のTips共有も兼ねて本記事を投稿させていただきました...!
手法の理解や統計的な解釈が誤っている点がある気がするので、その際はぜひお気軽に優しくコメントいただけますと喜びます:)

なお、本記事で使用しているサンプルデータや処理の流れは[Geographic Data Science with Python](https://geographicdata.science/book/intro.html)を参考にしております:)
英語だけどPythonで空間統計的な分析をやってる資料は少ないのでめちゃめちゃ良さそうです...!

# 空間的自己相関とは

"空間的自己相関(Spatial Autocorrelation)"とは、「距離が近いほど事物の性質が似る(あるいは異なる)」という空間データの性質(特徴?)の一つです。("空間的依存性(Spatial Dependence)"という用語も用いられるようですが、実際には両者とも自己相関の意味で用いられる事が多いようです...!)
ちなみに他にも空間データの性質として、"空間的異質性(Spatial Heterogeneity)"という性質もあります。

空間的自己相関は以下の２つに大別されます。

- 距離の近いデータが似たような傾向を示す「正の空間的自己相関」
- 距離の近いデータが非常に異なった値を示す「負の空間的自己相関」

これらは「距離が近い事物はより強く影響しあう」というToblerの地理学の第一法則(First law of geopgraphy)として知られるものです。

自己相関は、数学的には次のような積率条件で表されます。

$$
Cov(y_i, y_j) = E(y_i y_j) - E(y_i)E(y_j) \neq 0, \forall i \neq j
$$

ここで、各記号の意味は以下です。

- $y_i$と$y_j$は観測地点i及びjにおける分析対象のデータ
- $Cov(a,b)$はaとbの共分散、$E(a)$はaの期待値
- $\forall$は"任意の、全ての"

そして上述したデータ間の相関が0でないという事に関して、空間構造、空間的な相互作用、空間的な位置関係という観点から解釈できる場合に、「空間的」自己相関であると表現されます。

# スケールに応じた空間的自己相関の評価・検定

一口に空間的自己相関といっても、実際には様々な空間スケールが存在します。

- 例)不動産価格
  - 関東地方で大域的に見ると...東京都心部では不動産価格が高い。地方部では低い。
  - 東京都心部の中で局所的に見ると...高級住宅地の存在

前者に対応する、データの全体的な空間的自己相関の有無に対応する評価指標(検定統計量)は、Global Indicators of Spatial Association(GISA)と呼ばれます。
一方で、後者に対応するホットスポット(平均以上の値の集積)やクールスポット(平均以下の値の集積)などの局所的な空間的自己相関の有無に関する評価指標(検定統計量)はLocal Indicator of Spatial Association(LISA)と呼ばれます。
GISAでは「データに空間的な自己相関が存在するか」という点に着目する一方で、LISAでは「空間的自己相関がどこに存在するか」という点に着目しているのが違いのようですね...!

## GISAの一つ：Global Moran's I

GISAで有名な指標の一つが、Global Moran's I 統計量。これは相関係数のアナロジー(応用)らしい...。
Global Morans'Iは次式で定義される。

$$
I = \frac{n}{S_0} \cdot
\frac{\sum_i \sum_j w_{ij}(y_i-\bar{y})(y_j - \bar{y})}{\sum_i (y_i - \bar{y})^2}

= \frac{n}{\sum_{i}\sum_{j} w_{ij}}
\frac{\sum_{i}\sum_{j} w_{ij} z_i z_j}{\sum_{i} z_i^2}
$$

ここで、nはサンプル数、$\sum_{i}\sum_{j} w_{ij} = S_0$は基準化定数(重み行列の全要素の和)であり、重み行列が行標準化(各行の和が1に基準化)されている場合、$n = S_0$となり第一項が消える為、グローバルモランはシンプルな形になる。

相関係数と同様に-1～1までの値を取り、1に近い事は正の空間的自己相関の存在を示唆し、逆に-1に近い事は負の空間的自己相関の存在を示唆します。

## LISAの一つ：Local Moran's I

LISAの有名な指標の一つがローカルモラン(Local Moran's I)統計量。次式のように定義される。

$$
I_i = \frac{(y_i -\bar{y})}{m_2}
\sum_{j} w_{ij} (y_j -\bar{y})
= \frac{z_i}{m_2}
\sum_{j} w_{ij} z_{j} \\
m_2 = \frac{\sum_{i} (y_i - \bar{y})^2}{n}
= \frac{\sum_{i} z_i^2}{n}
$$

ここで、$m_2 = \frac{\sum_{i} (y_i - \bar{y})^2}{n}$は比例定数。このように$I_i$は、自身の値の平均値からの偏差と、近傍集合における観測値の平均からの偏差との**類似度**として定義される。
すなわち、自身の値が、近傍の値と似通った値を取れば$I_i$は正の大きな値を取り、非常に異なった値を取れば$I_i$は負の値を取る。一方、周囲の値との間に関連性がなければ、$I_i$は0に近い値をとる。

ローカルモランの和を取ると、

$$
\sum_i I_i = \sum_i (\frac{y_i - \bar{y}}{m_2} \sum_j w_{ij}(y_j -\bar{y}))
$$

となり、これをグローバルモランと比較すると、

$$
I = \frac{\sum_i I_i}{S}
$$

という関係が得られ、ローカルモランの和とグローバルモランは比例関係にある事がわかるようです。

# Pythonでグローバルモランとローカルモランの計算を試してみる

では実際に計算してみます。[Pysal](https://pysal.org/)パッケージを使用します。(PythonのSpatial Statisticsのライブラリって他に良いのあるのかな...?)

今回のグローバルモランとローカルモランの計算には、`pysal.explore.esda`モジュールを適用します。(esdaはExploratory Spatial Data Anaysisの略ですね)
また、上述した2つの指標の計算において、各Observationの位置関係(近隣のObservationか否かの関係)を数式に埋め込む為に空間重み行列(Spatial Weight Matrix)を作成する必要があるのですが、`pysal.lib.weights`モジュールを用いて作成します。

```python
import os

# Graphics
import matplotlib.pyplot as plt
import seaborn
from pysal.viz import splot
from splot.esda import plot_moran
import contextily

# Analysis
import geopandas as gpd
import pandas as pd
from pysal.explore import esda
from pysal.lib import weights
from numpy.random import seed
```

## サンプルデータを用意

まずはサンプルデータとなる空間データを用意します。
今回はBrexit投票(イギリスがEU残留か離脱かを決める国民投票)のデータを使用させていただきます。
[Geographic Data Science with Python のDataset](https://geographicdata.science/book/data/brexit/brexit_cleaning.html)からダウンロードし、ローカルに保存します。

以下の2種類のデータを組み合わせて空間データを作成します。

- `brexit_vote.csv` (イギリス内の各districtのBrexit投票結果に関するデータ)
- `local_authority_districts.geojson` (イギリス内のdistrictsの境界の地理情報データ)

以下のような感じで`load_brefix_data`関数を用意し、main()内で呼び出してみます。

```python
def load_brefix_data(brexit_data_dir: str) -> gpd.GeoDataFrame:
    os.path.join(brexit_data_dir, r"brexit_vote.csv")
    df_brexit_vote = pd.read_csv(
        os.path.join(brexit_data_dir, r"brexit_vote.csv"),
    )
    gdf_eu_districts = gpd.read_file(os.path.join(brexit_data_dir, r"local_authority_districts.geojson"))

    gdf_brexit = pd.merge(
        left=gdf_eu_districts, right=df_brexit_vote, how="inner", left_on="lad16cd", right_on="Area_Code"
    )
    return gdf_brexit

def main():
    gdf_brexit = load_brefix_data(r"hogehoge\data\brexit")
    print(gdf_brexit.head())
```

きちんと各districtのBrexit投票の結果が格納された、空間データ(`geopandas.GeoDataFrame`)が作成されていますね。

```
 objectid    lad16cd                      lad16nm  ... Pct_Remain  Pct_Leave  Pct_Rejected
0         1  E06000001                   Hartlepool  ...      30.43      69.57          0.07
1         2  E06000002                Middlesbrough  ...      34.52      65.48          0.06
2         3  E06000003         Redcar and Cleveland  ...      33.81      66.19          0.04
3         4  E06000004             Stockton-on-Tees  ...      38.27      61.73          0.04
4        10  E06000010  Kingston upon Hull, City of  ...      32.38      67.62          0.07
```

せっかく空間データを作成したので、まずはBrexit投票で得られた「各districtのEU離脱派の割合」のChoroplethマップを描画してみます。
「各districtのEU離脱派の割合」は`Pct_Leave`カラムに格納されています。
今回は以下の`create_choropleth_map`関数を定義し、main()内でChoroplethマップを描画してみます。

```python
def create_choropleth_map(gdf: gpd.GeoDataFrame, target_col: str, draw_axes: Axes) -> Axes:
    gdf = gdf.to_crs(crs=DEFAULT_CRS)
    gdf.plot(
        column=target_col,
        cmap="viridis",
        scheme="quantiles",
        k=5,
        edgecolor="white",
        linewidth=0.0,
        alpha=0.75,
        legend=True,
        legend_kwds={"loc": 2},
        ax=draw_axes,
    )
    contextily.add_basemap(
        ax=draw_axes,
        crs=DEFAULT_CRS,
        source=contextily.providers.Stamen.TerrainBackground,
    )
    draw_axes.set_axis_off()

def main():
    gdf_brexit = load_brefix_data(r"Statistics\Spatial Statistics\notebooks\data\brexit")

    fig_obj, axes_obj = plt.subplots(1, figsize=(9, 9))
    axes_obj = create_choropleth_map(gdf=gdf_brexit, target_col="Pct_Leave", draw_axes=axes_obj)
    fig_obj.savefig("Pct_Leave_choropleth_map.jpg")
```

保存された画像が以下になります。

![](https://i.imgur.com/rszpPAW.jpg)

北側では紫色の(=離脱派の割合が低い)districtが多い、南東側に黄色の(=離脱派の割合が高い)districtが集まっている事が何となくわかりますね。

さてここから、この観測値(「各districtのEU離脱派の割合」)に対して空間的自己相関の評価を行ってみます...!

## 空間重み行列を作成

モランi統計量を算出する準備として、観測地(位置情報付きの観測値)間の位置関係(近隣の観測地か否かの関係)を数式に埋め込む為に空間重み行列(Spatial Weight Matrix)を作成する必要がありましたね。`pysal.lib.weights`モジュールを用いて作成します。

今回は、n=8のn nearest neigbhorに基づく空間重み行列を適用しました。「観測地iから最も近いn=8個の観測地を、近隣の観測地jとみなす」という事ですね。
できあがる空間重み行列は、$len(gdf_brexit) \times len(gdf_brexit)$の形で、各行の8個の要素に1が入り、残りの要素は0で埋められています。

作成した空間重み行列は、main()内で`spatial_weight`に格納しています。
また各行の要素の総和が1になるように行標準化を行っています。

```python
def main():
    # 略

    spatial_weight_matrix = weights.KNN.from_dataframe(df=gdf_brexit, k=8)
    spatial_weight_matrix.transform = "R"  # Row-standardization
```

## グローバルモランを計算

さて「観測されたデータ全体として空間的自己相関を持っているか否か」を評価する為に、グローバルモラン統計量を算出してみます。

以下の`compute_global_moran_i_statistic`関数を用意し、main()内で呼び出します。

```python
def compute_global_moran_i_statistic(
    gdf: gpd.GeoDataFrame, spatial_weight_matrix: weights.KNN, target_col: str
) -> esda.moran.Moran:
    moran_i_obj = esda.moran.Moran(y=gdf[target_col], w=spatial_weight_matrix)

    fig_obj, axes_obj = plot_moran(moran=moran_i_obj)
    fig_obj.savefig("Pct_Leave_moran_plot.jpg")

    return moran_i_obj


def main():
    # 略

    global_moran_obj = compute_global_moran_i_statistic(gdf_brexit, spatial_weight_matrix, "Pct_Leave")
    print(f"[LOG] moran i statistic: {global_moran_obj.I}")
    print(f"[LOG] empirical p-value: {global_moran_obj.p_sim}")
```

`esda.moran.Moran`クラスに観測データと空間重み行列を渡してInitializeする事で、グローバルモラン統計量の計算及び検定が実施されます。グローバルモラン統計量の値は`.I`属性、経験的p値(?)の値は`.p_sim`属性にアクセスする事で取得できます。

また`compute_global_moran_i_statistic`関数内では、グローバルモランの計算後、モランプロットと呼ばれる図を描画しています。描画には、Pysalと連携した描画パッケージ[splot](https://splot.readthedocs.io/en/latest/index.html)の`esda.plot_moran()`関数を用いています。

結果として以下が出力されました。

```
[LOG] moran i statistic: 0.6487282576278296
[LOG] empirical p-value: 0.001
```

![](https://i.imgur.com/GOEaWSj.jpg)

グローバルモラン統計量の値、及びモランプロットの結果から、「観測データ(「各districtのEU離脱派の割合」)はデータ全体として正の空間的自己相関がありそうだ...!」という事が評価されました。
つまりデータ全体として、「"離脱派の割合の高い地域"の近隣の地域は"離脱派の割合の高い地域"である傾向が強い(その逆もしかり)」事がわかりました。

## ローカルモランを計算

続いて、「観測データのどこに空間的自己相関が発生していそうか」を評価する為にローカルモラン統計量を算出してみます。
`compute_local_moran_i`関数を定義し、グローバルモラン同様に観測データと空間重み行列を渡して`esda.moran.Moran_Local`クラスをInitializeする事で、ローカルモランの計算・検定が実行されます。

生成された`esda.moran.Moran_Local`インスタンスの`.Is`属性に、各観測地のローカルモラン統計量が`numpy.ndarray`型で格納されています。各観測地のローカルモラン統計量の検定結果としての経験的p値も`.p_sim`属性に`numpy.ndarray`型で格納されています。

```python
def compute_local_moran_i(gdf: gpd.GeoDataFrame, spatial_weight_matrix: weights.KNN, target_col: str):

    local_moran_i_obj = esda.moran.Moran_Local(y=gdf[target_col], w=spatial_weight_matrix)

    return local_moran_i_obj


def main():
    # 略
    local_moran_obj = compute_local_moran_i(gdf=gdf_brexit, spatial_weight_matrix=spatial_weight, target_col="obs")
    print(f"[LOG] moran i statistic: {local_moran_obj.Is}")
    print(f"[LOG] empirical p-value: {local_moran_obj.p_sim}")
```

出力されるローカルモランの結果は以下のようになります。想定通り、各観測地に対して統計量とp値が算出されていますね。

```
[LOG] moran i statistic: [ 1.03935390e+00  8.65206126e-01  9.03412503e-01  6.55111797e-01
1.30305057e+00  4.92360117e-01 -9.33986921e-02  2.02530247e-01
3.57668324e-02  5.52237916e-01  1.80390307e-02  1.77251099e-01
4.67981456e-01  1.85867606e+00 -4.81871433e-01  2.49555079e-01
1.37333552e+00 -7.33093499e-02 -1.50991602e-01 -1.43404597e-01
...
[LOG] empirical p-value: [0.016 0.007 0.007 0.006 0.001 0.013 0.195 0.016 0.424 0.001 0.333 0.045
0.151 0.001 0.096 0.025 0.001 0.304 0.034 0.023 0.132 0.001 0.019 0.266
0.367 0.232 0.384 0.258 0.001 0.119 0.157 0.357 0.124 0.088 0.001 0.001
...
```

## ローカルモラン統計量の算出＆検定結果を用いてChoroplethマップを作成

ローカルモラン統計量に関しては、数値だけ見ても「どの地域に空間的自己相関が発生しているか」把握しづらいので、得られた統計量とp値を用いてChoroplethマップを作成してみます。
今回はローカルモラン統計量をもとに、以下の4つのChoroplethマップを作成しました。

- 1⃣ローカルモラン統計量の値
- 2⃣モランプロット散布図(観測値$y_i$と近隣観測値$y_j$の平均値の関係の散布図)において象限毎に分けたもの
- 3⃣ローカルモラン統計量の検定における経験的p値
- 4⃣モランプロット散布図において象限毎に分けたもの

`ChoroplethMapLISA`クラスを定義して、4つのChoroplethマップを作成する処理を実装しています。実装した`create_local_moran_i_choropleth_maps`メソッドをmain()で呼び出し、4種のChoroplethマップが描画された`Figure`オブジェクトを取得します。

```python
class ChoroplethMapLISA:
    def create_local_moran_i_choropleth_maps(self, gdf: gpd.GeoDataFrame, local_moran_obj: esda.moran.Moran_Local):
        gdf = gdf.to_crs(crs=DEFAULT_CRS)

        fig_obj, axes_obj = plt.subplots(nrows=2, ncols=2, figsize=(12, 12))

        axes_obj = axes_obj.flatten()  # Make the axes accessible with single indexing

        axes_obj[0] = self._choropleth_local_statistic(axes_obj[0], gdf, local_moran_obj)
        axes_obj[1] = self._choropleth_quadrant_categories(axes_obj[1], gdf, local_moran_obj)
        axes_obj[2] = self._choropleth_significance_map(axes_obj[2], gdf, local_moran_obj)
        axes_obj[3] = self._choropleth_cluster_map(axes_obj[3], gdf, local_moran_obj)

        fig_obj.tight_layout()
        fig_obj.savefig("local_moran_i_choropleth_maps.jpg")

    def _choropleth_local_statistic(
        self, axes_obj: Axes, gdf: gpd.GeoDataFrame, local_moran_obj: esda.moran.Moran_Local
    ) -> Axes:
        df_temp = gdf.assign(Is=local_moran_obj.Is)
        axes_obj = df_temp.plot(
            column="Is",
            cmap="plasma",
            scheme="quantiles",
            k=5,
            edgecolor="white",
            linewidth=0.1,
            alpha=0.75,
            legend=True,
            ax=axes_obj,
        )
        axes_obj = self._arrange_axis_obj(axes_obj, "local_moran_i_statistic")
        return axes_obj

    def _choropleth_quadrant_categories(
        self, axes_obj: Axes, gdf: gpd.GeoDataFrame, local_moran_obj: esda.moran.Moran_Local
    ) -> Axes:
        lisa_cluster(moran_loc=local_moran_obj, gdf=gdf, p=1, ax=axes_obj)  # 有意水準の設定(1に設定すると全てのObservationが有意として扱われる)
        axes_obj = self._arrange_axis_obj(axes_obj, "scatterplot_quadrant")
        return axes_obj

    def _choropleth_significance_map(
        self, axes_obj: Axes, gdf: gpd.GeoDataFrame, local_moran_obj: esda.moran.Moran_Local
    ) -> Axes:
        # Recode 1 to "Significant and 0 to "Non-significant"
        significance_dammy_val_series = pd.Series(
            data=1 * (local_moran_obj.p_sim < 0.05),  # Assign 1 if significant, 0 otherwise
            index=gdf.index,  # Use the index in the original data
        )
        df_temp = gdf.assign(significance_dammy=significance_dammy_val_series)
        axes_obj = df_temp.plot(
            column="significance_dammy",
            categorical=True,
            k=2,
            cmap="Paired",
            linewidth=0.1,
            edgecolor="white",
            legend=True,
            ax=axes_obj,
        )

        axes_obj = self._arrange_axis_obj(axes_obj, "statistical_significance")

        return axes_obj

    def _choropleth_cluster_map(
        self, axes_obj: Axes, gdf: gpd.GeoDataFrame, local_moran_obj: esda.moran.Moran_Local
    ) -> Axes:
        lisa_cluster(moran_loc=local_moran_obj, gdf=gdf, p=0.05, ax=axes_obj)

        axes_obj = self._arrange_axis_obj(axes_obj, "moran_cluster_map")

        return axes_obj

    def _arrange_axis_obj(self, axes_obj: Axes, axes_title: str) -> Axes:
        axes_obj.set_axis_off()
        axes_obj.set_title(label=axes_title)

        contextily.add_basemap(
            ax=axes_obj,
            crs=DEFAULT_CRS,
            source=contextily.providers.Stamen.TerrainBackground,
        )
        return axes_obj

def main():
    # 略

    choropleth_obj = ChoroplethMapLISA()
    fig_obj = choropleth_obj.create_local_moran_i_choropleth_map(gdf_brexit, local_moran_obj)
    fig_obj.savefig("Pct_Leave_local_moran_i_choropleth_maps.jpg")
```

描画された4つのChoroprethマップが以下になります。

- 1⃣ローカルモラン統計量の値 => 左上
- 2⃣モランプロット散布図(観測値$y_i$と近隣観測値$y_j$の平均値の関係の散布図)において象限毎に分けたもの => 右上
- 3⃣ローカルモラン統計量の検定における経験的p値 => 左下
- 4⃣モランプロット散布図において象限毎に分けたもの => 右下

![](https://i.imgur.com/4vtJXhz.jpg)

### 左上の図

左上の図において、黄色(正の空間的自己相関の傾向が強い)と紫色(負の空間的自己相関の傾向が強い)の地域は、ローカルモラン統計量の絶対値(正と負)が最大である事を示しています。しかし注意が必要なのは、正の空間的自己相関の傾向が強い黄色の地域に関して、**「EU離脱派の割合」が高い地域か低い地域かは区別できない**事です！

### 右上の図

続いて右上の図を見てみます。この図は、各観測地iの"モランプロット散布図における象限内の位置"で色分けされています。
モランプロットは、観測値$y_i$と近隣観測値$y_j$の平均値(Spatial Lagと言ったりします)の関係の散布図です。なので、各観測地i各象限に存在する事の意味合いは以下のようになります。

- 第一象限：$y_i$は正の値であり、近隣の観測値(Spatial Lag)も正の値を取る傾向にある。(ホットスポット, HH)
- 第二象限：$y_i$は負の値であり、近隣の観測値(Spatial Lag)は正の値を取る傾向にある。(ドーナツと呼ばれるらしい..., LH)
- 第三象限：$y_i$は負の値であり、近隣の観測値(Spatial Lag)も負の値を取る傾向にある。(コールドスポット, LL)
- 第四象限：$y_i$は正の値であり、近隣の観測値(Spatial Lag)は負の値を取る傾向にある。(ダイヤモンド・イン・ザ・ラフと呼ばれるらしい..., HL)

右上の図を見ると、北部はEU離脱派の割合が低く(コールドスポットが多い)、南東部ではEU離脱派の割合が高い地域が集まっている(ホットスポットが多い)ように見えますね。

ちなみにこの「モランプロット散布図における象限内の位置」の情報は`esda.moran.Moran_Local`オブジェクトの`.q`属性に記録されています。

```python
local_moran_obj.q[:10]
=> array([1, 1, 1, 1, 1, 1, 4, 1, 4, 1])
```

### 左下 & 右下の図

しかし、**最初の2つのマップの解釈には注意が必要**です。なぜなら、Localな値の**基本的な統計的有意性は考慮されていないから**です。

左下の図は、各地域のローカルモランの統計的有意性の有無(p <= 0.05なら有意とみなす)で色分けしています。このマップを見ると、かなりの数の地域が、純粋な偶然と見なせるほど小さな統計値を有している(=すなわち、ローカルな空間的自己相関が存在するとはいえない＝Spatial Raodomnessの仮定を棄却できない)ことがわかります。

そして右下の図では、統計的有意性を考慮して"モランプロット散布図における象限内の位置"で色分けしています。

これらの図から以下の示唆が得られます。

- 純粋な偶然という考えを否定できるほど強いローカルな空間的自己相関を持つ地域が半分程度であること
- EU 離脱支持率が低い地域が明確に3つある事
- 支持率の高い地域が集中しているように見えるが、北部と南東部の地域だけが、空間的自己相関が強く、且つそれが偶然で発生した誤差である可能性が低い

# おわりに

本記事では、Pythonによる空間的自己相関の評価指標(グローバルモランとローカルモラン)の計算について投稿してみました。

自身の研究では、シミュレーションした結果をChoroprethマップとして描画する事はあっても、空間的な分布傾向や凝集度合いを定量的な数値で評価する事はなかったので、本稿のような評価指標を適用する事で新たな示唆が得られるのでは...と感じました！

今後は空間的自己相関や空間的異質性を考慮した統計モデリングを触ってみようと思っています：）
読んでいただきありがとうございました！

お互い難しくも素敵な研究 or 分析ライフを～：）

# 参考

- [Geographic Data Science with Python, Global Spatial Autocorrelation](https://geographicdata.science/book/notebooks/06_spatial_autocorrelation.html)
- [Geographic Data Science with Python, Local Spatial Autocorrelation](https://geographicdata.science/book/notebooks/07_local_autocorrelation.html)
- [空間統計学: 自然科学から人文・社会科学まで (統計ライブラリー)](https://www.amazon.co.jp/%E7%A9%BA%E9%96%93%E7%B5%B1%E8%A8%88%E5%AD%A6-%E8%87%AA%E7%84%B6%E7%A7%91%E5%AD%A6%E3%81%8B%E3%82%89%E4%BA%BA%E6%96%87%E3%83%BB%E7%A4%BE%E4%BC%9A%E7%A7%91%E5%AD%A6%E3%81%BE%E3%81%A7-%E7%B5%B1%E8%A8%88%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%83%BC-%E7%80%AC%E8%B0%B7-%E5%89%B5/dp/4254128312/ref=asc_df_4254128312/?tag=jpgo-22&linkCode=df0&hvadid=295700463796&hvpos=&hvnetw=g&hvrand=4198896870091686871&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1009461&hvtargid=pla-525444269971&psc=1&th=1&psc=1)
