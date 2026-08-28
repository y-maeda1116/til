# 2016 MacBook ProをOCLPで新しいmacOSへ更新する手順

> **OCLP**は「OpenCore Legacy Patcher」の略です。  
> Appleの正式サポート外となったMacに、新しいmacOSをインストールするための非公式ツールです。
>
> この作業には、起動不能・データ消失・一部機能の不具合といったリスクがあります。業務用端末では特に慎重に実施してください。

## 1. 対象機種

2016年モデルのMacBook Proは、OCLPの対応機種に含まれています。

| モデル | モデル識別子 |
|---|---|
| MacBook Pro 13-inch, 2016, Thunderbolt 3 × 2 | `MacBookPro13,1` |
| MacBook Pro 13-inch, 2016, Thunderbolt 3 × 4 | `MacBookPro13,2` |
| MacBook Pro 15-inch, 2016 | `MacBookPro13,3` |

参考：[OCLP Supported Models](https://dortania.github.io/OpenCore-Legacy-Patcher/MODELS.html)

### モデル識別子の確認方法

1. Appleメニューから「このMacについて」を開く
2. 「詳しい情報」→「システムレポート」を開く
3. 「ハードウェアの概要」にある「機種ID」を確認する

ターミナルでは、次のコマンドでも確認できます。

```bash
system_profiler SPHardwareDataType | grep "Model Identifier"
```

---

## 2. 推奨するmacOS

2026年8月時点では、安定性を優先するなら **macOS Sequoia 15**を候補とします。

- 2016 MacBook ProのApple公式最終対応OSはmacOS Monterey
- OCLPを使用すると、SonomaやSequoiaをインストール可能
- macOS Tahoe 26はOCLP側でも対応上の課題があるため、業務利用では推奨しない
- 安定性を最優先する場合は、Montereyのまま使用する選択肢も検討する

参考：

- [macOS SequoiaのOCLP対応状況](https://dortania.github.io/OpenCore-Legacy-Patcher/SEQUOIA-DROP.html)
- [macOS Tahoe 26のOCLP対応状況](https://github.com/dortania/OpenCore-Legacy-Patcher/issues/1167)

---

## 3. 事前準備

### 必要なもの

- 2016 MacBook Pro
- **32GB以上のUSBメモリ**
- 安定したインターネット接続
- Macの管理者パスワード
- 電源アダプター
- 外付けSSDまたはバックアップ先
- 2〜4時間程度の作業時間

SonomaやSequoiaでは、インストーラーとパッチが16GBのUSBメモリに収まらない場合があるため、32GB以上が推奨されています。

参考：[Creating macOS Installers](https://dortania.github.io/OpenCore-Legacy-Patcher/INSTALLER.html)

### 作業前チェック

- [ ] Time Machineなどで完全バックアップを取得した
- [ ] 重要ファイルを別媒体にもコピーした
- [ ] Macが電源アダプターに接続されている
- [ ] USBメモリ内の必要なデータを退避した
- [ ] Macの管理者パスワードを確認した
- [ ] 現在のmacOSでディスクエラーがないことを確認した
- [ ] OCLP公式サイトの既知の問題を確認した

---

## 4. 先にmacOS Montereyを最新状態にする

可能であれば、OCLPを使用する前にAppleが正式対応しているmacOS Montereyを最新状態にします。

1. 「システム環境設定」を開く
2. 「ソフトウェア・アップデート」を開く
3. Montereyで提供される更新をすべて適用する
4. 再起動する

これは、Mac本体のファームウェアを可能な限り最新状態にするためです。

---

## 5. FileVaultの状態を確認する

1. 「システム環境設定」を開く
2. 「セキュリティとプライバシー」を開く
3. 「FileVault」を確認する
4. 復旧キーを安全な場所に保存する

アップデート時のトラブルを減らすため、必要に応じて事前にFileVaultを解除します。解除する場合は、復号が完了してから次へ進んでください。

> 2016 MacBook ProにはT1チップが搭載されています。Sonoma以降では、OCLPによるT1関連のパッチが必要です。既存の指紋情報が削除される可能性もあるため、Touch IDやApple Payを利用している場合は注意してください。  
> また、内部ディスク全体の消去はT1関連の問題につながる可能性があるため、特別な理由がなければ**インプレースアップグレード**を推奨します。

参考：[macOS SonomaにおけるT1の注意事項](https://dortania.github.io/OpenCore-Legacy-Patcher/SONOMA-DROP.html)

---

## 6. OCLPをダウンロードする

1. OCLP公式GitHubを開く  
   [OpenCore Legacy Patcher公式GitHub](https://github.com/dortania/OpenCore-Legacy-Patcher)
2. 「Releases」を開く
3. 最新の**正式版**を選択する
4. `OpenCore-Patcher-GUI.app.zip`をダウンロードする
5. ZIPファイルを展開する
6. `OpenCore-Patcher.app`を「アプリケーション」フォルダへ移動する

> ベータ版やNightly Buildは、特別な理由がない限り使用しないでください。  
> OCLPは必ずDortaniaの公式GitHubから入手してください。

初回起動時に警告が表示される場合は、次の手順で開きます。

1. 「システム設定」または「システム環境設定」を開く
2. 「プライバシーとセキュリティ」を開く
3. OCLPに対して「このまま開く」を選択する

---

## 7. macOSインストーラーUSBを作成する

> この操作を行うと、USBメモリ内のデータはすべて消去されます。

1. 32GB以上のUSBメモリを接続する
2. `OpenCore-Patcher.app`を起動する
3. `Create macOS Installer`を選択する
4. `Download macOS Installer`を選択する
5. `macOS Sequoia`の最新安定版を選択する
6. ダウンロード完了まで待つ
7. `Create macOS Installer`に戻る
8. ダウンロードしたSequoiaのインストーラーを選択する
9. 書き込み先のUSBメモリを選択する
10. 管理者パスワードを入力する
11. 作成完了まで待つ

USBメモリの速度によっては、30分以上かかる場合があります。

---

## 8. USBメモリへOpenCoreをインストールする

インストーラーUSBを作成した後、OpenCoreの起動領域もUSBメモリに書き込みます。

1. OCLPのメイン画面へ戻る
2. `Build and Install OpenCore`を選択する
3. `Build OpenCore`を実行する
4. ビルド完了後、`Install OpenCore`を選択する
5. インストーラーを作成したUSBメモリを選択する
6. USBメモリ内のEFIパーティションを選択する
7. 管理者パスワードを入力する
8. インストール完了を確認する

通常はOCLPが実行中のMacを自動判定します。別のMacでUSBを作成する場合は、OCLPの設定で対象モデルを正しく指定してください。

参考：[Building and Installing OpenCore](https://dortania.github.io/OpenCore-Legacy-Patcher/BUILD.html)

---

## 9. USBのOpenCoreから起動する

1. USBメモリを接続したままMacを再起動する
2. 起動音が鳴った直後から`Option`キーを押し続ける
3. Appleの起動ディスク選択画面を表示する
4. OpenCoreのアイコンが付いた`EFI Boot`を選択する
5. OpenCoreの起動選択画面が表示されたら、`Install macOS Sequoia`を選択する

`EFI Boot`を選択するときに`Control`キーを押しながら決定すると、標準の起動先として記録できる場合があります。

参考：[Booting OpenCore and macOS](https://dortania.github.io/OpenCore-Legacy-Patcher/BOOT.html)

---

## 10. macOSをインストールする

### 推奨：既存環境を残してアップグレードする

1. macOSインストーラーを起動する
2. 言語を選択する
3. 「macOSをインストール」を選択する
4. インストール先として、現在macOSが入っている内蔵ディスクを選択する
5. 画面の指示に従ってインストールする

### 再起動時の注意

インストール中は複数回再起動します。

再起動後にインストールが進まない場合は、毎回次の手順を実施します。

1. `Option`キーを押しながら起動する
2. USBメモリの`EFI Boot`を選択する
3. OpenCoreの画面で、内蔵ディスクのmacOSインストーラーまたはmacOSボリュームを選択する

> インストール途中でUSBメモリを抜かないでください。  
> 進捗バーが長時間止まって見えても、すぐに強制終了しないでください。

---

## 11. 内蔵ディスクへOpenCoreをインストールする

macOSの初期設定が終わった段階では、USBメモリを抜くと起動できない可能性があります。そのため、OpenCoreを内蔵ディスクへインストールします。

1. USBメモリを接続した状態でmacOSを起動する
2. `OpenCore-Patcher.app`を起動する
3. `Build and Install OpenCore`を選択する
4. `Build OpenCore`を実行する
5. `Install OpenCore`を選択する
6. インストール先として**内蔵SSD**を選択する
7. 内蔵SSDのEFIパーティションを選択する
8. 管理者パスワードを入力する
9. インストール完了後に再起動する
10. `Option`キーを押しながら起動する
11. 内蔵SSD側の`EFI Boot`を選択する
12. macOSが正常に起動することを確認する
13. Macを終了し、USBメモリを外す
14. USBなしで起動できることを確認する

---

## 12. Post-Install Root Patchを適用する

新しいmacOSでグラフィック、Wi-Fi、Bluetooth、Touch Bar、Touch IDなどを動作させるために、追加パッチが必要な場合があります。

1. `OpenCore-Patcher.app`を起動する
2. `Post-Install Root Patch`を選択する
3. `Start Root Patching`を実行する
4. 管理者パスワードを入力する
5. パッチ適用完了後に再起動する

OCLPから再度パッチを求められた場合は、指示に従って実行します。

> SIPの設定はOCLPが機種とOSに合わせて調整します。明確な理由がない限り、OCLPのSIP設定を手動変更しないでください。

参考：[OCLP Post-Installation](https://dortania.github.io/OpenCore-Legacy-Patcher/POST-INSTALL.html)

---

## 13. 動作確認

更新後は、次の項目を確認します。

- [ ] USBメモリなしで起動できる
- [ ] Wi-Fiへ接続できる
- [ ] Bluetoothが使用できる
- [ ] キーボードとトラックパッドが動作する
- [ ] ディスプレイの明るさを調整できる
- [ ] スリープと復帰が正常に動作する
- [ ] 音声出力とマイクが使用できる
- [ ] USB-C／Thunderboltポートが使用できる
- [ ] 外部ディスプレイへ出力できる
- [ ] Touch Barが動作する
- [ ] Touch IDが動作する
- [ ] カメラが動作する
- [ ] グラフィック表示が滑らかである
- [ ] Time Machineバックアップを再開できる
- [ ] 業務で必要なアプリが起動する

問題がある場合は、OCLPを起動して`Post-Install Root Patch`を再実行します。

参考：[OCLP Hardware Troubleshooting](https://dortania.github.io/OpenCore-Legacy-Patcher/TROUBLESHOOT-HARDWARE.html)

---

## 14. macOSをアップデートするときの手順

OCLP環境では、通常のMacよりも更新手順が増えます。

1. macOSアップデート前にバックアップを取得する
2. OCLPの公式GitHubで、新しいmacOS更新への対応状況を確認する
3. OCLPを対応済みの最新版へ更新する
4. `Build and Install OpenCore`を実行する
5. OpenCoreを内蔵SSDへ再インストールする
6. Macを再起動する
7. 「システム設定」→「一般」→「ソフトウェアアップデート」を実行する
8. macOS更新後にOCLPを起動する
9. `Post-Install Root Patch`を再実行する
10. 再起動して動作確認する

> macOSの大型アップデートを先に実行しないでください。  
> 必ず、対応するOCLPが公開されていることを確認してから更新します。

---

## 15. 起動できない場合

### USBメモリを使って起動する

1. 作成したOCLPインストーラーUSBを接続する
2. `Option`キーを押しながら起動する
3. USB側の`EFI Boot`を選択する
4. OpenCoreから内蔵ディスクのmacOSを選択する
5. 起動後、OpenCoreを内蔵SSDへ再インストールする

### NVRAMをリセットする

1. OpenCoreの起動選択画面を表示する
2. スペースキーを押して補助項目を表示する
3. `Reset NVRAM`を選択する
4. 再起動後、USB側の`EFI Boot`から起動し直す

### キーボードやトラックパッドが動かない場合

- 外付けUSBキーボード・マウスを接続する
- macOSを起動する
- OCLPの`Post-Install Root Patch`を再実行する
- 再起動する

---

## 16. 元のmacOSへ戻す方法

OCLPを削除するだけでは、macOS自体は元に戻りません。Montereyへ戻す場合は、原則として次の作業が必要です。

1. 必要なデータをバックアップする
2. Apple公式のmacOS Montereyインストーラーを準備する
3. Montereyを再インストールする
4. 必要に応じてTime Machineからデータを復元する

作業前に、OCLP用USBとは別に復旧用USBを準備しておくと安全です。

---

## 重要事項

- OCLPはApple公式のツールではない
- OS更新後にRoot Patchの再適用が必要になることがある
- Touch ID、Touch Bar、Apple Pay、スリープなどに不具合が出る可能性がある
- セキュリティ面では、正式対応OSを使用するMacより不利になる場合がある
- 本番業務で利用する場合は、まず予備機で検証する
- OCLP用USBは、復旧用として保管する
- 不具合がなければ、OCLPやmacOSを必要以上に頻繁に更新しない
- macOS Tahoe 26への更新は、OCLP公式の安定対応を確認するまで避ける

## 公式リンク

- [OpenCore Legacy Patcher公式サイト](https://dortania.github.io/OpenCore-Legacy-Patcher/)
- [公式GitHub](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [対応機種一覧](https://dortania.github.io/OpenCore-Legacy-Patcher/MODELS.html)
- [インストーラー作成手順](https://dortania.github.io/OpenCore-Legacy-Patcher/INSTALLER.html)
- [OpenCoreのビルドとインストール](https://dortania.github.io/OpenCore-Legacy-Patcher/BUILD.html)
- [起動手順](https://dortania.github.io/OpenCore-Legacy-Patcher/BOOT.html)
- [インストール後の設定](https://dortania.github.io/OpenCore-Legacy-Patcher/POST-INSTALL.html)
