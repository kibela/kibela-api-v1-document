# Kibela Web API

Kibela Web API は、Kibelaのデータにアクセスするツールを開発するためのWeb APIです。他サービスからKibelaへのインポートツールなどを想定しています。

## Table of Contents

<!-- TOC depthFrom:2 anchorMode:github.com -->

- [Table of Contents](#table-of-contents)
- [概要](#概要)
- [アクセストークン](#アクセストークン)
- [エンドポイント](#エンドポイント)
- [リクエストヘッダ](#リクエストヘッダ)
  - [JSON](#json)
  - [MessagePack](#messagepack)
- [リクエストボディ](#リクエストボディ)
- [サンプルコード](#サンプルコード)
- [利用制限](#利用制限)
  - [1秒あたりのリクエスト数](#1秒あたりのリクエスト数)
  - [1リクエストごとに消費できるコスト](#1リクエストごとに消費できるコスト)
  - [アクセストークンごとのレートリミット](#アクセストークンごとのレートリミット)
  - [チームごと（プラン別）の1時間あたりのコスト上限](#チームごとプラン別の1時間あたりのコスト上限)
  - [コストの計算方法](#コストの計算方法)
- [ロギング](#ロギング)
- [リファレンスマニュアル](#リファレンスマニュアル)
- [フィードバックとバグレポート](#フィードバックとバグレポート)

<!-- /TOC -->

## 概要

Kibela Web APIは[GraphQL](https://graphql.org/)として提供されています。また、GraphQLの拡張仕様である[Relay GraphQL Server Specification](https://relay.dev/docs/guides/graphql-server-specification/)にも準拠しています。

## アクセストークン

「設定」→「個人用アクセストークン」からアクセストークンを生成してください。ゲスト以外のメンバーは誰でもアクセストークンを生成・使用できます。

https://my.kibe.la/settings/access_tokens

それぞれのアクセストークンには説明を書けます。関連するURL、たとえば該当アクセストークンを利用するツールのリポジトリURLなどをマークダウンで受け付けるようになっています。

アクセストークンは無期限で有効です。不要なアクセストークンは明示的に無効化 (revoke) してください。なお、ユーザーがチームから削除されたときはそのユーザーが作成したアクセストークンはすべて無効化されます。

## エンドポイント

`https://${TEAM_NAME}.kibe.la/api/v1`

`${TEAM_NAME}` はお使いのチーム名にしてください。

このエンドポイントはPOSTリクエストのみ受け付けます。

## リクエストヘッダ

次のヘッダを指定してください。 `${ACCESS_TOKEN}` はアクセストークンに置き換えてください。

* `Authorization: Bearer ${ACCESS_TOKEN}`
* `Content-Type: application/json` or `Content-Type: application/x-msgpack`
* `Accept: application/json` or `Accept: application/x-msgpack, application/json`

また、`User-Agent` ヘッダは指定することを奨励します。 `User-Agent` はアクセストークンごとの使用履歴（ログ）にも記録されます。

### JSON

リクエストやレスポンスでJSONフォーマットを利用する場合は、リクエストヘッダに `Content-Type: application/json` や `Accept: application/json` を指定してください。

* `Content-Type: application/json`
* `Accept: application/json`

### MessagePack

リクエストとレスポンスはそれぞれ[MessagePack](https://msgpack.org/)フォーマットも利用できます。

リクエストをMessagePackにする場合は`Content-Type: application/x-msgpack`を、レスポンスをMessagePackにする場合は`Accept: application/x-msgpack, application/json`をそれぞれ指定してください。

* `Content-Type: application/x-msgpack`
* `Accept: application/x-msgpack, application/json`

現在のところ、エラーレスポンスはJSONを返すことがあります。必ずレスポンスヘッダの`Content-Type`をみてデコードしてください。

MessagePackはバイナリの転送においてオーバーヘッドがなく、また多くのMessagePack処理系においてレスポンスボディのストリーミングデコードが可能であるためJSONよりも遥かに高速にリクエスト・レスポンス処理を行えます。Kibela Web APIを利用するツールは、特に運用フェーズではなるべくMessagePackを使うことを奨励します。

なお、MessagePackを利用する場合、Kibela GraphQL schemaにおける `DateTime` 型はMessagePackのtimestamp typeにマップされます。timestamp typeの詳細についてはお使いのMessagePackシリアライザのドキュメントを参照ください。

## リクエストボディ

リクエストボディには`query`と`variables`を与えてください。`query`パラメータはGraphQL query文字列です。 `variables`はクエリに定義した変数をオブジェクトで与えてください。

これらのクエリパラメータはGraphQLの仕様に従っています。したがって、任意のGraphQL clientを利用できるはずです。

## サンプルコード

シンプルなサンプルスクリプトは次のリポジトリにあります。

* TypeScript: https://github.com/kibela/hello-kibela.ts

またcurlを用いたシンプルな例を次に示します。
お使いのKibelaのサブドメイン名と個人用アクセストークンをセットした上で実行できます。

```bash
export SUBDOMAIN='YOUR-SUBDOMAIN'
export SECRET='secret/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'

curl "https://${SUBDOMAIN}.kibe.la/api/v1" \
  -H "Authorization: Bearer ${SECRET}" \
  -X POST \
  -d '{"query": "query { currentUser { realName } }", "variables": {}}' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'User-Agent: KibelaAPITestFromCurl'
```

## 利用制限

Kibela Web APIは過剰な負荷を避けるためにいくつかの利用制限を掛けています。

### 1秒あたりのリクエスト数

まず、1秒あたりのリクエスト数です。これは、**1秒につき最大10リクエスト**を超えてはいけません。リクエストを連投するときは最低でも100msあけるようにしてください。この制限をこえるとHTTP status code 429 Too Many Request を返します。

### 1リクエストごとに消費できるコスト

1リクエストごとに最大コストが定められており、これを超えるとレスポンスボディの `errors.extensions.code=REQUEST_LIMIT_EXCEEDED` を返します。HTTP status codeは200です。

エラーレスポンス（抜粋）

```json
{
  "errors": [
    {
      "extensions": {
        "code": "REQUEST_LIMIT_EXCEEDED",
        "cost": "10101",
        "maxCostPerRequest": "10000"
      }
    }
  ]
}
```

このエラーは再送しても必ず同じエラーを返すので、クエリを組み直してください。

リクエストごとの最大コストは 10,000 です。

### アクセストークンごとのレートリミット

短時間に大量のリクエストが送信された場合、アクセストークンごとに一時的なリクエスト制限が掛かります。

- 制限の目安：直近30秒間の消費コストの合計が 30,000 cost を超えると制限に達します。最大コスト（10,000 cost）のリクエストを間隔をあけずに連投した場合、4回目で制限に達します。
- 制限時の挙動：一時的な制限に達した場合、エラーレスポンスが返されます。
- 解除方法：エラーレスポンス内に含まれる `waitMilliseconds` フィールドに示されたミリ秒数だけ待機したのち、リクエストを再試行してください（最大待ち時間は約10秒です）。

なお、消費したコストは1msに1 costずつ回復します。そのためリクエストの間隔をあければ、上記より多くのリクエストを送ることができます。

制限に達すると次のエラーレスポンスを返します。

* アクセストークンのバジェット超過: `TOKEN_BUDGET_EXHAUSTED`

エラーレスポンス（抜粋）

```json
{
  "errors": [
    {
      "extensions": {
        "code": "TOKEN_BUDGET_EXHAUSTED",
        "cost": "5101",
        "consumed": "30100",
        "waitMilliseconds": "5201",
        "tokenBurstLimit": "30000"
      }
    }
  ]
}
```

このとき、`waitMilliseconds` に示されたミリ秒数だけ待機してからリクエストを再試行してください。最大待ち時間は約10秒です。

### チームごと（プラン別）の1時間あたりのコスト上限

ご契約のプランごとに、1時間あたりに消費できる合計コストの予算 (budget) が定められています。

| プラン | 1時間あたりのコスト上限 |
| --- | --- |
| コミュニティ | 20,000 cost |
| ライト | 1,000,000 cost |
| スタンダード | 上限なし ※ |
| エンタープライズ | 上限なし ※ |

※ スタンダード・エンタープライズプランは原則として上限を設けていないため、大規模なデータ連携やAIサービスからのアクセスにもご利用いただけます。ただし、サービス全体の安定運用に影響を与えるような極端な利用が確認された場合は、個別に制限を掛ける場合があります。

チームの予算を超過した場合は次のエラーになります。

* チームのバジェット超過: `TEAM_BUDGET_EXHAUSTED`

エラーレスポンス（抜粋）

```json
{
  "errors": [
    {
      "extensions": {
        "code": "TEAM_BUDGET_EXHAUSTED",
        "cost": "5101",
        "consumedInTeam": "1000100",
        "waitMilliseconds": "1800000",
        "teamBudgetPerHour": "1000000"
      }
    }
  ]
}
```

こちらも `waitMilliseconds` に示されたミリ秒数だけ待機してからリクエストを再試行してください。

### コストの計算方法

コストは基本的にGraphQLのフィールド1つにつき1 costです。ただし、1リクエストごとに基本コストが設定されており、かならず基本コスト分消費します。

また、connectionフィールドはかならず `first` または `last` パラメータで取得数を明示する必要があり、その「取得数 × childre node」がconnection全体のコストになります。

計算されたコストは `Query.budget.cost` クエリでみることができます。

なお、一部のフィールドはコストが通常よりも高いことがあります。たとえば、全文検索やmarkdownのレンダリング（ `contentHtml` など）はコストを高めに設定しています。

消費コストについてはバランスを調整中です。調整が終わったら詳細を公開します。

## ロギング

`query`パラメータの値はログに保存される対象です。このログはチームの管理者にも解放される予定です。秘匿値は`query`パラメータに書かず、`variables`パラメータとして与えてください。

## リファレンスマニュアル

リファレンスマニュアルは現在、「設定」→「個人用アクセストークン」→「Web API console」 (GraphiQL) の "Documentation Explorer" から利用できます。

https://my.kibe.la/api/console

## フィードバックとバグレポート

このドキュメントのissuesでフィードバックとバグレポートを受け付けています。起票は日本語ないし英語でお願いいたします。

https://github.com/kibela/kibela-api-v1-document/issues
