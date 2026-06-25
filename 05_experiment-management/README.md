# 第5回：実験の実行と管理

この回では、長時間かかる学習ジョブをバックグラウンドで実行し、結果を整理・記録する方法を学びます。

---

## 目標

- [x] `tmux` でターミナルセッションを管理する
- [x] 長時間実行のジョブを安全に動かす
- [x] 実行中のプロセスを確認・停止する
- [x] 実験結果を整理して Git で管理する

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

## 確認チェックリスト

- [ ] `tmux new -s test` でセッションを作成できた
- [ ] `Ctrl+B → D` でセッションから離脱できた
- [ ] `tmux attach -t test` でセッションに戻れた
- [ ] `ps aux | grep python` でプロセスを確認できた
- [ ] `nvidia-smi` で GPU 使用状況を確認できた
- [ ] `.gitignore` を作成して不要ファイルを除外できた
- [ ] 実験結果をディレクトリに整理して保存できた

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
