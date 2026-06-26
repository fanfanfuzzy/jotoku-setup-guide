# 演習1：二値分類（PyTorch）

Iris データセットを使って、2種類の花を分類するモデルを PyTorch で実装します。

---

## 目標

- [x] データの読み込みと前処理を理解する
- [x] PyTorch でロジスティック回帰（二値分類）モデルを構築する
- [x] 学習ループ（forward → loss → backward → update）を理解する
- [x] 決定境界を可視化する

---

## 背景：二値分類とは？

「2つのグループのどちらに属するか」を予測する問題です。

例：
- メールが「スパム」か「正常」か
- 画像が「猫」か「犬」か
- 花の種類が「setosa」か「versicolor」か ← 今回はこれ

---

## ステップ 1：スクリプトの作成

`binary_classification.py` を作成：

```python
# binary_classification.py - Irisデータセットで二値分類
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
# クラス0 (setosa) と クラス1 (versicolor) のみ使用
# 特徴量は sepal_length と sepal_width の2つだけ
x_data = iris.data[:100, :2]  # 最初の100個、2特徴量
y_data = iris.target[:100]     # 0 or 1

print(f"データ形状: x={x_data.shape}, y={y_data.shape}")
print(f"クラス0 (setosa): {(y_data == 0).sum()}個")
print(f"クラス1 (versicolor): {(y_data == 1).sum()}個")

# 学習データと検証データに分割
x_train, x_test, y_train, y_test = train_test_split(
    x_data, y_data, test_size=0.3, random_state=42
)

# NumPy配列 → PyTorch テンソルに変換
x_train_t = torch.FloatTensor(x_train).to(device)
y_train_t = torch.FloatTensor(y_train).to(device)
x_test_t = torch.FloatTensor(x_test).to(device)
y_test_t = torch.FloatTensor(y_test).to(device)

# === 2. モデルの定義 ===
# ロジスティック回帰 = 線形変換 + シグモイド関数
model = nn.Sequential(
    nn.Linear(2, 1),  # 入力2次元 → 出力1次元
    nn.Sigmoid()       # 0〜1の確率に変換
).to(device)

print(f"\nモデル構造:")
print(model)

# === 3. 損失関数とオプティマイザ ===
loss_fn = nn.BCELoss()  # Binary Cross Entropy（二値分類用）
optimizer = optim.SGD(model.parameters(), lr=0.1)

# === 4. 学習ループ ===
print("\n学習開始...")
history = []

for epoch in range(1000):
    # Forward: 予測値を計算
    y_pred = model(x_train_t).squeeze()
    
    # Loss: 損失を計算
    loss = loss_fn(y_pred, y_train_t)
    
    # Backward: 勾配を計算
    optimizer.zero_grad()
    loss.backward()
    
    # Update: パラメータを更新
    optimizer.step()
    
    # 記録
    if (epoch + 1) % 100 == 0:
        # 精度の計算
        with torch.no_grad():
            pred_class = (y_pred > 0.5).float()
            accuracy = (pred_class == y_train_t).float().mean()
        history.append((epoch + 1, loss.item(), accuracy.item()))
        print(f"  Epoch {epoch+1:4d} | Loss: {loss.item():.4f} | Accuracy: {accuracy.item():.4f}")

# === 5. テストデータで評価 ===
model.eval()
with torch.no_grad():
    y_test_pred = model(x_test_t).squeeze()
    test_class = (y_test_pred > 0.5).float()
    test_accuracy = (test_class == y_test_t).float().mean()
    print(f"\nテスト精度: {test_accuracy.item():.4f}")

# === 6. 決定境界の可視化 ===
print("\n決定境界を描画中...")

# メッシュグリッドを作成
x_min, x_max = x_data[:, 0].min() - 0.5, x_data[:, 0].max() + 0.5
y_min, y_max = x_data[:, 1].min() - 0.5, x_data[:, 1].max() + 0.5
xx, yy = np.meshgrid(
    np.linspace(x_min, x_max, 200),
    np.linspace(y_min, y_max, 200)
)
grid = torch.FloatTensor(np.c_[xx.ravel(), yy.ravel()]).to(device)

model.eval()
with torch.no_grad():
    zz = model(grid).cpu().numpy().reshape(xx.shape)

# プロット
plt.figure(figsize=(8, 6))
plt.contourf(xx, yy, zz, levels=[0, 0.5, 1], alpha=0.3, colors=['blue', 'red'])
plt.contour(xx, yy, zz, levels=[0.5], colors='black', linewidths=2)

# データ点をプロット
class0 = x_data[y_data == 0]
class1 = x_data[y_data == 1]
plt.scatter(class0[:, 0], class0[:, 1], c='blue', marker='x', s=50, label='setosa')
plt.scatter(class1[:, 0], class1[:, 1], c='red', marker='o', s=50, label='versicolor')

plt.xlabel('sepal length (cm)')
plt.ylabel('sepal width (cm)')
plt.title('Binary Classification - Decision Boundary')
plt.legend()
plt.savefig('binary_classification_result.png', dpi=100, bbox_inches='tight')
plt.close()
print("結果を binary_classification_result.png に保存しました")
print("\n完了！")
```

