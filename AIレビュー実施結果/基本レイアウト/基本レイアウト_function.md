# 基本レイアウト レビュー結果

## 指摘事項

### 1. getAuthMe が認証結果ではなく固定ユーザーを返却しており、認証を実質バイパスできる
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/010.システム共通/基本レイアウト.md の初期表示で「ログインユーザーの権限を取得」「ヘッダーにログインユーザー名を表示」とあり、実ログインユーザー情報を返す前提。
  - 実装: functions/auth/getAuthMe/app.py で AuthenticatedUser を固定値生成して 200 応答しており、トークン検証・ユーザー照合が行われていない。
- 指摘内容:
  - 現状はリクエストの認証状態に関係なく固定ユーザーを返却できるため、ユーザー名表示・権限制御の前提が崩れる。仕様の「ログインユーザー情報取得」を満たしていない。
- 期待されるテスト:
  - 有効トークン時のみ 200 となること。
  - 無効/欠落トークン時は 401 となること。
  - トークン内テナントと要求テナント不一致時は 403 となること。

### 2. getAuthMe の実装と service 実装が分離しており、認証処理本体が未使用のまま
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py, shared/services/authService.py
- 根拠:
  - 設計書: spec/詳細設計/openapi/paths/auth.yaml の /auth/me は認証済みユーザー情報返却 API。
  - 実装: functions/auth/getAuthMe/app.py は shared/services/authService.py の get_auth_me を呼ばず、コメントアウトされたダミー実装を返却。
- 指摘内容:
  - サービス層に定義された分岐・例外処理・ユーザー照合ロジックが実行されないため、設計どおりの振る舞い検証ができない。
- 期待されるテスト:
  - handler から service が呼ばれることを前提に、成功/失敗分岐を API レベルで確認するテスト。

### 3. 認証関連の秘密鍵/トークン検証実装にセキュリティリスクが残る
- 優先度: High
- 種別: 実装不備
- 対象: shared/services/authService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/020.ログイン/ログイン.md では外部認証基盤を前提にした認証フロー定義。
  - 実装: shared/services/authService.py に固定 SECRET 定義、JWT 処理の例外を包括的に握りつぶす分岐があり、失敗理由・検証保証が弱い。
- 指摘内容:
  - 秘密情報管理と署名検証の厳密性が不足し、運用時に不正トークン受け入れや障害解析困難を招く。
- 期待されるテスト:
  - 署名不正トークン、期限切れ、形式不正でそれぞれ適切なエラーを返すこと。
  - 鍵取得失敗時のエラー制御を確認すること。

### 4. getAuthConfig の tenantId 入力検証不足により不正入力がそのまま service/DB へ流れる
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/auth/getAuthConfig/app.py
- 根拠:
  - 設計書: spec/詳細設計/openapi/paths/auth.yaml の /public/auth/config/{tenantId} は tenantId 指定前提。
  - 実装: functions/auth/getAuthConfig/app.py は pathParameters 未設定時に空文字を使用し、形式チェックなしで service に委譲。
- 指摘内容:
  - 必須入力・形式検証が不足し、400 系で弾くべき入力が内部処理に到達する。
- 期待されるテスト:
  - pathParameters が null/空/別キー/不正形式 tenantId の各ケースでエラー応答を検証。

### 5. API テストがダミー実装依存で、仕様検証になっていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/auth/test_getAuthMe.py, tests/functions/auth/test_getAuthConfig.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の単体テスト観点（正常系・異常系・入力バリエーション・分岐条件）を満たす必要。
  - 実装: test_getAuthMe は固定レスポンス前提の検証中心、test_getAuthConfig はモック中心で tenantId 異常値や依存失敗分岐の網羅が不足。
- 指摘内容:
  - 仕様どおりの認証・認可・例外制御を保証できず、実装変更時に回帰を検出しにくい。
- 期待されるテスト:
  - 401/403/404/500 の使い分け検証。
  - tenantId 入力バリエーションと DB 例外時応答の検証。
  - レスポンススキーマ（UUID 文字列化、authorities 配列）の検証。

### 6. テナント抽出と権限制御の結合テストが不足し、基本レイアウトの表示制御リスクが残る
- 優先度: Middle
- 種別: テスト不足
- 対象: shared/core/middleware.py, tests/shared/core/test_middleware.py, tests/functions/auth/test_getAuthMe.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/010.システム共通/基本レイアウト.md は権限に応じた機能表示を要求。
  - 実装: middleware 単体では tenant_code 抽出を検証しているが、getAuthMe と組み合わせた権限返却の結合観点が不足。
- 指摘内容:
  - サブドメイン/Host の揺れにより誤 tenant_code となった際、誤権限表示を防げるか確認できない。
- 期待されるテスト:
  - Host パターン別に tenant_code 抽出→認証結果反映までを確認する API 結合テスト。

### 7. 基本レイアウト設計書単体では getAuthConfig の仕様トレースが不十分
- 優先度: Low
- 種別: 設計確認事項
- 対象: 機能「基本レイアウト」と API getAuthConfig
- 根拠:
  - 設計書: spec/基本設計/機能設計/010.システム共通/基本レイアウト.md は画面表示要件中心で、getAuthConfig の入出力仕様詳細は直接記載が薄い。
  - 実装: getAuthConfig はログイン/認証方式判定に関係する API。
- 指摘内容:
  - 本機能レビューで getAuthConfig を厳密評価するには、ログイン機能設計および OpenAPI を正とする適用範囲の明確化が必要。
- 期待されるテスト:
  - なし（設計スコープ確認事項）。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - getAuthMe: 実トークン認証成功時のユーザー情報返却確認が不足。
  - getAuthConfig: tenantId 有効時の設定返却バリエーション確認が不足。
- 入力値バリエーション不足
  - tenantId の null/空/不正形式、Cookie/Authorization 欠落、Host 形式揺れの検証不足。
- 境界値不足
  - JWT 期限境界、tenantId 文字列境界、authority 配列件数境界の検証不足。
- 分岐条件不足
  - 認証失敗（署名不正/期限切れ/tenant 不一致）や tenant 不存在分岐の網羅不足。
- データ入出力不足
  - UUID の JSON 変換、authorities 配列、外部 IdP 設定配列の返却形式検証不足。
- 非更新確認不足
  - 認証/設定取得 API 実行時に DB 更新が発生しないことの確認不足。
- 異常系不足
  - DB 接続失敗、外部依存失敗、想定例外ごとのステータス・エラーコード検証不足。

## 総評
- High 指摘は、getAuthMe の固定値返却と認証処理未接続により、認証・権限制御の前提を満たせない点が中心。
- 基本レイアウトで要求される「ログインユーザー名表示」「権限による機能表示」の品質保証に対して、実装とテストの双方で不足がある。
- 修正優先度は、1) getAuthMe 実装接続と認証厳密化、2) getAuthConfig 入力検証、3) 仕様ベースの異常系テスト拡充の順が妥当。

## 残留リスク・確認できなかった範囲
- spec/050.テスト/テスト観点.md の詳細記述（単体テスト観点の各項目定義）を全文ベースでの完全トレースは未確認。
- 認証基盤連携（Cognito/JWKS/SAML）の実環境依存設定はリポジトリ外設定のため、実接続挙動は未確認。
- 基本レイアウト機能に対するフロントエンド側の最終表示制御テスト（画面 E2E）との突合は本レビュー対象外。