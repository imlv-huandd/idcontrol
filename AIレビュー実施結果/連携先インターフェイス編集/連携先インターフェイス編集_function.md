# 連携先インターフェイス編集 レビュー結果

## 指摘事項

### 1. 入力不正時に 500 エラーとなる（リクエストバリデーション例外未ハンドリング）
- 優先度: High
- 種別: 実装不備
- 対象: lowerInterfaces create/update API
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md L42-L44（異常系は不正入力時の結果が設計通りであること）
  - 実装: functions/lowerInterfaces/createRelationSystemNormalLowerInterface/app.py L32-L37、functions/lowerInterfaces/createRelationSystemIntegratedLowerInterface/app.py L32-L37、functions/lowerInterfaces/updateRelationSystemNormalLowerInterface/app.py L33-L38、functions/lowerInterfaces/updateRelationSystemIntegratedLowerInterface/app.py L33-L38（json.loads と Pydantic モデル生成が try の外）
- 指摘内容:
  - ボディ JSON 形式不正やスキーマ不一致時に発生する例外がハンドリングされず、API が 500 応答となる。
  - クライアント入力起因のエラーがサーバ内部エラーとして扱われ、利用者影響が大きい。
- 期待されるテスト:
  - JSON 不正（構文エラー）で 4xx（想定 400）を返すこと。
  - 必須項目欠落・型不一致・Enum 不正値で 4xx（想定 400）を返すこと。

### 2. ServiceException を一律 500 で返却しており、業務エラー/権限エラー/排他エラーの分類が不正
- 優先度: High
- 種別: 実装不備
- 対象: lowerInterfaces create/update API の例外マッピング
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md L33-L44（分岐条件・異常系で条件ごとに期待結果が変わること）
  - 実装: functions/lowerInterfaces/createRelationSystemNormalLowerInterface/app.py L62-L64（他3 API も同様に ServiceException を 500 返却）、shared/services/lowerInterfaceService.py L190-L191（COM01001 権限不足）、L471-L473（COM01010 排他競合）、L547-L549（COM01011 変更なし）
- 指摘内容:
  - サービス層は原因別にコードを分けている一方、API 層で全て 500 に潰している。
  - 権限不足や競合など利用者に再操作可能なケースでも内部障害扱いになり、運用・UI 制御・障害切り分けに悪影響。
- 期待されるテスト:
  - COM01001 発生時に 403、COM01010 発生時に 409、入力起因/業務バリデーション系で 400 系を返すマッピングテスト。

### 3. listRelationSystems API が受け取ったフィルタ条件を未使用
- 優先度: Middle
- 種別: 実装不備
- 対象: relationSystems/listRelationSystems/app.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md L27-L36（入力値バリエーション・分岐条件・データ入出力）
  - 実装: functions/relationSystems/listRelationSystems/app.py L29-L31（system_name/upper_or_lower/interface_name を取得）および L35（tenant_id のみで検索、取得した3条件を未使用）
- 指摘内容:
  - API はフィルタパラメータを受理しているが、結果形成に反映していない。
  - 仕様上パラメータを公開している場合、利用者期待と不一致になる。
- 期待されるテスト:
  - systemName/upperOrLower/interfaceName 各単体フィルタ、複合フィルタ、未指定時の全件返却を検証するテスト。

### 4. 連携先インターフェイス編集の主要 API（get/create/update）に対応する単体テストコードが不足
- 優先度: Middle
- 種別: テスト不足
- 対象: lowerInterfaces API 群
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md L24-L44（正常系、境界値、分岐、異常系などの網羅が必要）
  - 実装: functions/lowerInterfaces/getRelationSystemLowerInterface/app.py、functions/lowerInterfaces/createRelationSystemNormalLowerInterface/app.py、functions/lowerInterfaces/createRelationSystemIntegratedLowerInterface/app.py、functions/lowerInterfaces/updateRelationSystemNormalLowerInterface/app.py、functions/lowerInterfaces/updateRelationSystemIntegratedLowerInterface/app.py
