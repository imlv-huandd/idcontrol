# ユーザー権限編集 レビュー結果

## 指摘事項

### 1. 連携システム操作者セクションが設計より広く表示される
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L109-L111) では、連携システム操作者セクションは「ログインユーザーがテナント管理者または連携システム管理者権限を1つ以上持つ場合のみ表示」と定義されている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L358) で、表示条件に `isRelationSystemUser` が含まれている。
- 指摘内容:
  - 連携システムユーザー権限のログインユーザーにも操作者セクションが表示されるため、設計書の表示制御と一致していない。
- 期待されるテスト:
  - ログインユーザーが連携システムユーザーのみの場合に操作者セクションが非表示になること。
  - ログインユーザーがテナント管理者、または連携システム管理者の場合に操作者セクションが表示されること。

### 2. グループ権限の重複排除が実装されていない
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L73) で、グループ権限は「重複排除して改行区切り」で表示すると定義されている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L291-L305) では、取得した `groupPermissions` をそのまま並べ替えて表示しており、重複排除処理がない。
- 指摘内容:
  - 複数グループに同一権限が付与されている場合、同じ権限が重複して画面表示される。
- 期待されるテスト:
  - 同一権限が複数グループから返るケースで、表示が1件に集約されること。
  - グループ権限が0件のケースで「（未設定）」が表示されること。

### 3. キャンセル時の未保存変更確認がない
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L161) では、入力内容が変更されている場合に確認ダイアログ(COM01008)を表示すると定義されている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L445) のキャンセルボタンは、そのまま一覧画面へ遷移している。
- 指摘内容:
  - 変更済みかどうかの判定がなく、未保存の権限変更を破棄する確認が表示されない。
- 期待されるテスト:
  - 変更後にキャンセルを押したとき、確認ダイアログが表示されること。
  - ダイアログでキャンセルした場合、画面遷移せず編集状態が維持されること。
  - ダイアログで確定した場合のみ一覧へ遷移すること。

### 4. 初期取得失敗時の遷移制御と更新ガードが不足している
- 優先度: Middle
- 種別: 実装不備
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L123) では、ユーザーが存在しない場合はCOM01009を表示してユーザー権限検索画面へ遷移すると定義されている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L175-L201) は取得失敗時にアラート表示のみで、検索画面への遷移がない。さらに [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L245) で `userPermissions.times` を直接参照しており、初期取得失敗時に submit すると例外化する。
- 指摘内容:
  - ユーザー未存在や初期取得失敗のときに設計通りの遷移にならず、更新ボタンも有効なため、エラー状態で更新処理へ進めてしまう。
- 期待されるテスト:
  - getUser 失敗時にCOM01009が表示され、検索画面へ遷移すること。
  - 初期取得失敗後に更新を押しても例外にならず、更新処理が抑止されること。
  - API失敗時にローディングが解除され、エラーメッセージだけが表示されること。

### 5. 連携システムの表示順がシステム名昇順になっていない
- 優先度: Low
- 種別: 実装不備
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L86) で、連携システム管理者／操作者セクションのシステム表示順はシステム名の昇順と定義されている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L68-L74) の `getRelationSystemEntries()` はソートを行わず、[frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L345-L376) でも取得順のまま描画している。
- 指摘内容:
  - システム一覧の表示順が設計書どおりに保証されないため、複数件存在する場合に見た目が不定になる。
- 期待されるテスト:
  - 連携システムが複数件ある場合に、システム名昇順で表示されること。
  - 取得順が昇順でなくても、画面上は昇順に整列されること。

### 6. ログイン権限別の表示分岐を検証する単体テストがない
- 優先度: Middle
- 種別: テスト不足
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L109-L111) で、権限ごとに表示対象が厳密に分かれている。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L338-L358) は権限条件分岐が多く、連携システム操作者の表示条件も誤って広い。
- 指摘内容:
  - 現状テストコードがない前提では、テナント管理者、連携システム管理者、連携システムユーザー、マスターデータ管理者、一般ユーザーの表示分岐が未検証で、誤表示を見逃しやすい。
- 期待されるテスト:
  - 各ログイン権限で表示されるチェックボックス/セクションを検証するケース。
  - 連携システムユーザーでは操作者セクションが出ないことを検証するケース。

### 7. 更新・キャンセル・異常系の単体テストが不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: frontend/src/pages/permissions/UserPermissionEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md#L123-L158) では、ユーザー不存在、確認ダイアログ、タイムスタンプ付き更新、操作ログ記録など複数の分岐がある。
  - 実装: [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L175-L201) [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L245-L275) [frontend/src/pages/permissions/UserPermissionEdit.jsx](frontend/src/pages/permissions/UserPermissionEdit.jsx#L445-L446) に、API失敗・未取得状態・キャンセル遷移の分岐がある。
- 指摘内容:
  - 正常系だけでなく、ユーザー不存在、初期取得失敗、更新失敗、未保存キャンセルの異常系が未検証のため、仕様逸脱や実行時例外を見逃しやすい。
- 期待されるテスト:
  - getUser/getUserPermissions/listRelationSystems の失敗時にエラー表示が出ること。
  - 更新失敗時にアラートが出て画面遷移しないこと。
  - 変更なしのキャンセルと変更ありのキャンセルで挙動が分かれること。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足: 初期表示でユーザー情報、現在権限、グループ権限、初期チェック状態が設計どおりに出るケースがない。
- 分岐条件不足: ログイン権限ごとの表示分岐、連携システムユーザーの非表示、更新可否の分岐を検証するケースがない。
- 入力値バリエーション不足: 権限の組み合わせ、複数連携システム選択、グループ権限の重複あり/なしを検証するケースがない。
- データ入出力不足: 初期取得APIの戻り値と更新ペイロードの対応、タイムスタンプ付き更新、表示順の保証を検証するケースがない。
- 非更新確認不足: 未保存変更がある場合のキャンセル確認と、取消後に更新されないことを検証するケースがない。
- 異常系不足: ユーザー不存在、初期取得失敗、更新失敗、null状態での更新押下を検証するケースがない。

## 総評
High相当のセキュリティ問題は見当たりませんでしたが、画面制御と初期表示/終了時の分岐に設計逸脱が複数あります。
特に、連携システム操作者セクションの表示条件、未保存キャンセル、初期取得失敗時の遷移と更新ガードは、実利用で誤操作や例外につながりやすいです。
テストコードがない前提では、権限分岐と異常系の検証を優先して追加する必要があります。

## 残留リスク・確認できなかった範囲
- 実画面での操作確認と自動テスト実行は行っていないため、ブラウザ上の挙動は静的レビューに限られます。
- バックエンドAPIの実行結果に依存する部分は、フロント実装のみで完全には確認できません。