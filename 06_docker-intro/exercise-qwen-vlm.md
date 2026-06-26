# 演習：Qwen2.5-VL で画像理解（Vision Language Model）

この演習では、Alibaba が公開した **Qwen2.5-VL**（Vision Language Model）を Docker 環境で動かし、画像を入力して自然言語で理解・説明させるタスクを体験します。

---

## この演習で学ぶこと

- マルチモーダル基盤モデル（画像+言語）の利用方法
- 公式 Docker イメージを使ったモデル環境構築
- プロンプトエンジニアリング（画像に対する質問の仕方）

---

## Qwen2.5-VL とは

**Qwen2.5-VL** は Alibaba Qwen チームが開発した Vision-Language モデルです。

```
入力: 画像 + テキスト（質問や指示）
  ↓
Qwen2.5-VL モデル（事前学習済み）
  ↓
出力: 自然言語による回答（説明、分析、OCRなど）
```

> 💡 **ポイント**: 画像の内容を「見て理解し、言葉で説明する」AIです。OCR（文字認識）、図表の解釈、物体の説明など幅広いタスクに対応します。

### モデルサイズ

| モデル | パラメータ数 | VRAM目安 | 用途 |
|--------|-------------|----------|------|
| Qwen2.5-VL-3B-Instruct | 30億 | ~7GB | 学習用（推奨） |
| Qwen2.5-VL-7B-Instruct | 70億 | ~16GB | バランス型 |
| Qwen2.5-VL-72B-Instruct | 720億 | ~150GB | 最高性能 |

この演習では **3B（30億パラメータ）** を使います。DGX-Spark で十分動作します。

---

## ステップ 1：プロジェクトの準備

### 1.1 作業ディレクトリの作成

```bash
mkdir -p ~/qwen-vlm-exercise && cd ~/qwen-vlm-exercise
```

### 1.2 ディレクトリ構成

```
qwen-vlm-exercise/
├── Dockerfile
├── images/          ← テスト用画像を置く
├── results/         ← 出力結果の保存先
├── run_qwen.py      ← 推論スクリプト
└── docker-compose.yml
```

```bash
mkdir -p images results
```

---

## ステップ 2：Dockerfile の作成

```dockerfile
# Dockerfile
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-devel

WORKDIR /workspace

# システムパッケージ
RUN apt-get update && apt-get install -y \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Qwen2.5-VL に必要なライブラリ
RUN pip install --no-cache-dir \
    "transformers>=4.45.0" \
    accelerate \
    qwen-vl-utils \
    pillow \
    torchvision

# モデルは初回実行時に自動ダウンロードされる（HuggingFace cache）
# 事前にダウンロードしたい場合は以下を有効化：
# RUN python -c "from transformers import AutoModelForImageTextToText, AutoProcessor; \
#     AutoModelForImageTextToText.from_pretrained('Qwen/Qwen2.5-VL-3B-Instruct'); \
#     AutoProcessor.from_pretrained('Qwen/Qwen2.5-VL-3B-Instruct')"

CMD ["bash"]
```

> 💡 **公式 Docker イメージ** `qwenllm/qwenvl:2.5-cu121` も利用可能ですが、ここでは Dockerfile の書き方を学ぶために自作します。

---

## ステップ 3：イメージのビルドとコンテナ起動

### 3.1 ビルド

```bash
cd ~/qwen-vlm-exercise
docker build -t qwen-vlm:v1 .
```

> ⏱ ライブラリのインストールに数分かかります。

### 3.2 テスト画像の準備

```bash
# サンプル画像をダウンロード（風景写真）
wget -O images/sample.jpg "https://upload.wikimedia.org/wikipedia/commons/thumb/1/1e/Sunrise_over_the_sea.jpg/1280px-Sunrise_over_the_sea.jpg"
```

自分の画像を使う場合は `images/` フォルダに置いてください。

### 3.3 コンテナの起動

```bash
docker run --gpus all -it \
  -v $(pwd)/images:/workspace/images \
  -v $(pwd)/results:/workspace/results \
  qwen-vlm:v1 bash
```

