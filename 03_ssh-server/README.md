# 第3回：SSHでサーバー接続

この回では、ターミナル（コマンドライン）の基本操作を学び、SSH を使って DGX-Spark サーバーに接続します。  
C208教室内で実際にサーバーにログインし、自分のアカウントを作成して、SSH鍵認証を設定するところまで行います。

---

## 目標

- [x] ターミナルの基本コマンドを使える
- [x] C208 のネットワークに接続し、DGX-Spark に SSH でアクセスする
- [x] 自分のユーザーアカウントを作成する
- [x] SSH 鍵を作成し、鍵認証でログインできるようにする
- [x] VS Code Remote-SSH でサーバーに接続する

---

## 前提条件

- C208 教室にいること（DGX-Spark はローカルネットワーク内にあるため）
- VS Code がインストール済みであること（第1回参照）

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

## ステップ 2：C208 のネットワークに接続する

### 2.1 DGX-Spark について

```
┌──────────┐     C208 shuhari WiFi     ┌──────────────┐
│  自分のPC │  ─────────────────────→   │ DGX-Spark    │
│ (クライアント)│                          │ (spark-jotoku)│
└──────────┘                            │ 192.168.50.218│
                                        └──────────────┘
```

**DGX-Spark (spark-jotoku)** は C208 のローカルネットワーク内にある高性能GPUサーバーです。  
ローカルネットワーク内にあるため、**C208 の WiFi に接続した状態**でないとアクセスできません。

### 2.2 WiFi に接続する

1. PCの WiFi 設定を開く
2. SSID: **`admin`** を選択
3. パスワード: **C208 の DGX-Spark 本体に貼ってある付箋を確認してください**

> ⚠️ **注意**: WiFi パスワードはセキュリティのため、このドキュメントには記載していません。C208 教室内の DGX-Spark（情報特別演習用）本体に付箋で貼ってありますので、現地で確認してください。

### 2.3 接続の確認

WiFi 接続後、サーバーに到達できるか確認：

```bash
ping 192.168.50.218
```

**期待される出力：**
```
PING 192.168.50.218: 64 bytes from 192.168.50.218: icmp_seq=0 time=2.1 ms
```

→ 応答があればネットワーク接続OK。`Ctrl + C` で停止。

応答がない場合は WiFi の接続先が正しいか確認してください。

---

## ステップ 3：admin アカウントで SSH 接続する

最初は共有の admin アカウントを使ってサーバーにログインします。

### 3.1 SSH 接続

```bash
ssh admin@192.168.50.218
```

