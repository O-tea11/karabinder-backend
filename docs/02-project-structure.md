# 初期構成の設計

バックエンド（FastAPI）の初期構成を設計したドキュメントです。
**何を作るか・なぜその形にするか・何を使うか**を決めるための検討メモで、実装はこの合意の後に段階的に進めます。

作成日: 2026-11-24

---

## 1. 設計の方針

| 方針 | 理由 |
|---|---|
| **一度に全部作らない** | 学習目的のため。1 フェーズごとに「動く状態」を作って理解してから次へ進む |
| **最初から本番と同じ土台を使う** | SQLite で始めて後から PostgreSQL に移すと、`JSONB` や `pgvector` が使えず書き直しになる。最初から PostgreSQL にする |
| **層を分ける** | 「HTTP の処理」「DB 操作」「データの形」を別ファイルに分けると、どこを直せばよいか迷わない |
| **設定値はコードに書かない** | DB のパスワードや API キーは `.env` に置き、`pydantic-settings` で型付きで読む |
| **重い処理は API プロセスから分離** | 文字起こし・OCR は別プロセス（ワーカー）。ただし Phase 5 までは作らない |

---

## 2. ディレクトリ構成の 2 つの流派

FastAPI は構成を強制しないので、どちらかを選ぶ必要があります。

### (A) レイヤー別（技術的な役割で分ける）

```
app/
├── models/        # DB のテーブル定義を全部ここに
├── schemas/       # API の入出力の形を全部ここに
├── crud/          # DB 操作を全部ここに
└── api/routes/    # エンドポイントを全部ここに
```

### (B) 機能別（ドメインで分ける）

```
app/
├── cards/
│   ├── models.py
│   ├── schemas.py
│   ├── crud.py
│   └── router.py
└── conversations/
    ├── models.py
    └── ...
```

### 比較と結論

| | (A) レイヤー別 | (B) 機能別 |
|---|---|---|
| 向く規模 | 小〜中（〜20 テーブル程度） | 大（チームが機能ごとに分かれる規模） |
| 1 つの機能を追加するとき | 4 つのディレクトリを触る | 1 つのディレクトリで完結 |
| 学習しやすさ | ◎「models と schemas の違い」が構成から分かる | △ 最初は違いが見えにくい |
| 公式テンプレート | こちらが採用（full-stack-fastapi-template） | - |

**→ 今回は (A) レイヤー別を採用します。**
理由は、テーブル数が 6 個程度（organizations / users / cards / conversations / notes / recordings）で小規模なこと、
そして **「SQLAlchemy の model」と「Pydantic の schema」の役割の違いを体で覚えるのが学習目的に合う**ことです。

---

## 3. 完成形のディレクトリ構成

```
karabinder-backend/
├── .env.example              # 環境変数のテンプレート（Git に含める。値はダミー）
├── .env                      # 実際の値（.gitignore で除外。Git に入れない）
├── .gitignore
├── .python-version           # 使う Python のバージョンを固定（uv が読む）
├── pyproject.toml            # 依存ライブラリ・ツール設定を 1 ファイルに集約
├── uv.lock                   # 依存の厳密なバージョン（Git に含める）
├── README.md                 # セットアップ手順
├── docker-compose.yml        # PostgreSQL(pgvector) と Redis をローカルで動かす
├── alembic.ini               # Alembic の設定
├── alembic/
│   ├── env.py                # マイグレーション実行時の設定（app の設定を読むよう改造する）
│   └── versions/             # 自動生成されるマイグレーションファイル
├── app/
│   ├── __init__.py
│   ├── main.py               # ★FastAPI インスタンス生成・ルーター登録・CORS
│   ├── core/
│   │   ├── config.py         # ★環境変数を Settings クラスとして読む
│   │   └── security.py       # パスワードハッシュ・JWT（Phase 5）
│   ├── db/
│   │   ├── base.py           # ★DeclarativeBase（全モデルの親クラス）
│   │   ├── engine.py         # ★engine と SessionLocal の生成
│   │   └── session.py        # ★get_db（Depends で注入する DB セッション）
│   ├── models/               # ★SQLAlchemy = DB のテーブル
│   │   ├── __init__.py
│   │   ├── card.py
│   │   └── conversation.py
│   ├── schemas/              # ★Pydantic = API の入出力
│   │   ├── __init__.py
│   │   ├── card.py
│   │   └── conversation.py
│   ├── crud/                 # ★DB 操作の関数（SELECT/INSERT/UPDATE/DELETE）
│   │   ├── __init__.py
│   │   └── card.py
│   ├── api/
│   │   ├── deps.py           # 共通の依存関係（get_current_user など）
│   │   └── routes/
│   │       ├── __init__.py   # ★全ルーターを束ねる api_router
│   │       ├── health.py     # ★動作確認用
│   │       ├── cards.py
│   │       └── conversations.py
│   └── workers/              # ジョブキューのタスク（Phase 5 以降）
│       ├── __init__.py
│       ├── queue.py          # RQ/Celery の設定
│       └── tasks/
│           ├── ocr.py
│           ├── transcribe.py
│           └── summarize.py
├── tests/
│   ├── conftest.py           # ★テスト用 DB・テストクライアントの準備
│   └── test_health.py
└── docs/                     # 学習ドキュメント（既存）
```

