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
  - [1時間ごとに消費できるコスト](#1時間ごとに消費できるコスト)
  - [コストの計算方法](#コストの計算方法)
- [ロギング](#ロギング)
- [リファレンスマニュアル](#リファレンスマニュアル)
- [フィードバックとバグレポート](#フィードバックとバグレポート)
- [今後の展望](#今後の展望)

<!-- /TOC -->

## 概要

Kibela Web APIは[GraphQL](https://graphql.org/)として提供されています。また、GraphQLの拡張仕様である[Relay GraphQL Server Specification](https://facebook.github.io/relay/docs/en/graphql-server-specification.html)にも準拠しています。

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

なお、MessagePackを利用する場合、Kibela GraphQL schemaにおける `DateTime` 型はMessagePackのtimestamp typeにマップされます。timestamp typeの詳細にういてはお使いのMessagePackシリアライザのドキュメントを参照ください。

## リクエストボディ

リクエストボディには`query`と`variables`を与えてください。`query`パラメータはGraphQL query文字列です。 `variables`はクエリに定義した変数をオブジェクトで与えてください。

これらのクエリパラメータはGraphQLの仕様に従っています。したがって、任意のGraphQL clientを利用できるはずです。

## サンプルコード

シンプルなサンプルスクリプトは次のリポジトリにあります。

* TypeScript: https://github.com/kibela/hello-kibela.ts

## 利用制限

Kibela Web APIは過剰な負荷を避けるためにいくつかの利用制限を掛けています。

※ 個々の具体的な制限については現在調整中です。制限は予告なく変更することがあります。

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
      }
    }
  ]
}
```

このエラーは再送しても必ず同じエラーを返すので、クエリを組み直してください。

現在、リクエストごとの最大コストは _10,000_ です。この値は予告なく変える可能性があります。

### 1時間ごとに消費できるコスト

また、1時間ごとに消費できるコストの予算 (budget) がアクセストークンとチームごとに定められており、これを超えると次のエラーになります。

* アクセストークンのバジェット超過: `TOKEN_BUDGET_EXHAUSTED`
* チームのバジェット超過: `TEAM_BUDGET_EXHAUSTED`

エラーレスポンス（抜粋）

```json
{
  "errors": [
    {
      "extensions": {
        "code": "TOKEN_BUDGET_EXHAUSTED",
        "waitMilliseconds": 1000,
      }
    }
  ]
}
```

このとき、同じクエリをリクエストするのであれば `waitMilliseconds` だけ待てば再度可能になります。

なお、予算は1msに1回復します。

現在、アクセストークンの1時間ごとの予算は _300,000_ です。また、チームの1時間ごとの予算は _3,000,000_ です。この値は予告なく変える可能性があります。

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

## 今後の展望

将来的に次のような拡張を予定しています。

* OAuth 2.0 への対応
* Access Token自体の情報をreadonlyでチーム全体に公開するか検討中
* Web API利用制限・予算管理の調整
