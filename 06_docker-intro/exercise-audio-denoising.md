# 演習：音声ノイズ除去（Speech Enhancement with Demucs）

この演習では、Meta が公開した **Demucs（denoiser）** を Docker 環境で動かし、ノイズの入った音声をクリーンにするタスクを体験します。事前学習済みモデルの利用から、自分でファインチューニング（追加学習）するところまでを学びます。

---

## この演習で学ぶこと

- 音声ノイズ除去（Speech Enhancement）の基本概念
- 事前学習済みモデルを使った推論
- 小規模データでのファインチューニング（学習ループ）
- W&B で学習過程を可視化

---

## Demucs (denoiser) とは

**Demucs** は Meta AI が開発した音声ノイズ除去モデルです。時間領域の波形を直接処理する U-Net 型アーキテクチャを採用しています。

```
入力: ノイズ入り音声（雑音が混ざった会話、環境音など）
  ↓
Demucs モデル（事前学習済み or ファインチューニング済み）
  ↓
出力: クリーン音声（ノイズが除去された音声）
```

> 💡 **ポイント**: DNSチャレンジ（Microsoft主催の音声ノイズ除去コンペ）のデータで学習済みモデルが公開されており、すぐに使えます。さらに自分のデータで追加学習（ファインチューニング）もできます。

### 事前学習済みモデル

| モデル名 | パラメータ | 特徴 |
|---------|-----------|------|
| dns48 | H=48 | 軽量、リアルタイム対応 |
| dns64 | H=64 | 中規模、高品質 |
| master64 | H=64 | DNS + Valentini で学習、最高品質 |

この演習では **dns48**（軽量版）から始めます。

---

## ステップ 1：プロジェクトの準備

### 1.1 作業ディレクトリの作成

```bash
mkdir -p ~/audio-denoising-exercise && cd ~/audio-denoising-exercise
```

### 1.2 ディレクトリ構成

```
audio-denoising-exercise/
├── Dockerfile
├── audio/
│   ├── noisy/       ← ノイズ入り音声
│   └── clean/       ← クリーン音声（学習用）
├── results/         ← ノイズ除去結果
├── run_denoise.py   ← 推論スクリプト
├── train_denoise.py ← 学習スクリプト
└── docker-compose.yml
```

```bash
mkdir -p audio/noisy audio/clean results
```

---

## ステップ 2：Dockerfile の作成

```dockerfile
# Dockerfile
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-devel

WORKDIR /workspace

# システムパッケージ（音声処理に必要）
RUN apt-get update && apt-get install -y \
    git \
    wget \
    sox \
    libsox-dev \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

# denoiser（Meta公式）のインストール
RUN git clone https://github.com/facebookresearch/denoiser.git /workspace/denoiser \
    && cd /workspace/denoiser \
    && pip install --no-cache-dir -e .

# 追加ライブラリ
RUN pip install --no-cache-dir \
    torchaudio \
    soundfile \
    matplotlib \
    numpy \
    pesq \
    pystoi

CMD ["bash"]
```

---

## ステップ 3：イメージのビルドとコンテナ起動

### 3.1 ビルド

```bash
cd ~/audio-denoising-exercise
docker build -t audio-denoise:v1 .
```

### 3.2 テスト音声の準備

ノイズ入り音声を用意します。簡易的にホワイトノイズを生成します：

```bash
# コンテナ起動
docker run --gpus all -it \
  -v $(pwd)/audio:/workspace/audio \
  -v $(pwd)/results:/workspace/results \
  audio-denoise:v1 bash
```

コンテナ内で：

```python
# generate_test_audio.py — テスト用のノイズ入り音声を生成
import torch
import torchaudio
import numpy as np

# サンプリングレート
sr = 16000
duration = 3  # 3秒

# クリーン音声（サイン波の組み合わせで疑似音声）
t = torch.linspace(0, duration, sr * duration)
clean = 0.5 * torch.sin(2 * np.pi * 440 * t) + 0.3 * torch.sin(2 * np.pi * 880 * t)
clean = clean.unsqueeze(0)  # (1, samples)

# ノイズ
noise = 0.1 * torch.randn_like(clean)

# ノイズ入り音声
noisy = clean + noise

# 保存
torchaudio.save("/workspace/audio/clean/sample_clean.wav", clean, sr)
torchaudio.save("/workspace/audio/noisy/sample_noisy.wav", noisy, sr)
print(f"クリーン音声: /workspace/audio/clean/sample_clean.wav")
print(f"ノイズ音声:   /workspace/audio/noisy/sample_noisy.wav")
print(f"サンプリングレート: {sr} Hz, 長さ: {duration} 秒")
```

```bash
python generate_test_audio.py
```

> 💡 実際の研究では、録音した音声や公開データセット（VoiceBank-DEMAND等）を使います。

---

## ステップ 4：事前学習済みモデルで推論（ノイズ除去）

