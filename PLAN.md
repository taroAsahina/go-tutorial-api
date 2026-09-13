# 計画: Clean Architecture で API の 1 path を通す

Go 初学者向け。依存関係の向きと各層の役割を体験する。HTTP は標準ライブラリ、永続化は PostgreSQL、クエリは SQLC を使う。

## ゴール

`GET /tasks` を叩くと、PostgreSQL に入っているタスク一覧が JSON で返る。

リクエストは次の順で全層を通る。

```
HTTP リクエスト
  → handler
  → usecase
  → port（インターフェース）
  → gateway
  → driver（SQLC が生成したコード + DB 接続）
  → PostgreSQL
  → 同じ道を戻って JSON レスポンス
```

成功時の例:

```http
GET /tasks HTTP/1.1
```

```json
[
  {"id": "1", "title": "Learn Go"},
  {"id": "2", "title": "Build an API"}
]
```

DB は Docker で立ち上げる。compose と初期 SQL はプロジェクト直下の `environments/` に置く。

## SQLC について

SQLC は ORM ではない。自分で書いた SQL から、型付きの Go コードを生成するツール。

- スキーマと `SELECT` を自分で書く
- `sqlc generate` で Go の関数ができる
- driver はその生成コードを呼ぶ
- usecase / port は SQLC を知らない

GORM のような ORM は使わない。

## やらないこと（今回の範囲外）

- POST / PUT / DELETE
- 認証、バリデーション、エラーハンドリングの本格実装
- フレームワーク（Gin, Echo, Chi など）
- マイグレーションツール（golang-migrate など）。初期化は Docker の init SQL で行う

## 層の役割

依存の向きは **内側（usecase / port）を、外側（handler / gateway / driver）が使う**。usecase は「PostgreSQL か」「SQLC か」を知らない。

| 層 | 役割 | 知ってよいこと |
| --- | --- | --- |
| handler | HTTP の入口。リクエストを受け、usecase を呼び、JSON を書く | HTTP、usecase |
| usecase | 「タスク一覧を返す」という処理そのもの | port、domain |
| port | usecase が使う契約（インターフェース） | domain |
| gateway | port を実装する。domain と driver のデータの橋渡し | port、driver、domain |
| driver | DB 接続と SQLC 生成コードの呼び出し | SQLC、DB、自分のデータ形式 |

domain（`Task` 型）は層というより、各層が共有する中身の定義。

```
handler ──► usecase ──► port ◄── gateway ──► driver ──► PostgreSQL
                │                              │
                └── domain                     └── SQLC 生成コード
```

## ディレクトリ構成

```
.
├── PLAN.md
├── go.mod
├── sqlc.yaml
├── cmd/api/main.go
├── environments/
│   ├── docker-compose.yml
│   └── postgres/
│       ├── 01_schema.sql
│       └── 02_seed.sql
└── internal/
    ├── domain/task.go
    ├── port/task.go
    ├── driver/postgres/
    │   ├── query.sql
    │   ├── generated/          # sqlc generate の出力（手で書かない）
    │   └── client.go
    ├── gateway/task.go
    ├── usecase/list_tasks.go
    └── handler/task.go
```

## タスク一覧

ペアプロでは、完了報告を受けてから次へ進む。

### Task 1: environments で PostgreSQL を立てる

`environments/` を作り、Docker で PostgreSQL を起動する。

- `environments/docker-compose.yml` を置く
- イメージは PostgreSQL 16
- ポートは `5432`
- ユーザー / パスワード / DB 名はどれも `app` でよい
- 初期化 SQL 用に `./postgres` を `/docker-entrypoint-initdb.d` へマウントする
- 起動コマンドはリポジトリ直下から:

```bash
docker compose -f environments/docker-compose.yml up -d
```

完了条件: コンテナが起動し、`localhost:5432` に接続できる。

### Task 2: スキーマと初期データ

`environments/postgres/` に SQL を置く。Docker が初回起動時に流す。

- `01_schema.sql`: `tasks` テーブル（`id TEXT PRIMARY KEY`, `title TEXT NOT NULL`）
- `02_seed.sql`: 2 件 INSERT（id=`1` / `2`、title はゴールの JSON と同じ）
- compose を作り直して SQL を反映する

```bash
docker compose -f environments/docker-compose.yml down -v
docker compose -f environments/docker-compose.yml up -d
```

`-v` は volume も消す。init SQL は空の volume のときだけ走る。

完了条件: DB に入って `SELECT * FROM tasks;` で 2 件見える。

### Task 3: domain

`internal/domain/task.go` を作る。

- `Task` 構造体を定義する（フィールド: `ID string`, `Title string`）
- JSON で返すため、フィールド名は大文字で始める（Go の公開ルール）
- `json` タグを付ける（`id`, `title`）

完了条件: `Task` 型がコンパイルできる。

### Task 4: port

`internal/port/task.go` を作る。

- usecase が使うインターフェース `TaskRepository` を定義する
- メソッドは `FindAll(ctx context.Context) ([]domain.Task, error)` だけ

完了条件: インターフェースが domain にだけ依存している。

### Task 5: SQLC の設定とコード生成

SQL を書いて、型付きの Go を生成する。

- リポジトリ直下に `sqlc.yaml` を置く
  - engine: `postgresql`
  - schema: `environments/postgres/01_schema.sql`
  - queries: `internal/driver/postgres/query.sql`
  - 出力先: `internal/driver/postgres/generated`
- `query.sql` にタスク全件取得の `SELECT` を書く
- `sqlc generate` を実行する

完了条件: `generated/` に Go ファイルができ、一覧取得の関数がある。

### Task 6: driver

`internal/driver/postgres/client.go` を作る。

- PostgreSQL へ接続する（`pgx`）
- SQLC 生成コードを呼んで行を取る
- ここでは port を実装しなくてよい。DB アクセスの置き場にする

完了条件: driver から DB の 2 件を取れる。

### Task 7: gateway

`internal/gateway/task.go` を作る。

- driver を受け取る
- `port.TaskRepository` を満たす
- `FindAll` で driver からデータを取り、`[]domain.Task` にして返す

完了条件: `var _ port.TaskRepository = (*TaskGateway)(nil)` が通る。

### Task 8: usecase

`internal/usecase/list_tasks.go` を作る。

- `TaskRepository` を受け取る
- `Execute` で repository の `FindAll` を呼ぶ
- HTTP も driver も SQLC も知らない

完了条件: usecase の import に `net/http` も driver も SQLC 生成パッケージもない。

### Task 9: handler

`internal/handler/task.go` を作る。

- usecase を受け取る
- `List` メソッドで `GET /tasks` を処理する
- usecase の結果を JSON で書き、失敗時は 500 を返す

完了条件: handler が gateway / driver を import していない。

### Task 10: 配線と起動

`cmd/api/main.go` を作る。

- driver → gateway → usecase → handler の順で組み立てる
- `net/http` の ServeMux で `GET /tasks` を登録する
- `:8080` で待ち受ける

完了条件: `go run ./cmd/api` でサーバーが起動する。

### Task 11: 1 path の確認

別ターミナルで:

```bash
curl -i http://localhost:8080/tasks
```

完了条件:

- HTTP ステータスが 200
- seed の 2 件が JSON で返る
- レスポンスが handler → usecase → port → gateway → driver → PostgreSQL を通った結果である
