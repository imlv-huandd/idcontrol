# 組織ツリー レビュー結果

## 指摘事項

### 1. 保存前確認メッセージ COM01006 が未実装
- 優先度: High
- 種別: 実装不備
- 対象: [frontend/src/pages/organizations/OrgTreeEdit.jsx](frontend/src/pages/organizations/OrgTreeEdit.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L103](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L103) 保存ボタン押下時に確認メッセージ(COM01006)を表示し、キャンセル時は終了すると定義されている。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx#L395](frontend/src/pages/organizations/OrgTreeEdit.jsx#L395) の handleSaveClick では確認ダイアログ表示なしで即時 PUT 実行している。
- 指摘内容:
  - 設計必須の確認ステップが欠落しており、誤操作で即時保存される。
- 期待されるテスト:
  - 保存押下時に確認ダイアログが表示されること。
  - 確認ダイアログでキャンセルした場合に API が呼ばれないこと。
  - 確認ダイアログで OK の場合のみ API が呼ばれること。

### 2. 表示順を更新しない要件に対し、保存時 viewIndex が破壊される
- 優先度: High
- 種別: 実装不備
- 対象: [frontend/src/pages/organizations/OrgTreeEdit.jsx](frontend/src/pages/organizations/OrgTreeEdit.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L97](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L97) ツリー移動では表示順の更新は行わないと定義されている。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx#L110](frontend/src/pages/organizations/OrgTreeEdit.jsx#L110) で viewIndex を削除し、[frontend/src/pages/organizations/OrgTreeEdit.jsx#L528](frontend/src/pages/organizations/OrgTreeEdit.jsx#L528) で未保持の viewIndex を parseInt(n.viewIndex ?? 1) として常に 1 に寄せている。
- 指摘内容:
  - 保存 payload の viewIndex が実データを保持できず、表示順に関するデータ不整合を引き起こすリスクが高い。
- 期待されるテスト:
  - 既存の複数 sibling が異なる viewIndex を持つ状態で移動保存後、非移動ノードの viewIndex が不変であること。
  - 移動ノード配下を含め、保存 payload の viewIndex が元データどおりであること。

### 3. 保存後リロード要件が未実装
- 優先度: Middle
- 種別: 実装不備
- 対象: [frontend/src/pages/organizations/OrgTreeEdit.jsx](frontend/src/pages/organizations/OrgTreeEdit.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L128](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L128) 保存後は画面をリロードすると定義されている。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx#L574](frontend/src/pages/organizations/OrgTreeEdit.jsx#L574) で alert 表示のみで、リロード処理がない。
- 指摘内容:
  - 保存後に最新履歴候補や画面状態が再初期化されず、設計と整合しない画面状態が残る可能性がある。
- 期待されるテスト:
  - 保存成功時に画面初期表示相当の再読み込みが実行されること。
  - 再読み込み後に履歴候補・選択状態・ツリー表示が最新化されること。

### 4. 不正ドロップ要件のうち「ルート構造不成立」時の即時エラーが画面で担保されていない
- 優先度: Middle
- 種別: 設計確認事項
- 対象: 組織ツリー ドラッグ&ドロップ機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L99](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L99) と [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L100](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L100) で、循環参照およびルート構造不成立をドロップ時エラーとして取り消し表示する要件。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx#L333](frontend/src/pages/organizations/OrgTreeEdit.jsx#L333) では循環参照のみを明示チェックし、ルート構造不成立の専用判定・専用メッセージはない。
- 指摘内容:
  - ルート構造不成立の判定責務をサーバー側の保存時エラーに寄せている設計解釈に見えるため、設計意図との整合確認が必要。
- 期待されるテスト:
  - ルート構造を崩す操作を試行した場合にドロップ時点で拒否されること。
  - 拒否時に設計で想定するエラーメッセージが表示されること。