---

## ステップ 4：画像理解の実行（単一画像）

コンテナ内で以下のスクリプトを作成・実行します：

```python
# run_qwen.py — 画像を入力して自然言語で説明させる
import torch
from transformers import Qwen2_5_VLForConditionalGeneration, AutoProcessor
from qwen_vl_utils import process_vision_info
from PIL import Image
import json
import os

print("=" * 50)
print("Qwen2.5-VL 画像理解デモ")
print("=" * 50)

# モデルとプロセッサの読み込み（初回はダウンロードに時間がかかる）
print("\n[1/3] モデルを読み込み中...")
model = Qwen2_5_VLForConditionalGeneration.from_pretrained(
    "Qwen/Qwen2.5-VL-3B-Instruct",
    torch_dtype=torch.float16,
    device_map="auto",
)
processor = AutoProcessor.from_pretrained("Qwen/Qwen2.5-VL-3B-Instruct")
print("  → モデル読み込み完了")

# 画像の読み込み
image_path = "/workspace/images/sample.jpg"
print(f"\n[2/3] 画像を読み込み中: {image_path}")

# メッセージの構築（チャット形式）
messages = [
    {
        "role": "user",
        "content": [
            {"type": "image", "image": f"file://{image_path}"},
            {"type": "text", "text": "この画像に何が写っていますか？詳しく説明してください。"},
        ],
    }
]

# 推論の実行
print("\n[3/3] 推論実行中...")
text = processor.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
image_inputs, video_inputs = process_vision_info(messages)
inputs = processor(
    text=[text],
    images=image_inputs,
    videos=video_inputs,
    padding=True,
    return_tensors="pt",
).to(model.device)

generated_ids = model.generate(**inputs, max_new_tokens=512)
generated_ids_trimmed = [
    out_ids[len(in_ids):] for in_ids, out_ids in zip(inputs.input_ids, generated_ids)
]
output_text = processor.batch_decode(
    generated_ids_trimmed, skip_special_tokens=True, clean_up_tokenization_spaces=False
)

# 結果の表示
print("\n" + "=" * 50)
print("回答:")
print("=" * 50)
print(output_text[0])

# 結果の保存
result = {
    "image": image_path,
    "question": "この画像に何が写っていますか？詳しく説明してください。",
    "answer": output_text[0],
}
output_path = "/workspace/results/result_single.json"
with open(output_path, "w", encoding="utf-8") as f:
    json.dump(result, f, ensure_ascii=False, indent=2)
print(f"\n結果を保存: {output_path}")
```

### 実行

```bash
# コンテナ内で
python run_qwen.py
```

> ⏱ 初回はモデルのダウンロード（~7GB）に時間がかかります。2回目以降はキャッシュから読み込まれます。

---

## ステップ 5：複数の質問を試す

異なるプロンプトで同じ画像に質問してみましょう：

```python
# run_qwen_multi.py — 複数の質問を試す
import torch
from transformers import Qwen2_5_VLForConditionalGeneration, AutoProcessor
from qwen_vl_utils import process_vision_info
import json

# モデル読み込み
model = Qwen2_5_VLForConditionalGeneration.from_pretrained(
    "Qwen/Qwen2.5-VL-3B-Instruct",
    torch_dtype=torch.float16,
    device_map="auto",
)
processor = AutoProcessor.from_pretrained("Qwen/Qwen2.5-VL-3B-Instruct")

image_path = "/workspace/images/sample.jpg"

# いろいろな質問を試す
questions = [
    "この画像を一言で表現してください。",
    "この画像の色使いについて分析してください。",
    "この画像はいつ、どこで撮影されたと思いますか？根拠も教えてください。",
    "この画像を見て、俳句を一つ詠んでください。",
]

results = []
for i, question in enumerate(questions):
    print(f"\n--- 質問 {i+1}: {question}")

    messages = [
        {
            "role": "user",
            "content": [
                {"type": "image", "image": f"file://{image_path}"},
                {"type": "text", "text": question},
            ],
        }
    ]

    text = processor.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
    image_inputs, video_inputs = process_vision_info(messages)
    inputs = processor(
        text=[text],
        images=image_inputs,
        videos=video_inputs,
        padding=True,
        return_tensors="pt",
    ).to(model.device)

    generated_ids = model.generate(**inputs, max_new_tokens=256)
    generated_ids_trimmed = [
        out_ids[len(in_ids):] for in_ids, out_ids in zip(inputs.input_ids, generated_ids)
    ]
    answer = processor.batch_decode(
        generated_ids_trimmed, skip_special_tokens=True, clean_up_tokenization_spaces=False
    )[0]

    print(f"  回答: {answer}")
    results.append({"question": question, "answer": answer})

# 結果の保存
with open("/workspace/results/result_multi.json", "w", encoding="utf-8") as f:
    json.dump(results, f, ensure_ascii=False, indent=2)
print("\n結果を保存: /workspace/results/result_multi.json")
```

