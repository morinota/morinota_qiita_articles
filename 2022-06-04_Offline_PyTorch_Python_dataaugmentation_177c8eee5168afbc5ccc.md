<!--
title:   画像認識におけるOffline Data AugmentationをPytorchで実装してみる
tags:    Offline,PyTorch,Python,dataaugmentation,画像認識
id:      177c8eee5168afbc5ccc
private: false
-->
# はじめに

大学院での研究活動において**画像認識タスクにおけるoffline data augmentation**を適用してみようと思い、Googleしたところ、online data augmentationの記事が多く、パッとoffline data augmentationを実装する方法が分からなかったので、ちょろちょろとPytorchのDatasetを用いて実装してみました。

最終的にたどり着いた実装方法としては、非常にシンプルなのですが、以下の通りです。

- 手順1: 元の画像データをDatasetで読み込む
- 手順2: Data augmentationを実行
- 手順3: それを再度ディレクトリにExport
- 手順4: 「元画像＋ augmentatedされた画像」を再度Datasetで読み込んでモデリングに使用する

もし「他にもっと良い実装方法を知ってる！」という方がいましたら、ぜひコメントいただければ嬉しいです：）

# そもそも実装したい「Offline Data Augmentation」とは?

Data AugumentationとOffline & Onlineに関して、現時点での自分の理解を軽くまとめます(間違ってたら優しめに教えていただけると嬉しいです)

- Data Augmentation : 学習用のデータに対して、何らかの「変換」を施すことでデータを水増しす方法論。特に画像データに対しては、画像の"反転"や"回転"の「変換」が施される。Augumentationを実行するタイミングによって、Offline AugumentationとOnline Augumentationに分類でき、それぞれ目的が異なる。
- Offline Data Augmentation :
  - 元画像に対して学習前にData Augmentationを適用し、単純に画像の枚数を増やす手法。
  - タイミング：学習前。事前に。
  - 目的：学習データ数の水増し？
- Online Data Augmentation :
  - モデル学習時に、ミニバッチ毎にDataLoaderからモデルにデータを流し込む際に、Data Augmentationを適用する手法。
  - タイミング：ミニバッチ毎にデータをモデルに入力する直前。
  - 目的：汎化性能の向上。
    - 学習データ数自体は同じ。
    - 同じミニバッチでも、各Epoch毎に、入力される画像が変わり得る(各画像に対するData augmentation適用の有無を確率的に決定する事で！)

# いざ実装

前述した通り、最終的にたどり着いた実装方法としては、以下の非常にシンプルなものでした。

- 手順0: Data Augmentation用のDatasetを定義する。
- 手順1: Data augmentation用のtransformsを用意。
- 手順2: 元の画像データをDatasetで読み込み、Data Augmentation適用
- 手順3: それを再度ディレクトリにExport
- 手順4: 「元画像＋ augmentatedされた画像」を再度Datasetで読み込んでモデリングに使用する

## 前提条件

## 手順0: Data Augmentation用のDatasetを定義する。

まず、以下がData Augmentation用のDatasetのソースコードになります。
Datasetクラスの定義に必要な`__init__()`, `__getitem__()`, `__len__()`に加えて、各ラベル毎のディレクトリから画像を吸い上げる為に`_find_classes()`, `_make_dataset()`の二つのメソッドを定義しています。

後者の二つのメソッドは、torchvision.datasets.ImageFolderクラスの定義を参考にしています。というかほぼほぼ写経です！
各メソッドの概要としては

- `__init__()`:Datasetクラスのコンストラクタ
- `_find_classes()`：コンストラクタで渡されたrootディレクトリから、各クラスディレクトリを探し、画像分類における各クラスラベルを取得する
- `_make_dataset()`：オリジナルの画像のファイルパス一覧と対応するクラスラベルを取得する。
- `__getitem__()`:index のサンプルが要求されたときに返す処理を実装
- `__len__()`:Datasetのサンプルの長さを返す。

**唯一アレンジした点**としては、あくまで**オリジナルの画像(Data Augmentationで水増しされた画像ではない)のみを各クラスディレクトリから読み込む**為に、`_make_dataset()`内で**正規表現を使って読み込む画像を選別**しています。

