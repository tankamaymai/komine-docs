# 小峰霊園CRM タスク管理

作成日: 2026-01-24
最終更新: 2026-02-02

## タスク管理方針

**すべてのタスクはGitHub Issuesで管理しています。**

| リポジトリ | 用途 | Issues |
|-----------|------|--------|
| [komine--cemetery-crm-backend](https://github.com/tankamaymai/komine--cemetery-crm-backend) | バックエンド | [Issues](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues) |
| [komine-frontend-crm-v2](https://github.com/tankamaymai/komine-frontend-crm-v2) | フロントエンド | [Issues](https://github.com/tankamaymai/komine-frontend-crm-v2/issues) |
| [komine-docs](https://github.com/zaitsu82/komine-docs) | ドキュメント | [Issues](https://github.com/zaitsu82/komine-docs/issues) |
| [komine-infra](https://github.com/zaitsu82/komine-infra) | インフラ（CDK） | [Issues](https://github.com/zaitsu82/komine-infra/issues) |

---

## バックエンド Issues

| # | タイトル | 状態 |
|---|---------|------|
| [#34](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/34) | [FEATURE] 契約追加API（分譲対応） | Open |
| [#33](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/33) | [FEATURE] 書類生成機能の改善（Excel/PDF統一） | Open |
| [#30](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/30) | [FEATURE] スタッフ新規作成API実装 | Open |
| [#29](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/29) | [BUG] 台帳問い合わせの検索が100件までしか機能しない | Open |
| [#28](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/28) | [BUG] 履歴情報を自動記録方式に変更 | Open |

---

## フロントエンド Issues

| # | タイトル | 状態 |
|---|---------|------|
| [#8](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/8) | [CI] GitHub Actions CI/CD整備 | Open |
| [#7](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/7) | [FEATURE] スタッフ新規作成UI有効化 | Open |
| [#6](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/6) | [FEATURE] 書類生成UI改善（フォーマット選択・一括生成） | Open |
| [#5](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/5) | [UX] 一覧画面のデータキャッシュ・状態保持 | Open |
| [#4](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/4) | [UI] 画面UIの統一（合祀管理画面基準） | Open |
| [#3](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/3) | [BUG] 台帳問い合わせ検索のサーバーサイド移行 | Open |
| [#2](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/2) | [BUG] 履歴情報の表示UI修正（履歴追加ボタン削除） | Open |
| [#1](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/1) | [BUG] 区画詳細画面で全タブに対応履歴が表示される | Open |

---

## ドキュメント Issues

| # | タイトル | 状態 |
|---|---------|------|
| [#1](https://github.com/zaitsu82/komine-docs/issues/1) | [DOCS] API仕様書の整備 | Open |
| [#2](https://github.com/zaitsu82/komine-docs/issues/2) | [DOCS] 運用マニュアル作成 | Open |
| [#3](https://github.com/zaitsu82/komine-docs/issues/3) | [DOCS] デプロイ手順書作成 | Open |

---

## インフラ Issues

| # | タイトル | 状態 |
|---|---------|------|
| [#1](https://github.com/zaitsu82/komine-infra/issues/1) | [INFRA] VPC・ネットワーク設計（CDK） | Open |
| [#2](https://github.com/zaitsu82/komine-infra/issues/2) | [INFRA] ECS/Fargate バックエンド構成（CDK） | Open |
| [#3](https://github.com/zaitsu82/komine-infra/issues/3) | [INFRA] RDS PostgreSQL構成（CDK） | Open |
| [#4](https://github.com/zaitsu82/komine-infra/issues/4) | [INFRA] CloudFront + S3 フロントエンド配信（CDK） | Open |
| [#5](https://github.com/zaitsu82/komine-infra/issues/5) | [INFRA] CI/CD パイプライン構成（CDK） | Open |

---

## 優先度

### 最優先（Critical）
1. Frontend [#1](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/1) - タブ表示不具合
2. Backend [#28](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/28) + Frontend [#2](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/2) - 履歴自動記録

### 高優先（High）
3. Backend [#30](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/30) + Frontend [#7](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/7) - スタッフ新規作成
4. Backend [#29](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/29) + Frontend [#3](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/3) - サーバーサイド検索

### 中優先（Medium）
5. Frontend [#4](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/4) - UI統一
6. Frontend [#5](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/5) - データキャッシュ
7. Backend [#33](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/33) + Frontend [#6](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/6) - 書類生成改善
8. Frontend [#8](https://github.com/tankamaymai/komine-frontend-crm-v2/issues/8) - CI/CD整備

### 低優先（Low）
9. Docs [#1](https://github.com/zaitsu82/komine-docs/issues/1), [#2](https://github.com/zaitsu82/komine-docs/issues/2), [#3](https://github.com/zaitsu82/komine-docs/issues/3) - ドキュメント整備
10. Infra [#1](https://github.com/zaitsu82/komine-infra/issues/1)〜[#5](https://github.com/zaitsu82/komine-infra/issues/5) - インフラ構築

---

## 完了済みタスク

### Phase 9.1 契約管理ロジック（2026-02-01完了）
- [x] 契約ライフサイクル設計書作成 → `docs/design/契約ライフサイクル設計書.md`
- [x] ContractStatus enum追加 → `prisma/schema.prisma`
- [x] 状態遷移バリデーションサービス → `src/plots/services/contractStatusService.ts`
- [x] ユニットテスト（54テスト）→ `tests/plots/services/contractStatusService.test.ts`

### 過去の完了Phase
- Phase 1-5: API統合基盤（2026-01-27完了）
- Phase 7: 区画管理API強化（2026-01-31完了）
- Phase 8: 全体リファクタリング（2026-01-31完了）

### クローズ済みIssue
- Backend [#27](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/27) - タブ表示不具合（Closed）
- Backend [#31](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/31) - UI統一（Closed）
- Backend [#32](https://github.com/tankamaymai/komine--cemetery-crm-backend/issues/32) - データキャッシュ（Closed）
