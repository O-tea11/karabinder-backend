# FastAPI まとめ（バックエンドフレームワークの選定理由と基礎）

このドキュメントは、名刺管理アプリ（会話メモ付き）のバックエンドに **FastAPI** を採用した理由と、
その土台になっている仕組みを理解するための学習メモです。

作成日: 2026-11-24 / 対象バージョン: FastAPI 0.165.0, Pydantic 2.14.x, Starlette 1.10.x, Uvicorn 0.54.x

---

## 1. FastAPI とは何か

**FastAPI は、Python で「Web API」を作るためのフレームワーク**です。
HTML ページを返す Web サイトを作るためのもの（Django のような全部入りフレームワーク）ではなく、
スマホアプリや React から呼ばれる **JSON を返すサーバー** を作ることに特化しています。

今回のアプリは

```
Expo アプリ  ──(HTTP + JSON)──>  FastAPI  ──>  PostgreSQL
```

という構成で、画面は Expo 側が描くので、サーバーは JSON を返すだけでよい。
この形にぴったり合うのが FastAPI です。

### 一番の特徴：「Python の型ヒントがそのまま仕様になる」

FastAPI の設計思想は一言でいうと **「関数の型ヒントを書けば、それ以外は全部フレームワークがやる」** です。

```python
@app.get("/cards/{card_id}")
def get_card(card_id: int) -> CardRead:
    ...
```

この `card_id: int` と `-> CardRead` という **型ヒント** を書いただけで、FastAPI は次の 4 つを自動でやってくれます。

| 自動でやってくれること | 普通のフレームワークだと |
|---|---|
| URL の `{card_id}` を取り出して `int` に変換 | 自分で `int(request.args["card_id"])` と書く |
| 変換できなければ 422 エラーを JSON で返す | 自分で try/except を書く |
| レスポンスを `CardRead` の形に整える・余計な項目を落とす | 自分で dict を組み立てる |
| OpenAPI（API 仕様書）を生成する | 別途 Swagger 用の YAML を手書きする |

「型を書く」＝「バリデーションとドキュメントを書く」ことになるので、**同じことを二度書かなくて済む**。
これが FastAPI を使う最大の理由です。

---

## 2. このプロジェクトで FastAPI を選んだ理由

| 理由 | 詳細 |
|---|---|
| **① OpenAPI が自動生成される** | フロント担当（Expo/TypeScript）が別メンバーなので、API の形を文書で合意する必要がある。FastAPI は `/docs` に仕様書を自動で出し、そこから `openapi-typescript` で **TypeScript の型を生成**できる。手書きの仕様書とコードがズレる問題が起きない。 |
| **② Pydantic でバリデーションできる** | LLM の出力（要約・タグ付けの JSON）を検証する方針になっている。FastAPI は Pydantic が標準で組み込まれているので、**API の入出力と LLM 出力の検証を同じ道具で書ける**。 |
| **③ 非同期処理（async）が標準** | 名刺画像のアップロード、音声チャンクのアップロード、外部 LLM API の呼び出しなど **待ち時間が長い I/O** が多い。async なら「待っている間に他のリクエストを処理」できる。 |
| **④ Python である必要がある** | 文字起こし（faster-whisper）、OCR（YomiToku）、話者分離（pyannote.audio）、ベクトル化はすべて Python のライブラリ。**AI 処理とサーバーを同じ言語で書ける**ことが決定的に重要。 |
| **⑤ 学習コストが低い** | 「関数に型を書いてデコレータを付ける」だけで動く。Django のような独自概念（settings, apps, middleware の作法）が少ない。 |

---

## 3. FastAPI は何の上に乗っているのか（3層構造）

FastAPI は一からすべてを作っているのではなく、**2 つのライブラリを組み合わせた薄い層**です。
これを知っておくと、エラーメッセージを読むときに「これは誰が出したエラーか」が分かります。