```python
class Dataset_augmentation(Dataset):
    def __init__(self, root: str, transform=None) -> None:
        self.root = root

        self.classes, self.class_to_idx = self._find_classes(root)
        self.samples = self._make_dataset()
        self.targets = [s[1] for s in self.samples]
        self.transform = transform

    def _find_classes(self, img_dir: str) -> Tuple[List, Dict]:
        """コンストラクタで渡されたrootディレクトリから、各クラスディレクトリを探し、画像分類における各クラスラベルを取得するinnor関数。
        """
        classes = [d.name for d in os.scandir(
            img_dir) if d.is_dir()]  # 各クラスのdirのパスを取得
        classes.sort()
        class_to_idx = {classes[i]: i for i in range(len(classes))}
        return classes, class_to_idx

    def _make_dataset(self):
        """指定したディレクトリ内の画像ファイルのパス一覧を取得するinnor関数
        """
        images = []
        for class_name in sorted(self.class_to_idx.keys()):
            class_dir = os.path.join(self.root, class_name)
            for root, _, fnames in sorted(os.walk(class_dir, followlinks=True)):
                for fname in sorted(fnames):

                    # オリジナルの画像だけ取ってくる。
                    if re.compile(pattern='[0-9]+\.(jpg|JPG)').search(fname):
                        path = os.path.join(root, fname)
                        item = (path, self.class_to_idx[class_name])
                        images.append(item)

        return images

    def __getitem__(self, index):
        """index のサンプルが要求されたときに返す処理を実装

        Parameters
        ----------
        index : _type_
            _description_

        Returns
        -------
        _type_
            _description_
        """

        path, target = self.samples[index]
        # 入力側の画像データを配列で読み込み
        image: Image.Image = Image.open(path)
        image = image.convert(mode='RGB')
        if self.transform is not None:
            image = self.transform(image)

        return image, target  # 返値

    def __len__(self):
        return len(self.samples)
```

## 手順1: Data augmentation用のtransformsを用意。

続いて、Data Augmentation用のtransformsを用意していきます。
今回は、「**Data Augmentation手法を一つ引数で渡して、それに該当する処理のtransforms.Composeオブジェクトを返す関数**」として`get_transform_for_data_augmentation()`関数を定義しました。

実際に関数を使用する際は、適用したいData Augmentation手法名をListに格納しておいて、ループ処理で関数を回していく事を想定しています。こんな風に...

```python
 augumentated_type_list = [
        'horizontal_frip',
        'random_rotation',  # ランダムに回転を行う
        # 'random_erasing',
        'random_affine',  # ランダムにアフィン変換を行う。
        'random_perspective',  # ランダムに射影変換を行う.
        'color_jitter',  # ランダムに明るさ、コントラスト、彩度、色相を変化させる.
        'random_resized_crop',  # ランダムに切り抜いた後にリサイズを行う.
    ]
  for data_augumentated_type in augumentated_type_list:
      data_transform = get_transform_for_data_augmentation(
                augumentated_type=data_augumentated_type
            )
```

なお、定義した`get_transform_for_data_augmentation()`関数のソースコードは以下です。(if文ばかりでかっこよくはない気がします...)
引数`augumentated_type`に応じて、`transforms.Compose`オブジェクトの生成時に渡すリスト`transforms_data_aug`の中身を変更させるようにしています。

```python
def get_transform_for_data_augmentation(augumentated_type='horizontal_frip')->transforms.Compose:
    transforms_data_aug = []
    if augumentated_type == 'horizontal_frip':
        transforms_data_aug.append(
            transforms.RandomHorizontalFlip(p=1.0)
        )
    if augumentated_type == 'random_rotation':
        transforms_data_aug.append(
            transforms.RandomRotation(degrees=[-15, 15])
        )
    if augumentated_type == 'random_erasing':
        transforms_data_aug.append(
            transforms.RandomErasing(p=1)
        )
    if augumentated_type == 'random_perspective':
        transforms_data_aug.append(
            transforms.RandomPerspective(distortion_scale=0.5,
                                         p=1.0, interpolation=3)
        )
    if augumentated_type == 'random_resized_crop':
        transforms_data_aug.append(
            transforms.RandomResizedCrop(
                size=150, scale=(0.08, 1.0), ratio=(3/4, 4/3)
            )
        )
    if augumentated_type == 'color_jitter':
        transforms_data_aug.append(
            transforms.ColorJitter(
                brightness=0.5, contrast=0.5, saturation=0.5
            )
        )

    if augumentated_type == 'random_affine':
        transforms_data_aug.append(
            transforms.RandomAffine(
                degrees=[-10, 10], translate=(0.1, 0.1), scale=(0.5, 1.5)
            )
        )

    return transforms.Compose(transforms=transforms_data_aug)
```

## 手順2: 元の画像データをDatasetで読み込み、Data Augmentation適用 & 手順3: それを再度ディレクトリにExport

実際にData Augmentationを適用する際は以下のようになります。

