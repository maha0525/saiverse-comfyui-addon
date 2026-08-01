# SAIVerse ComfyUI アドオン — ローカル画像生成

ローカル PC で起動している [ComfyUI](https://github.com/comfyanonymous/ComfyUI) サーバーに接続し、SAIVerse のペルソナが画像を生成できるようにするアドオンです。生成された画像は SAIVerse のアイテムとして登録され、ペルソナが所持・参照できます。

クラウド API を使わないため、**生成そのものに API 料金はかかりません**。代わりに、画像生成を実行できる GPU 環境と、ComfyUI・生成モデルのセットアップが必要です。

## 必要なもの

| 何 | 補足 |
|---|---|
| ComfyUI 本体 | 別途インストール。インストール方法は [ComfyUI 公式](https://github.com/comfyanonymous/ComfyUI) を参照 |
| 画像生成モデル | 同梱ワークフロー (Anima) を使う場合は下記の3ファイル |
| GPU | ComfyUI の動作要件に準じます (CPU でも動きますが非常に遅くなります) |

このアドオン自体に追加の Python 依存はありません (SAIVerse 本体の環境だけで動きます)。

## インストール

1. このリポジトリを SAIVerse の `expansion_data/` 配下に clone します。**ディレクトリ名は `saiverse-comfyui-addon` のままにしてください** (ディレクトリ名がツールの名前空間として使われます):

   ```bash
   cd <SAIVerseのルート>/expansion_data
   git clone <このリポジトリのURL> saiverse-comfyui-addon
   ```

2. Playbook を取り込みます:

   ```bash
   python scripts/import_all_playbooks.py --force
   ```

3. SAIVerse を再起動します。起動ログに `saiverse-comfyui-addon__generate_image_local` の登録が出ていれば読み込み成功です。

## ComfyUI 側の準備

### 1. ComfyUI を起動する

ComfyUI を通常どおり起動してください。既定では `http://127.0.0.1:8188` で待ち受けます。

**SAIVerse はこのサーバーが起動していることを前提に動きます。** 起動していない状態でペルソナが画像生成を実行すると、「ComfyUIサーバーに接続できませんでした」というエラーになります (SAIVerse 側が ComfyUI を自動起動することはありません)。

ComfyUI を別のポートや別マシンで動かしている場合は、SAIVerse の `.env` に接続先を設定します:

```
COMFYUI_BASE_URL=http://127.0.0.1:8188
```

### 2. モデルを配置する (同梱ワークフロー Anima を使う場合)

同梱の `tools/comfyui_workflows/Anima.json` は [Anima](https://huggingface.co/circlestone-labs/Anima) (自然言語プロンプト対応のアニメ系モデル) を使うワークフローです。以下の3ファイルを配布元 (Hugging Face など) から入手し、ComfyUI の `models/` 配下に配置してください:

| ファイル | 置き場所 (ComfyUI の models/ 配下) |
|---|---|
| `anima-base-v1.0.safetensors` | `diffusion_models/` (または `unet/`) |
| `qwen_3_06b_base.safetensors` | `text_encoders/` (または `clip/`) |
| `qwen_image_vae.safetensors` | `vae/` |

ファイルが足りない状態で生成を実行すると、`ComfyUI node errors` としてどのノードで失敗したかがエラーメッセージに含まれます。

## 使い方

導入後、ペルソナは Playbook「ローカル画像生成」(`generate_image_local`) を通じて画像を生成できます。ペルソナに「〜の画像を描いて」と頼む、またはペルソナが自発的に生成を選ぶと、次の流れで動きます:

1. LLM が依頼内容から英語の自然言語プロンプトと日本語タイトルを組み立てる
2. ComfyUI にワークフローを送信し、完了までポーリング (最大5分、60秒進捗なしで打ち切り)
3. 生成画像を取得し、SAIVerse のアイテムとして登録

Anima は Stable Diffusion 系のタグ羅列ではなく**自然文の英語プロンプト**で最も良い結果を出します。プロンプトの組み立ては Playbook 内の LLM ノードが面倒を見るので、ユーザー側で意識する必要はありません。

## 別のワークフロー・モデルを使う

`tools/comfyui_workflows/` に ComfyUI の **API フォーマット**でエクスポートしたワークフロー JSON を置くと、ツールの `workflow_file` 引数で指定できます (Playbook 既定は `Anima.json`)。

ワークフロー側の約束事:

- **プロンプト差し込み先**: `CLIPTextEncode` ノードのタイトルに `Positive` / `Negative` を含めておくと、それぞれ positive / negative プロンプトが差し込まれます。タイトルで判別できない場合は、最初に見つかった `CLIPTextEncode` 2つが順に positive / negative として扱われます
- **シード**: `KSampler` ノードの `seed` は実行のたびにランダム値で上書きされます
- **出力**: `SaveImage` 系ノードの出力画像のうち、最初の1枚が SAIVerse に取り込まれます

API フォーマットでのエクスポートは、ComfyUI の設定で Dev mode を有効にすると表示される「Save (API Format)」から行えます。

## トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| 「ComfyUIサーバーに接続できませんでした」 | ComfyUI が起動していない、または `COMFYUI_BASE_URL` が違う。ブラウザで `http://127.0.0.1:8188` を開いて ComfyUI の画面が出るか確認 |
| 「ComfyUI node errors」 | ワークフローが要求するモデルファイルが ComfyUI 側に無いことが大半。エラーに含まれるノード情報で足りないものを特定 |
| 「タイムアウトしました (300秒)」 | 生成に5分以上かかっている。GPU 性能に対してワークフローが重すぎる可能性。steps や解像度を下げたワークフローを試す |
| 「停滞しています (60秒間進捗なし)」 | ComfyUI 側でキューが詰まっているか、モデル読み込みに時間がかかっている。ComfyUI のコンソールログを確認 |
| ツールが登録されない | ディレクトリ名が `saiverse-comfyui-addon` になっているか確認。SAIVerse の起動ログで tools 読み込みのエラーを確認 |

## ライセンス

Apache License 2.0 (LICENSE ファイル参照)。ComfyUI および各生成モデルにはそれぞれのライセンスが適用されます。
