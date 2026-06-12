# 役職検索 レビュー結果

## 指摘事項

### 1. テナント設定取得APIで認可チェックが欠落している
- 優先度: High
- 種別: 実装不備
- 対象: functions/tenants/getTenant/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md では、テナント設定（役職コードカラム名）を前提として画面表示する仕様であり、テナント境界を越えた参照は想定していない。
  - 実装: functions/tenants/getTenant/app.py で get_current_user(event) および login_tenant_id の利用がコメントアウトされ、tenantId パラメータに対する認可検証が行われていない。
- 指摘内容:
  - 認証は通るが認可が実質未実装のため、tenantId を指定して他テナント設定を取得できる余地がある。権限制御要件として重大な欠陥。
- 期待されるテスト:
  - ログインユーザーの tenantId と異なる tenantId を指定した場合に 403 となること。
  - tenantId 未指定/不正値時の 400/401/403 の境界を検証すること。

### 2. CSV出力の項目順が設計仕様と不一致
- 優先度: High
- 種別: 実装不備
- 対象: functions/organizationPositions/searchOrgPositionsCsv/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md の「CSVファイルの出力」で、出力項目順（テナント役職ID→有効開始日→有効終了日→役職名称→廃止）を明示。
  - 実装: functions/organizationPositions/searchOrgPositionsCsv/app.py では列順が設計順と一致しない実装箇所がある。
- 指摘内容:
  - 仕様書ベースの下流連携（利用者の取り込みマクロ、運用手順）に影響するため、項目順の不一致は重大。
- 期待されるテスト:
  - ヘッダ順とデータ列順が設計通りであることをCSV文字列で厳密比較するテスト。
  - 0件時 COM01003、権限不足時 COM01001 を返すテスト。

### 3. CSV出力エラー時のメッセージ制御が設計のメッセージID運用に追随できていない
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/PositionSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md のバリデーション定義で COM01001、イベント詳細で0件時 COM01003 を明示。
  - 実装: frontend/src/pages/organizations/PositionSearch.jsx の handleExportCsv はエラー時に response.json().message や err.message をそのまま表示しており、メッセージID前提の表示制御がない。
- 指摘内容:
  - API応答形式差異や想定外レスポンス時に、ユーザーへ仕様外メッセージが露出する可能性がある。
- 期待されるテスト:
  - 403（権限なし）、0件、500、非JSONエラー本文時の表示メッセージを検証するテスト。

### 4. 詳細ダイアログクローズ時に無条件で再検索され、未検索状態の画面挙動を崩す
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/PositionSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md の画面初期表示で「検索結果エリア非表示」「ページング非表示」を定義。
  - 実装: frontend/src/pages/organizations/PositionSearch.jsx の PositionDetail onCancel で loadOrgPosition() を常時実行し、未検索でも hasSearched が true 化されうる。
- 指摘内容:
  - 初期表示/未検索時のUI状態を保持できず、ユーザー操作意図と異なるタイミングで検索実行される。
- 期待されるテスト:
  - 未検索状態で新規登録→キャンセルした場合に検索結果エリアが表示されないこと。
  - 既検索状態ではクローズ時に同条件・同ページ再取得されること。

### 5. 廃止確認ダイアログのクライアント入力チェックが最小要件のみで境界ケースの防御が弱い
- 優先度: Middle
- 種別: 設計確認事項
- 対象: frontend/src/pages/organizations/AbolishPositionDialog.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md ではクライアント側は「入力チェック」、サーバー側で廃止対象チェック（有効開始日と廃止日の整合）を定義。
  - 実装: frontend/src/pages/organizations/AbolishPositionDialog.jsx は必須・日付形式のみで、日付範囲の事前ガードは未実装。
- 指摘内容:
  - 設計上サーバー判定でも成立するが、UXと不要API呼び出し抑制の観点で、境界値（有効開始日前など）のクライアント早期検出方針を明確化すべき。
- 期待されるテスト:
  - 有効開始日より前の廃止日入力時の挙動（クライアント警告 or サーバーエラー表示）を仕様として固定し検証。

