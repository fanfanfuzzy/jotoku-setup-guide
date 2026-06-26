# 演習3：深層学習 — 手書き数字認識（PyTorch）

MNIST データセット（手書き数字 0〜9）を分類する深層ニューラルネットワークを PyTorch で実装します。

---

## 目標

- [x] 多層ニューラルネットワーク（隠れ層あり）の構造を理解する
- [x] ReLU 活性化関数の役割を理解する
- [x] ミニバッチ学習と DataLoader の使い方を学ぶ
- [x] GPU による学習の高速化を体験する

---

## 背景：なぜ「深層」が必要？

演習1・2では「線形」モデル（直線で分けるだけ）でした。
しかし、複雑なデータ（画像認識など）では直線だけでは分離できません。

```
線形モデル:      入力 → [線形変換] → 出力
深層モデル:      入力 → [線形] → [ReLU] → [線形] → [ReLU] → [線形] → 出力
                         ↑ 隠れ層1          ↑ 隠れ層2          ↑ 出力層
```

**隠れ層を増やす（= 深くする）** ことで、より複雑なパターンを学習できます。

### ReLU（Rectified Linear Unit）

```
ReLU(x) = max(0, x)

  x が正 → そのまま通す
  x が負 → 0にする
```

シンプルだけど強力。深い層でも勾配が消えにくい（学習が進みやすい）。

---

## ステップ 1：スクリプトの作成

`deep_learning_mnist.py` を作成：

