# 設計: 外部アプリ呼び出し Lambda の VPC 無効化オプション

## 設計方針

- **追加のみ・削除なし**：既存コードは一切削除しない、後方互換を完全に保つ
- 新パラメータ `disableExAppVpc: boolean`（デフォルト `false`）
- `true` にした場合のみ、VPC/NAT/EIP/VPCE を一切作らず、Lambda は VPC 外で動作
- 既存の `vpcIdForInvokeExApp` 分岐は温存

## 動作モード（3 択）

| `disableExAppVpc` | `vpcIdForInvokeExApp` | 挙動 | NAT 等 |
|---|---|---|---|
| `false` （既定） | `""` | 新規 VPC を作成（従来挙動） | ✅ 作る |
| `false` （既定） | `"vpc-xxxx"` | 既存 VPC を参照 | その VPC のもの |
| **`true`（新規追加）** | 無視 | **VPC を使わない（サーバレス）** | ❌ 作らない |

## ネットワーク的な変化

### VPC モード（従来）
```
[Private Subnet の Lambda] → [NAT GW + EIP] → [IGW] → インターネット
```

### VPC 無効モード（新規）
```
[Lambda（AWS 管理 NW）] → インターネット
```
送信元 IP は AWS の動的 IP プールから払い出される（毎回変動・固定不可）。

## 修正点一覧（追加のみ）

### 1. `packages/cdk/lib/stack-input.ts`

`vpcIdForInvokeExApp` の隣に 1 行追加。
```ts
disableExAppVpc: z.boolean().default(false),
```

### 2. `packages/cdk/lib/generative-ai-use-cases-stack.ts`

`TeamAccessControlStack` 呼び出しの props に 1 行追加。
```ts
disableExAppVpc: params.disableExAppVpc,
```

### 3. `packages/cdk/lib/team-access-control-stack.ts`

- Props インターフェースに 1 行追加：`disableExAppVpc: boolean;`
- Construct 呼び出しに 1 行追加：`disableExAppVpc: props.disableExAppVpc,`

### 4. `packages/cdk/lib/construct/team-access-control.ts`

#### Props 拡張
```ts
disableExAppVpc: boolean;
```

#### VPC 生成ロジックを 3 択化
既存の if/else をラップする形で外側に分岐を 1 段追加：
```ts
let vpcForLambda: IVpc | undefined;
if (props.disableExAppVpc) {
  vpcForLambda = undefined;
} else if (props.vpcId && props.vpcId !== '') {
  vpcForLambda = Vpc.fromLookup(this, 'LookupExistingVpc', { vpcId: props.vpcId });
} else {
  const invokeExAppVpc = new InvokeExAppLambdaVpc(this, 'InvokeExAppVpc', { ... });
  vpcForLambda = invokeExAppVpc.vpc;
}
```

#### Lambda 2 つの VPC 設定を条件付きに
`pollExAppStatusFunction` と `invokeExAppFunction` の両方で、
```ts
vpc: vpcForLambda,
vpcSubnets: vpcForLambda ? { subnetType: SubnetType.PRIVATE_WITH_EGRESS } : undefined,
```
`vpc: undefined` を渡すと CDK は VPC 外 Lambda として生成する仕様を利用。

### 5. `packages/cdk/lib/construct/invoke-exapp-lambda-vpc.ts`

**変更なし**。`disableExAppVpc: true` の時に呼ばれなくなるだけで、ファイルは温存。

## 使い方

`packages/cdk/parameter.ts`（または環境ごとの設定ファイル）に追記：
```ts
disableExAppVpc: true,    // コスト削減モード（NAT/VPCE/EIP を作らない）
```
再デプロイで自動的にリソースが削除される：
```bash
cd packages/cdk
npx cdk diff
npx cdk deploy GenerativeAiUseCasesStack<env>
```

戻す場合は `false` にして再デプロイ（または該当行を削除）。

## トレードオフ

| 観点 | VPC モード | VPC 無効モード |
|---|---|---|
| 月額コスト | +$85〜90 | $0 |
| 送信元 IP の固定 | 可能（NAT EIP） | 不可（AWS プール） |
| 外部アプリ側 IP allowlist | 利用可 | 不可（API キー認証で代替） |
| Lambda コールドスタート | 遅い（ENI 作成） | 速い |
| 運用負荷 | NAT 等の管理あり | ほぼゼロ |

## リスクと対策

| リスク | 対策 |
|---|---|
| 既存ユーザーが意図せず VPC を消してしまう | デフォルト `false`、明示的に `true` 指定が必要 |
| IP allowlist 運用中ユーザーへの影響 | README で明示、外部アプリ側を API キー認証に切替推奨 |
| CDK の差分削除でロールバック失敗 | `cdk diff` で事前確認、別環境で検証してから本番反映 |

## 受け入れ条件への対応

requirement.md の受け入れ条件すべてを満たす。
