# 演習：SAM（Segment Anything Model）で画像領域分割

この演習では、Meta が公開した **SAM 2**（Segment Anything Model 2）を Docker 環境で動かし、画像の領域分割（セグメンテーション）タスクを体験します。

---

## この演習で学ぶこと

- 公式推奨の Docker 環境でモデルを動かす流れ
- 大規模事前学習モデル（Foundation Model）の利用方法
- プロンプトベースのセグメンテーションの仕組み

---

## SAM とは

**Segment Anything Model (SAM)** は Meta AI が開発した画像セグメンテーションの基盤モデルです。

```
入力: 画像 + プロンプト（点、ボックス、テキスト）
  ↓
SAM モデル（事前学習済み）
  ↓
出力: セグメンテーションマスク（物体の輪郭）
```

> 💡 **ポイント**: 学習データを用意しなくても、点やボックスを指定するだけで「そこにある物体」を切り出せます。研究でのアノテーション作業や、ナノシートの領域分割などに応用できます。

---

## ステップ 1：プロジェクトの準備

### 1.1 作業ディレクトリの作成

```bash
mkdir -p ~/sam-docker-exercise && cd ~/sam-docker-exercise
```

### 1.2 Dockerfile の作成

```dockerfile
# Dockerfile
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-devel

WORKDIR /workspace

# システムパッケージ
RUN apt-get update && apt-get install -y \
    git \
    wget \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# SAM 2 のインストール
RUN git clone https://github.com/facebookresearch/sam2.git /workspace/sam2
WORKDIR /workspace/sam2
RUN pip install -e .
RUN pip install matplotlib opencv-python jupyter

# チェックポイントのダウンロード（tiny: 約155MB）
RUN mkdir -p checkpoints && \
    wget -q -O checkpoints/sam2.1_hiera_tiny.pt \
    https://dl.fbaipublicfiles.com/segment_anything_2/092824/sam2.1_hiera_tiny.pt

WORKDIR /workspace
```

### 1.3 requirements.txt（追加パッケージ）

```
matplotlib
opencv-python
numpy
```

---

## ステップ 2：Docker イメージのビルド

```bash
cd ~/sam-docker-exercise
docker build -t sam2-exercise:v1 .
```

ビルドには数分かかります（PyTorch イメージのダウンロード + SAM2 インストール）。

**確認：**
```bash
docker images | grep sam2
```

```
sam2-exercise   v1   xxxxxxxxxxxx   xx seconds ago   約8GB
```

---

## ステップ 3：コンテナの起動

```bash
docker run --gpus all -it \
  -v $(pwd)/images:/workspace/images \
  -v $(pwd)/results:/workspace/results \
  sam2-exercise:v1 bash
```

| オプション | 意味 |
|-----------|------|
| `--gpus all` | GPU を使用可能にする |
| `-it` | 対話モードで入る |
| `-v $(pwd)/images:/workspace/images` | 入力画像フォルダをマウント |
| `-v $(pwd)/results:/workspace/results` | 結果の保存先をマウント |

コンテナ内で確認：
```bash
python -c "import sam2; print('SAM2 ready!')"
python -c "import torch; print(f'CUDA: {torch.cuda.is_available()}')"
```

---

## ステップ 4：テスト画像の準備

コンテナの**外側**（ホスト側）で：

```bash
mkdir -p ~/sam-docker-exercise/images ~/sam-docker-exercise/results
```

テスト画像を用意する方法：
- 自分の画像を `images/` フォルダに置く
- または、SAM2リポジトリのサンプルを使う（コンテナ内で）：

```bash
# コンテナ内で
cp /workspace/sam2/notebooks/images/truck.jpg /workspace/images/
```

---

## ステップ 5：SAM で推論を実行する

コンテナ内で Python スクリプトを作成：

