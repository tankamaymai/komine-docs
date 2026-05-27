# API仕様書（小嶺霊園CRM バックエンド）

小嶺霊園CRM バックエンドAPIの仕様書。`komine-crm-backend/swagger.yaml`（OpenAPI 3.0）を正本（single source of truth）とし、本書はそれを読みやすく整理した日本語リファレンスである。フィールド単位の厳密な定義・examplesは `swagger.yaml` および Swagger UI（`GET /api-docs`）を参照すること。

- **対象バージョン**: API v1（`info.version: 1.0.0`）
- **正本**: `komine-crm-backend/swagger.yaml`
- **対話的ドキュメント**: `http://localhost:4000/api-docs`（Swagger UI）
- **認証詳細**: `SECURITY_BEST_PRACTICES.md`

---

## 1. 概要

### ベースURL

| 環境 | URL |
|------|-----|
| ローカル開発 | `http://localhost:4000` |
| APIプレフィックス | `/api/v1` |

全エンドポイントは `/api/v1/{entity}` のパスを取る。

### コンテンツタイプ

- リクエスト/レスポンス: `application/json`（特記を除く）
- ファイルアップロード: `multipart/form-data`
- CSV出力: `text/csv`
- ファイルダウンロード: `application/octet-stream`

---

## 2. 認証・認可

### 認証方式

Supabase Auth が発行する **JWT（Bearer トークン）** を使用する。`POST /api/v1/auth/login` 以外の全エンドポイントで認証が必須。

```
Authorization: Bearer <access_token>
```

- JWT はバックエンドの認証ミドルウェア（`src/middleware/auth.ts`）で検証される。
- Staff テーブルのレコードと `supabase_uid` で紐づく。
- Supabase 環境変数（`SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY`）が未設定の場合、認証系エンドポイントは **503 Service Unavailable** を返す。

### ロール階層と権限

権限は `viewer → operator → manager → admin` の階層。権限マトリクスは `src/middleware/permission.ts` で定義。

| ロール | 権限の概要 |
|--------|-----------|
| `viewer` | ほぼ全データの参照のみ |
| `operator` | レコードの作成・更新 |
| `manager` | レコードの削除・レポート参照 |
| `admin` | 全機能（スタッフ管理・マスタ変更を含む） |

- スタッフの新規登録・更新・削除（`/staff` の write 系）は **admin 専用**。
- 一括登録/編集（`/plots/bulk`）は **manager または admin** が必要。
- パブリックなサインアップは提供しない。ユーザー追加は admin による招待フローのみ（詳細は backend `SETUP.md`）。

---

## 3. 共通レスポンス形式

### 成功レスポンス

```jsonc
{
  "success": true,
  "data": { /* T */ }
}
```

### エラーレスポンス

```jsonc
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります",
    "details": [
      { "field": "email", "message": "メールアドレスの形式が正しくありません" }
    ]
  }
}
```

### HTTPステータスとエラーコード

| ステータス | code | 意味 |
|-----------|------|------|
| 400 | `VALIDATION_ERROR` | バリデーションエラー |
| 401 | `UNAUTHORIZED` | 未認証（トークン無効・欠落） |
| 403 | `FORBIDDEN` | 権限不足 |
| 404 | `NOT_FOUND` | リソースが存在しない |
| 409 | `CONFLICT` | データ競合（重複キー等。Prisma P2002 由来） |
| 503 | `SERVICE_UNAVAILABLE` | Supabase 認証サービス利用不可 |

Prisma エラーはグローバルエラーハンドラ（`src/middleware/errorHandler.ts`）で自動マッピングされる（P2002→409、P2025→404 等）。

---

## 4. ページネーション

一覧系エンドポイントは共通して以下のクエリパラメータを取る。

| パラメータ | 型 | デフォルト | 制約 |
|-----------|----|-----------|------|
| `page` | integer | 1 | `>= 1` |
| `limit` | integer | 20 | `1〜100` |

