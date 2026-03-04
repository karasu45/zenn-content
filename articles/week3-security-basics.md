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

> **Windows の場合**: `openssl` コマンドは標準では入っていません。[Git for Windows](https://gitforwindows.org/) をインストールするとGit Bashから使えます。または、[SSL Labs](https://www.ssllabs.com/ssltest/) でWebブラウザからチェックすることもできます。

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
your-domain.com.  IN  TXT  "v=spf1 include:<メール送信サービスの指定値> ~all"
```
`include:` の部分は、実際に使っているメール送信サービスのドキュメントで確認してください。例:
- **Resend**: `include:resend.com`（[公式ドキュメント](https://resend.com/docs/dashboard/domains/introduction)で最新値を確認）
- **SendGrid**: `include:sendgrid.net`
- **AWS SES**: `include:amazonses.com`

> 複数サービスを使っている場合は `include:` を並べます。ただし、SPFのDNSルックアップは**10回まで**の制限があるので注意してください。

### DMARCレコード
```
_dmarc.your-domain.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@your-domain.com"
```
DMARCは**段階的に強化**するのが推奨です:

1. `p=none`（監視のみ）: まずはレポートを収集して、正規メールが誤判定されないか確認
2. `p=quarantine`（迷惑メールフォルダに振り分け）: 問題がなければ2〜4週間後に移行
3. `p=reject`（拒否）: 十分なデータが集まったら最終段階に

> **いきなり `p=quarantine` や `p=reject` を設定すると、正規のメール（通知メール等）が届かなくなるリスクがあります。**必ず `p=none` から始めてください。

### DKIMの設定

SPF・DMARCと並んで**DKIM（DomainKeys Identified Mail）**の設定も重要です。DKIMはメールに電子署名を付与し、送信途中での改ざんを検知します。

SPF + DKIM + DMARCの**三点セット**で設定することで、メールの信頼性が大幅に向上します。DKIMの設定方法はメール送信サービスによって異なるため、各サービスのドキュメントを参照してください。

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
your-domain.com.  IN  CAA  0 issuewild "letsencrypt.org"
```
- `issue`: 通常の証明書の発行を許可するCA
- `issuewild`: **ワイルドカード証明書**の発行を許可するCA

ワイルドカード証明書を使わない場合でも、`issuewild` を明示的に設定しておくと、意図しないワイルドカード証明書の発行を防げます。

## 5. Cookie設定の見直し

認証にCookieを使っている場合（Supabase Authなど）、Cookie属性の設定が適切か確認しましょう。

| 属性 | 推奨値 | 効果 |
|------|--------|------|
| `Secure` | true | HTTPS接続でのみ送信 |
| `HttpOnly` | true | JavaScriptからアクセス不可（XSS対策） |
| `SameSite` | `Lax` or `Strict` | CSRF攻撃の防止 |
| `Path` | `/` | 適切なパスに限定 |

**SameSite属性の3つの値**:
- `Strict`: クロスサイトリクエストではCookieを一切送信しない（最も厳格）
- `Lax`: リンククリック等の「安全な」ナビゲーションではCookieを送信するが、POSTリクエスト等では送信しない（推奨）
- `None`: クロスサイトリクエストでもCookieを送信（`Secure` 属性が必須。サードパーティ連携が必要な場合のみ）

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

> **注意**: この記事で紹介した対策は「設定レイヤー」の防御です。アプリケーション自体の脆弱性（SQLインジェクション、認証の不備など、OWASP Top 10に含まれるもの）は別途対策が必要です。セキュリティ対策に「これで完璧」はないので、継続的な学習と改善を心がけましょう。

質問やフィードバックがあればコメントで教えてください。
