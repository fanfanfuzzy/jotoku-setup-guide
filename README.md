# 情報特別演習 環境構築ガイド

このリポジトリは、情報特別演習の学生向け環境構築ガイドです。  
各回の手順書をステップバイステップで追いながら、研究に必要な基本ツールの使い方を身につけます。

## 目次

| 回 | テーマ | 内容 |
|----|--------|------|
| 第1回 | [PCの準備とColab体験](./01_pc-setup-and-colab/README.md) | VS Code導入、Colabで Python体験、GitHubアカウント作成 |
| 第2回 | [Gitの基本](./02_git-basics/README.md) | git init/add/commit、GitHubにpush、VS CodeのGit機能 |
| 第3回 | [SSHでサーバー接続](./03_ssh-server/README.md) | ターミナル基本操作、SSH鍵、VS Code Remote-SSH |
| 第4回 | [サーバー上でPythonを動かす](./04_python-on-server/README.md) | 仮想環境、pip、GPU確認、分類演習 |
| 第5回 | [実験の実行と管理](./05_experiment-management/README.md) | バックグラウンド実行、結果整理、Git管理 |
| 第6回 | [Docker入門](./06_docker-intro/README.md) | コンテナの概念、Dockerfile作成 |

## ColabとVS Codeの使い分け

| | Google Colab | VS Code |
|---|---|---|
| 用途 | 試す・学ぶ | 作る・管理する |
| 環境構築 | 不要（ブラウザだけ） | インストール必要 |
| GPU | 無料枠あり | サーバー接続が必要 |
| 自宅利用 | ✅ どこでも | ✅（ローカル） / サーバーはC208のみ |
| ファイル管理 | ドライブ保存 | ローカル + Git |

> **Colab** = ノートとペン（手軽にメモ・お試し）  
> **VS Code** = 作業机と工具箱（本格的にものを作る）

## 計算環境

| 環境 | アクセス | 用途 |
|------|----------|------|
| Google Colab | どこでも（ブラウザ） | 学習、小規模実験 |
| DGX-Spark (spark-jotoku) | C208教室内（shuhari WiFi） | 本格的なGPU学習・評価 |

## 最終目標

小さくてもよいので「**調査 → 実装 → 実行 → 評価 → まとめ**」を一通り経験すること。
