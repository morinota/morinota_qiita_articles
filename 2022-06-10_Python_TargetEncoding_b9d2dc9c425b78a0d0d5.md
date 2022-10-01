<!--
title:   時系列性を考慮したターゲットエンコーディング特徴量の生成をPythonで実装してみた
tags:    Python,TargetEncoding,機械学習,特徴量エンジニアリング
id:      b9d2dc9c425b78a0d0d5
private: false
-->
# はじめに

**時系列性を持つデータに対してターゲットエンコーディング**をしたいと思っていて、いざ実装する時に手が詰まったので、メモ代わりに実装を残しておきます。
「もっと良い方法知ってるのに...」や「実装面でもっとこうするといいのにな」等のImpressionがありましたら、ぜひコメントでご指摘していただければ嬉しいです:)

# ターゲットエンコーディングとは

ターゲットエンコーディングとは何かについて、以下に自分の理解をまとめておきます。

- ターゲットエンコーディングとは、
  - 教師あり機械学習において、**ターゲット(統計学的な文脈では目的変数・被説明変数?)の値を用いて特徴量を生成する**方法論。
  - Kaggleのコンペ等でもよく用いられているみたい。
  - **一方でリークが発生しやすく**、特徴量の生成段階での工夫が必要。

# 本記事で使うデモデータ

本記事では、以下の時系列性を持ったログデータを想定します。

```python
import pandas as pd
df = pd.DataFrame({
        "user_id": ["A", "A", "A", "A", "A", "B", "B", "B", "B", "B"],
        "item_id": ["C", "D", "D", "D", "D", "D", "D", "D", "E", "E"],
        "click": [1, 0, 0, 1, 0, 0, 1, 1, 0, 1],
        "datetime": [
            '2019-12-6 00:00:01',
            '2019-12-6 00:00:02',
            '2019-12-6 00:00:05',
            '2019-12-7 00:00:02',
            '2019-12-7 00:00:04',
            '2019-12-8 00:00:05',
            '2019-12-8 00:00:06',
            '2019-12-9 00:00:07',
            '2019-12-9 00:00:08',
            '2019-12-10 00:00:01',
        ]
    })
```

各カラムの意味は以下を想定しています。

- df: 一定期間、あるユーザ達がアイテムの広告を閲覧した時のクリックしたかどうかのログ。
- user_id : ユーザ毎の一意のid
- item_id : アイテム毎の一意のid
- click : 広告を閲覧した際、clickしたかどうか
- datetime: 広告閲覧が発生した日時

このデータを用いて、ユーザへの広告配信を最適化する(CTRを高める)為に、ユーザの各アイテムに対するクリック率(=ひいてはユーザの嗜好?)を予測するモデルを構築する事を想定します。

リークを気にしない場合のターゲットエンコーディングであれば、単に以下のように、各ユーザ×各アイテムのclickカラムの平均値を取得します。

```python
df_target_enc = df_prev_date.groupby(['user_id', 'item_id'])['click'].mean().reset_index()
df_target_enc.rename(
            columns={'click': 'user_id-item_id-target_enc'}, inplace=True)
df = pd.merge(df, df_target_enc, on=['user_id', 'item_id'], how='left')
```

しかしこの場合だとリークが発生しており、これを特徴量として使ったとしても、学習データに対してのみ高精度、非学習データに対しては精度が大きく下がるようなモデルになってしまうと思われます。

# 時系列性を考慮したターゲットエンコーディング特徴量の生成

そこでリーク発生を防ぐ為に、時系列性を考慮して、各ログよりも時系列が前のログのみを使ってターゲットエンコーディングを実行します。

今回の場合は、**各ログの前日以前のログ**を使って、ターゲットエンコーディング特徴量を作る処理を実装してみました。

以下、コードになります。

