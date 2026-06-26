# 第5回：実験の実行と管理

この回では、長時間かかる学習ジョブをバックグラウンドで実行し、結果を整理・記録する方法を学びます。

---

## 目標

- [x] `tmux` でターミナルセッションを管理する
- [x] 長時間実行のジョブを安全に動かす
- [x] 実行中のプロセスを確認・停止する
- [x] 実験結果を整理して Git で管理する
- [x] Weights & Biases (W&B) で実験を記録・可視化する

---

## ステップ 1：問題を理解する

### なぜバックグラウンド実行が必要？

SSH でサーバーに接続してプログラムを実行中に...

```
学習を開始（2時間かかる）
    ↓
WiFiが切れた / PCを閉じた / SSH接続が切断
    ↓
プログラムが強制終了される ← これを防ぎたい！
```

**解決策**: `tmux` を使ってサーバー上にセッションを作る。SSH が切れてもプログラムは動き続ける。

---

## ステップ 2：tmux の基本

### 2.1 tmux とは

```
┌─── SSH接続 ────────────────────────────┐
│                                         │
│  ┌─── tmux セッション ──────────────┐   │
│  │                                   │   │
│  │  プログラム実行中...              │   │  ← SSH が切れても
│  │                                   │   │    ここは生き続ける
│  └───────────────────────────────────┘   │
│                                         │
└─────────────────────────────────────────┘
```

### 2.2 基本操作

```bash
# 新しいセッションを作成
tmux new -s 実験名

# セッションから離脱（プログラムは動き続ける）
# Ctrl+B を押してから D を押す

# セッション一覧を表示
tmux ls

# セッションに戻る
tmux attach -t 実験名

# セッションを終了
tmux kill-session -t 実験名
```

### 2.3 実践

```bash
# 1. セッションを作成
tmux new -s training

# 2. 何かプログラムを実行（例：10秒カウント）
for i in $(seq 1 10); do echo "Count: $i"; sleep 1; done

# 3. Ctrl+B → D で離脱

# 4. 別の作業ができる（プログラムはバックグラウンドで動いている）
tmux ls  # セッションが残っていることを確認

# 5. セッションに戻る
tmux attach -t training
```

### 2.4 tmux の便利なキー操作

`Ctrl+B` を押してから：

| キー | 操作 |
|------|------|
| `D` | セッションから離脱 |
| `C` | 新しいウィンドウを作成 |
| `N` | 次のウィンドウに移動 |
| `P` | 前のウィンドウに移動 |
| `%` | 画面を左右に分割 |
| `"` | 画面を上下に分割 |
| `矢印キー` | 分割画面を移動 |

---

## ステップ 3：プロセスの確認と停止

### 3.1 実行中のプロセスを確認

```bash
# 全プロセスを表示
ps aux | grep python

# GPU を使っているプロセスを確認
nvidia-smi
```

### 3.2 プロセスの停止

```bash
# プロセスIDで停止
kill プロセスID

# 強制停止（上で止まらない場合）
kill -9 プロセスID
```

### 3.3 リアルタイム監視

```bash
# CPU/メモリの監視
htop

# GPUの継続監視（2秒ごとに更新）
watch -n 2 nvidia-smi
```

> `htop` を終了するには `Q` キー、`watch` を止めるには `Ctrl+C`

---

## ステップ 4：実験の整理

### 4.1 ディレクトリ構成の例

```
my-experiment/
├── .venv/              ← 仮想環境
├── src/                ← ソースコード
│   ├── train.py
│   ├── model.py
│   └── utils.py
├── data/               ← データ（Git管理しない）
│   └── dataset.csv
├── results/            ← 実験結果
│   ├── 2024-01-15_exp01/
│   │   ├── model.pth
│   │   ├── log.txt
│   │   └── metrics.json
│   └── 2024-01-16_exp02/
│       ├── model.pth
│       ├── log.txt
│       └── metrics.json
├── requirements.txt    ← ライブラリ一覧
├── .gitignore          ← Git除外設定
└── README.md           ← プロジェクト説明
```

### 4.2 .gitignore の作成

Git で管理しないファイルを指定：

```bash
# .gitignore
.venv/
data/
results/*.pth
__pycache__/
*.pyc
.DS_Store
```

> 💡 **ポイント**: 大きなファイル（データ、モデルの重み）は Git に入れない。コードと設定ファイルだけを管理する。

### 4.3 実験ログを残す

`train.py` の中でログを保存するようにする：

