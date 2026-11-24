# karabinder-backend

名刺管理アプリ（会話メモ付き）のバックエンドです。
イベントでの名刺交換時の会話を録音・文字起こしし、名刺情報と紐づけて「誰とどんな話をしたか」を残すことを目的としています。

**構成**: FastAPI(バックエンド) + PostgreSQL/pgvector + Expo(React Native、フロントエンド)

| リポジトリ | 内容 |
|---|---|
| **karabinder-backend**（このリポジトリ） | FastAPI / SQLAlchemy / Alembic。REST API を提供する |
| [karabinder-frontend](https://github.com/ad2605at-ctrl/karabinder-frontend) | Expo(React Native) / TypeScript。アプリの画面 |

フロントとバックエンドは REST API(JSON) で連携し、FastAPI が出力する OpenAPI 仕様からフロント側の TypeScript 型を生成します。

> **現在の状況**: 設計フェーズです。`docs/` に学習・設計ドキュメントがあり、アプリ本体（`app/`）はこれから Phase 1 から順に実装します。
> そのため以下の「セットアップ」「テスト・Lint の実行」は Phase 1 実装後に有効になります。

---

## ディレクトリ構成

```
karabinder-backend/
├── .env.example              # 環境変数のテンプレート(値はダミー。Git に含める)
├── .python-version           # 使う Python のバージョンを固定(uv が読む)
├── pyproject.toml            # 依存ライブラリ・ツール設定
├── uv.lock                   # 依存の厳密なバージョン(Git に含める)
├── docker-compose.yml        # PostgreSQL(pgvector) と Redis をローカルで起動
├── alembic.ini               # マイグレーション設定
├── alembic/versions/         # マイグレーションファイル
├── app/
│   ├── main.py               # FastAPI インスタンス生成・ルーター登録・CORS
│   ├── core/                 # 設定読み込み(config.py)・認証基盤(security.py)
│   ├── db/                   # engine / SessionLocal / get_db / DeclarativeBase
│   ├── models/               # SQLAlchemy モデル(= DB のテーブル)
│   ├── schemas/              # Pydantic スキーマ(= API の入出力)
│   ├── crud/                 # DB 操作の関数
│   ├── api/
│   │   ├── deps.py           # 共通の依存関係(get_current_user など)
│   │   └── routes/           # エンドポイント(health, cards, conversations...)
│   └── workers/              # ジョブキューのタスク(OCR・文字起こし・要約)
├── tests/
└── docs/                     # 学習・設計ドキュメント
```

各層の責務とリクエストが通る道は [docs/02-project-structure.md](docs/02-project-structure.md) を参照してください。

### なぜ `models/` と `schemas/` を分けるのか

`models/` は DB のテーブル構造（SQLAlchemy）、`schemas/` は API でやり取りする形（Pydantic）です。
両者を分けることで、DB に持っているが API では返したくない項目を自然に除外できます。

---

## ドキュメント

| No. | ドキュメント | 内容 |
|---|---|---|
| 01 | [FastAPI まとめ](docs/01-fastapi.md) | フレームワークの選定理由、Starlette / Pydantic / ASGI、依存性注入、async の使い分け |
| 02 | [初期構成の設計](docs/02-project-structure.md) | ディレクトリ構成、各層の責務、ツール選定、Phase 分け |

---

## 前提条件

ローカルで動かす場合は以下が必要です。

- **Python 3.13**（uv が自動で用意するので手動インストールは不要）
  - 3.14 以上は使えません。OCR 候補の YomiToku が `<3.14` を要求しているためです。
- **uv**（パッケージ管理）
- **Docker / Docker Compose**（PostgreSQL と Redis をローカルで動かすため）
- **Git**

```powershell
winget install --id astral-sh.uv
winget install --id Docker.DockerDesktop
```

導入後、`uv --version` と `docker --version` が応答すれば準備完了です。

> **なぜ uv か**: Python 本体のバージョン管理まで uv 1 つで済むため、Windows でも pyenv を別途入れる必要がありません。
> インストールも Rust 実装で高速です（詳細は [docs/02-project-structure.md](docs/02-project-structure.md) §5.1）。

---

## セットアップ

### 1. データベースを起動する

```bash
docker compose up -d
```

PostgreSQL（pgvector 入り）が `localhost:5432`、Redis が `localhost:6379` で起動します。
`docker compose ps` で状態、`docker compose logs db` でログを確認できます。

### 2. 依存ライブラリをインストールする

```bash
uv sync
```

`uv.lock` に記録された**厳密なバージョン**で環境が再現されます。Python 3.13 も uv が自動で用意します。

### 3. 環境変数を設定する

```bash
cp .env.example .env
```

`.env` には DB の接続先や API キーを書きます。**`.env` は絶対にコミットしないでください**（`.gitignore` 済み）。

### 4. マイグレーションを適用する

```bash
uv run alembic upgrade head
```

### 5. 開発サーバーを起動する

```bash
uv run fastapi dev app/main.py
```

| URL | 内容 |
|---|---|
| http://localhost:8000/health | 疎通確認 |
| http://localhost:8000/docs | Swagger UI。ブラウザから API を試せる |
| http://localhost:8000/openapi.json | OpenAPI 仕様（フロントの型生成に使う） |

### フロントエンド側の型生成

フロント担当は、起動中のバックエンドから TypeScript の型を生成できます。

```bash
npx openapi-typescript http://localhost:8000/openapi.json -o src/types/api.d.ts
```

これによりバックエンドの変更がフロントの型エラーとして現れ、仕様のズレを防げます。

---

## テスト・Lint の実行

PR を出す前に、CI と同じチェックをローカルで流しておくと手戻りが減ります。

```bash
uv run ruff check .          # Lint(問題の検出)
```

```bash
uv run ruff format .         # 自動整形
```

```bash
uv run mypy app              # 型チェック
```

```bash
uv run pytest                # テスト
```

> **Ruff** は従来の flake8(lint) + black(整形) + isort(import 並べ替え) を 1 つにまとめたツールです。
> Rust 実装で数十倍速く、設定も `pyproject.toml` に集約できます。

---

## チーム内での約束事

- `main` ブランチへの直 push は禁止。作業は feature ブランチを切り、Pull Request 経由でマージする。
- PR のタイトル・本文・コミットメッセージの書き方は「[PR とコミットメッセージの書き方](#pr-とコミットメッセージの書き方)」に従う（本文が空の PR は不可）。
- PR を作成すると GitHub Actions の CI（`.github/workflows/ci.yml`）が自動実行され、lint / 型チェック / テストが通ることを確認してからマージする。
- CI を発火させたくない変更（ドキュメントのみ等）は、コミットメッセージに `[skip ci]` を含めると GitHub Actions の実行をスキップできる。
- 大きいバイナリファイルはリポジトリにコミットしない（必要になった時に `.gitignore` へ追記する）。
  - 具体例: 文字起こしモデルのキャッシュ（faster-whisper の `large-v3-turbo` など数 GB）、OCR モデル、録音した音声ファイル、名刺画像。
  - 画像・音声は S3 互換のオブジェクトストレージに置き、DB には URL のみ保存する方針。
- branch の命名規則は `(name)/(変更箇所)`。
  - 例: `nakai/update-readme`、`nakai/cards-crud`
- 環境変数（API キーなど秘匿情報）は `.env` に置き、コミットしない。
  - 新しい環境変数を追加したら、**値を書かずに** `.env.example` にキーだけ追記して共有する。

### このプロジェクト特有の注意

- **会話の録音データ・文字起こしテキストはリポジトリに入れない。** 第三者の個人情報を含むため、
  テスト用の音声も実際の会話ではなくダミーを使う。
- 外部 API に音声やテキストを送る実装を追加する場合は、PR の `## 設計メモ / スコープ外` に
  「何を・どこへ送るか」を明記する（個人情報保護法の確認対象になるため）。

---

## 開発の流れ

初めて開発に参加する場合は、大まかに以下の流れになります。

1. GitHub の Issues から着手するタスクを選ぶ（担当が決まっていないものを選ぶか、チームに確認する）。
2. `main` を最新化してから、命名規則（`(name)/(変更箇所)`）に沿ってブランチを切る。

```bash
git checkout main
git pull origin main
git checkout -b (name)/(変更箇所)
```

3. 上記の「セットアップ」を参考にローカルで起動し、動作を確認しながら実装する。
4. コミット前に「テスト・Lint の実行」のコマンドをローカルで流し、エラーがないことを確認する。
5. 変更を push し、GitHub 上で Pull Request を作成する（対象は `main`）。本文は自動で入る雛形（`.github/pull_request_template.md`）を埋める。詳細は「[PR とコミットメッセージの書き方](#pr-とコミットメッセージの書き方)」。
6. PR 作成をトリガーに CI が自動実行されるので、通過を確認する。ドキュメントのみの変更など CI を回す必要がない場合は、コミットメッセージに `[skip ci]` を含める。
7. レビューを受けてマージする。

わからないことがあれば、遠慮せずチームに質問してください。

---

## PR とコミットメッセージの書き方

レビューと、あとから履歴を追う人（数週間後の自分を含む）のために、PR とコミットメッセージには「何を・なぜ」を残します。
**本文が空の PR・要約だけのコミットメッセージはレビューできないので不可**です。

### PR タイトル

`<領域>: <日本語の要約> (#Issue番号)` の形式にします。

| 領域 | 対象 |
|---|---|
| `api:` | エンドポイント・スキーマの追加や変更（`app/api/`、`app/schemas/`） |
| `db:` | モデル・マイグレーション（`app/models/`、`alembic/`） |
| `worker:` | ジョブキュー・OCR・文字起こし・要約（`app/workers/`） |
| `infra:` | Docker・CI・依存ライブラリ・設定（`docker-compose.yml`、`pyproject.toml`） |
| `docs:` | ドキュメントのみの変更 |

横断的な変更は `feat:` / `fix:` / `refactor:` などでも可です。

- 例: `api: 名刺の CRUD エンドポイントを追加 (#12)`
- 例: `db: conversations テーブルに embedding カラムを追加 (#21)`
- 例: `worker: 録音チャンクの文字起こしジョブを追加 (#33)`

### PR 本文

新しい PR を作ると `.github/pull_request_template.md` の雛形が入るので、それを埋めます。

| セクション | 必須 | 内容 |
|---|---|---|
| `Closes #NN` | ○ | 対応する Issue。複数なら複数行。これで Issue が自動クローズ・相互リンクされる |
| `## 概要` | ○ | 何を・なぜ。親 Issue との関係。1〜3 文 |
| `## 変更点` | ○ | 主要な変更をファイル / 観点ごとに箇条書き。新規ファイルは `(新規)`。sub-issue をまとめる PR はチェックリストでも可 |
| `## 設計メモ / スコープ外` | 任意 | 自明でない実装判断・既知の制約・今回扱わない範囲・follow-up issue 番号。レビューの疑問を先回りで潰す |
| `## 動作確認` | ○ | 実際に流したチェックとその結果。CI と同じチェック（下記）＋手動確認した手順・結果 |

- レビュー指摘に対応したら、対応内容と対応しなかったもの（＋理由）を PR にコメントで残します。
- 別 issue に切り出したものは PR 本文か commit メッセージに issue 番号を書きます（例: `follow-up: ページネーション未対応 → #117`）。
- API の形が変わる PR では、`## 変更点` にエンドポイントとリクエスト / レスポンスの差分を書きます。
  フロント側の型生成に影響するため、**フロント担当がレビューで気づけるようにする**ことが目的です。

### コミットメッセージ

- 1 行目: `<領域>: <要約>`（PR タイトルと同じ書式。`#Issue番号` は本文の `Closes #NN` に寄せてもよい）
- 本文: 変更の理由・背景・非自明な判断を書く。差分を読めば分かる「何をしたか」だけでなく「**なぜそうしたか**」を残す
- 対応 Issue は本文に `Closes #NN`
- WIP の細かいコミットを積んだ PR でも、少なくとも PR 本文はまとめて上記の体裁にする

```
api: 名刺一覧に業界での絞り込みを追加

課題感での検索(#12)の前段として、まず業界による単純な絞り込みを実装した。
意味検索(pgvector)は埋め込みモデルの選定が終わっていないため別 PR とする。

follow-up: 課題感での意味検索 → #45

Closes #38
```

### CI をスキップしたいとき

ドキュメントのみの変更など CI を回す必要がない場合は、コミットメッセージに `[skip ci]` を含めます。

---

## CI

`.github/workflows/ci.yml` で以下を自動実行します（`main` ブランチへの push および全ブランチからの Pull Request がトリガー）。
デプロイ（CD）は現時点では対象外で、まずローカルで動くことを優先します。

- Ruff による lint・フォーマット確認（`ruff check` / `ruff format --check`）
- mypy による型チェック
- pytest によるテスト

フロントエンド（ESLint・型チェック・テスト）は [karabinder-frontend](https://github.com/ad2605at-ctrl/karabinder-frontend) 側の CI で実行します。

> CI の設定ファイルは、Phase 1 で `pyproject.toml` と最初のテストを作った後に追加します。
> チェック対象が存在しない状態で CI を置いても意味がないためです。

---

## 困ったときは

- `uv: command not found` → uv が入っていないか、PATH が通っていません。PowerShell を再起動して `uv --version` を試してください。
- `docker: command not found` / `Cannot connect to the Docker daemon` → Docker Desktop が起動していません。タスクトレイのクジラのアイコンが動いている状態か確認してください。
- DB に繋がらない（`connection refused`） → `docker compose ps` でコンテナが `running` か確認。`.env` の `DATABASE_URL` のポートが `docker-compose.yml` と一致しているかも確認してください。
- `alembic upgrade head` でエラー → DB が起動しているか、`alembic/versions/` に未適用のマイグレーションが残っていないか確認してください。
- ポートが使用中というエラー → 既に起動している `fastapi dev` や `docker compose` が残っていないか確認して停止してください（`docker compose down`）。
- `/docs` が開けない → サーバーの起動ログにエラーが出ていないか確認してください。`.env` の値が足りないと起動時点で落ちます（pydantic-settings が型検証するため）。