---

## ステップ 2：実行

```bash
# 仮想環境を有効化した状態で
pip install scikit-learn matplotlib
python binary_classification.py
```

**期待される出力：**
```
使用デバイス: cuda
データ形状: x=(100, 2), y=(100,)
クラス0 (setosa): 50個
クラス1 (versicolor): 50個

モデル構造:
Sequential(
  (0): Linear(in_features=2, out_features=1, bias=True)
  (1): Sigmoid()
)

学習開始...
  Epoch  100 | Loss: 0.3842 | Accuracy: 0.8857
  Epoch  200 | Loss: 0.2951 | Accuracy: 0.9429
  ...
  Epoch 1000 | Loss: 0.1823 | Accuracy: 1.0000

テスト精度: 1.0000

決定境界を描画中...
結果を binary_classification_result.png に保存しました

完了！
```

---

## コードの解説

### データの流れ

```
Iris データ (NumPy)
    ↓ torch.FloatTensor()
PyTorch テンソル
    ↓ model(x)
予測値 (0〜1の確率)
    ↓ BCELoss
損失値
    ↓ .backward()
勾配計算
    ↓ optimizer.step()
パラメータ更新
```

### 重要な概念

| 概念 | 説明 |
|------|------|
| `nn.Linear(2, 1)` | 2入力→1出力の線形変換（$y = wx + b$） |
| `nn.Sigmoid()` | 出力を0〜1に変換する関数 |
| `BCELoss` | 二値分類用の損失関数（Binary Cross Entropy） |
| `optimizer.zero_grad()` | 勾配をリセット（毎回必要） |
| `loss.backward()` | 逆伝播で勾配を計算 |
| `optimizer.step()` | 勾配に基づいてパラメータを更新 |

### シグモイド関数とは？

```
入力 → Linear → シグモイド → 出力(0〜1)

  大きい正の値 → 1に近い（クラス1の確率が高い）
  0に近い値   → 0.5（どちらとも言えない）
  大きい負の値 → 0に近い（クラス0の確率が高い）
```

---

## 確認チェックリスト

- [ ] スクリプトがエラーなく実行できた
- [ ] テスト精度が 0.9 以上になった
- [ ] `binary_classification_result.png` が生成された
- [ ] 決定境界が2つのクラスを分離していることを確認した

---

## チャレンジ課題（余裕があれば）

1. 学習率 `lr` を変えると結果がどう変わるか試してみよう（0.01, 0.1, 1.0）
2. 特徴量を4つ全て使うとどうなるか（`iris.data[:100, :4]` にして `nn.Linear(4, 1)`）
3. `nn.SGD` を `nn.Adam` に変えると学習はどう変わるか