```python
import json
import os
from datetime import datetime

# 実験ディレクトリの作成
exp_name = datetime.now().strftime("%Y-%m-%d_%H%M%S")
exp_dir = f"results/{exp_name}"
os.makedirs(exp_dir, exist_ok=True)

# ... 学習処理 ...

# メトリクスの保存
metrics = {
    "epochs": 100,
    "final_loss": 0.025,
    "accuracy": 0.953,
    "learning_rate": 0.001,
}
with open(f"{exp_dir}/metrics.json", "w") as f:
    json.dump(metrics, f, indent=2)

print(f"結果を保存しました: {exp_dir}")
```

---

## ステップ 5：出力のリダイレクト

### 5.1 ログファイルに保存

```bash
# 標準出力をファイルに保存
python train.py > results/log.txt 2>&1

# 画面にも表示しながらファイルにも保存
python train.py 2>&1 | tee results/log.txt
```

### 5.2 tmux + ログ保存の組み合わせ

```bash
# セッション作成 → 学習開始 → 離脱
tmux new -s exp01
python train.py 2>&1 | tee results/exp01_log.txt
# Ctrl+B → D で離脱

# あとでログを確認
tail -f results/exp01_log.txt   # リアルタイムでログの末尾を表示
```

---

## ステップ 6：実験を Git で管理する

### 6.1 コードの変更をコミット

```bash
git add src/ requirements.txt .gitignore
git commit -m "実験01: 学習率0.001, epoch100"
```

### 6.2 実験ノートとしてのコミットメッセージ

良いコミットメッセージの例：
```
exp: 学習率を0.01→0.001に変更、accuracy 93%→95%に改善
exp: バッチサイズ64に変更、メモリ不足を解消
fix: データ前処理のバグ修正（正規化の順序）
```

### 6.3 結果のサマリーを README に記録

```markdown
## 実験記録

| 日付 | 設定 | 結果 |
|------|------|------|
| 01/15 | lr=0.01, epoch=50 | acc=93.2% |
| 01/16 | lr=0.001, epoch=100 | acc=95.3% ← best |
| 01/17 | lr=0.001, epoch=200 | acc=95.1%（過学習気味）|
```

---

## ステップ 7：Weights & Biases (W&B) で実験を記録する

### 7.1 W&B とは

**Weights & Biases (W&B)** は、機械学習の実験を自動的に記録・可視化するプラットフォームです。

```
従来の方法:
  print(f"epoch={e}, loss={loss}")  → ターミナルのログに埋もれる
  json.dump(metrics, f)            → ファイルを手動で比較

W&B を使うと:
  wandb.log({"loss": loss})        → 自動でグラフ化、ブラウザで確認
```

### 7.2 なぜ W&B を使うのか

| 目的 | 手動管理 | W&B |
|------|----------|-----|
| 学習曲線の確認 | matplotlib で毎回描画 | リアルタイムでブラウザに表示 |
| ハイパーパラメータ比較 | 表を手動作成 | 自動で比較テーブル生成 |
| 実験の再現 | 「あの設定なんだっけ…」 | 全設定が自動保存 |
| チーム共有 | ファイルを送り合う | URLを共有するだけ |
| GPU使用率の監視 | `nvidia-smi` を別ターミナルで実行 | 自動でシステムメトリクス記録 |

> 💡 **ポイント**: W&B は「実験ノート」をクラウドに自動保存してくれるツールです。無料プランで個人利用には十分です。

### 7.3 セットアップ

#### インストール

```bash
# venv が有効な状態で
pip install wandb
```

#### アカウント作成とログイン

1. https://wandb.ai/site にアクセスしてアカウント作成（GitHub連携が楽）
2. ターミナルでログイン：

```bash
wandb login
```

API キーの入力を求められるので、https://wandb.ai/authorize からコピーして貼り付けます。

> ⚠️ **注意**: API キーは他人に見せないこと。`.netrc` に自動保存されるので、毎回入力する必要はありません。

### 7.4 PyTorch の学習ループに組み込む

第4回の演習3（MNIST深層学習）に W&B を追加した例：

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
from sklearn.datasets import fetch_openml
from sklearn.model_selection import train_test_split
import numpy as np
import wandb

# W&B の初期化
wandb.init(
    project="jotoku-mnist",        # プロジェクト名
    name="exp01-baseline",         # この実験の名前
    config={                       # ハイパーパラメータを記録
        "learning_rate": 0.001,
        "epochs": 20,
        "batch_size": 256,
        "hidden_sizes": [256, 128],
        "optimizer": "Adam",
    }
)
config = wandb.config  # 設定を変数として使える

# デバイス設定
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Device: {device}")

