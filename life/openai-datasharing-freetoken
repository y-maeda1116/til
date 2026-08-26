# OpenAI API データ共有・無料トークン設定確認

## 概要

OpenAI APIでは、「Share inputs and outputs with OpenAI」を有効にし、無料トークンプログラムの対象となっている場合、共有対象のAPI通信に無料トークンが適用される。

Tier 1〜2の場合、GPT-5.6 Terra／Lunaなどの対象モデル群を、合計で1日最大250万トークンまで無料で利用できる。

> APIへ入力した内容と出力結果がOpenAIへ共有され、モデルの評価・改善・学習などに利用される可能性がある。  
> 社内機密、個人情報、顧客情報、未公開情報などは送信しないこと。

## 設定・確認手順

### 1. API残高を確認する

OpenAI APIの無料トークンを利用するには、APIアカウントの残高がプラスである必要がある。

- 例：$5をAPIクレジットとして入金
- ChatGPT Plus／Businessなどの契約料金は、API残高とは別扱い
- $5を入金しただけで、無料トークンの対象になることが保証されるわけではない

### 2. 無料トークンの対象資格を確認する

以下の設定画面を開く。

https://platform.openai.com/settings/organization/data-controls/sharing

画面に次の表示があることを確認する。

> You're eligible for free daily usage on traffic shared with OpenAI

この表示がない場合、現時点では無料トークンプログラムの対象外。

### 3. データ共有を有効にする

「Share inputs and outputs with OpenAI」を、以下のいずれかに設定する。

- Enabled
- Enabled for selected projects

社内利用の場合は、影響範囲を限定するため、原則として「Enabled for selected projects」を選択し、検証専用プロジェクトのみを対象にする。

設定変更にはOrganization Owner権限が必要。

### 4. 登録完了を確認する

設定後、次の表示が出ることを確認する。

> You’re enrolled for complimentary daily tokens

この表示があれば、無料トークンプログラムへの登録が完了している。

## Tier 1〜2の無料枠

### GPT-5.6 Terra／Lunaを含むモデル群

- 1日最大：2,500,000トークン
- 入力トークンと出力トークンの合計
- 対象モデル間で無料枠を共有
- TerraとLunaでそれぞれ250万トークンではない

主な対象モデル：

- `gpt-5.6-terra`
- `gpt-5.6-luna`
- `gpt-5.4-mini`
- `gpt-5.4-nano`
- `gpt-5-mini`
- `gpt-5-nano`
- `gpt-4.1-mini`
- `gpt-4.1-nano`
- `gpt-4o-mini`
- `o4-mini`

### 上位モデル群

GPT-5.6 Solなどの上位モデル群は、Tier 1〜2の場合、合計で1日最大250,000トークン。

主な対象モデル：

- `gpt-5.6-sol`
- `gpt-5.5`
- `gpt-5.4`
- `gpt-5.2`
- `gpt-5.1`
- `gpt-5`
- `gpt-4.1`
- `gpt-4o`
- `o1`
- `o3`

## 無料枠の適用条件

- データ共有を有効にしたプロジェクトのAPI通信のみ対象
- 入力トークンと出力トークンの両方をカウント
- 無料枠は毎日UTC 0:00にリセット
- 日本時間では毎日9:00にリセット
- 日次上限を超えた利用は通常料金で請求
- 無料枠を超えるリクエストは、超過分だけでなく、そのリクエスト全体が課金対象
- プログラム内容は将来変更・終了される可能性がある

## 無料枠の対象外

以下は無料トークンの対象外。

- ファインチューニング済みモデル
- ファインチューニングの学習処理
- Evals
- Web Searchなどのツール利用料
- Computer Useなどのツール利用料
- データ共有を有効にしていないプロジェクト
- 無料枠の対象モデル一覧に含まれないモデル

## 利用実績の確認方法

Usage Dashboardを開く。

https://platform.openai.com/settings/organization/usage

確認項目：

- Input tokens
- Output tokens
- Costs
- Service tier

無料トークンは、Service tierで次のように表示される。

- `data sharing incentive tier - input tokens`
- `data sharing incentive tier - output tokens`

無料枠として処理されたトークンはUsageには表示されるが、Costsには課金額として表示されない。

## データ取り扱い上の注意

データ共有を有効にしたプロジェクトでは、入力・出力がOpenAIへ共有され、モデルの品質評価、改善、研究、学習などに利用される可能性がある。

以下の情報は入力しない。

- 個人情報
- 顧客情報
- 問い合わせ履歴
- 社内機密情報
- 未公開の事業情報
- 契約書や見積書
- 認証情報
- APIキー、パスワード、アクセストークン
- 未公開のソースコード
- 第三者から守秘義務を負って預かっている情報

必要に応じて、事前に以下の処理を行う。

- 個人名やメールアドレスの削除
- 顧客名・企業名の匿名化
- IDや識別子のマスキング
- 認証情報やシークレットの除去
- 実データをダミーデータへ置換

## 推奨構成

社内・本番用途と、無料トークンを利用する検証用途を分離する。

### 本番・社内データ用プロジェクト

- データ共有：無効
- 機密情報・個人情報：社内ルールに基づいて利用
- 必要に応じて法人契約やZero Data Retentionを検討

### 無料枠検証用プロジェクト

- データ共有：有効
- 機密情報・個人情報：入力禁止
- 公開情報、ダミーデータ、合成データのみ利用
- 予算上限と利用アラートを設定

## 最終確認チェックリスト

- [ ] API残高がプラスになっている
- [ ] 無料トークン対象資格の表示がある
- [ ] Organization Ownerが設定を実施している
- [ ] データ共有を有効にするプロジェクトを限定した
- [ ] 登録完了の表示を確認した
- [ ] 対象モデルの正確なモデルIDを使用している
- [ ] Usage Dashboardで無料トークンの適用を確認した
- [ ] 入力・出力の両方を集計対象にしている
- [ ] 機密情報・個人情報を入力しない運用ルールを設けた
- [ ] 通常課金に備えて予算上限・アラートを設定した

## 公式情報

- データ共有設定  
  https://platform.openai.com/settings/organization/data-controls/sharing

- Usage Dashboard  
  https://platform.openai.com/settings/organization/usage

- OpenAI公式ヘルプ  
  https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai
