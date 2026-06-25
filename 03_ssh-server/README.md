# 第3回：SSHでサーバー接続

この回では、ターミナル（コマンドライン）の基本操作を学び、SSH を使って DGX-Spark サーバーに接続します。  
さらに VS Code から直接サーバー上のファイルを編集できるようにします。

---

## 目標

- [x] ターミナルの基本コマンドを使える
- [x] SSH 鍵を作成する
- [x] SSH でサーバーに接続する
- [x] VS Code Remote-SSH でサーバーに接続する

---

## ステップ 1：ターミナルの基本操作

### 1.1 ターミナルを開く

- **Windows**: VS Code のターミナル（`Ctrl + `` `）、または Git Bash
- **Mac**: アプリケーション → ユーティリティ → ターミナル

### 1.2 基本コマンド一覧

| コマンド | 意味 | 例 |
|----------|------|-----|
| `pwd` | 今いる場所を表示 | `pwd` → `/home/user` |
| `ls` | ファイル一覧を表示 | `ls` → `file1.py  file2.py` |
| `ls -la` | 詳細一覧（隠しファイル含む） | `ls -la` |
| `cd` | ディレクトリを移動 | `cd Documents` |
| `cd ..` | 一つ上に戻る | `cd ..` |
| `mkdir` | フォルダを作成 | `mkdir my-project` |
| `touch` | 空ファイルを作成 | `touch hello.py` |
| `cat` | ファイルの中身を表示 | `cat hello.py` |
| `rm` | ファイルを削除 | `rm hello.py` |
| `cp` | ファイルをコピー | `cp a.py b.py` |
| `mv` | ファイルを移動/名前変更 | `mv old.py new.py` |

### 1.3 練習

以下を順番に実行して、動作を確認してください：

```bash
pwd                    # 今いる場所を確認
mkdir test-folder      # フォルダを作成
cd test-folder         # フォルダに移動
pwd                    # 移動したことを確認
touch sample.txt       # ファイルを作成
ls                     # ファイルがあることを確認
echo "Hello" > sample.txt  # ファイルに書き込み
cat sample.txt         # 中身を確認 → "Hello"
cd ..                  # 一つ上に戻る
rm -r test-folder      # フォルダごと削除
```

> 💡 **ポイント**: `Tab` キーを押すとファイル名やコマンドが自動補完されます。積極的に使いましょう。

---

## ステップ 2：SSH とは

### 2.1 概念

```
┌──────────┐         SSH          ┌──────────────┐
│  自分のPC │  ───────────────→   │ DGX-Spark    │
│ (クライアント)│                    │ (サーバー)    │
└──────────┘                      └──────────────┘
```

**SSH (Secure Shell)** = ネットワーク越しに別のコンピュータを操作するための仕組み

- 自分のPC から DGX-Spark サーバーにログインできる
- サーバー上でコマンドを実行できる
- ファイルの転送もできる

### 2.2 接続に必要なもの

| 項目 | 値 |
|------|-----|
| サーバーのIPアドレス | `192.168.50.218` (spark-jotoku) |
| ユーザー名 | 先生から配布されたもの |
| 認証方式 | SSH鍵（次のステップで作成） |
| ネットワーク | C208教室の shuhari WiFi (SSID: admin) |

> ⚠️ **注意**: DGX-Spark は C208 のローカルネットワーク内にあるため、**shuhari WiFi に接続した状態**でないとアクセスできません。

---

## ステップ 3：SSH 鍵の作成

### 3.1 鍵の仕組み

SSH鍵は「錠前と鍵」のペアです：
- **公開鍵 (public key)** = 錠前（サーバーに置く）
- **秘密鍵 (private key)** = 鍵（自分のPCに保管、絶対に他人に渡さない）

### 3.2 鍵の作成

ターミナルで以下を実行：

```bash
ssh-keygen -t ed25519 -C "自分のメールアドレス"
```

質問に対する回答：
```
Enter file in which to save the key: （そのままEnter）
Enter passphrase: （そのままEnter、またはパスワードを設定）
Enter same passphrase again: （同じものを入力）
```

### 3.3 確認

```bash
ls ~/.ssh/
```

**出力：**
```
id_ed25519       ← 秘密鍵（絶対に他人に渡さない！）
id_ed25519.pub   ← 公開鍵（サーバーに登録する）
```

### 3.4 公開鍵の内容を確認

```bash
cat ~/.ssh/id_ed25519.pub
```

出力された文字列（`ssh-ed25519 AAAA...` から始まる1行）をコピーしておきます。

---

## ステップ 4：サーバーに SSH 接続する

### 4.1 初回接続（パスワード認証）

先生からユーザー名とパスワードを受け取ったら：

```bash
ssh ユーザー名@192.168.50.218
```

初回は以下のメッセージが出ます：
```
Are you sure you want to continue connecting (yes/no)?
```
→ `yes` と入力して Enter

パスワードを入力（画面に表示されないが入力されている）

### 4.2 公開鍵の登録

サーバーにログインした状態で：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ここに公開鍵の文字列を貼り付け" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 4.3 鍵認証で接続テスト

一度 `exit` でサーバーから抜けて、再度接続：

```bash
exit
ssh ユーザー名@192.168.50.218
```

パスワードを聞かれずにログインできれば成功！

---

## ステップ 5：SSH の設定ファイルを作る

毎回 `ssh ユーザー名@192.168.50.218` と打つのは面倒なので、設定ファイルを作ります。

### 5.1 設定ファイルの作成

自分のPC のターミナルで：

```bash
# Windowsの場合
notepad ~/.ssh/config

# Macの場合
nano ~/.ssh/config
```

以下の内容を書いて保存：

```
Host spark-jotoku
    HostName 192.168.50.218
    User 自分のユーザー名
    IdentityFile ~/.ssh/id_ed25519
```

### 5.2 簡略化された接続

これで以下のように短く接続できます：

```bash
ssh spark-jotoku
```

---

## ステップ 6：VS Code Remote-SSH で接続

### 6.1 拡張機能のインストール

1. VS Code を開く
2. 拡張機能（四角4つアイコン）をクリック
3. `Remote - SSH` と検索
4. 「Remote - SSH」（Microsoft製）をインストール

### 6.2 接続

1. VS Code 左下の青い `><` アイコンをクリック
2. 「Connect to Host...」を選択
3. `spark-jotoku` を選択（ステップ5で設定した名前）
4. 新しいウィンドウが開き、サーバーに接続される

### 6.3 フォルダを開く

接続後：
1. 「フォルダーを開く」をクリック
2. サーバー上のフォルダ（例：`/home/自分のユーザー名`）を選択
3. サーバー上のファイルを VS Code で直接編集できる！

### 6.4 ターミナルを開く

VS Code 内で `Ctrl + `` ` → サーバー上のターミナルが開く

> 💡 **ポイント**: VS Code Remote-SSH を使えば、まるで自分のPCのファイルのように、サーバー上のファイルを編集・実行できます。

---

## 確認チェックリスト

- [ ] `pwd`, `ls`, `cd`, `mkdir` など基本コマンドが使える
- [ ] SSH 鍵ペアを作成した（`~/.ssh/id_ed25519` と `id_ed25519.pub`）
- [ ] `ssh spark-jotoku` でサーバーに接続できる
- [ ] VS Code Remote-SSH でサーバーに接続できる
- [ ] VS Code からサーバー上のファイルを開ける

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `Connection refused` | サーバーが起動していない or WiFi未接続 | shuhari WiFi に接続しているか確認 |
| `Permission denied` | 鍵が正しく設定されていない | 公開鍵の登録手順を再確認 |
| `Connection timed out` | IPアドレスが間違い or ネットワーク外 | C208のWiFiに接続しているか確認 |
| `Host key verification failed` | サーバーの鍵が変わった | `ssh-keygen -R 192.168.50.218` で古い鍵を削除 |

---

## 次回予告

**第4回：サーバー上でPythonを動かす**  
DGX-Spark 上で Python 仮想環境を作り、PyTorch で GPU を使ったプログラムを実行します。
