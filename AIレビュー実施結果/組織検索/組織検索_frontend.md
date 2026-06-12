# 組織検索 レビュー結果

## 指摘事項

### 1. 検索実行時に未定義関数呼び出しで処理が失敗する
- 優先度: High
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 検索ボタン押下。検索結果を一覧表示する前提で、検索処理が正常完了する必要がある。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:134 で setHasSearched(true) を呼び出しているが、state定義が存在しない。
- 指摘内容:
  - 検索成功時に ReferenceError が発生し、検索結果表示や後続処理が中断される可能性がある。
- 期待されるテスト:
  - 検索API成功レスポンス時に例外なく一覧表示されること。
  - 検索API成功時に state更新がすべて実行され、エラー表示が出ないこと。

### 2. 廃止済レコードの廃止ボタンが無効化されていない
- 優先度: High
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 項目定義 No.17。廃止済の組織は廃止ボタンを無効化する仕様。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:337 の廃止済ボタンに disabled 属性がなく、クラスに btn:disabled を文字列指定しているのみ。
- 指摘内容:
  - 廃止済データに対しても操作可能に見える実装で、仕様に反する。誤操作誘発や不要API呼び出しのリスクがある。
- 期待されるテスト:
  - isAbolished=true の行で廃止ボタンが disabled になること。
  - disabled 行でクリックしても廃止ダイアログが開かないこと。

### 3. 廃止実行後の再検索が仕様と異なり、二重実行も発生する
- 優先度: High
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 廃止実行ボタン押下。現在ページで再検索し、該当なし時は最終ページ表示。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:121 は該当なし時に 1ページ目へフォールバック。frontend/src/pages/organizations/OrganizationSearch.jsx:236 と frontend/src/pages/organizations/OrganizationSearch.jsx:246 で handleSearch() を重複実行。
- 指摘内容:
  - 仕様の最終ページ遷移と不一致で、ユーザーが意図しないページへ遷移する。加えて二重検索により不要な通信と画面ちらつきが発生する。
- 期待されるテスト:
  - 廃止後に現在ページが空になった場合、最終ページに遷移すること。
  - 廃止成功時に検索API呼び出しが1回であること。

### 4. フロント側の権限制御（ボタン表示制御）が未実装
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 項目定義 No.17。CSV出力はテナント管理者・マスターデータ管理者・ダウンロード権限時のみ表示。廃止ボタンはテナント管理者・マスターデータ管理者時のみ表示。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:300 および frontend/src/pages/organizations/OrganizationSearch.jsx:326 で権限判定なしにボタン表示。frontend/src/common/UserAuthority.jsx には権限判定関数が存在。
- 指摘内容:
  - サーバー側で拒否される前提でも、UI仕様逸脱かつ不要操作導線となる。利用者に誤解を与える。
- 期待されるテスト:
  - 権限ごとに CSV出力/廃止ボタンの表示・非表示が仕様どおりになること。
  - 無権限時に当該操作UIが表示されないこと。

### 5. 検索条件ラベルが仕様文言と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 項目定義 No.10。検索条件は「検索基準日」。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:283 で「有効終了日」を表示。
- 指摘内容:
  - 仕様文言と異なり、ユーザーが入力意図を誤認する。
- 期待されるテスト:
  - 画面初期表示時に検索条件ラベルが設計書文言と一致すること。

### 6. 詳細表示へのパラメータ受け渡しが設計記述と不一致
- 優先度: Middle
- 種別: 設計確認事項
- 対象: 組織検索機能（詳細ボタン）
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 詳細ボタン押下。組織IDと検索基準日をパラメータとして遷移。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:334 で organizationId, startDate, endDate を渡しており、検索基準日は受け渡していない。
- 指摘内容:
  - 仕様の「検索基準日で対象レコードを特定する」前提と異なる実装に見える。意図的変更か設計未更新かを確認すべき。
- 期待されるテスト:
  - 詳細ボタン押下時に、仕様で要求されるパラメータが引き渡されること。

