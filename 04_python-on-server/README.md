# 第4回：サーバー上でPythonを動かす

この回では、DGX-Spark サーバー上で Python の仮想環境を作り、PyTorch で GPU を使ったプログラムを実行します。

---

## 目標

- [x] Python 仮想環境 (venv) を作成・管理する
- [x] pip でライブラリをインストールする
- [x] GPU の状態を確認する
- [x] PyTorch で GPU を使った計算を実行する

---

## ステップ 1：サーバーに接続

前回設定した SSH で接続：

```bash
ssh spark-jotoku
```

VS Code Remote-SSH で接続してもOKです。

---

## ステップ 2：Python のバージョン確認

```bash
python3 --version
```

**期待される出力：**
```
Python 3.10.x  （またはそれ以上）
```

---

## ステップ 3：仮想環境 (venv) の作成

### 3.1 なぜ仮想環境が必要？

```
システム全体の Python（触ると壊れる可能性あり）
    │
    ├── プロジェクトA の仮想環境（numpy 1.24, torch 2.0）
    ├── プロジェクトB の仮想環境（numpy 1.26, torch 2.1）
    └── プロジェクトC の仮想環境（tensorflow 2.15）
```

> 💡 **ポイント**: プロジェクトごとに独立した環境を作ることで、ライブラリのバージョン衝突を防ぎます。

### 3.2 プロジェクトフォルダを作成

```bash
mkdir ~/my-gpu-project
cd ~/my-gpu-project
```

### 3.3 仮想環境を作成

```bash
python3 -m venv .venv
```

> `.venv` というフォルダが作成されます（中にPython環境一式が入る）

### 3.4 仮想環境を有効化（activate）

```bash
source .venv/bin/activate
```

プロンプトの先頭に `(.venv)` が表示されれば成功：
```
(.venv) user@spark-jotoku:~/my-gpu-project$
```

### 3.5 仮想環境を無効化（deactivate）

```bash
deactivate
```

> 💡 **ポイント**: 作業を始めるときは `source .venv/bin/activate`、終わるときは `deactivate`。忘れがちなので注意！

---

## ステップ 4：ライブラリのインストール

### 4.1 pip の基本

仮想環境を有効化した状態で：

```bash
# pip 自体を最新にする
pip install --upgrade pip

# ライブラリをインストール
pip install numpy matplotlib
```

### 4.2 PyTorch のインストール

```bash
pip install torch torchvision
```

> ⚠️ DGX-Spark の CUDA バージョンによっては別のコマンドが必要な場合があります。  
> 先生の指示に従ってください。

### 4.3 インストール済みライブラリの確認

```bash
pip list
```

### 4.4 requirements.txt で管理

インストールしたライブラリを記録しておく：

```bash
pip freeze > requirements.txt
```

他のPCや環境で同じライブラリを入れたいとき：

```bash
pip install -r requirements.txt
```

---

## ステップ 5：GPU の確認

### 5.1 nvidia-smi コマンド

```bash
nvidia-smi
```

**出力例：**
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.xx.xx    Driver Version: 535.xx.xx    CUDA Version: 12.x     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVIDIA GH200 ...    On   | 00000000:01:00.0 Off |                    0 |
| N/A   35C    P0    45W / 900W |    500MiB / 97871MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

確認ポイント：
- **GPU Name**: GPUの種類
- **Memory-Usage**: メモリの使用状況
- **GPU-Util**: 使用率（0%なら空いている）

### 5.2 PyTorch から GPU を確認

Python を起動して確認：

```bash
python3
```

```python
import torch

# CUDAが使えるか確認
print(torch.cuda.is_available())  # True なら OK

# GPUの名前
print(torch.cuda.get_device_name(0))

# GPUの数
print(torch.cuda.device_count())
```

`Ctrl + D` で Python を終了

---

## ステップ 6：GPU を使った計算を実行

### 6.1 スクリプトの作成

VS Code または `nano` で `gpu_test.py` を作成：

```python
# gpu_test.py - GPU動作確認スクリプト
import torch
import time

# --- デバイスの設定 ---
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"使用デバイス: {device}")

if device.type == "cuda":
    print(f"GPU名: {torch.cuda.get_device_name(0)}")
    print(f"GPUメモリ: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")

# --- CPU vs GPU の速度比較 ---
size = 10000

# CPU で行列計算
print("\n--- CPU ---")
a_cpu = torch.randn(size, size)
b_cpu = torch.randn(size, size)

start = time.time()
c_cpu = torch.mm(a_cpu, b_cpu)
cpu_time = time.time() - start
print(f"行列積 ({size}x{size}): {cpu_time:.3f} 秒")

# GPU で行列計算
if device.type == "cuda":
    print("\n--- GPU ---")
    a_gpu = torch.randn(size, size, device=device)
    b_gpu = torch.randn(size, size, device=device)
    
    # ウォームアップ
    torch.mm(a_gpu, b_gpu)
    torch.cuda.synchronize()
    
    start = time.time()
    c_gpu = torch.mm(a_gpu, b_gpu)
    torch.cuda.synchronize()
    gpu_time = time.time() - start
    print(f"行列積 ({size}x{size}): {gpu_time:.3f} 秒")
    
    print(f"\n速度比: GPU は CPU の {cpu_time / gpu_time:.1f} 倍高速")
```

