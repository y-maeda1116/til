# MacBook Airでcmux＋Claude Codeを使う手順

## 1. 構成と前提条件

**cmuxをターミナルとして使い、その中でClaude Codeを動かす**構成です。

- **cmux**：ワークスペースや画面分割を管理するターミナルアプリ
- **Claude Code**：コードの調査・修正・テストなどを支援するAIツール

cmuxはlibghosttyを利用した独立したアプリなので、**Ghosttyの別途インストールは不要**です。[[1]](https://cmux.com/docs/getting-started)

### 必要なもの

| 項目 | 条件 |
| --- | --- |
| Mac | Apple Silicon／IntelのMacBook Air |
| OS | **macOS 14以降** |
| 通信 | インターネット接続 |
| アカウント | 対応するClaudeの有料契約、またはClaude Consoleなど |

Claude Code単体はmacOS 13以降に対応していますが、今回はcmuxに合わせてmacOS 14以降が必要です。[[1]](https://cmux.com/docs/getting-started)[[2]](https://code.claude.com/docs/en/setup)

---

## 2. cmuxをインストールする

### 方法A：公式サイトからインストール【おすすめ】

1. [cmux公式サイト](https://cmux.com/)を開く。
2. **Download for Mac**からDMGファイルをダウンロードする。
3. DMGを開き、**cmuxをApplicationsフォルダにドラッグ**する。
4. Applicationsからcmuxを起動する。
5. macOSから確認が表示された場合は、配布元を確認して開く。

左側のワークスペース一覧とターミナル画面が表示されれば完了です。DMG版は自動更新に対応しています。[[1]](https://cmux.com/docs/getting-started)

### 方法B：Homebrewでインストール

すでにHomebrewを使っている場合は、以下でも導入できます。**方法Aと両方実行する必要はありません。**

```bash
brew tap manaflow-ai/cmux
brew install --cask cmux
```

Homebrew経由で更新する場合：

```bash
brew upgrade --cask cmux
```

以上はcmux公式のインストール手順です。[[1]](https://cmux.com/docs/getting-started)

---

## 3. Claude Codeをインストールする

**以降のコマンドは、cmuxのターミナル画面で実行します。**

すでにClaude Codeをインストール済みの場合は、まず以下を確認します。

```bash
claude --version
```

バージョンが表示されれば、再インストールは不要です。

### 未導入の場合

公式のネイティブインストーラーを実行します。

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

インストール後、新しいターミナルペインを開いて確認します。

```bash
claude --version
```

この方式では、**Claude Code自体のためにNode.jsやHomebrewを先に用意する必要はありません**。開発対象のアプリに必要なNode.js・Pythonなどは別途導入します。[[3]](https://code.claude.com/docs/en/quickstart)[[2]](https://code.claude.com/docs/en/setup)

> このコマンドは、公式サイトから取得したスクリプトを実行します。
> 

---

## 4. 練習用フォルダで起動する

まずは、新しいフォルダで試します。

```bash
mkdir -p ~/Projects/claude-practice
cd ~/Projects/claude-practice
claude
```

初回は、画面の案内に従ってブラウザでログインします。[[3]](https://code.claude.com/docs/en/quickstart)

### 課金に関する注意

- Claudeのサブスクリプションと、Claude ConsoleのAPIクレジットは別の利用・課金経路です。
- `ANTHROPIC_API_KEY`が設定されている環境では、APIキー利用の確認が表示されることがあります。
- 意図した契約・課金先で利用しているか確認してください。[[3]](https://code.claude.com/docs/en/quickstart)

### 最初の依頼例

```
このフォルダに、シンプルなToDoリストのHTMLを作りたいです。

まず、作成するファイルと実装方針を説明してください。
私が承認するまでは、ファイルの作成やコマンドの実行はしないでください。
```

---

## 5. cmuxで作業環境を整理する

### おすすめの構成

**1つのプロジェクト／タスクにつき、1つのワークスペース**を基本にします。

```
cmux
├── 練習用
│   └── Claude Code
├── プロジェクトA
│   ├── Claude Code
│   └── テスト・Git操作用ターミナル
└── プロジェクトB
    ├── Claude Code
    └── 開発サーバー用ターミナル
```

Claude Code用と手動操作用でペインを分けると、AIの処理を待ちながらログや差分を確認できます。

### 便利なショートカット

| 操作 | ショートカット |
| --- | --- |
| ワークスペースを選択 | `⌘ + 1〜9` |
| ワークスペース名を変更 | `⌘ + Shift + R` |
| 右側にペインを分割 | `⌘ + D` |
| 下側にペインを分割 | `⌘ + Shift + D` |
| 現在のペインを拡大／元に戻す | `⌘ + Shift + Enter` |
| コマンドパレット | `⌘ + Shift + P` |

ショートカットは設定で変更できます。[[4]](https://cmux.com/docs/keyboard-shortcuts)[[1]](https://cmux.com/docs/getting-started)

### 複数のClaude Codeを同時に使う場合

同じフォルダ・同じブランチを複数のClaude Codeから同時に変更させると、変更が競合する可能性があります。

最初はプロジェクトを分け、同一リポジトリで並行開発する場合は、Git worktreeなどで作業ディレクトリを分離するのがおすすめです。

---

## 6. 通知を確認する

cmuxには、Claude Codeの完了・入力待ちを通知リングや未読バッジで知らせる機能があります。まずは標準状態で動作を確認します。[[5]](https://cmux.com/agents/claude-code)

### cmux内で通知をテスト

```bash
cmux notify --title "動作確認" --body "cmuxの通知テストです"
```

これは**cmuxの通知経路のテスト**です。Claude Codeとの連携も確認するため、短い依頼を出し、別のワークスペースへ移動して完了時の表示を確認します。[[6]](https://cmux.com/docs/notifications)

### 通知が表示されない場合

1. macOSの「システム設定 → 通知」でcmuxの設定を確認する。
2. 集中モードなどで通知が抑制されていないか確認する。
3. cmuxとClaude Codeを更新する。
4. 必要に応じて[公式の通知・Claude Code hooks設定](https://cmux.com/docs/notifications)を確認する。

> 既存の`~/.claude/settings.json`がある場合は、通知設定を追加する際にファイル全体を上書きしないよう注意してください。
> 

---

## 7. Claude Codeの基本操作

### cmuxのシェルで使うコマンド

```bash
# 起動
claude

# 現在のフォルダの直近の会話を継続
claude -c

# 過去の会話を選んで再開
claude -r

# 動作・更新状況を診断
claude doctor
```

### Claude Codeの起動後に使うコマンド

| コマンド | 用途 |
| --- | --- |
| `/help` | 利用可能な操作を確認 |
| `/login` | ログイン・再認証 |
| `/clear` | 現在の会話をクリア |
| `/exit` | Claude Codeを終了 |

以上はClaude Code公式の基本操作です。[[3]](https://code.claude.com/docs/en/quickstart)[[2]](https://code.claude.com/docs/en/setup)

複数行の指示を書く際、`Shift + Enter`が改行にならない場合は、**`Ctrl + J`**で改行できます。[[7]](https://code.claude.com/docs/en/terminal-config)

### 既存プロジェクトでの依頼例

```
このプロジェクトの構成と起動方法を調査してください。

- 最初は読み取り・調査のみ行ってください
- ファイル変更や依存関係の追加は行わないでください
- 調査結果と、次に必要な作業を箇条書きでまとめてください
```

---

## 8. トラブルシューティング

### `claude: command not found`になる

まず、新しいペインまたはcmuxの再起動後に確認します。

```bash
claude --version
```

それでも見つからない場合：

```bash
~/.local/bin/claude --version
```

これで動くなら、PATHの設定が原因と考えられます。ネイティブ版の起動ファイルは通常、`~/.local/bin/claude`にあります。[[2]](https://code.claude.com/docs/en/setup)

標準的なzsh環境では、`~/.zshrc`に以下を追加します。すでに同じ設定がある場合は重複追加しません。

```bash
export PATH="$HOME/.local/bin:$PATH"
```

設定を反映します。

```bash
source ~/.zshrc
claude --version
```

### cmux以外で`cmux`コマンドを使いたい

Claude Codeを動かすだけなら、この設定は不要です。

別のターミナルからもcmuxを操作する場合は、cmuxでコマンドパレットを開き、以下を実行します。

```
Shell Command: Install 'cmux' in PATH
```

cmux内では、通常は追加設定なしで`cmux`コマンドを使用できます。[[1]](https://cmux.com/docs/getting-started)

### アプリを閉じても処理は続く？

**cmuxを閉じても、すべての処理がそのまま続くわけではありません。**

cmuxはレイアウトやメタデータを復元しますが、任意の実行中プロセスの状態を保存するわけではありません。対応するエージェントの会話再開には、フックなどの条件があります。[[1]](https://cmux.com/docs/getting-started)

処理中は終了を避け、再起動後は作業フォルダで以下を使って会話を再開します。

```bash
claude -c
```

または：

```bash
claude -r
```

ただし、**会話の再開と実行中コマンドの継続は別**です。[[3]](https://code.claude.com/docs/en/quickstart)

---

## 9. セットアップ完了チェックリスト

- [ ]  macOS 14以降である
- [ ]  cmuxが起動する
- [ ]  `claude --version`でバージョンが表示される
- [ ]  意図したアカウント・課金先でログインしている
- [ ]  練習用フォルダで依頼を試した
- [ ]  画面分割とワークスペース切り替えを確認した
- [ ]  通知の動作を確認した
- [ ]  `claude -c`で会話を再開できることを確認した

**まずは「cmuxを導入 → Claude Codeを導入 → 練習用フォルダで起動」の3段階で始め、通知や並行作業は動作確認後に広げるのがおすすめです。**