```
┌─────────────────────────────────────────────┐
│  FastAPI                                     │  ← 型ヒントを読んで両者を繋ぐ層
│  ・ルーティングの定義（@app.get）              │
│  ・依存性注入（Depends）                      │
│  ・OpenAPI 生成                              │
├──────────────────────┬──────────────────────┤
│  Starlette            │  Pydantic            │
│  (Web の部分)          │  (データ検証の部分)    │
│  ・HTTP リクエスト/     │  ・型に沿っているか検証 │
│    レスポンス          │  ・JSON ⇄ Python     │
│  ・ルーティング         │    オブジェクト変換    │
│  ・ミドルウェア         │  ・エラーメッセージ生成 │
│  ・WebSocket          │                      │
├──────────────────────┴──────────────────────┤
│  ASGI サーバー（Uvicorn）                      │  ← 実際に TCP を listen する人
└─────────────────────────────────────────────┘
```

### 3.1 Starlette ─ Web の土台

- **役割**: HTTP リクエストを受け取り、レスポンスを返す部分。ルーティング、ミドルウェア、CORS、WebSocket、静的ファイル配信。
- `from fastapi import Request` の `Request` は、実体は Starlette の `Request` です。
- **なぜ有用か**: HTTP の低レベルな処理（ヘッダのパース、ボディのストリーミング受信など）を自分で書かなくて済む。
  今回は **音声チャンクのアップロード**（大きなファイル）があるので、ストリーミングでボディを受け取れるのは重要。

### 3.2 Pydantic ─ データ検証の土台

- **役割**: 「このデータはこの形をしているか」を検証し、していれば Python オブジェクトに変換する。
- Pydantic v2 は**検証処理の中核が Rust で書かれている**（pydantic-core）ので非常に速い。v1 から 5〜50 倍速くなっている。
- **なぜ有用か**: 外から来るデータは信用できない。「`email` は本当にメールアドレスの形か」「`phone` は文字列か」を
  **1 箇所（スキーマ定義）に書くだけ**で全エンドポイントに効く。
- **今回の使いどころ**:
  1. API のリクエストボディ検証（名刺の編集内容など）
  2. API のレスポンス整形（DB モデルから必要な項目だけ返す）
  3. **LLM の出力検証**（要約・タグの JSON が期待した形か。これが一番ありがたい）
  4. 設定値の読み込み（`pydantic-settings` で環境変数を型付きで読む）

### 3.3 ASGI と Uvicorn ─ 実行する人

**ASGI (Asynchronous Server Gateway Interface)** は「Python の Web アプリとサーバーの間の約束事」です。

| | WSGI（古い規格） | ASGI（新しい規格） |
|---|---|---|
| 代表的なフレームワーク | Flask, Django(〜3) | FastAPI, Starlette, Django(4+) |
| 非同期処理 | できない | できる |
| WebSocket | できない | できる |
| 1リクエストの扱い | 1スレッドが占有して待つ | 待ち時間に他を処理できる |

**Uvicorn** はその ASGI を実装したサーバーで、`uvicorn main:app` で起動します。
内部で `uvloop`（C で書かれた高速なイベントループ）と `httptools` を使うため速い。

> **なぜ「サーバー」が別に必要？**
> FastAPI は「リクエストが来たらこう返す」というルールの集まりでしかなく、
> 実際にポートを開いて TCP 接続を受け付けるのは Uvicorn の仕事です。
> 本番では `gunicorn` でワーカープロセスを複数管理し、各ワーカーが uvicorn で動く構成も使われます。

---

## 4. 基本の書き方（1行ずつ解説）

```python
# ① FastAPI 本体と、エラーを返すためのクラスをインポート
from fastapi import FastAPI, HTTPException

# ② Pydantic の BaseModel = 「データの形」を定義するための基底クラス
from pydantic import BaseModel

# ③ アプリの本体を作る。title/version はそのまま OpenAPI の情報になる
app = FastAPI(title="Karabinder API", version="0.1.0")


# ④ リクエストで受け取るデータの形（入力用スキーマ）
class CardCreate(BaseModel):
    name: str                 # 必須（省略すると 422 エラー）
    company: str | None = None  # 省略可能（デフォルト None）


# ⑤ レスポンスで返すデータの形（出力用スキーマ）
class CardRead(BaseModel):
    id: int
    name: str
    company: str | None = None


# ⑥ デコレータ：「POST /cards」というエンドポイントとして登録する
#    status_code=201 → 作成に成功したら HTTP 201 Created を返す
@app.post("/cards", status_code=201)
def create_card(payload: CardCreate) -> CardRead:
    #  ↑ payload の型が Pydantic モデル → FastAPI は「これはリクエストボディだ」と判断し、
    #    JSON を読んで CardCreate に変換してから関数を呼ぶ。
    #    形が合わなければ関数は呼ばれず、422 とエラー内容の JSON が返る。
    new_id = 1  # 実際は DB に INSERT して採番された id
    return CardRead(id=new_id, name=payload.name, company=payload.company)


@app.get("/cards/{card_id}")
def get_card(card_id: int) -> CardRead:
    #  ↑ card_id は URL の中にある名前と一致 → FastAPI は「パスパラメータだ」と判断する
    #    /cards/abc のようなアクセスは int に変換できないので 422 が返る
    if card_id != 1:
        # 意図的にエラーを返したいときは HTTPException を raise する
        raise HTTPException(status_code=404, detail="Card not found")
    return CardRead(id=1, name="山田太郎", company="株式会社サンプル")
```

