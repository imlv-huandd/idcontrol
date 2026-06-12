# 役職情報詳細 レビュー結果

## 指摘事項

### 1. 役職未存在時に COM01009 ではなく例外クラッシュになる
- 優先度: High
- 種別: 実装不備
- 対象: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L127-L147)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) に「役職が存在しない場合、エラーメッセージ(COM01009)を表示して終了」と定義。
  - 実装: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L127-L147) で役職不在の明示判定がなく、履歴先頭参照で例外化する経路がある。
- 指摘内容:
  - 設計で求める業務エラー返却ではなく、内部例外に落ちる可能性がある。
- 期待されるテスト:
  - 異常系: 存在しない orgPositionId 指定時に COM01009 相当の業務エラーが返ること。

### 2. 更新時の楽観ロック不一致で null 参照経路が発生する
- 優先度: High
- 種別: 実装不備
- 対象: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L166-L175)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) の更新仕様では、更新対象が見つからない場合 COM01010 を返すことが定義。
  - 実装: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L166-L175) で取得結果不在時の防御が不十分な経路があり、後続比較で例外化し得る。
- 指摘内容:
  - 並行更新時に業務エラー化されず、500 系に倒れるリスクがある。
- 期待されるテスト:
  - 異常系: `times` 不一致時に COM01010 が返り、例外クラッシュしないこと。

### 3. ハンドラ単体テストがサービス層を全面モックし、主要仕様を検証できていない
- 優先度: High
- 種別: テスト不足
- 対象: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) でサーバーサイドバリデーション（権限、重複、変更有無、履歴追加等）が明示。
  - 実装: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py#L35-L218) は create/get/update のサービス呼び出しを中心にモックし、仕様本体の検証が弱い。
- 指摘内容:
  - 正常/異常で HTTP ステータスのみ確認する構成が多く、仕様エラーコードや業務分岐の正しさを担保できない。
- 期待されるテスト:
  - 正常系/異常系/入力値バリエーション: ORG05001, ORG05002, ORG05003, COM01009, COM01010 の個別検証。
  - 分岐条件: 有効開始日が同一/変更の2分岐でDB更新形態が変わること。

### 4. getOrgPosition のクエリ入力検証が不足している
- 優先度: Middle
- 種別: 実装不備
- 対象: [functions/organizationPositions/getOrgPosition/app.py](functions/organizationPositions/getOrgPosition/app.py#L31-L38)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) で「パラメータで指定された役職ID」を前提に履歴取得。
  - 実装: [functions/organizationPositions/getOrgPosition/app.py](functions/organizationPositions/getOrgPosition/app.py#L31-L38) では orgPositionId 空文字や baseDate 形式不正の事前検証がない。
- 指摘内容:
  - 不正入力時に業務エラーへ正規化されず、下流依存で予期せぬ例外化の余地がある。
- 期待されるテスト:
  - 入力値バリエーション: orgPositionId 未指定/不正形式、baseDate 不正形式でのエラー応答。

### 5. 追加属性比較・変換処理が実装依存で壊れやすい
- 優先度: Middle
- 種別: 実装不備
- 対象: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L337-L360)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) で追加情報は可変項目として動的に扱う要件。
  - 実装: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L337-L360) は dict 変換や getattr ベースの比較に依存し、属性構造変化に脆い。
- 指摘内容:
  - 属性定義追加・変更時に比較漏れや例外の温床となる。
- 期待されるテスト:
  - データ入出力/分岐条件: 可変属性の有無、未知キー、空値、型差異で差分判定が仕様どおりになること。

