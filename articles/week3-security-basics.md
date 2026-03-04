---
title: "個人開発のWebサービス、最低限やっておくべきセキュリティ対策5選"
emoji: "🔒"
type: "tech"
topics: ["セキュリティ", "Next.js", "個人開発", "Web開発"]
published: false
---

## はじめに

個人開発でWebサービスを公開するとき、セキュリティ対策をどこまでやるべきか迷いませんか？

企業のように専任のセキュリティチームがいるわけでもなく、かといって「何もしない」は不安。自分も最初は「Vercelにデプロイすれば大丈夫でしょ」くらいに思っていたけど、実際に調べてみると穴だらけだった。

この記事では、**個人開発者がこれだけやっておけば最低限OK**というセキュリティ対策5つをまとめます。どれも無料で、30分〜1時間あれば全部設定できます。

## 1. HTTPS化（そもそも大前提）

2026年の今、HTTPSは「対策」というより「前提条件」です。VercelやCloudflareを使っていれば自動的にHTTPS化されますが、確認すべきポイントがあります。

- `http://` でアクセスしたとき、`https://` にリダイレクトされるか
- 混合コンテンツ（Mixed Content）がないか — HTTPS のページから HTTP のリソースを読み込んでいないか
- SSL証明書の有効期限が切れていないか

### 確認方法
```bash
# リダイレクト確認
curl -I http://your-site.com

# SSL証明書の有効期限確認
echo | openssl s_client -connect your-site.com:443 2>/dev/null | openssl x509 -noout -dates
```

Let's Encryptの証明書は90日で期限切れになるので、自動更新が設定されているか確認しておきましょう。Vercel/Cloudflareなら自動管理されます。

## 2. セキュリティヘッダーの設定

前回の記事で詳しく書きましたが、改めて最低限の3つを挙げます。

| ヘッダー | 効果 |
|---------|------|
| `Strict-Transport-Security` | HTTPS強制 |
| `X-Content-Type-Options: nosniff` | MIMEスニッフィング防止 |
| `X-Frame-Options: DENY` | クリックジャッキング防止 |

この3つは設定値も固定で、副作用もほぼありません。迷わず入れましょう。

CSP（Content-Security-Policy）もできれば設定したいですが、フレームワークやライブラリとの相性で調整が必要な場合があるので、まずは `Report-Only` モードで試すのがおすすめです。

## 3. DNS設定（SPF / DMARC）

自分のドメインからメールを送る場合（通知メール等）、SPFとDMARCを設定しないと、なりすましメールの踏み台にされるリスクがあります。設定していないドメインは、フィッシングメールの送信元として悪用されやすくなります。

### SPFレコード
```
your-domain.com.  IN  TXT  "v=spf1 include:_spf.google.com include:amazonses.com ~all"
```
`include:` の部分は、実際に使っているメール送信サービスに合わせてください（Resend、SendGrid、AWS SES等）。

### DMARCレコード
```
_dmarc.your-domain.com.  IN  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@your-domain.com"
```
`p=quarantine` は「SPF/DKIMに合格しないメールを迷惑メールフォルダに入れる」という指定です。

### 確認方法
```bash
# SPF確認
dig TXT your-domain.com +short

# DMARC確認
dig TXT _dmarc.your-domain.com +short
```

## 4. SSL証明書の管理

「HTTPS化した」で安心していると、証明書の期限切れでサイトにアクセスできなくなることがあります。ブラウザは期限切れの証明書を持つサイトに対して警告画面を表示するため、ユーザーの信頼を一瞬で失います。

### やるべきこと
- 自動更新が設定されているか確認
- 証明書の有効期限を監視する仕組みを入れる（後述）
- CAA（Certificate Authority Authorization）レコードを設定して、意図しないCAからの発行を防ぐ

### CAAレコード例
```
your-domain.com.  IN  CAA  0 issue "letsencrypt.org"
```
これにより、Let's Encrypt以外のCAが証明書を発行できなくなります。

## 5. Cookie設定の見直し

認証にCookieを使っている場合（Supabase Authなど）、Cookie属性の設定が適切か確認しましょう。

| 属性 | 推奨値 | 効果 |
|------|--------|------|
| `Secure` | true | HTTPS接続でのみ送信 |
| `HttpOnly` | true | JavaScriptからアクセス不可（XSS対策） |
| `SameSite` | `Lax` or `Strict` | CSRF攻撃の防止 |
| `Path` | `/` | 適切なパスに限定 |

Supabaseを使っている場合、`@supabase/ssr` パッケージがこれらを適切に設定してくれます。ただし、カスタムでCookieを扱う場合は手動で確認が必要です。

## おまけ: 無料で使えるチェックツール

| ツール | チェック内容 |
|--------|------------|
| [securityheaders.com](https://securityheaders.com/) | セキュリティヘッダー |
| [SSL Labs](https://www.ssllabs.com/ssltest/) | SSL/TLS設定 |
| [MX Toolbox](https://mxtoolbox.com/) | DNS設定（SPF/DMARC） |
| DevTools | Cookie属性、混合コンテンツ |

## まとめ

| 対策 | 作業時間 | 効果 |
|------|---------|------|
| HTTPS確認 | 5分 | 通信暗号化 |
| セキュリティヘッダー | 10分 | XSS/クリックジャッキング防止 |
| DNS設定 | 15分 | なりすまし防止 |
| SSL証明書管理 | 10分 | 期限切れ防止 |
| Cookie設定 | 10分 | セッション保護 |

合計50分。これで「最低限のセキュリティはやってる」と言えます。

完璧ではないけど、**何もしていない状態と比べれば雲泥の差**です。まずはこの5つから始めて、余裕ができたらWAFや脆弱性スキャンなども検討していきましょう。

質問やフィードバックがあればコメントで教えてください。
