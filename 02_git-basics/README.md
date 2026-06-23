# 第2回：Gitの基本

この回では、コードの変更履歴を管理する「Git」の基本を学びます。  
Colab で書いたコードを `.py` ファイルにして、GitHub にアップロードするところまで行います。

---

## 目標

- [x] Git の基本概念を理解する
- [x] `git init` → `git add` → `git commit` の流れを体験する
- [x] GitHub にリポジトリを作って push する
- [x] VS Code の Git 機能を使う

---

## ステップ 1：Git とは何か

### 1.1 なぜ Git が必要？

プログラムを書いていると、こんなことが起きます：

```
my_code.py
my_code_v2.py
my_code_v2_final.py
my_code_v2_final_really_final.py  ← こうなりがち
```

**Git** を使えば、1つのファイルのまま変更履歴を全て記録できます。

> 💡 **ポイント**: Git = 「コードのタイムマシン」。いつでも過去の状態に戻れます。

### 1.2 基本用語

| 用語 | 意味 | たとえ |
|------|------|--------|
| **リポジトリ (repository)** | プロジェクトの保存場所 | フォルダ全体 |
| **コミット (commit)** | 変更を記録すること | セーブポイント |
| **プッシュ (push)** | ローカルの記録をGitHubに送ること | クラウドにバックアップ |
| **クローン (clone)** | GitHubからコピーを取得すること | ダウンロード |

---

## ステップ 2：Git のインストール確認

### Windows の場合

1. VS Code のターミナルを開く（`Ctrl + `` `  またはメニュー → ターミナル → 新しいターミナル）
2. 以下を入力して Enter：

```bash
git --version
```

**期待される出力：**
```
git version 2.xx.x
```

もし「コマンドが見つかりません」と出たら：
- 👉 https://git-scm.com/ からインストール

### Mac の場合

ターミナルで：
```bash
git --version
```

インストールされていない場合、自動でインストールが始まります（Xcode Command Line Tools）。

---

## ステップ 3：Git の初期設定

ターミナルで以下を実行（**自分の名前とメールアドレスに書き換えて**）：

```bash
git config --global user.name "Taro Yamada"
git config --global user.email "taro@example.com"
```

> ⚠️ **注意**: メールアドレスは GitHub アカウントに登録したものと同じにしてください。

確認：
```bash
git config --global user.name
git config --global user.email
```

---

## ステップ 4：最初のリポジトリを作る

### 4.1 フォルダを作成

```bash
mkdir my-first-repo
cd my-first-repo
```

### 4.2 Git リポジトリとして初期化

```bash
git init
```

**出力：**
```
Initialized empty Git repository in .../my-first-repo/.git/
```

> 💡 **ポイント**: `git init` をすると、そのフォルダが Git で管理されるようになります。`.git` という隠しフォルダが作られます。

---

## ステップ 5：ファイルを作ってコミットする

### 5.1 Python ファイルを作成

VS Code で `my-first-repo` フォルダを開き、`hello.py` を新規作成：

```python
# hello.py - 最初のPythonスクリプト

def greet(name):
    return f"Hello, {name}!"

if __name__ == "__main__":
    message = greet("World")
    print(message)
```

### 5.2 状態を確認

```bash
git status
```

**出力：**
```
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        hello.py
```

→ 「hello.py が新しいファイルとして検出されたが、まだ記録されていない」という意味

### 5.3 ファイルをステージング（記録対象に追加）

```bash
git add hello.py
```

もう一度確認：
```bash
git status
```

**出力：**
```
Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   hello.py
```

→ 「hello.py を次のコミットに含める準備ができた」

### 5.4 コミット（記録する）

```bash
git commit -m "最初のコミット: hello.pyを追加"
```

**出力：**
```
[main (root-commit) abc1234] 最初のコミット: hello.pyを追加
 1 file changed, 7 insertions(+)
 create mode 100644 hello.py
```

> 💡 **ポイント**: `-m` の後の文字列が「コミットメッセージ」です。何を変更したかを短く書きます。

---

## ステップ 6：変更してもう一度コミット

### 6.1 ファイルを編集

`hello.py` に機能を追加：

```python
# hello.py - 最初のPythonスクリプト

def greet(name):
    return f"Hello, {name}!"

def add(a, b):
    """2つの数を足す"""
    return a + b

if __name__ == "__main__":
    message = greet("World")
    print(message)
    
    result = add(3, 5)
    print(f"3 + 5 = {result}")
```

### 6.2 差分を確認

```bash
git diff
```

→ 赤い行（削除）と緑の行（追加）で変更点が表示される

### 6.3 コミット

```bash
git add hello.py
git commit -m "add関数を追加"
```

### 6.4 履歴を確認

```bash
git log --oneline
```

**出力：**
```
def5678 add関数を追加
abc1234 最初のコミット: hello.pyを追加
```

→ 2つのセーブポイントができた！

---

## ステップ 7：GitHub にプッシュする

### 7.1 GitHub でリポジトリを作成

1. https://github.com/new にアクセス
2. Repository name: `my-first-repo`
3. **「Add a README file」のチェックを外す**（重要！）
4. 「Create repository」をクリック

### 7.2 ローカルと GitHub を接続

GitHub の画面に表示されるコマンドを実行：

```bash
git remote add origin https://github.com/あなたのユーザー名/my-first-repo.git
git branch -M main
git push -u origin main
```

### 7.3 確認

GitHub のリポジトリページをリロード → `hello.py` が表示されていれば成功！

---

## ステップ 8：VS Code の Git 機能を使う

### 8.1 ソース管理パネル

1. VS Code の左側バーで、分岐のようなアイコン（ソース管理）をクリック
2. 変更したファイルが一覧表示される

### 8.2 GUI でコミット

1. ファイルを編集して保存
2. ソース管理パネルで変更されたファイルの横の「+」をクリック（= `git add`）
3. 上部のメッセージ欄にコミットメッセージを入力
4. ✓ ボタンをクリック（= `git commit`）

### 8.3 GUI でプッシュ

- ソース管理パネル上部の「...」→「Push」をクリック

> 💡 **ポイント**: コマンドラインでもGUIでも、やっていることは同じです。最初はコマンドで覚えて、慣れたらGUIを使うと効率的です。

---

## まとめ：Git の基本フロー

```
ファイルを編集
    ↓
git add ファイル名    ← 記録対象に追加
    ↓
git commit -m "メッセージ"  ← 記録する
    ↓
git push              ← GitHub に送る
```

---

## 確認チェックリスト

- [ ] `git --version` でバージョンが表示される
- [ ] `git config` で名前とメールを設定した
- [ ] `git init` でリポジトリを作成できた
- [ ] ファイルを作って `git add` → `git commit` できた
- [ ] `git log` でコミット履歴が確認できた
- [ ] GitHub にリポジトリを作成して `git push` できた
- [ ] VS Code のソース管理パネルが使える

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `fatal: not a git repository` | `git init` していないフォルダで実行 | `git init` を実行する |
| `nothing to commit` | 変更がない or `git add` を忘れている | ファイルを変更してから `git add` |
| `failed to push` | GitHub側にも変更がある | `git pull` してから `git push` |
| `Permission denied` | 認証の問題 | GitHub のパスワードやトークンを確認 |

---

## 次回予告

**第3回：SSHでサーバー接続**  
VS Code から DGX-Spark サーバーに接続し、サーバー上でプログラムを実行する方法を学びます。
