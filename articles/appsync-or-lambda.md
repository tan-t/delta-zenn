---
title: "結局プロダクションでAppSyncって使えるの？"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql","appsync","serverless"]
published: false
---

# 言いたいこと
- かなり使えるが、lambdaに集約したほうがメンテナビリティは高いと思う
- かなり使えるのは事実なのでうちでは頑張って使っている

# AWS AppSyncとは？
- AppSyncは、ローコードでGraphQLサーバーを構築できるサービスである。
- 基本思想としては、「リクエストを、AWSサービスへのリクエスト（≒CLIコマンド）に**変換する**」という感じ
- 位置付けとしてはAPI Gatewayに似ている。

- 例えば、「DynamoDBからデータを取ってくる」とか、「ElasticSearchからデータを取ってくる」とか、「Lambdaを叩く」とかがGraphQL越しにできる。

- 通常、API Gateway + lambda + DynamoDB のようなアーキテクチャの場合、データや検索など比較的簡単な処理に対してもいちいちlambdaを書く必要があったのに対し、ローコードでかつDynamoDBなどの各サービスに直接リクエストできるというのが強み

# AWS AppSyncをめぐるエコシステム
- GraphQL
- IAM
- VTL, mapping, DataSource
- Amplify
- lambda

## GraphQL
各自でググってクレメンス

## IAM
- AppSyncの各Type / FieldはAWSのリソースとして認識される。
- なので、IAMのポリシーとして特定のType / Fieldへのアクセスをブロックしたり許可したりできる。

## VTL, mapping, DataSource

- 各Type.fieldに対して、「データソース」を割り当てる必要がある。
  - 例えば、`Query.getUser` に、`UserTable`というデータソースを割り当てる

- 「データソース」は、AWSサービスを抽象化したマスタ。
  - 例えば、`UserTable`というデータソースはDynamoDBの特定のテーブル(例えば、`User`)に紐づいている、と設定する

- そのうえで、VTLという言語でリクエストをAWSサービスへのリクエストに変換するプログラムを実装する。
  - これが所謂リゾルバーの「実装」になる。
  - 例えばこんな感じ。
```vtl
{
  "version": "2017-02-28",
  "operation": "GetItem",
  "key":  {
    "id": $util.dynamodb.toDynamoDBJson($ctx.args.id)
  } #end
}
```
- すべてまとめると、`Query.getUser`はDynamoDBの`User`というテーブルに`{id: ${クエリの引数の「id」で渡ってきた値}}`というキーでGetItemする、というリゾルバーの実装が完成する。

まあ、getItemぐらいだと別にいいのだが、この作業が死ぬほど面倒。  
ローコードとは・・・？

## Amplify

### Amplify CLI
- AmplifyとAmplify Consoleはほぼ別物なので注意。
- いわゆるAmplify CLIと呼ばれるフレームワークは、思想としてはfirebaseに似ているが中身としてはCLI ベースで謎のJSONメタデータを管理し、cloudformationを生成するツール。
- Amplify APIは上述のAppSyncの設定ファイルをGraphQLスキーマ定義からディレクティブベースで生成してくれる。
- 要は、 `type User{}`を定義すると、上述の`Query.getUser`をはじめとして`Mutation.createUser` `Mutation.updateUser`などのCRUDオペレーションとそのVTL / mapping / データソース定義を生やしてくれるなど。
- Amplify AuthはCognitoの設定を生やしてくれたりする。
- かなりがっつりしたフレームワークなので、Amplifyで管理していないAWSのリソース（SQSやStepFunctionsなど）を利用しようとした場合の自由度がかなり低い。
- lambdaを利用するためには`amplify add function`を叩く必要があるのだが、デフォルトでjsのpackageを生成するので頗る開発ビリティが低い。

### Amplify js
- Amplify は、CLIで作成したバックエンドをフロントエンドから利用するためのクライアントライブラリを持っている。
- 中身は単なるAWS SDKのラッパーなので、利用するバックエンドはAmplify CLIで作成したものである必要はない。
- CognitoやIAMでの認証を備えたAppSyncのAPIクライアントとして、普通に使い勝手がよい。
- ただ、なぜか**ドメインがAppSync標準のものでないと認証情報が上手く渡せなくなる**という仕様になっている。


## lambda  (+ API Gateway)
- GraphQLといえど、中身は普通のHTTPリクエストである。
- なので、API Gatewayで`/graphql` エンドポイントを開けておき、lambda内にapollo-express-serverを立てるというのも手になる。
- lambda内で普通にアプリケーションコードとしてリゾルバーを実装する必要があるため、AppSyncのように、ローコードでAWSサービスにアクセスできるということはなくなるが、管理するリソースは逆に少なくなる。

# アーキテクチャの案
## Amplify js + AppSync(手実装)
### Pros
- lambdaの記述量が減る。
- IAMでType / Fieldレベルでの認可が可能。
- Cognitoとのインテグレーションが容易。
- serverless fwなどのIaCツールが利用しやすく、AWSの他のリソースのプロビジョニングが楽。

### Cons
- lambdaとVTLとで永続層へのアクセスロジックを二重管理することになる。
- 複雑な権限制御などをやろうとすると結局lambdaを使うことになる。
- **毎回ドメインが変わる(Amplify jsの制約)**
- AppSyncの実装が面倒。
- 特定のリソースへのリクエスト中に別のリソースを参照しづらい。（データソースを明示的に指定するため）

## Amplify js + Amplify API
### Pros
- lambdaの記述量が減る。
- IAMでType / Fieldレベルでの認可が可能。
- Cognitoとのインテグレーションが容易。
- AppSyncの実装が楽。
- Cognitoでのかなりきめ細かな認可が自動生成可能。

### Cons
- lambdaとVTLとで永続層へのアクセスロジックを二重管理することになる。
- 複雑な権限制御などをやろうとすると結局lambdaを使うことになる。
- **毎回ドメインが変わる(Amplify jsの制約)**
- 特定のリソースへのリクエスト中に別のリソースを参照しづらい。（データソースを明示的に指定するため）
- AWSの他のリソースのプロビジョニングが大変になる
- **DynamoDBのテーブルやインデックスが増えがち**

## Apollo Client + Apollo Server on Lambda over API Gateway
### Pros
- 永続層へのアクセスロジックが集約できる。
- 認可やロギング、エラーハンドリングなどのミドルウェアを追加しやすい。
- あるリソースへのリクエスト時に、別のリソースを参照しやすい。

### Cons
- アプリケーションの実装量は多くなる。
- 全てのリクエストでlambdaをinvokeするため、起動のオーバーヘッドが発生する。
- Subscriptionを実装できない。
- 関数が一つになってしまうため、ログの管理やデプロイが大変。

