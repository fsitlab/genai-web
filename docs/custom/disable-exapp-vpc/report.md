# 実装レポート: 外部アプリ呼び出し Lambda の VPC 無効化オプション

## 実装サマリ

design.md / plan.md の通り、追加のみ・削除なしで 4 ファイルに変更を実施。Lambda コードおよび `InvokeExAppLambdaVpc` コンストラクトには手を入れていない。

## 変更ファイル一覧

| ファイル | 追加内容 |
|---|---|
| `packages/cdk/lib/stack-input.ts` | Zod スキーマに `disableExAppVpc: z.boolean().default(false)` |
| `packages/cdk/lib/generative-ai-use-cases-stack.ts` | `TeamAccessControlStack` に props 伝搬 |
| `packages/cdk/lib/team-access-control-stack.ts` | Props 拡張 + Construct 呼び出しに伝搬 |
| `packages/cdk/lib/construct/team-access-control.ts` | Props 拡張、VPC 生成を 3 択分岐、Lambda 2 つの `vpc` / `vpcSubnets` を条件付き |

総追加行数：約 10 行。削除：0 行。

## design / plan からの差分

差分なし。設計通りに実装完了。

## トラブルと対処

### 1. `npm install` 未実施で TypeScript の型定義が見つからない
- 症状: 最初の `npx tsc --noEmit` で `Cannot find type definition file for 'node'`
- 対処: ルートで `npm install` 実行（モノレポ構成のため）
- 結果: 型チェックは全て通過

### 2. `cdk synth` がアカウント未設定で失敗
- 症状: `Unable to parse environment specification "aws:///us-east-1". Expected format: aws://account/region`
- 原因: 本タスクの変更とは無関係の pre-existing な設定状況（AWS アカウント ID が `parameter.ts` に未設定）
- 対処: 静的検証としては TypeScript 型チェックの通過で十分と判断、`cdk synth` は実環境で実行
- 影響: なし（本変更による回帰ではない）

## レビュー結果

### コードレビュー（general-purpose エージェント）
- design.md と完全一致
- 後方互換性維持（デフォルト `false`）
- TypeScript 型整合性 OK（`IVpc | undefined` への変更は局所、`NodejsFunction.vpc` は optional のため波及なし）
- CDK 仕様上 `vpc: undefined` は VPC 外 Lambda となり正しい
- 改善余地（任意）：両パラメータ同時指定時の警告、`parameter.template.ts` 等のサンプル設定追記

### 機密情報レビュー（general-purpose エージェント）
- AWS アカウント ID / アクセスキー / トークン / パスワード等：検出なし
- 顧客名 / 案件名 / 社内固有名詞：検出なし
- 内部ホスト・ドメイン・Slack/Wiki/Confluence URL：検出なし
- 個人氏名 / メールアドレス：検出なし
- 内部 IP：`10.0.0.0/16` は CDK で新規作成する VPC の CIDR で公開可
- **結論：公開リポジトリ公開可**

## 動作確認方法（実環境）

```bash
# コスト削減モード ON
# packages/cdk/parameter.ts（または該当の設定ファイル）に追記
disableExAppVpc: true

cd packages/cdk
npx cdk diff   # NAT/EIP/VPCE/VPC が削除予定として表示されること
npx cdk deploy GenerativeAiUseCasesStack<env>
```

戻す場合は `false` にして再デプロイ、または該当行を削除。

## 期待されるコスト削減

| リソース | 個数 | 月額削減 |
|---|---|---|
| NAT Gateway | 2 | 約 $64 |
| EIP | 2 | 約 $7 |
| Interface VPC Endpoint (Secrets Manager) | 1 | 約 $15 |
| **合計** | | **約 $85〜90/月** |

## 残課題 / フォローアップ候補

- `parameter.template.ts` などサンプル設定ファイルへの記述追加（必要に応じて）
- GitHub Actions の cron で `disableExAppVpc` を夜間/週末に自動切替するワークフロー整備
- README への「コスト削減モード」セクション追加
- 外部アプリ側の認証強化（API キーローテーション / HMAC / OAuth 等）の運用ガイド
