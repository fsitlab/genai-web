# 要件: 外部アプリ呼び出し Lambda の VPC 無効化オプション

## 背景

セルフホスト環境で AI アプリ開発を行う際、開発期間外（夜間・週末・長期休暇）にも以下の高額 AWS リソースが稼働し続け、想定外の課金が発生した。

### 課金の主因（マルチ AZ 構成での発生額・概算）

| リソース | 個数 | 月額概算 |
|---|---|---|
| NAT Gateway | 2（AZ ごと 1 個） | 約 $64 |
| Elastic IP（NAT 用） | 2 | 約 $7 |
| Interface VPC Endpoint（Secrets Manager） | 1（ENI × 2AZ） | 約 $15 |
| Gateway VPC Endpoint（DynamoDB） | 1 | $0（無料） |
| **合計** | | **約 $85 〜 90/月** |

これらは全て `packages/cdk/lib/construct/invoke-exapp-lambda-vpc.ts` 内で生成される。

## なぜこれらが作られているか

外部アプリ呼び出し Lambda（`invokeExApp.ts` / `pollExAppStatus.ts`）は、ユーザーが登録した任意の外部 API エンドポイントを `fetch()` で呼び出す。これを VPC 内 Lambda にすることで、NAT Gateway の EIP を経由した固定送信元 IP で外部アプリに到達でき、外部アプリ側で IP allowlist による絞り込みが可能になる、という設計意図がある。

ただし以下の理由で、ネットワーク層の IP 絞り込みは必須ではない。

- 外部アプリの API キーは Secrets Manager で管理済み
- 全通信が HTTPS
- 必要なら HMAC 署名 / OAuth2 / SigV4 等の認証強化が可能
- 一般的な SaaS 連携（Stripe / Slack 等）も IP 絞り込みは行っていない

## 目的

1. 開発停止期間の AWS コストを **約 $85/月** 削減できる手段を提供する
2. ただし既存ユーザーの環境（IP 絞り込みを利用中）には**影響を与えない**
3. CDK パラメータの `true/false` 切り替えだけでオン/オフできる
4. 切り替え後の再デプロイで自動的にリソースが削除・再生成される

## 非目標

- 既存の VPC ベース構成（`vpcIdForInvokeExApp` の挙動）の変更
- Lambda 関数コード（`lambda/*.ts`）の変更
- 他のスタック（GuardrailStack / AppDomainStack / CloudFrontWafStack）への影響

## 受け入れ条件

- [ ] 新パラメータ `disableExAppVpc` を `stack-input.ts` に追加（デフォルト `false`）
- [ ] `disableExAppVpc: false`（または未指定）の場合、従来通り VPC・NAT・EIP・VPCE が生成される
- [ ] `disableExAppVpc: true` の場合、VPC・NAT・EIP・VPCE が一切生成されず、Lambda 2 つが VPC 外で動作する
- [ ] Lambda コード本体は変更しない
- [ ] 既存の `vpcIdForInvokeExApp`（既存 VPC 参照モード）は機能を維持する
- [ ] ドキュメント（本ディレクトリ配下）が整備される