ページネーションのレスポンス構造はエンドポイントにより2種類ある:

- **`pagination` ネスト型**（区画在庫・書類等）: `{ items: [...], pagination: { page, limit, total, totalPages } }`
- **スタッフ一覧**: `{ staff: [...], pagination: {...} }`

> 注: `@komine/types` の `PaginatedResponse` は `{ items, pagination: {...} }` 形式。フロントエンドは歴史的経緯で `{ items, total, page, ... }` のフラット形式を使う箇所があり、アダプタ層で吸収している。

---

## 5. エンドポイント一覧

全 **39エンドポイント / 6タグ**。

| タグ | 数 | 概要 |
|------|---|------|
| [Auth](#6-auth-認証) | 1 | ログイン |
| [Plots](#7-plots-区画管理) | 16 | 区画・契約・在庫管理 |
| [Staff](#8-staff-スタッフ管理) | 5 | スタッフ管理（admin 系） |
| [Documents](#9-documents-書類管理) | 9 | 書類メタデータ・PDF生成・ファイル |
| [Masters](#10-masters-マスタデータ) | 6 | マスタデータ参照 |
| [Yucho](#11-yucho-ゆうちょ連携) | 2 | ゆうちょ請求データ・CSV出力 |

---

## 6. Auth（認証）

| メソッド | パス | 認証 | 概要 |
|---------|------|------|------|
| POST | `/api/v1/auth/login` | 不要 | ログイン |

### POST /api/v1/auth/login

メールアドレスとパスワードでログインし、JWT を取得する。

**リクエストボディ**

| フィールド | 型 | 必須 | 説明 |
|-----------|----|------|------|
| `email` | string(email) | ✓ | メールアドレス |
| `password` | string | ✓ | パスワード |

**レスポンス (200)**: `data.user`（[StaffPublic](#staff-スキーマ)）+ `data.session`（`access_token` / `refresh_token` / `expires_at`）

**エラー**: 401（認証失敗）、503（Supabase 未設定）

---

## 7. Plots（区画管理）

システムの中核。**区画ベース（plot-centric）** のデータモデル：`PhysicalPlot (物理区画) → ContractPlot (契約区画) → SaleContract → Customer`。物理区画は分割販売（1物理区画に複数契約）をサポートする。

| メソッド | パス | 認証 | 概要 |
|---------|------|------|------|
| GET | `/api/v1/plots` | ✓ | 契約区画一覧取得（検索・ページネーション） |
| POST | `/api/v1/plots` | ✓ | 新規契約区画作成（物理+契約+販売契約+顧客） |
| GET | `/api/v1/plots/grave-classifications` | ✓ | 区画区分 distinct 値取得（フィルタ用・暫定） |
| POST | `/api/v1/plots/bulk` | ✓(manager+) | 区画情報一括登録（最大500件・部分成功） |
| PUT | `/api/v1/plots/bulk` | ✓(manager+) | 区画情報一括編集（plotNumber マッチング・部分成功） |
| GET | `/api/v1/plots/{id}` | ✓ | 契約区画詳細取得 |
| PUT | `/api/v1/plots/{id}` | ✓ | 契約区画更新（部分更新対応） |
| DELETE | `/api/v1/plots/{id}` | ✓ | 契約区画削除（論理削除） |
| POST | `/api/v1/plots/{id}/restore` | ✓ | 契約区画の復活（terminated → active） |
| GET | `/api/v1/plots/{id}/contracts` | ✓ | 物理区画の契約一覧取得（分割販売） |
| POST | `/api/v1/plots/{id}/contracts` | ✓ | 物理区画に新規契約追加（分割販売） |
| GET | `/api/v1/plots/inventory/summary` | ✓ | 在庫全体サマリー取得 |
| GET | `/api/v1/plots/inventory/periods` | ✓ | 期別サマリー取得 |
| GET | `/api/v1/plots/inventory/sections` | ✓ | セクション別集計取得 |
| GET | `/api/v1/plots/inventory/areas` | ✓ | 面積別集計取得 |
| GET | `/api/v1/plots/{id}/inventory` | ✓ | 物理区画の在庫状況取得 |

### GET /api/v1/plots — 契約区画一覧

**クエリ**: `page`, `limit`, `search`（区画番号・顧客名）, `usageStatus`, `cemeteryType`, `graveKind`(int), `graveKubun`(int), `graveType`(int)

**レスポンス (200)**: `data` = [ContractPlotSummary](#区画スキーマ) の配列

### POST /api/v1/plots — 新規契約区画作成

物理区画と契約区画を同時作成。顧客・販売契約・オプション情報（WorkInfo / BillingInfo / UsageFee / ManagementFee 等）も含む。リクエストボディは `CreatePlotInput`。

**レスポンス (201)**: `data.id`（作成された ContractPlot の UUID）+ `data.message`
**エラー**: 400 / 409（区画番号重複等）

### POST /api/v1/plots/bulk — 一括登録

`items[]`（1〜500件）。各行は独立トランザクションで処理され、1行の失敗が他行に波及しない**部分成功モデル**。全行失敗でも 201 を返し `failed[]` に詳細が入る。

**レスポンス (201)**: `data.totalRequested` / `data.succeeded` / `data.failed[]`（`row`, `plotNumber`, `error`）/ `data.results[]`（成功行）

### PUT /api/v1/plots/bulk — 一括編集

`plotNumber`（区画番号）でマッチングして更新。未指定フィールド=変更しない、null 明示=クリア。最大500件・部分成功。**レスポンス (200)** 構造は一括登録に準ずる。

### GET /api/v1/plots/{id} — 詳細取得

**レスポンス (200)**: `data` = [ContractPlotDetail](#区画スキーマ)（物理区画・顧客・販売契約・オプション情報を含む）

### PUT /api/v1/plots/{id} — 更新

部分更新対応（`contractPlot`, `saleContract`, `customer`, `workInfo`, `billingInfo`, `usageFee`, `managementFee`）。リクエストボディは `UpdatePlotInput`。

### DELETE /api/v1/plots/{id} — 削除（論理削除）

関連する販売契約・使用料・管理料・請求情報・勤務先情報も削除。顧客は他契約が無い場合のみ削除。物理区画のステータスは自動更新。

### POST /api/v1/plots/{id}/restore — 復活

`terminated` の契約区画を `active` に戻す（誤操作リカバリ）。`reason`（1〜200文字、必須、空白のみ不可）は履歴に記録される。現ステータスが `terminated` でない場合は **409**。

### GET / POST /api/v1/plots/{id}/contracts — 物理区画の契約

- GET: 物理区画に紐づく全契約を取得（分割販売時は複数）。`data` = [PhysicalPlotContracts](#区画スキーマ)。
- POST: 既存物理区画に新契約を追加。契約面積を検証し物理区画ステータスを自動更新。ボディは `CreatePlotContractInput`。

### 在庫系

| エンドポイント | クエリ | レスポンス `data` |
|---------------|--------|------------------|
| `GET /inventory/summary` | — | InventorySummary（総数・使用済・残・使用率・総/残面積） |
| `GET /inventory/periods` | `period`（第1期〜第4期等） | `periods[]`（PeriodSummary） |
| `GET /inventory/sections` | `period`, `status`(available/partially_sold/sold_out), `search`, `sortBy`, `sortOrder`, `page`, `limit` | `items[]`（SectionInventoryItem）+ `pagination` |
| `GET /inventory/areas` | `period`, `search`, `sortBy`, `sortOrder`, `page`, `limit` | `items[]`（AreaInventoryItem）+ `pagination` |
| `GET /{id}/inventory` | — | PhysicalPlotInventory（総/割当済/利用可能面積・利用率・ステータス） |

`period` の enum: `第1期`, `第2期`, `第3期`, `第3期樹林部`, `第4期`

---

## 8. Staff（スタッフ管理）

write 系（POST/PUT/DELETE）は **admin 専用**。一覧・詳細の参照は権限に応じる。

| メソッド | パス | 認証 | 概要 |
|---------|------|------|------|
| GET | `/api/v1/staff` | ✓ | スタッフ一覧取得（削除済みは除外） |
| POST | `/api/v1/staff` | ✓(admin) | スタッフ新規登録（Supabase 連携） |
| GET | `/api/v1/staff/{id}` | ✓ | スタッフ詳細取得 |
| PUT | `/api/v1/staff/{id}` | ✓(admin) | スタッフ情報更新（Supabase メール同期） |
| DELETE | `/api/v1/staff/{id}` | ✓(admin) | スタッフ削除（ソフトデリート） |

### GET /api/v1/staff

**クエリ**: `page`, `limit`, `role`（viewer/operator/manager/admin）, `is_active`(boolean)
**レスポンス (200)**: `data.staff[]`（[StaffAdmin](#staff-スキーマ)）+ `data.pagination`（`page`/`limit`/`total`/`totalPages`）

### POST /api/v1/staff

Supabase Admin API でユーザーを作成し、取得した `supabase_uid` で Staff レコードを作成（トランザクション・ロールバック対応）。ボディは `CreateStaffRequest`。**エラー**: 400 / 409（メール重複・Supabase 残存等）

### PUT /api/v1/staff/{id}

名前・メール・役割を更新。Supabase 側のメールも同期。ボディは `UpdateStaffRequest`。

### DELETE /api/v1/staff/{id}

`deleted_at` に現在日時を設定、`is_active` を false に。物理削除はしない。

> パスパラメータ `id` は **integer**（スタッフは整数ID。区画・書類は UUID）。

---

## 9. Documents（書類管理）

| メソッド | パス | 認証 | 概要 |
|---------|------|------|------|
| GET | `/api/v1/documents` | ✓ | 書類一覧取得（検索・フィルタ・ページネーション） |
| POST | `/api/v1/documents` | ✓ | 書類メタデータ作成 |
| POST | `/api/v1/documents/generate-pdf` | ✓ | PDF生成（Base64返却） |
| GET | `/api/v1/documents/{id}` | ✓ | 書類詳細取得 |
| PUT | `/api/v1/documents/{id}` | ✓ | 書類更新 |
| DELETE | `/api/v1/documents/{id}` | ✓ | 書類削除（論理削除） |
| POST | `/api/v1/documents/{id}/upload` | ✓ | ファイルアップロード（最大10MB） |
| GET | `/api/v1/documents/{id}/download` | ✓ | ダウンロードURL取得（Pre-signed・15分） |
| GET | `/api/v1/documents/file/{fileKey}` | ✓ | ローカルファイルダウンロード（開発環境用） |

### GET /api/v1/documents

**クエリ**: `page`, `limit`, `contractPlotId`(uuid), `customerId`(uuid), `type`（[DocumentType](#書類スキーマ)）, `status`（[DocumentStatus](#書類スキーマ)）, `sortBy`（created_at/updated_at/name/type/status）, `sortOrder`（asc/desc, 既定 desc）
**レスポンス (200)**: `items[]`（DocumentListItem）+ `pagination`

### POST /api/v1/documents

ボディは `CreateDocumentRequest`（`type` / `name` 必須、`contractPlotId` / `customerId` / `templateType` / `templateData` / `notes` 任意）。**レスポンス (201)**: DocumentDetail

### POST /api/v1/documents/generate-pdf

テンプレートから PDF を生成し Base64 で返却。ボディは `GeneratePdfRequest`（`templateType` / `templateData` 必須）。`templateType` の enum: `invoice`, `postcard`, `permit`, `envelope-letter`, `envelope-base`, `payment-guide`。`documentId` 指定時はファイルを保存。**レスポンス (200)**: `GeneratePdfResponse`（`documentId` / `pdf`(Base64) / `mimeType` / `fileSize`）

### PUT / DELETE /api/v1/documents/{id}

- PUT: メタデータ更新（`name` / `description` / `status` / `templateData` / `notes`）。ボディ `UpdateDocumentRequest`。
- DELETE: `deleted_at` 設定による論理削除。

### ファイル系

- `POST /{id}/upload`: `multipart/form-data` の `file`（PDF/Word/Excel/画像、最大10MB）。レスポンスは `id` / `fileName` / `fileSize` / `mimeType`。
- `GET /{id}/download`: Pre-signed URL（有効期限15分=900秒）+ `fileName` / `mimeType` / `expiresIn`。
- `GET /file/{fileKey}`: ローカルストレージから直接ダウンロード（`application/octet-stream`、開発環境用）。

---

## 10. Masters（マスタデータ）

全エンドポイント GET・参照のみ。有効な（`isActive`）データを表示順（`sortOrder`）で返す。

| メソッド | パス | 概要 | レスポンス `data` |
|---------|------|------|------------------|
| GET | `/api/v1/masters/section-name` | 区画名マスタ | SectionNameMaster[] |
| GET | `/api/v1/masters/relationship` | 続柄マスタ（レガシー sykbnn KBNNO=2009） | RelationshipMaster[] |
| GET | `/api/v1/masters/contractor` | 工事業者マスタ | ContractorMaster[] |
| GET | `/api/v1/masters/direction` | 方角マスタ（KBNNO=2024、code=int文字列） | DirectionMaster[] |
| GET | `/api/v1/masters/position` | 位置マスタ（KBNNO=2025、code=int文字列） | PositionMaster[] |
| GET | `/api/v1/masters/all` | 全マスタ一括取得 | `{ sectionName[], relationship[], contractor[] }` |

各マスタ共通フィールド: `id`(int), `code`(string), `name`(string), `description`(string?), `sortOrder`(int?), `isActive`(bool)。

---

## 11. Yucho（ゆうちょ連携）

管理料・合祀料金の自動払込み連携。

| メソッド | パス | 認証 | 概要 |
|---------|------|------|------|
| GET | `/api/v1/yucho/billing` | ✓ | ゆうちょ請求対象データ取得 |
| GET | `/api/v1/yucho/export` | ✓ | ゆうちょ自動払込み用CSV出力 |

### GET /api/v1/yucho/billing

指定期間の請求対象データ（契約区画・顧客・口座・請求金額）を集約。

**クエリ**: `year`(必須), `month`(任意), `category`（management/collective/all, 既定 all）, `status`（unbilled/billed/paid/all, 既定 unbilled）

**レスポンス (200)** `data`:
- `period`: `{ year, month? }`
- `items[]`: `category` / `sourceId` / `contractPlotId` / `plotNumber` / `areaName` / `customerId?` / `customerName?` / `customerNameKana?` / `billingAmount` / `billingStatus` / `scheduledDate?` / `billingMonth?` / `billingInfo?`（`bankName` / `branchName` / `accountType` / `accountNumber` / `accountHolder`）
- `summary`: `totalCount` / `totalAmount` / `byCategory`

> 件数の可視化（backend #172 対応）: `summary` には実出力件数 `exportableCount`・除外件数 `excludedNoAccountCount` 等が追加されている。口座未登録で CSV から除外されるレコードを検知できる。

### GET /api/v1/yucho/export

請求データから払込み用 CSV を生成して返す（`text/csv`）。

**クエリ**: `year`(必須), `month`, `category`, `status`, `format`（既定 zengin）, `transferDate`(必須, `YYYY-MM-DD`), `clientCode`(必須, 最大10桁), `clientName`(必須, 半角40文字), `bankCode`(既定 9900=ゆうちょ), `branchCode`(既定 000)

> ⚠️ **CSVフォーマットは改修中**: `swagger.yaml` 上は全銀協フォーマット（120バイト固定長）と記載があるが、ゆうちょ自動払込みの正フォーマットは **カンマ区切りCSV・Shift-JIS** であることが実ファイルで確認されている。是正は backend issue #168 で進行中。最新の確定仕様は #168 を参照。

---

## 12. 主要スキーマ定義

フィールド単位の完全な定義は `swagger.yaml` の `components.schemas`（全45スキーマ）を参照。代表的なものを以下に整理する。

### 区画スキーマ

- `ContractPlotSummary` — 一覧用の契約区画サマリー
- `ContractPlotDetail` — 詳細（物理区画・顧客・販売契約・オプション情報）
- `CreatePlotInput` / `UpdatePlotInput` — 作成/更新リクエスト
- `CreatePlotContractInput` — 物理区画への契約追加
- `PhysicalPlotContracts` — 物理区画とその契約群
- `PhysicalPlotInventory` — 物理区画の在庫状況
- `InventorySummary` / `PeriodSummary` / `SectionInventoryItem` / `AreaInventoryItem` — 在庫集計
- `Pagination` — ページネーション情報

### Staff スキーマ

- `Staff` — 内部処理・DB操作用（API レスポンスには使わない。`supabase_uid` を含む）
- `StaffPublic` — ログインレスポンス用の公開情報
- `StaffAdmin` — admin 向けスタッフ情報
- `CreateStaffRequest` / `UpdateStaffRequest` — 登録/更新リクエスト

共通フィールド: `id`(int), `name`(≤100), `email`(≤254), `role`(viewer/operator/manager/admin), `is_active`, `last_login_at?`, `created_at`

### 書類スキーマ

- `DocumentType` (enum): `invoice`, `postcard`, `contract`, `permit`, `envelope_letter`, `envelope_base`, `other`
- `DocumentStatus` (enum): `draft`, `generated`, `sent`, `archived`
- `DocumentListItem` / `DocumentDetail` — 一覧/詳細
- `CreateDocumentRequest` / `UpdateDocumentRequest` — 作成/更新
- `GeneratePdfRequest` / `GeneratePdfResponse` — PDF生成

### マスタスキーマ

`SectionNameMaster`, `RelationshipMaster`, `ContractorMaster`, `DirectionMaster`, `PositionMaster`（共通フィールドは [10. Masters](#10-masters-マスタデータ) 参照）

### エラースキーマ

`ErrorResponse` および各ステータス用レスポンス: `BadRequest`(400), `Unauthorized`(401), `Forbidden`(403), `NotFound`(404), `Conflict`(409), `ServiceUnavailable`(503)

---

## 13. 参照

| 資料 | 場所 | 用途 |
|------|------|------|
| OpenAPI 正本 | `komine-crm-backend/swagger.yaml` | フィールド単位の厳密な定義・examples |
| Swagger UI | `http://localhost:4000/api-docs` | 対話的な仕様閲覧・Try it out |
| 認証・認可詳細 | `SECURITY_BEST_PRACTICES.md` | 認証方針・権限マトリクス |
| 区画管理設計 | `design/PLOT_MANAGEMENT_SPEC.md` | 区画ベースのデータモデル詳細 |
| 合祀管理設計 | `design/COLLECTIVE_BURIAL_SPEC.md` | 合祀管理の業務仕様 |
| 契約ライフサイクル | `design/契約ライフサイクル設計書.md` | 契約状態遷移（vacant/active/terminated） |

---

> 本書は `swagger.yaml` から整理した読み物であり、仕様の正本は常に `swagger.yaml`。エンドポイント追加・変更時は `swagger.yaml` を更新し（`npm run swagger:validate` で検証）、本書も追従させること。