### 6.2 実行

```bash
python gpu_test.py
```

**期待される出力例：**
```
使用デバイス: cuda
GPU名: NVIDIA GH200 ...
GPUメモリ: 96.0 GB

--- CPU ---
行列積 (10000x10000): 4.521 秒

--- GPU ---
行列積 (10000x10000): 0.089 秒

速度比: GPU は CPU の 50.8 倍高速
```

> 💡 **ポイント**: 大きな行列計算ではGPUが圧倒的に速い。深層学習（ディープラーニング）ではこの並列計算能力を活かします。

---

## ステップ 7：簡単なニューラルネットワーク

### 7.1 スクリプトの作成

`simple_nn.py` を作成：

```python
# simple_nn.py - 簡単なニューラルネットワーク
import torch
import torch.nn as nn
import torch.optim as optim

# デバイスの設定
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"使用デバイス: {device}")

# --- データの作成（y = 2x + 1 を学習させる） ---
x_train = torch.linspace(-5, 5, 100).reshape(-1, 1).to(device)
y_train = (2 * x_train + 1 + torch.randn_like(x_train) * 0.5).to(device)

# --- モデルの定義 ---
model = nn.Sequential(
    nn.Linear(1, 16),
    nn.ReLU(),
    nn.Linear(16, 1)
).to(device)

# --- 学習 ---
optimizer = optim.Adam(model.parameters(), lr=0.01)
loss_fn = nn.MSELoss()

print("学習開始...")
for epoch in range(500):
    pred = model(x_train)
    loss = loss_fn(pred, y_train)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    if (epoch + 1) % 100 == 0:
        print(f"  Epoch {epoch+1:4d} | Loss: {loss.item():.4f}")

# --- テスト ---
model.eval()
with torch.no_grad():
    test_x = torch.tensor([[3.0]], device=device)
    pred_y = model(test_x)
    print(f"\n入力 x=3.0 → 予測 y={pred_y.item():.2f}（正解: 7.0）")

print("完了！")
```

### 7.2 実行

```bash
python simple_nn.py
```

**期待される出力例：**
```
使用デバイス: cuda
学習開始...
  Epoch  100 | Loss: 0.3521
  Epoch  200 | Loss: 0.2845
  Epoch  300 | Loss: 0.2612
  Epoch  400 | Loss: 0.2534
  Epoch  500 | Loss: 0.2501

入力 x=3.0 → 予測 y=6.95（正解: 7.0）
完了！
```

---

## 確認チェックリスト

- [ ] 仮想環境を作成・有効化できた（`source .venv/bin/activate`）
- [ ] `pip install` でライブラリをインストールできた
- [ ] `nvidia-smi` で GPU の状態を確認できた
- [ ] `torch.cuda.is_available()` が `True` を返した
- [ ] `gpu_test.py` を実行して CPU と GPU の速度差を確認した
- [ ] `simple_nn.py` を実行してニューラルネットワークが学習できた
- [ ] 演習1：二値分類が正しく動作した
- [ ] 演習2：多値分類が正しく動作した
- [ ] 演習3：MNISTの深層学習でテスト精度97%以上を達成した

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `torch.cuda.is_available()` が `False` | CUDA未対応のPyTorch / ドライバ問題 | `pip install torch` のコマンドを確認 |
| `CUDA out of memory` | GPUメモリ不足 | バッチサイズを小さくする / 他のプロセスを確認 |
| `ModuleNotFoundError` | ライブラリ未インストール | `pip install ライブラリ名` |
| `(.venv)` が表示されない | 仮想環境が有効化されていない | `source .venv/bin/activate` |

---

---

## 分類演習（ステップ 8〜10）

ステップ 7 までで PyTorch の基本を学びました。ここからは、実際の分類問題に取り組みます。

| 演習 | テーマ | 内容 |
|------|--------|------|
| [演習1](./exercise-01_binary-classification.md) | 二値分類 | Iris 2クラス、シグモイド関数、BCELoss |
| [演習2](./exercise-02_multi-classification.md) | 多値分類 | Iris 3クラス、Softmax、CrossEntropyLoss |
| [演習3](./exercise-03_deep-learning.md) | 深層学習 | MNIST手書き数字、多層NN、ミニバッチ学習 |

### 学習の流れ

```
ステップ 7: y=2x+1 を学習（回帰）
    ↓ 「予測値が数値」から「予測値がクラス」へ
演習1: 二値分類（2択: setosa か versicolor か）
    ↓ 「2クラス」から「3クラス以上」へ
演習2: 多値分類（3択: setosa / versicolor / virginica）
    ↓ 「浅いモデル」から「深いモデル」へ
演習3: 深層学習（MNIST 手書き数字認識、隠れ層あり）
```

---

## 次回予告

**第5回：実験の実行と管理**  
長時間の学習ジョブをバックグラウンドで実行し、結果を整理する方法を学びます。
