# 第1回：PCの準備とColab体験

この回では、研究で使う基本的なツールをPCに準備し、Google Colabで初めてのPythonプログラムを動かします。

---

## 目標

- [x] VS Code をインストールする
- [x] Google Colab で Python を動かす
- [x] GitHub アカウントを作成する

---

## ステップ 1：VS Code のインストール

### 1.1 ダウンロード

1. ブラウザで以下のURLを開く：  
   👉 https://code.visualstudio.com/
2. 自分のOS（Windows / Mac）に合ったボタンをクリックしてダウンロード
3. ダウンロードしたファイルを実行してインストール

### 1.2 インストール確認

1. VS Code を起動する
2. 画面が表示されれば OK

### 1.3 日本語化（任意）

1. 左側のアイコンから四角が4つ並んだアイコン（拡張機能）をクリック
2. 検索欄に `Japanese Language Pack` と入力
3. 「Install」をクリック
4. VS Code を再起動

> 💡 **ポイント**: VS Code はこれからずっと使うメインのエディタです。まずは起動できることだけ確認すればOKです。

---

## ステップ 2：Google Colab で Python を動かす

### 2.1 Colab を開く

1. ブラウザで以下のURLを開く：  
   👉 https://colab.research.google.com/
2. Google アカウントでログイン（持っていない場合は作成）
3. 「ノートブックを新規作成」をクリック

### 2.2 Hello World

コードセル（灰色の入力欄）に以下を入力して、▶ ボタン（またはShift+Enter）で実行：

```python
print("Hello, World!")
```

**期待される出力：**
```
Hello, World!
```

> 💡 **ポイント**: これがPythonの最も基本的なプログラムです。`print()` は画面に文字を表示する命令です。

### 2.3 簡単な計算

新しいセルを追加（「+ コード」ボタン）して、以下を実行：

```python
# 足し算
a = 3
b = 5
print(a + b)
```

**期待される出力：**
```
8
```

もう少し試してみましょう：

```python
# いろいろな計算
print(10 - 3)   # 引き算 → 7
print(4 * 6)    # 掛け算 → 24
print(10 / 3)   # 割り算 → 3.333...
print(2 ** 10)  # 2の10乗 → 1024
```

### 2.4 リスト（配列）を使う

```python
# 数値のリスト
numbers = [1, 4, 9, 16, 25]
print(numbers)
print(len(numbers))  # 要素数 → 5
print(sum(numbers))  # 合計 → 55
```

### 2.5 matplotlib でグラフを描く

```python
import matplotlib.pyplot as plt

# データ
x = [1, 2, 3, 4, 5]
y = [1, 4, 9, 16, 25]

# グラフを描画
plt.plot(x, y, marker='o')
plt.xlabel('x')
plt.ylabel('y = x^2')
plt.title('My First Graph')
plt.grid(True)
plt.show()
```

**期待される結果：** x² のグラフが表示される

> 💡 **ポイント**: `matplotlib` は Python で最もよく使われるグラフ描画ライブラリです。研究で結果を可視化するときに必ず使います。

### 2.6 もう少し実践的な例

```python
import numpy as np
import matplotlib.pyplot as plt

# sin関数を描く
x = np.linspace(0, 2 * np.pi, 100)  # 0〜2πを100分割
y = np.sin(x)

plt.figure(figsize=(8, 4))
plt.plot(x, y, label='sin(x)')
plt.plot(x, np.cos(x), label='cos(x)', linestyle='--')
plt.xlabel('x')
plt.ylabel('y')
plt.title('Trigonometric Functions')
plt.legend()
plt.grid(True)
plt.show()
```

**期待される結果：** sin と cos の曲線が重なって表示される

---

## ステップ 3：GitHub アカウントの作成

### 3.1 アカウント作成

1. ブラウザで以下のURLを開く：  
   👉 https://github.com/
2. 「Sign up」をクリック
3. 以下を入力：
   - メールアドレス（大学のメールアドレス推奨）
   - パスワード
   - ユーザー名（英数字、短くて覚えやすいもの）
4. メール認証を完了

### 3.2 プロフィール設定

1. 右上のアイコン →「Settings」
2. 「Public profile」で名前を設定

> 💡 **ポイント**: GitHub はコードを保存・共有するためのサービスです。プログラマーの「作品集」のような役割もあります。次回（第2回）で実際にコードをアップロードします。

---

## 確認チェックリスト

以下が全てできたらこの回は完了です：

- [ ] VS Code が起動できる
- [ ] Colab で `print("Hello, World!")` が実行できた
- [ ] Colab で計算ができた（足し算、掛け算など）
- [ ] Colab で matplotlib のグラフが表示された
- [ ] GitHub アカウントを作成した

---

## 困ったときは

- **Colab が動かない**: ブラウザのキャッシュをクリア、またはシークレットウィンドウで試す
- **VS Code がインストールできない**: OSのバージョンを確認（古すぎると非対応）
- **GitHub のメール認証が来ない**: 迷惑メールフォルダを確認

---

## 次回予告

**第2回：Gitの基本**  
Colab で書いたコードを `.py` ファイルにして、Git で管理する方法を学びます。