### 7. 廃止日の状態管理実装が不安定で将来不具合の温床
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/AbolishDialog.jsx, frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 廃止確認ダイアログ。廃止日は日付入力として扱う。
  - 実装: frontend/src/pages/organizations/AbolishDialog.jsx:59 で文字列stateに対しオブジェクト形式で更新。frontend/src/pages/organizations/OrganizationSearch.jsx:222 では abolishDate.baseDate を参照。
- 指摘内容:
  - 廃止日の型が一貫しておらず、入力・送信処理の保守性が低い。入力UIの value 未指定もあり状態不整合リスクがある。
- 期待されるテスト:
  - 廃止日入力値が画面表示と送信payloadに一貫して反映されること。
  - 初期値、入力後、再表示時で型不整合が発生しないこと。

### 8. フロントエンド単体テストが未整備で、仕様観点の検証が未実施
- 優先度: Middle
- 種別: テスト不足
- 対象: 組織検索機能（frontend/src/pages/organizations/OrganizationSearch.jsx ほか）
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の 単体テスト観点（正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系）。
  - 実装: フロントエンドの OrganizationSearch 関連テストコードが未作成。既存 tests/functions/organizations/test_organizations.py はAPIハンドラ中心でUI仕様を検証していない。
- 指摘内容:
  - 画面仕様に対する品質担保ができておらず、回帰不具合を検知できない。
- 期待されるテスト:
  - 単体テスト観点に準拠した UI/状態/API連携テストを最低限一式追加すること。

### 9. 画面内の未使用・不整合状態管理が残存
- 優先度: Low
- 種別: 実装不備
- 対象: frontend/src/pages/organizations/OrganizationSearch.jsx
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md の 画面初期表示・検索表示制御。
  - 実装: frontend/src/pages/organizations/OrganizationSearch.jsx:27 の organizations state が表示に未使用。frontend/src/pages/organizations/OrganizationSearch.jsx:49 で modalInstance.current を参照するが宣言なし。
- 指摘内容:
  - 不要状態や不整合コードが可読性を下げ、将来改修時のバグ混入リスクを高める。
- 期待されるテスト:
  - 状態管理の単純化後も、初期表示/検索/ダイアログ操作の挙動が変わらないこと。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - 検索成功時の一覧表示、0件表示、詳細モーダル表示、廃止成功メッセージ表示のUI検証がない。
- 入力値バリエーション不足:
  - tenantOrgId/orgName/orgNameAll/orgNameDisplay/baseDate/includeAbolished の組合せ別検証がない。
- 境界値不足:
  - ページ境界（1ページ目/最終ページ/削除後ページ欠損）、日付入力境界、空文字境界の検証がない。
- 分岐条件不足:
  - 権限あり/なし、isAbolished true/false、API成功/失敗、検索結果あり/なしの分岐テストがない。
- データ入出力不足:
  - APIクエリ文字列の組み立て、CSVダウンロード生成、日付表示変換（yyyy/mm/dd）の検証がない。
- 非更新確認不足:
  - クリア操作時に不要なAPI呼び出しを行わないこと、廃止済行で廃止処理が走らないことの検証がない。
- 異常系不足:
  - 通信失敗、権限不足(COM01001)、対象なし(COM01003/COM01009)、競合(COM01010)、入力エラー(COM02001/COM02006/ORG01001)表示の検証がない。

## 総評
- High指摘は、検索処理中断（未定義関数）、廃止ボタン無効化不備、廃止後ページング/再検索の仕様逸脱であり、いずれもユーザー操作に直接影響する。
- Middle指摘では権限制御UI未実装とパラメータ仕様不一致、テスト未整備が品質リスクの中心。
- まず High を優先修正し、次に権限制御とテスト基盤整備を実施する順序が妥当。

## 残留リスク・確認できなかった範囲
- 詳細設計配下に組織検索の該当設計書を確認できず、基本設計との完全トレースに制約がある。
- レビュー観点配下に組織検索向けの個別観点文書は未確認。
- サーバー側（functions/shared）の実装詳細までは本レビューの主対象外のため、権限・検索条件の最終担保は別途APIレビューが必要。