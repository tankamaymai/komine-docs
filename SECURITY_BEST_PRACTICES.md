# セキュリティベストプラクティス

小嶺霊園CRMシステムのセキュリティ実装ガイドライン

作成日: 2026-01-28

---

## 1. 認証・認可

### 1.1 認証方式

本システムは**Supabase認証**と**HttpOnly Cookie**を使用したセキュアな認証を実装しています。

```
┌─────────────────────────────────────────────────────────────┐
│                    認証フロー                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ユーザーがログイン情報を送信                              │
│     POST /api/v1/auth/login                                 │
│     { email, password }                                     │
│                                                             │
│  2. バックエンドがSupabaseで認証                             │
│     Supabase Auth → JWT発行                                 │
│                                                             │
│  3. HttpOnly Cookieでトークンを設定                          │
│     Set-Cookie: access_token=xxx; HttpOnly; Secure; ...     │
│     Set-Cookie: refresh_token=xxx; HttpOnly; Secure; ...    │
│                                                             │
│  4. 以降のリクエストはCookieが自動送信                        │
│     Cookie: access_token=xxx; refresh_token=xxx             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 トークン管理

| トークン種別 | 保存場所 | 有効期限 | 用途 |
|------------|---------|---------|------|
| アクセストークン | HttpOnly Cookie | 1時間 | API認証 |
| リフレッシュトークン | HttpOnly Cookie | 7日間 | トークン更新 |
| 有効期限情報 | localStorage | - | UI表示・事前リフレッシュ |

### 1.3 Cookie設定

```typescript
// バックエンド: src/auth/authController.ts
const COOKIE_OPTIONS = {
  httpOnly: true,           // JavaScriptからアクセス不可
  secure: isProduction,     // 本番環境ではHTTPS必須
  sameSite: 'lax',          // CSRF対策
  path: '/',                // 全パスで有効
};
```

**重要なセキュリティフラグ:**

| フラグ | 説明 | 設定値 |
|-------|-----|-------|
| `HttpOnly` | JavaScriptからのアクセスを禁止（XSS対策） | `true` |
| `Secure` | HTTPS接続時のみ送信 | 本番: `true`, 開発: `false` |
| `SameSite` | クロスサイトリクエスト制御（CSRF対策） | `Lax` |

### 1.4 ロールベースアクセス制御 (RBAC)

```
┌─────────────────────────────────────────────────────────────┐
│                    権限マトリックス                          │
├──────────┬──────┬──────┬──────┬──────┬──────┬──────────────┤
│ 権限     │ 読取 │ 作成 │ 更新 │ 削除 │ 管理 │ 対象ロール   │
├──────────┼──────┼──────┼──────┼──────┼──────┼──────────────┤
│ viewer   │  ✓   │  -   │  -   │  -   │  -   │ 閲覧者       │
│ operator │  ✓   │  ✓   │  ✓   │  -   │  -   │ オペレーター │
│ manager  │  ✓   │  ✓   │  ✓   │  ✓   │  -   │ マネージャー │
│ admin    │  ✓   │  ✓   │  ✓   │  ✓   │  ✓   │ 管理者       │
└──────────┴──────┴──────┴──────┴──────┴──────┴──────────────┘
```

---

## 2. XSS (クロスサイトスクリプティング) 対策

### 2.1 実装済み対策

1. **HttpOnly Cookie使用**
   - トークンがJavaScriptからアクセス不可
   - XSS攻撃でトークンが窃取されるリスクを排除

2. **入力サニタイゼーション**
   ```typescript
   // バックエンド: src/middleware/security.ts
   app.use(sanitizeInput); // 全リクエストの入力値をサニタイズ
   ```

3. **Content Security Policy (CSP)**
   ```typescript
   // Helmetミドルウェアで設定
   app.use(getHelmetOptions());
   ```

### 2.2 フロントエンド注意事項

```typescript
// 危険: ユーザー入力をそのままHTMLに挿入
element.innerHTML = userInput; // NG

// 安全: Reactの自動エスケープを利用
<div>{userInput}</div> // OK（Reactが自動エスケープ）