### 引数の型で「どこから来た値か」が決まる

FastAPI は関数の引数を見て、値の取得元を自動判別します。これが独特なので覚えておくべき点です。

| 引数の書き方 | FastAPI の解釈 |
|---|---|
| 名前が URL の `{...}` と一致 | **パスパラメータ**（`/cards/{card_id}` → `card_id: int`） |
| `int`/`str`/`bool` などの単純な型で、URL に無い名前 | **クエリパラメータ**（`/cards?q=山田` → `q: str`） |
| Pydantic モデル | **リクエストボディ**（JSON） |
| `UploadFile` | **アップロードされたファイル**（multipart/form-data） |
| `Depends(...)` | **依存性注入**（後述） |

---

## 5. Pydantic スキーマを「入力用」「出力用」で分ける理由

今回のような実務的な構成では、**DB のモデル（SQLAlchemy）と API のスキーマ（Pydantic）を分ける**のが定石です。

```
 SQLAlchemy の Card     ←→    Pydantic の CardCreate / CardRead / CardUpdate
 （DB のテーブル構造）           （API でやり取りする形）
```

なぜ分けるのか:

1. **返してはいけない項目があるから**。`cards` テーブルに登録ユーザーの内部 ID や原本画像の内部パスが
   あっても、API では返したくない。出力用スキーマに書かなければ自動的に除外される。
2. **入力と出力で必要な項目が違うから**。作成時に `id` は要らないが、返すときは必要。
3. **編集（PATCH）は全項目が省略可能だから**。`CardUpdate` は全フィールドを `| None = None` にする。

```python
class CardBase(BaseModel):
    name: str
    company: str | None = None
    department: str | None = None

class CardCreate(CardBase):
    pass                        # 作成時は id なし

class CardUpdate(BaseModel):    # 編集時は全部省略可
    name: str | None = None
    company: str | None = None
    department: str | None = None

class CardRead(CardBase):
    id: int
    created_at: datetime
    # SQLAlchemy のオブジェクトをそのまま渡せるようにする設定
    model_config = {"from_attributes": True}
```

`from_attributes=True` は「dict ではなく、属性アクセスできるオブジェクト（= SQLAlchemy のモデル）からも
変換してよい」という指定です。これがないと `CardRead.model_validate(card_orm)` が失敗します。
（Pydantic v1 では `orm_mode = True` という名前でした。）

---

## 6. 依存性注入（Depends）── FastAPI で最も重要な機能

**依存性注入 (Dependency Injection)** は「関数が必要なものを、関数の外から渡してもらう」仕組みです。
FastAPI ではこれが `Depends` 一つで書けます。

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session

def get_db():
    """DB セッションを作って渡し、処理が終わったら必ず閉じる"""
    db = SessionLocal()
    try:
        yield db          # ← ここで関数に渡される
    finally:
        db.close()        # ← レスポンスを返した後に必ず実行される

# Annotated を使う書き方（現在の推奨）
DbSession = Annotated[Session, Depends(get_db)]

@app.get("/cards")
def list_cards(db: DbSession) -> list[CardRead]:
    return db.query(Card).all()