```python
# run_denoise.py — 事前学習済み dns48 でノイズ除去
import torch
import torchaudio
from denoiser import pretrained
from denoiser.dsp import convert_audio

print("=" * 50)
print("Demucs 音声ノイズ除去デモ")
print("=" * 50)

# 事前学習済みモデルの読み込み
print("\n[1/3] モデルを読み込み中（dns48）...")
model = pretrained.dns48()
model.eval()
if torch.cuda.is_available():
    model = model.cuda()
    print("  → GPU モードで実行")
else:
    print("  → CPU モードで実行")

# ノイズ入り音声の読み込み
print("\n[2/3] 音声ファイルを読み込み中...")
noisy_path = "/workspace/audio/noisy/sample_noisy.wav"
wav, sr = torchaudio.load(noisy_path)
print(f"  → ファイル: {noisy_path}")
print(f"  → サンプリングレート: {sr} Hz")
print(f"  → 長さ: {wav.shape[1] / sr:.2f} 秒")

# モデルの期待するサンプリングレートに変換
wav = convert_audio(wav, sr, model.sample_rate, model.chin)
if torch.cuda.is_available():
    wav = wav.cuda()

# 推論（ノイズ除去）
print("\n[3/3] ノイズ除去実行中...")
with torch.no_grad():
    denoised = model(wav.unsqueeze(0))[0]

# 結果の保存
output_path = "/workspace/results/denoised_dns48.wav"
torchaudio.save(output_path, denoised.cpu(), model.sample_rate)
print(f"\n結果を保存: {output_path}")
print("=" * 50)
print("完了！ noisy → denoised の音声を比較してみてください。")
```

### 実行

```bash
python run_denoise.py
```

---

## ステップ 5：ファインチューニング（追加学習）

自分のデータでモデルを追加学習する方法を学びます。

```python
# train_denoise.py — 小規模データでファインチューニング
import torch
import torchaudio
import numpy as np
from denoiser import pretrained
from denoiser.dsp import convert_audio

print("=" * 50)
print("Demucs ファインチューニング デモ")
print("=" * 50)

# 事前学習済みモデルを読み込み（ここから追加学習する）
print("\n[1/4] 事前学習済みモデル (dns48) を読み込み中...")
model = pretrained.dns48()
model.train()
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)
print(f"  → デバイス: {device}")

# 学習データの準備（簡易版: ランダムノイズを付加した擬似データ）
print("\n[2/4] 学習データを生成中...")
sr = model.sample_rate
duration = 2  # 2秒

def generate_pair(sr, duration):
    """クリーン/ノイズペアを生成"""
    t = torch.linspace(0, duration, sr * duration)
    # ランダムな周波数のサイン波
    freq = np.random.uniform(200, 1000)
    clean = 0.5 * torch.sin(2 * np.pi * freq * t)
    clean = clean.unsqueeze(0)  # (1, samples)
    # ランダムな強度のノイズ
    noise_level = np.random.uniform(0.05, 0.2)
    noisy = clean + noise_level * torch.randn_like(clean)
    return noisy.to(device), clean.to(device)

# 学習ループ
print("\n[3/4] 学習開始...")
optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)
num_epochs = 20
batch_size = 4

losses = []
for epoch in range(num_epochs):
    epoch_loss = 0.0
    for _ in range(batch_size):
        noisy, clean = generate_pair(sr, duration)

        # forward
        estimated = model(noisy.unsqueeze(0))
        # L1 loss（波形の差）
        loss = torch.nn.functional.l1_loss(estimated, clean.unsqueeze(0))

        # backward
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        epoch_loss += loss.item()

    avg_loss = epoch_loss / batch_size
    losses.append(avg_loss)

    if (epoch + 1) % 5 == 0:
        print(f"  Epoch {epoch+1:3d}/{num_epochs} | Loss: {avg_loss:.6f}")

# 学習結果の確認
print(f"\n  初期 Loss: {losses[0]:.6f}")
print(f"  最終 Loss: {losses[-1]:.6f}")
print(f"  改善率:    {(1 - losses[-1]/losses[0]) * 100:.1f}%")

# 学習曲線の保存
print("\n[4/4] 学習曲線を保存中...")
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

plt.figure(figsize=(8, 4))
plt.plot(range(1, num_epochs + 1), losses, "b-o", markersize=3)
plt.xlabel("Epoch")
plt.ylabel("L1 Loss")
plt.title("Demucs Fine-tuning Loss Curve")
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("/workspace/results/training_curve.png", dpi=100)
print("  → /workspace/results/training_curve.png")

# ファインチューニング済みモデルの保存
model_path = "/workspace/results/model_finetuned.pt"
torch.save(model.state_dict(), model_path)
print(f"  → モデル保存: {model_path}")

print("\n" + "=" * 50)
print("ファインチューニング完了！")
print("次のステップ: 実際の音声データで学習してみましょう。")
```

### 実行

```bash
python train_denoise.py
```

**期待される出力：**
```
  Epoch   5/20 | Loss: 0.023456
  Epoch  10/20 | Loss: 0.015678
  Epoch  15/20 | Loss: 0.012345
  Epoch  20/20 | Loss: 0.010234

  初期 Loss: 0.034567
  最終 Loss: 0.010234
  改善率:    70.4%
```