★ = Phase 1〜3 で作るファイル。

---

## 4. リクエストが通る道（各層の責務）

構成を理解する一番の近道は、**1 本のリクエストがどこを通るか**を追うことです。

```
① クライアント（Expo）
      │  POST /cards  {"name": "山田太郎", "company": "株式会社サンプル"}
      ▼
② app/main.py            ← アプリの入口。CORS を通し、api_router に渡す
      ▼
③ app/api/routes/cards.py ← URL と関数の対応。HTTP のことだけを扱う
      │   引数 payload: CardCreate   → ④で検証されたデータが入る
      │   引数 db: DbSession         → ⑤で作られたセッションが入る
      ▼
④ app/schemas/card.py    ← JSON が正しい形か検証し Python オブジェクトに変換
      │                     （形が違えばここで 422 を返して終了。③は呼ばれない）
      ▼
⑤ app/db/session.py      ← get_db が DB セッションを作って渡す（終了後に必ず閉じる）
      ▼
⑥ app/crud/card.py       ← create_card(db, payload) 実際の DB 操作
      ▼
⑦ app/models/card.py     ← Card クラス = cards テーブル。SQL はここから生成される
      ▼
⑧ PostgreSQL
      │  INSERT INTO cards ... RETURNING id
      ▼
⑥→③ 戻ってきた Card オブジェクトを
④ app/schemas/card.py の CardRead に変換（余計な項目を落とす）
      ▼
① クライアントに JSON を返す  {"id": 1, "name": "山田太郎", ...}
```

### なぜ `crud/` を挟むのか（route に直接 SQL を書かない理由）

| 理由 | 具体例 |
|---|---|
| **同じ DB 操作を複数箇所から使う** | 「名刺を 1 件取得」は `GET /cards/{id}`、`PATCH /cards/{id}`、ワーカーの OCR 完了処理、の 3 箇所から必要になる |
| **ワーカーからも使える** | ワーカーは HTTP を通らないので route の関数は呼べないが、`crud` の関数なら呼べる |
| **テストしやすい** | DB 操作だけを HTTP 抜きでテストできる |

> 小規模なら `crud` を省いて route に直接書く流派もあります。
> 今回は**ワーカーから同じ DB 操作を呼ぶ必要が確実にある**ため、最初から分けます。

---

## 5. ツール選定

### 5.1 パッケージ管理：**uv** を採用

| | uv | Poetry | pip + requirements.txt |
|---|---|---|---|
| 速さ | ◎（Rust 製。10〜100倍速い） | △ | ○ |
| ロックファイル | ◎ `uv.lock` | ◎ `poetry.lock` | ✕（`pip freeze` は不正確） |
| Python 本体の管理 | ◎ バージョンもインストールしてくれる | ✕ 別途 pyenv が必要 | ✕ |
| 標準準拠 | ◎ `pyproject.toml` の標準仕様 | △ 独自セクションあり | ✕ |
| Windows | ◎ | ○ | ○ |

**採用理由**: Python 本体のバージョン管理まで uv 1 つで済むのが大きい。Windows でも `pyenv` の導入に悩まずに
`.python-version` を置くだけで同じバージョンが揃う。ML 系ライブラリ（torch など）のインストールも速い。

### 5.2 Lint / Formatter：**Ruff** を採用

