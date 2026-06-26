# 第6回：Docker入門

この回では、Docker を使って実験環境を丸ごとパッケージ化する方法を学びます。  
「自分のPCでは動いたのにサーバーでは動かない」問題を解決します。

---

## 目標

- [x] Docker の概念を理解する
- [x] 既存の Docker イメージを使う
- [x] 自分で Dockerfile を書いてイメージをビルドする
- [x] コンテナ内で GPU を使う
- [x] 実践：SAM（Segment Anything Model）を Docker で動かす

---

## ステップ 1：Docker とは何か

### 1.1 問題

```
自分のPC:     Python 3.10 + PyTorch 2.0 + CUDA 11.8 → 動く ✅
先輩のPC:     Python 3.8  + PyTorch 1.13 + CUDA 11.3 → 動かない ❌
サーバー:     Python 3.11 + PyTorch 2.1 + CUDA 12.1 → エラー ❌
```

→ 環境が違うと同じコードが動かない

### 1.2 Docker の解決策

```
┌─────────────────────────────────────────┐
│ Docker コンテナ                          │
│  ┌─────────────────────────────────┐    │
│  │ OS: Ubuntu 22.04                │    │
│  │ Python: 3.10                    │    │
│  │ PyTorch: 2.0                    │    │
│  │ CUDA: 11.8                      │    │
│  │ 自分のコード: train.py          │    │
│  └─────────────────────────────────┘    │
│                                         │
│  → どのマシンでも同じ環境で動く！        │
└─────────────────────────────────────────┘
```

> 💡 **たとえ**: Docker = 「引っ越しのとき、部屋ごと箱詰めして持っていける仕組み」

### 1.3 基本用語

| 用語 | 意味 | たとえ |
|------|------|--------|
| **イメージ (Image)** | 環境の設計図 | レシピ |
| **コンテナ (Container)** | イメージから作った実行環境 | レシピから作った料理 |
| **Dockerfile** | イメージの作り方を書いたファイル | レシピの書かれた紙 |
| **Docker Hub** | イメージの共有サイト | レシピ共有サイト |

---

## ステップ 2：Docker の確認

### 2.1 インストール確認

サーバーで：

```bash
docker --version
```

**期待される出力：**
```
Docker version 24.x.x, build ...
```

GPU 対応の確認：
```bash
nvidia-docker --version
# または
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

---

## ステップ 3：既存イメージを使う

### 3.1 イメージの取得と実行

```bash
# PyTorchの公式イメージを使う
docker run -it --gpus all pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime bash
```

コンテナの中に入ったら：
```bash
python -c "import torch; print(torch.cuda.is_available())"
# → True
```

`exit` でコンテナから出る

### 3.2 Docker の基本コマンド

```bash
# イメージ一覧
docker images

# 実行中のコンテナ一覧
docker ps

# 全コンテナ一覧（停止中も含む）
docker ps -a

# コンテナの停止
docker stop コンテナID

# コンテナの削除
docker rm コンテナID

# イメージの削除
docker rmi イメージ名
```

---

## ステップ 4：自分の Dockerfile を書く

### 4.1 プロジェクト構成

```
my-docker-project/
├── Dockerfile
├── requirements.txt
├── src/
│   └── train.py
└── data/
```

### 4.2 Dockerfile の作成

```dockerfile
# ベースイメージ（PyTorch + CUDA入り）
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime

# 作業ディレクトリの設定
WORKDIR /app

# 必要なライブラリをインストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ソースコードをコピー
COPY src/ ./src/

# デフォルトのコマンド
CMD ["python", "src/train.py"]
```

### 4.3 requirements.txt

```
numpy>=1.24
matplotlib>=3.7
scikit-learn>=1.3
```

### 4.4 イメージのビルド

```bash
docker build -t my-experiment:v1 .
```

- `-t my-experiment:v1` : イメージに名前とバージョンをつける
- `.` : 現在のディレクトリの Dockerfile を使う

### 4.5 コンテナの実行

```bash
# GPU付きで実行
docker run --gpus all my-experiment:v1

# 対話モードで中に入る
docker run --gpus all -it my-experiment:v1 bash

# データフォルダをマウント（コンテナ内からホストのファイルにアクセス）
docker run --gpus all -v $(pwd)/data:/app/data -v $(pwd)/results:/app/results my-experiment:v1
```

---

## ステップ 5：よく使うパターン

### 5.1 開発用コンテナ（マウントして使う）

```bash
# ソースコードをマウントして、コンテナ内で編集・実行
docker run --gpus all -it \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/results:/app/results \
  my-experiment:v1 bash
