# WindowsからMac miniへSSHで繋いでClaude Codeを使うときの構成ガイド

Windows PCからSSHでMac miniに接続し、Claude Codeを複数リポジトリで並行運用する場合の推奨構成をまとめます。Windows Updateによる再起動でSSHが切断されても作業が継続できる構成を主眼にしています。

> **想定環境**
> - クライアント: Windows PC（Windows Updateにより定期的に再起動が発生）
> - サーバ: Mac mini（常時稼働、実作業マシン）
> - 用途: Claude Codeを5リポジトリ程度で並行起動。共通の「セキュリティベース」リポジトリを他リポジトリから参照。
> - エディタはあまり使わない（ターミナル中心）。VS Codeはインストール済み。

---

## 目次

- [課題](#課題)
- [推奨アーキテクチャ](#推奨アーキテクチャ)
- [1. Mac mini側のセッション永続化(tmux)](#1-mac-mini側のセッション永続化tmux)
- [2. tmuxセッション設計](#2-tmuxセッション設計)
- [3. 起動スクリプトで1コマンド復元](#3-起動スクリプトで1コマンド復元)
- [4. セキュリティリポジトリの横断参照](#4-セキュリティリポジトリの横断参照)
- [5. SSHクライアント側の設定](#5-sshクライアント側の設定)
- [6. moshで切断耐性をさらに上げる](#6-moshで切断耐性をさらに上げる)
- [7. Windows Updateを制御する](#7-windows-updateを制御する)
- [8. VS Code Remote-SSHを「横断参照ビューア」として併用](#8-vs-code-remote-sshを横断参照ビューアとして併用)
- [9. リモートデスクトップという選択肢](#9-リモートデスクトップという選択肢)
- [10. リモートデスクトップツールの比較・選定](#10-リモートデスクトップツールの比較選定)
- [導入の優先順位](#導入の優先順位)
- [構成チェックリスト](#構成チェックリスト)

---

## 課題

- Windows UpdateでPCが再起動するとSSHセッションが切断され、Mac mini側で動いていたClaude Codeのターミナルタブがすべて失われる。
- 5リポジトリ分の作業環境を毎回立ち上げ直すのが負担。
- 共通の `security` リポジトリを他リポジトリから参照する横断的な使い方をしているため、リポジトリ間を行き来する操作も発生する。

## 推奨アーキテクチャ

​
[Windows]
├─ Windows Terminal (or mosh client) ──SSH/mosh──┐
└─ VS Code (Remote-SSH)         ─────────────────┤
▼
[Mac mini]
├─ tmux session: work
│    ├─ window: security
│    ├─ window: repo1
│    ├─ window: repo2
│    ├─ window: repo3
│    ├─ window: repo4
│    └─ window: repo5
└─ Claude Code 常駐

- **Mac側でtmuxにすべての状態を持たせる**ことが本質的な解。
- Windows側は「Macに繋ぐ端末」と捉え、いつ落ちても良い状態にしておく。

---

## 1. Mac mini側のセッション永続化(tmux)

SSHが切れてもプロセスが死なないよう、Claude Codeは必ずtmux内で実行します。

​
インストール
brew install tmux
初回: セッション作成
tmux new -s work
切断するときは Ctrl+b → d (デタッチ)
次回接続時に再アタッチ
tmux attach -t work

### 推奨 `~/.tmux.conf`

​
set -g mouse on              # マウスでpane/window操作
set -g history-limit 100000  # スクロールバック拡張
set -g base-index 1          # window番号を1始まりに
setw -g pane-base-index 1
set -g status-interval 5
set -g default-terminal "screen-256color"

---

## 2. tmuxセッション設計

「ターミナルのタブ = tmuxのwindow」と読み替えるのが基本です。

| 旧概念 | tmuxでの対応 | 役割 |
|---|---|---|
| ターミナルアプリ | tmuxサーバ(常駐) | 全体保持 |
| ウィンドウ全体 | session | プロジェクト単位 |
| タブ | window | リポジトリ単位 |
| 画面分割 | pane | 同一リポジトリ内でclaude/git/logを並べる |

### セッション構成例

​
session: work
├─ window 0: security   (~/project/security)   ← 共通参照リポジトリ
├─ window 1: repo1      (~/project/repo1)
├─ window 2: repo2      (~/project/repo2)
├─ window 3: repo3      (~/project/repo3)
├─ window 4: repo4      (~/project/repo4)
└─ window 5: repo5      (~/project/repo5)

### よく使うキーバインド

| 操作 | キー |
|---|---|
| デタッチ | `Ctrl+b → d` |
| 新規window | `Ctrl+b → c` |
| window切替 | `Ctrl+b → 数字` / `Ctrl+b → n` / `Ctrl+b → p` |
| window一覧(検索) | `Ctrl+b → w` |
| window名変更 | `Ctrl+b → ,` |
| 縦分割 | `Ctrl+b → %` |
| 横分割 | `Ctrl+b → "` |
| pane移動 | `Ctrl+b → 矢印` |
| pane最大化 | `Ctrl+b → z` |
| session一覧 | `Ctrl+b → s` |

---

## 3. 起動スクリプトで1コマンド復元

5リポジトリを毎回手で立ち上げるのは現実的でないため、構成をスクリプト化します。

### シンプル版 `~/bin/work.sh`

​
#!/bin/bash
S=work
tmux has-session -t $S 2>/dev/null
if [ $? != 0 ]; then
tmux new-session -d -s $S -n security -c ~/project/security
tmux new-window  -t $S    -n repo1    -c ~/project/repo1
tmux new-window  -t $S    -n repo2    -c ~/project/repo2
tmux new-window  -t $S    -n repo3    -c ~/project/repo3
tmux new-window  -t $S    -n repo4    -c ~/project/repo4
tmux new-window  -t $S    -n repo5    -c ~/project/repo5
各windowでclaudeを自動起動したい場合は有効化
for w in security repo1 repo2 repo3 repo4 repo5; do
tmux send-keys -t $S:$w 'claude' C-m
done
fi
tmux attach -t $S
​
chmod +x ~/bin/work.sh

SSH接続後 `work.sh` の1コマンドで完全復元できます。

### tmuxinator版 `~/.tmuxinator/work.yml`

​
name: work
windows:
security:
root: ~/project/security
panes:
claude
repo1:
root: ~/project/repo1
panes:
claude
repo2:
root: ~/project/repo2
panes:
claude
repo3:
root: ~/project/repo3
panes:
claude
repo4:
root: ~/project/repo4
panes:
claude
repo5:
root: ~/project/repo5
panes:
claude

起動: `mux start work`

---

## 4. セキュリティリポジトリの横断参照

共通の `security` リポジトリを他リポジトリから参照する場合、以下のいずれか(または併用)で快適性が大きく上がります。

### (a) シンボリックリンク方式

​
cd ~/project/repo1
ln -s ~/project/security ./security-ref

各リポジトリの中から `./security-ref/...` で参照可能。`.gitignore` に `security-ref` を追加すること。

### (b) 親ディレクトリでclaudeを起動するwindowを追加

横断調査・横断変更が必要なときだけ使う「全体ビュー」用windowを別途用意。

​
tmux new-window -t work -n all -c ~/project
tmux send-keys -t work:all 'claude' C-m

普段は各リポジトリのclaudeを使い、横断調査時のみ `all` windowに切り替える。

### (c) pane分割でsecurityを常時並べる

特定リポジトリでsecurityの参照頻度が高い場合、そのwindow内をpane分割。

​
window: repo1
+--------------------+--------------------+
claude (repo1)
less/git (security)
+--------------------+--------------------+

---

## 5. SSHクライアント側の設定

### Windows側 `~/.ssh/config`

​
Host macmini
HostName <Mac miniのIP / ホスト名>
User <username>
ServerAliveInterval 30
ServerAliveCountMax 6
TCPKeepAlive yes

### Mac側 `/etc/ssh/sshd_config`

​
ClientAliveInterval 30
ClientAliveCountMax 10

短時間のネットワーク瞬断であれば、これだけで耐えられる場合もあります。

---

## 6. moshで切断耐性をさらに上げる

`mosh`(Mobile Shell)は、ネットワーク瞬断・IP変更があってもセッションを維持できるSSHの代替です。Windows Update程度の短時間切断なら**自動復旧**します。

​
Mac側
brew install mosh
Windows側
- WSLにmosh-clientをインストール
- もしくはMobaXtermなどのクライアントを利用
接続例(tmuxと併用)
mosh user@macmini -- tmux attach -t work

> **ファイアウォール:** moshはUDP 60000〜61000を使う。Mac側で許可が必要。

---

## 7. Windows Updateを制御する

Mac側の対策を入れた上で、Windows側でも以下を設定すると効果的です。

- **アクティブ時間の設定**: `設定 → Windows Update → アクティブ時間` を業務時間帯に最大18時間まで拡張。
- **更新の一時停止**: 最大5週間まで延期可能。
- **再起動オプション**: 「更新可能になったらすぐに再起動する」をOFF。
- **グループポリシー(Pro以上)**: 「自動更新を構成する」で「ダウンロードと自動インストールを通知」に変更。
- 長時間作業前は `services.msc` で **Windows Updateサービスを手動停止**(自己責任、業務PCのポリシーに従うこと)。

---

## 8. VS Code Remote-SSHを「横断参照ビューア」として併用

エディタとして本格的に使わなくても、**横断grep・ファイル俯瞰用**としてVS Codeを常駐させると効率が上がります。

1. WindowsのVS Codeに拡張機能 **Remote - SSH** を入れる。
2. Mac miniに接続。
3. `work.code-workspace` を作ってsecurity + repo1〜5をMulti-rootとして登録。

​
{
"folders": [
{ "path": "/Users/<user>/project/security" },
{ "path": "/Users/<user>/project/repo1" },
{ "path": "/Users/<user>/project/repo2" },
{ "path": "/Users/<user>/project/repo3" },
{ "path": "/Users/<user>/project/repo4" },
{ "path": "/Users/<user>/project/repo5" }
]
}

- `Ctrl+Shift+F` で **6リポジトリ横断grep**
- `Ctrl+P` で全リポジトリのファイルにジャンプ
- 編集はターミナル(Claude Code)中心、VS Codeは読む専用
- VS Code側が落ちてもMacのtmux/claudeは無傷

---

## 9. リモートデスクトップという選択肢

ここまでの構成でWindows Update問題は実用上ほぼ解消できますが、**さらに根本的に切り離したい**場合は「Mac miniをリモートデスクトップで操作する」アプローチがあります。

### メリット

- MacのGUI(ターミナル、エディタ、ブラウザ)をそのまま使える。
- Windows再起動 = ただの画面切断。Mac側のセッションは完全に独立。
- ターミナルアプリのタブ管理など、これまでの感覚を維持しやすい。

### デメリット

- ネットワーク帯域と遅延が必要。
- キーボードショートカット(特にCommand/Control)の差異に注意。
- SSH+tmuxほど軽快ではない。

---

## 10. リモートデスクトップツールの比較・選定

Mac miniをターゲットにする場合の主要候補を整理します。

| ツール | プロトコル | 特徴 | 体感速度 | コスト | 向き |
|---|---|---|---|---|---|
| **macOS画面共有(Screen Sharing)** | VNC(独自拡張) | macOS標準。設定 → 一般 → 共有 → 画面共有をオンにするだけ | 標準的 | 無料 | LAN内 / 社内ネットワーク |
| **Jump Desktop** | Fluid (独自) / RDP / VNC | Mac向け定番。Fluidプロトコルが高速・低遅延。ファイル転送・マルチモニタ対応 | 速い | 有料(買い切り or サブスク) | LAN/WAN両対応、品質重視 |
| **Parsec** | 独自(低遅延ストリーミング) | 元はゲーム向け。フレームレート・遅延ともに最良クラス。MacはホストにできるがApple Siliconは制約あり、要確認 | 非常に速い | 無料(Team版は有料) | 高解像度・滑らかさ最優先 |
| **AnyDesk** | 独自(DeskRT) | 軽量、クロスプラットフォーム、業務利用実績多数。社内ポリシーで承認されているケースが多い | 速い | 個人無料 / 商用有料 | 業務利用、サポート用途 |
| **TeamViewer** | 独自 | 古参で導入容易。ただし商用判定が厳しめ | 標準的 | 個人無料 / 商用有料 | 既存導入があれば |
| **Chrome Remote Desktop** | 独自(WebRTC) | ブラウザだけで動く。導入が極めて簡単 | 標準的 | 無料 | お手軽用途、緊急避難 |
| **NoMachine** | NXプロトコル | 高速。Macクライアントは安定。設定はやや上級者向け | 速い | 個人無料 | 技術者向け |
| **Microsoft Remote Desktop(RDP)** | RDP | Macは**ホストにはできない**(クライアントのみ) | — | — | Mac miniを操作する用途では非対応 |

### 選定の考え方

#### 同一LAN(社内ネットワーク)で完結する場合

→ **macOS画面共有(VNC)** で十分なケースが多い。追加コスト・追加ソフト不要。
操作感に不満があれば **Jump Desktop**(Fluid)へアップグレード。

#### 自宅 ⇄ 社内などWAN越え or 高遅延が気になる場合

→ **Jump Desktop** または **Parsec**。
- **Jump Desktop**: 業務利用との親和性、ファイル転送、マルチモニタなど機能面が充実。
- **Parsec**: とにかく滑らかさ・低遅延が欲しいなら最有力。ただし業務PCで導入可否を要確認。

#### 業務PCで導入ハードルが低いものを選びたい

→ **AnyDesk** が無難。商用利用ポリシーが明確で、IT部門の承認が下りやすい。

#### とにかく今すぐ試したい

→ **Chrome Remote Desktop**。Googleアカウントとブラウザだけで完結。常用には不向きだが、評価には十分。

### ネットワーク・セキュリティ上の注意

- **VPN経由が原則**: インターネット越しの場合、VPN越しに接続する構成が安全。直接ポート公開は避ける。
- **強力なログインパスワード + アカウントロック**: 画面共有を有効化する場合は必須。
- **2要素認証対応のツールを選ぶ**: Jump Desktop / AnyDesk / Parsecなどは2FAに対応。
- **社内ポリシー確認**: 業務利用の場合は情報システム部門の許可を得ること。
- **macOSのアクセシビリティ・画面収録・入力監視**: 各種ツールに権限付与が必要。導入時に設定 → プライバシーとセキュリティで許可。

### おすすめ構成例

| シナリオ | 推奨 |
|---|---|
| 社内LAN・コスト最小 | macOS画面共有(Screen Sharing) |
| 社内LAN・操作感重視 | Jump Desktop(Fluid) |
| 在宅 ⇄ 社内・滑らかさ最優先 | Parsec(導入可否をIT部門に確認) |
| 業務利用・IT部門承認を取りやすい | AnyDesk |
| ターミナルだけで足りる場合 | リモートデスクトップは使わず **SSH + tmux + mosh** で十分 |

---

## 導入の優先順位

1. **Mac miniにtmuxを導入し、Claude Codeをtmux内で起動する**(最重要)
2. **`work.sh` で起動を1コマンド化**
3. **`~/.tmux.conf` にmouse on / history-limit設定**
4. **SSH configに `ServerAliveInterval 30` を追加**
5. **(任意)moshを導入して切断耐性を強化**
6. **(任意)VS Code Remote-SSHでMulti-root Workspaceを作成、横断参照ビューアとして併用**
7. **(任意)リモートデスクトップを評価**(SSH+tmuxで不足を感じた場合のみ)

---

## 構成チェックリスト

- [ ] Mac miniにtmuxインストール
- [ ] `~/.tmux.conf` 作成(mouse on / history-limit / base-index)
- [ ] `~/bin/work.sh` または `tmuxinator` で起動を自動化
- [ ] securityリポジトリの参照方式を決定(symlink / 親dir claude / pane分割)
- [ ] Windows側 `~/.ssh/config` にKeepAlive設定
- [ ] Mac側 `sshd_config` にClientAlive設定
- [ ] Windows Updateのアクティブ時間・再起動オプション見直し
- [ ] (任意)mosh導入
- [ ] (任意)VS Code Remote-SSH + Multi-root Workspace
- [ ] (任意)リモートデスクトップツールを比較・選定

---

## ライセンス

本ドキュメントは社内・個人での自由な再配布・改変を想定しています。必要に応じてMIT Licenseなどを付