```bash
cat << 'EOF' > /workspace/run_sam.py
import numpy as np
import torch
import matplotlib.pyplot as plt
from sam2.build_sam import build_sam2
from sam2.sam2_image_predictor import SAM2ImagePredictor
from PIL import Image
import os

# デバイス設定
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Device: {device}")

# モデルの読み込み
sam2_checkpoint = "/workspace/sam2/checkpoints/sam2.1_hiera_tiny.pt"
model_cfg = "configs/sam2.1/sam2.1_hiera_t.yaml"

sam2_model = build_sam2(model_cfg, sam2_checkpoint, device=device)
predictor = SAM2ImagePredictor(sam2_model)

# 画像の読み込み
image_path = "/workspace/images/truck.jpg"
image = np.array(Image.open(image_path))
print(f"Image shape: {image.shape}")

# 画像をモデルに設定
predictor.set_image(image)

# ポイントプロンプトで推論（画像の中心付近を指定）
h, w = image.shape[:2]
input_point = np.array([[w // 2, h // 2]])  # 画像中心
input_label = np.array([1])  # 1 = 前景

masks, scores, logits = predictor.predict(
    point_coords=input_point,
    point_labels=input_label,
    multimask_output=True,  # 複数候補を出力
)

print(f"Masks generated: {len(masks)}")
print(f"Scores: {scores}")

# 結果の可視化
fig, axes = plt.subplots(1, 4, figsize=(20, 5))

# 元画像
axes[0].imshow(image)
axes[0].plot(input_point[0, 0], input_point[0, 1], "r*", markersize=15)
axes[0].set_title("Input (red star = prompt)")
axes[0].axis("off")

# 各マスク候補
for i, (mask, score) in enumerate(zip(masks, scores)):
    axes[i + 1].imshow(image)
    axes[i + 1].imshow(mask, alpha=0.5, cmap="jet")
    axes[i + 1].set_title(f"Mask {i+1} (score: {score:.3f})")
    axes[i + 1].axis("off")

plt.tight_layout()
os.makedirs("/workspace/results", exist_ok=True)
plt.savefig("/workspace/results/sam_result.png", dpi=150, bbox_inches="tight")
plt.close()

print("Result saved to /workspace/results/sam_result.png")
EOF
```

実行：
```bash
cd /workspace/sam2 && python /workspace/run_sam.py
```

---

## ステップ 6：バウンディングボックスで推論

ポイントだけでなく、ボックスでも領域を指定できます：

```bash
cat << 'EOF' > /workspace/run_sam_box.py
import numpy as np
import torch
import matplotlib.pyplot as plt
from sam2.build_sam import build_sam2
from sam2.sam2_image_predictor import SAM2ImagePredictor
from PIL import Image
import os

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# モデル読み込み
sam2_checkpoint = "/workspace/sam2/checkpoints/sam2.1_hiera_tiny.pt"
model_cfg = "configs/sam2.1/sam2.1_hiera_t.yaml"
sam2_model = build_sam2(model_cfg, sam2_checkpoint, device=device)
predictor = SAM2ImagePredictor(sam2_model)

# 画像読み込み
image_path = "/workspace/images/truck.jpg"
image = np.array(Image.open(image_path))
predictor.set_image(image)

# バウンディングボックスで推論 [x_min, y_min, x_max, y_max]
h, w = image.shape[:2]
input_box = np.array([w * 0.1, h * 0.1, w * 0.9, h * 0.9])

masks, scores, _ = predictor.predict(
    box=input_box[None, :],
    multimask_output=False,
)

# 可視化
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

axes[0].imshow(image)
rect = plt.Rectangle(
    (input_box[0], input_box[1]),
    input_box[2] - input_box[0],
    input_box[3] - input_box[1],
    linewidth=2, edgecolor="r", facecolor="none"
)
axes[0].add_patch(rect)
axes[0].set_title("Input (red box = prompt)")
axes[0].axis("off")

axes[1].imshow(image)
axes[1].imshow(masks[0], alpha=0.5, cmap="jet")
axes[1].set_title(f"Segmentation (score: {scores[0]:.3f})")
axes[1].axis("off")

plt.tight_layout()
os.makedirs("/workspace/results", exist_ok=True)
plt.savefig("/workspace/results/sam_box_result.png", dpi=150, bbox_inches="tight")
plt.close()

print("Result saved to /workspace/results/sam_box_result.png")
EOF
```