従来は **flake8**（lint）+ **black**（整形）+ **isort**（import 並べ替え）の 3 つを入れる必要がありました。
Ruff はこれを **1 つのツールで、Rust 実装で数十倍速く**こなします。設定も `pyproject.toml` に書けます。

```bash
uv run ruff check .      # 問題の検出
uv run ruff format .     # 自動整形
```

### 5.3 型チェック：**mypy** を採用

FastAPI は型ヒント前提のフレームワークなので、型チェッカを入れると**実行前にバグが見つかります**。

| | mypy | pyright |
|---|---|---|
| 歴史・情報量 | ◎ 事実上の標準 | ○ |
| 速さ | △ | ◎ |
| SQLAlchemy 2.0 対応 | ◎（2.0 は型情報を同梱しプラグイン不要） | ◎ |

**採用理由**: 情報量が多く学習に向くため。速さが問題になったら pyright/ty に乗り換えても設定は小さい。

### 5.4 テスト：**pytest** + FastAPI の `TestClient`

```python
def test_health(client):
    res = client.get("/health")
    assert res.status_code == 200
```

`TestClient` は内部で **httpx** を使い、**サーバーを起動せずに**アプリへ直接リクエストを投げます。
そのためテストが速く、ポートの衝突も起きません。

### 5.5 設定読み込み：**pydantic-settings**

```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()   # 起動時に .env を読み、型が違えば即エラーで落ちる
```

**なぜ `os.environ` を直接使わないのか**: `os.environ["DATABASE_URL"]` は全部 `str` で返り、
設定ミスが**実行中に初めて**分かります。pydantic-settings なら**起動時に落ちる**ので、
本番で深夜に気づく事故を防げます。

### 5.6 ローカル DB：**Docker Compose**（PostgreSQL + pgvector、Redis）