### 6. 更新時の操作ログ記録値が仕様意図とずれる可能性
- 優先度: Middle
- 種別: 設計確認事項
- 対象: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L228-L236)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) で操作内容「役職情報更新:[テナント役職ID]」を記録。
  - 実装: [shared/services/organizationPositionService.py](shared/services/organizationPositionService.py#L228-L236) は更新時のどの値（変更前/変更後）を採用するかが読み取りづらい。
- 指摘内容:
  - 監査ログの解釈が実運用でぶれる懸念があるため、仕様の明文化と実装整合が必要。
- 期待されるテスト:
  - データ入出力: テナント役職ID変更あり/なしで、ログメッセージの期待値を固定化。

### 7. 境界値テスト（文字数・数値・日付）が不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) のバリデーション定義で 100 文字・3桁・日付形式等を要求。
  - 実装: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py#L104-L218) で境界入力の明示ケースが不足。
- 指摘内容:
  - 入力制約の境界不具合を検知できない。
- 期待されるテスト:
  - 境界値/入力値バリエーション: 100/101文字、999/1000、日付不正、必須欠落、空文字。

### 8. 非更新確認（他データ不変）のテストが不足している
- 優先度: Low
- 種別: テスト不足
- 対象: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py)
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L61-L63) に非更新確認観点が定義。
  - 実装: [tests/functions/organizationPositions/test_organizationPositions.py](tests/functions/organizationPositions/test_organizationPositions.py) では対象外データが更新されないことの検証が見当たらない。
- 指摘内容:
  - 更新系で副作用の混入を早期検知できない。
- 期待されるテスト:
  - 非更新確認: 他テナント・他役職レコードが不変であることの確認。

### 9. getOrganizationPositionAttribute のテストが空配列中心でマッピング検証が弱い
- 優先度: Low
- 種別: テスト不足
- 対象: [tests/functions/organizationPositionAttribute/test_organizationPositionAttribute.py](tests/functions/organizationPositionAttribute/test_organizationPositionAttribute.py#L12-L20)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/役職情報詳細.md](spec/基本設計/機能設計/070.組織情報/役職情報詳細.md) で追加情報項目定義を元に画面生成する前提。
  - 実装: [tests/functions/organizationPositionAttribute/test_organizationPositionAttribute.py](tests/functions/organizationPositionAttribute/test_organizationPositionAttribute.py#L12-L20) は空データ中心で、複数属性の整形確認が不足。
- 指摘内容:
  - 属性定義の変換不備を見逃す可能性がある。
- 期待されるテスト:
  - 正常系/データ入出力: 複数属性（入力型違い、必須、変更不可、表示順）のレスポンス整形確認。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - create/update で仕様どおりの業務結果（DB更新内容、ログ内容）まで確認するテストが不足。
- 入力値バリエーション不足:
  - orgPositionId/baseDate の不正入力、必須欠落、空文字パターンが不足。
- 境界値不足:
  - 100/101文字、999/1000、日付境界（有効開始日変更時の前後関係）不足。
- 分岐条件不足:
  - 更新時の「有効開始日が同じ/変わる」分岐、変更有無チェック分岐が不足。
- データ入出力不足:
  - 可変属性の差分判定、操作ログ内容、履歴更新後の値整合の検証が不足。
- 非更新確認不足:
  - 対象外レコード不変（他テナント・他役職）の確認不足。
- 異常系不足:
  - COM01009/COM01010/ORG05001/ORG05002/ORG05003 の個別エラー検証不足。

## 総評
- High 指摘は、役職未存在時と更新競合時の例外化、およびテスト構成の不足に集中しており、リリース前に是正が必要です。
- 実装は主要フローを備えていますが、業務エラーの返し分けと可変属性処理の堅牢性にリスクが残ります。
- 優先順位は、1) 例外化経路の業務エラー化、2) 更新分岐の堅牢化、3) 仕様準拠テスト拡充の順が妥当です。

## 残留リスク・確認できなかった範囲
- getOrganizationPositionAttribute の参照権限要件は、提示設計書内で明確に特定できず断定不可（設計確認事項として扱い）。
- 取得したテストは主に functions ハンドラ単体であり、サービス/リポジトリ層の直接単体テスト有無は限定的でした。
- 特になし