```python
# deep_learning_mnist.py - MNISTで深層学習（手書き数字認識）
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import fetch_openml
from sklearn.model_selection import train_test_split
import time

# --- デバイスの設定 ---
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"使用デバイス: {device}")

# === 1. データの準備 ===
print("\nMNISTデータを読み込み中（初回は数分かかります）...")
mnist = fetch_openml('mnist_784', version=1, as_frame=False, parser='auto')
x_data = mnist.data.astype(np.float32) / 255.0  # 0〜1に正規化
y_data = mnist.target.astype(np.int64)

print(f"データ形状: x={x_data.shape}, y={y_data.shape}")
print(f"画像サイズ: 28x28 = 784ピクセル")
print(f"クラス数: 10（数字 0〜9）")

# 学習データと検証データに分割
x_train, x_test, y_train, y_test = train_test_split(
    x_data, y_data, test_size=10000, random_state=42
)
print(f"学習データ: {x_train.shape[0]}個")
print(f"テストデータ: {x_test.shape[0]}個")

# PyTorch DataLoader の作成
train_dataset = TensorDataset(
    torch.FloatTensor(x_train),
    torch.LongTensor(y_train)
)
test_dataset = TensorDataset(
    torch.FloatTensor(x_test),
    torch.LongTensor(y_test)
)

batch_size = 256
train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=True)
test_loader = DataLoader(test_dataset, batch_size=batch_size, shuffle=False)

print(f"ミニバッチサイズ: {batch_size}")
print(f"学習バッチ数: {len(train_loader)}")

# === 2. モデルの定義 ===
class DeepNN(nn.Module):
    """3層の深層ニューラルネットワーク"""
    def __init__(self):
        super().__init__()
        self.network = nn.Sequential(
            # 隠れ層1: 784 → 256
            nn.Linear(784, 256),
            nn.ReLU(),
            
            # 隠れ層2: 256 → 128
            nn.Linear(256, 128),
            nn.ReLU(),
            
            # 出力層: 128 → 10
            nn.Linear(128, 10)
        )
    
    def forward(self, x):
        return self.network(x)

model = DeepNN().to(device)
print(f"\nモデル構造:")
print(model)

# パラメータ数を表示
total_params = sum(p.numel() for p in model.parameters())
print(f"総パラメータ数: {total_params:,}")

# === 3. 損失関数とオプティマイザ ===
loss_fn = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# === 4. 学習ループ ===
print("\n学習開始...")
num_epochs = 20
history = {'train_loss': [], 'train_acc': [], 'test_acc': []}

start_time = time.time()

for epoch in range(num_epochs):
    # --- 学習フェーズ ---
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for batch_x, batch_y in train_loader:
        batch_x = batch_x.to(device)
        batch_y = batch_y.to(device)
        
        # Forward
        outputs = model(batch_x)
        loss = loss_fn(outputs, batch_y)
        
        # Backward
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 統計
        running_loss += loss.item() * batch_x.size(0)
        _, predicted = torch.max(outputs, 1)
        total += batch_y.size(0)
        correct += (predicted == batch_y).sum().item()
    
    train_loss = running_loss / total
    train_acc = correct / total
    
    # --- 評価フェーズ ---
    model.eval()
    test_correct = 0
    test_total = 0
    
    with torch.no_grad():
        for batch_x, batch_y in test_loader:
            batch_x = batch_x.to(device)
            batch_y = batch_y.to(device)
            outputs = model(batch_x)
            _, predicted = torch.max(outputs, 1)
            test_total += batch_y.size(0)
            test_correct += (predicted == batch_y).sum().item()
    
    test_acc = test_correct / test_total
    
    # 記録
    history['train_loss'].append(train_loss)
    history['train_acc'].append(train_acc)
    history['test_acc'].append(test_acc)
    
    print(f"  Epoch {epoch+1:2d}/{num_epochs} | "
          f"Loss: {train_loss:.4f} | "
          f"Train Acc: {train_acc:.4f} | "
          f"Test Acc: {test_acc:.4f}")

elapsed = time.time() - start_time
print(f"\n学習完了！ 所要時間: {elapsed:.1f}秒")
print(f"最終テスト精度: {history['test_acc'][-1]:.4f}")

# === 5. 学習曲線の描画 ===
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))

epochs_range = range(1, num_epochs + 1)

ax1.plot(epochs_range, history['train_loss'], 'b-o', markersize=3)
ax1.set_xlabel('Epoch')
ax1.set_ylabel('Loss')
ax1.set_title('Training Loss')
ax1.grid(True)

ax2.plot(epochs_range, history['train_acc'], 'b-o', markersize=3, label='Train')
ax2.plot(epochs_range, history['test_acc'], 'r-s', markersize=3, label='Test')
ax2.set_xlabel('Epoch')
ax2.set_ylabel('Accuracy')
ax2.set_title('Accuracy')
ax2.legend()
ax2.grid(True)
ax2.set_ylim(0.9, 1.0)

plt.tight_layout()
plt.savefig('deep_learning_curves.png', dpi=100, bbox_inches='tight')
plt.close()
print("\n学習曲線を deep_learning_curves.png に保存しました")

# === 6. 予測結果の可視化 ===
model.eval()
with torch.no_grad():
    sample_x = torch.FloatTensor(x_test[:20]).to(device)
    sample_pred = torch.argmax(model(sample_x), dim=1).cpu().numpy()

fig, axes = plt.subplots(2, 10, figsize=(15, 4))
for i in range(20):
    ax = axes[i // 10, i % 10]
    ax.imshow(x_test[i].reshape(28, 28), cmap='gray_r')
    color = 'green' if sample_pred[i] == y_test[i] else 'red'
    ax.set_title(f'{sample_pred[i]}', color=color, fontsize=12)
    ax.axis('off')

plt.suptitle('Predictions (green=correct, red=wrong)', fontsize=14)
plt.tight_layout()
plt.savefig('deep_learning_predictions.png', dpi=100, bbox_inches='tight')
plt.close()
print("予測結果を deep_learning_predictions.png に保存しました")

# === 7. モデルの保存 ===
torch.save(model.state_dict(), 'mnist_model.pt')
print(f"モデルを mnist_model.pt に保存しました（{total_params:,} パラメータ）")
print("\n完了！")
```

---

## ステップ 2：実行

```bash
python deep_learning_mnist.py
```

**期待される出力：**
```
使用デバイス: cuda

MNISTデータを読み込み中（初回は数分かかります）...
データ形状: x=(70000, 784), y=(70000,)
画像サイズ: 28x28 = 784ピクセル
クラス数: 10（数字 0〜9）
学習データ: 60000個
テストデータ: 10000個
ミニバッチサイズ: 256
学習バッチ数: 235

モデル構造:
DeepNN(
  (network): Sequential(
    (0): Linear(in_features=784, out_features=256, bias=True)
    (1): ReLU()
    (2): Linear(in_features=256, out_features=128, bias=True)
    (3): ReLU()
    (4): Linear(in_features=128, out_features=10, bias=True)
  )
)
総パラメータ数: 235,146

学習開始...
  Epoch  1/20 | Loss: 0.3215 | Train Acc: 0.9082 | Test Acc: 0.9534
  Epoch  2/20 | Loss: 0.1342 | Train Acc: 0.9607 | Test Acc: 0.9638
  ...
  Epoch 20/20 | Loss: 0.0123 | Train Acc: 0.9971 | Test Acc: 0.9782

学習完了！ 所要時間: 25.3秒
最終テスト精度: 0.9782
```