```

### なぜこれが嬉しいのか

| 用途 | 書くもの | 効果 |
|---|---|---|
| **DB セッション管理** | `get_db` | 全エンドポイントで「開いて閉じる」を書かなくて済む。閉じ忘れによる接続リークが起きない |
| **認証** | `get_current_user` | `user: CurrentUser` と引数に書くだけでログイン必須になる。トークン検証コードは 1 箇所 |
| **権限チェック** | `require_same_org` | 「同じ組織の名刺しか見られない」を全エンドポイントに同じ形で適用できる（社内共有の要件に直結） |
| **テスト** | `app.dependency_overrides` | 本物の DB を使う `get_db` を、テスト用 DB に**差し替えられる**。これが依存性注入の最大の利点 |

```python
# テストでの差し替え例：本物の DB の代わりにテスト用 DB を使わせる
app.dependency_overrides[get_db] = get_test_db
```

`yield` を使った依存関係は、**レスポンスを返した後に後片付けが走る**という点が重要です。
Python の `contextmanager` と同じ考え方で、「前処理 → 本処理 → 後処理」を 1 つの関数に書けます。

---

## 7. async と def の使い分け（このプロジェクトで特に重要）

FastAPI はエンドポイントを `def`（同期）でも `async def`（非同期）でも書けますが、**挙動が違います**。

| 書き方 | 実行場所 | 向いている処理 |
|---|---|---|
| `async def` | メインのイベントループ上で直接実行 | `await` できる非同期 I/O（httpx での外部 API 呼び出し、非同期 DB ドライバ） |
| `def` | **別のスレッドプール**に逃がして実行 | 同期的なライブラリを使う処理（同期版 SQLAlchemy、OpenCV など） |

### 絶対にやってはいけないこと

```python
# ❌ 悪い例：async def の中でブロッキング処理をする
@app.post("/transcribe")
async def transcribe(file: UploadFile):
    text = whisper_model.transcribe(path)   # CPU を数十秒占有する
    return {"text": text}
```

`async def` の中で「待つ」処理を `await` なしで書くと、**イベントループ全体が止まり、
他の全リクエストが応答しなくなります**。これが FastAPI で一番よくある落とし穴です。

今回のアプリは文字起こしが **音声 1 分あたり 150 秒** かかる想定なので、
そもそも API プロセス内で実行してはいけません。設計方針どおり:

```
FastAPI（受け付けるだけ。すぐ 202 を返す）
   ↓ ジョブをキューに入れる
Redis / RQ・Celery
   ↓
