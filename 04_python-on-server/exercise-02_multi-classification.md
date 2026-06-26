# 演習2：多値分類（PyTorch）

Iris データセットの3種類の花すべてを分類するモデルを PyTorch で実装します。

---

## 目標

- [x] 多値分類（3クラス以上）の仕組みを理解する
- [x] Softmax 関数と CrossEntropyLoss の役割を理解する
- [x] PyTorch の `nn.Module` を使ったモデル定義を学ぶ
- [x] 学習曲線（Loss / Accuracy）を描画する

---

## 背景：多値分類とは？

演習1では「2つのうちどちら？」でしたが、今回は「3つ以上のうちどれ？」を予測します。

```
二値分類:  setosa か versicolor か → シグモイド関数（0〜1の確率）
多値分類:  setosa か versicolor か virginica か → ソフトマックス関数（合計1の確率分布）
```

### Softmax 関数のイメージ

```
入力: [2.0, 1.0, 0.1]
    ↓ softmax
出力: [0.66, 0.24, 0.10]  ← 合計すると 1.0
    → 「クラス0の確率66%、クラス1の確率24%、クラス2の確率10%」
```

---

## ステップ 1：スクリプトの作成

`multi_classification.py` を作成：

```python
# multi_classification.py - Irisデータセットで多値分類（3クラス）
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

# --- デバイスの設定 ---
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"使用デバイス: {device}")

# === 1. データの準備 ===
iris = load_iris()
x_data = iris.data[:, [0, 2]]  # sepal_length と petal_length の2特徴量
y_data = iris.target            # 0, 1, 2 の3クラス

print(f"データ形状: x={x_data.shape}, y={y_data.shape}")
print(f"クラス0 (setosa): {(y_data == 0).sum()}個")
print(f"クラス1 (versicolor): {(y_data == 1).sum()}個")
print(f"クラス2 (virginica): {(y_data == 2).sum()}個")

# 学習データと検証データに分割
x_train, x_test, y_train, y_test = train_test_split(
    x_data, y_data, test_size=0.3, random_state=42
)

# PyTorch テンソルに変換
x_train_t = torch.FloatTensor(x_train).to(device)
y_train_t = torch.LongTensor(y_train).to(device)   # 多値分類はLongTensor
x_test_t = torch.FloatTensor(x_test).to(device)
y_test_t = torch.LongTensor(y_test).to(device)

# === 2. モデルの定義（nn.Module を使う方法） ===
class IrisClassifier(nn.Module):
    def __init__(self, input_dim, num_classes):
        super().__init__()
        self.linear = nn.Linear(input_dim, num_classes)
    
    def forward(self, x):
        return self.linear(x)  # Softmax は CrossEntropyLoss に含まれる

model = IrisClassifier(input_dim=2, num_classes=3).to(device)
print(f"\nモデル構造:")
print(model)
print(f"パラメータ数: {sum(p.numel() for p in model.parameters())}")

# === 3. 損失関数とオプティマイザ ===
loss_fn = nn.CrossEntropyLoss()  # 多値分類用（softmax + NLLLoss を内包）
optimizer = optim.SGD(model.parameters(), lr=0.1)

# === 4. 学習ループ ===
print("\n学習開始...")
train_losses = []
train_accuracies = []
test_accuracies = []

for epoch in range(2000):
    model.train()
    
    # Forward
    logits = model(x_train_t)
    loss = loss_fn(logits, y_train_t)
    
    # Backward
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # 記録（200エポックごと）
    if (epoch + 1) % 200 == 0:
        model.eval()
        with torch.no_grad():
            # 学習データの精度
            train_pred = torch.argmax(model(x_train_t), dim=1)
            train_acc = (train_pred == y_train_t).float().mean().item()
            
            # テストデータの精度
            test_pred = torch.argmax(model(x_test_t), dim=1)
            test_acc = (test_pred == y_test_t).float().mean().item()
        
        train_losses.append(loss.item())
        train_accuracies.append(train_acc)
        test_accuracies.append(test_acc)
        
        print(f"  Epoch {epoch+1:4d} | Loss: {loss.item():.4f} | "
              f"Train Acc: {train_acc:.4f} | Test Acc: {test_acc:.4f}")

# === 5. 最終評価 ===
model.eval()
with torch.no_grad():
    test_logits = model(x_test_t)
    test_probs = torch.softmax(test_logits, dim=1)
    test_pred = torch.argmax(test_logits, dim=1)
    test_acc = (test_pred == y_test_t).float().mean()

print(f"\n最終テスト精度: {test_acc.item():.4f}")

# 予測確率の例を表示
print("\n--- 予測例（最初の5個） ---")
print(f"{'実際':>4} | {'予測':>4} | {'確率 [setosa, versicolor, virginica]'}")
print("-" * 60)
for i in range(5):
    probs = test_probs[i].cpu().numpy()
    print(f"  {y_test[i]:>2}  |   {test_pred[i].item():>2}  | "
          f"[{probs[0]:.3f}, {probs[1]:.3f}, {probs[2]:.3f}]")

# === 6. 学習曲線の描画 ===
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))

epochs_plot = range(200, 2001, 200)

# 損失関数
ax1.plot(epochs_plot, train_losses, 'b-o', markersize=4)
ax1.set_xlabel('Epoch')
ax1.set_ylabel('Loss')
ax1.set_title('Training Loss')
ax1.grid(True)

# 精度
ax2.plot(epochs_plot, train_accuracies, 'b-o', markersize=4, label='Train')
ax2.plot(epochs_plot, test_accuracies, 'r-s', markersize=4, label='Test')
ax2.set_xlabel('Epoch')
ax2.set_ylabel('Accuracy')
ax2.set_title('Accuracy')
ax2.legend()
ax2.grid(True)
ax2.set_ylim(0, 1.05)

plt.tight_layout()
plt.savefig('multi_classification_curves.png', dpi=100, bbox_inches='tight')
plt.close()
print("\n学習曲線を multi_classification_curves.png に保存しました")

# === 7. 決定境界の可視化 ===
x_min, x_max = x_data[:, 0].min() - 0.5, x_data[:, 0].max() + 0.5
y_min, y_max = x_data[:, 1].min() - 0.5, x_data[:, 1].max() + 0.5
xx, yy = np.meshgrid(
    np.linspace(x_min, x_max, 200),
    np.linspace(y_min, y_max, 200)
)
grid = torch.FloatTensor(np.c_[xx.ravel(), yy.ravel()]).to(device)

model.eval()
with torch.no_grad():
    grid_pred = torch.argmax(model(grid), dim=1).cpu().numpy().reshape(xx.shape)

plt.figure(figsize=(8, 6))
plt.contourf(xx, yy, grid_pred, alpha=0.3, cmap='viridis')

colors = ['blue', 'green', 'red']
markers = ['x', 'o', '+']
names = ['setosa', 'versicolor', 'virginica']
for i in range(3):
    mask = y_data == i
    plt.scatter(x_data[mask, 0], x_data[mask, 1],
                c=colors[i], marker=markers[i], s=50, label=names[i])

plt.xlabel('sepal length (cm)')
plt.ylabel('petal length (cm)')
plt.title('Multi-class Classification - Decision Boundary')
plt.legend()
plt.savefig('multi_classification_boundary.png', dpi=100, bbox_inches='tight')
plt.close()
print("決定境界を multi_classification_boundary.png に保存しました")
print("\n完了！")
```

