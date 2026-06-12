# コード変換グループ編集 レビュー結果

## 指摘事項

### 1. 削除ボタンの1行時非活性制御が未実装
- 優先度: High
- 種別: 実装不備
- 対象: CodeConversionGroupEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md](spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md#L108) 行が1行しかない場合は削除ボタン非活性と定義
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L475) 削除ボタンの disabled が loading のみで、codes.length===1 条件がない
- 指摘内容:
  - 設計では最終1行の削除は禁止だが、実装は1行でも削除できるため設計逸脱。0件状態を許容し、ユーザー操作で不要なエラー遷移を誘発する。
- 期待されるテスト:
  - codes.length=1 で削除ボタンが非活性、codes.length>=2 で活性になることを画面テストで検証する。

### 2. 変換後値重複チェックの仕様が設計書に未定義
- 優先度: High
- 種別: 設計確認事項
- 対象: CodeConversionGroupEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md](spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md#L140) バリデーション定義には変換元値重複のみ定義され、変換後値重複の記載がない
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L250) afterValue に対する重複チェックを実装
- 指摘内容:
  - 実装は変換後値重複をエラー扱いするが、設計根拠がない。業務仕様として必要なら設計書追記、不要なら実装見直しが必要。
- 期待されるテスト:
  - afterValue が重複した場合にエラーとする/しないの期待仕様を確定し、確定仕様どおりの振る舞いを検証する。

### 3. コード変換名称重複時のメッセージ運用が設計定義と不整合
- 優先度: Middle
- 種別: 実装不備
- 対象: CodeConversionGroupEdit.jsx, messageJP.ts
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md](spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md#L143) 重複チェックは変換元値に対して COD04002 を定義
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L234) コード変換名称重複でも COD04002 を使用。 [frontend/src/constants/messageJP.ts](frontend/src/constants/messageJP.ts#L160) COD04002 は値重複メッセージ
- 指摘内容:
  - コード変換名称重複のエラー定義が設計上明確でないまま、変換元値用メッセージを流用している。利用者にとってエラー対象項目が曖昧。
- 期待されるテスト:
  - コード変換名称重複時に表示されるメッセージが仕様定義と一致することを検証する。

### 4. 重複エラーメッセージの割当てが項目別に不明瞭
- 優先度: Middle
- 種別: 実装不備
- 対象: CodeConversionGroupEdit.jsx, messageJP.ts
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md](spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md#L143) 変換元値重複チェックのメッセージIDを定義
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L242) beforeValue 重複で COD04001 を使用。 [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L250) afterValue 重複でも COD04001 を使用。 [frontend/src/constants/messageJP.ts](frontend/src/constants/messageJP.ts#L159) COD04001 はグループ名重複メッセージ
- 指摘内容:
  - 項目とメッセージIDの対応が崩れており、重複エラーがどの仕様に基づくものか判別しにくい。保守時の誤改修リスクが高い。
- 期待されるテスト:
  - groupName、codeConversionName、beforeValue、afterValue の重複シナリオを分離し、各項目で期待メッセージID/文言が出ることを検証する。

### 5. フロントエンド単体テストが未整備
- 優先度: Middle
- 種別: テスト不足
- 対象: コード変換グループ編集機能
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L20) 以降で単体テスト観点（正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系）を定義
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx) に対応する frontend/src 配下の test/spec ファイルが未検出。 [tests/functions/codeConversionGroups/test_codeConversionGroups.py](tests/functions/codeConversionGroups/test_codeConversionGroups.py#L1) は functions レイヤーのテストで画面テストではない
- 指摘内容:
  - 画面仕様に対する自動テストが無いため、分岐・異常系・権限制御の回帰を検知できない。
- 期待されるテスト:
  - 最低限、正常登録/更新、入力必須、100文字境界、重複、0件、削除確認、キャンセル確認、編集読込失敗(COM01009)、更新競合(COM01010)、権限別選択肢制御を網羅する。

### 6. 成功時遷移の1.5秒遅延が設計に明記されていない
- 優先度: Low
- 種別: 設計確認事項
- 対象: CodeConversionGroupEdit.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md](spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md#L112) 完了後は検索画面に遷移と記載
  - 実装: [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L325) と [frontend/src/pages/codes/CodeConversionGroupEdit.jsx](frontend/src/pages/codes/CodeConversionGroupEdit.jsx#L384) で 1500ms 後に遷移
- 指摘内容:
  - 現行実装は UX 配慮として妥当だが、遷移タイミングの仕様差分が設計に残っていない。
- 期待されるテスト:
  - 成功メッセージ表示後に遅延遷移する仕様を採用する場合、遷移タイミングをテストで固定する。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - 新規登録成功、編集更新成功、行追加、行並び替え後の viewIndex 再採番の確認が未自動化。
- 入力値バリエーション不足:
  - groupName/adminSystem/codeConversionName/beforeValue/afterValue の組み合わせ網羅がない。
- 境界値不足:
  - 100文字ちょうど、101文字、空白のみ入力、最小件数1件の境界確認がない。
- 分岐条件不足:
  - TENANT_ADMIN と RELATION_SYSTEM_ADMIN の選択肢分岐、isEditMode の表示分岐、確認ダイアログ confirm/cancel 分岐が未検証。
- データ入出力不足:
  - 取得API結果の画面反映、更新時 body 生成（times/viewIndex/codeConversionId）の妥当性検証がない。
- 非更新確認不足:
  - hasChanged=false 時キャンセル即遷移、hasChanged=true 時 COM01008 表示・キャンセルで遷移抑止の確認がない。
- 異常系不足:
  - 初期取得失敗(COM01009)、登録/更新失敗時メッセージ、通信失敗時 alert 表示の検証がない。

## 総評
- High 指摘として、削除ボタンの非活性要件未実装と、変換後値重複チェックの設計未定義がリリース前に要是正です。
- Middle 指摘は主にメッセージID運用の不整合と画面テスト不在で、将来の誤改修・回帰検知漏れリスクが高い状態です。
- 修正優先順位は、1) 削除活性制御 2) 重複チェック仕様確定 3) メッセージID整理 4) 主要分岐の自動テスト整備を推奨します。

## 残留リスク・確認できなかった範囲
- 変換後値重複チェックの業務要件は設計書に記載がなく、仕様確定が必要。
- フロントの E2E テスト基盤有無（Playwright/Cypress 等）は本レビュー範囲で未確認。
- バックエンド側の最終メッセージID返却がフロント表示文言と完全一致するかは、API統合試験で要確認。