---

## コードの解説

### モデルの構造

```
入力 (784ピクセル)
    ↓ Linear(784, 256) + ReLU    ← 隠れ層1
256ノード
    ↓ Linear(256, 128) + ReLU    ← 隠れ層2
128ノード
    ↓ Linear(128, 10)            ← 出力層
10クラス（数字 0〜9）
```

### 演習1・2との違い

| 項目 | 演習1・2 | 演習3 |
|------|----------|-------|
| データ | Iris (150個, 2〜4特徴量) | MNIST (70000個, 784特徴量) |
| モデル | 線形（隠れ層なし） | 深層（隠れ層2つ） |
| 活性化関数 | Sigmoid/なし | ReLU |
| 学習方法 | 全データ一括 | ミニバッチ（256個ずつ） |
| オプティマイザ | SGD | Adam |
| 精度 | 85〜100% | 97〜98% |

### DataLoader とは？

```python
train_loader = DataLoader(dataset, batch_size=256, shuffle=True)

for batch_x, batch_y in train_loader:
    # batch_x: 256個のデータ
    # batch_y: 256個のラベル
    ...
```

- データを `batch_size` 個ずつ小分けにして渡してくれる
- `shuffle=True`: エポックごとにデータの順番をシャッフル
- 大きなデータセットでも効率的に学習できる

### Adam オプティマイザ

```
SGD:  パラメータ -= 学習率 × 勾配        ← 一定の速度で進む
Adam: パラメータ -= 適応的学習率 × 勾配   ← データに応じて速度を調整
```

Adam は SGD より収束が速く、学習率の調整が楽。実用上のデフォルト選択肢。

---

## GPU と CPU の速度比較

同じ学習を CPU で実行すると所要時間が大きく異なります：

| デバイス | 所要時間（目安） |
|----------|-----------------|
| GPU (DGX-Spark) | 20〜30秒 |
| CPU | 2〜5分 |

GPU の並列計算能力が、大量の行列演算で威力を発揮します。

---

## 確認チェックリスト

- [ ] スクリプトがエラーなく実行できた
- [ ] テスト精度が 0.97 以上になった
- [ ] 学習曲線のグラフ（`deep_learning_curves.png`）が生成された
- [ ] 予測結果のグラフ（`deep_learning_predictions.png`）が生成された
- [ ] モデルファイル（`mnist_model.pt`）が保存された
- [ ] GPU で実行されていることを確認した（「使用デバイス: cuda」）

---

## Google Colab で実行する場合

自宅など DGX-Spark に接続できない環境では、Google Colab で同じコードを実行できます。

1. [Google Colab](https://colab.research.google.com/) を開く
2. 「新しいノートブック」を作成
3. ランタイム → 「ランタイムのタイプを変更」→ **GPU** を選択
4. セルに以下を入力して実行：

```python
!pip install scikit-learn matplotlib
```

5. 次のセルに上記のスクリプト全体をコピー＆ペーストして実行

> 💡 MNIST データのダウンロードに数分かかる場合があります。Colab のGPU無料枠でも十分高速に学習できます（20エポックで1〜2分程度）。

---

## チャレンジ課題（余裕があれば）

1. 隠れ層を1つ（256ノードのみ）にすると精度はどう変わるか
2. 隠れ層を3つに増やすとどうなるか
3. `Dropout` を追加してみよう（過学習防止）：
   ```python
   self.network = nn.Sequential(
       nn.Linear(784, 256),
       nn.ReLU(),
       nn.Dropout(0.2),      # 20%のノードをランダムに無効化
       nn.Linear(256, 128),
       nn.ReLU(),
       nn.Dropout(0.2),
       nn.Linear(128, 10)
   )
   ```
4. 保存したモデルを読み込んで、好きな画像を認識させてみよう：
   ```python
   model = DeepNN().to(device)
   model.load_state_dict(torch.load('mnist_model.pt'))
   model.eval()
   ```