### 6. 単体テスト観点（spec/050）に対してフロントエンドのテストが未整備
- 優先度: Middle
- 種別: テスト不足
- 対象: 役職検索機能（frontend/src/pages/organizations/PositionSearch.jsx, AbolishPositionDialog.jsx）
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md で 正常系/入力値バリエーション/境界値/分岐条件/データ入出力/非更新確認/異常系 の観点を定義。
  - 実装: フロントエンド側の対応テストコードが未作成（前提条件と一致）。
- 指摘内容:
  - 画面の主要分岐（権限表示、検索0件、廃止済ボタン非活性、通信失敗時表示）が継続的に保証されない。
- 期待されるテスト:
  - 正常系: 検索成功、0件表示、詳細遷移、新規登録遷移、CSV出力成功。
  - 異常系: 各API失敗（401/403/400/500）時のエラー表示。
  - 分岐: download権限有無でCSVボタン表示切替、isAbolishedで廃止ボタン活性切替。

### 7. 既存バックエンドテストが仕様上重要な分岐を十分に担保していない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/organizationPositions/test_organizationPositions.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の分岐条件・非更新確認・異常系観点。
  - 実装: tests/functions/organizationPositions/test_organizationPositions.py では、権限不足分岐、CSV出力分岐、times不一致時の非更新確認、0件時メッセージ等の網羅が不足。
- 指摘内容:
  - フロントの期待挙動を支えるAPI契約がテストで固定化されておらず、回帰リスクが高い。
- 期待されるテスト:
  - 403時レスポンス、0件時COM01003、更新失敗時COM01010、times不一致でDB非更新を追加。

### 8. 未使用状態と命名ゆれがあり可読性・保守性を低下させる
- 優先度: Low
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/PositionSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md は ConfirmDialog の利用イベントを定義していない。
  - 実装: confirmDialog state と ConfirmDialog 描画が実質未使用、handleAbolist の命名揺れ、openOrgPositionDetail 呼び出しで不要引数が渡されている。
- 指摘内容:
  - 挙動への直接影響は小さいが、将来改修時の誤読・誤用リスクになる。
- 期待されるテスト:
  - なし（リファクタ対象）。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 検索成功時の一覧表示、詳細遷移、新規登録遷移、CSV出力成功の画面テストが未整備。
- 入力値バリエーション不足
  - テナント役職ID/役職名称/検索基準日/廃止済含むの組み合わせによる表示差分の検証が未整備。
- 境界値不足
  - 検索基準日未入力、廃止日境界（有効開始日当日/前日）、最終ページ遷移境界の検証が未整備。
- 分岐条件不足
  - download権限有無、isAbolished true/false、API結果0件/1件/複数件の分岐検証が不足。
- データ入出力不足
  - CSVヘッダ順・列順・文字コード、APIレスポンス項目の画面反映（日付整形・9999/12/31非表示）の検証が不足。
- 非更新確認不足
  - times不一致時に更新されないこと、権限不足時に状態変更されないことの検証が不足。
- 異常系不足
  - 401/403/400/500、非JSONエラーレスポンス、通信失敗時のエラー表示と復帰動作の検証が未整備。

## 総評
- High指摘は、テナント設定取得APIの認可欠落とCSV仕様不一致であり、いずれもリリース前の修正必須です。
- 画面自体の基本機能は概ね実装されていますが、エラー時制御と状態遷移で設計意図からのズレが残っています。
- テストはフロント未作成に加え、API側も仕様固定に必要な分岐が不足しており、回帰リスクが高い状態です。
- 優先順位は High修正 → API契約テスト補強 → フロント単体テスト整備 の順が妥当です。

## 残留リスク・確認できなかった範囲
- spec/詳細設計/openapi/paths/org-positions.yaml のoperation定義と実装の完全一致までは本レビューで未確認。
- 画面共通コンポーネント（Pagination, AlertMessage, LoadingModal）の内部実装詳細は本レビューで深掘り未実施。
- getTenant APIの認可要件が別設計書で定義されている場合、その整合は追加確認が必要。