---

## ステップ 6：結果の確認

### ホスト側で確認

```bash
# コンテナを exit した後、ホスト側で
cat results/result_single.json | python -m json.tool
```

### DGX-Spark から自分の PC に転送

```bash
# 自分のPC側で実行
scp ユーザー名@192.168.50.218:~/qwen-vlm-exercise/results/*.json ./
```

---

## ステップ 7：docker compose で管理する

### docker-compose.yml

ホスト側で `run_qwen.py` を保存してから使います：

```yaml
# docker-compose.yml
services:
  qwen:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./images:/workspace/images
      - ./results:/workspace/results
      - ./run_qwen.py:/workspace/run_qwen.py
      - huggingface_cache:/root/.cache/huggingface
    command: python /workspace/run_qwen.py

volumes:
  huggingface_cache:
```

> 💡 `huggingface_cache` を名前付きボリュームにすることで、コンテナを削除してもダウンロード済みモデルが保持されます（~7GBの再ダウンロード回避）。

### 実行

```bash
# ホスト側で run_qwen.py を保存（ステップ4の内容をファイルに書き出す）
docker compose up
```

---

## チャレンジ課題

### Challenge 1：自分の研究画像を理解させる

自分の研究で使う画像（顕微鏡画像、グラフ、実験結果など）を `images/` に置いて質問してみましょう。

### Challenge 2：OCR（文字認識）を試す

文字が写っている画像に対して：
```python
{"type": "text", "text": "この画像に書かれている文字をすべて読み取ってください。"}
```

### Challenge 3：複数画像の比較

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "image", "image": "file:///workspace/images/image1.jpg"},
            {"type": "image", "image": "file:///workspace/images/image2.jpg"},
            {"type": "text", "text": "2つの画像の違いを説明してください。"},
        ],
    }
]
```

### Challenge 4：W&B に結果を記録

```python
import wandb

wandb.init(project="qwen-vlm-exercise")
wandb.log({
    "image": wandb.Image(image_path),
    "question": question,
    "answer": answer,
})
```

---

## Google Colab で実行する場合

Docker が使えない環境（自宅など）では、Colab で同等の体験ができます：

```python
# Colab のセル1：インストール
!pip install transformers accelerate qwen-vl-utils torch torchvision

# Colab のセル2：ランタイムを「GPU」に変更してから実行
# （ランタイム → ランタイムのタイプを変更 → GPU）
```

> ⚠️ Colab無料枠のGPU（T4, 15GB VRAM）で 3B モデルは動作します。7B以上は Colab Pro が必要です。

---

## まとめ

| 学んだこと | コマンド/概念 |
|-----------|--------------|
| VLM の概念 | 画像+テキスト → 自然言語回答 |
| Docker でモデル環境構築 | Dockerfile, docker build |
| HuggingFace モデルの利用 | transformers, AutoProcessor |
| プロンプトエンジニアリング | 質問の仕方で回答が変わる |
| ボリュームでキャッシュ保持 | 名前付きボリューム |