```python
augumentated_type_list = [
        'horizontal_frip',
        'random_rotation',  # ランダムに回転を行う
        # 'random_erasing',
        'random_affine',  # ランダムにアフィン変換を行う。
        'random_perspective',  # ランダムに射影変換を行う.
        'color_jitter',  # ランダムに明るさ、コントラスト、彩度、色相を変化させる.
        'random_resized_crop',  # ランダムに切り抜いた後にリサイズを行う.
    ]
image_dir = '各クラスディレクトリが入ったrootディレクトリのパス'
for data_augumentated_type in augumentated_type_list:
    # transfroms.Composeオブジェクトを取得
    data_transform = get_transform_for_data_augmentation(
                augumentated_type=data_augumentated_type
            )
    # transformsを引数に渡して、Datasetオブジェクトを生成
    dataset_augmentated = Dataset_augmentation(
                root=image_dir, transform=data_transform)

    # data augmentationが適用された各画像をimage_dirにexport
    save_augumentated_images(
                dataset_augmentated, augumentated_type=(str(i) + data_augumentated_type))

```

また、ここでは「data augmentationが適用された各画像をimage*dirにexport」する処理として、以下の`save_augumentated_images`関数を定義しています。
出力する各画像のファイル名は`オリジナルのファイル名*{augumentated_type}.jpg`となっています。

```python
def save_augumentated_images(dataset_augmentated: Dataset_augmentation, augumentated_type: str = 'horizontal_frip'):
    idx_to_class = {idx: label for label,
                    idx in dataset_augmentated.class_to_idx.items()}

    # Augmentatedされた各画像を保存
    for i in range(dataset_augmentated.__len__()):
        # filepath生成の処理
        base_path = dataset_augmentated.samples[i][0]
        base_file_name = os.path.basename(base_path)
        base_dir_name = os.path.dirname(base_path)
        augumented_file_name = base_file_name.split(
            '.')[0] + f'_{augumentated_type}.jpg'

        file_path_augumented = os.path.join(
            base_dir_name, augumented_file_name)

        # augumentated した画像を保存
        image_object: Image.Image = dataset_augmentated[i][0]
        image_object.save(fp=file_path_augumented)
```

## 手順4: 「元画像＋ augmentatedされた画像」を再度Datasetで読み込んでモデリングに使用する

この手順に関しては、手順0「Data Augmentation用のDatasetを定義する。」で定義された`Dataset_augmentation`クラスの`_make_dataset()`メソッドの正規表現の選別の箇所を削ればOKです。というか`torchvision.datasets.ImageFolder`クラスを使えば全く問題ありません！

## おまけの処理「ストレージの圧迫を防ぐ為、Data Augmentationされた画像のみ削除する」

この処理は不要かもしれませんが、一応、作成してみました。
私の場合は、「Offline Data Augmentation＝＞モデル訓練」の後に、下記の関数を実行して、Augmentationされていないオリジナルの画像のみが最終的にディレクトリに残るようにしています：）(単にストレージの圧迫を防ぐ為です)

```python
def delete_data_augmentated_files(image_dir):
    dataset_augmentated = Dataset_augmentation(
        root=image_dir, transform=None)

    def _delete_file(path: str):
        os.remove(path)
    # pathを一つずつ見ていって、オリジナル以外を削除
    for class_name in sorted(dataset_augmentated.class_to_idx.keys()):
        class_dir = os.path.join(dataset_augmentated.root, class_name)
        for root, _, fnames in sorted(os.walk(class_dir, followlinks=True)):
            for fname in sorted(fnames):
                # オリジナルの画像だけは残す
                if re.compile(pattern='[0-9]+\.(jpg|JPG)').search(fname):
                    pass
                else:  # data augmentated した画像は削除
                    file_path = os.path.join(root, fname)
                    _delete_file(path=file_path)
```

# おわりに

今回はPytorchのDatasetを用いてoffline data augmentationをちょろちょろと実装してみました。そもそもOnline Augmentationの記事が多いというのは、**もしかするとOffline Augmentationはあまり使われない方法なのかも**しれないなと思い始めています。

まだ画像データの扱いに慣れていませんが、今後研究活動でも適用してきたい技術分野なので、色々と試していきたいと思います。
最後までお読みいただき、本当にありがとうございました：）

# 参考

- https://github.com/chainer/chainercv/issues/683
- https://medium.com/nanonets/how-to-use-deep-learning-when-you-have-limited-data-part-2-data-augmentation-c26971dc8ced
- https://www.codexa.net/data_augmentation_python_keras/