ワーカープロセス（別プロセス・別コンテナ。ここで文字起こし）
```

とします。FastAPI 側の役割は「ファイルを保存してジョブ ID を返す」だけなので一瞬で終わります。

### `BackgroundTasks` ではなくジョブキューを使う理由

FastAPI には `BackgroundTasks` という簡易な後処理機能がありますが、**同じプロセス内**で動きます。つまり

- API サーバーの CPU を奪う（文字起こしは CPU を使い切る）
- プロセスが再起動すると**処理が消える**（リトライできない）
- 進捗を外から確認できない

今回は「夜間に処理して翌朝までに完了」「`GET /jobs/{id}` で状態確認」という要件があるので、
**Redis に状態を持つジョブキュー（RQ か Celery）が必須**です。
`BackgroundTasks` は「アップロード完了後に一時ファイルを削除する」程度の軽い用途に留めます。

---

## 8. 自動ドキュメント（OpenAPI）とフロント連携

アプリを起動すると、**何も書かなくても**以下が使えます。

| URL | 内容 |
|---|---|
| `/docs` | Swagger UI。ブラウザから実際にリクエストを送って試せる |
| `/redoc` | ReDoc。読み物として見やすい仕様書 |
| `/openapi.json` | 機械が読む用の OpenAPI 仕様（JSON） |

この `/openapi.json` が**フロント連携の要**になります。

```bash
# バックエンドの仕様から TypeScript の型を生成する
npx openapi-typescript http://localhost:8000/openapi.json -o src/types/api.d.ts
```

これで Expo 側は `CardRead` 型を手で書く必要がなくなり、
**バックエンドの変更がフロントの型エラーとして現れる**ようになります。
「10. 最初に取り組むこと」の 4 番（API 仕様をフロント担当と共有）はこの仕組みで実現します。

> `orval` を使うと型だけでなく、React Query 用の呼び出し関数まで生成できます。
> まずは `openapi-typescript` でシンプルに始めて、必要になったら検討するのが良いです。

---

## 9. 他のフレームワークとの比較（なぜ FastAPI なのか）

| フレームワーク | 特徴 | 今回採用しない理由 |
|---|---|---|
| **Flask** | 最小限。自由度が高い | バリデーション・OpenAPI・async すべて別ライブラリを組み合わせる必要がある。結果的に FastAPI を自作することになる |
| **Django + Django REST Framework** | 管理画面・認証・ORM が全部入り。実績が最も多い | 今回は画面を Expo が持つので管理画面の価値が低い。Serializer は型ヒントベースでなく記述量が多い。OpenAPI 生成も後付け |
| **Django Ninja** | Django の上で FastAPI 風に書ける | Django の重さを引き受けることになる。今回 Django の資産（admin, auth）をあまり使わない |
| **Litestar** | FastAPI に似た後発。DI がより強力で性能も良い | コミュニティ・事例・ドキュメントの量が FastAPI に劣る。学習目的なら情報が多い方が良い |
| **FastAPI** | 型ヒント駆動、OpenAPI 自動生成、async 対応 | ✅ **採用** |

### 補足：Django の管理画面が欲しくなったら

「社内共有」「アクセス制御」の要件が膨らんで管理画面が欲しくなった場合、
FastAPI でも **SQLAdmin**（SQLAlchemy ベースの管理画面ライブラリ）を後から足せます。
最初から Django にする必要はありません。

---

## 10. ディレクトリ構成の定石

FastAPI は構成を強制しないので、自分で決める必要があります。
今回の規模（名刺・会話・録音・ジョブ）なら、**役割で分ける**構成が扱いやすいです。

```
karabinder-backend/
├── app/
│   ├── main.py              # FastAPI インスタンス生成、ルーター登録、CORS 設定
│   ├── core/
│   │   ├── config.py        # 環境変数の読み込み（pydantic-settings）
│   │   └── security.py      # パスワードハッシュ、トークン生成
│   ├── db/
│   │   ├── base.py          # SQLAlchemy の Base, engine, SessionLocal
│   │   └── session.py       # get_db（Depends で使う）
│   ├── models/              # SQLAlchemy モデル（= DB のテーブル）
│   │   ├── card.py
│   │   └── conversation.py
│   ├── schemas/             # Pydantic スキーマ（= API の入出力）
│   │   ├── card.py
│   │   └── conversation.py
│   ├── api/
│   │   ├── deps.py          # 共通の依存関係（get_current_user など）
│   │   └── routes/
│   │       ├── cards.py     # /cards 系のエンドポイント
│   │       └── conversations.py
│   ├── crud/                # DB 操作の関数（モデルとルートの間の層）
│   └── workers/             # ジョブキューのタスク（OCR・文字起こし・要約）
├── alembic/                 # マイグレーション
├── tests/
└── pyproject.toml
```

ポイント: **`models`（DB の形）と `schemas`（API の形）を別ディレクトリにする**。
名前が同じ `Card` で混ざるとバグの原因になるので、最初から分けておきます。

`APIRouter` を使うと、エンドポイントをファイル分割できます。

```python
# app/api/routes/cards.py
from fastapi import APIRouter
router = APIRouter(prefix="/cards", tags=["cards"])

@router.get("")          # 実際のパスは /cards
def list_cards(): ...

