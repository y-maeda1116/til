2026/03/11時点のメモ
# iPhoneからClaude Codeを使うための環境構築メモ

## 背景・要件

- 自宅のWindows機でClaude Code（[Z.ai](http://Z.ai)経由GLMバックエンド）を利用中
- 外出先のiPhoneからもClaude Codeを操作したい
- 現在はChrome リモートデスクトップを使っているが、入力がしにくいのでやめたい
- 主な用途は個人の趣味・学習
- 自宅ネットワークは動的IP、ポート開放は未確認
- iPhoneはテキストベース（ターミナル）操作を希望
- Windows機の常時起動はOK

---

## 接続方式の基本構成

どの案でも共通するスタック：

| 要素 | 選定 |
| --- | --- |
| **VPN / トンネル** | Tailscale（無料・ポート開放不要・NAT越え自動） |
| **iOSターミナル** | Termius（無料版で十分）or Blink Shell（Mosh対応、買い切り約¥2,800） |
| **セッション維持** | tmux（SSH切断してもセッション継続） |

### 使い方の流れ

```
iPhone → Tailscaleで自宅ネットワークに接続
       → SSHアプリでサーバー機に接続
       → tmux attach → Claude Code操作
```

---

## 予算別プラン

### プランA：追加費用ほぼゼロ（既存Windows機を活用）

- Windows機でWSL2（Ubuntu）をインストール
- WSL2内にClaude Code環境を構築
- OpenSSHサーバーを有効化、Tailscaleをインストール
- **メリット**: 追加ハードなし、すぐ始められる
- **デメリット**: WSL2のSSH/tmux環境にやや癖がある

### プランB：1〜3万円（ミニLinux機を追加）

- Raspberry Pi 5 (8GB: 約1.5万円) や N100ミニPC にUbuntuを導入
- 低消費電力で常時起動OK、Linux環境はClaude Codeとの相性◎

### プランC：約8〜10万円（Mac mini M4）★最終的なおすすめ

- Mac mini M4 (24GB/256GB) + 外付けSSD
- 詳細は後述

### プランD：月額課金型（クラウドVPS）

- 低価格VPS（月500〜2,000円）にClaude Codeを入れてSSH接続
- 自宅の常時起動不要だが、ランニングコストが継続的に発生

---

## N100ミニPCの現状（2026年3月時点）

### メモリ価格の高騰

- **DDR5**: 2025年7月比で約4.4倍に高騰。32GB DDR5-6000キットが$80→$432前後
- **DDR4**: 生産縮小でDDR5より値上がりペースが速い局面も
- **原因**: AIデータセンター需要の爆増 + メーカーのHBM/DDR5への生産シフト + 供給不足
- **見通し**: メモリ不足は2027年Q4まで続く見込み

### 主要メーカーの価格（16GB/512GB）

| メーカー | モデル | 参考価格 | 備考 |
| --- | --- | --- | --- |
| Beelink | Mini S12 Pro | 約$500〜680（7.5〜10万円） | 以前は$160〜200 |
| MinisForum | UN100L | 約$200前後（3万円前後） | LPDDR5。値上げ予告あり |
| MinisForum | UN100P | 約$200〜220（3〜3.3万円） | DDR4、2.5GbE LAN付き |
| GMKtec | NucBox G3 | 約$260〜310（4〜4.7万円） | G2は在庫切れ |

> ⚠️ MinisForumは「DRAM・SSD価格高騰により全モデル値上げ」を公式発表済み。
> 

---

## Mac mini M4 を推す理由

N100ミニPCが8万円前後まで高騰している現在、Mac mini M4（$499〜）はコスパで逆転。

| 比較項目 | N100ミニPC（高騰後） | Mac mini M4 |
| --- | --- | --- |
| 価格 | 約5〜10万円 | $499〜（約7.5万円〜） |
| メモリ | 16GB DDR4/DDR5 | 16GB〜 統合メモリ |
| CPU性能 | N100（4コア、低性能） | M4（5〜10倍高性能） |
| 省電力 | 低め | アイドル約7W |
| ローカルLLM | 厳しい | 20Bクラスが動く |
| リセールバリュー | ほぼなし | 高い |

### その他のMac miniの利点

- macOSのターミナル環境（Homebrew/tmux/SSH/Docker）がネイティブで快適、WSL2不要
- Apple Siliconのエコシステム（Ollama, llama.cpp, MLX等が最適化済み）
- macOSのアップデートが5〜7年は見込める

### WindowsからMac miniの操作

| やりたいこと | 接続方法 |
| --- | --- |
| Claude Code / ターミナル操作 | SSH + tmux（メイン） |
| GUIアプリの操作 | VNC（macOS標準の「画面共有」） |
| ファイル転送 | scp / SFTP |

### 初期セットアップ

- **初回起動のみ** モニター（テレビのHDMI等）とキーボードが必要
- Apple ID設定 → Wi-Fi接続 → 「画面共有」「リモートログイン」をON
- その後はヘッドレス（モニターなし）で運用可能
- VNC接続時にモニターなしで解像度がおかしい場合は「HDMIダミープラグ」（数百円）で解決

---

## Claude Codeの必要スペック（公式）

| 項目 | 要件 |
| --- | --- |
| OS | macOS 13.0+ / Windows 10+ / Ubuntu 20.04+ 等 |
| **RAM** | **4GB以上** |
| ネットワーク | インターネット接続必須 |
| シェル | Bash, Zsh, PowerShell, CMD |

> Claude Codeはローカルで推論せず、API経由でクラウド側に処理を投げるため、PC側のスペック要件は非常に軽い。
> 

---

## ローカルLLMのコーディング能力検証

> 参考記事: [Qwen3.5の中規模モデル(122B/35B/27B/9B)をコーディングエージェントで試してみる](https://nowokay.hatenablog.com/entry/2026/03/08/205807)（きしだのHatena, 2026/03/08）
> 

### 検証タスク

- Java 25 + Java SE標準ライブラリのみ（フレームワークなし）でWeb版TODO管理アプリを作成

### モデル別の結果

#### 🧠 モデル別コーディング能力

| モデル | アクティブ | 量子化 | メモリ使用量 | SWE | 実用度 |
| --- | --- | --- | --- | --- | --- |
| Qwen3.5 122B | 10B | Q4_K_XL | 79〜89GB | 72.0 | ✅ 実用レベル |
| Qwen3.5 35B | 3B | Q4_K_XL | 24〜35GB | 69.2 | ⚠️ 関係ない修正をしがち |
| Qwen3.5 27B | Dense | Q4_K_XL | 20〜74GB | 72.4 | △ ヒントがあれば修正可能 |
| Qwen3.5 9B | Dense | Q4_K_XL | 8〜42GB | — | ❌ コンパイルすら通らず |
| Qwen3-Coder-Next 80B | 3B | Q3_K_XL | 35〜48GB | 70.6 | ⚠️ 35Bと同程度 |
| **gpt-oss-20b** | 3.6B | MXFP4 | **12〜16GB** | 60.7 | **✅ かなり賢い** |

※ メモリ使用量はコンテキスト長の設定（4000〜最大）による変動（LM Studio見積もり）

#### 🚀 122B

- HTTPサーバーはすぐ動作、POST処理の修正で正常動作
- ただし修正時に別のバグを生むことがあり、短絡的修正の傾向

#### 🔧 27B（Denseモデル）

- 最初は「不可能」と判断
- 修正で既存機能を壊すことが多い
- ヒントを与えると最終的に動作（人間のサポート必要）

#### ⚠️ 35B

- 画面表示は成功するが、HTML出力が途中で切れてボタンが動かない
- 指摘しても問題と関係ない修正を試みる
- 最終的に完全修正には至らず

#### ❌ 9B

- 基本的な構文理解が弱く、コンパイル可能なコードを書けない

#### 🤖 Qwen3-Coder-Next 80B

- RTX 4060 Ti 16GBで動作検証
- HTML出力途中切れ問題でボタンが動かず、修正方向がずれて失敗

#### 🏆 gpt-oss-20b

- ベンチマーク（SWE: 60.7）はQwenより低い
- しかし **バグ切り分けと安全な修正能力が高い**
- 27Bが直せなかった問題を一発修正
- エージェント対応は未成熟だが、修正担当としては最も信頼できる

### 🧩 Qwen3.5シリーズ共通の傾向

- 難しいコードやバグ特定は苦手
- モデルサイズ相応にコーディング能力は向上
- OpenCodeでのエージェント動作は全モデル問題なし
- コーディング以外の用途ではどのモデルも比較的高性能で実用的

### 📈 コーディングエージェントで重要な能力

> SWEベンチマークで計られるような難しいロジックが書ける能力より、**普通のコードで普通に出てくる普通のバグをちゃんと切り分けて、他に影響を与えず修正する能力**が大事。
> 

### 📌 実用的な使い方（役割分担）

- **27B / 35B** → 初期コード生成
- **gpt-oss-20b** → バグ修正担当
- この組み合わせが最も安定

### 📌 総合結論

- **35B以下単体でのコーディングエージェント運用は厳しい**
- ただし初期コード生成には使える
- **gpt-oss-20bと組み合わせると実用性が高い**

---

## Mac miniでのローカルLLM運用（gpt-oss-20bを中心に）

### gpt-oss-20bのMac mini対応状況

| メモリ | gpt-oss-20b (12〜16GB) | 運用余裕 |
| --- | --- | --- |
| 16GB | ⚠️ OS込みでギリギリ、スワップ発生の可能性 | 低い |
| **24GB** | **✅ OS(4〜5GB) + モデル(12〜16GB) = 余裕あり** | **十分** |
| 32GB | ✅ 余裕大、27Bも短コンテキストで動作可 | 高いが過剰投資気味 |

### 24GBでの実用的な運用

```
【コーディングエージェント（メイン）】
  └ Z.ai / Claude Code（API経由）→ スペック不問、高品質

【ローカルLLM補助（サブ）】
  └ gpt-oss-20b（Ollama or LM Studio）
    ├ バグ修正の補助（記事で高評価）
    ├ チャット・要約・翻訳
    └ オフライン時の簡易利用

【応用パターン】
  └ 27B/35Bで初期コード生成 → gpt-oss-20bでバグ修正
    （24GBなら27Bを短コンテキストで動かし切り替え可能）
```

### 現実的なおすすめ

- **初期コード生成もAPI（[Z.ai](http://Z.ai)）に任せて、ローカルのgpt-oss-20bはちょっとした修正やオフライン用に使う**のが最もストレスのない運用

---

## Mac mini メモリ選びの最終結論

| メモリ | 追加費用 | 評価 |
| --- | --- | --- |
| 16GB（標準） | なし | Claude Code（API型）には十分。ローカルLLMは20Bがギリギリ |
| **24GB** | **+3万円** | **★おすすめ。gpt-oss-20bが快適に動く。コスパ最良（+50%メモリで+3万円）** |
| 32GB | +6万円 | 27B/35Bが動くが、コーディング用途での実用性は微妙。過剰投資 |

> ⚠️ Mac miniのメモリは購入後に増設不可（オンボード）。迷うなら上を選ぶのが鉄則。
> 

### 判断ポイント

- コーディングエージェントをまともにローカルで動かすには122B（メモリ80GB超）が必要 → Mac mini 32GBでも不足
- ローカルLLMのためだけに32GBに+6万円は過剰
- **24GB（+3万円）がベストバランス**

---

## おすすめ購入構成

| 項目 | 選択 | 費用 |
| --- | --- | --- |
| Mac mini M4 | **24GB** / 256GB | 約10.5万円 |
| 外付けUSB-C SSD 1TB | Samsung T7等 | 約5,000〜8,000円 |
| Tailscale | 無料プラン | 0円 |
| Termius（iOS） | 無料版 | 0円 |
| **合計** |  | **約11〜11.5万円** |
Tailscale（timescaleではなく）ですね！追記用のセクションです：

## 付録：Mac購入前のWindows + Tailscale + iPhoneセットアップ手順

既存のWindows機を使い、iPhoneからSSHでClaude Codeを操作する暫定環境の構築手順。

### 1. Windows側：WSL2のインストール

```powershell
# PowerShell（管理者）で実行
wsl --install
```

- 再起動後、Ubuntuが自動で起動しユーザー名・パスワードを設定
- WSL2内でClaude Code環境を構築しておく

### 2. Windows側：WSL2にtmuxをインストール

```bash
# WSL2（Ubuntu）内で実行
sudo apt update
sudo apt install tmux -y
```

### 3. Windows側：OpenSSHサーバーを有効化

1. **設定** → **アプリ** → **オプション機能** → **機能を追加** → **OpenSSH サーバー** をインストール
2. **サービス**（services.msc）を開く
3. **OpenSSH SSH Server** を右クリック → **プロパティ**
    - スタートアップの種類を **「自動」** に変更
    - **「開始」** をクリック

### 4. Windows側：Tailscaleのインストール

1. https://tailscale.com/download からWindowsクライアントをダウンロード・インストール
2. Tailscaleアカウントを作成してログイン（Google/Microsoft/GitHub等のSSO）
3. タスクトレイにTailscaleアイコンが表示され、接続状態になればOK
4. Tailscale管理画面（https://login.tailscale.com/admin/machines）で自分のWindows機の名前とIPを確認

### 5. iPhone側：Tailscaleのインストール

1. App Storeから **Tailscale** をインストール
2. Windows側と **同じアカウント** でログイン
3. VPNの設定許可を求められるので許可
4. 接続すると、Windows機が一覧に表示される

### 6. iPhone側：SSHアプリのインストールと設定

**Termius（無料版）の場合：**

1. App Storeから **Termius** をインストール
2. **Hosts** → **+** → **New Host** で以下を設定：
    - **Hostname**: Windows機のTailscale名 or `100.x.x.x`（Tailscale IP）
    - **Port**: `22`
    - **Username**: Windowsのユーザー名
    - **Password**: Windowsのパスワード

### 7. 接続と操作の流れ

```bash
# iPhoneのTermiusからSSH接続後

# WSL2に入る
wsl

# tmuxセッションを作成（初回）
tmux new -s claude

# Claude Codeを起動
claude

# --- 外出先から再接続する場合 ---

# SSH接続後
wsl

# 既存のtmuxセッションに再接続
tmux attach -t claude
```

### 8. Tips

- **tmuxのセッションはSSH切断後も維持される**。電車で圏外→復帰してもtmux attachで続きから
- Tailscaleは **iPhone側でVPNをONにしないと接続できない**。接続できない場合はまずTailscaleアプリを確認
- Windows側のファイアウォールでSSH（ポート22）がブロックされる場合がある。その場合：
    
    ```powershell
    # PowerShell（管理者）で実行
    New-NetFirewallRule -Name "OpenSSH-Server" -DisplayName "OpenSSH Server (sshd)" -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
    ```
    
- WSL2内で直接SSHサーバーを立てる方法もあるが、Windows側のOpenSSHから `wsl` コマンドで入る方がシンプルで安定

## 付録：claude-code-starter-kit の導入

> 参考: https://github.com/cloudnative-co/claude-code-starter-kit
> 

Cloud Native社が公開しているClaude Code環境のセットアップキット。

インタラクティブウィザードで、[CLAUDE.md](http://CLAUDE.md)・MCP・hooks等を含む開発環境をワンコマンドで構築できる。

### 前提条件

- Claude Codeがインストール済みであること
- macOS or Linux（WSL2含む）
- Git, Node.js がインストール済み

### Mac mini（macOS）での導入手順

```bash
# 1. Homebrewのインストール（未導入の場合）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. 必要なツールのインストール
brew install node git

# 3. Claude Codeのインストール
npm install -g @anthropic-ai/claude-code

# 4. Claude Codeの初回認証
claude
# → ブラウザが開き、認証フローを完了する
# → Z.aiを使う場合は別途設定が必要

# 5. プロジェクトディレクトリの作成・移動
mkdir -p ~/projects/my-project
cd ~/projects/my-project

# 6. claude-code-starter-kit のインストール
curl -fsSL https://raw.githubusercontent.com/cloudnative-co/claude-code-starter-kit/main/install.sh | bash

# 7. ウィザードが起動するので、対話形式で設定
# → プロファイル選択（用途に応じて）
# → CLAUDE.md、MCP設定、hooks等が自動生成される
```

### WSL2（Windows暫定環境）での導入手順

```bash
# WSL2(Ubuntu)にSSH接続後

# 1. 必要なツールのインストール
sudo apt update
sudo apt install -y nodejs npm git curl

# 2. Node.jsのバージョンが古い場合はnvmで最新を入れる
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
source ~/.bashrc
nvm install --lts

# 3. Claude Codeのインストール
npm install -g @anthropic-ai/claude-code

# 4. Claude Codeの初回認証
claude

# 5. プロジェクトディレクトリの作成・移動
mkdir -p ~/projects/my-project
cd ~/projects/my-project

# 6. claude-code-starter-kit のインストール
curl -fsSL https://raw.githubusercontent.com/cloudnative-co/claude-code-starter-kit/main/install.sh | bash

# 7. ウィザードに従って設定
```

### 生成されるファイル構成（想定）

```
~/projects/my-project/
├── CLAUDE.md              # プロジェクト指示・メモリファイル
├── .claude/
│   ├── settings.json      # Claude Code設定
│   ├── skills/            # スキル定義
│   └── hooks/             # フック設定
└── ...
```

### Tips

- **プロジェクトごとにインストール**する想定。グローバルではなく各プロジェクトのルートで実行
- [CLAUDE.md](http://CLAUDE.md)はプロジェクト固有の指示をClaude Codeに記憶させるファイル。カスタマイズ推奨
- starter-kitの設定は `.claude/` 以下に格納されるため、**Gitでチーム共有も可能**