```

> 💡 `-v ホスト側:コンテナ側` でフォルダを共有。ホスト側で編集した内容がコンテナ内にも反映される。

### 5.2 実験実行用（結果だけ取り出す）

```bash
docker run --gpus all \
  -v $(pwd)/results:/app/results \
  my-experiment:v1 \
  python src/train.py --epochs 100 --lr 0.001
```

### 5.3 Jupyter Notebook を Docker で

```bash
docker run --gpus all -p 8888:8888 \
  -v $(pwd):/app \
  my-experiment:v1 \
  jupyter notebook --ip=0.0.0.0 --allow-root --no-browser
```

→ ブラウザで `http://サーバーIP:8888` にアクセス

---

## ステップ 6：docker compose（複数コンテナの管理）

### 6.1 docker-compose.yml の例

```yaml
version: "3.8"

services:
  training:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./src:/app/src
      - ./data:/app/data
      - ./results:/app/results
    command: python src/train.py
```

### 6.2 実行

```bash
# 起動
docker compose up

# バックグラウンドで起動
docker compose up -d

# ログを見る
docker compose logs -f

# 停止
docker compose down
```

---

## ステップ 7：Dockerfile を書くコツ

### 7.1 レイヤーの順序を意識する

```dockerfile
# 良い例：変わりにくいものを上に
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
WORKDIR /app

# ① ライブラリ（あまり変わらない）→ キャッシュが効く
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ② ソースコード（頻繁に変わる）→ ここだけ再ビルド
COPY src/ ./src/

CMD ["python", "src/train.py"]
```

### 7.2 .dockerignore

不要なファイルをイメージに含めない：

```
# .dockerignore
.venv/
results/
data/
__pycache__/
.git/
*.pyc
```

---

## 確認チェックリスト

- [ ] `docker --version` でバージョンが表示される
- [ ] 既存イメージでコンテナを起動できた
- [ ] コンテナ内で `nvidia-smi` が動作する
- [ ] 自分で Dockerfile を書いてビルドできた
- [ ] `-v` オプションでフォルダをマウントできた
- [ ] コンテナ内でプログラムを実行して結果を取得できた

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `permission denied` | Docker の権限がない | `sudo` を付ける or グループに追加 |
| `CUDA error: no kernel image` | CUDAバージョン不一致 | ベースイメージのCUDAバージョンを確認 |
| `no space left on device` | ディスク容量不足 | `docker system prune` で不要イメージ削除 |
| `port is already allocated` | ポートが使用中 | 別のポート番号を指定 |

---

## まとめ：全6回の振り返り

| 回 | 学んだこと | キーコマンド |
|----|-----------|-------------|
| 第1回 | PC準備・Colab | `print()`, matplotlib |
| 第2回 | Git基本 | `git add`, `git commit`, `git push` |
| 第3回 | SSH接続 | `ssh`, VS Code Remote-SSH |
| 第4回 | GPU計算 | `nvidia-smi`, `torch.cuda` |
| 第5回 | 実験管理 | `tmux`, `.gitignore`, ログ保存, W&B |
| 第6回 | Docker | `docker build`, `docker run`, SAM演習 |

これで「**調査 → 実装 → 実行 → 評価 → まとめ**」に必要な基本ツールが揃いました！

---

## 演習：基盤モデル実践（選択式）

Docker の基本を理解したら、実際の基盤モデルを Docker 環境で動かしてみましょう。  
**以下から自分の研究テーマに近いものを1つ以上選んでください：**

| 演習 | テーマ | 分野 | 内容 |
|------|--------|------|------|
| [SAM で画像領域分割](./exercise-sam-segmentation.md) | Segment Anything Model | 画像（セグメンテーション） | 公式Docker環境でSAM2を動かし、点やボックスで物体を切り出す |
| [Qwen VLM で画像理解](./exercise-qwen-vlm.md) | Qwen2.5-VL | 画像+言語（VLM） | 画像を入力して自然言語で説明・分析させる |
| [音声ノイズ除去](./exercise-audio-denoising.md) | Demucs (Meta) | 音声処理 | ノイズ入り音声をクリーンにする＋ファインチューニング体験 |

> 💡 どの演習も共通して「**Docker環境構築 → 公式モデル導入 → タスク実行**」のパターンを学びます。

---

## 次のステップ（発展）

- **Weights & Biases (W&B)**: 実験の追跡・可視化ツール（第5回で紹介）
- **GitHub Actions**: 自動テスト・自動ビルド
- **VS Code Dev Containers**: Docker + VS Code の統合開発
- **Hydra / MLflow**: 実験設定と結果の管理フレームワーク