- 指摘内容:
  - tests/functions 配下で当該 API 群に対するテストが確認できず、回帰検知ができない。
  - 既存テストは relationSystems/list 系に偏在しており、編集機能の主処理をカバーできていない。
- 期待されるテスト:
  - 各 API について正常系、権限不足、存在しないID、入力不正、依存先例外、期待レスポンス形状を網羅するテスト。

### 5. lowerInterfaceService の重要分岐（循環参照・自己参照・境界値・排他）に対する仕様検証テストが不足
- 優先度: Middle
- 種別: テスト不足
- 対象: shared/services/lowerInterfaceService.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md L27-L44（入力値バリエーション、境界値、分岐条件、非更新確認、異常系）
  - 実装: shared/services/lowerInterfaceService.py L495-L501（循環参照検出）、L1008-L1016（checkOffset/checkTime 値域）、L1039-L1042（項目重複）、L1047-L1050（自己参照禁止）、L471-L473/L723-L724（times不一致）
- 指摘内容:
  - 仕様リスクの高い分岐に対して、単体テストの存在が確認できない。
  - 変更時に業務ルール破壊や回帰不具合を検知できない構成。
- 期待されるテスト:
  - 循環参照あり/なし、自己参照、存在しない参照先、checkOffset 下限/上限/範囲外、checkTime 下限/上限/範囲外、times一致/不一致、変更なし更新を個別検証するテスト。

### 6. 権限要件とエラー応答仕様の設計記述が不足しており、実装・テスト方針がぶれやすい
- 優先度: Low
- 種別: 設計確認事項
- 対象: 連携先インターフェイス編集 機能設計書
- 根拠:
  - 設計書: spec/基本設計/機能設計/090.連携システム/連携先/連携先インターフェイス編集.md（画面項目・業務仕様中心で、HTTPステータス/エラーコード対応・権限要件の明示が弱い）
  - 実装: shared/services/lowerInterfaceService.py L190-L191（TENANT_ADMIN 権限チェック）、functions/lowerInterfaces/*/app.py（例外と HTTP 応答の実装）
- 指摘内容:
  - 設計書に API 応答規約（例外コードと HTTP ステータス対応）が明確でないため、実装側で一律 500 のような誤実装を招きやすい。
- 期待されるテスト:
  - なし（まず設計書で応答規約を明文化し、以後それに準拠したテストを定義すること）。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - lowerInterfaces の get/create/update 正常系テストが不足。
- 入力値バリエーション不足:
  - interfaceType/checkType/definitionType の組合せ、連携しない選択時の分岐、統合定義有無の検証不足。
- 境界値不足:
  - checkOffset（-99～99）、checkTime（0～23）、項目名文字数上限（100）などの境界未検証。
- 分岐条件不足:
  - 通常/統合、前段IF有無、循環参照有無、自己参照禁止、times不一致分岐のテスト不足。
- データ入出力不足:
  - listRelationSystems の各フィルタ反映、レスポンス整形（upper/lower interfaces）の条件別検証不足。
- 非更新確認不足:
  - 変更なし更新（COM01011）時に更新されないこと、不要な副作用がないことの検証不足。
- 異常系不足:
  - 不正JSON、必須欠落、型不正、権限不足、対象なし、楽観排他競合、依存先失敗時の応答検証不足。

## 総評
High 指摘は「入力不正時の 500 化」と「ServiceException の一律 500 化」で、利用者影響と運用影響が大きく、優先修正が必要です。
次点で、編集機能本体の単体テスト不足が顕著で、仕様逸脱や回帰を検知できない状態です。
まずは API 応答マッピングの是正と lowerInterfaces 一式の異常系・境界値テスト整備を最優先に進めるべきです。

## 残留リスク・確認できなかった範囲
- spec/詳細設計/openapi/paths/relation-systems/lower-interfaces.yaml の最新版との厳密突合は未実施（本レビューは提示設計書と実装・既存テストを主対象に実施）。
- 画面側（frontend）からの実際の呼び出しパラメータ制御、E2Eでのエラー表示整合は未確認。
