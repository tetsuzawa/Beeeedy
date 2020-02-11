# Beeeedy
VR Sound web application project

# 開発方針

- 新しい機能はissueを建てる
- 追加した機能はPRを建てる
- 設計
- Circle CIでCI環境を構築してから
- GitHubでタグを切る
- バージョンはgoogleのバージョンニングを参照
- protoで設計
- swaggerに落とし込む


# 開発環境

- インフラ
  - Terraform
  - AWS
    - Route53
    - VPC
    - ECR
    - ECS
    - RDS
    - [Restroomers](https://github.com/jinugasachio/Restroomers.com)を参考
  - 余裕があればGCPも検討

- CI/CD
  - Circle Ci

- バックエンド
  - ディレクトリ構成は[Google API Design Guide](https://cloud.google.com/apis/design)に準拠
  - gRPCとREST APIを共存
  - Goで記述
  - [Gorm](https://github.com/jinzhu/gorm)
  - 

- フロントエンド
  - Nuxt.js
  - axios
  - Bulma
  - Webpack
  - 

# 設計

## はじめに

1. gRPC API を中心にREST APIも設計
2. gRPC API では、プロトコル バッファを使用して API サーフェスを定義し、API サービス構成を使用して API サービス（HTTP マッピング、ロギング、モニタリングなど）を構成します。
3. HTTP マッピング機能は、Google API と Cloud Endpoints gRPC API によって JSON / HTTP からプロトコル バッファ / RPC へのコード変換に使用されます。

## リソース試行の設計

次の手順で設計することが推奨されている

- API で提供するリソースのタイプを決定する。
- リソース間の関係を決定する。
- タイプと関係に基づいてリソース名のスキームを決定する。
- リソース スキーマを決定する。
- 最小限の一連のメソッドをリソースにアタッチする。

## リソース名

`"//library.googleapis.com/v1/shelves/shelf1/books/book2"`

のように`/<API バージョン>/<コレクションID>/<リソースID>/<コレクションID>/<リソースID>`とする

## 標準フィールド

異なる API 間で、同じコンセプトには同じ名前とセマンティクスが使用されるようになります。

## エラー

```protobuf
package google.rpc;

message Status {
  // A simple error code that can be easily handled by the client. The
  // actual error code is defined by `google.rpc.Code`.
  int32 code = 1;

  // A developer-facing human-readable error message in English. It should
  // both explain the error and offer an actionable resolution to it.
  string message = 2;

  // Additional error information that the client code can use to handle
  // the error, such as retry delay or a help link.
  repeated google.protobuf.Any details = 3;
}
```

これに従う

### HTTP マッピング

proto3 メッセージにはネイティブ JSON エンコードがありますが、Google の API プラットフォームでは、下位互換性を維持するため、Google JSON REST API に別のエラースキーマを使用しています。

スキーマ:

```proto
// The error schema for Google REST APIs. NOTE: this schema is not used for
// other wire protocols.
message Error {
  // This message has the same semantics as `google.rpc.Status`. It has an extra
  // field `status` for backward compatibility with Google API Client Library.
  message Status {
    // This corresponds to `google.rpc.Status.code`.
    int32 code = 1;
    // This corresponds to `google.rpc.Status.message`.
    string message = 2;
    // This is the enum version for `google.rpc.Status.code`.
    google.rpc.Code status = 4;
    // This corresponds to `google.rpc.Status.details`.
    repeated google.protobuf.Any details = 5;
  }
  // The actual error payload. The nested message structure is for backward
  // compatibility with Google API client libraries. It also makes the error
  // more readable to developers.
  Status error = 1;
}
```


## 命名規則

### パッケージ名
API .proto ファイルで宣言されたパッケージ名は、プロダクト名やサービス名と一貫したものにすることが推奨されます。バージョンを含む API のパッケージ名の末尾は、バージョンにする必要があります。次に例を示します。

// Google Calendar API
package google.calendar.v3;

### コレクション ID
コレクション ID では、複数形、lowerCamelCase、アメリカ英語のスペルとセマンティクスを使用することが推奨されます。たとえば、events、children、deletedEvents です。

### インターフェース名
pubsub.googleapis.com などのサービス名と混同しないように、「インターフェース名」という用語は、.proto ファイル内での service の定義に使用する名前を指します。

```proto
// Library is the interface name.
service Library {
  rpc ListBooks(...) returns (...);
  rpc ...
}
```

サービス名は一連の API の実際の実装への参照であり、インターフェース名は API の抽象的な定義を指すと考えることができます。

インターフェース名には、Calendar や blob などの直感的な名詞を使用することが推奨されます。

### メソッド名

メソッド名は、UpperCamelCase で VerbNoun の命名規則に従うことが推奨されます。通常、名詞はリソースタイプを表します。

## 一般的な設計パターン

### 空のレスポンス

標準の Delete メソッドでは、「ソフト」削除を行う場合を除き、google.protobuf.Empty が返されることが推奨されます。「ソフト」削除の場合、このメソッドでは、削除が進行中であることを示すように状態が更新されたリソースが返されることが推奨されます。

カスタム メソッドでは、空の場合でも独自の XxxResponse メッセージが返されることが推奨されます。

### 文法の構文
API 設計では、多くの場合、受け入れ可能なテキスト入力など、特定のデータ形式に対して単純な文法を定義する必要があります。API 間で一貫したデベロッパー エクスペリエンスを提供し、容易に習得できるように、API 設計者は Extended Backus-Naur Form（EBNF）構文の次のバリアントを使用してそのような文法を定義する必要があります。

```proto
Production  = name "=" [ Expression ] ";" ;
Expression  = Alternative { "|" Alternative } ;
Alternative = Term { Term } ;
Term        = name | TOKEN | Group | Option | Repetition ;
Group       = "(" Expression ")" ;
Option      = "[" Expression "]" ;
Repetition  = "{" Expression "}" ;
```

### 整数型
uint32 や fixed32 などの符号なしの整数型は、Java、JavaScript、OpenAPI などの一部の重要なプログラミング言語やシステムでサポートされていないため、API 設計では使用しないことが推奨されます。


## ドキュメント
### API の説明
API の説明は、API でできることを記述する能動態動詞で始まるフレーズです。.proto ファイルでは、次の例のように、API の説明は対応する service にコメントとして追加します。

```proto
// Manages books and shelves in a simple digital library.
service LibraryService {
...
}
```

### フィールドとパラメータの説明

フィールド値が required、input only、output only の場合は、フィールドの説明の最初にドキュメント化する必要があります。デフォルトでは、すべてのフィールドとパラメータは省略可能です。例:

```proto
message Table {
  // Required. The resource name of the table.
  string name = 1;
  // Input only. Whether to dry run the table creation.
  bool dryrun = 2;
  // Output only. The timestamp when the table was created. Assigned by
  // the server.
  Timestamp create_time = 3;
  // The display name of the table.
  string display_name = 4;
}
```


## バージョニング

1. 互換性のない API の変更を行う場合には、MAJOR バージョン
2. 下位互換性のある方法で機能を追加する場合には、MINOR バージョン
3. 下位互換性のあるバグ修正を行う場合には、PATCH バージョン


## ディレクトリ構成

- API ディレクトリ
  - リポジトリの前提条件
    - BUILD - ビルドファイル。
    - METADATA - ビルド メタデータ ファイル。
    - OWNERS - API ディレクトリのオーナー。

  - 構成ファイル
    - {service}.yaml - ベースライン サービス構成ファイル。これは、google.api.Service proto メッセージの YAML 表現です。
    - prod.yaml - 本番デルタサービス構成ファイル。
    - staging.yaml - ステージング デルタサービス構成ファイル。
    - test.yaml - テスト デルタサービス構成ファイル。
    - local.yaml - ローカル デルタサービス構成ファイル。

  - ドキュメント ファイル
    - README.md - メインの readme ファイル。これには、一般的な作成概要、技術的な説明などを含めます。
    - doc/* - 技術ドキュメント ファイル。Markdown 形式にします。

  - インターフェース定義
    - v[0-9]*/* - このような各ディレクトリには、API のメジャー バージョン、主に proto ファイルとビルド スクリプトが含まれています。
    - {subapi}/v[0-9]*/* - 各 {subapi} ディレクトリには、サブ API のインターフェース定義が含まれています。各サブ API は、独自の独立したメジャー バージョンを持つことができます。
    - type/* - 異なる API 間、同じ API の異なるバージョン間、API とサービス実装の間で共有される型を含む proto ファイル。type/* の下にある型の定義には、リリースされた後は互換性が損なわれるような変更を加えないでください。