---

## ステップ 6：結果の確認

### ホスト側で音声を確認

```bash
# コンテナを exit した後
ls results/
# → denoised_dns48.wav  training_curve.png  model_finetuned.pt
```

### 音声の比較

```bash
# 自分のPCに転送して聴き比べ
scp ユーザー名@192.168.50.218:~/audio-denoising-exercise/audio/noisy/sample_noisy.wav ./
scp ユーザー名@192.168.50.218:~/audio-denoising-exercise/results/denoised_dns48.wav ./
```

---

## ステップ 7：docker compose で管理する

### docker-compose.yml

```yaml
# docker-compose.yml
services:
  denoise:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./audio:/workspace/audio
      - ./results:/workspace/results
      - ./run_denoise.py:/workspace/run_denoise.py
      - ./train_denoise.py:/workspace/train_denoise.py
    command: python /workspace/run_denoise.py

  train:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./audio:/workspace/audio
      - ./results:/workspace/results
      - ./train_denoise.py:/workspace/train_denoise.py
    command: python /workspace/train_denoise.py
    profiles:
      - training
```

### 実行

```bash
# 推論（ノイズ除去）
docker compose up denoise

# 学習
docker compose --profile training up train
```

---

## チャレンジ課題

### Challenge 1：実際の音声で試す

自分で録音した音声（スマホの録音アプリなど）を `audio/noisy/` に置いて、ノイズ除去を試してみましょう。

### Challenge 2：VoiceBank-DEMAND データセットで学習

公開データセットを使って本格的に学習してみましょう：

```bash
# VoiceBank-DEMAND データセットのダウンロード（公式 denoiser の手順）
cd /workspace/denoiser
# egs/valentini/conf.yml を編集してパスを設定
python -m denoiser.audio dataset=valentini
```

### Challenge 3：音質評価指標を計算

```python
from pesq import pesq
from pystoi import stoi
import torchaudio

# クリーン音声と処理後音声を読み込み
clean, sr = torchaudio.load("/workspace/audio/clean/sample_clean.wav")
denoised, _ = torchaudio.load("/workspace/results/denoised_dns48.wav")

# 長さを揃える（モデル処理で微妙に長さが変わる場合がある）
min_len = min(clean.shape[1], denoised.shape[1])
clean = clean[:, :min_len]
denoised = denoised[:, :min_len]

# PESQ（音声品質）: -0.5 ~ 4.5（高いほど良い）
pesq_score = pesq(sr, clean.squeeze().numpy(), denoised.squeeze().numpy(), "wb")
print(f"PESQ: {pesq_score:.3f}")

# STOI（明瞭度）: 0 ~ 1（高いほど良い）
stoi_score = stoi(clean.squeeze().numpy(), denoised.squeeze().numpy(), sr, extended=False)
print(f"STOI: {stoi_score:.3f}")
```

### Challenge 4：W&B で学習を追跡

`train_denoise.py` に W&B を組み込みましょう：

```python
import wandb

wandb.init(project="audio-denoising")
# 学習ループ内で
wandb.log({"epoch": epoch, "loss": avg_loss})
# 音声もログ
wandb.log({"noisy": wandb.Audio(noisy_np, sample_rate=sr),
           "denoised": wandb.Audio(denoised_np, sample_rate=sr)})
```

---

## Google Colab で実行する場合

Docker が使えない環境（自宅など）では、Colab で同等の体験ができます：

```python
# Colab のセル1：インストール
!git clone https://github.com/facebookresearch/denoiser.git
%cd denoiser
!pip install -e .
!pip install torchaudio soundfile pesq pystoi

# Colab のセル2：ランタイムを「GPU」に変更してから実行
# （ランタイム → ランタイムのタイプを変更 → GPU）
```

> ⚠️ Colab でも GPU なしで動作しますが、学習は GPU ありの方が圧倒的に速いです。

---

## 発展：WavLM を使った高性能ノイズ除去

より高性能なノイズ除去を目指す場合は、**WavLM**（Microsoft）を特徴抽出器として使う方法があります：

```
WavLM（自己教師あり事前学習）
  → 音声の「意味的な表現」を抽出
  → ノイズ除去ネットワークに入力
  → 少ないデータでも高品質なノイズ除去が可能
```

研究テーマとして発展させたい場合は、以下を参照してください：
- [WavLM 公式リポジトリ](https://github.com/microsoft/unilm/tree/master/wavlm)
- [PASE: DeWavLM による音声ノイズ除去](https://github.com/cisco-open/pase)

---

## まとめ

| 学んだこと | コマンド/概念 |
|-----------|--------------|
| 音声ノイズ除去の概念 | ノイズ入り → モデル → クリーン音声 |
| 事前学習済みモデルの利用 | `pretrained.dns48()` |
| ファインチューニング | optimizer, loss, 学習ループ |
| 音質評価 | PESQ, STOI |
| Docker での音声処理環境 | sox, ffmpeg, torchaudio |