```yaml
services:
  db:
    image: pgvector/pgvector:pg17   # PostgreSQL に pgvector が入った公式イメージ
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: karabinder
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:8-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**なぜ Docker か**: Windows に PostgreSQL を直接インストールすると、バージョン差異や
`pgvector` 拡張のビルドでつまずきやすい。`docker compose up -d` だけで
**pgvector 入りの PostgreSQL が即座に揃う**。不要になれば消すだけなので PC が汚れない。

> Docker Desktop が使えない場合は SQLite で Phase 3 まで進める代替案もあります（§7 参照）。

---

## 6. Python のバージョン：**3.13** に固定

PyPI 上の各ライブラリの対応状況を実際に確認しました（2026-11-24 時点）。

| ライブラリ | 最新版 | 対応 Python |
|---|---|---|
| torch | 2.15.0 | >=3.11 |
| faster-whisper | 1.2.1 | >=3.9 |
| **yomitoku** | 0.16.3 | **>=3.10, <3.14** ← 上限あり |
| pyannote.audio | 4.1.1 | >=3.11 |
| fastapi | 0.165.0 | >=3.8 |
| sqlalchemy | 2.1.9 | >=3.8 |

**結論**: OCR 候補の **YomiToku が `<3.14` を要求**しているため、**3.13 が上限**。
3.13 なら全ライブラリが対応しているので **3.13 に固定**します。

これを `.python-version` に書いておくと、uv が自動でその Python を用意します。
**後から ML ライブラリを入れる段階で「Python が新しすぎて入らない」事故を防ぐ**ための事前確認です。

---

## 7. 最初の DB をどうするか（2 案）

| | 案1: Docker で PostgreSQL（推奨） | 案2: まず SQLite |
|---|---|---|
| 準備 | Docker Desktop のインストールが必要 | 不要（Python 標準） |
| pgvector（意味検索） | ◎ そのまま使える | ✕ 使えない |
| `JSONB`・配列型 | ◎ | ✕ 使えない |
| Alembic の学習 | ◎ 本番と同じ挙動 | △ SQLite は `ALTER TABLE` の制約が多く、本番と挙動が違う |
| 後からの移行コスト | なし | モデル定義と一部クエリの書き直しが必要 |

**推奨は案1**。設計書に「PostgreSQL + pgvector」と決まっており、SQLite では
`Vector` 型も `JSONB` も使えないため、どこかで必ず書き直しになります。

---

## 8. Phase 分け（実装の順番）

| Phase | 作るもの | 完了条件（これができたら次へ） | 学べること |
|---|---|---|---|
| **1** | `pyproject.toml`, `app/main.py`, `/health`, `config.py`, `.gitignore`, `README`, Ruff/mypy 設定, 最初のテスト | `uv run fastapi dev` でサーバーが起動し、`/docs` が開けて `/health` が 200 を返す | FastAPI の起動の仕組み、uv、設定の読み込み |
| **2** | `docker-compose.yml`, `db/`（engine・session・Base）, Alembic 初期化, `Card` モデル, 最初のマイグレーション | `alembic upgrade head` で `cards` テーブルが PostgreSQL に作られる | SQLAlchemy 2.0 のモデル定義、マイグレーションの仕組み |
| **3** | `schemas/card.py`, `crud/card.py`, `routes/cards.py`（CRUD 5 本）, テスト | `/docs` から名刺の登録・取得・更新・削除が全部動く | Pydantic スキーマ、依存性注入、HTTP ステータスの使い分け |
| **4** | `Conversation`・`Note` モデルとリレーション、ネストしたレスポンス | `GET /cards/{id}` が紐づく会話記録を含めて返す | リレーション（1対多）、N+1 問題と `selectinload` |
| **5** | 認証（users/organizations）、画像・音声アップロード、S3 互換ストレージ、Redis + RQ、ワーカー | 名刺画像をアップロードするとジョブが積まれ、`/jobs/{id}` で状態が見える | 認証、ファイルアップロード、非同期ジョブ |
| **6** | OCR・文字起こし・要約の実処理、pgvector での意味検索 | 名刺画像から項目が抽出され、会話の要約とタグが付く | 外部ライブラリ連携、ベクトル検索 |

**Phase 1 は意図的に DB を触りません。** 先に「FastAPI が起動して `/docs` が出る」最小状態を作り、
FastAPI 単体の理解を固めてから DB を足します。問題が起きたときの切り分けも楽になります。

---

## 9. 命名・設計のルール（最初に決めておく）

| 項目 | ルール | 理由 |
|---|---|---|
| テーブル名 | 複数形・スネークケース（`cards`, `conversations`） | SQL の慣習 |
| モデルのクラス名 | 単数形・パスカルケース（`Card`） | 1 行 = 1 件を表すため |
| スキーマの命名 | `CardCreate` / `CardUpdate` / `CardRead` | 用途が名前から分かる |
| 主キー | `id`（整数の自動採番）※外部公開 ID は Phase 5 で UUID 検討 | 単純で join しやすい |
| 日時 | `created_at` / `updated_at` を全テーブルに、**タイムゾーン付き**で持つ | イベントの時刻が重要なアプリなので UTC で保存し表示時に変換 |
| 削除 | Phase 3 では物理削除。共有機能を入れる Phase 5 で論理削除（`deleted_at`）を検討 | 誤削除の復旧が必要になる可能性 |
| API のパス | 複数形・ケバブケース（`/cards`, `/conversations/{id}/draft-message`） | REST の一般的な慣習 |

---

## 10. 決定事項（2026-11-24 合意）

| 項目 | 決定 | 根拠 |
|---|---|---|
| パッケージ管理 | **uv** | Python 本体のバージョン管理まで 1 つで済み、Windows で pyenv が不要（§5.1） |
| ローカル DB | **Docker Compose で PostgreSQL（pgvector イメージ）** | 本番と同じ環境で Alembic を学べ、pgvector / JSONB が最初から使える（§7 案1） |
| Python バージョン | **3.13** | YomiToku が `<3.14` を要求（§6） |
| ディレクトリ構成 | **レイヤー別** | テーブル数が小規模で、model と schema の役割の違いが構成から見える（§2） |
| 実装の順番 | **Phase 1 から順に** | FastAPI 単体を動かしてから DB を足す（§8） |

### 事前に必要なもの（ローカル環境）

```powershell
winget install --id astral-sh.uv       # uv の導入
winget install --id Docker.DockerDesktop   # Docker Desktop の導入（未導入の場合）
```

導入後、`uv --version` と `docker --version` が応答すれば準備完了です。

---

## 次のステップ

- Phase 1 の実装（ファイル 8 個程度。1 つずつ「何のためのファイルか」を説明しながら作る）