### 5. メッセージ定数運用と UI 表示方式が一部不統一
- 優先度: Low
- 種別: 実装不備
- 対象: [frontend/src/pages/organizations/OrgTreeEdit.jsx](frontend/src/pages/organizations/OrgTreeEdit.jsx)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L103](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L103) と [spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L127](spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md#L127) でメッセージ ID ベースの制御を前提としている。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx#L574](frontend/src/pages/organizations/OrgTreeEdit.jsx#L574) および [frontend/src/pages/organizations/OrgTreeEdit.jsx#L337](frontend/src/pages/organizations/OrgTreeEdit.jsx#L337) で alert の固定文言を使用している。
- 指摘内容:
  - 同画面内で InforDialog/messages と alert 固定文言が混在し、文言統制と UX 一貫性が低下している。
- 期待されるテスト:
  - 成功時・エラー時ともにメッセージ定数経由の表示となること。
  - メッセージ ID ごとの表示文言が仕様どおりであること。

### 6. 単体テスト観点に対するフロントエンドテストが未整備
- 優先度: Middle
- 種別: テスト不足
- 対象: 組織ツリー フロントエンド全体
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md#L21](spec/050.テスト/テスト観点.md#L21), [spec/050.テスト/テスト観点.md#L24](spec/050.テスト/テスト観点.md#L24), [spec/050.テスト/テスト観点.md#L27](spec/050.テスト/テスト観点.md#L27), [spec/050.テスト/テスト観点.md#L30](spec/050.テスト/テスト観点.md#L30), [spec/050.テスト/テスト観点.md#L33](spec/050.テスト/テスト観点.md#L33), [spec/050.テスト/テスト観点.md#L36](spec/050.テスト/テスト観点.md#L36), [spec/050.テスト/テスト観点.md#L39](spec/050.テスト/テスト観点.md#L39) に単体テスト観点が定義されている。
  - 実装: [frontend/src/pages/organizations/OrgTreeEdit.jsx](frontend/src/pages/organizations/OrgTreeEdit.jsx) に対応するフロントエンドテストが未作成前提であり、仕様検証の自動化が未実施。
- 指摘内容:
  - 仕様観点ベースの回帰検知ができず、保存確認や編集可否、日付境界、API 異常時の挙動退行を検出できない。
- 期待されるテスト:
  - 正常系: 初期表示、履歴変更、編集遷移、保存成功。
  - 入力値バリエーション: 適用開始日 未入力/形式不正/範囲外/範囲内。
  - 境界値: endDate=9999-12-31 時のみ編集可、start/end 境界日入力。
  - 分岐条件: 権限あり/なし、isEditMode true/false、hasEdits true/false。
  - データ入出力: API 成功時のツリー反映、payload 親子関係と orgIdPath 妥当性。
  - 非更新確認: 確認ダイアログキャンセル、キャンセルボタン押下時の復元。
  - 異常系: API 4xx/5xx、競合系エラーコード、不正ドロップ時のエラー表示。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - 初期表示から履歴選択、編集遷移、保存完了までの E2E 的シナリオ検証が未整備。
- 入力値バリエーション不足:
  - 適用開始日の未入力、不正形式、trim 空白、範囲外入力の網羅が未整備。
- 境界値不足:
  - 有効開始日ちょうど、有効終了日ちょうど、9999-12-31 末端条件の検証が未整備。
- 分岐条件不足:
  - 権限別表示制御、編集モード時 disable 制御、履歴終了日別の活性制御が未整備。
- データ入出力不足:
  - buildTreeDataFromApi の親子構築、flattenNodes payload の parentOrganizationId/orgIdPath/viewIndex 妥当性検証が未整備。
- 非更新確認不足:
  - 保存確認キャンセル時の非更新、キャンセルボタンでのスナップショット復元検証が未整備。
- 異常系不足:
  - GET/PUT 通信失敗時、バックエンドエラーコード別メッセージ表示、不正ドロップ時挙動の検証が未整備。

## 総評
High 指摘は、保存前確認欠落と viewIndex 破壊リスクの 2 点で、どちらも仕様逸脱かつ誤更新につながるため優先修正が必要です。
次点として、保存後リロード未実装と不正ドロップ要件の解釈差を解消し、画面状態整合と UX の仕様準拠を担保してください。
フロントエンドテストが未整備のため、上記修正と同時に spec/050 準拠の観点テストを最低限追加しないと回帰リスクが高い状態です。

## 残留リスク・確認できなかった範囲
- 詳細設計書（spec/詳細設計 配下）での画面側エラー表示方式の規定を確認できなかったため、alert 許容可否の最終判断は未確定。
- API 仕様書上の viewIndex 取り扱い（必須/任意、未指定時挙動）を本レビューでは未確認。フロント保存 payload の期待契約は別途確認が必要。