```bash
cd /workspace/sam2 && python /workspace/run_sam_box.py
```

---

## ステップ 7：結果の確認

コンテナから出た後、ホスト側で：

```bash
ls ~/sam-docker-exercise/results/
# sam_result.png
# sam_box_result.png
```

VS Code でファイルを開いて結果を確認できます。または `scp` で手元の PC にコピー：

```bash
# 手元のPCから
scp spark-jotoku:~/sam-docker-exercise/results/*.png ./
```

---

## ステップ 8：docker compose で管理する

繰り返し実行するなら `docker-compose.yml` が便利です。まず、ステップ5で作成した `run_sam.py` をホスト側にも保存しておきます：

```bash
# ホスト側で run_sam.py を保存（ステップ5の内容をファイルに書き出す）
# ※ コンテナ内で作成したスクリプトはコンテナを削除すると消えるため、
#   ホスト側にも保存して volume マウントで使えるようにする
```

```yaml
# docker-compose.yml
services:
  sam:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./images:/workspace/images
      - ./results:/workspace/results
      - ./run_sam.py:/workspace/run_sam.py
    command: bash -c "cd /workspace/sam2 && python /workspace/run_sam.py"
```

```bash
# ビルドと実行
docker compose up --build

# 対話モードで入りたい場合
docker compose run sam bash
```

---

## チャレンジ課題

### 課題 1：自分の画像で試す

1. 自分の画像（研究の顕微鏡画像など）を `images/` に置く
2. `run_sam.py` のプロンプト位置を調整して実行
3. 結果を観察：SAM はどの程度正確に領域を切り出せるか？

### 課題 2：複数ポイントで精度を上げる

```python
# 前景ポイントを増やす
input_point = np.array([[100, 200], [150, 250], [200, 200]])
input_label = np.array([1, 1, 1])  # すべて前景

# 背景ポイントも追加
input_point = np.array([[100, 200], [150, 250], [50, 50]])
input_label = np.array([1, 1, 0])  # 0 = 背景
```

### 課題 3：W&B と組み合わせる

第5回で学んだ W&B を使って、プロンプト条件と結果スコアを記録してみましょう：

```python
import wandb

wandb.init(project="jotoku-sam-experiment")

wandb.log({
    "prompt_type": "point",
    "num_points": len(input_point),
    "best_score": float(scores.max()),
    "image": wandb.Image(image),
    "mask": wandb.Image(masks[0] * 255),
})

wandb.finish()
```

---

## まとめ

| ステップ | やったこと |
|----------|-----------|
| Dockerfile 作成 | PyTorch + SAM2 の環境を定義 |
| イメージビルド | `docker build` で環境を構築 |
| コンテナ起動 | GPU + ボリュームマウントで実行 |
| モデル読み込み | 公式チェックポイントの利用 |
| 推論実行 | ポイント/ボックスプロンプトで領域分割 |
| 結果取得 | マウントしたフォルダ経由で結果を取り出す |

### Docker を使うメリット（この演習での実感）

- SAM2 の複雑な依存関係（CUDA、PyTorch、OpenCV等）を Dockerfile 一つで管理
- チームメンバーが同じ `docker build` で同じ環境を再現できる
- サーバーのシステム環境を汚さない

---

## 参考リンク

- [SAM 2 公式リポジトリ](https://github.com/facebookresearch/sam2)
- [SAM 2 論文](https://ai.meta.com/research/publications/sam-2-segment-anything-in-images-and-videos/)
- [Docker 公式ドキュメント](https://docs.docker.com/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)

---

## Google Colab で実行する場合

Colab では Docker は使えませんが、SAM 2 を直接インストールして同じ推論を試せます：

```python
# Colab での実行
!pip install git+https://github.com/facebookresearch/sam2.git
!wget -q -O sam2.1_hiera_tiny.pt https://dl.fbaipublicfiles.com/segment_anything_2/092824/sam2.1_hiera_tiny.pt

# あとは run_sam.py と同じコードで実行可能
```

> 💡 Colab で動作確認 → DGX-Spark の Docker 環境で本格実行、という流れが効率的です。
