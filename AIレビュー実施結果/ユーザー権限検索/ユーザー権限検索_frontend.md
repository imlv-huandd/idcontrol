# ユーザー権限検索 レビュー結果

## 指摘事項

- 重大な指摘事項はなし（High該当なし）

### 1. 検索結果の並び順が設計の「メールアドレス昇順」と一致していない
- 優先度: Middle
- 種別: 実装不備
- 対象: [shared/repositories/usermasterRepository.py](shared/repositories/usermasterRepository.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) に「検索結果エリアのソート順はメールアドレスの昇順」と記載。
  - 実装: [shared/repositories/usermasterRepository.py](shared/repositories/usermasterRepository.py) で `Usermaster.login_id.asc()` によりソートされており、メールアドレス基準になっていない。
- 指摘内容:
  - 画面表示順が設計と異なるため、利用者が期待する順序で検索結果を確認できない。
- 期待されるテスト:
  - メールアドレスが逆順/同値を含むデータで、返却順がメールアドレス昇順になることを API テストで検証する。

### 2. 検索ボタン押下時にページ番号を1ページ目へ確実に戻していない
- 優先度: Middle
- 種別: 実装不備
- 対象: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) の「検索ボタン押下」で「ページネーションの現在ページを1ページ目に設定」と定義。
  - 実装: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx) の `handleSearch` は `page=currentPage` を使用し、検索ボタン押下時に `setCurrentPage(1)` を行っていない。
- 指摘内容:
  - 条件変更後の再検索で前回ページ番号が維持され、結果が0件に見える/期待外のページを表示する可能性がある。
- 期待されるテスト:
  - 2ページ目以降を表示中に検索条件を変更して検索した際、`currentPage=1` で再検索されること。

### 3. 通信失敗時のエラー文言が画面に表示されない
- 優先度: Middle
- 種別: 実装不備
- 対象: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) はイベント処理を定義しているが、障害時の運用性を担保するエラー表示要件の明記は弱い。
  - 実装: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx) で `setError(...)` は実行されるが、`error` を描画していない。`AlertMessage` import も未使用。
- 指摘内容:
  - APIエラー時に利用者へ失敗理由が伝わらず、再試行判断や問い合わせ時の情報が不足する。
- 期待されるテスト:
  - API 500/ネットワークエラー時にエラーメッセージ領域が表示されること。
  - エラー後に再検索成功した際、エラー表示がクリアされること。

### 4. 連携システム権限の表示形式が設計の括弧表記と異なる
- 優先度: Middle
- 種別: 実装不備
- 対象: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) に「連携システム管理者/操作者は末尾に『（【連携システム名】）』を付与」と記載。
  - 実装: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx) の `getPermissionDisplay` は `ラベル: systemDisplayName` 形式で表示。
- 指摘内容:
  - 表記ゆれにより、設計書・他画面・帳票との整合が崩れる可能性がある。
- 期待されるテスト:
  - `RELATION_SYSTEM_ADMIN`/`RELATION_SYSTEM_USER` で「権限名（システム名）」の形式になること。
  - その他権限ではシステム名が付与されないこと。

### 5. フロントエンド単体テストが未整備で、仕様回帰を検知できない
- 優先度: Middle
- 種別: テスト不足
- 対象: ユーザー権限検索機能（フロントエンド）
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md) の「単体テスト観点（正常系/入力値バリエーション/境界値/分岐条件/データ入出力/非更新確認/異常系）」。
  - 実装: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx) に対する `.test`/`.spec` が存在しない（SubAgent探索結果）。
- 指摘内容:
  - 画面条件入力、検索実行、ページング、エラー表示、編集遷移などの主要仕様に対して自動検証が無く、改修時の退行リスクが高い。
- 期待されるテスト:
  - 正常系: 条件入力→検索→結果表示、0件表示、編集遷移。
  - 分岐: `includeGroupPermissions` ON/OFF、権限選択あり/なし。
  - 異常系: APIエラー時メッセージ表示。
  - 非更新確認: クリア押下で結果エリア非表示、検索条件初期化。

### 6. 未使用 state/import が残存し、保守時の誤読要因になっている
- 優先度: Low
- 種別: 実装不備
- 対象: [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) に確認ダイアログ要件は明記されていない。
  - 実装: `AlertMessage` import、`hasSearched`、`confirmDialog`（open制御以外）が実質未使用。
- 指摘内容:
  - 実際に使われないコードが残ることで、仕様の存在有無が判別しづらくなり、将来改修の理解コストが増える。
- 期待されるテスト:
  - なし。

### 7. ルートパラメータ名の設計意図が不明瞭
- 優先度: Low
- 種別: 設計確認事項
- 対象: [frontend/src/App.jsx](frontend/src/App.jsx), [frontend/src/pages/permissions/UserPermissionSearch.jsx](frontend/src/pages/permissions/UserPermissionSearch.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) は「ユーザーIDをパラメータとして遷移」と定義。
  - 実装: 遷移元は `/user-permissions/${user.userId}/edit`、ルート定義は `/user-permissions/:id/edit`。
- 指摘内容:
  - 動作は可能だが、`id` が `userId` であることがコード上で明示されず、別IDとの混同余地がある。
- 期待されるテスト:
  - ルート遷移時に編集画面側が期待するIDでAPI呼び出しされること。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 条件入力→検索結果表示、0件時メッセージ表示、編集遷移の画面テストが未実装。
- 入力値バリエーション不足
  - `permissions` 未選択/複数選択、`includeGroupPermissions` ON/OFF、各検索条件の単独/組み合わせ入力の検証が未実装。
- 境界値不足
  - ページング境界（1ページ目、最終ページ、20件ちょうど/21件）の表示・再検索動作テストが未実装。
- 分岐条件不足
  - 連携システム権限のみシステム名付与、その他権限は付与しない分岐の検証が未実装。
- データ入出力不足
  - APIクエリパラメータ（`permissions` 連結、`includeGroupPermissions`、`page`、`itemsPerPage`）の送信値検証が未実装。
- 非更新確認不足
  - クリア時に検索結果エリア非表示・条件初期化されること、および不要API呼び出しが無いことの検証が未実装。
- 異常系不足
  - API 4xx/5xx/通信失敗時のエラー表示・回復動作テストが未実装。

## 総評
- High指摘はありませんが、設計との乖離として「検索結果ソート順」「検索時ページ初期化」「連携システム権限の表記」に実装不一致があります。
- 画面の主要導線（検索・ページング・エラー表示）に対するフロントエンド自動テストが未整備で、回帰検知力が不足しています。
- 優先順位は、まず仕様乖離（並び順・ページ初期化・エラー表示）を修正し、その後に単体テストの最小セットを追加するのが妥当です。

## 残留リスク・確認できなかった範囲
- [spec/詳細設計/openapi/paths/users.yaml](spec/詳細設計/openapi/paths/users.yaml) はユーザー権限取得/更新API中心で、検索APIの詳細定義が十分に確認できませんでした。
- 画面の権限制御（閲覧可否）の要件定義が参照設計書内で明確でなく、アクセス制御の妥当性は本レビュー範囲外です。