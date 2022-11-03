<!--
title:   [Python × GIS]Polygonデータに含まれるPointデータの数をカウントするPointInPolygonクラスをgeopandasパッケージで実装してみた
tags:    Python
id:      4f5d0ec64c8a83cb482b
private: false
-->


# はじめに

今回はPythonの`geopandas`パッケージを使って、各Polygonデータ内のPointデータの数をカウントする`PointInPolygon`クラスを実装してみます。
Pointデータの分布をより大きな空間単位に集計するようなケースは良くあるのですが、私が良く使う`geopandas`ではそういった関数やクラスは実装されていないので、今回の試みに至りました。

また、最近リーダブルコードを読み、何となく読みやすいコード設計を意識したので、「ここの設計をこうした方が良いのでは...?」という考えがありましたら、ぜひコメント頂ければ嬉しいです：）

![](https://miro.medium.com/max/720/1*BgGTyexqblcdUS_dVtg4Hw.png)

# polygonデータ中のpointデータの数をカウントするクラスを設計する

`geopandas`の`GeoDataFrame.sjoin()`メソッドを活用する事で、polygonデータ中のpointデータの数をカウントする`PointInPolygon`クラスを定義していきます。

```python
class PointInPolygon:
    DEFAULT_CRS = "EPSG:4326"

    def __init__(self) -> None:
        pass
```

今回はインスタンス変数に保存するような情報はないので、コンストラクタは空っぽです。
(インスタンス変数はメソッド内でアクセスしやすい利点がありますが、PointInPolygonインスタンス生成後にメモリに残ってしまいます。候補となるのはpointデータとPolygonデータですが、GeoDataFrameはそれなりにサイズがある為、今回はコンストラクタにてインスタンス変数として保存せず、単にメソッドの引数&返り値としてメソッド間で受け渡していく事にしました。)

## `count_point_in_polygon()`メソッドの設計

PointInPolygonインスタンスが生成された後、main側で呼び出されるpublicメソッドとして`count_point_in_polygon()`を設計します。

```python
class PointInPolygon:
    # 省略

    def count_point_in_polygon(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoSeries, pd.Series]:
        gdf_point, gdf_polygon = self._make_same_crs(gdf_point, gdf_polygon)

        gdf_point_joined = self._join_point_with_polygon(gdf_point, gdf_polygon)

        point_count_each_polygon = self._count_num_point_each_polygon(gdf_point_joined, gdf_polygon)

        return point_count_each_polygon
```

このメソッドは引数として、PointデータとPolygonデータを`geopandas.GeoDataFrame`型で受け取ります。返り値は、各Polygonデータ内のPointデータの数が格納された`geopandas.GeoSeries`です。以下の様な感じで、返り値をPolygonデータの新しいカラムとして追加する事を想定しています。

```python
point_in_polygon = PointInPolygon()
gdf_polygon["point_count"] = point_in_polygon.count_point_in_polygon(
    gdf_point,
    gdf_polygon,
)
```

メソッドの処理内容ですが、このpublicメソッドでは3つのPrivateメソッドが呼ばれています。

- まず`_make_same_crs()`で二つのGISデータ(`GeoDataFrame`)のCRS(Coordinate Reference System, 座標参照系)を同じものに揃えます。
  - CRSはGISデータにおける位置の表し方の方法論の事です。このCRSの指定が異なると、正しく空間結合ができなくなってしまいます。
  - CRSに関してより詳しい解説は、[QGIS 第1回　座標参照系（CRS）とは？](https://www.aeroasahi.co.jp/qgis/post/2020/02/crs_01/#:~:text=%E5%BA%A7%E6%A8%99%E5%8F%82%E7%85%A7%E7%B3%BB%EF%BC%88CRS%3ACoordinate,%E5%BA%A7%E6%A8%99%E5%80%A4%E3%81%A7%E6%95%99%E3%81%88%E3%81%BE%E3%81%99%E3%80%82)などを見ると良いかと思います...!
- 続いて`_join_point_with_polygon()`で、空間結合を行います。返り値はPolygonデータの情報が結合されたPointデータになります。
- 最後に、`self._count_num_point_each_polygon()`で、各Polygonデータに含まれるPointデータの数を集計します。

各Privateメソッドの中身は以下のようになっています。

```python
class PointInPolygon:
    # 省略

    def _make_same_crs(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Tuple[gpd.GeoDataFrame, gpd.GeoDataFrame]:
        """二つのGISデータのCRS(Coordinate Reference System, 座標参照系)を同じものに揃えます"""
        gdf_point = gdf_point.to_crs(crs=PointInPolygon.DEFAULT_CRS)
        gdf_polygon = gdf_polygon.to_crs(crs=PointInPolygon.DEFAULT_CRS)
        return gdf_point, gdf_polygon

    def _join_point_with_polygon(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoDataFrame, pd.DataFrame]:
        """PointとPolygonで空間結合を行います。返り値はPolygonデータの情報が結合されたPointデータ"""
        gdf_point_joined = gpd.sjoin(
            left_df=gdf_point,
            right_df=gdf_polygon,
            how="left",
            op="within",
        )

        return gdf_point_joined

    def _count_num_point_each_polygon(
        self,
        gdf_point_joined: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoSeries, pd.Series]:
        """Polygonデータの情報が結合されたPointデータと、Polygonデータを受け取り、各Polygonデータに含まれるPointデータの数を集計"""
        series_point_count_each_polygon_exclude_null = gdf_point_joined.value_counts(subset="index_right")

        df_point_count_each_polygon_exclude_null = pd.DataFrame(
            series_point_count_each_polygon_exclude_null, columns=["point_count_in_polygon"]
        )  # pd.mergeする為にpd.Seriesからpd.DataFrameに変換する必要がある

        df_point_count_each_polygon_include_null = pd.merge(
            left=gdf_polygon,
            right=df_point_count_each_polygon_exclude_null,
            left_index=True,
            right_index=True,
            how="left",
        )["point_count_in_polygon"]

        return df_point_count_each_polygon_include_null.fillna(0)
```

# 最終的に実装されたクラス

```python
from typing import Dict, Tuple, Union

import geopandas as gpd
import pandas as pd


class PointInPolygon:
    DEFAULT_CRS = "EPSG:4326"

    def __init__(self) -> None:
        pass

    def count_point_in_polygon(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoSeries, pd.Series]:
        gdf_point, gdf_polygon = self._make_same_crs(gdf_point, gdf_polygon)

        gdf_point_joined = self._join_point_with_polygon(gdf_point, gdf_polygon)

        point_count_each_polygon = self._count_num_point_each_polygon(gdf_point_joined, gdf_polygon)

        return point_count_each_polygon

    def _make_same_crs(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Tuple[gpd.GeoDataFrame, gpd.GeoDataFrame]:
        """二つのGISデータのCRS(Coordinate Reference System, 座標参照系)を同じものに揃えます"""
        gdf_point = gdf_point.to_crs(crs=PointInPolygon.DEFAULT_CRS)
        gdf_polygon = gdf_polygon.to_crs(crs=PointInPolygon.DEFAULT_CRS)
        return gdf_point, gdf_polygon

    def _join_point_with_polygon(
        self,
        gdf_point: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoDataFrame, pd.DataFrame]:
        """PointとPolygonで空間結合を行います。返り値はPolygonデータの情報が結合されたPointデータ"""
        gdf_point_joined = gpd.sjoin(
            left_df=gdf_point,
            right_df=gdf_polygon,
            how="left",
            op="within",
        )

        return gdf_point_joined

    def _count_num_point_each_polygon(
        self,
        gdf_point_joined: gpd.GeoDataFrame,
        gdf_polygon: gpd.GeoDataFrame,
    ) -> Union[gpd.GeoSeries, pd.Series]:
        """Polygonデータの情報が結合されたPointデータと、Polygonデータを受け取り、各Polygonデータに含まれるPointデータの数を集計"""
        series_point_count_each_polygon_exclude_null = gdf_point_joined.value_counts(subset="index_right")

        df_point_count_each_polygon_exclude_null = pd.DataFrame(
            series_point_count_each_polygon_exclude_null, columns=["point_count_in_polygon"]
        )  # pd.mergeする為にpd.Seriesからpd.DataFrameに変換する必要がある

        df_point_count_each_polygon_include_null = pd.merge(
            left=gdf_polygon,
            right=df_point_count_each_polygon_exclude_null,
            left_index=True,
            right_index=True,
            how="left",
        )["point_count_in_polygon"]

        return df_point_count_each_polygon_include_null.fillna(0)
```

# 呼び出し方(実際に動作確認してみる)

実際に`PointInPolygon`クラスを用いて、各Polygonデータに含まれるPointデータの数を取得してみます。
動作確認で使用するPointデータとPolygonデータは、以下のような、ニューヨーク州におけるWifiスポット(Pointデータ)とニューヨーク州における各Communityの境界線(Polygonデータ)を用います。

![](https://miro.medium.com/max/720/1*BgGTyexqblcdUS_dVtg4Hw.png)

2つのGISデータのファイルパスを指定して`GeoDataFrame`として取得した後、`PointInPolygon`クラスのインスタンスを作成、`count_point_in_polygon()`メソッドを呼び出します。
返り値として各Polygonデータ内のPointデータの数が格納された`geopandas.GeoSeries`を受け取り、Polygonデータの新しいカラムとして追加します。
最終的にPolygonデータの上から5行をコンソール出力しています。

```python
if __name__ == "__main__":
    # 動作確認
    nyc_wifi_geojson = "https://data.cityofnewyork.us/resource/yjub-udmw.geojson"
    nyc_community_districts_geojson = (
        "https://data.cityofnewyork.us/api/geospatial/yfnk-k7r4?method=export&format=GeoJSON"
    )

    gdf_nyc_wifi = gpd.read_file(nyc_wifi_geojson)
    gdf_nyc_community_districts = gpd.read_file(nyc_community_districts_geojson)

    point_in_polygon = PointInPolygon()
    gdf_nyc_community_districts["point_count"] = point_in_polygon.count_point_in_polygon(
        gdf_point=gdf_nyc_wifi,
        gdf_polygon=gdf_nyc_community_districts,
    )
    print(f"[DEBUG]\n{gdf_nyc_community_districts.head()}")
```

コンソール出力された結果は以下です。最も左の`point_count`カラムに各Community内のWifiスポットの数が格納されています:)

```
 boro_cd     shape_area     shape_leng                                           geometry  point_count
0     404   65739662.358  37018.3738576  MULTIPOLYGON (((-73.84751 40.73901, -73.84801 ...          7.0
1     304  56662612.9747  37007.8067266  MULTIPOLYGON (((-73.89647 40.68234, -73.89653 ...          7.0
2     303  79461502.7281  36213.6713934  MULTIPOLYGON (((-73.91805 40.68721, -73.91800 ...         11.0
3     308   45603786.747  38232.8870882  MULTIPOLYGON (((-73.95829 40.67983, -73.95596 ...         10.0
4     316  51768916.7179  32997.5918911  MULTIPOLYGON (((-73.90405 40.67922, -73.90349 ...         11.0
```

# おわりに

今回はPythonの`geopandas`パッケージを使って、各Polygonデータ内のPointデータの数をカウントする`PointInPolygon`クラスを実装してみました。
こういった、Pointデータの分布をより大きな空間単位に集計するようなケースは良くあり、そこそこ汎用性が高いと思うので、私の場合はsetup.pyを記述してpip installして自作パッケージとして再利用しています。

最後までお読みいただき、ありがとうございました...!!
もし空間データの扱いやコードの設計等に少しでも気になる事がありましたら、ぜひカジュアルにコメント頂けると嬉しいです:)

では、お互い、難しくも楽しい分析ライフを満喫しましょう～：）