---

## ステップ 2：実行

```bash
python multi_classification.py
```

**期待される出力：**
```
使用デバイス: cuda
データ形状: x=(150, 2), y=(150,)
クラス0 (setosa): 50個
クラス1 (versicolor): 50個
クラス2 (virginica): 50個

モデル構造:
IrisClassifier(
  (linear): Linear(in_features=2, out_features=3, bias=True)
)
パラメータ数: 9

学習開始...
  Epoch  200 | Loss: 0.5432 | Train Acc: 0.8095 | Test Acc: 0.8000
  Epoch  400 | Loss: 0.4215 | Train Acc: 0.8286 | Test Acc: 0.8222
  ...
  Epoch 2000 | Loss: 0.2834 | Train Acc: 0.8571 | Test Acc: 0.8444

最終テスト精度: 0.8444

--- 予測例（最初の5個） ---
実際 | 予測 | 確率 [setosa, versicolor, virginica]
------------------------------------------------------------
   1  |    1  | [0.012, 0.723, 0.265]
   0  |    0  | [0.981, 0.018, 0.001]
   ...
```

---

## コードの解説

### 演習1との違い

| 項目 | 演習1（二値分類） | 演習2（多値分類） |
|------|-------------------|-------------------|
| クラス数 | 2 | 3 |
| 出力層 | 1ノード + Sigmoid | 3ノード + Softmax |
| 損失関数 | `BCELoss` | `CrossEntropyLoss` |
| ラベルの型 | `FloatTensor` | `LongTensor`（整数） |
| 予測方法 | `> 0.5` で判定 | `argmax` で最大確率のクラス |

### `nn.Module` を使ったモデル定義

```python
class IrisClassifier(nn.Module):
    def __init__(self, input_dim, num_classes):
        super().__init__()
        self.linear = nn.Linear(input_dim, num_classes)
    
    def forward(self, x):
        return self.linear(x)
```

- `__init__`: 層（パラメータ）を定義
- `forward`: データの流れを定義
- `nn.Sequential` より柔軟にモデルを構築できる

### CrossEntropyLoss の中身

```
CrossEntropyLoss = Softmax + NegativeLogLikelihood

入力(logits): [2.0, 1.0, 0.1]
    ↓ Softmax
確率: [0.66, 0.24, 0.10]
    ↓ -log(正解クラスの確率)
損失値: -log(0.66) = 0.42  （正解がクラス0の場合）
```

> PyTorch の `CrossEntropyLoss` は内部で Softmax を計算するため、モデルの出力に Softmax を付ける必要はありません。

---

## 確認チェックリスト

- [ ] スクリプトがエラーなく実行できた
- [ ] テスト精度が 0.8 以上になった
- [ ] 学習曲線のグラフ（`multi_classification_curves.png`）が生成された
- [ ] 決定境界のグラフ（`multi_classification_boundary.png`）が生成された
- [ ] `CrossEntropyLoss` と `BCELoss` の使い分けが理解できた

---

## チャレンジ課題（余裕があれば）

1. 4つの特徴量すべてを使うとテスト精度はどう変わるか
2. `model.train()` と `model.eval()` の違いを調べてみよう
3. 混同行列（Confusion Matrix）を表示してみよう：
   ```python
   from sklearn.metrics import confusion_matrix, classification_report
   print(confusion_matrix(y_test, test_pred.cpu().numpy()))
   print(classification_report(y_test, test_pred.cpu().numpy(), 
                               target_names=iris.target_names))
   ```