# データ準備（省略：演習3と同じ）
# ...

# モデル定義
class DeepNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(784, config.hidden_sizes[0]),
            nn.ReLU(),
            nn.Linear(config.hidden_sizes[0], config.hidden_sizes[1]),
            nn.ReLU(),
            nn.Linear(config.hidden_sizes[1], 10)
        )

    def forward(self, x):
        return self.network(x)

model = DeepNN().to(device)
optimizer = optim.Adam(model.parameters(), lr=config.learning_rate)
loss_fn = nn.CrossEntropyLoss()

# W&B でモデル構造を監視
wandb.watch(model, log="all", log_freq=100)

# 学習ループ
for epoch in range(config.epochs):
    model.train()
    epoch_loss = 0
    correct = 0
    total = 0

    for batch_x, batch_y in train_loader:
        batch_x, batch_y = batch_x.to(device), batch_y.to(device)

        outputs = model(batch_x)
        loss = loss_fn(outputs, batch_y)

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        epoch_loss += loss.item()
        _, predicted = torch.max(outputs, 1)
        correct += (predicted == batch_y).sum().item()
        total += batch_y.size(0)

    train_acc = correct / total

    # W&B にログ送信（これだけで自動グラフ化）
    wandb.log({
        "epoch": epoch,
        "train_loss": epoch_loss / len(train_loader),
        "train_accuracy": train_acc,
    })

    print(f"Epoch {epoch+1}/{config.epochs} - Loss: {epoch_loss/len(train_loader):.4f}, Acc: {train_acc:.4f}")

# テスト精度を記録
model.eval()
# ... テスト評価 ...
wandb.log({"test_accuracy": test_acc})

# 実験を終了
wandb.finish()
```

### 7.5 ダッシュボードの見方

`wandb.init()` を実行すると、ターミナルに URL が表示されます：

```
wandb: Syncing run exp01-baseline
wandb: ⭐️ View project at https://wandb.ai/あなたのユーザー名/jotoku-mnist
wandb: 🚀 View run at https://wandb.ai/あなたのユーザー名/jotoku-mnist/runs/xxxxx
```

ブラウザで開くと：

| タブ | 内容 |
|------|------|
| **Charts** | loss、accuracy などのグラフ（リアルタイム更新） |
| **System** | GPU使用率、メモリ使用量、CPU使用率 |
| **Logs** | print文の出力 |
| **Files** | 保存したファイル（モデル等） |
| **Config** | ハイパーパラメータ一覧 |

### 7.6 実験を比較する

複数の実験を実行した後、W&B のプロジェクトページで：

1. 実験を複数選択
2. 「Compare」ボタンをクリック
3. 設定の違いと結果の違いが一目で分かる

例：
```
| Run          | lr    | hidden  | test_acc |
|--------------|-------|---------|----------|
| exp01-base   | 0.001 | [256,128] | 97.1%  |
| exp02-small  | 0.001 | [128,64]  | 95.8%  |
| exp03-fast   | 0.01  | [256,128] | 96.5%  |
```

### 7.7 Colab で W&B を使う場合

```python
!pip install wandb
import wandb
wandb.login()  # API キーを入力
```

> 💡 Colab でも DGX-Spark でも同じ `wandb.init(project="...")` を使えば、同じプロジェクトに実験が蓄積されます。

---

## 確認チェックリスト

- [ ] `tmux new -s test` でセッションを作成できた
- [ ] `Ctrl+B → D` でセッションから離脱できた
- [ ] `tmux attach -t test` でセッションに戻れた
- [ ] `ps aux | grep python` でプロセスを確認できた
- [ ] `nvidia-smi` で GPU 使用状況を確認できた
- [ ] `.gitignore` を作成して不要ファイルを除外できた
- [ ] 実験結果をディレクトリに整理して保存できた
- [ ] W&B にログインできた（`wandb login`）
- [ ] 学習ループに `wandb.log()` を追加して実験を記録できた
- [ ] W&B ダッシュボードでグラフを確認できた

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `sessions should be nested with care` | tmux の中で tmux を開こうとした | 一度 `exit` してから新しいセッションを作る |
| `no sessions` | tmux セッションがない | `tmux new -s 名前` で新規作成 |
| プログラムが止められない | `Ctrl+C` が効かない | 別ターミナルから `kill` コマンド |
| ディスク容量不足 | 結果ファイルが大きすぎる | `df -h` で確認、不要ファイルを削除 |

---

## 次回予告

**第6回：Docker入門**  
環境を丸ごとパッケージ化する Docker を学び、再現可能な実験環境を作ります。