// 安全: テキストノードとして挿入
element.textContent = userInput; // OK
```

---

## 3. CSRF (クロスサイトリクエストフォージェリ) 対策

### 3.1 実装済み対策

1. **SameSite Cookie属性**
   ```typescript
   sameSite: 'lax' // クロスサイトリクエストでCookieを送信しない
   ```

2. **CORS設定**
   ```typescript
   // バックエンド: src/middleware/security.ts
   getCorsOptions() {
     origin: isProduction ? allowedOrigins : true,
     credentials: true,
     methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
   }
   ```

3. **状態変更操作はPOST/PUT/DELETE**
   - GET リクエストでは状態を変更しない
   - 全ての変更操作は適切なHTTPメソッドを使用

### 3.2 SameSite設定の選択理由

| 値 | 動作 | 選択理由 |
|----|-----|---------|
| `Strict` | クロスサイトリクエストで一切送信しない | 外部サイトからのリンクでも再ログインが必要になる |
| **`Lax`** | GETリクエストのみクロスサイトで送信 | ユーザビリティとセキュリティのバランスが良い |
| `None` | 常に送信（Secure必須） | サードパーティCookie用、本システムでは不要 |

---

## 4. その他のセキュリティ対策

### 4.1 レート制限

```typescript
// バックエンド: src/middleware/security.ts

// グローバルレート制限
app.use(createRateLimiter()); // 100リクエスト/15分

// 認証エンドポイント専用（ブルートフォース対策）
authRoutes.post('/login', authRateLimiter); // 5リクエスト/15分
```

### 4.2 セキュリティヘッダー

```typescript
// Helmetミドルウェアで以下のヘッダーを設定
{
  contentSecurityPolicy: true,      // XSS対策
  crossOriginEmbedderPolicy: true,  // クロスオリジン分離
  crossOriginOpenerPolicy: true,    // Spectreなどの攻撃対策
  crossOriginResourcePolicy: true,  // リソース読み込み制限
  dnsPrefetchControl: true,         // DNS先読み制御
  frameguard: true,                 // クリックジャッキング対策
  hidePoweredBy: true,              // サーバー情報隠蔽
  hsts: true,                       // HTTPS強制
  ieNoOpen: true,                   // IE用セキュリティ
  noSniff: true,                    // MIMEスニッフィング防止
  originAgentCluster: true,         // オリジン分離
  permittedCrossDomainPolicies: true,
  referrerPolicy: true,             // リファラー制御
  xssFilter: true,                  // XSSフィルター
}
```

### 4.3 HTTP Parameter Pollution (HPP) 対策

```typescript
// バックエンド: src/middleware/security.ts
app.use(hppProtection); // 重複パラメータを防止
```

### 4.4 SQL Injection 対策

```typescript
// Prisma ORMを使用（パラメータ化クエリが自動適用）
// 危険な生SQLは使用しない

// 安全: Prismaのクエリビルダー
const user = await prisma.user.findUnique({
  where: { email: userInput }, // 自動的にエスケープ
});

// 危険: 生SQL（使用禁止）
// prisma.$queryRaw`SELECT * FROM users WHERE email = ${userInput}`
```

---

## 5. 本番環境チェックリスト

### 5.1 必須設定

- [ ] `NODE_ENV=production` を設定
- [ ] `ALLOWED_ORIGINS` に本番ドメインのみを設定
- [ ] HTTPS を有効化（SSL証明書の設定）
- [ ] `Secure` フラグが有効になっていることを確認
- [ ] データベース接続にSSLを使用

### 5.2 環境変数

```bash
# 本番環境 .env
NODE_ENV=production
ALLOWED_ORIGINS=https://your-domain.com
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=xxx  # 秘密鍵、絶対に公開しない
DATABASE_URL=postgresql://...?sslmode=require
```

### 5.3 推奨事項

1. **定期的なセキュリティ監査**
   ```bash
   npm audit  # 脆弱性チェック
   ```

2. **依存パッケージの更新**
   - Dependabotを有効化（設定済み）
   - 定期的に `npm update` を実行

3. **ログの監視**
   - 不正アクセス試行の監視
   - 認証失敗の多発をアラート

4. **バックアップ**
   - データベースの定期バックアップ
   - 暗号化された安全な場所に保存

---

## 6. インシデント対応

### 6.1 トークン漏洩時

1. 該当ユーザーのセッションを無効化
2. Supabase管理画面からユーザーを一時停止
3. パスワードリセットを強制
4. 漏洩経路の調査

### 6.2 不正アクセス検知時

1. 該当IPアドレスをブロック
2. 影響範囲の調査
3. 必要に応じて全ユーザーのセッションを無効化
4. ログの保全

---

## 7. 参考資料

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Supabase Auth Security](https://supabase.com/docs/guides/auth/security)
- [Express Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [MDN Web Security](https://developer.mozilla.org/en-US/docs/Web/Security)

---

## 変更履歴

| 日付 | 変更内容 |
|-----|---------|
| 2026-01-28 | 初版作成（HttpOnly Cookie実装に伴い作成） |