```python
def time_seris_target_encoding(df: pd.DataFrame, category_cols: List[str], target_col: str, timeseries_col: str, target_enc_name: str):
    """前日までのログを使ったターゲットエンコーディング。
    一日毎に集計表が保存される。
    """
    # datitimeから日付を取り出す
    df[timeseries_col+'_date'] = pd.to_datetime(df[timeseries_col]).dt.date
    # 日付のユニーク値を取り出す
    date_uniques = df[timeseries_col+'_date'].unique()
    print(date_uniques)
    # 各「日付」での結果を格納するDictを用意
    target_enc_with_date_dict = {date: None for date in date_uniques}

    df_target_enc_with_date = pd.DataFrame()
    for date in date_uniques:
        # ログデータから、dateの前日以前のログのみを抽出
        mask = df[timeseries_col] < pd.to_datetime(date)
        df_prev_date = df[mask]
        # 指定されたカテゴリのターゲットラベルの平均値を集計
        df_target_enc = df_prev_date.groupby(
            category_cols)[target_col].mean().reset_index()
        df_target_enc.rename(
            columns={target_col: target_enc_name}, inplace=True)
        # 元データとのmergeに備えてdateカラムを作る
        df_target_enc['date'] = date
        df_target_enc_with_date = pd.concat(
            [df_target_enc_with_date, df_target_enc])
        print(df_target_enc)

        # 各日付毎のターゲットエンコーディング値をDictに保存
        target_enc_with_date_dict[date] = df_target_enc

    # 各dateでのdf_target_encを格納したdictと、マージしやすいdf_target_enc_with_dateの両方を返す。
    return target_enc_with_date_dict, df_target_enc_with_date


if __name__ == '__main__':
    _, df_target_enc = time_seris_target_encoding(
        df, category_cols=['user_id', 'item_id'],
        target_col='click',
        target_enc_name='user_id-item_id-target_enc',
        timeseries_col='datetime'
        )
    print(df_target_enc)
    # ターゲットエンコーディング特徴量を元データにマージする
    df['date'] = pd.to_datetime(df['datetime']).dt.date
    df = pd.merge(df, df_target_enc, on=[
                  'date', 'user_id', 'item_id'], how='left')
    print(df)
```

結果として、以下のターゲットエンコーディング特徴量が生成されます。

```
  user_id item_id  user_id-item_id-target_enc        date
0       A       C                    1.000000  2019-12-07
1       A       D                    0.000000  2019-12-07
0       A       C                    1.000000  2019-12-08
1       A       D                    0.250000  2019-12-08
0       A       C                    1.000000  2019-12-09
1       A       D                    0.250000  2019-12-09
2       B       D                    0.500000  2019-12-09
0       A       C                    1.000000  2019-12-10
1       A       D                    0.250000  2019-12-10
2       B       D                    0.666667  2019-12-10
3       B       E                    0.000000  2019-12-10
```

元のログデータにマージした結果が以下です。

```
 user_id item_id  click            datetime datetime_date        date  user_id-item_id-target_enc
0       A       C      1 2019-12-06 00:00:01    2019-12-06  2019-12-06                         NaN
1       A       D      0 2019-12-06 00:00:02    2019-12-06  2019-12-06                         NaN
2       A       D      0 2019-12-06 00:00:05    2019-12-06  2019-12-06                         NaN
3       A       D      1 2019-12-07 00:00:02    2019-12-07  2019-12-07                         0.0
4       A       D      0 2019-12-07 00:00:04    2019-12-07  2019-12-07                         0.0
5       B       D      0 2019-12-08 00:00:05    2019-12-08  2019-12-08                         NaN
6       B       D      1 2019-12-08 00:00:06    2019-12-08  2019-12-08                         NaN
7       B       D      1 2019-12-09 00:00:07    2019-12-09  2019-12-09                         0.5
8       B       E      0 2019-12-09 00:00:08    2019-12-09  2019-12-09                         NaN
9       B       E      1 2019-12-10 00:00:01    2019-12-10  2019-12-10                         0.0
```

なんか、NaNがとても多いですね...。

2019-12-06のログに関してはそれ以前のデータが溜まっていないので仕方ないとして、
2019-12-08や2019-12-09にもNaNが含まれているのは、そのログのユーザ＊アイテムの組み合わせのログがこれまで発生していないからですね。
まあでもTree系のモデルならNaNにも対応してるからヨシ！
今回はログデータが10行&５日間のサンプルなので、このようなNaNばかりの特徴量になってしまいましたが、より多くのログデータを扱う場合は効果を発揮するのではないでしょうか？
なお仮に予測したいデータが2019-12-11以降の場合は、訓練データ期間の最終日までで作成したターゲットエンコーディング特徴量を使えますし、毎回前日に次の日のクリック率を予測したい場合はたぶんターゲットエンコーディング特徴量を更新していく事も可能ですね...!