初回は以下のメッセージが出ます：
```
The authenticity of host '192.168.50.218' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
→ `yes` と入力して Enter

パスワードを聞かれます：
```
admin@192.168.50.218's password:
```
→ **DGX-Spark 本体の付箋に記載されている admin パスワード** を入力（画面に表示されませんが入力されています）

### 3.2 ログイン成功の確認

ログインに成功すると、プロンプトが変わります：
```
admin@spark-jotoku:~$
```

以下のコマンドで確認：
```bash
whoami       # → admin
hostname     # → spark-jotoku
```

---

## ステップ 4：自分のユーザーアカウントを作成する

admin でログインした状態で、自分専用のアカウントを作成します。

### 4.1 アカウントの作成

`自分のユーザー名` を決めてください（例：名前のローマ字 `tanaka`, `suzuki` など）。

```bash
sudo adduser 自分のユーザー名
```

以下の質問に回答します：
```
New password:            ← 自分のパスワードを設定（覚えておく！）
Retype new password:     ← 同じパスワードをもう一度
Full Name []:            ← 氏名（空Enterでも可）
Room Number []:          ← 空Enter
Work Phone []:           ← 空Enter
Home Phone []:           ← 空Enter
Other []:                ← 空Enter
Is the information correct? [Y/n]   ← Y
```

### 4.2 sudo 権限の付与（必要に応じて）

作成したアカウントに管理者権限を付与する場合：

```bash
sudo usermod -aG sudo 自分のユーザー名
```

### 4.3 アカウント作成の確認

```bash
id 自分のユーザー名
```

**期待される出力（例）：**
```
uid=1001(tanaka) gid=1001(tanaka) groups=1001(tanaka),27(sudo)
```

### 4.4 admin からログアウト

```bash
exit
```

---

## ステップ 5：自分のアカウントで SSH 接続する

### 5.1 パスワード認証でログイン

作成した自分のアカウントで接続：

```bash
ssh 自分のユーザー名@192.168.50.218
```

ステップ4で設定したパスワードを入力 → ログイン成功

```bash
whoami       # → 自分のユーザー名が表示されるはず
```

確認できたら一旦ログアウト：
```bash
exit
```

---

## ステップ 6：SSH 鍵を作成する（自分のPC側）

パスワード認証のままだと毎回パスワードを入力する必要があるため、SSH鍵認証を設定します。

### 6.1 鍵の仕組み

SSH鍵は「錠前と鍵」のペアです：
- **公開鍵 (public key)** = 錠前（サーバーに置く）
- **秘密鍵 (private key)** = 鍵（自分のPCに保管、**絶対に他人に渡さない**）

### 6.2 鍵の作成（自分のPC のターミナルで実行）

> ⚠️ これは**自分のPC**で実行します。サーバーにログインした状態ではありません。

```bash
ssh-keygen -t ed25519 -C "自分のメールアドレス"
```

質問に対する回答：
```
Enter file in which to save the key (/home/ユーザー名/.ssh/id_ed25519): （そのままEnter）
Enter passphrase (empty for no passphrase): （そのままEnter、またはパスフレーズを設定）
Enter same passphrase again: （同じものを入力）
```

### 6.3 鍵の確認

```bash
ls ~/.ssh/
```

**出力：**
```
id_ed25519       ← 秘密鍵（絶対に他人に渡さない！）
id_ed25519.pub   ← 公開鍵（サーバーに登録する）
```

### 6.4 公開鍵の内容を表示

```bash
cat ~/.ssh/id_ed25519.pub
```

出力された文字列（`ssh-ed25519 AAAA...` から始まる1行）を**コピー**しておきます。

---

## ステップ 7：公開鍵をサーバーに登録する

### 7.1 方法A：ssh-copy-id を使う（簡単）

自分のPCから以下を実行：

```bash
ssh-copy-id 自分のユーザー名@192.168.50.218
```

パスワードを聞かれるので入力 → 自動的に公開鍵が登録されます。

### 7.2 方法B：手動で登録する

方法A がうまくいかない場合（Windowsなど）：

1. パスワード認証でサーバーにログイン：
```bash
ssh 自分のユーザー名@192.168.50.218
```

2. サーバー上で公開鍵を登録：
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ここにステップ6.4でコピーした公開鍵を貼り付け" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

3. ログアウト：
```bash
exit
```

### 7.3 鍵認証の確認

再度 SSH 接続を試みます：

```bash
ssh 自分のユーザー名@192.168.50.218
```

**パスワードを聞かれずにログインできれば成功！** 🎉

---

## ステップ 8：SSH 設定ファイルを作る

毎回 `ssh ユーザー名@192.168.50.218` と打つのは面倒なので、設定ファイルを作ります。

### 8.1 設定ファイルの作成

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

### 8.2 簡略化された接続

これで以下のように短く接続できます：

```bash
ssh spark-jotoku
```

---

## ステップ 9：VS Code Remote-SSH で接続

### 9.1 拡張機能のインストール

1. VS Code を開く
2. 拡張機能（四角4つアイコン）をクリック
3. `Remote - SSH` と検索
4. 「Remote - SSH」（Microsoft製）をインストール

### 9.2 接続

1. VS Code 左下の青い `><` アイコンをクリック
2. 「Connect to Host...」を選択
3. `spark-jotoku` を選択（ステップ8で設定した名前）
4. 新しいウィンドウが開き、サーバーに接続される

### 9.3 フォルダを開く

接続後：
1. 「フォルダーを開く」をクリック
2. `/home/自分のユーザー名` を選択
3. サーバー上のファイルを VS Code で直接編集できる！

### 9.4 ターミナルを開く

VS Code 内で `Ctrl + `` ` → サーバー上のターミナルが開く

> 💡 **ポイント**: VS Code Remote-SSH を使えば、まるで自分のPCのファイルのように、サーバー上のファイルを編集・実行できます。

---

## 全体の流れまとめ

```
1. C208 で WiFi (admin) に接続
       ↓
2. ssh admin@192.168.50.218 でサーバーにログイン
       ↓
3. sudo adduser で自分のアカウント作成
       ↓
4. exit → 自分のアカウントでパスワードログイン確認
       ↓
5. 自分のPCで SSH鍵を作成 (ssh-keygen)
       ↓
6. 公開鍵をサーバーに登録 (ssh-copy-id)
       ↓
7. パスワードなしで接続できることを確認
       ↓
8. ~/.ssh/config を設定 → ssh spark-jotoku で接続
       ↓
9. VS Code Remote-SSH で快適に開発
```

---

## 確認チェックリスト

- [ ] C208 の WiFi (SSID: admin) に接続した
- [ ] `ping 192.168.50.218` で応答がある
- [ ] admin アカウントで SSH ログインできた
- [ ] `sudo adduser` で自分のアカウントを作成した
- [ ] 自分のアカウントでパスワード認証ログインできた
- [ ] SSH 鍵ペアを作成した（`~/.ssh/id_ed25519` と `id_ed25519.pub`）
- [ ] 公開鍵をサーバーに登録した
- [ ] パスワードなしで `ssh spark-jotoku` で接続できる
- [ ] VS Code Remote-SSH でサーバーに接続できる

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `ping: Request timeout` | WiFi に接続していない or 接続先が違う | SSID が `admin` になっているか確認 |
| `Connection refused` | サーバーのSSHサービスが起動していない | 先生に連絡 |
| `Permission denied (password)` | パスワードが間違い | 大文字小文字に注意して再入力 |
| `Permission denied (publickey)` | 鍵が正しく登録されていない | ステップ7を再確認、authorized_keys の内容を確認 |
| `Connection timed out` | ネットワーク外からアクセスしている | C208の WiFi に接続しているか確認 |
| `Host key verification failed` | サーバーの鍵が変わった | `ssh-keygen -R 192.168.50.218` で古い鍵を削除 |
| `adduser: command not found` | admin に sudo 権限がない場合 | 先生に連絡 |
| `ssh-copy-id: command not found` | Windows で使えない場合がある | 方法B（手動登録）を使う |

---

## 次回予告

**第4回：サーバー上でPythonを動かす**  
DGX-Spark 上で Python 仮想環境を作り、PyTorch で GPU を使ったプログラムを実行します。