# app/main.py
from app.api.routes import cards
app.include_router(cards.router)
```

`tags` は `/docs` 上でエンドポイントをグループ化するための見出しになります。

---

## 11. このプロジェクトの API を FastAPI でどう書くか（対応表）

| 設計書の API | FastAPI での実装イメージ | 使う機能 |
|---|---|---|
| `GET /cards`（検索） | クエリパラメータ `q`, `company`, `industry`, `from`, `to` を引数に取る | クエリパラメータ、ページネーション |
| `POST /cards`（画像アップロード） | `file: UploadFile` で受け取り、S3 に保存 → OCR ジョブを enqueue → `202 Accepted` + job_id を返す | `UploadFile`, ジョブキュー |
| `GET /cards/{id}` | `response_model` に会話・メモを含むネストしたスキーマを指定 | ネストした Pydantic モデル |
| `PATCH /cards/{id}` | `CardUpdate`（全項目 optional）を受け、`model_dump(exclude_unset=True)` で**送られた項目だけ**更新 | `exclude_unset` |
| `POST /recordings` | 音声チャンクを `UploadFile` で受け、保存して文字起こしジョブを enqueue | ストリーミング受信 |
| `POST /conversations/{id}/draft-message` | LLM API を `await` で呼ぶので `async def` が適切 | async + httpx |
| `GET /jobs/{id}` | Redis からジョブ状態を読んで返す | - |
| `GET /export` | `StreamingResponse` で CSV を少しずつ返す（全件をメモリに載せない） | `StreamingResponse` |

### 意味検索（pgvector）は FastAPI のどこに入るか

「課題感での検索」は、`GET /cards` のハンドラ内で

1. 検索文字列を埋め込みモデルでベクトル化
2. SQLAlchemy + pgvector で `ORDER BY embedding <=> :query_vec LIMIT n`

という流れになります。FastAPI 側は普通のクエリパラメータを受けるだけで、特別なことはしません。

---

## 12. 落とし穴・注意点

1. **`async def` の中でブロッキング処理をしない**（7 章）。同期ライブラリを使うなら `def` で書く。
   どうしても async の中で呼ぶなら `await run_in_threadpool(...)` を使う。
2. **バージョンが 0.x だが実質安定**。FastAPI は 1.0 に達していないが広く本番利用されている。
   ただしマイナーバージョンで破壊的変更が入ることがあるので、**`pyproject.toml` でバージョンを固定**する。
3. **Pydantic v1 の情報が混ざる**。ネット上の記事は v1 前提のものが多い。
   `orm_mode` → `from_attributes`、`.dict()` → `.model_dump()`、`@validator` → `@field_validator` と
   名前が変わっているので、**公式ドキュメントの v2 を見る**。
4. **CORS 設定を忘れる**。Expo の Web 版やブラウザから叩くとき `CORSMiddleware` が必要。
5. **`response_model` を省略しない**。戻り値の型ヒントを書いておけば、意図しない項目（パスワードハッシュなど）の
   漏洩を防げる。
6. **同期 SQLAlchemy と async を混ぜない**。最初は**同期版 SQLAlchemy + `def` のエンドポイント**で統一するのが安全。
   async SQLAlchemy は難易度が上がるので、必要性が出てから移行する。

---

## 13. 最小の動作確認手順

```bash
# 依存をインストール（uv を使う場合）
uv init
uv add "fastapi[standard]" uvicorn
```

```bash
# 開発サーバーを起動（--reload でコード変更時に自動再起動）
uv run uvicorn app.main:app --reload --port 8000
```

ブラウザで `http://localhost:8000/docs` を開けば、Swagger UI から動作確認できます。

> `fastapi[standard]` には uvicorn や httpx など開発に必要なものが一緒に入ります。
> `fastapi dev app/main.py` という CLI コマンドでも起動できます。

---

## 14. まとめ

- FastAPI は **「Python の型ヒントを書けば、検証とドキュメントが自動で付いてくる」** フレームワーク。
- 土台は **Starlette（Web）+ Pydantic（検証）**。エラーの出どころを切り分けるときに役立つ知識。
- このプロジェクトでは
  - **OpenAPI 自動生成** → 別メンバーのフロントと型レベルで連携できる
  - **Pydantic** → API 入出力と LLM 出力検証を同じ道具で書ける
  - **Python** → faster-whisper / YomiToku / pyannote をそのまま使える
  という 3 点が決定的な採用理由。
- **重い処理（文字起こし・OCR・要約）は FastAPI の中で実行しない**。別プロセスのワーカーに逃がす。
- `Depends` による依存性注入が、**DB セッション管理・認証・権限チェック・テスト**の全てを支える中心機能。

---

## 15. 参考リンク

- FastAPI 公式ドキュメント: https://fastapi.tiangolo.com/
- FastAPI チュートリアル（日本語あり）: https://fastapi.tiangolo.com/ja/tutorial/
- Pydantic v2 公式: https://docs.pydantic.dev/latest/
- Starlette 公式: https://www.starlette.io/
- ASGI 仕様: https://asgi.readthedocs.io/
- openapi-typescript: https://github.com/openapi-ts/openapi-typescript

---

## 次のステップ（このドキュメントの続き）

1. **SQLAlchemy 2.0 + Alembic のまとめ** ← ORM とマイグレーションの理解
2. **プロジェクトの雛形作成** ← `app/main.py` と `/health` エンドポイントだけの最小構成を動かす
3. **`cards` の CRUD 実装** ← 本ドキュメントの内容を実際に書いてみる
